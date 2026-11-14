# Laporan Analisis Source Code Mendalam  
## Tiko-Agent vs Hermes Agent

**Tanggal:** 14 November 2026  
**Metode:** Inspeksi langsung source code + dokumentasi arsitektur resmi kedua proyek  
**Repo Tiko:** https://github.com/kiosmarmer-commits/Tiko-Agent (Dart monorepo, ~505 file .dart, dibuat 7 Nov 2026)  
**Repo Hermes:** https://github.com/NousResearch/hermes-agent (Python + TypeScript, codebase sangat besar, ~263k stars, v0.23.x)

---

## 1. Metodologi Analisis

### Tiko-Agent
- Tree recursive penuh (650+ file)
- Pembacaan langsung source inti:
  - `packages/agent_core/lib/src/agent_loop.dart`
  - `packages/agent_core/lib/src/agent_team.dart`
  - `packages/tiko_runtime/lib/src/reflection_loop.dart`
  - `packages/agent_memory/lib/src/memory_layers.dart` + `rag_pipeline.dart`
  - `packages/agent_security/lib/src/prompt_guard.dart`
  - Struktur package, Web UI, Flutter client, INSTALL, seluruh docs parity
- CI history + commit terbaru

### Hermes Agent
- Dokumentasi arsitektur resmi (`architecture.md`, `agent-loop.md`, `AGENTS.md`)
- Struktur `agent/` (conversation_loop.py, turn_*.py, prompt_builder, memory_manager, dll.)
- Entry points (`run_agent.py`, `cli.py`, `gateway/run.py`)
- Data flow CLI / Gateway / Cron
- Subsystem tools, session storage, context compression, plugin system
- Tidak dilakukan full line-by-line audit seluruh codebase (terlalu besar), tetapi analisis level arsitektur + modul kunci sudah mendalam

---

## 2. Perbandingan Arsitektur Inti (Source-Level)

### 2.1 Agent Loop

| Aspek | Tiko-Agent (`agent_loop.dart`) | Hermes Agent (`conversation_loop.py` + `turn_*.py`) |
|-------|--------------------------------|-----------------------------------------------------|
| **Gaya** | ReAct-style klasik dengan budget guard | Turn-based orchestration yang sangat terurai (banyak phase file) |
| **Struktur** | Satu class `AgentLoop` yang cukup monolitik tapi modular (inject router, contextBuilder, gate, executor, reflector, dll.) | Facade `AIAgent` + `run_conversation()` yang dipecah ke banyak modul: `turn_iteration_prep`, `turn_api_call`, `turn_tool_round`, `turn_finalizer`, `turn_api_error`, dll. |
| **Budget** | `AgentConfig`: maxIterations, maxTokensPerTask, maxWallClockSec, maxToolCalls | Iteration budget + run-budget wall-clock + parent/child tracking |
| **Parallel tools** | `ParallelToolRunner` + `isToolParallelSafe` (hanya tool yang diaudit boleh paralel) | ThreadPoolExecutor (hingga 8 worker) + sequential fallback |
| **Loop detection** | `loop_detector.dart` | `repetition_guard.py` (runaway repetition) |
| **Streaming** | `onTextDelta` callback | Native streaming + interruptible API call (background thread + event) |
| **Reflection** | Optional `reflector` + `enableReflection` + `autoReflectOnTools` | Built-in learning loop + skill self-improvement (lebih terintegrasi) |
| **Kematangan** | Bersih, type-safe, mudah di-test, tapi masih “textbook ReAct” | Sangat production-hardened (interrupt, fallback, compression mid-turn, cache, multiple API modes) |

**Kesimpulan Agent Loop:**  
Tiko lebih bersih dan mudah dipahami (Dart + dependency injection jelas). Hermes jauh lebih robust untuk production (interruptible calls, multiple API modes, compression, repetition guard, prompt caching native).

### 2.2 Multi-Agent / Subagent Orchestration

**Tiko (`agent_team.dart`)** — diimplementasikan secara eksplisit:

```dart
enum AgentRole { planner, researcher, coder, critic, summarizer, custom }

class AgentSpec { id, role, systemPrompt, toolAllowlist, modelTier, maxTurns }
class AgentRegistry { register / byRole }
class Blackboard { shared slots + revision + writers }
class TaskGraph { DAG + cycle detection }
class TeamOrchestrator { topological waves + maxConcurrency + synthesis }
```

- Parallel wave execution
- Dependency injection executor (pure + testable)
- Blackboard sebagai shared memory antar agent

**Hermes:**  
Memiliki subagent / delegation dengan isolation, budget inheritance, dan permission. Namun implementasinya tersebar (tools + agent runtime helpers) dan tidak se-deklaratif DAG seperti Tiko. Fokus Hermes lebih ke “satu agent yang tumbuh” daripada multi-agent orchestration eksplisit.

**Keunggulan Tiko di sini:** Model multi-agent lebih terstruktur dan mudah diaudit.

### 2.3 Learning Loop & Reflection

**Tiko (`reflection_loop.dart` + SelfLearningEngine):**

- `Lesson` object: trigger → insight → action + timesReinforced
- Reflection setelah trajectory (sukses → positive lesson, gagal → corrective)
- `buildContextPrompt()` menyuntikkan lesson relevan ke prompt berikutnya
- Skill synthesis terpisah (`skill_synthesis.dart`)
- Persistensi JSON

**Hermes:**  
Closed learning loop yang sudah terbukti:
- Setelah complex task → tulis `SKILL.md` otomatis
- Skill self-improve saat digunakan
- Periodic nudge ke MEMORY.md / USER.md
- FTS5 session search + LLM summarization untuk recall lintas sesi

**Perbedaan filosofis:**  
Tiko memisahkan **reflection (lesson)** dan **skill synthesis**. Hermes menggabungkan keduanya dalam satu closed loop yang lebih organik dan sudah teruji di lapangan.

### 2.4 Memory System

**Tiko (`memory_layers.dart` + RAG):**

```dart
enum MemoryLayerType { episodic, semantic, procedural }

// Ebbinghaus-style decay
double retention(DateTime now, {required double halfLifeDays})
double effectiveScore(...) // importance × retention × access boost
```

- Layered memory (episodic → semantic consolidation)
- Advanced RAG: chunking, query expansion, hybrid (dense + BM25 + RRF), rerank, sitasi
- External memory providers (HTTP adapter Mem0-style)
- SQLite FTS5 + JSON fallback + vector index

**Hermes:**  
- File-based (`MEMORY.md`, `USER.md`) + SQLite session store + FTS5
- Memory manager + pluggable MemoryProvider
- Context compressor (lossy summarization)
- Lebih sederhana di permukaan, tapi sangat terintegrasi dengan prompt builder dan session lineage

**Keunggulan Tiko:** Model kognitif (layered + decay) lebih eksplisit dan advanced RAG pipeline lebih lengkap di level source.  
**Keunggulan Hermes:** Integrasi dengan session/prompt jauh lebih matang dan terbukti.

### 2.5 Security Layer

**Tiko (`prompt_guard.dart`):**

- Content-plane guard (bukan hanya tool gate)
- Deteksi prompt injection (PI-001 s/d PI-006): instruction override, system prompt extraction, chat-template delimiter, persona jailbreak, base64 blob, embedded tool-call
- Output leak filter: API key, GitHub token, private key, bearer, email, nomor telepon Indonesia
- `GuardVerdict`: allow / sanitize / block
- Anomaly rate detector
- Provenance-aware (web/email/mcp dinaikkan severity)

**Hermes:**  
- Command approval
- DM pairing
- Container isolation (banyak backend)
- Path guards
- Credential store protection
- Beberapa CVE historis sudah diperbaiki

**Keunggulan Tiko:** Prompt injection & PII leak detection di content plane sangat eksplisit dan terdokumentasi di source.  
**Keunggulan Hermes:** Isolation runtime (Docker/SSH/Modal dll.) jauh lebih lengkap.

### 2.6 Tools, MCP, Skills

| Aspek | Tiko | Hermes |
|-------|------|--------|
| Tool registry | Ya + parallelSafe flag | Ya + 70+ tools / 28 toolsets |
| MCP | stdio + Streamable HTTP + reconnect | Ya + dynamic |
| Skills format | agentskills.io compatible + converter | SKILL.md + Skills Hub resmi |
| Skills Hub | Client + security scanner + checksum | Marketplace resmi yang hidup |
| Browser | Local Chromium + Browserbase + Browser Use | Multiple backends |
| Sandbox | Conditional (host-dependent) | 7 terminal backends |

Hermes unggul signifikan di volume tools dan ekosistem skills yang sudah digunakan komunitas.

---

## 3. UI Desktop, Menu & Instalasi (Source + Packaging)

### Tiko
- **Web UI** (`apps/agentd/web`): Tab horizontal — Chat, Memori, Status, Approvals, Sessions, Skills, **Tugas (Kanban 4 kolom)**, Bantuan
- Kanban + live WebSocket event feed + audit log viewer (read-only)
- Flutter client: thin client ke daemon API (Chat, Memori, Status, Approvals, Sessions, Help)
- Desktop packaging: Flutter scaffolding + self-contained bootstrap (spawn `agentd`) masih P2
- Instalasi: `git clone` → `dart pub get` → `tiko setup` → `tiko doctor` → `tiko chat` / jalankan agentd

### Hermes
- Native Desktop app (Electron/React) signed, automatic update, guided onboarding
- Sidebar + chat-first, Group Chats, pinned Kanban, browser workspace
- TUI full-featured
- Instalasi: one-liner `curl | bash` / PowerShell / download installer
- Data di `~/.hermes` (mudah diinspeksi)

**Kesimpulan UI/UX:** Hermes jauh lebih mature untuk end-user. Tiko masih developer-oriented meskipun sudah punya fondasi tab + kanban yang bagus.

---

## 4. Desain Kode & Maintainability

| Aspek | Tiko | Hermes |
|-------|------|--------|
| Bahasa | Dart 3.8+ (type-safe, null-safe) | Python (dinamis) + TypeScript |
| Modularitas | Monorepo package sangat bersih | Facade + banyak sibling module (sangat terurai) |
| Testability | Dependency injection kuat, mudah unit test | Banyak test (~25k tests) |
| Ukuran | Kecil, mudah di-navigate | Sangat besar, butuh AGENTS.md + routing table |
| Dokumentasi internal | Sangat lengkap (bahasa Indonesia) + parity tracking | Dokumentasi arsitektur resmi sangat bagus |
| Fail-closed | Kuat (security, sandbox, secret) | Ada, tapi lebih kompleks |

Tiko menang di **kejelasan dan type safety**. Hermes menang di **battle-tested complexity handling**.

---

## 5. Ringkasan Kelebihan & Kekurangan (Source-Informed)

### Tiko-Agent — Kelebihan Source-Level
1. Arsitektur multi-agent (DAG + Blackboard + role) lebih deklaratif dan bersih.
2. Layered memory + Ebbinghaus decay + Advanced RAG pipeline lebih eksplisit.
3. PromptGuard content-plane security sangat detail (injection + leak rules).
4. Reflection loop + Lesson object terstruktur.
5. Type-safety Dart + monorepo yang mudah diaudit.
6. Dokumentasi internal yang jujur tentang gap vs Hermes.

### Tiko-Agent — Kekurangan Source-Level
1. Agent loop masih “textbook” dibandingkan Hermes yang sudah sangat hardened.
2. Belum ada distribusi binary / installer yang matang.
3. Ekosistem tools & skills masih jauh di belakang.
4. UI masih thin client; belum native polished experience.
5. Usia sangat muda → banyak fitur “implemented + unit tested” belum proven production.

### Hermes Agent — Kelebihan Source-Level
1. Conversation loop production-grade (interruptible, multi API mode, compression, fallback, repetition guard).
2. Learning loop yang sudah terbukti di skala besar.
3. 20+ gateway + Skills Hub + plugin ecosystem yang hidup.
4. Desktop app + TUI + instalasi yang sangat mudah.
5. Session storage + context management sangat matang.

### Hermes Agent — Kekurangan Source-Level
1. Codebase sangat kompleks (banyak sibling file) → learning curve tinggi.
2. Multi-agent orchestration tidak se-deklaratif Tiko.
3. Memory model lebih sederhana di permukaan (file-based + FTS).
4. Beberapa CVE historis menunjukkan permukaan serangan yang perlu dijaga.

---

## 6. Tabel Perbandingan Fungsi Kunci (Setelah Source Inspection)

| Fungsi | Tiko (Source) | Hermes (Source/Docs) | Pemenang |
|--------|---------------|----------------------|----------|
| Agent Loop robustness | Baik (budget + loop detector) | Sangat tinggi | Hermes |
| Multi-agent orchestration | Sangat baik (DAG eksplisit) | Ada tapi kurang deklaratif | **Tiko** |
| Learning / Reflection | Baik (Lesson + synthesis) | Excellent (proven closed loop) | Hermes |
| Memory model | Advanced (layered + decay + RAG) | Matang & terintegrasi | Seimbang (Tiko lebih advanced, Hermes lebih proven) |
| Prompt / Content security | Sangat baik (PromptGuard) | Baik (approval + isolation) | **Tiko** (content plane) |
| Tool & Skills ecosystem | Baik (ada fondasi) | Excellent | Hermes |
| UI Desktop polish | Sedang | Sangat tinggi | Hermes |
| Instalasi user baru | Rendah | Sangat tinggi | Hermes |
| Type safety & auditability | Tinggi | Sedang | **Tiko** |
| Production readiness | Early | Tinggi | Hermes |

---

## 7. Kesimpulan Akhir (Setelah Analisis Source)

Setelah memeriksa source code kedua proyek secara lebih mendalam:

- **Tiko-Agent** adalah reimplementasi yang **ambisius, bersih, dan well-architected**. Beberapa bagian (multi-agent DAG, layered memory + decay, PromptGuard, reflection Lesson) bahkan lebih terstruktur dan modern daripada yang terlihat di Hermes. Ini proyek yang sangat menarik bagi developer yang ingin memahami / membangun agent modern dari first principle dalam stack type-safe.

- **Hermes Agent** tetap unggul telak sebagai **produk siap pakai**. Conversation loop-nya jauh lebih hardened, learning loop sudah terbukti di skala besar, ekosistem skills/gateway sangat kaya, dan pengalaman instalasi + Desktop jauh lebih baik.

**Rekomendasi praktis:**
- Untuk **menggunakan agent sehari-hari** → Hermes Agent.
- Untuk **mempelajari arsitektur agent modern**, berkontribusi, atau membangun varian Dart/Flutter → Tiko-Agent sangat layak diikuti dan dikontribusi.
- Tiko memiliki fondasi teknis yang cukup kuat untuk mengejar parity, tetapi masih membutuhkan waktu signifikan di packaging, polishing UI, dan accumulation of real-world skills/usage data.

---

**File ini menggantikan laporan sebelumnya dengan kedalaman source-level yang lebih tinggi.**  
Analisis tetap terbatas oleh skala Hermes (tidak full line-by-line seluruh repo), tetapi modul-modul kunci kedua proyek sudah diperiksa secara langsung.
