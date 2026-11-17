# Tiko-Agent — Audit Repository dan Rancangan GUI Terpadu Android, Desktop, dan Web

**Tanggal audit:** 18 November 2026  
**Repository:** <https://github.com/kiosmarmer-commits/Tiko-Agent>  
**Branch:** `main`  
**Commit acuan:** `dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae`  
**Tree SHA:** `cadadf63ecf83c8be5c489bcb4823b5cf31495b6`  
**Bahasa utama:** Dart  
**Status dokumen:** hasil inspeksi source dan dokumen pada snapshot commit di atas; rancangan implementasi, bukan klaim bahwa seluruh rancangan sudah dibuat di repository.

---

## 0. Ringkasan eksekutif

Tiko-Agent sudah mempunyai fondasi agentic yang jauh lebih luas daripada aplikasi chat biasa. Di dalam monorepo terdapat runtime, provider AI, tool registry, sandbox, MCP, browser automation, memory/RAG, skill lifecycle, scheduler, gateway pesan, plugin, API daemon, CLI/TUI, serta klien Flutter. Ada pula dua implementasi GUI yang berbeda: klien Flutter (`apps/agent_flutter`) dan web UI HTML/JavaScript yang disajikan `agentd` (`apps/agentd/web`).

**Masalah GUI terpenting bukan kekurangan modul backend, melainkan fragmentasi produk.** Flutter dan Web UI menggunakan runtime/API yang sama, tetapi navigasi, komponen, cakupan fitur, penyimpanan preferensi, dan cara merender hasil agent tidak sama. Flutter memiliki dashboard runtime dan renderer GenUI; web UI memiliki layar Skills, Tasks/Kanban, event feed, dan audit log. Tidak satu pun dari keduanya saat ini menyajikan seluruh kemampuan Tiko dalam satu control center seperti Hermes Desktop/Web Dashboard.

### Rekomendasi utama

1. **Jadikan `apps/agent_flutter` sebagai aplikasi utama lintas platform** untuk Android, Windows, macOS, Linux, dan browser. Gunakan satu desain responsif dan satu model navigasi; platform hanya mengubah tata letak dan integrasi native.
2. **Jadikan `apps/agentd/web` sebagai UI fallback/transisi atau halaman diagnostik minimal**, bukan aplikasi produk kedua yang terus berkembang dengan fitur berbeda. Pindahkan fitur web yang masih bernilai (Skills feedback, Tasks, Live Events, Audit Logs) ke Flutter UI menggunakan API yang sudah ada.
3. **Pertahankan `agentd` sebagai daemon/control plane**. Flutter tetap bertindak sebagai client. Desktop dapat tetap memakai bundled daemon bootstrap; Android secara default menjadi remote-control client ke daemon lokal/server yang terjangkau. Jangan mengasumsikan Android memiliki kemampuan daemon mandiri sebelum lifecycle, izin, background mode dan pengujian perangkat dituntaskan.
4. Bangun UI berdasarkan fitur nyata Tiko: **Chat Workspace, Sessions, Agents/Bots, Tasks & Automations, Skills, Memory, Providers & Models, Tools & MCP, Channels, Security & Approvals, Logs & Diagnostics, Settings**.
5. **Tutup celah GenUI lintas klien.** Saat ini renderer GenUI ada di Flutter, tetapi belum ada renderer web setara; action/form dikirim sebagai teks `/ui ...` dan belum memiliki handler khusus agentd. Definisikan event/action contract terstruktur di API, lalu render dan validasi pada Android/Desktop/Web.
6. Pisahkan pengaturan menjadi **preferensi perangkat**, **konfigurasi profil/agent di daemon**, dan **credential/secret**. Jangan menyimpan API key atau bearer token permanen di `localStorage` untuk browser yang tidak dipercaya.

### Penilaian ringkas

| Area | Kondisi pada snapshot | Penilaian |
|---|---|---|
| Runtime, tools, provider, memory, scheduler | Banyak modul tersedia; beberapa butuh config/provider/software eksternal | Fondasi kuat; harus dipetakan ke UI capability |
| Flutter GUI | Chat, dashboard, approval, sessions, memory, settings, GenUI | Klien utama yang tepat untuk dikembangkan, tetapi menu manajemen belum lengkap |
| Web GUI `agentd/web` | Chat, memory, status, approvals, sessions, skills feedback, tasks, event feed, audit logs | Lebih banyak layar administrasi, tetapi implementasi berbeda dan sederhana |
| Android | Flutter UI responsif; API client | Cocok sebagai klien mobile; release/signing dan device QA belum dibuktikan dari CI |
| Desktop | Flutter app; CI membangun Linux/Windows/macOS dan membundel daemon | Jalur desktop self-contained sudah menjadi fondasi yang bagus |
| Flutter Web | Build web ada di CI | Belum sama dengan Web UI `agentd/web`; integrasi produk perlu diputuskan |
| GenUI | DSL dan renderer Flutter terimplementasi | Belum parity lintas platform dan action belum terikat ke typed handler |
| Hermes parity di GUI | Banyak kemampuan backend, belum seluruhnya punya halaman kontrol GUI | Gap terbesar adalah product surface, bukan menambah fitur backend secara membabi buta |

---

## 1. Ruang lingkup, metode, dan batas audit

### 1.1 Ruang lingkup

Audit ini mencakup:

- Metadata repository, branch, commit, tree dan inventaris file.
- README, dokumen arsitektur dan indeks dokumentasi.
- Sumber aplikasi Flutter: shell utama, API client, chat controller, settings, theme, GenUI dan UI pendukung.
- Sumber web: `index.html`, `app.js`, `app.css`, UI README, test harness.
- Entry point daemon `agentd` dan dokumen backend yang relevan untuk fitur UI.
- Dokumen status Hermes-parity dan closure D/E terbaru pada snapshot audit.
- Dokumentasi resmi Hermes Desktop dan Hermes Web Dashboard sebagai referensi pola UX, bukan untuk menyalin identitas visual atau aset.

### 1.2 Metode dan keterbatasan download

Repository diakses melalui GitHub API/connector pada branch `main`, termasuk recursive Git tree dan pembacaan source file utama. Tree berisi **850 file blob** dengan total ukuran file sekitar **4,99 MB**. Ukuran repository pada metadata GitHub tercatat sekitar **2,5 MB** (ukuran repository yang dilaporkan GitHub tidak identik dengan penjumlahan seluruh file working tree).

Upaya `git clone` ke filesystem kerja tidak berhasil karena environment tidak dapat menyelesaikan DNS `github.com`. Oleh sebab itu:

- Source repository **berhasil diperiksa secara langsung melalui GitHub**, tetapi ZIP repository tidak berhasil dibuat di filesystem sesi ini.
- Laporan ini tidak mengklaim sudah menjalankan `dart analyze`, `dart test`, `flutter test`, build, atau uji perangkat pada snapshot terbaru.
- Hasil test yang disebut di bawah berasal dari dokumen proyek pada commit tersebut dan harus diperlakukan sebagai catatan verifikasi yang dibuat oleh proyek, bukan hasil eksekusi ulang dalam audit ini.
- Jalur download ZIP langsung dari GitHub: <https://github.com/kiosmarmer-commits/Tiko-Agent/archive/refs/heads/main.zip>.
- Permalink commit yang diaudit: <https://github.com/kiosmarmer-commits/Tiko-Agent/commit/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae>.

### 1.3 Kategori status

Gunakan label berikut agar rencana tidak mencampur kode yang sudah aktif dengan ide:

- **Implemented** — source dan wiring aktif dapat ditemukan.
- **Implemented — conditional** — kode ada, tetapi tergantung secret, config, software host, binary, provider, atau akun pihak ketiga.
- **Partial** — sebagian alur sudah ada; belum lengkap untuk pengalaman produk end-to-end.
- **UI gap** — backend atau command ada, tetapi belum ada pengelolaan yang layak melalui GUI.
- **Blocked / external dependency** — membutuhkan infrastruktur, kredensial nyata, native binary, atau perangkat fisik.
- **Proposed** — rancangan dalam dokumen ini; belum diklaim sebagai implementasi.

---

## 2. Repository yang diaudit

### 2.1 Snapshot

| Item | Nilai |
|---|---|
| Repository | `kiosmarmer-commits/Tiko-Agent` |
| Branch | `main` |
| Commit | `dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae` |
| Tanggal commit | 17 November 2026 |
| Commit message | `feat(install): installer sekali-perintah install.sh (Dart SDK otomatis + compile + pasang prefix) (E5)` |
| Git tree | `cadadf63ecf83c8be5c489bcb4823b5cf31495b6` |
| File blobs pada recursive tree | 850 |
| Jumlah byte file pada tree | 4.990.781 byte |
| Bahasa utama | Dart |
| License pada metadata | GPL-3.0 |
| Visibility | Public |

### 2.2 Peta monorepo

```text
Tiko-Agent/
├── apps/
│   ├── agentd/                 # daemon, HTTP/WebSocket API, static Web UI
│   ├── agent_flutter/          # Flutter client Android/Desktop/Web
│   ├── agent_cli/              # CLI agent
│   ├── tiko/                   # CLI utama Tiko
│   ├── simpul/                 # aplikasi/alat Simpul
│   └── tiko_widgets/           # reusable Flutter UI widgets
├── packages/
│   ├── agent_core/             # AgentLoop, router, context, skills, team, GenUI
│   ├── agent_providers/        # provider model dan routing provider
│   ├── agent_tools/            # built-in tools, sandbox, MCP, browser, API, ACP
│   ├── agent_memory/           # SQLite/JSON, FTS, vector, RAG, memory layers
│   ├── agent_gateway/          # HTTP/WS, Telegram, Discord, Email, webhooks, voice
│   ├── agent_scheduler/        # cron/job store/scheduler
│   ├── agent_security/         # permission, prompt guard, redaction, audit/security
│   ├── tiko_runtime/           # composition root, config, daemon/runtime services
│   ├── agent_calc/             # calculator/parser/evaluator/sandbox
│   ├── simpul_core/            # graph, notes, vault, indexing, search
│   ├── prediksi/               # prediksi/analisis statistik/pasar
│   └── tiko_tui/               # terminal UI
├── config/
├── data/
├── docs/
├── skills/
├── scripts/
└── .github/workflows/          # CI/build automation
```

### 2.3 Arsitektur produk saat ini

```text
Android Flutter ─┐
Desktop Flutter ─┼── HTTP + WebSocket ──> agentd ──> TikoRuntime
Flutter Web ─────┘                         │           ├── AgentLoop / Router
                                           │           ├── Providers / Models
Web HTML/JS ───────────────────────────────┘           ├── Tools / MCP / Browser
                                                       ├── Memory / RAG / Skills
CLI / TUI ────────────────────────────────────────────┤
Telegram / Discord / Slack / WhatsApp / Email / etc ──┤
                                                       ├── Approval / Security
                                                       ├── Scheduler / Jobs
                                                       └── Audit / Events / Storage
```

**Interpretasi arsitektur:** runtime Tiko sudah dipisahkan dari UI. Ini tepat untuk produk multi-platform. Flutter dan web seharusnya menjadi beberapa presentasi dari satu backend yang sama, bukan masing-masing membangun ulang semantik agent.

---

## 3. Temuan source dan kemampuan Tiko yang relevan untuk GUI

### 3.1 Agent runtime dan orkestrasi

Ditemukan modul `agent_core` dan `tiko_runtime` yang menyediakan agent loop, routing provider, budgeting, context building/expansion, plan-execute, cancellation, parallel tool runner, subagent, team orchestration, feedback, trajectory, self-learning dan reflection.

Implikasi untuk GUI:

- Chat tidak cukup hanya menampilkan bubble user dan jawaban akhir.
- Pengguna perlu melihat **status run**, langkah rencana, tool yang sedang bekerja, subagent yang berjalan, waktu, hasil dan error.
- Kontrol run minimal: Stop/Cancel, Retry eksplisit, Copy, Continue, dan Open Details.
- Retry tidak boleh otomatis mengirim prompt ulang setelah WebSocket terputus. Source `ChatController` secara eksplisit menghindari retry otomatis agar tool side effects tidak terduplikasi.
- Multi-agent perlu halaman roster dan visualisasi task graph, tetapi status setiap agent harus berasal dari runtime event/data nyata.

### 3.2 Provider AI dan routing model

README mengidentifikasi adapter OpenAI, Anthropic, Gemini, dan Ollama-compatible; routing tier, provider health/circuit breaker dan capability probing juga tersedia pada modul yang relevan.

UI yang dibutuhkan:

- Daftar provider dengan status koneksi, metode auth, model terdeteksi, default, prioritas dan health.
- Pemilihan model utama per profile/bot serta pilihan model untuk task/auxiliary bila backend mendukungnya.
- Tombol Test Connection dan refresh model catalog.
- Indikator fallback saat router berpindah provider, bukan hanya label provider generik.
- Pengelolaan credential harus berada di backend/credential store; jangan memasukkan API key ke preference umum Flutter atau browser `localStorage`.

Status penting: dukungan provider di library tidak membuktikan kredensial provider sudah dipasang ataupun koneksi live sudah diverifikasi.

### 3.3 Tools, MCP, browser dan sandbox

`agent_tools` mencakup registri tools, built-in tools, code/shell execution, script runner, path safety, SSRF protection, MCP stdio/HTTP/OAuth, browser CDP dan cloud browser backends, image generation, search, files, patching, calculator, subagent, plugin tools dan utility lain.

UI yang dibutuhkan:

- Tool Catalog dengan kategori: Files, Web, Browser, Code, System, Data, Media, MCP, Custom.
- Detail setiap tool: deskripsi, input schema, risk class, permission/capability, status enablement, asal (built-in/MCP/plugin), kebutuhan dependency, dan kapan meminta approval.
- MCP Servers: tambah/edit/hapus, transport (stdio/HTTP), auth, daftar tools, health, start/stop/reconnect, log dan status consent.
- Sandbox: backend aktif, status capability, batas workspace, mode safe, resource limits, warning ketika OS isolation tidak tersedia.
- Browser: local vs cloud backend, browser executable/session, remote connection status dan cleanup.
- Image generation/media: model/provider, kredensial tersambung, file output, progres queue, dan biaya/kuota bila API benar-benar memberikan datanya.

**Peringatan:** dokumen D-series menyatakan Wasm sandbox belum tersedia pada dependency Dart matang; Android/Termux sandbox QA membutuhkan perangkat nyata. Isolasi plugin melalui proses terpisah dan consent bukan bukti bahwa OS sandbox sudah kuat.

### 3.4 Memory, search dan RAG

`agent_memory` menyediakan SQLite/JSON fallback, FTS5/BM25, vector index, hybrid retrieval/RAG, memory layers dan provider adapter. Dokumen proyek mencatat curated memory `MEMORY.md` / `USER.md`, memory nudges dan konsolidasi.

UI yang dibutuhkan:

- Memory Explorer: cari, filter, lihat sumber, jenis, owner/session, relevansi dan waktu.
- Curated Memory Editor untuk `MEMORY.md` dan `USER.md`, dengan batas penggunaan karakter, perubahan yang diusulkan dan riwayat.
- RAG Sources: dokumen/file yang sudah di-ingest, status parsing, chunking, embedding status, indexing, error, refresh/delete.
- Layered Memory: Episodic, Semantic, Procedural; tampilkan sebagai kategori dan bukan klaim bahwa semua backing store memiliki UX grafis sudah siap.
- HDC Memory perlu dijelaskan sebagai memori asosiatif trigram deterministik dengan FIFO maksimum 1.000 dalam dokumen D2, bukan pengganti semantic/vector memory dan bukan kapasitas tidak terbatas.

Flutter saat ini hanya menampilkan list fakta memori dan tombol delete; web UI juga menampilkan memori dasar. Belum ada UI yang layak untuk curated memory, RAG source management, provenance atau layer explorer.

### 3.5 Skills dan learning loop

Kode/dokumen proyek menyebut skill parse/validate/scan, draft/trial/approved/retired, approval gate, feedback 👍/👎, skill hub client, agentskills.io manifest support, local install, share/export-import dan `/learn`.

UI yang dibutuhkan:

- Skills Library dengan search, kategori, status, enabled/disabled, source/version.
- Skill Detail: SKILL.md, tool allowlist, triggers, permission, test/evaluation, metrics, feedback, history.
- Workflow lifecycle yang jelas: Draft → Validate/Scan → Evaluate → Approve → Active → Retire/Rollback.
- Install Local, Import Package, Browse Hub bila hub server dikonfigurasi.
- Review diff sebelum skill dipromosikan; feedback tidak boleh otomatis mempromosikan draft.
- Learning dashboard: feedback signal, lesson, skill generated, trajectory. Pisahkan metric yang dihitung dari insight model.

Web UI saat ini sudah menyediakan daftar skills dan feedback thumbs. Flutter UI yang diperiksa belum memiliki tab Skills. Skills Hub client ada, tetapi registry server publik dicatat blocked di dokumen E-series karena butuh infrastruktur.

### 3.6 Bots, profiles dan multi-agent teams

`docs/BOT-MODE-2026-11-15.md` menjelaskan BotStore persist per bot, bot persona/model/greeting, binding sesi, tool allowlist fail-closed, command `/bot new/persona/model/allow/import/use/off/show/delete`, serta import SillyTavern character cards JSON/PNG. Routing grup gateway menggunakan BotGroupRouter pada shared store.

UI yang dibutuhkan:

- Roster bot/agent dengan avatar, nama, role, provider/model, status, recent activity.
- Create/Edit Bot Wizard: nama, deskripsi, persona, model default, greeting, tool allowlist, skills, MCP, profile, safe mode.
- Preview prompt/persona dan dry run sebelum aktivasi.
- Activate/Deactivate per session; assign session to bot.
- Team view dengan role delegasi, task graph, progress dan hasil per subagent.

Status sekarang: fitur Bot Mode ada pada backend/command; audit source Flutter tidak menemukan GUI manajemen bot. Ini adalah UI gap yang cukup besar karena command `/bot` tidak mudah ditemukan pengguna baru.

### 3.7 Scheduler, tasks dan automation

`agent_scheduler` menyediakan scheduler/cron dan job store. Agent manager web UI telah mempunyai endpoint dan tab Tasks dengan kolom backlog/doing/done/failed, event feed dan audit log.

UI yang dibutuhkan:

- Automation list dengan nama, jadwal, timezone, prompt ringkas, delivery channel, enabled/paused/error, last run dan next run.
- Create/Edit schedule wizard, human-readable preview cron, timezone picker, delivery destination, enable/disable, Run now, history dan safe delete confirmation.
- Task board dengan detail, owner/agent, status, progress, artifact, timestamps, retry/cancel.
- Calendar view opsional untuk cron jobs.

Web UI punya Task Kanban dan audit logs; Flutter dashboard hanya menampilkan event terbaru dan approvals. **Jangan menyamakan task board backend dengan scheduler UI**: job CRUD/run/pause/resume harus dipastikan ada endpoint/handler sebelum kontrolnya ditampilkan.

### 3.8 Channels dan gateway

`agent_gateway` serta `agentd` memuat Telegram, Discord, Email/IMAP, Slack webhook, WhatsApp webhook, Signal, Matrix, HTTP/WebSocket, voice/STT/TTS.

UI yang dibutuhkan:

- Channel cards: Telegram, Discord, Slack, WhatsApp, Email, Signal, Matrix, Webhook.
- Connection wizard, secret entry via backend credential store, allowlist, webhook callback info, health, last message, reconnect/restart status.
- Hanya menampilkan kemampuan yang tersedia dalam konfigurasi aktual; jangan menampilkan semua channel sebagai Connected hanya karena class/library ada.
- Status setup channel menjelaskan izin platform dan credential yang kurang.

Status: banyak gateway bersifat conditional pada env/secret/akun. SMTP live relay dan gateway live smoke tests dicatat membutuhkan kredensial nyata. Discord voice belum terbukti production-ready di semua jalur: closure parity mencatat `UnsupportedVoiceEncryptor` untuk skema `xsalsa20_poly1305` dan butuh implementasi FFI sodium untuk transmisi nyata. TTS tersedia sebagai library/provider, tetapi jalur otomatis setiap balasan menjadi audio belum lengkap menurut happy-path manual.

### 3.9 Security, approvals dan audit

Sistem keamanan mencakup permission gate, PromptGuard/prompt injection defense, redaction, approval broker/ledger, audit log, plugin consent, rate anomaly detection dan token anomaly detection (opt-in).

UI yang dibutuhkan:

- Approval Inbox global lintas channel: tool, alasan, arguments yang sudah disanitasi, risk, sesi, requestor, waktu kedaluwarsa, approve/deny.
- Detail risk dan efek samping sebelum izinkan.
- Audit trail immutable/read-only pada UI, filter by date/session/tool/risk/actor.
- Security Center: safe mode, sandbox, plugin consents, secret state, token anomaly, prompt guard events, active sessions.
- Kill switch / stop all runs dengan confirmation kuat bila runtime API mendukungnya.

Flutter saat ini memiliki approval inbox dan approve/deny. Web UI juga memiliki approvals. Web task view membaca audit logs. Kedua GUI belum menyatukan seluruh security policy ke satu halaman kontrol.

---

## 4. Audit kondisi GUI saat ini

### 4.1 Flutter (`apps/agent_flutter`)

File utama:

- `lib/main.dart` — app shell, routing tab, chat, dashboard, approvals, sessions, memory, mobile More.
- `lib/agent_api_client.dart` — HTTP + WebSocket client ke daemon.
- `lib/ui/chat_controller.dart` — chat state dan stream events.
- `lib/ui/settings_view.dart` — Appearance, Connection, Behavior, Daemon.
- `lib/ui/settings_store.dart` — preference lokal via `shared_preferences`.
- `lib/ui/theme.dart` — SimpulTheme / palette / spacing.
- `lib/ui/gen_ui_view.dart` — renderer GenUI.
- `lib/ui/graph_view.dart`, `note_editor.dart`, `index_dashboard.dart`, `command_palette.dart` — fitur UI pendukung lain dalam source tree; integrasi dan penempatan sebagai workspace utama perlu dipetakan sebelum memindahkan semuanya ke navigasi utama.

**Desktop / layar lebar (breakpoint saat ini 900 px):** `NavigationRail` dengan Chat, Dashboard, Approvals, Sessions, Memory, Settings.

**Android / layar sempit:** `NavigationBar` bawah dengan Chat, Home, Approve, Sessions, More; More membawa Memory dan Settings.

Kelebihan:

- Satu basis Flutter untuk Android, desktop, dan build web.
- Responsive shell awal sudah ada.
- Chat streaming lewat WebSocket.
- Pending confirmation/approval terlihat pada chat.
- Dashboard menunjukkan provider, memory backend, resource profile, safe mode, sandbox, routing tier, MCP, doctor dan event.
- Settings punya tema system/dark/light, accent, API URL/token, Ollama host, vault path, auto-lock dan power saver.
- Desktop bootstrap bisa spawn/reuse bundled daemon melalui `DesktopDaemon.ensure`.
- Error HTTP admin tidak disamarkan sebagai data kosong; dokumen pengujian mengunci 401/403/500.
- WebSocket memakai one-time ticket agar bearer token utama tidak dikirim pada URL query.

Kesenjangan dan perbaikan:

1. **Sessions tidak memuat histori transcript.** `SessionsPage` memilih ID sesi, lalu `chat.clearChat()`; tidak ada paging/replay riwayat pesan yang terlihat di UI client ini.
2. Tidak ada UI create/rename/archive/delete/resume session pada Flutter yang diperiksa.
3. Tidak ada Bot Manager/Profiles GUI walaupun BotStore dan `/bot` command tersedia.
4. Tidak ada halaman Tools/MCP/Plugins, provider/account/model manager, gateway/channel setup, scheduler UI, skill lifecycle manager, automation CRUD, audit log browser dan task board pada shell Flutter saat ini.
5. Dashboard berisi kartu tekstual; belum menjadi command center dengan detail tool trace, artifact panel, task progress dan analytics.
6. UI tidak memiliki panel run-inspector yang konsisten untuk plan/steps/tool/subagents.
7. Chat messages tampak disimpan pada list runtime controller di memori; bukan berarti transcript historis sudah dimuat dari server.
8. UI Settings belum memisahkan global profile settings, workspace settings, daemon settings dan device settings.
9. Token disuntik melalui `dart-define` atau input manual. Perlu secure credential store dan masking; pastikan mekanisme untuk web/Android tidak menyimpan token mentah secara tidak aman.
10. Aplikasi Android pada CI hanya terbukti menghasilkan **debug APK**. Ini bukan bukti release signing / Play Store package siap.

### 4.2 Web UI (`apps/agentd/web`)

File utama:

- `index.html` — tab dan markup.
- `app.js` — API helper, chat stream, polling, approval, skills, tasks, logs, WebSocket events.
- `app.css` — palette dan layout responsive.
- `app.test.cjs` — Node test harness.

Navigasi yang ditemukan:

- Chat
- Memori
- Status
- Approvals
- Sessions
- Skills
- Tugas (Tasks, event feed, audit log viewer)
- Bantuan (API URL dan bearer token)

Kelebihan:

- Memiliki beberapa layar admin yang belum muncul di Flutter, khususnya Skills, task Kanban, event feed dan audit logs.
- Chat memakai WebSocket stream dan confirmation box.
- WebSocket event observer melakukan reconnect dengan exponential backoff untuk feed event.
- UI memakai DOM node + `textContent`, bukan interpolasi API data langsung sebagai HTML mentah.
- Memiliki loading, empty dan error states.
- Ada test Node untuk perilaku frontend.

Kesenjangan:

1. Web UI merupakan proyek HTML/CSS/JS terpisah dari Flutter; visual, state, navigasi dan komponen tidak konsisten.
2. Halaman Status sebagian besar menampilkan JSON mentah, bukan observability dashboard yang bisa dibaca pengguna umum.
3. Pengaturan web untuk koneksi disimpan di `localStorage`; jangan memakai pola ini untuk menyimpan credential tingkat provider/API key.
4. Web UI belum merender GenUI blocks menjadi widget interaktif.
5. Fitur CRUD bot, provider, model, tools, MCP, gateways, scheduler dan plugin belum menjadi management console yang lengkap.
6. Tasks UI pada web merupakan board + feed + logs; bukan seluruh automation lifecycle manager.
7. Uji `scripts/web_ui_validate.sh` hanya melakukan `node --check` dan Node tests, bukan visual regression di Chrome/Firefox/Safari/Android browser.

### 4.3 Perbandingan Flutter dan Web

| Fitur | Flutter Desktop | Flutter Android | Web `agentd/web` | Status kesetaraan |
|---|---:|---:|---:|---|
| Chat streaming | Ya | Ya | Ya | Secara dasar ada di semua; UI berbeda |
| GenUI render | Ya | Ya | Belum | Tidak setara |
| Dashboard runtime | Ya | Versi mobile | Status JSON | Tidak setara |
| Approvals | Ya | Ya | Ya | Fungsi dasar setara |
| Sessions list/select | Ya | Ya | Ya | Transcript management belum lengkap |
| Memory facts | Ya | More page | Ya | Fungsi dasar ada; tidak ada curated editor |
| Skills listing/feedback | Belum pada nav Flutter | Belum | Ya | Tidak setara |
| Task Kanban | Belum | Belum | Ya | Tidak setara |
| Live event feed | Dashboard recent events/poll | Dashboard | Ya, WS observer | Tidak setara |
| Audit logs | Belum | Belum | Ya | Tidak setara |
| Provider/model management | Settings parsial | Settings parsial | Tidak lengkap | Tidak setara |
| Bot manager | Belum | Belum | Belum | UI gap global |
| Gateway setup | Belum | Belum | Belum | UI gap global |
| Tool/MCP/plugin management | Belum | Belum | Belum | UI gap global |
| Cron/automation management | Belum | Belum | Belum | UI gap global |
| Settings | Theme/connection/behavior | Sama, lebih ringkas via More | Connection only + browser storage | Tidak setara |

**Kesimpulan audit UI:** Flutter paling cocok menjadi basis tunggal karena sudah menargetkan Android/Desktop/Web, sedangkan web HTML/JS menyediakan daftar fungsi admin yang perlu dipertahankan dan dipindah. Hindari dua UI yang berkembang dengan daftar fitur masing-masing.

---

## 5. Hermes sebagai referensi UX: apa yang ditiru secara konseptual

Referensi yang digunakan adalah dokumentasi resmi Hermes Desktop dan Web Dashboard yang diakses pada 18 November 2026:

- <https://hermes-agent.nousresearch.com/docs/user-guide/desktop>
- <https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard>
- <https://hermes-agent.nousresearch.com/docs/user-guide/configuration>

Hermes Desktop menggambarkan aplikasi chat-first dengan sidebar, beberapa percakapan simultan, provider/model settings, toolset, MCP, messaging, skills, memory graph, cron, profile dan command palette. Pengguna dapat mengatur pane, membuka inspector/terminal/files, dan berpindah antara Simple dan Advanced layout. Hermes Web Dashboard berfokus pada pengelolaan instalasi melalui browser: profile, skills, MCP, API keys, sessions, logs dan cron jobs. Dashboard default dijalankan pada loopback; bind publik memerlukan auth.

### 5.1 Pola Hermes yang sebaiknya diadopsi

| Pola UX | Manfaat bagi pengguna Tiko | Bentuk implementasi Tiko |
|---|---|---|
| Chat-first workspace | Pengguna baru langsung mulai tanpa mencari menu | Chat sebagai default Home; onboarding menghubungkan provider |
| Persistent sessions sidebar | Mudah membuka kembali percakapan | Session list dengan search, pin, rename, archive, unread/running |
| Profile / bot selector | Memisahkan agent dengan persona, model, skills | Bot/Agent roster di sidebar dan Bot Manager |
| Live tool activity | Pengguna paham apa yang sedang dikerjakan agent | Activity timeline + expandable tool call card |
| Right inspector / panes | Debug dan review tanpa meninggalkan percakapan | Run inspector, context, files/artifacts, diff, event details |
| Provider + model picker | Ganti model dengan kontrol yang jelas | Model selector + provider manager + test connection |
| Skills & toolsets manager | Kemampuan agent bisa diaudit dan disetel | Skills lifecycle page + Tool Catalog |
| MCP management | Setup integrasi tanpa CLI | MCP wizard + health/log/consent |
| Cron/job manager | Mengelola automation secara visual | Automation center, schedule editor, run history |
| Command palette | Navigasi dan aksi cepat | Ctrl/Cmd+K desktop; mobile search/action sheet |
| Simple / Advanced mode | UX sederhana bagi pemula tanpa membatasi ahli | Mode Beginner dan Advanced, same feature/API |
| Shared state across surfaces | Sesi dan memori konsisten | Semua klien akses daemon/profile store yang sama |

### 5.2 Hal yang tidak boleh ditiru mentah-mentah

- Jangan menyalin logo, warna merek, ilustrasi, wording produk atau aset Hermes.
- Jangan menampilkan fitur seakan sudah tersedia bila backend Tiko belum menyediakan route/wiring/permission.
- Jangan membuat settings web menulis config file langsung dari browser; semua perubahan konfigurasi harus melewati daemon dengan validasi, audit, dan approval bila diperlukan.
- Jangan memaksa tampilan desktop tiga kolom pada telepon. Gunakan struktur informasi yang sama, tetapi layout sesuai ruang.
- Jangan menampilkan tool trace/agent state spekulatif. Semua status harus berasal dari event/response runtime.

---

## 6. Rancangan produk: Tiko Workspace

### 6.1 Prinsip desain

1. **One agent, one runtime, multiple clients.** Semua platform terhubung pada runtime, profile, memory, skills, jobs dan approvals yang sama.
2. **Chat-first, admin-ready.** Halaman awal tetap percakapan; fitur teknis berada di menu yang terstruktur.
3. **Progressive disclosure.** Pengguna baru melihat tombol dan setting penting dahulu; detail provider, tool schema, risk, raw log dan debugging ada dalam mode Advanced.
4. **Capability-driven UI.** Menu diaktifkan berdasarkan capability/status runtime dan permission pengguna, bukan hanya karena kode class ditemukan.
5. **No silent failure.** Error koneksi, provider, tool, auth dan approval tampil dengan status yang dapat ditindaklanjuti.
6. **Safe by default.** Semua aksi mutasi sensitif memerlukan confirmation yang sebanding dengan risk.
7. **Responsive by layout, consistent by semantics.** Menu, istilah, state, warna status dan interaksi inti sama; komposisi panel menyesuaikan perangkat.

### 6.2 Information Architecture — menu utama

Susunan yang direkomendasikan:

```text
TIKO WORKSPACE
│
├── Workspace
│   ├── Chat
│   ├── Sessions
│   ├── Projects / Workspaces
│   └── Artifacts
│
├── Agents
│   ├── Agent & Bot Roster
│   ├── Create Agent
│   ├── Teams / Delegation
│   └── Agent Activity
│
├── Automations
│   ├── Tasks Board
│   ├── Scheduled Jobs (Cron)
│   ├── Run History
│   └── Delivery Channels
│
├── Knowledge
│   ├── Memory Explorer
│   ├── Curated Memory (MEMORY.md / USER.md)
│   ├── Knowledge Sources / RAG
│   └── Skills Library
│
├── Tools & Integrations
│   ├── Tool Catalog
│   ├── MCP Servers
│   ├── Plugins & Permissions
│   ├── Providers & Models
│   └── Messaging Channels
│
├── Security & Operations
│   ├── Approval Inbox
│   ├── Audit Logs
│   ├── Runtime Health / Doctor
│   ├── Event Stream
│   └── Resource Usage
│
└── Settings
    ├── General / Appearance
    ├── Profile & Persona
    ├── Model Defaults
    ├── Tools & Safety
    ├── Storage & Memory
    ├── Network & API
    ├── Notifications
    └── About / Diagnostics
```

**Catatan produk:** Artifacts di atas adalah rekomendasi UX (hasil file, gambar, dokumen, diff, export). Inventory dan API artifact perlu diaudit/ditambahkan sebelum UI final dibuat. Bila belum ada API, sembunyikan atau tandai sebagai roadmap.

### 6.3 Struktur global shell

**Desktop / browser lebar**

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ TIKO   [Profile ▾] [Workspace ▾] [Model ▾]   ● Runtime   [⌕] [Bell] [Avatar] │
├───────────────┬───────────────────────────────────────────┬────────────────┤
│ LEFT SIDEBAR  │              MAIN WORKSPACE                │ RIGHT INSPECTOR│
│               │                                           │                │
│ + New Chat    │ Chat / Sessions / Agent / Jobs / Settings │ Run details    │
│ Search        │                                           │ Tool activity  │
│ Sessions      │  transcript / cards / dashboard           │ Context        │
│ Bots          │                                           │ Files/Artifacts│
│               │  composer                                 │ Approval       │
│ Navigation    │                                           │ Timeline       │
│               │                                           │                │
│ Runtime ●     │                                           │                │
└───────────────┴───────────────────────────────────────────┴────────────────┘
```

- Global topbar tinggi sekitar 56–64 px.
- Sidebar kiri 248–280 px, bisa collapse menjadi rail 64–72 px.
- Area content min-width 480 px.
- Inspector kanan 300–360 px, dapat disembunyikan.
- Panel resizing/layout presets dapat menyusul setelah alur inti stabil.
- Header menampilkan runtime/connection state, bukan hanya provider online.

**Android / layar sempit**

```text
┌──────────────────────────────┐
│ ☰  Tiko   Agent ▾    ● Online│
├──────────────────────────────┤
│                              │
│        CHAT / CONTENT        │
│                              │
│   tool cards / GenUI         │
│                              │
│                              │
├──────────────────────────────┤
│ [Attach] Tulis pesan... [➤]  │
├──────────────────────────────┤
│ Chat  Agents  Tasks  More     │
└──────────────────────────────┘
```

- Gunakan Material 3, bottom navigation dengan 4 item: Chat, Agents, Tasks, More. Approval pending tampil sebagai badge pada More dan/atau persistent inbox chip di topbar.
- More menampung Memory, Skills, Integrations, Security, Logs, Settings.
- Global search membuka full-screen search/action palette.
- Halaman approval detail sebaiknya full screen, bukan dialog sempit dengan args yang terpotong.
- Side inspector pada desktop berubah menjadi bottom sheet/drill-down page pada Android.

**Web**

- Gunakan Flutter Web build yang sama dengan Flutter Desktop/Mobile.
- Untuk browser lebar, tampilkan shell desktop; pada mobile web gunakan layout Android responsive.
- Gunakan URL routing untuk halaman utama dan detail, misalnya `/workspace/chat/<sessionId>`, `/agents`, `/automations`, `/skills`, `/settings/providers`.
- Perilaku back/forward browser harus konsisten dan state session dapat dipulihkan.
- Autentikasi web wajib dirancang berbeda dari desktop lokal; jangan bergantung pada token permanen di browser localStorage.

### 6.4 Rancangan visual

Pertahankan token dasar yang sudah tersedia di `apps/agent_flutter/lib/ui/theme.dart` sebagai titik awal:

| Token | Dark (awal) | Penggunaan |
|---|---|---|
| Background | `#0F1115` | kanvas utama |
| Panel | `#171A21` | sidebar, cards, inspector |
| Accent | `#2A9D8F` | primary actions, selection |
| User bubble | `#1B3A4B` | pesan user |
| Assistant bubble | `#1E2430` | pesan assistant |
| Border | `#2A2F3A` | separators dan outline |
| Muted | `#9AA0A6` | secondary text |
| Warning | `#E9C46A` | awaiting approval / caution |
| Danger | `#E76F51` | failure / destructive |

Light theme yang sudah ada di `SimpulTheme` harus dipertahankan dan diuji kontrasnya.

**Perlu ditambahkan sebagai design tokens:**

- `surfaceRaised`, `surfaceHover`, `surfaceSelected`, `surfaceOverlay`.
- `success`, `info`, `warning`, `danger` dengan pasangan foreground/background yang lolos contrast.
- Typography scale: display 28–32, page title 22–24, section 16–18, body 14–16, caption 12–13.
- Spacing 4/8/12/16/24/32; radius 8/12/16; elevation terbatas.
- Status badge: Online, Degraded, Offline, Busy, Needs approval, Error.
- Motion: 120–200 ms untuk hover/expand; hormati reduce-motion.

Hindari terlalu banyak kartu dan badge. Gunakan hierarki tipografi, whitespace, divider halus dan status kontekstual. Tiko perlu terlihat profesional, bukan dashboard admin yang padat.

---

## 7. Spesifikasi tiap layar

### 7.1 Onboarding / First-run setup

Tujuan: pengguna baru bisa mengirim pesan pertama tanpa membaca dokumentasi CLI.

**Langkah tampilan:**

1. Welcome screen: “Selamat datang di Tiko Agent”. Pilih Beginner atau Advanced.
2. Pilih mode runtime:
   - Desktop: Start bundled daemon / Connect to existing daemon.
   - Android: Connect to local network/server (jelaskan Android client tidak otomatis menjalankan daemon dalam konfigurasi yang terverifikasi saat audit).
   - Web: connect via protected server URL.
3. Pilih provider yang tersedia: OpenAI-compatible, Anthropic, Gemini, Ollama, atau provider lain yang benar-benar terdaftar oleh runtime.
4. Input credential melalui secure setup screen; submit ke daemon credential manager. Token tidak dipantulkan ke log/UI.
5. Test Connection; tampilkan error yang actionable.
6. Pilih default model dan optionally reasoning/latency preference bila runtime menyediakan field tersebut.
7. Pilih default safety: Safe Mode ON; konfirmasi berisiko ON; workspace directory bila diperlukan.
8. Test chat dengan prompt contoh sederhana.
9. Selesai → Chat Workspace.

**DoD:** onboarding dapat resume setelah app ditutup; secret tidak disimpan ke local app preferences; pengguna dapat skip setup dengan local/mock provider bila memang tersedia.

### 7.2 Chat Workspace

Desktop content:

- Header: session title, bot/profile, provider/model, connection indicator, kebijakan safety.
- Transcript: Markdown, code blocks, table, images/files, citations bila runtime mengirimnya.
- Live activity rows: Thinking/Planning, Tool call, Subagent, Browser, Memory retrieval, Approval pending, Completed/Failed.
- Tool row dapat expand: nama tool, ringkasan args (sanitized), start/end time, duration, risk, output ringkas, error, tombol view log.
- Artifact cards untuk file/gambar/dokumen/diff hanya jika artifact service ada.
- Composer: multiline, send/stop, attachment, slash commands, context picker, model selector, voice input jika tersambung.
- Scroll anchor: jangan paksa scroll ke bawah jika pengguna sedang membaca pesan lama.
- Search in conversation dan “load earlier messages” setelah transcript API tersedia.

Android:

- Header ringkas dan scroll transcript.
- Tool activity collapsed by default; tap untuk detail.
- Composer pinned di bawah, safe-area aware.
- Approval request menjadi panel yang menempel di atas composer atau halaman detail.

**Penting:** streaming failure tidak boleh memicu pengiriman ulang prompt secara otomatis. Tampilkan “Periksa status sesi sebelum retry” dan sediakan Retry eksplisit dengan peringatan potensi hasil sudah diproses.

### 7.3 Sessions Manager

Setiap row:

- Title (auto-title atau user title), profile/bot, channel, project/workspace.
- Last message preview, last active, status running/idle/failed, unread marker.
- Actions: open, rename, pin, archive, delete (confirmation), export.

Fungsi tambahan:

- Search/filter by profile, provider, channel, date, project.
- Restore transcript page-by-page, preserve reading position locally.
- Start new chat from profile/bot.
- Multi-select export/archive.

**Backend gap:** Flutter API client yang diaudit hanya list sessions; belum ada detail transcript/page/load/create/rename/archive methods yang terlihat. Sediakan endpoint dan contract dahulu sebelum mendesain tombol aktif.

### 7.4 Agents / Bots Manager

Roster view:

- Avatar/initial, agent name, role, status, model/provider, skills count, allowed tools, last activity.
- Filter: active, idle, needs setup, error.
- Create Agent button.

Agent builder wizard:

1. Identity: id, name, description, avatar.
2. Persona: SOUL/persona instructions, greeting, constraints.
3. Model: provider + model default.
4. Skills: choose approved skills.
5. Tools: explicit allowlist; empty allowlist must stay fail-closed.
6. Integrations/MCP: choose allowed servers/tools.
7. Memory: isolated or shared policy; owner context.
8. Safety: approval policy, risk limits, workspace boundaries.
9. Review configuration (diff/summary), save draft, then enable.

Team/Delegation view:

- TaskGraph DAG, node owner, status, dependencies, retry, duration.
- Blackboard/shared output and final aggregation.
- Cancel team run and inspect subagent sessions.

**Backend basis:** BotStore and `/bot` commands are documented; no Bot Manager GUI route was found during this audit. Add dedicated typed REST endpoints rather than requiring UI to send slash commands as opaque text.

### 7.5 Tasks & Automations

Tabs:

- **Tasks**: Kanban (Backlog / Doing / Done / Failed), list and detail.
- **Schedules**: cron jobs, pause/resume, next run, timezone, delivery.
- **History**: timeline of completed/failed automation runs.

Task detail:

- title, prompt/description, assigned agent, dependencies, status, progress events, result, output artifacts, retry/cancel.

Schedule builder:

- natural-language preview (optional), cron expression, timezone (IANA), start/end, delivery target, retry policy, enabled/paused.
- Show human-friendly next 3 run times before saving.
- Run now and Pause/Resume.
- Avoid claiming task state can be edited until APIs support mutations.

### 7.6 Skills Center

Tabs:

- Installed
- Drafts / Needs approval
- Browse Hub (only if remote registry configured)
- Metrics / Feedback

Skill detail:

- Manifest and rendered SKILL.md.
- Triggers, allowed tools, risk, dependencies.
- Status/lifecycle history.
- Static scanner warnings and evaluation results.
- Usage, success rate, 👍/👎 feedback.
- Preview diff, approve, retire, rollback, export/share, install local.

Security UX:

- New/imported skill defaults to Draft/Trial as backend requires; do not silently grant tool permissions.
- Install packages with SHA-256 verification and collision/path safety; no UI bypass.
- Marketplace registry/signature features shown as unavailable until infrastructure and signing design exist.

### 7.7 Memory & Knowledge Center

Overview:

- Counts by type and layer.
- Search box.
- Storage backend and health label (SQLite/FTS/vector/JSON fallback).
- Embedding availability and last indexing status.

Subsections:

- Facts / episodic entries.
- Semantic / curated memory.
- Procedural memory (skills) shortcut.
- Sources and RAG indexes.
- `MEMORY.md` and `USER.md` editor with preview, size budget, diff, save and revert.

Every memory card should reveal source/provenance where possible and provide delete/forget control with confirmation. A memory list from `/v1/memories` alone does not prove provenance or full layer admin API exists; add those endpoints where needed.

### 7.8 Providers & Models

Provider cards should show:

- Display name, provider kind, endpoint, configured profile, status, latency (when measured), last error, model count.
- Credential status only: Connected / Missing / Expired — never reveal secret value.
- Test connection, refresh model list, edit/delete (permission gated).

Model picker should show model/provider, context capability, features (vision, tools, JSON, streaming) when capability probe returns them. Default model can be per profile or per bot; explicit selection scope must be visible to avoid changing the wrong profile.

### 7.9 Tools & MCP Manager

Tool catalog view:

- Category, enabled state, risk class, source, requirements.
- Detail includes JSON schema as Advanced disclosure, effective permission, sandbox constraints, audit last use.
- Enable/disable changes must be backend validated and audited.

MCP manager:

- Add server wizard with transport, command/args or URL, auth method, env/secret references.
- Validate configuration; connect; enumerate tool list; grant/revoke capability consent; inspect logs; disable/remove.
- Do not run a tool just because it appears in an MCP catalog; tool invocation still passes permission/risk checks.

### 7.10 Channels / Messaging

Cards: Telegram, Discord, Slack, WhatsApp, Email/IMAP, Signal, Matrix, Webhook/API.

Each channel page:

- Setup checklist; credential status; allowlist; webhook/redirect details; health; recent delivery errors; restart/reconnect state.
- Test connection or send test message only where the backend has a safe explicit API.
- Show “Needs credentials” or “Not yet verified” accurately.

### 7.11 Security & Approval Center

Approval inbox columns:

- Waiting, Approved, Denied, Expired.
- Tool, risk, session, requesting agent/channel, timestamp.

Approval detail:

- What the tool will do, sanitized args, affected files/host/network, why approval is needed, expiry.
- Buttons Approve once / Deny; future grant only if policy specifically supports persistent scoped grants.
- On Android show critical actions as full-screen confirmation with no accidental adjacent button taps.

Security page:

- Safe mode, permission policy, sandbox and network status.
- Plugin permissions and process status.
- PromptGuard/security events and token anomaly detector only if enabled.
- API token/device sessions management.

### 7.12 Logs, Events & Diagnostics

- Runtime Health: `/v1/status`, `/v1/capabilities`, `/v1/doctor` with human-readable diagnoses.
- Event stream: filter by event category/session/agent/severity.
- Audit logs: list + tail + pagination/download, redacted secrets.
- Provider health and circuit state.
- Export diagnostics bundle with secrets redacted.

Web UI currently has a useful first version of event feed and audit log view; port it into Flutter shared widgets.

### 7.13 Settings Center

Settings harus dikelompokkan menurut ruang lingkup:

**A. Device / App Preferences (lokal perangkat)**

- Theme: system/dark/light, accent.
- Language, compact density, font scale.
- Notification preferences.
- Keyboard shortcuts (desktop/web).
- Auto-lock and local cache preferences.
- Mobile battery/data saver.

**B. Workspace / Project**

- Workspace root / allowed directories.
- Context files and project instructions.
- Default agent/bot.
- Project-specific tool allowlist.

**C. Profile / Agent (daemon-managed)**

- Name/persona/SOUL.
- Provider/model default.
- Skills, tools/toolsets, MCP access.
- Memory policy and sharing.
- Approval policy / risk limit.

**D. Runtime / Daemon**

- API bind address and port (with warning), runtime profile, resource profile.
- Safe mode, scheduler, event retention, storage backend, health endpoints.
- Log level and diagnostic settings.

**E. Credentials / Secrets (server-managed)**

- Provider API keys/OAuth tokens, messaging tokens, browser keys, SMTP/IMAP credentials.
- Add/rotate/remove secret; never reveal stored secret after save.
- Credential encryption status and passphrase readiness warning.

**Existing setting mapping:**

| Existing key/field | New group | Notes |
|---|---|---|
| themeMode | Device / App | already local `shared_preferences` |
| accent | Device / App | already local |
| baseUrl | Device connection | validate URL and TLS requirement |
| token | Secure connection credential | current input; improve encrypted/secure storage |
| ollamaHost | Provider / connection | profile/global scope must be explicit |
| vaultPath | Workspace / Storage | path picker on desktop; URI/document picker on Android |
| autoLockMinutes | Security / Device | distinguish app lock vs vault lock semantics |
| powerSaver | Device / Performance | native battery state integration needed |
| API bind/port | Runtime / Daemon | not just client setting |
| model/provider/keys | Profile / Credentials | new manager UI + API |

---

## 8. API & data contract yang dibutuhkan untuk satu GUI

### 8.1 Endpoint yang sudah tampak digunakan / didokumentasikan

| API | Bukti pada source/docs | Konsumsi UI |
|---|---|---|
| `GET /health` | Flutter client health check | Connection test |
| `GET /v1/status` | Flutter + Web | Runtime/provider status |
| `GET /v1/capabilities` | Flutter | Menu gating, capability badges |
| `GET /v1/doctor` | Flutter | Health diagnostics |
| `GET /v1/sessions` | Flutter + Web | Session list |
| `GET /v1/approvals` | Flutter + Web | Approval inbox |
| `POST /v1/approvals/{id}` | Flutter + Web | Approve/deny |
| `GET /v1/events` | Flutter | Recent event feed |
| `GET /v1/memories` | Flutter + Web | Memory facts list |
| `DELETE /v1/memories/{id}` | Flutter | Forget/delete memory |
| `POST /v1/messages` | Flutter client | Non-stream chat send |
| `POST /v1/auth/ticket` | Flutter/Web stream setup | One-time WebSocket ticket |
| `GET /v1/stream` (WS upgrade) | Flutter/Web | Chat/event stream |
| `GET /v1/tasks` | Agent Manager UI docs | Kanban read model |
| `GET /v1/logs` | Agent Manager UI docs | Log list |
| `GET /v1/logs/{name}` | Agent Manager UI docs | Log tail |
| `GET /v1/skills` | Web UI source | Skills listing |
| `POST /v1/skills/{slug}/feedback` | Agent Manager / feedback docs | Skill feedback |
| `POST /v1/sessions/{id}/feedback` | Feedback docs | Session/turn feedback |
| `POST /v1/chat/completions`, `GET /v1/models` | OpenAI-compatible server docs | External clients; separate from UI control API unless intentionally unified |

**Catatan:** tabel ini adalah inventory yang ditemukan lewat client/UI dan dokumen. Sebelum implementasi, uji route pada daemon dan tandai request/response schema sebagai kontrak OpenAPI atau typed Dart models. Adanya route di dokumentasi tidak sama dengan uji live pada perangkat.

### 8.2 Endpoint baru yang direkomendasikan

Gunakan prefix versi yang konsisten dan struktur REST. Nama berikut proposal, perlu diselaraskan dengan convention API aktual:

| Domain | Proposal endpoint | Tujuan |
|---|---|---|
| Chat | `GET /v1/sessions/{id}/messages?cursor=` | transcript pagination |
| Sessions | `POST /v1/sessions`, `PATCH /v1/sessions/{id}`, `POST /v1/sessions/{id}/archive` | create/rename/archive |
| Bots | `GET/POST /v1/bots`, `GET/PATCH/DELETE /v1/bots/{id}` | BotStore CRUD |
| Bots | `POST /v1/bots/{id}/activate`, `/deactivate` | bind/unbind session |
| Providers | `GET/POST /v1/providers`, `PATCH/DELETE /v1/providers/{id}` | provider configuration |
| Providers | `POST /v1/providers/{id}/test`, `GET /v1/providers/{id}/models` | test + models |
| Secrets | `PUT/DELETE /v1/secrets/{ref}` | server-side secret references only |
| Tools | `GET /v1/tools`, `GET /v1/tools/{name}` | catalog and details |
| Tools | `PATCH /v1/tools/{name}/policy` | enablement / scoped permission |
| MCP | `GET/POST /v1/mcp/servers`, `PATCH/DELETE /v1/mcp/servers/{id}` | manage MCP config |
| MCP | `POST /v1/mcp/servers/{id}/connect`, `/disconnect`, `/consent` | lifecycle and consent |
| Plugins | `GET /v1/plugins`, `POST /v1/plugins/{id}/consent` | plugin control |
| Skills | `GET /v1/skills/{slug}`, `POST /v1/skills/{slug}/approve`, `/retire`, `/rollback` | lifecycle and review |
| Skills Hub | `GET /v1/skills/hub/search?q=`, `POST /v1/skills/hub/install` | optional remote hub |
| Memory | `GET/PATCH /v1/memory/curated` | MEMORY.md / USER.md read/write with validation |
| Memory | `POST /v1/memory/search`, `GET /v1/memory/sources` | search/provenance/RAG source list |
| Memory | `POST /v1/memory/reindex`, `DELETE /v1/memory/sources/{id}` | ingestion lifecycle |
| Jobs | `GET/POST /v1/jobs`, `PATCH/DELETE /v1/jobs/{id}` | cron CRUD |
| Jobs | `POST /v1/jobs/{id}/run`, `/pause`, `/resume` | job operations |
| Tasks | `GET /v1/tasks/{id}`, `POST /v1/tasks/{id}/cancel`, `/retry` | task detail/actions |
| Channels | `GET/POST /v1/channels`, `PATCH/DELETE /v1/channels/{id}` | gateway UI |
| Channels | `POST /v1/channels/{id}/test`, `/restart` | connection health |
| Artifacts | `GET /v1/artifacts`, `GET /v1/artifacts/{id}` | result files/images/diffs |
| Observability | `GET /v1/metrics`, `GET /v1/events?cursor=` | health/resource/event paging |
| GenUI | `POST /v1/ui/actions` or typed event `ui_action` | safely dispatch button/form action |

### 8.3 Prinsip API

- Semua route mutasi wajib authenticated, validated, audited, dan idempotent sesuai jenis aksi.
- Setiap resource perlu id stabil; list memakai cursor/page dan filter.
- Error response typed (`code`, `message`, `details`, `retryable`, `request_id`).
- Return effective config dan capability; jangan membuat UI menebak hasil penyimpanan.
- Jangan expose raw config secrets atau complete environment variables.
- WebSocket event harus punya `event_id`, `occurred_at`, `session_id`, `agent_id`, `type`, `payload`, `schema_version`.
- Tambahkan event `run.started`, `run.plan`, `tool.started`, `tool.completed`, `tool.failed`, `subagent.started`, `subagent.completed`, `approval.required`, `approval.decided`, `run.completed`, `run.cancelled` bila belum tersedia dengan kontrak yang konsisten.
- UI action harus typed; jangan menjadikan JSON action ID sebagai perintah shell atau handler arbitrer.

### 8.4 GenUI action contract — prioritas tinggi

Kondisi dokumentasi D1: model mengeluarkan DSL allowlist `genui`, Flutter merender widget; form/action kembali sebagai pesan teks `/ui <actionId> <json>`. D1 secara jujur mencatat agentd belum memiliki handler khusus bagi format itu.

Rancangan perbaikan:

1. Setiap GenUI block berisi `spec_id`, `actions[]` yang terikat ke daftar action yang diizinkan runtime, form schema, dan expiry/session scope.
2. Renderer mengirim `POST /v1/ui/actions` dengan `{session_id, spec_id, action_id, values, idempotency_key}`.
3. Server memvalidasi action id terhadap spec yang pernah dikeluarkan runtime, tipe field, batas ukuran, role/permission, expiry, dan session ownership.
4. Server mengirim hasil sebagai event stream terstruktur (`ui_action.result`) atau `tool` result.
5. Web, Android dan desktop menggunakan parser/renderer widget yang sama melalui `tiko_widgets` atau package shared; schema tetap grammar-closed.
6. Tambahkan test negatif: action id palsu, spec expired, form tampered, duplicate idempotency key, unknown field, oversized payload, unauthorized session.

---

## 9. Arsitektur implementasi yang direkomendasikan

### 9.1 Satu aplikasi Flutter untuk tiga platform

```text
apps/agent_flutter/
├── lib/
│   ├── app/
│   │   ├── app.dart
│   │   ├── app_router.dart
│   │   ├── app_bootstrap.dart
│   │   └── capability_menu.dart
│   ├── shell/
│   │   ├── desktop_shell.dart
│   │   ├── mobile_shell.dart
│   │   ├── web_shell.dart
│   │   ├── sidebar.dart
│   │   ├── bottom_nav.dart
│   │   └── inspector_panel.dart
│   ├── features/
│   │   ├── chat/
│   │   ├── sessions/
│   │   ├── agents/
│   │   ├── automations/
│   │   ├── skills/
│   │   ├── memory/
│   │   ├── providers/
│   │   ├── tools/
│   │   ├── mcp/
│   │   ├── channels/
│   │   ├── approvals/
│   │   ├── logs/
│   │   └── settings/
│   ├── data/
│   │   ├── api/
│   │   ├── repositories/
│   │   └── models/
│   ├── core/
│   │   ├── design_system/
│   │   ├── security/
│   │   ├── state/
│   │   ├── navigation/
│   │   └── utils/
│   └── main.dart
└── test/
```

Folder layout adalah proposal struktur kerja. Refactor harus bertahap; jangan memindahkan seluruh `main.dart` sekaligus sebelum regression tests siap.

### 9.2 Lapisan aplikasi

1. **Presentation:** reusable widgets, responsive shell, route/page.
2. **Feature state:** controller/notifier per domain (chat, sessions, providers, etc.).
3. **Repository:** typed methods dan cache/invalidation policy.
4. **API transport:** HTTP, WebSocket, tickets, auth, retry policy.
5. **Runtime:** `agentd` + `TikoRuntime`; UI tidak menjalankan model/tool secara langsung.

### 9.3 Komponen bersama

Paket `apps/tiko_widgets` sudah ada untuk widget reusable. Kembangkan atau pecah menjadi shared modules:

- `TikoAppBar`, `RuntimeStatus`, `ProfileSwitcher`, `ModelPicker`.
- `SessionTile`, `AgentCard`, `TaskCard`, `ApprovalCard`.
- `MessageBubble`, `MarkdownBlock`, `ToolActivityCard`, `RunTimeline`.
- `GenUiMessageView`, `GenUiView`, form controls dan action dispatch client.
- `EmptyState`, `ErrorState`, `LoadingSkeleton`, `ConfirmSheet`.
- `SettingsSection`, `SecretField`, `ConnectionStateCard`.

Komponen tidak boleh melakukan network request secara tersembunyi. Terima data dan callback, sementara repository/controller mengurus async.

### 9.4 State dan sinkronisasi lintas platform

- Remote config/profile/agents/sessions/jobs/skills/memory/approvals adalah sumber kebenaran daemon.
- Device preferences tetap lokal: theme, accent, panel width, keyboard shortcuts, last-open tab, scroll position.
- Session transcript harus diambil dari daemon dan disimpan cache secara bounded; perubahan lintas klien diterima melalui event stream.
- Hindari polling penuh setiap 15 detik untuk semua domain. Gunakan WebSocket event stream untuk invalidation dan polling fallback berkala.
- Gunakan generation/request id agar hasil fetch lama tidak menimpa state yang baru (pola generasi `refreshAdmin` yang sudah ada bisa dipertahankan).
- Cache API perlu TTL spesifik: status/health pendek; provider config dan skills sedang; transcript paginated; credential never cache in UI.

### 9.5 Backend lifecycle per platform

**Desktop:** pertahankan `DesktopDaemon.ensure` dan binary bundled `agentd`. Berikan status Spawned/Reused/Manual, home path dan log path; tambahkan kontrol shutdown/restart hanya melalui endpoint terautentikasi.

**Android:** pendekatan awal adalah client-first. Pengguna menghubungkan Android ke daemon desktop/server di LAN/VPN/host. Fitur daemon on-device atau Termux service menjadi fase terpisah karena perlu keputusan mengenai background execution, lifecycle, wakelock/battery optimization, file permissions, local binaries, sandbox dan update mechanism. Jangan menyatakan Android mobile app setara daemon desktop sebelum QA real device.

**Web:** build Flutter Web. Idealnya serve dari domain/origin yang sama dengan gateway UI atau reverse proxy yang aman. Dashboard browser lokal/remote memerlukan auth yang kuat, TLS untuk non-loopback, Origin/CSRF policy dan rate limiting.

---

## 10. Tahapan implementasi — step by step

Tahapan di bawah disusun berdasarkan dependensi dan risiko, bukan jadwal kalender. Kerjakan dan merge per milestone dengan test + dokumentasi di setiap tahap.

### Tahap 0 — Kunci baseline dan spesifikasi kontrak

**Tujuan:** pastikan tidak ada fitur lama hilang ketika UI digabung.

1. Checkout commit `dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae` dan catat working tree bersih.
2. Jalankan `dart --version`, `flutter --version`, `node --version` dan `git status`.
3. Jalankan validasi yang ada:
   - `dart analyze --fatal-warnings`
   - `dart test` pada workspace/paket sesuai CI
   - `bash scripts/web_ui_validate.sh`
   - `cd apps/agent_flutter && flutter pub get && flutter analyze && flutter test`
   - `flutter build web --release`
   - desktop build jobs dan Android build sesuai workflow.
4. Export route list/response schema dari `HttpApiServer`; bandingkan dengan `AgentApiClient`, `app.js` dan docs.
5. Buat feature parity matrix baseline yang menyebut route, status backend, client surfaces, test evidence.
6. Simpan UI screenshots baseline pada desktop, Android dan browser untuk regression visual.

**Output:** `docs/GUI-BASELINE-2026-11.md`, API inventory, screenshots, daftar known gaps.  
**Kriteria selesai:** baseline tests diketahui; semua route/UI screens punya status bukti; tidak ada fitur lama hilang dalam migrasi.

### Tahap 1 — Design system dan shell responsif tunggal

1. Tetapkan token warna, typography, spacing, radius, elevation, states dan breakpoints.
2. Refactor `main.dart` menjadi `AppBootstrap`, `AppRouter`, `DesktopShell`, `MobileShell` dan `WebShell`.
3. Gunakan shared sidebar navigation; implementasi desktop rail collapse.
4. Implementasi Android bottom nav + drawer/More.
5. Implementasi browser URL routing/back/forward.
6. Pastikan label dan menu domain sama lintas platform; hanya presentasi berubah.
7. Tambah theme dark/light/system dan accent controls dengan test.

**Output:** app shell baru, design tokens, route tests, golden/widget tests.  
**Kriteria selesai:** semua platform memakai satu route map dan komponen bersama; navigasi dapat diakses keyboard/screen reader.

### Tahap 2 — Satukan API client dan repository layer

1. Pisahkan `AgentApiClient` menjadi transport + domain repositories (chat, session, approval, memory, task, skill, provider, MCP, etc.).
2. Definisikan typed Dart models dan serializers; hindari raw `Map<String,dynamic>` di widget layer.
3. Standardisasi API error model (`unauthorized`, `forbidden`, `not_found`, `conflict`, `rate_limited`, `server`, `network`, `timeout`).
4. Implementasikan auth token exchange dan lifecycle secure storage per platform.
5. Buat event stream client yang satu sumber untuk run events, notifications dan admin invalidation.
6. Terapkan bounded request timeout, cancellation, request ids, idempotency keys untuk mutasi.

**Output:** `data/api`, `data/repositories`, typed DTO dan test contract.  
**Kriteria selesai:** UI tidak memanggil fetch/http langsung; test dapat mengganti repository dengan fake; tidak ada secret dalam log.

### Tahap 3 — Sessions & Chat Workspace parity

1. Tambahkan API load transcript page-by-page, session details, create, rename, archive, delete, export.
2. Bangun session sidebar dengan search, filter, pin, active/running/unread indicator.
3. Pulihkan transcript saat session dipilih; jangan memanggil clearChat untuk session historis.
4. Persist read position dan load-earlier paging.
5. Pindahkan chat bubble, Markdown, code, attachment preview dan tool activity ke `tiko_widgets`.
6. Tambahkan run inspector: plan, tools, subagents, outputs, failures, timings.
7. Tambahkan stop/cancel dan safe retry flow; jangan auto-resend saat stream disconnect.
8. Sediakan search in conversation.

**Output:** Chat Workspace konsisten di desktop, Android dan web.  
**Kriteria selesai:** session yang dibuat di web bisa dibuka di Android/desktop dengan transcript yang sama; approval event muncul lintas client.

### Tahap 4 — GenUI universal dan typed actions

1. Pertahankan grammar tertutup D1 dan batas depth/node/text/ID.
2. Tambahkan typed server-side action registry dan `/v1/ui/actions`.
3. Implementasikan action spec registry per session dengan expiry + ownership.
4. Pindahkan renderer ke reusable package yang kompatibel dengan Flutter Web dan native.
5. Pastikan fallback invalid block tetap terlihat sebagai plain text, bukan UI yang setengah dirender.
6. Tambahkan tests for tampering, repeated action, invalid field, expired spec, unauthorized session.

**Output:** GenUI render/action identik pada tiga platform.  
**Kriteria selesai:** form/button pada Web, Android, Desktop benar-benar memanggil action handler yang tervalidasi, bukan sekadar mengirim perintah teks bebas.

### Tahap 5 — Approval, Tasks, Scheduler dan Audit UI

1. Port task Kanban dari web UI menjadi Flutter feature page.
2. Tambahkan task detail, progress, assigned agent, output, cancel/retry APIs bila belum ada.
3. Buat scheduler CRUD UI setelah endpoint job terverifikasi/ditambahkan.
4. Gabungkan approval inbox Flutter + web; detail risk/action/payload sanitized.
5. Port event feed dan log browser web ke shared widgets.
6. Tambahkan filter/search/pagination dan diagnostic export dengan redaction.

**Output:** Automation Center dan Security Operations.  
**Kriteria selesai:** semua approval pending bisa ditangani dari tiap platform; tidak ada operasi job mutasi yang hanya berjalan di web atau hanya di CLI.

### Tahap 6 — Agents/Bots & Teams UI

1. Tambahkan REST CRUD `BotStore`.
2. Implementasikan roster + profile switcher.
3. Buat create/edit wizard untuk persona/model/skills/tools allowlist.
4. Buat approval review untuk perubahan permission.
5. Tambahkan active binding per session dan bot group routing status.
6. Implementasikan Team/Delegation graph dari task DAG dan subagent events.
7. Tampilkan history bot/group chat dan status tiap agent.

**Output:** Agent Manager.  
**Kriteria selesai:** pengguna baru dapat membuat bot, memilih model, mengatur allowlist dan menjalankan satu tugas tanpa perintah `/bot` manual.

### Tahap 7 — Skills, Memory, Knowledge dan Learning Center

1. Port Skills list + feedback dari web ke Flutter.
2. Tambahkan skill detail, scanner results, evaluate/promote/approve/retire/rollback.
3. Tambahkan local install dan hub search hanya untuk backend yang benar-benar tersedia.
4. Tambahkan curated memory endpoints dan editor MEMORY.md/USER.md.
5. Tambahkan RAG sources/index status/search; provenance dan delete.
6. Tambahkan learning timeline dan self-evolution metrics dengan label jelas untuk estimated/observed.

**Output:** Knowledge Center.  
**Kriteria selesai:** seluruh lifecycle skill dapat dioperasikan lewat GUI; semua write memakai validation/security gate yang sama dengan CLI/runtime.

### Tahap 8 — Providers, Tools, MCP, Plugins dan Channels

1. Implementasi provider registry CRUD/test/model list.
2. Implementasi CredentialManager integration; enkripsi at rest dan key lifecycle.
3. Build Tool Catalog dan permission/risk detail.
4. Build MCP server manager, consent grant/revoke, reconnect/log.
5. Build plugin manager dengan process health, capability consent dan revoke.
6. Build channel setup wizard dengan status actual runtime; tidak expose secret.
7. Tampilkan configuration changes diff + Apply + restart status bila perlu.

**Output:** Integrations & Capabilities Center.  
**Kriteria selesai:** setup integrasi standar dapat dilakukan dari GUI tanpa edit YAML/env manual, kecuali advanced escape hatch.

### Tahap 9 — Android product readiness

1. Konfirmasi Android hanya remote client pada rilis awal; jelaskan koneksi daemon.
2. Implement secure token handling (Android Keystore via plugin/native wrapper bila diperlukan).
3. Implement file picker SAF/document URI, share sheet dan notification deep links.
4. Implement Android lifecycle: background/foreground, process recreation, network change, offline banner.
5. Buat pending approval notifications dengan deep link ke request.
6. Uji keyboard, screen sizes, orientation, font scaling, TalkBack, battery/data saver.
7. Siapkan release flavor, keystore/signing, versioning, R8/proguard policy bila relevan, privacy/data safety notes.
8. Uji APK/AAB Release pada perangkat fisik; CI debug APK saja tidak cukup.

**Output:** Android release candidate.  
**Kriteria selesai:** tidak ada credential leak; app pulih setelah process kill/network loss; approval notifications and deep links work; release artifact terverifikasi.

### Tahap 10 — Web product readiness dan keamanan deployment

1. `flutter build web --release` menjadi UI web resmi.
2. Tentukan deployment: static hosting + API gateway atau same-origin `agentd` reverse proxy.
3. Pastikan HTTPS untuk non-loopback; CORS allowlist; Origin checks; CSRF mitigation; CSP; rate limit.
4. Implement browser auth session/short-lived tokens; jangan memakai raw persistent bearer in localStorage.
5. Route deep links dan back/forward; refresh on nested route.
6. Uji Chrome, Firefox, Safari/WebKit, Edge, mobile browser.
7. Maintain minimal admin/fallback web page only if still needed; sunset duplicated HTML UI by feature freeze and migration notice.

**Output:** one official responsive web client.  
**Kriteria selesai:** web and Flutter views have same feature contracts and user-facing terms; server deployment has auth hardening.

### Tahap 11 — Quality gates, accessibility dan telemetry

1. Unit test for API client/repositories and state controllers.
2. Widget tests for each screen and responsive breakpoints.
3. Golden tests for core screens in dark/light.
4. Integration tests against fake agentd server.
5. End-to-end tests for login/provider connection/chat/tool/approval/session/skills/jobs.
6. Device test Android real hardware, desktop OS matrix, browsers.
7. Security tests for auth, IDOR/session ownership, XSS, CSRF, SSRF, path traversal, secret redaction, approval bypass, replay/idempotency.
8. Performance budgets: app startup, first chat render, memory list pagination, stream latency, large session transcript.
9. Test offline/daemon unavailable/provider degraded/timeouts and recovery.
10. Update docs and changelog in same PR as feature changes.

**Output:** release quality gate with traceable CI artifacts.  
**Kriteria selesai:** all supported target builds + test sets pass; no known P0/P1 UI security issues; visual snapshots approved.

---

## 11. Prioritas backlog

### P0 — wajib sebelum menyatakan GUI terpadu

| ID | Item | Alasan | Acceptance criteria |
|---|---|---|---|
| GUI-P0-01 | Satu shell/nav untuk Flutter Android/Desktop/Web | Fragmentasi produk adalah masalah utama | Satu route map dan design tokens bersama |
| GUI-P0-02 | Session transcript load/paging + CRUD dasar | Sessions sekarang list/select saja | Buka session lama dengan isi lengkap |
| GUI-P0-03 | API contract/error model typed | UI saat ini bergantung Map mentah | Semua page memakai repository/DTO |
| GUI-P0-04 | GenUI actions typed server-side | Action/form belum benar-benar punya handler khusus | Validated server action + tests |
| GUI-P0-05 | Secure credential/session handling | Browser localStorage dan manual token entry berisiko | Tidak ada secret raw di log/UI/local preferences |
| GUI-P0-06 | Cross-client approvals | Tool risk membutuhkan keputusan | Satu approval state terlihat di semua client |

### P1 — fitur parity agent management

| ID | Item | Alasan | Acceptance criteria |
|---|---|---|---|
| GUI-P1-01 | Tasks board Flutter/web shared | Web sudah punya board, Flutter belum | Task board identical data on all clients |
| GUI-P1-02 | Event feed + logs Flutter | Operasional/debug | Searchable logs + live events |
| GUI-P1-03 | Skills Manager Flutter | Skill lifecycle belum GUI di Flutter | Draft-review-approve-active-retire |
| GUI-P1-04 | Bot/Agent Manager | Bot mode hanya command | Create/edit/activate bots via GUI |
| GUI-P1-05 | Provider & Model Manager | Settings provider parsial | Test connection, model list, default selection |
| GUI-P1-06 | MCP/Plugin/Tools Manager | Banyak capability tersembunyi | Setup/consent/health via GUI |
| GUI-P1-07 | Channels setup | Gateway tergantung env/secret | Wizard + health + allowlist |
| GUI-P1-08 | Settings scope model | Device/global/profile tercampur | Explicit Applies to scope |

### P2 — diferensiator dan polish

| ID | Item | Manfaat | Acceptance criteria |
|---|---|---|---|
| GUI-P2-01 | Simple/Advanced mode | Pengguna baru dan power user | Mode layout tanpa mengubah kemampuan runtime |
| GUI-P2-02 | Command palette | Produktivitas desktop/web | Ctrl/Cmd+K action/search |
| GUI-P2-03 | Multi-pane layout | Monitor luas & multitasking | Session, files, inspector panes |
| GUI-P2-04 | Artifacts & diff viewer | Hasil agent dapat direview | Download/preview only through safe APIs |
| GUI-P2-05 | Memory graph / learning timeline | Memahami progres pengetahuan agent | Accurate provenance + edit/delete controls |
| GUI-P2-06 | Voice UI | Hands-free/mobile use | STT/TTS verified end-to-end per provider |
| GUI-P2-07 | Desktop OS notifications | Background run completion | Permission + deep link |
| GUI-P2-08 | PWA install and offline shell | Web accessibility | Offline state explicit, writes queued safely |

---

## 12. Risiko dan mitigasi

| Risiko | Severity | Mitigasi |
|---|---|---|
| Dua frontend tetap berkembang sendiri-sendiri | High | Feature freeze pada HTML/JS; migrasi ke Flutter; maintain fallback read-only sementara |
| UI menampilkan fitur backend yang belum wired | High | Capability-driven menu dan feature status; disabled states explain missing prerequisites |
| GenUI action hanya menjadi teks bebas | High | Typed action API, allowlist, spec/session binding, idempotency |
| Token API disimpan di browser localStorage | High | Browser session auth / short-lived token; server-side session cookie if deployment supports it |
| Android tidak dapat menjangkau daemon karena loopback | High | Onboarding koneksi; bind 0.0.0.0 hanya dengan TLS/auth/firewall; prefer VPN/tunnel secure |
| Android background service dibangun sebelum QA | High | Rilis awal remote client; local daemon/Termux sebagai milestone terpisah |
| Approval UX menampilkan argumen tidak lengkap | High | Sanitized structured preview, full detail page, explicit target/effect |
| Pengguna menyamakan “library ada” dengan “integrasi live siap” | Medium | Label conditional, setup requirements, health checks |
| Secret encryption belum diaktifkan | High | Credential setup checks for passphrase; warn if legacy plaintext fallback is active |
| Web UI tests hanya unit/syntax | Medium | Browser E2E and visual regression CI |
| Session list tanpa transcript history | High | Transcript paging API and persistence integration |
| Terlalu banyak menu membuat pemula bingung | Medium | Simple mode, progressive disclosure, onboarding, search/action palette |
| Full UI refactor merusak fungsi yang sudah ada | High | Move by features with regression tests; no big-bang rewrite |

### 12.1 Catatan keamanan khusus credential

Dokumen `docs/CREDENTIAL-ENCRYPTION-2026-11-15.md` menjelaskan credential dapat disegel menggunakan `SecretStore` saat passphrase tersedia melalui `TIKO_SECRET_PASSPHRASE`; formatnya TSS2, memakai XChaCha20-Poly1305 dengan kunci Argon2id. Tanpa passphrase, kompatibilitas lama mempertahankan JSON plaintext dengan permission file `0600`. Karena itu GUI perlu menunjukkan **encryption status** dan memperingatkan operator bila credential store berjalan pada fallback plaintext. Jangan menampilkan badge “encrypted” hanya berdasarkan keberadaan file.

### 12.2 Jaringan dan Android

Klien Flutter menggunakan API base URL; README menunjukkan emulator Android mengakses host dev via `10.0.2.2`, sedangkan perangkat fisik perlu endpoint yang bisa dijangkau. Daemon default bind loopback. Remote access membutuhkan konfigurasi host, TLS, firewall dan Origin policy yang sesuai. Jangan meminta pengguna membuka API ke internet publik tanpa deployment hardening.

---

## 13. Test plan dan acceptance criteria lintas platform

### 13.1 Feature parity checklist

Untuk setiap fitur di tabel, uji skenario ini di Android, Desktop dan Web:

- Menu/entry point tersedia pada pola navigasi yang sesuai.
- Capability disabled menjelaskan prasyarat, bukan error generik.
- Loading/empty/error/offline/permission-denied states tersedia.
- Create/update/delete actions memanggil API yang sama.
- Perubahan yang dibuat dari satu platform terlihat di platform lain setelah event/poll.
- Deep link/session id membuka record yang benar.
- Tidak ada data rahasia yang muncul pada UI/log/error report.

### 13.2 Core E2E tests

1. First run → configure provider → test → first chat response.
2. Create session → send request → tool activity → result → close app → reopen session and transcript.
3. High-risk tool request → approval pending → approve from second platform → first client receives event → tool proceeds once.
4. Deny/expire approval → no tool side effect.
5. Stream drops after server accepted prompt → UI does not auto-resend; user sees safe retry guidance.
6. Generate GenUI form → submit → server validates action and idempotency → result event updates UI.
7. Create bot → set persona/model/tool allowlist → activate → chat → deactivate.
8. Create task/job → run → event stream update → view log/result.
9. Install skill locally → scanner → draft → evaluate → approve → feedback.
10. Add MCP server → consent → enumerate tools → revoke consent → invocation fails closed.
11. Update credential → verify encrypted storage → restart daemon → provider test.
12. Browser session expires → auth refresh/login flow; no bearer leaked via URL.

### 13.3 Build matrix

| Target | Build gate | Additional verification |
|---|---|---|
| Android | `flutter build apk --release` (and AAB if Play distribution) | Real device, Keystore, notification/deep link, file picker, network changes |
| Windows | `flutter build windows --release` | Bundled agentd, installer/upgrade/uninstall, firewall, path handling |
| macOS | `flutter build macos --release` | Signing/notarization, sandbox permissions, bundled daemon lifecycle |
| Linux | `flutter build linux --release` | Package format, permissions, daemon child process, desktop file |
| Web | `flutter build web --release` | Browser E2E, HTTPS, cookies/auth, refresh deep links, CSP/CORS |

Catatan: CI pada snapshot memiliki Flutter analyze/test/build web, desktop jobs untuk Linux/Windows/macOS dengan daemon bundle, serta job Android debug APK. Release signing dan real-device QA tidak boleh dianggap selesai hanya dari job debug.

---

## 14. Panduan operasional setelah desain diputuskan

### 14.1 Menjalankan daemon untuk development

Dari root repository:

```bash
dart pub get
dart run apps/agentd/bin/agentd.dart --api
```

Daemon menyiapkan API credential sesuai mekanisme `agentd`; file token lokal ditulis pada data directory dengan permission aman. Jangan commit token atau memasukkannya ke issue/log.

### 14.2 Menjalankan Flutter client

```bash
cd apps/agent_flutter
flutter pub get
flutter run --dart-define=API_BASE=http://127.0.0.1:8787
```

Untuk Android Emulator yang terhubung ke daemon host dev, gunakan base URL `http://10.0.2.2:8787`. Pada perangkat fisik gunakan alamat endpoint yang aman dan terjangkau; jangan menganggap `127.0.0.1` mengarah ke komputer host.

### 14.3 Web UI lama selama migrasi

```bash
dart run apps/agentd/bin/agentd.dart --api
```

Web UI lama disajikan oleh `agentd` pada alamat lokal yang dikonfigurasi. Dokumen proyek mengarahkan pengguna mengakses local UI dan memasukkan API base/token dari mesin tepercaya. UI tersebut sebaiknya dipertahankan hanya sampai seluruh fitur (Tasks, Skills, Logs, Events) sudah dimigrasikan ke Flutter Workspace.

### 14.4 Validasi yang tersedia

```bash
bash scripts/web_ui_validate.sh
bash scripts/flutter_validate.sh
```

Jalankan juga Dart workspace tests mengikuti `.github/workflows/ci.yml`. Perlu diketahui script Flutter validation membutuhkan Flutter SDK. Audit ini tidak mengklaim telah mengeksekusi seluruh perintah tersebut.

---

## 15. Keputusan yang direkomendasikan

### Keputusan arsitektur

**Pilih Flutter sebagai satu-satunya aplikasi GUI utama.** Flutter sudah menjadi client bersama lintas Android/Desktop/Web dan punya design tokens serta GenUI renderer. `apps/tiko_widgets` memberi tempat untuk mengembangkan komponen reusable. HTML/JS web UI tetap menjadi fallback sementara sampai fitur web telah dipindahkan, kemudian dibekukan atau dihapus setelah migration acceptance selesai.

### Keputusan produk

- Jadikan Chat Workspace sebagai homepage.
- Tampilkan Sessions dan Agents/Bots sebagai bagian penting sidebar.
- Tampilkan Approvals sebagai inbox global dan status badge.
- Gabungkan Tasks dan Scheduler dalam Automation Center tetapi pisahkan semantik task vs recurring job.
- Satukan Skills, Memory dan RAG dalam Knowledge Center.
- Satukan Provider, Tools, MCP, Plugins dan Channels dalam Integrations Center.
- Jadikan Logs/Events/Doctor sebagai Security & Operations Center.
- Pisahkan setting per-device, per-workspace, per-profile dan server secrets.
- Tambahkan Beginner/Simple dan Advanced mode; bukan dua aplikasi yang berbeda.

### Urutan implementasi paling aman

**Jangan mulai dari menggambar seluruh layar atau menulis ulang frontend.** Mulai dari API contract dan unified shell; setelah itu tutup session transcript dan GenUI action contract; lalu port Skills/Tasks/Logs dari web ke Flutter; kemudian implementasikan Agent Manager, Provider/Tools/MCP/Channels; terakhir hardening Android/Web release dan UX polish.

Ini menghindari situasi UI cantik tetapi tombol tidak terhubung, atau dua platform memproses state yang berbeda.

---

## 16. Referensi source utama

Semua link code di bawah merujuk ke branch/snapshot yang diperiksa. Untuk reproduksi audit, gunakan commit `dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae`.

### Repository dan aplikasi

- [README.md](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/README.md)
- [Flutter app shell — main.dart](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agent_flutter/lib/main.dart)
- [Flutter API client](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agent_flutter/lib/agent_api_client.dart)
- [Flutter chat controller](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agent_flutter/lib/ui/chat_controller.dart)
- [Flutter settings view](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agent_flutter/lib/ui/settings_view.dart)
- [Flutter settings store](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agent_flutter/lib/ui/settings_store.dart)
- [Flutter theme](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agent_flutter/lib/ui/theme.dart)
- [Flutter UI README](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agent_flutter/README.md)
- [agentd web index](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agentd/web/index.html)
- [agentd web app.js](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agentd/web/app.js)
- [agentd web CSS](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agentd/web/app.css)
- [agentd Web UI README](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agentd/web/README-UI-REPAIR.md)
- [agentd entry point](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/apps/agentd/bin/agentd.dart)
- [CI workflow](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/.github/workflows/ci.yml)

### Dokumen fitur dan status terbaru

- [Architecture index](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/00-INDEX.md)
- [Hermes parity gaps](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/HERMES-PARITY-GAPS-2026-11-10.md)
- [Hermes parity batch](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/HERMES-PARITY-BATCH-2026-11-13.md)
- [Agent Manager UI](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/AGENT-MANAGER-UI-2026-11-13.md)
- [Bot Mode](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/BOT-MODE-2026-11-15.md)
- [Curated Memory and Learn](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/LEARN-CURATED-MEMORY-2026-11-15.md)
- [Credential encryption](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/CREDENTIAL-ENCRYPTION-2026-11-15.md)
- [Plugin isolation](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/PLUGIN-ISOLATION-2026-11-15.md)
- [Feedback loop](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/FEEDBACK-LOOP-2026-11-16.md)
- [Flutter WebSocket edge tests](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/FLUTTER-WS-EDGE-2026-11-16.md)
- [GenUI D1](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/GENUI-D1-2026-11-17.md)
- [D-series closure](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/D-SERIES-CLOSURE-2026-11-17.md)
- [E-series closure](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/E-SERIES-CLOSURE-2026-11-17.md)
- [Roadmap closure audit](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/ROADMAP-CLOSURE-AUDIT-2026-11-17.md)
- [Happy path manual](https://github.com/kiosmarmer-commits/Tiko-Agent/blob/dcdea3b1dbafcfb05dd4658463b0ea9a22db41ae/docs/10-HAPPY-PATH-MANUAL.md)

### Referensi resmi Hermes

- [Hermes Desktop documentation](https://hermes-agent.nousresearch.com/docs/user-guide/desktop)
- [Hermes Web Dashboard documentation](https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard)
- [Hermes configuration documentation](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)

---

## 17. Kesimpulan akhir

Tiko-Agent tidak perlu memulai ulang dari nol untuk mencapai pengalaman GUI sekelas agent desktop modern. Fondasi penting—daemon/runtime separation, Flutter multi-platform client, WebSocket streaming, approvals, memory, skills, tasks/events web UI, Bot Mode, reusable widgets dan GenUI—sudah terlihat pada source branch `main`.

Yang paling mendesak adalah menyatukan pengalaman tersebut menjadi satu produk: **satu Workspace, satu navigasi, satu API contract, satu model session, satu approval flow dan satu design system**. Pencapaian paling bernilai bukan menambah puluhan halaman sekaligus, melainkan memastikan setiap kemampuan yang ada dapat ditemukan, dipahami, diatur secara aman, dan bekerja dengan cara yang konsisten pada Android, Desktop dan Web.

Urutan yang disarankan: **Unified Shell → API/Session parity → Chat + Run Inspector → GenUI actions → Approvals/Tasks/Logs → Agents/Bots → Skills/Memory → Providers/Tools/MCP/Channels → Android/Web hardening.**

