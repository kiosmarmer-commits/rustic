# AUDIT MENYELURUH TIKO-AGENT DAN PERBANDINGAN DENGAN AI AGENT LAIN

**Tanggal audit:** 19 November 2026  
**Repositori:** <https://github.com/kiosmarmer-commits/Tiko-Agent>  
**Branch:** `main`  
**Commit yang diperiksa:** `42bd125ee7a4cb96278bd0a6eddb72afb6e224e2`  
**Commit:** merge PR #111 — `feat(gui): Voice UI end-to-end (dikte STT + balasan suara TTS)`  
**Jenis pemeriksaan:** static code review berbasis source tree, file kode, dokumentasi proyek, test suite, workflow CI, dan dokumentasi resmi produk pembanding.

---

## 1. Ringkasan eksekutif

### Kesimpulan utama

Tiko-Agent sudah mempunyai fondasi yang cukup matang untuk menjadi **self-hosted general-purpose agent platform**. Ia memiliki lebih dari sekadar chat dan tool calling: runtime ReAct, routing model, beberapa provider, memory/RAG, skill lifecycle, multi-agent, MCP, browser automation, scheduler, gateway pesan, permission gate, sandbox, observability, API, CLI, Flutter, dan dashboard web.

Dari sisi cakupan teknis, Tiko sudah mendekati kelas platform seperti Hermes Agent dan OpenClaw pada beberapa area backend. Namun, **cakupan fitur tidak sama dengan kualitas produk atau parity yang sudah terbukti**. Pesaing seperti Codex, Cursor, Claude Code, Antigravity, dan Nod unggul bukan hanya karena memiliki tools, tetapi karena alur kerja mereka terasa utuh: pengguna memahami apa yang akan dilakukan agent, dapat menyetujui atau membatalkan tindakan, melihat perubahan, memulihkan keadaan, dan mengevaluasi hasil.

Penilaian akhir audit:

- **Fondasi backend:** kuat dan luas.
- **Arsitektur modular:** cukup baik pada pembagian package, tetapi composition root dan beberapa UI entrypoint terlalu besar.
- **Algoritma agent:** sudah melampaui chatbot sederhana; masih perlu penguatan planner, recovery, evaluasi, dan multi-agent isolation.
- **Memory dan RAG:** menarik dan relatif maju; dokumentasinya belum konsisten dengan implementasi terbaru.
- **Security:** ada sejumlah komponen yang benar-benar penting—permission gate, capability, taint, sanitizer, prompt guard, credential manager, audit, dan OS sandbox. Namun, schema validation, timeout/cancellation, isolasi role, dan UX approval masih perlu diperketat.
- **UI/UX:** sudah lintas Flutter dan web, memiliki banyak panel admin, command palette, run inspector, dan voice. Masih ada ketimpangan kemampuan antarpermukaan, sebagian pengaturan belum dapat diedit dari web, dan pengalaman coding-agent belum setara dengan IDE agent khusus.
- **Kematangan operasional:** workflow CI luas, tetapi status CI spesifik untuk commit terbaru tidak berhasil diverifikasi melalui status/check yang tersedia pada saat audit.

**Rekomendasi strategis:** jangan mengejar jumlah fitur baru terlebih dahulu. Prioritaskan keandalan alur tindakan, isolasi agent, approval yang dapat dijelaskan, recovery/checkpoint, konsistensi UI, dan benchmark end-to-end. Dengan itu, fitur yang sudah ada akan terasa jauh lebih kuat.

### Posisi produk yang disarankan

Tiko sebaiknya tidak diposisikan hanya sebagai “klon Hermes”, “klon Codex”, atau “chatbot lokal”. Posisi paling menjanjikan adalah:

> **Tiko — Personal and Developer Agent Workbench: satu agent self-hosted untuk chat, coding, riset, otomasi, memori pribadi, dan integrasi aplikasi, dengan kontrol izin yang transparan dan UI konsisten di Android, desktop, serta web.**

Keunggulan diferensiasi yang patut dipertahankan: self-hosted, pilihan provider, dukungan bahasa Indonesia, Flutter lintas platform, gateway multi-channel, memory/RAG, skill yang dapat dievaluasi, serta permission dan sandbox yang dikendalikan runtime.

---

## 2. Cakupan audit dan batas verifikasi

### 2.1 Data repositori

Snapshot branch `main` yang diperiksa menunjukan:

| Item | Hasil inventaris |
|---|---:|
| File dalam Git tree | 957 |
| Dokumen Markdown | 141 |
| File test yang terdeteksi pada tree | 346 |
| Package pada direktori `packages/` | 12 |
| Workflow GitHub Actions utama | `.github/workflows/ci.yml` |

Angka 346 adalah **jumlah file test**, bukan jumlah test case. Jumlah test case aktual harus diambil dari laporan eksekusi test.

Commit terbaru pada snapshot audit adalah merge PR #111 terkait voice UI: STT/dikte pada composer dan TTS/balasan suara pada Flutter dan web.

### 2.2 Area yang diperiksa

Pemeriksaan mencakup inventaris package, README, struktur runtime, agent loop, router, context builder, multi-agent orchestration, tool registry, sandbox, runtime composition, gateway/UI, dokumentasi arsitektur, dokumen parity, beberapa test kritis, serta isi CI workflow.

File penting yang diperiksa lebih dekat:

- `packages/agent_core/lib/src/agent_loop.dart`
- `packages/agent_core/lib/src/router.dart`
- `packages/agent_core/lib/src/context_builder.dart`
- `packages/agent_core/lib/src/agent_team.dart`
- `packages/agent_tools/lib/src/registry.dart`
- `packages/agent_tools/lib/src/sandbox.dart`
- `packages/tiko_runtime/lib/src/tiko_runtime.dart`
- `packages/tiko_runtime/lib/src/tiko_runtime_internals.dart`
- `packages/tiko_runtime/lib/src/team_tool.dart`
- `apps/agentd/bin/agentd.dart`
- `apps/agentd/web/app.js`
- `apps/agent_flutter/lib/main.dart`
- `apps/tiko_widgets/`
- `.github/workflows/ci.yml`

### 2.3 Batasan penting

Audit ini merupakan **static audit**, bukan hasil menjalankan seluruh 957 file.

- Saya tidak menjalankan `dart test`, `flutter test`, `flutter analyze`, build Android, atau build desktop pada mesin fisik dalam audit ini.
- Saya tidak melakukan penetration test dinamis terhadap daemon, MCP server, browser, Telegram, Discord, WhatsApp, email, maupun provider nyata.
- Status/check CI khusus commit `42bd125...` tidak tersedia dari hasil pemeriksaan status GitHub yang dapat diakses. Workflow CI ada dan cakupannya cukup luas, tetapi saya tidak menyatakan commit terbaru sudah hijau atau merah tanpa bukti run.
- Klaim fitur dari dokumen proyek diperlakukan sebagai klaim implementasi sampai terdapat jalur kode dan test yang mendukungnya.

Karena itu, temuan dibagi menjadi **bug yang dapat ditunjuk pada source**, **risiko desain yang perlu diuji**, **gap integrasi/UI**, dan **inkonsistensi dokumentasi**.

---

## 3. Peta arsitektur Tiko-Agent

### 3.1 Susunan utama

| Lapisan | Package / aplikasi | Tanggung jawab |
|---|---|---|
| Agent core | `agent_core` | ReAct loop, router, context, budget, compaction, plan/execute, skills, task graph, evaluator |
| Provider | `agent_providers` | Adapter LLM, provider routing, embedding, health/circuit breaker, prompt cache |
| Tool execution | `agent_tools` | Registry, browser, filesystem, shell/code execution, checkpoint, MCP, API/ACP, subagent |
| Memory | `agent_memory` | SQLite/JSON store, FTS/BM25, vector, RAG, layered memory, external memory providers |
| Security | `agent_security` | Permission gate, prompt guard, secret redaction, credential handling, shell policy, kill switch, audit-related controls |
| Gateway | `agent_gateway` | HTTP/WebSocket, Telegram, Discord, Slack, WhatsApp, email, webhook, voice dan channel adapter |
| Scheduler | `agent_scheduler` | Job store, cron/schedule, delivery/retry |
| Composition root | `tiko_runtime` | Menggabungkan provider, memory, tool, security, session, scheduler, plugin, skill, subagent dan gateway |
| Daemon | `apps/agentd` | HTTP API, event stream, server lifecycle dan web UI |
| Client | `apps/agent_flutter` | Flutter untuk Android, web, desktop |
| Widget library | `apps/tiko_widgets` | GenUI renderer dan komponen chat reusable |
| CLI/TUI | `apps/tiko`, `apps/agent_cli`, `tiko_tui` | Interaksi terminal, setup, konfigurasi, diagnosis dan ACP |

### 3.2 Alur satu permintaan

Alur konseptual runtime:

1. Pesan masuk dari CLI, Flutter, web, API, atau gateway.
2. Runtime mengidentifikasi session, user, channel, bot/persona, serta lock sesi.
3. Router memilih provider/tier berdasarkan kompleksitas, kebutuhan tool, dan sinyal risiko.
4. RuntimeContext menyusun system context, history, memory, persona, skill, dan tool schema.
5. AgentLoop menjalankan siklus model → tool request → permission check → eksekusi → tool result → context update.
6. Budget guard, loop detector, cancellation, dan compaction menjaga proses agar tidak berputar tanpa batas.
7. Output disanitasi, history/memory dicatat, telemetry dan audit diperbarui, lalu jawaban dikirim ke permukaan pengguna.

Hal positif: `TikoRuntime` membangun `AgentLoop` untuk satu turn di jalur `_chatWithStatusCore`, bukan memakai satu objek loop global yang sama untuk semua turn. Lock juga berbasis session. Ini mengurangi risiko state mutable antar-session yang sering muncul pada agent loop singleton. Namun, shared store, scheduler, approval broker, tool process, dan event emitter tetap memerlukan pengujian concurrency terpisah.

### 3.3 Kelebihan arsitektur

- Pemisahan core/provider/tools/memory/security/gateway cukup jelas.
- Host dapat memasang provider dan executor tanpa menanam ketergantungan UI di core.
- `RuntimeContext` dapat menyaring tool berdasarkan role/session dan mengurangi jumlah schema yang dikirim ke model.
- Runtime memiliki titik komposisi yang jelas untuk permission, memory, provider, skill, scheduler, dan channel.
- CLI, web, Flutter, dan gateway dapat menggunakan runtime yang sama.
- Ada mode eco/resource profile dan fallback storage, relevan untuk mesin terbatas dan Android/Termux.

### 3.4 Kelemahan arsitektur

- `tiko_runtime.dart` berukuran sekitar 148 KB; `apps/agent_flutter/lib/main.dart` sekitar 83 KB; `apps/agentd/web/app.js` sekitar 80 KB; `http_api.dart` sekitar 59 KB; `tiko_config.dart` sekitar 62 KB. Ini menandakan konsentrasi logika yang tinggi pada beberapa file entrypoint.
- Banyak fitur harus di-wire manual ke composition root. Riwayat dokumen proyek sendiri berulang kali mencatat modul yang semula hanya tersedia sebagai package kemudian perlu diintegrasikan ke runtime/daemon.
- UI Flutter dan web belum sepenuhnya mempunyai kemampuan edit yang setara.
- Kontrak event stream belum sekuat kontrak API typed: dokumentasi terbaru menyebut payload web masih raw `Map`, sementara WebSocket belum memiliki `event_id` dan `schema_version` yang baku.
- Beberapa dokumentasi inti masih merekam snapshot arsitektur dan status CI lama sehingga sulit dijadikan sumber kebenaran tunggal.

**Rekomendasi arsitektur:** pertahankan package boundaries, tetapi pecah runtime berdasarkan service/domain (chat orchestration, tool policy, provider lifecycle, memory lifecycle, skills, jobs, channel routing, security telemetry) dan gunakan interface yang eksplisit. Composition root harus hanya merakit service, bukan menjadi tempat semua aturan bisnis.

---

## 4. Audit algoritma dan kemampuan agent

### 4.1 ReAct agent loop

Tiko menggunakan pendekatan ReAct: model membaca konteks, menghasilkan jawaban atau tool call, runtime menjalankan tool setelah izin, hasil tool masuk kembali ke konteks, dan siklus berulang sampai selesai atau budget habis.

Komponen yang terlihat di implementasi:

- batas iterasi, token, waktu, dan jumlah tool call;
- deteksi pengulangan tool dan argumen yang sama;
- parallel tool runner untuk tool yang ditandai aman paralel;
- cancellation token yang diteruskan ke konteks tool;
- permission/approval sebelum eksekusi;
- context compaction untuk menjaga jendela konteks;
- sanitasi output dan pencatatan tool trace.

**Penilaian:** fondasi loop sudah tepat. Perbedaan dengan agen coding terbaik bukan pada loop dasar, tetapi pada kualitas recovery, state management, patch safety, evaluasi hasil, dan pengalaman mengawasi proses.

### 4.2 Router dan provider selection

Router mendukung tier provider, fallback, health state, serta jalur pengambilan keputusan menggunakan decision provider atau heuristic. Saat decision provider tidak tersedia atau gagal, sistem dapat kembali ke heuristic.

Kekuatan:

- model tier dapat diatur;
- provider health menggunakan state closed/open/half-open;
- fallback mengurangi ketergantungan pada satu provider;
- runtime mempunyai API untuk health/circuit-breaker yang dibagi dengan router;
- latensi provider dapat dicatat.

Kelemahan/risiko:

- heuristic berbasis teks dan panjang prompt tidak cukup untuk menjadi satu-satunya penentu kompleksitas atau risiko;
- hasil klasifikasi model jangan pernah menjadi satu-satunya kontrol keamanan;
- belum ada benchmark terukur yang menunjukkan routing benar-benar menurunkan biaya tanpa mengurangi keberhasilan.

**Rekomendasi:** routing harus memisahkan tiga keputusan: (a) kelas tugas, (b) model/tier optimal berdasarkan latency/biaya/kualitas, dan (c) policy keamanan. Gunakan sinyal aktual seperti jumlah file, jenis tool, riwayat kegagalan, token budget, latency provider, dan hasil benchmark, bukan hanya panjang teks.

### 4.3 Memory dan RAG

Tiko mempunyai memory store berbasis SQLite dan JSON fallback, FTS5/BM25, vector index, hybrid retrieval, owner scope, provenance/confidence, TTL/state, layered memory, curated memory, dan memory provider eksternal.

Salah satu batch optimisasi terbaru mendokumentasikan:

- `SqliteVecIndex` dipasang sebagai default saat SQLite tersedia;
- embedding chunk dicache saat ingest;
- hybrid rerank dengan formula `0.4*fusedNorm + 0.4*semantic + 0.2*overlap` ketika embedder aktif;
- jalur leksikal dipertahankan ketika embedder tidak ada.

Kekuatan:

- memory bukan sekadar menempelkan seluruh transcript; ada retrieval;
- owner scope mencegah retrieval lintas pemilik;
- retrieved memory diperlakukan sebagai data tidak tepercaya, bukan instruksi sistem;
- memory dapat dipisahkan menjadi episodic/semantic dan mengalami decay;
- tersedia external memory provider.

Kelemahan/risiko:

- dokumen `docs/04-MEMORI-DB-PENCARIAN.md` masih menyebut runtime selalu memakai `BruteForceVectorIndex` dan menegaskan `sqlite-vec` belum menjadi default. Pernyataan itu bertentangan dengan source `tiko_runtime.dart` dan dokumen optimisasi 13 November.
- formula retrieval belum cukup untuk membuktikan kualitas recall; dibutuhkan corpus tanya-jawab yang dapat mengukur Precision@K, Recall@K, MRR/NDCG, freshness, dan false memory.
- konsolidasi memory via LLM tetap berisiko mengubah detail, menghilangkan qualifier, atau menggabungkan fakta yang tidak kompatibel. Proyek sudah cukup jujur menyatakan proses ini opt-in dan tidak menghapus original entry; pertahankan prinsip tersebut.

**Rekomendasi:** buat evaluasi retrieval 300–1.000 query yang diberi label; setiap fakta harus menyimpan provenance, timestamp, confidence, scope, dan status verifikasi. UI Memory harus memberi bukti “fakta ini berasal dari pesan/file mana”, serta tombol edit, forget, merge, dan conflict resolution.

### 4.4 Self-learning dan skill lifecycle

Tiko mengembangkan skill lifecycle yang lebih matang daripada “agent menulis file skill lalu langsung mempercayainya”. Jalur yang didokumentasikan meliputi draft → scan → golden evaluation → trial/approval → approved, dengan rollback/retired. Ada feedback 👍/👎, metric ledger, sink, reflection/lesson, trajectory export, dan prompt-injection eval corpus Indonesia.

Kekuatan:

- skill tidak langsung mendapat privilege hanya karena dihasilkan model;
- ada scan, evaluasi, dan approval;
- feedback end-to-end mulai tersedia pada API dan web UI;
- ada evaluasi prompt injection Indonesia sebanyak 40 contoh berlabel;
- trajectory dapat diekspor untuk pipeline training eksternal.

Batasan:

- istilah “self-learning” jangan disamakan dengan model yang memperbarui bobot/parameter secara mandiri;
- trajectory export dan reward function belum sama dengan reinforcement learning training;
- refleksi dan pembuatan skill dapat memperbaiki prosedur, tetapi tetap memerlukan test dan evaluasi independen;
- korpus 40 prompt injection berguna sebagai regression seed, tetapi terlalu kecil untuk menyimpulkan ketahanan keamanan secara umum.

**Rekomendasi:** gunakan istilah yang akurat seperti *skill learning*, *self-improvement loop*, atau *procedure optimization*. Bangun benchmark berlapis, jangan menyatakan agen “belajar sendiri” bila tidak ada metrik sebelum/sesudah dan bukti skill tidak merusak task lain.

### 4.5 Multi-agent dan task orchestration

Tiko mempunyai `AgentSpec`, `AgentRegistry`, `TaskGraph`, DAG validation, topological waves, `Blackboard`, parallel execution dengan batas concurrency, dan synthesizer. `AgentTeamTool` menghubungkan orchestrator dengan turn runtime. Ada pula `SpawnSubagentTool` dengan depth/budget/allowed tools.

Kekuatan:

- node independen dapat diproses paralel;
- dependensi task diperiksa;
- output upstream dapat dimasukkan ke prompt node berikutnya;
- budget/tier dapat ditentukan untuk anggota team;
- cancellation telah dirancang dan diteruskan ke beberapa lapisan.

Gap:

- tidak semua anggota agent mempunyai workspace, filesystem root, credential scope, session store, dan model profile yang benar-benar terisolasi seperti multi-agent OpenClaw;
- empty `toolAllowlist` pada `AgentSpec` berarti mewarisi seluruh tool registry host. Ini fleksibel, tetapi berbahaya sebagai default untuk agent role yang tidak dimaksudkan memiliki akses penuh;
- terdapat bug cancellation pada hasil node tersisa (lihat bagian bug);
- `systemPrompt` anggota team disisipkan menjadi teks di prompt user, bukan dikirim sebagai system message terpisah (lihat temuan).

**Rekomendasi:** jadikan agent role konfigurasi keamanan utuh: system prompt, tools, workspace, network, credential, memory scope, model tier, token budget, max turns, dan approval mode. Default tool access harus minimal; inheritance harus memerlukan opsi eksplisit.

### 4.6 Security dan permission

Komponen security yang teridentifikasi meliputi permission matrix, ask/auto/deny mode, capability-aware gate, capability token, taint/provenance, prompt guard, sanitizer, Tirith/shell policy, credential manager, secret store, plugin consent, token anomaly detector, audit trail, kill switch/safety state, dan OS sandbox seperti bubblewrap, Docker, SSH, Modal, dan Daytona.

Kekuatan:

- approval tidak hanya bergantung instruksi prompt; ada gate runtime;
- capability token dapat diikat ke tool/session/argumen;
- unknown tool dapat ditolak oleh gate;
- `SandboxKind.none` sengaja tidak menjalankan child process host;
- Docker sandbox menggunakan network none, read-only root, drop capabilities, no-new-privileges, limit resource, dan workspace mount;
- browser dan tool output dapat ditandai sebagai data tidak tepercaya;
- credential/secrets tidak seharusnya masuk ke prompt model.

Kelemahan/risiko:

- schema validation di ToolRegistry hanya subset JSON Schema dan tidak menolak semua constraint yang tidak dipahami;
- timeout Future tidak membunuh proses/side effect yang sedang berjalan;
- policy per plugin/tool harus lebih granular dan bisa diaudit;
- Security Center masih read-only; tidak ada tombol panic/kill switch/safe mode dari GUI;
- security snapshot hanya menunjukkan sebagian ringkas; beberapa counter in-memory reset saat restart dan audit card menampilkan 15 item terakhir;
- dokumentasi sandbox membuat pernyataan yang bertentangan dengan cabang kode `dispatchFor`.

**Rekomendasi:** policy harus menjadi kontrol server/runtime, bukan prompt. Setiap side-effect tool harus mempunyai precondition, idempotency key, permission scope, cancellation handler, resource scope, dan audit record. Approval harus menunjukkan target tindakan dan data yang akan berubah.

---

## 5. Bug, risiko, dan inkonsistensi yang ditemukan

Severity yang dipakai:

- **P0:** dapat menyebabkan akses tidak sah, kehilangan data, atau tindakan eksternal yang tidak dapat dipulihkan; harus menahan rilis bila terekspos.
- **P1:** risiko fungsional/keamanan serius; harus diperbaiki sebelum mengandalkan fitur bersangkutan untuk penggunaan produksi.
- **P2:** bug/ketidakkonsistenan moderat atau gap UX/operasional yang harus masuk roadmap dekat.
- **P3:** peningkatan kualitas dan pemeliharaan.

### 5.1 P1 — Bug pencatatan node saat multi-agent dibatalkan

**Lokasi:** `packages/agent_core/lib/src/agent_team.dart`, sekitar baris 315.  
**Bukti:** kode menggunakan `graph.waves().skip(results.length)`.

`results.length` menghitung jumlah task result yang sudah selesai, bukan nomor wave DAG. Jika wave pertama berisi 2 node dan wave kedua berisi 1 node, setelah wave pertama selesai `results.length == 2`. Ketika cancellation terjadi sebelum wave kedua, `graph.waves().skip(2)` melewati dua wave, sehingga node pada wave kedua tidak tercatat sebagai cancelled result.

**Dampak:** hasil team dapat kehilangan task yang tidak pernah dijalankan. UI, synthesizer, dan caller dapat salah mengira graph sudah selesai tanpa mengetahui node mana yang terlewat.

**Perbaikan:** hitung `final waves = graph.waves()` lalu iterasi dengan indeks. Ketika cancellation terjadi pada indeks `i`, catat semua node dari `waves.skip(i)` sebagai cancelled. Jangan memakai jumlah result sebagai indeks wave.

**Test wajib:**

1. cancellation sebelum task pertama;
2. cancellation setelah wave pertama berisi dua atau lebih node selesai;
3. cancellation ketika batch paralel sedang berjalan;
4. seluruh task harus muncul tepat satu kali dalam `results`;
5. semua node downstream yang tak dijalankan harus berstatus cancelled, bukan hilang.

### 5.2 P1 — `AgentSpec.systemPrompt` bukan system message sesungguhnya

**Lokasi:** `packages/tiko_runtime/lib/src/team_tool.dart`, sekitar baris 147–150; wiring runtime di `packages/tiko_runtime/lib/src/tiko_runtime.dart` sekitar baris 1168–1176.

`AgentTeamTool` menggabungkan `agent.systemPrompt` dengan `node.prompt` menjadi satu string, lalu memberikan string itu ke runner. Runner runtime meneruskan string tersebut sebagai `prompt` pada `rt.chatWithStatus(...)`. Dengan demikian, instruksi yang bernama `systemPrompt` masuk sebagai isi pesan user, bukan system role terpisah di `RuntimeContext`.

**Dampak:** planner/researcher/coder/critic tidak memiliki pemisahan instruksi setingkat system role. Instruksi role lebih mudah ditimpa oleh konten task atau instruksi lain di user message. Kontrak API juga menamai field tersebut lebih kuat daripada perilaku aktualnya.

**Perbaikan:** berikan `personaPrompt`/`agentInstructions` sebagai parameter terpisah ke runtime, lalu masukkan melalui system context. Alternatifnya, buat `AgentTurnRequest` dengan `systemInstructions`, `userTask`, `toolPolicy`, `workspace`, dan `budget` yang terpisah.

**Test wajib:** buat agent dengan system instruction yang membatasi tugas, berikan user prompt yang bertentangan, dan pastikan system instruction tetap berada pada system message/context serta kebijakan tool tidak berubah.

### 5.3 P1 — Timeout tool tidak menghentikan future/aksi yang sedang berjalan

**Lokasi:** `packages/agent_tools/lib/src/registry.dart`, sekitar baris 165.

Executor melakukan `.timeout(Duration(seconds: tool.spec.timeoutSec))`. Semantik Dart `Future.timeout` adalah berhenti menunggu future sumber dan mengembalikan timeout; future sumber dapat tetap berjalan dan menyelesaikan aksi belakangan.

**Dampak:** jika tool melakukan aksi eksternal (mengirim email, update tiket, menulis file, mengirim pesan, atau mengubah resource), runtime dapat melaporkan timeout sementara aksi yang asli masih berjalan. Pengguna dapat mencoba ulang dan menyebabkan aksi ganda.

**Perbaikan:**

- definisikan `CancellationToken` per tool call;
- wajibkan adapter tool meneruskan token ke HTTP client/process/browser;
- ketika timeout, kirim cancel/abort dan tunggu status penghentian terbatas;
- untuk operasi eksternal, gunakan idempotency key berbasis session + turn + tool + args;
- tampilkan status `timed_out_but_may_still_be_running` jika backend tidak dapat menjamin pembatalan.

**Test wajib:** tool fake melakukan side effect tertunda; setelah timeout, pastikan side effect dibatalkan atau sistem mengungkap status yang belum pasti dan tidak mengizinkan retry non-idempotent diam-diam.

### 5.4 P1/P2 — Validasi input tool tidak mencakup seluruh constraint JSON Schema

**Lokasi:** `packages/agent_tools/lib/src/registry.dart`, sekitar baris 75–125.

Validator menyebut dirinya sebagai subset JSON Schema. Pemeriksaan yang terlihat mencakup tipe dasar, required, minLength, enum, dan object rekursif. Namun, array hanya diperiksa apakah berupa `List`; `items`, `minItems`, dan `maxItems` tidak diperiksa. `number` tidak menerapkan minimum/maximum, dan banyak constraint lain tidak didukung. Field tambahan juga diterima secara default tanpa menerapkan `additionalProperties: false`.

**Dampak:** argumen yang melewati validasi belum tentu memenuhi schema tool/MCP. Ini bisa menyebabkan kegagalan downstream, perilaku tidak diharapkan, atau validasi keamanan yang tampak ada tetapi tidak lengkap.

**Perbaikan:** gunakan implementasi JSON Schema validator yang kompatibel dengan subset yang dibutuhkan, atau buat validator subset yang secara eksplisit menolak schema keyword yang belum didukung. Untuk tool eksternal, default harus fail-closed saat schema tidak dapat divalidasi penuh.

**Test wajib:** nested object, array item type, min/max item, number bounds, regex/pattern, format, enum, unknown keyword, `additionalProperties: false`, dan schema MCP.

### 5.5 P2 — Dokumentasi sandbox bertentangan dengan kode

**Lokasi:** `packages/agent_tools/lib/src/sandbox.dart`, sekitar baris 184.

Komentar `dispatchFor` menyatakan `SandboxKind.none` tidak pernah dikembalikan. Namun, kode menetapkan `kind = SandboxKind.none` ketika tidak tersedia bubblewrap/Docker/SSH/Modal/Daytona, kemudian tetap mengembalikan `SandboxRunner(kind: kind)`.

**Penilaian:** ini adalah inkonsistensi komentar/API contract, bukan bukti bahwa eksekusi host tanpa sandbox diizinkan. Jalur `SandboxKind.none` sendiri memiliki guard yang menolak eksekusi proses, dan sejumlah caller secara eksplisit menonaktifkan script/MCP ketika sandbox tidak tersedia.

**Perbaikan:** ubah komentar menjadi “`dispatchFor` dapat mengembalikan runner dengan kind `none`; runner tersebut fail-closed dan caller harus menampilkan capability disabled”, atau ubah return type menjadi nullable/Result agar kontraknya eksplisit. Tambahkan test ketika tidak ada backend.

### 5.6 P2 — `toolAllowlist` kosong mewarisi semua tool host

**Lokasi:** `packages/agent_core/lib/src/agent_team.dart` pada dokumentasi `AgentSpec`; wiring di `tiko_runtime.dart` sekitar baris 1170.

Komentar `AgentSpec` menyatakan allowlist kosong berarti anggota mewarisi seluruh registry. Runtime juga meneruskan `null` filter saat list kosong.

**Dampak:** role yang hanya dimaksudkan untuk perencanaan atau ringkasan dapat memiliki tool yang tidak dibutuhkan bila konfigurasi lupa diisi. Ini bukan selalu bug karena perilaku tersebut disengaja, tetapi default ini memperlebar blast radius salah konfigurasi.

**Perbaikan:** jadikan default `deny all`; tambahkan `inheritHostTools: true` untuk pilihan eksplisit; sediakan default tool per role (planner read-only, researcher web/read, coder file/test, critic read-only).

### 5.7 P2 — Dokumentasi memory sudah kedaluwarsa

**Lokasi:** `docs/04-MEMORI-DB-PENCARIAN.md` dibandingkan source runtime dan `docs/HERMES-PARITY-BATCH-2026-11-13.md`.

Dokumen memory menyatakan runtime selalu memakai `BruteForceVectorIndex` dan menyebut `sqlite-vec` belum terhubung. Source `tiko_runtime.dart` justru membuat `SqliteVecIndex(sqlite.db)` sebagai index default saat SQLite tersedia; dokumen optimisasi 13 November juga menyatakan hal tersebut.

**Dampak:** developer atau agent lain dapat mengambil keputusan arsitektur salah, menghapus wiring yang sebenarnya aktif, atau menganggap fitur belum tersedia.

**Perbaikan:** dokumen arsitektur harus dihasilkan/ditinjau terhadap source terkini. Setiap dokumen status wajib memiliki commit SHA dan tanggal, serta badge `CURRENT`, `HISTORICAL`, atau `SUPERSEDED`.

### 5.8 P2 — Indeks dokumentasi mengarah ke baseline lama

**Lokasi:** `docs/00-INDEX.md`.

Indeks menyebut baseline `main` pada commit `dfa7a0...` dan status CI berdasarkan run lama; indeks juga mengatakan beberapa bundle belum dikirim dan belum tervalidasi. Ini bukan snapshot commit yang sedang diaudit (`42bd125...`).

**Dampak:** pengguna dan agent yang membaca indeks terlebih dahulu berisiko salah memahami keadaan branch utama.

**Perbaikan:** jadikan satu `STATUS.md`/`docs/STATUS-CURRENT.md` yang otomatis diperbarui pada CI, berisi commit SHA, tanggal, status checks, toolchain, test count, build target, serta bagian yang belum diverifikasi.

### 5.9 P2 — Klaim persentase paritas Hermes tidak mempunyai metodologi objektif

**Lokasi:** `HERMES-TIKO-COMPARISON-2026-11-10.md`.

Dokumen menyebut estimasi parity sekitar 93%. Angka ini merupakan estimasi internal dan tidak dilengkapi benchmark task set, bobot kategori, definisi “implemented vs integrated vs verified”, atau evaluasi UX independen.

**Dampak:** angka persentase dapat memberi kesan Tiko sudah setara secara keseluruhan padahal tool availability, pengalaman pengguna, stabilitas, dan keamanan masih mempunyai perbedaan besar.

**Perbaikan:** hapus angka parity tunggal. Ganti dengan matriks fitur: `Source exists`, `Runtime wired`, `Tested`, `Platform verified`, `Production ready`; tampilkan tanggal dan bukti untuk setiap baris.

### 5.10 P2 — Web UI dan API belum sepenuhnya typed/parity

**Lokasi:** `apps/agentd/web/app.js`, `packages/agent_gateway/lib/src/http_api.dart`, `docs/API-CONTRACT-TYPED-2026-11-18.md`, `docs/WEB-PARITY-GUI-2026-11-18.md`.

Flutter memiliki DTO dan repository typed, sedangkan web masih memakai raw `Map`; event WebSocket belum mempunyai `event_id`/`schema_version` baku dan belum memiliki korelasi request yang konsisten. Dokumentasi parity juga menyebut beberapa operasi web belum tersedia sebagai edit action: edit bot tertentu masih melalui Flutter/CLI, perubahan channel memerlukan env + restart, plugin uninstall/MCP admin tidak tersedia, dan Security Center read-only.

**Dampak:** ketidaksamaan perilaku antarklien, potensi crash saat payload berubah, dan kebingungan pengguna karena kontrol tampak tersedia di satu platform tetapi tidak di platform lain.

**Perbaikan:** OpenAPI/JSON Schema sebagai sumber kontrak, generate DTO untuk Flutter dan TypeScript, event envelope berversi, dan daftar capability per platform yang diekspos ke UI.

### 5.11 P3 — Entry point terlalu besar

File runtime dan UI utama cukup besar sehingga perubahan lintas fitur berisiko menciptakan regresi dan menyulitkan review. Pecah berdasarkan domain, bukan hanya memindahkan fungsi ke file baru: setiap domain harus punya service, DTO, test, serta API publik yang jelas.

---

## 6. Perbandingan dengan kompetitor

**Penting:** produk dalam daftar ini tidak seluruhnya berada dalam kategori yang sama. Hermes, OpenClaw, dan Nod adalah pembanding personal/general-purpose agent; Codex, Cursor, Claude Code, dan Antigravity lebih berorientasi pada coding/IDE agent. Karena itu, perbandingan dilakukan per kapabilitas, bukan menganggap semua produk identik.

### 6.1 Ringkasan matriks

| Dimensi | Tiko-Agent | Pembanding yang kuat | Penilaian |
|---|---|---|---|
| General agent + tool loop | ReAct, router, tool registry, memory, budget | Hermes, OpenClaw | Tiko sudah punya fondasi yang sebanding; reliabilitas end-to-end perlu benchmark |
| Multi-channel | HTTP, CLI, Flutter, Telegram/Discord/Slack/WhatsApp/email dan gateway lain | Hermes, OpenClaw | Salah satu kekuatan Tiko |
| Persistent memory | SQLite/JSON, FTS, vector, RAG, layered/curated memory | Hermes, Nod, OpenClaw | Kuat; butuh memory provenance UI dan benchmark recall |
| Skill learning | Draft/scan/evaluate/approve, feedback, metric ledger | Hermes, Claude Code skills, OpenClaw skills | Kuat secara konsep; promosi skill dan evaluasi lintas task perlu lebih transparan |
| Multi-agent | DAG, Blackboard, bounded concurrency, subagent | Codex, Cursor, Claude Code, Antigravity, OpenClaw | Sudah ada, tetapi isolasi workspace/credential/role dan beberapa bug harus dibenahi |
| Coding workflow | Shell, file tools, patch, checkpoint, browser, ACP | Codex, Cursor, Claude Code | Perlu diff-first UX, repo context, test loop, workspace snapshots dan Git integration yang lebih padu |
| Browser/computer use | CDP browser, cloud browser backend, tool execution | Nod, OpenClaw, Antigravity, Cursor | Ada basis; belum menunjukkan pengalaman computer-use end-to-end sekuat Nod |
| Approval/policy | Permission gate, capability, sandbox, audit | Nod, Codex, Claude Code | Fondasi kuat, tetapi perlu policy per aksi dan UX approval yang lebih jelas |
| Automation | Scheduler, jobs, hooks, gateway delivery | Hermes, OpenClaw, n8n | Kuat untuk schedule/channel; perlu visual workflow builder dan run history yang lebih utuh |
| UI/UX | Flutter + web dashboard + CLI/TUI | Cursor, Antigravity, Nod, Codex app | Fitur ada, tetapi konsistensi dan onboarding perlu ditingkatkan |
| Observability | Run inspector, tool trace, audit, events | Codex tracing, Nod Usage | Ada, tetapi event contract, query, persistence, dan korelasi masih bisa lebih baik |
| Setup | `install.sh`, setup/config/doctor | Hermes, OpenClaw, Nod | Kemajuan baik; installer CLI belum menginstal/mengonfigurasi seluruh Flutter/mobile stack |

### 6.2 Hermes Agent

**Fokus:** open-source autonomous assistant lintas terminal, desktop, messaging platform, dan IDE. Hermes menyediakan toolsets, skills progressive disclosure, persistent memory, context files, checkpoint/rollback, cron, subagent, programmatic code execution, hooks, MCP, provider routing, fallback, credential pools, API server, ACP, dan plugin.

**Kelebihan Hermes:**

- alur onboarding dan ekosistem penggunaan sudah matang;
- skill dipanggil secara on-demand sehingga prompt tidak harus membawa seluruh dokumentasi;
- checkpoint dan rollback menjadi bagian yang terlihat dari workflow;
- cron, subagent, gateway, provider routing dan memory provider menjadi pengalaman produk yang saling menyatu;
- skins/themes dan personalisasi lintas surface membantu produk terasa konsisten.

**Kelemahan/konsekuensi:** banyak fungsi dan integrasi dapat membuat konfigurasi menjadi kompleks; kualitas aktual tetap bergantung pada model, toolset yang diaktifkan, dan environment pengguna.

**Posisi Tiko:** sangat dekat pada cakupan backend. Tiko memiliki keunggulan UI Flutter lintas platform dan implementasi memory/skill/security yang luas. Prioritas parity bukan menambah jumlah tool, melainkan onboarding, skills marketplace yang dapat dipercaya, checkpoint UI, profile/persona, tema, integrasi hooks, dan test end-to-end untuk channel.

### 6.3 OpenAI Codex

**Fokus:** coding harness untuk pekerjaan multi-langkah, lokal maupun cloud. Codex menggabungkan sesi durable, sandbox environment, streaming/webhooks, tool/MCP, steer/continue, subagent, context compaction/recovery, dan tracing. Ada perbedaan antara CLI, produk Codex, serta Agents API yang menyediakan managed harness.

**Kelebihan Codex:**

- lifecycle session dan turn jelas;
- environment/sandbox dipisahkan dari application server;
- trace memperlihatkan model generation, tool calls, subagent, durasi dan status;
- approval dapat mengevaluasi shell, file, network, dan MCP permission escalation;
- sesi dapat dilanjutkan dan diarahkan selama tugas berjalan.

**Kekurangan/konsekuensi:** managed harness/cloud membawa ketergantungan pada layanan dan batasan deployment/data residency; bukan pengganti penuh bagi gateway personal self-hosted.

**Posisi Tiko:** harus meniru kekuatan harness, bukan meniru nama produk. Prioritasnya session state tahan restart, event log yang bisa diputar ulang, execution environment per task, approval stream, trace per tool, checkpoint, serta recovery yang eksplisit.

### 6.4 Google Antigravity

**Fokus:** agent-first IDE/workbench. Dokumentasi resminya menjelaskan reasoning model, tools, artifacts, knowledge, browser, Agent Manager, beberapa agent conversation paralel, serta mode/settings dan MCP/rules/workflows.

**Kelebihan Antigravity:**

- agent menjalankan tugas multi-step sambil menampilkan artifacts;
- beberapa agent dapat bekerja paralel;
- browser dapat dipakai sebagai tool untuk menguji dan memeriksa aplikasi;
- knowledge/rules/workflows membantu perilaku agent mengikuti workspace.

**Posisi Tiko:** Tiko punya DAG, browser tool, artifact/run UI, graph view, dan Flutter UI. Yang masih perlu dikejar adalah Agent Manager visual untuk mengatur agent paralel, artifact review yang menyatu dengan diff, serta knowledge/context per workspace yang mudah dikelola.

### 6.5 Cursor

**Fokus:** coding agent di editor. Agent dapat mencari codebase, mengedit banyak file, menjalankan terminal, memakai browser, dan mendelegasikan pekerjaan melalui Projects/coordinator. Mode Ask/Plan/Debug/Agent memisahkan eksplorasi read-only, perencanaan, debugging, dan eksekusi. Checkpoint memungkinkan restore atas perubahan.

**Kelebihan Cursor:**

- pengalaman editor-first sangat jelas;
- mode kerja sesuai tahap tugas;
- diff dan checkpoint membuat perubahan dapat diawasi;
- semantic codebase search dan project rules meningkatkan konteks;
- browser dapat dipakai untuk mengambil screenshot dan memvalidasi UI.

**Posisi Tiko:** perlu Coding Workspace yang menampilkan file tree, diff, patch approval, terminal, test result, checkpoint, dan Git status dalam satu panel. Tambahkan mode `Ask`, `Plan`, `Build`, `Debug`, dan `Review`, dengan aturan bahwa mode read-only tidak dapat mengubah file.

### 6.6 Claude Code

**Fokus:** coding agent berbasis terminal yang dapat diperluas dengan project instructions, skills, hooks, subagents, MCP, permission controls, serta rewind/checkpoints.

**Kelebihan Claude Code:**

- hooks deterministik pada event lifecycle (misalnya sebelum/sesudah tool, session start/stop);
- subagent memiliki context window dan tool access sendiri;
- skill/CLAUDE.md/rules mengatur instruksi sesuai konteks;
- permission modes dan tool allow/deny dapat di-review;
- checkpoint/rewind mendukung eksplorasi yang aman.

**Posisi Tiko:** skill dan subagent sudah ada; perlu kontrak system instruction yang benar-benar terpisah dari user prompt, pre/post tool hook yang konsisten, isolation per subagent, serta policy profile yang mudah diaudit.

### 6.7 Nod dari n8n

**Identifikasi:** “Nod AI Agent” pada permintaan ini cocok dengan **Nod**, personal agent open-source dari n8n, bukan sekadar AI Agent node pada n8n workflow builder.

**Fokus:** personal agent yang memakai komputer/browser untuk menyelesaikan errands, bekerja dengan email, calendar, Slack, Notion dan web, menyimpan memory serta login secara lokal, dan meminta persetujuan sebelum melakukan tindakan yang tidak tercakup oleh izin/rules.

**Kelebihan Nod:**

- trust policy berada di server, bukan hanya di prompt;
- setiap aksi aplikasi/website yang mengubah keadaan diperiksa terhadap permintaan dan rules;
- tindakan di luar izin memerlukan approval;
- Vault memasukkan password, OTP, atau data pembayaran tanpa menyerahkan nilai rahasia ke model;
- web/app/browser data diperlakukan sebagai untrusted data;
- memory dapat diperiksa dan dihapus dengan jejak sumber;
- Usage menunjukkan tindakan, login, dan biaya.

**Kekurangan/konsekuensi:** Nod berfokus pada personal productivity/computer use, bukan IDE coding agent penuh; integrasi tertentu bergantung service koneksi; kemampuan host/background tetap mengikuti kondisi produk dan mesin pengguna.

**Posisi Tiko:** ini pembanding yang sangat penting untuk desain keamanan tindakan nyata. Tiko sudah mempunyai permission gate, capability, sanitizer, credential manager dan audit. Agar setara, Tiko perlu server-side action policy untuk setiap mutasi, secrets vault yang terpisah dari context model, provenance memory yang bisa dilihat/diubah user, audit penggunaan/biaya, dan UI persetujuan yang menjelaskan konsekuensi.

### 6.8 Kesimpulan kompetitor

Tiko tidak perlu mengalahkan Cursor dalam kualitas editor integration dan tidak perlu mengalahkan Nod pada personal computer errands dalam satu rilis yang sama. Sasaran yang lebih realistis:

1. **Parity general agent:** Hermes + OpenClaw — tools, memory, skills, gateway, schedules, plugins, subagents.
2. **Parity coding agent:** Codex + Cursor + Claude Code — plan, workspace, diff, test, checkpoint, approval, resume, trace.
3. **Parity computer-use trust:** Nod — action policy, Vault, memory provenance, user approval, usage ledger.
4. **Parity agent workbench:** Antigravity — Agent Manager, parallel agent, browser, artifact, knowledge workspace.

---

## 7. UI/UX Tiko saat ini dan gap pengalaman

### 7.1 Kekuatan UI yang sudah tersedia

- Flutter client sebagai basis Android, web dan desktop.
- Web dashboard di `apps/agentd/web`.
- Chat dengan streaming dan tool approval.
- Panel status, memory, approvals, sessions, skills, tasks, bots, providers, plugins, channels, jobs, security, audit, artifacts, dan runs.
- Run Inspector dengan ringkasan run/tool step.
- Command palette pada web.
- GenUI renderer dan `tiko_widgets` reusable package.
- Voice dictation/STT dan TTS dari perangkat/browser.
- Mobile navigation memindahkan sebagian admin/advanced tools ke halaman More, sedangkan desktop dapat memakai navigation rail.

### 7.2 Gap UI/UX yang terkonfirmasi dari dokumentasi proyek

| Area | Kondisi yang tercatat | Dampak |
|---|---|---|
| Bot management web | Create/delete/bind/unbind ada; edit sebagian properti belum tersedia di web | Pengguna harus pindah ke Flutter/CLI |
| Channel management web | Snapshot read-only; pengaturan memerlukan env + restart | Tidak terasa seperti admin console lengkap |
| Plugin/MCP | Consent dan snapshot tersedia; uninstall/MCP management belum seluruhnya tersedia sebagai endpoint/UI | Siklus integrasi tidak tuntas di UI |
| Security Center | Read-only; tidak ada toggle panic/kill switch/safe mode | Pengguna tidak bisa mengendalikan emergency action dari panel |
| Audit UI | Security card hanya menampilkan subset terbaru; query/filter ada di surface terpisah dan belum semua data persisten | Investigasi insiden belum sepraktis trace tool khusus |
| API contract | Flutter DTO typed; web masih banyak raw Map; WS belum punya schema version/event ID | Risiko regressions antarklien |
| Voice | STT/TTS tergantung dukungan browser/OS; belum full duplex, barge-in atau hotword | Pengalaman voice belum sama dengan voice assistant real-time |
| Setup | Installer `install.sh` menyiapkan CLI/daemon, bukan keseluruhan instalasi Flutter dan konfigurasi provider | Onboarding pengguna non-teknis masih terpecah |

### 7.3 Rancangan navigasi yang disarankan

Satu information architecture untuk seluruh platform:

1. **Chat** — chat biasa, attachments, context references, voice, pilihan model/persona.
2. **Tasks** — task yang sedang berjalan, queue, subagent, deadline, approval yang menunggu.
3. **Workspace** — project/file tree, search, diff, terminal, tests, checkpoint, Git status.
4. **Agents** — agent profile, role, instructions, provider/model, tool policy, workspace dan budget.
5. **Automations** — scheduler/cron, workflow visual, webhook, channel delivery, run history.
6. **Memory** — facts, source/provenance, knowledge graph, search, edit, forget, conflict resolution.
7. **Skills** — marketplace/local install, draft, scan, eval, approve, rollback, metrics.
8. **Connections** — providers, MCP, apps, channels, OAuth/secrets, status dan scopes.
9. **Security** — approval queue, sandbox, tool policy, vault, audit, emergency stop.
10. **Settings** — account/profile, appearance, language, network, storage, telemetry, update.

Mobile tidak harus memuat 10 tab; cukup 4–5 menu utama dan menu More. Yang wajib sama adalah konsep, nama, dan state, bukan layout pixel-perfect.

### 7.4 Pola interaksi yang harus menjadi standar

- Setiap task punya timeline: `queued → planning → running → needs approval → verifying → completed/failed/cancelled`.
- Tampilkan rencana ringkas yang dapat diedit sebelum task kompleks berjalan.
- Tampilkan aksi tool, target resource, perubahan yang diusulkan, dan hasil; jangan tampilkan internal chain-of-thought.
- Setiap write/mutating action mempunyai patch/diff preview dan tombol approve/reject.
- Checkpoint dibuat sebelum perubahan besar; rollback mudah ditemukan.
- Approval harus menyebutkan: tindakan, target, risiko, file/data yang disentuh, domain/network access, konsekuensi, dan apakah izin sekali atau berulang.
- Kesalahan harus actionable: penyebab, langkah memperbaiki, tombol retry, dan apakah retry aman/idempotent.
- UI harus menunjukkan perbedaan “agent selesai” dan “task berhasil diverifikasi”.

---

## 8. Cara membuat Tiko setara atau lebih baik

### 8.1 Roadmap P1 — stabilitas dan trust (segera)

Prioritas perbaikan wajib sebelum Tiko dipakai untuk otomasi berisiko atau deployment produksi yang mengandalkan eksekusi tool.

1. Perbaiki bug cancellation `TeamOrchestrator` dan tambahkan regression tests.
2. Pindahkan role `systemPrompt` ke system context sesungguhnya.
3. Implementasikan cancellation yang menghentikan process/network/browser operation; jangan mengandalkan `Future.timeout`.
4. Tambahkan idempotency key pada seluruh action tool yang menimbulkan side effect.
5. Perkuat JSON Schema validator dan fail-closed untuk schema yang tidak didukung.
6. Pastikan proses shell/MCP/script tidak pernah jatuh ke host execution saat sandbox tidak tersedia.
7. Dapatkan CI run yang dapat dibuktikan untuk commit release dan jalankan device QA Android/Termux.

**Definition of done:** tidak ada task yang hilang ketika cancel; tidak ada tindakan ganda setelah timeout/retry; tool role tidak bisa mengakses tool di luar policy; CI + platform smoke tercatat pada SHA yang dirilis.

### 8.2 Roadmap P2 — harness dan multi-agent isolation (30 hari)

- Bangun `AgentSession`/`AgentTurn` durable dengan status, event, checkpoint, resource, token/cost, cancellation, retry dan resume.
- Gunakan event envelope berversi: `event_id`, `schema_version`, `session_id`, `turn_id`, `request_id`, `timestamp`, `type`, `payload`.
- Simpan tool trace terstruktur, hasil, durasi, approval, error code, dan idempotency state.
- Isolasi subagent: workspace, memory namespace, credential scope, tool allowlist, network rule, model, budget dan session history.
- Ubah default multi-agent tool access menjadi least privilege; allowlist kosong harus berarti deny-all kecuali pewarisan tool host diminta secara eksplisit.
- Tambahkan policy server-side untuk setiap aksi yang mengubah resource.
- Bangun Secrets Vault: credential tidak masuk prompt/tool context; tool menerima handle terbatas atau secret injection langsung.
- Tambahkan replay/debug view untuk tool call, tanpa mengekspos secret.
- Persist audit dan security counters, dengan retensi dan export yang dapat diatur.

### 8.3 Roadmap P2 — coding agent/workbench (30–60 hari)

- Workspace per project dengan file tree, symbol/code search dan `AGENTS.md`/project rules.
- Plan mode read-only yang menghasilkan langkah dan daftar file sebelum perubahan.
- Diff-first editing dengan approve/apply/reject.
- Checkpoint otomatis, Git branch/worktree, rollback dan perbandingan snapshot.
- Tool test/lint/build yang hasilnya terhubung ke task timeline.
- Browser subagent untuk screenshot, DOM, console/network log, dan visual regression.
- Agent Manager untuk melihat beberapa agent, status, budget, hasil, dan intervensi.
- Tampilkan artifact: laporan, patch, screenshot, CSV, JSON, dokumentasi, dan hasil build.

### 8.4 Roadmap P3 — personal agent & automation (60–90 hari)

- App connectors yang memiliki scope per tindakan dan approval untuk mutasi.
- Workflow builder visual: trigger → condition → agent → tool → approval → delivery.
- Human-in-the-loop pada langkah sensitif dengan resume setelah keputusan.
- Memory UI yang menunjukkan sumber, provenance, confidence, perubahan, dan alasan lupa.
- Scheduler timezone/DST yang benar, retry policy, idempotency, delivery log, dan on/off control.
- Mobile device capability (share sheet, file picker, voice, notification action) dengan consent yang jelas.
- Voice mode bertahap: streaming, barge-in dan wake-word hanya bila dukungan OS aman dan reliabel.

### 8.5 Diferensiasi agar bisa lebih baik, bukan hanya setara

1. **Bahasa Indonesia sebagai first-class:** UI, error, onboarding, prompt injection evaluation, dokumentasi dan skill pack Bahasa Indonesia.
2. **Self-hosted privacy:** local-first memory, provider dapat ditukar, enkripsi data dan audit yang dapat dibaca pengguna.
3. **Universal client:** Flutter Android/desktop dan web dengan shared contract yang stabil.
4. **Agent profile yang portable:** definisi role, skills, policy, tools, model dan memory scope dapat diekspor/impor secara aman.
5. **Open evaluation:** benchmark task lokal dan publik; hasil sebelum/sesudah perubahan tersedia dalam dashboard.
6. **Cost-aware routing:** tentukan model sesuai kebutuhan; tampilkan token, biaya, waktu dan kualitas per task.
7. **Safe autonomy:** agent boleh bekerja mandiri pada tugas low-risk, tetapi approval wajib pada tindakan high-impact dan tidak jelas.
8. **Domain packages:** personal admin, coding, research, sales/CRM, content/SEO, data analysis dan local business workflow dapat dipasang sebagai profile/skill bundle.

---

## 9. Rencana benchmark dan Definition of Done

Untuk mengklaim Tiko setara/lebih baik, gunakan evaluasi yang sama untuk setiap versi. Jangan hanya menghitung jumlah tool atau package.

### 9.1 Benchmark kemampuan

| Area | Dataset / skenario | Metrik |
|---|---|---|
| Coding | 100–300 bug/feature task di repo terkontrol | task success, test pass, regression rate, diff acceptance |
| Tool use | 100 task multi-tool dengan dependency | success rate, wrong tool rate, retries, wall-clock |
| Browser | 50–100 task website uji | completion rate, DOM/action accuracy, recovery setelah layout berubah |
| Memory | 300–500 pertanyaan lintas sesi + memory conflict | Recall@5, Precision@5, stale fact rate, false memory rate |
| Prompt injection | Corpus benign/adversarial multilingual 500+ | attack success rate, false positive rate, recall per family |
| Safety | Aksi write/delete/send/pay/auth | unauthorized action count harus 0 pada benchmark policy |
| Multi-agent | DAG kecil/sedang, node failure, cancellation, duplicate result | completion, lost-node rate, duplicate execution, cost overhead |
| Reliability | disconnect, restart, provider fail, timeout, cancel | recovery rate, orphan process count, replay correctness |
| Performance | baseline Android midrange, laptop, server | p50/p95 latency, RAM, CPU, startup time |
| UX | pengguna baru menyelesaikan setup + task pertama | time-to-first-success, completion rate, help-needed rate |

### 9.2 Gate rilis

Rilis tidak cukup hanya `dart analyze` dan unit test. Syarat rilis sebaiknya:

- strict analyzer seluruh package;
- seluruh test unit/integration/widget/web;
- mock provider e2e;
- build Flutter web, Android, Linux, Windows, macOS;
- API + WebSocket compatibility test;
- MCP negative test dan sandbox unavailable test;
- cancellation/timeout/idempotency test;
- dependency vulnerability scan dan provenance artifact;
- smoke test pada minimal satu Android fisik/Termux dan satu desktop nyata;
- regression benchmark agent dengan hasil dibanding versi sebelumnya.

### 9.3 Metrik produk yang harus terlihat di dashboard

- tingkat keberhasilan task end-to-end;
- persentase task yang perlu intervensi user;
- rata-rata dan p95 waktu penyelesaian;
- tokens dan biaya per task/provider/model;
- tool error, timeout, cancellation, dan retry;
- approval accepted/rejected/expired;
- safety blocks dan false-positive;
- memory retrieval success dan memory correction rate;
- checkpoint/rollback count;
- skill promotion success dan regression rate.

---

## 10. Prioritas implementasi yang direkomendasikan

| Prioritas | Item | Alasan | Ukuran selesai |
|---|---|---|---|
| P1 | Perbaiki TeamOrchestrator cancellation | Task dapat hilang dari hasil | Semua task selalu direkam tepat satu kali |
| P1 | SystemPrompt multi-agent menjadi system context | Peran agent belum memiliki authority yang sesuai namanya | Test system/user separation lulus |
| P1 | Hard cancellation + idempotency | Timeout bisa menyisakan aksi berjalan | Tidak ada duplicate side effect pada timeout/retry |
| P1 | Strict tool schema validation | Schema constraint dapat lolos tanpa dicek | Negative schema suite lulus |
| P2 | Least-privilege allowlist | Mengurangi blast radius | Empty list berarti deny-all kecuali diwariskan eksplisit |
| P1 | Durable turn/event state | Recovery dan tracing lebih andal | Task dapat resume setelah daemon restart |
| P1 | Isolasi workspace/credential subagent | Hindari akses lintas role | Per-agent isolation tests lulus |
| P1 | Unified typed API/event contract | Hindari drift Flutter/web | DTO generated, event versioned |
| P1 | Security Center controls + persistent audit | Respons insiden | Approval/panic/audit lengkap dan dapat dicari |
| P2 | Coding workspace dengan diff/checkpoint | Parity Codex/Cursor/Claude Code | End-to-end patch-review-test-rollback |
| P2 | Agent Manager visual | Parity Antigravity dan subagent UI | Multi-agent state/outputs bisa diawasi |
| P2 | Workflow builder | Parity automation tools | Flow dapat dibuat, dijalankan, dihentikan, diulang |
| P3 | Memory provenance UI dan profile marketplace | Diferensiasi produk | Import/approval/rollback profile terukur |

---

## 11. Hal yang sudah baik dan sebaiknya tidak dirombak tanpa alasan

1. **Pembagian package** antara core, provider, tool, memory, security dan gateway. Ini fondasi yang sehat.
2. **AgentLoop per turn + session lock**; jangan menggantinya dengan singleton loop global.
3. **Permission gate dan capability layer** sebagai otoritas runtime.
4. **Sandbox fail-closed** ketika OS isolation tidak tersedia.
5. **Skill draft/evaluate/approve lifecycle**; jangan memberi privilege langsung pada skill hasil generasi model.
6. **Owner-scoped memory dan untrusted retrieved context**; jangan melemahkan batas ini untuk mengejar prompt recall.
7. **Provider health state machine tunggal**; hindari membuat counter health kedua yang terpisah.
8. **Flutter + web + CLI** sebagai beberapa surface yang memakai runtime bersama.
9. **Test dan CI yang luas**; tingkatkan evidence terbaru, bukan mengganti pipeline yang sudah mencakup banyak target.
10. **Kejelasan batasan yang belum selesai** dalam dokumen terbaru; pertahankan kejujuran ini sebagai budaya engineering.

---

## 12. Kesimpulan akhir

Tiko-Agent sudah memiliki cakupan arsitektur yang luas dan bukan proyek agent sederhana. Ada implementasi nyata untuk ReAct, routing, memory/RAG, skill management, multi-agent, gateway, MCP, browser, security gate, sandbox, scheduler, API, Flutter, web UI, dan voice.

Namun, status “mempunyai fitur” belum berarti “setara dengan produk terbaik”. Saat ini gap terpenting Tiko bukan kurangnya tool, melainkan **jaminan bahwa setiap alur berjalan benar, bisa dihentikan, tidak mengeksekusi aksi ganda, dapat dipulihkan, aman per role, terlihat jelas oleh pengguna, dan konsisten di semua platform**.

Temuan yang paling penting untuk segera diperbaiki adalah:

1. bug cancellation DAG yang dapat membuat task tersisa hilang dari hasil;
2. instruksi `AgentSpec.systemPrompt` yang masih masuk sebagai user content;
3. timeout tool yang tidak otomatis menghentikan aksi dasarnya;
4. validator JSON Schema yang tidak memeriksa sejumlah constraint;
5. allowlist multi-agent kosong yang mewarisi semua tool;
6. dokumentasi memory/index/status yang tidak mengikuti source terbaru.

Setelah enam hal tersebut dibenahi, langkah paling bernilai adalah membuat harness durable seperti Codex, coding workspace dengan diff/checkpoint seperti Cursor/Claude Code, agent manager dan artifacts seperti Antigravity, serta action policy + vault + provenance seperti Nod. Di sisi general agent, pertahankan kekuatan Hermes/OpenClaw dalam gateway, skills, memory, schedule, dan multi-agent.

**Prioritas yang disarankan: reliability → security → recovery/observability → UI consistency → workflow parity → feature expansion.** Jika urutan ini diikuti, Tiko punya peluang yang lebih kuat untuk menjadi produk yang kompetitif dan dapat dipercaya, bukan hanya repositori dengan daftar fitur panjang.

---

## 13. Sumber dan referensi

### 13.1 Tiko-Agent

- [Repository utama](https://github.com/kiosmarmer-commits/Tiko-Agent)
- [Commit yang diaudit](https://github.com/kiosmarmer-commits/Tiko-Agent/commit/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2)
- [Agent loop](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/packages/agent_core/lib/src/agent_loop.dart)
- [Multi-agent DAG](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/packages/agent_core/lib/src/agent_team.dart)
- [Multi-agent tool wiring](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/packages/tiko_runtime/lib/src/team_tool.dart)
- [Tool registry dan validator](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/packages/agent_tools/lib/src/registry.dart)
- [Sandbox](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/packages/agent_tools/lib/src/sandbox.dart)
- [Tiko runtime](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/packages/tiko_runtime/lib/src/tiko_runtime.dart)
- [Team tests](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/packages/agent_core/test/agent_team_test.dart)
- [CI workflow](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/.github/workflows/ci.yml)
- [Dokumen status dan indeks](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/docs/00-INDEX.md)
- [Voice UI](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/docs/VOICE-UI-GUI-2026-11-18.md)
- [Web parity GUI](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/docs/WEB-PARITY-GUI-2026-11-18.md)
- [Security Center GUI](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/docs/SECURITY-CENTER-GUI-2026-11-18.md)
- [API contract typed](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/docs/API-CONTRACT-TYPED-2026-11-18.md)
- [Optimisasi Hermes parity 13 November](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/docs/HERMES-PARITY-BATCH-2026-11-13.md)
- [Dokumentasi memory yang perlu diperbarui](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/42bd125ee7a4cb96278bd0a6eddb72afb6e224e2/docs/04-MEMORI-DB-PENCARIAN.md)

### 13.2 Dokumentasi resmi pembanding

- Hermes Agent — [Features overview](https://hermes-agent.nousresearch.com/docs/user-guide/features/overview), [Tools & toolsets](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools), [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture)
- OpenAI Codex/Agents — [Agents API overview](https://developers.openai.com/api/docs/guides/agents-api/overview), [Architecture](https://developers.openai.com/api/docs/guides/agents-api/architecture), [Run and continue sessions](https://developers.openai.com/api/docs/guides/agents-api/sessions), [Tracing](https://developers.openai.com/api/docs/guides/agents-api/tracing), [Automatic review](https://developers.openai.com/api/docs/guides/agents-api/sessions/approvals)
- Google Antigravity — [Agent capabilities](https://antigravity.google/docs/agent-features)
- Cursor — [Agent overview](https://cursor.com/docs/agent/overview), [Agent mode](https://cursor.com/help/ai-features/agent)
- Claude Code — [User FAQ](https://support.claude.com/en/articles/14554922-claude-code-user-faq), [Power user tips](https://support.claude.com/en/articles/14554000-claude-code-power-user-tips), [Subagents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- OpenClaw — [Features](https://docs.openclaw.ai/concepts/features), [Multi-agent routing](https://docs.openclaw.ai/multi-agent), [Automation](https://docs.openclaw.ai/automation), [Tools](https://docs.openclaw.ai/tools)
- Nod by n8n — [Product site](https://nod.n8n.io/), [GitHub repository/README](https://github.com/n8n-io/nod)
- Dart SDK — [Future.timeout semantics](https://api.dart.dev/dart-async/Future/timeout.html)

---

**Akhir laporan.**
