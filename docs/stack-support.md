# Stack pendukung — Uptime situs layanan publik

Rencana, bukan hasil pengujian. Kolom delta mengganti komponen utama bila alternatif dipilih. Tidak menginstal atau mengaktifkan kandidat † tanpa persetujuan yang diperlukan. Catatan “belum commit/provisioning” dalam kutipan sumber merujuk tahap perencanaan; repository dokumentasi kini dibuat terpisah dari implementasi.

| ID | Auth / batas akses | Data & migrasi | Runtime / API / validasi | Komponen khusus | Operasional / biaya | Delta alternatif |
| --- | --- | --- | --- | --- | --- | --- |
| 311 | Dashboard publik; daftar probe hanya operator. | TimescaleDB; sqlc + pgx† + goose†; River schema internal. | Go Gin, net/http timeout; validasi allowlist tujuan. | River probes, retensi/agregasi; SSRF dan redirect dibatasi. | P Railway; PG extension, worker, egress probe; single-region dicantumkan. | Fastify Node + React/TypeBox; Drizzle/pg/drizzle-kit + pg-boss mengganti River; verifikasi host mendukung TimescaleDB sebelum deploy. |

## Cara membaca pilihan dan approval

- Nama tanpa versi adalah keluarga teknologi yang dipilih untuk rencana. Versi compiler/runtime/SDK/model harus dikunci bersama lockfile dan checksum pada fase validasi nanti, bukan menebak versi terbaru sekarang.
- Tanda **†** berarti tambahan dependency/tool/provider konkret yang belum memperoleh persetujuan khusus di palette. Ia boleh ditelaah sebagai kandidat, tetapi tidak boleh dipasang atau diaktifkan hanya karena tercantum di sini. Persetujuan umum merencanakan bukan persetujuan membeli atau menginstal. SDK/driver pendamping tetap perlu audit meskipun teknologi induknya disetujui.
- `none` / tanpa akun / tanpa DB merupakan pilihan yang disengaja. Library lokal, CLI dan browser-only tidak diberi server hanya agar terlihat lengkap.
- `S3` berarti AWS S3 sebagai kandidat cloud yang konkret, bukan resource yang sudah ada. Filesystem lokal hanya jalur demo; tidak membuktikan signed URL, multipart, lifecycle atau permission S3 bekerja. Data sensitif memakai object privat dengan akses terbatas, bukan bucket publik.
- Kolom operasional mencatat target dan penggerak biaya, bukan penawaran harga. Volume pengguna/event, retensi, ukuran objek, durasi komputasi dan harga wilayah belum cukup untuk mengeluarkan estimasi bulanan yang dapat dipercaya.
- Hal yang belum dipilih karena izin data, kontrak eksternal atau ketersediaan hardware ditulis sebagai gate. Tidak ada fallback diam-diam menjadi scraping tanpa izin, API berbayar, model cloud, PWA pengganti native, atau implementasi kriptografi sendiri.

## Kontrak pendukung bersama

### Auth adalah identitas; izin domain tetap milik aplikasi

Untuk aplikasi TS dengan runtime dan adapter sesuai, Better Auth adalah kandidat library utama; tidak menambah service TS kedua untuk aplikasi yang tidak memerlukannya. Schema auth yang dihasilkan masuk satu jalur migrasi yang ditunjuk, bukan dua migrator yang saling mengubah tabel. Session browser memerlukan kebijakan cookie/CSRF; token native memerlukan penyimpanan aman dan revocation yang sesuai. Pilihan per baris dapat memakai auth bawaan PocketBase, Clerk untuk Convex, GitHub App, atau tanpa akun bila lebih cocok.

Untuk aplikasi non-TS multiuser, Keycloak OIDC adalah kandidat IdP agar core domain tidak perlu menulis sistem password sendiri. Browser/native menggunakan authorization code dengan PKCE bila sesuai jenis client; tidak memakai password grant atau menyimpan client secret dalam browser/aplikasi native. Server memvalidasi issuer, audience, signature, expiry dan izin domain. Keycloak **menambah service dan database milik IdP, default PostgreSQL**; jika DB domain bukan PostgreSQL atau tidak ada DB domain, tambahan ini tetap harus dihitung. Pengecualian eksplisit alternatif610 memakai database identity MySQL terpisah pada instance MySQL, yang juga didukung Keycloak. Ia bukan dependency tersembunyi dan bukan default wajib bagi CLI, demo lokal atau situs anonim.

Tenant membership, owner/resource access, hubungan wali–murid, role pengawas, entitlement pembayaran dan policy medis tetap diperiksa di core aplikasi. Role IdP bukan pengganti pemeriksaan baris/data. Capability link hanya memberi akses sempit, dapat dicabut/kedaluwarsa; bukan token admin atau pengganti autentikasi semua pengguna. Integrasi GitHub App menggunakan installation permission yang dibatasi, bukan PAT bersama.

### Satu pemilik migrasi per schema

Driver/query generator mengikuti bahasa yang menulis transaksi. Tool migrasi dipilih per baris; tidak menjalankan Drizzle, goose dan SQLx bergantian pada tabel yang sama. Auth, domain, queue/workflow, dan sync bucket memiliki ownership schema yang terpisah secara eksplisit, walaupun dapat berbagi mesin DB. Framework-managed schema seperti Convex/PocketBase tidak ditulis langsung melalui driver SQL eksternal.

Alternatif mengganti driver, serializer, validator dan migrator milik aplikasinya; ia mewarisi kontrak bisnis, platform, privacy dan durability kecuali delta mengatakan lain. Pergantian pilihan stack bukan migrasi data yang sudah ada, karena belum ada aplikasi yang dibangun. Frontend TS tidak memiliki kredensial DB server.

### Durable jobs, sync dan realtime tidak saling menggantikan

Job/outbox dicatat atomik bersama transaksi domain bila diperlukan. Worker mengklaim pekerjaan, retry dengan batas, menyimpan hasil dan menangani duplikasi. JetStream untuk pekerjaan memakai file-backed storage, durable consumer dan acknowledgement setelah hasil durable; Core NATS hanya boleh untuk data ephemeral yang dapat dipulihkan. Tidak ada klaim exactly-once efek eksternal.

Semua PowerSync mencakup service, bucket storage PostgreSQL terpisah secara logis, auth, **Sync Streams** dan upload API tervalidasi. Sync Rules lama berstatus deprecated menurut dokumentasi saat dibaca; baseline baru memilih rilis yang mendukung Sync Streams, bukan mencampur YAML/contoh lama. Source MySQL memerlukan bucket database tambahan; status source/adapter perlu dipastikan kembali saat memilih versi. PowerSync tidak mengubah draft ujian menjadi submission accepted atau stok global menjadi pasti tersedia saat perangkat offline. SDK native Expo memakai adapter OP-SQLite, sementara web/Electron memakai adapter masing-masing yang dijelaskan pada713; tidak dianggap saling tertukar.

Jika Encore Pub/Sub dipilih, provider dan resource harus benar-benar ada dalam inventaris. Provider tambahan yang disebut dalam tabel merupakan kandidat, bukan kemampuan broker otomatis dari image Encore. Tidak ada broker baru pada baseline312 dan tidak ada Trigger.dev pada baseline512.

### Packaging dan protokol

Semua project tetap memiliki repository sendiri, monorepo di dalamnya. Pilihan bersama untuk bagian JS/TS adalah **pnpm workspaces**; Bun tetap boleh menjadi runtime/test runner ketika tabel utama memilihnya, bukan alasan mengganti bahasa/runtime. Cargo, Go modules, Gradle, Mix, dotnet, Gleam, Crystal, Nim, OCaml dan Zig tetap memiliki dependency/build native masing-masing. Tidak menambahkan Nx/Turborepo atau task runner lintas bahasa untuk semua project.

Pemanggilan build lintas bahasa dapat memakai workspace scripts sederhana pada fase implementasi, tetapi dependency graph native tidak dipindah ke package manager JS. CI yang direncanakan adalah GitHub Actions dengan checks unit, integrasi service nyata dan e2e mengikuti profil pada peta utama; belum ada workflow atau runner yang dibuat. Runtime/image dan dependency harus dipin saat validasi, bukan memakai tag latest di produksi.

API eksternal memiliki satu kontrak versioned: OpenAPI, Protobuf/Connect atau schema framework sesuai baris. Typed client tidak menggantikan validasi server. Rust–TS memakai salah satu boundary nyata: WASM, napi-rs pada Node, subprocess berversi, atau API/queue. Go alternatif tidak memuat addon Node. Generated code tidak dianggap aman hanya karena lolos parser.

### Navigasi dan state interface

Profil berikut berlaku untuk seluruh baris dengan interface terkait, termasuk alternatif; tidak mengubah pilihan React/Svelte/Solid/Astro atau native pada peta utama. Layar tunggal tanpa navigasi tidak perlu router tambahan.

| Interface | Pilihan pendukung | Batas |
| --- | --- | --- |
| React SPA | TanStack Router untuk multi-route; TanStack Query untuk remote server state; state React untuk interaksi lokal | Cache query bukan sumber saldo/stok/entitlement; tidak menambah Zustand/XState tanpa kebutuhan |
| Svelte SPA | svelte-spa-router† untuk multi-route; state bawaan Svelte dan fetch untuk akses API sederhana | Kandidat router hash, bukan migrasi ke SvelteKit; halaman yang memakai fragment untuk kunci/capability tidak membagi fragment dengan router |
| Solid SPA | Solid Router† untuk multi-route; state bawaan Solid dan fetch untuk akses API sederhana | Tidak mengubahnya menjadi SolidStart atau menyalin algoritma core ke state UI |
| TanStack Start / SvelteKit / SolidStart / Astro | Router dan data-loading bawaan framework; state island mengikuti React/Svelte/Solid yang memang dipilih | Tidak memasang router SPA kedua ke seluruh halaman SSR/SSG |
| Expo | Expo Router†; TanStack Query untuk remote API bila bukan data milik sync engine; state React lokal | Native screen dan deep link; bukan pengganti offline storage atau bukti plugin bekerja di Expo Go |
| Kotlin native | Toolkit dan persistence Android yang disebut pada baris501/509; tidak memakai library DOM | Build/test memerlukan toolchain dan runner/perangkat native |
| CLI/API-only | Tidak ada router/UI state | Tidak menambah dashboard hanya untuk menyeragamkan |

Data Convex tetap melalui client reactive Convex, draft PowerSync melalui SQLite/sync, dan dokumen Yjs melalui CRDT; jangan membuat salinan otoritatif kedua di query cache. Form sederhana memakai kontrol platform, schema validasi yang dipilih pada baris dan validasi server; belum menambah library form global. Browser-only tanpa data remote tidak memerlukan query library. Styling/komponen tetap mengikuti R1–R4, S1–S2, O1, A1, N1 dan K1 pada peta utama.

### Profil operasional dan biaya

| Profil | Target rencana dan layanan dasar | Batas biaya dan bukti berikutnya |
| --- | --- | --- |
| L | Binary/CLI, browser lokal, build mobile atau desktop; log lokal teredaksi; tanpa telemetry keluar secara default | Distribusi binary/aplikasi, ukuran model/aset, perangkat uji dan signing mungkin berbiaya; lokal tidak berarti semua distribusi gratis |
| S | Docker Compose lokal untuk server/DB/worker yang memang dipilih; JSON logs teredaksi, health/readiness, backup/restore, restart recovery | Belum menyewa VPS. Jika di-host, hitung API, worker, IdP, DB, broker, sync, storage, backup, traffic dan vendor; jangan hanya menghitung web process |
| P | Layanan publik kategori3, target Railway/VPS sesuai peta utama; static build bila tidak ada backend, rate limiting/abuse control bila ada API | Tidak otomatis memenuhi <US$20/bulan. Tambahan Keycloak/Baserow/Convex, bandwidth dan API model harus masuk estimasi |
| M | Managed service yang sudah dipilih, terutama Convex; export/restore dan batas penggunaan tetap perlu dirancang | Tidak membuat akun/berlangganan. Kuota, metering, lokasi data dan biaya auth/storage diperiksa sebelum aktivasi |

Profil dapat digabung, misalnya L+S untuk aplikasi native dengan server, atau P+M untuk web publik dengan Convex. Profil tidak menghapus target hosting pada tabel utama.

Observability bersama: JSON logs dengan request/job ID dan redaksi data sensitif sebagai baseline. OpenTelemetry dipakai pada sinyal/runtime yang didukung; ekspor hanya diaktifkan setelah tujuan dan privacy disepakati. Kandidat lingkungan observability lokal adalah **Grafana LGTM**, opsional untuk demo/test terpisah, bukan rangkaian layanan yang wajib dipasang per project kecil atau konfigurasi produksi yang sudah siap. Jangan memaksa SDK JavaScript ke runtime browser/WASM atau menganggap semua bahasa eksplorasi memiliki kematangan SDK yang sama. App lokal tidak mengirim dokumen, gaji, token atau prompt ke telemetry otomatis.

Untuk estimasi nanti, setiap project memerlukan: pengguna aktif dan puncak concurrency; volume/ukuran event dan objek; retensi+backup; durasi dan frekuensi job; egress; panggilan AI/WA/mail/payment; serta biaya runner/perangkat. Jumlah resource idle juga dihitung. Tidak ada tarif atau jaminan budget yang dikarang pada tahap ini.

### Provider dan model yang dipilih untuk rencana

Pilihan ini melengkapi baris yang hanya menulis WA, email atau LLM; tidak menambahkan integrasi tersebut ke project yang tidak membutuhkannya. Provider yang disebut khusus pada suatu baris mengungguli default ini. Semua komunikasi nyata, pengiriman data pengguna dan biaya tetap memerlukan approval.

| Kebutuhan | Kandidat konkret | Batas keputusan |
| --- | --- | --- |
| WA dan email | Meta WhatsApp Cloud API†; Resend† untuk email transaksional | Consent, template, domain, sandbox dan biaya belum disetujui; demo hanya menangkap intent lokal |
| Pembayaran | Midtrans† QRIS untuk pembayaran satu kali; Subscription API kartu/GoPay pada604 | Bukan QRIS recurring; onboarding, callback, refund dan rekonsiliasi masih gate |
| Ringkasan/label/penulisan cloud pada110/111/310/408/607/608/814 | OpenAI† `gpt-5.6-terra` sebagai kandidat awal, AI SDK pada TS; HTTP provider pada core Go/Rust | Pilihan untuk evaluasi, bukan klaim paling murah/akurat; schema output, token budget, consent, PII dan availability harus diuji; tidak mengubah115 menjadi benchmark satu model |
| Transkrip teks608/814 | OpenAI† `gpt-transcribe` melalui Audio Transcriptions API | Tidak mengasumsikan word timestamps, speaker diarization atau SRT dari endpoint teks; consent dan kualitas Bahasa harus dibuktikan |
| Subtitle bertiming615 | whisper.cpp† CLI dengan model `base` multilingual†, output SRT/VTT; FFmpeg† untuk media | Proses C/C++ eksternal tidak mengganti Hono/Node sebagai aplikasi utama; CPU/RAM, timing dan kualitas model perlu diuji, bukan API hosted Whisper |
| AI lokal809 | Transformers.js + ONNX Runtime Web, distribusi ONNX Qwen2.5-0.5B-Instruct† | Tidak memakai provider cloud; versi/hash artefak, operator/model compatibility, kualitas ID/EN dan restart offline tetap gate |

Pemilihan model814 dikoreksi saat konsolidasi: halaman resmi menandai `gpt-4.1-mini` dan hosted `whisper-1` deprecated. Karena itu keduanya tidak dijadikan baseline baru. Model dan provider tetap terpisah dari aturan bisnis, agar lifecycle model tidak memaksa pergantian bahasa/framework. Library/SDK yang belum menerima identifier/model baru harus di-upgrade setelah audit atau memakai API HTTP yang terdokumentasi, bukan menganggap dukungan otomatis. Tidak ada API model dipanggil pada pekerjaan dokumen ini.

### Konvensi kategori 3 dan 5

- **† = kandidat di luar palette, menunggu persetujuan**, termasuk driver/library non-TS yang disebut. SQLite standalone serta kandidat Phoenix/Reitit/Giraffe/Wisp/Kemal sudah boleh dipertimbangkan, tetapi tidak perlu disisipkan ke kategori ini. Izin ekosistem JS/TS modern tidak menggantikan pemeriksaan maintenance, lisensi dan keamanan. Tidak mengunci versi atau mengklaim kombinasi sudah kompatibel.
- **L:** proses/data lokal, tanpa telemetry remote; diagnostik lokal tanpa isi sensitif. **S:** Docker Compose lokal, JSON logs teredaksi, liveness/readiness, backup/restore; OpenTelemetry hanya ketika SDK runtime mendukung, export opt-in. **P:** target hosting kategori 3 yang sudah tertulis—Railway/VPS—bukan deployment baru; baseline S untuk server, L untuk komputasi browser. **M:** Convex managed yang sudah menjadi pilihan matriks; bukan provisioning baru. Anggaran **< US$20/bulan belum dijamin**; hitung compute, RAM, DB, backup, storage, egress, auth, scheduler dan API/model sesuai pemakaian.
- **Monorepo:** pnpm workspaces untuk TS, ditambah Go modules, Cargo atau Gradle sesuai bahasa; tidak menambah task runner. Platform native tetap aplikasi Android/Expo, bukan PWA. Web memakai komponen/UI matriks sebelumnya; Expo memakai komponen React Native/Expo, tidak mengimpor widget DOM. Kotlin memilih Jetpack Compose† + Material 3†, bukan label toolkit kosong.
- **Satu pemilik schema:** backend core memiliki schema bisnis dan urutan migrasinya. TS biasa: Drizzle + drizzle-kit; Go: sqlc + driver sesuai DB + goose†; Rust: SQLx† + SQLx migrations; Kotlin server: JDBC + Flyway† kecuali HTTP libSQL yang dinyatakan eksplisit. Schema internal Keycloak, River, DBOS dan PowerSync dimiliki alat masing-masing, tidak diduplikasi ORM aplikasi. Satu release job mengurutkan migrasi, bukan seluruh replica menjalankannya bersamaan.
- **Auth:** Better Auth untuk backend TS yang cocok; Expo memakai plugin Better Auth + SecureStore. Non-TS multiuser memakai Keycloak OIDC authorization-code + PKCE; validasi issuer, audience, expiry dan signature di core. Browser tidak menyimpan client secret. Tenant/role/ownership selalu diverifikasi core, bukan hanya UI/JWT role. Tool lokal tanpa akun tidak mendapat auth tambahan. Capability link adalah izin terbatas untuk objek/aksi/masa berlaku, bukan akun lengkap.
- **S3:** AWS S3 private adalah kandidat cloud saat diperlukan, menunggu approval biaya; signed URL berumur pendek bukan tautan sekali-pakai. Adapter filesystem untuk demo lokal **bukan bukti S3**. Permission, checksum, expiry, retensi dan abort upload harus dibuktikan pada penyimpanan target sebelum klaim selesai.
- **Turso pada rancangan ini berarti endpoint libSQL** agar driver tidak ambigu: TS `@libsql/client`; Go `libsql-client-go`† melalui `database/sql`; Kotlin Ktor HttpClient melalui SQL-over-HTTP. Jangan mengganti ke engine Turso baru tanpa meninjau driver dan semantik transaksi. Migrasi Go libSQL memakai SQL bernomor + tabel versi melalui driver yang sama, bukan mengasumsikan goose binary mengenali semua driver remote.

### Gate kategori 3 dan 5

1. **Approval dependency dan biaya:** setujui kandidat †, khususnya Room/Compose/Material 3/WorkManager, SQLx, pgx/goose, JDBC/Flyway dan Rust WASM. Library JS baru tetap perlu audit maintenance/lisensi; AGPL pada background-removal tidak diasumsikan sesuai semua rencana distribusi. Tidak memilih langganan berbayar, mengirim email/WA, atau menyalakan cloud di tahap ini.
2. **Native dan offline sesudah restart:** 501 harus membuat/membaca/mengubah catatan dan restore backup tanpa jaringan setelah process death. 509 harus menyimpan hasil panen dan outbox secara atomik, bertahan reboot, menangani retry duplikat, token kedaluwarsa, perubahan izin, konflik lintas perangkat dan perangkat hilang. WorkManager bukan jaminan waktu eksekusi tepat. 514 dan Expo lainnya online-first kecuali kontrak offline disetujui tersendiri; jangan menambahkan sync/storage lokal tanpa kebutuhan. Bukti native memerlukan Android runner/device; tidak ada pengujian tersebut pada pekerjaan dokumen ini.
3. **Gate509 alternatif:** PowerSync Service adalah layanan sungguhan di samping Huma. Pilih PostgreSQL source + database/schema bucket terpisah logis dengan role terpisah; migrasi aplikasi milik goose, bucket milik PowerSync. Satu server fisik hanya bila versi/config yang dipilih mendukung; bukan menghilangkan biaya bucket. Keycloak JWT/JWKS dan audience dikonfigurasi, **Sync Streams** scoped koperasi/petani menjadi baseline baru; Sync Rules legacy bukan konfigurasi baru yang dipilih. Kunci rilis yang mendukungnya dan jangan mencampur contoh. `uploadData` harus menuju Huma upload API yang memverifikasi authz/idempotency/konflik sebelum menulis source PG; PowerSync bukan pengganti write API. Pilih adapter native `@powersync/op-sqlite` + `@op-engineering/op-sqlite`, bukan mengganti dengan expo-sqlite; membutuhkan native development build, bukan asumsi Expo Go. Logout/account-switch wajib membersihkan atau mengisolasi local replica.
4. **Auth dan schema ownership:** Better Auth pada Encore312/515 memerlukan mounting raw handler + session bridge ke auth handler Encore; schema auth dapat digenerate, tetapi migrasi akhirnya dijalankan Encore saja. Convex307 Svelte memakai `setupAuth`/token fetch, bukan React provider; 514 Expo memakai provider resmi React. Uji expired token, refresh, revoke, lintas tenant dan capability read/write sebelum klaim integrasi selesai. Core authorization tetap wajib meski login berhasil.
5. **Outbox dan scheduler:** 312 PostgreSQL dan 512 libSQL memakai insert domain+outbox satu transaksi; worker memiliki lease, retry/backoff, dedup dan reconciliation. Railway/VPS self-host membutuhkan proses/scheduler eksplisit; deklarasi cron Encore tidak dianggap otomatis berjalan di semua hosting. Alternatif Rust502/505 mempertahankan PG outbox + scheduled worker tanpa broker NATS. WA/email harus idempotent dan sandbox dahulu, tanpa pengiriman sungguhan sebelum approval.
6. **Driver dan migrasi:** Turso engine/endpoint harus cocok dengan libSQL client yang dipilih. Ktor504 memakai HTTP baton/parameter binding; timeouts, rollback dan per-statement error harus dibuktikan, bukan diasumsikan sebagai JDBC. Migration path upgrade/rollback operasional dan restore harus diuji pada DB nyata. River/DBOS/PowerSync memiliki schema internal sendiri; jangan membuat dua migrator menguasai tabel sama.
7. **Hosting, security dan legal:** 308/313/315/316 tetap conditional GO; bandwidth/abuse/lisensi/regulasi tidak diselesaikan pilihan framework. 309 offline hanya sesudah seluruh model/WASM/font ter-cache dan proses restart diuji tanpa internet. Image/WASM, PDF/CLI/browser worker, consumer-package mirror dan sumber fiskal/kalender memerlukan bukti tersendiri. Tidak ada hasil test, angka load atau harga hosting yang dikarang.

## Sumber kontrak bersama

- [pnpm workspaces](https://pnpm.io/workspaces): dukungan workspace/package linking di satu repository; bukan pengganti toolchain native.
- [Keycloak OIDC](https://www.keycloak.org/securing-apps/oidc-layers): authorization, token, discovery, client types dan grant; password grant tidak dipilih.
- [Keycloak database](https://www.keycloak.org/server/db): penyimpanan user/client/realm di database relasional; PostgreSQL adalah pilihan, database dev bukan target produksi.
- [Better Auth database](https://www.better-auth.com/docs/concepts/database): adapter, schema generation, dan batas migrasi built-in dibanding adapter ORM.
- [OpenTelemetry language SDKs](https://opentelemetry.io/docs/languages/): kematangan berbeda per bahasa dan sinyal; bukan bukti instrumentation semua app sudah berjalan.
- [Grafana Docker LGTM](https://grafana.com/docs/opentelemetry/docker-lgtm/): lingkungan observability development/demo/testing, bukan produksi siap pakai.
- [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html): object/bucket, akses, lifecycle, region dan biaya berbasis penggunaan.
- [TanStack Router](https://tanstack.com/router/latest/docs/framework/react/overview) dan [Query](https://tanstack.com/query/latest/docs/framework/react/overview): navigasi React dan remote server state; cache bukan transaksi bisnis.
- [Solid Router](https://docs.solidjs.com/solid-router) dan [svelte-spa-router](https://github.com/ItalyPaleAle/svelte-spa-router): pilihan router masing-masing framework; yang terakhir berbasis hash dan mendokumentasikan Svelte5.
- [Expo Router](https://docs.expo.dev/router/introduction/): file-based navigation dan deep linking untuk Expo native; tidak memverifikasi integrasi perangkat.
- [PowerSync Sync Rules legacy](https://docs.powersync.com/sync/rules/overview): status deprecated dan arah Sync Streams untuk konfigurasi baru.
- [GPT-5.6 Terra](https://developers.openai.com/api/docs/models/gpt-5.6-terra) dan [GPT-Transcribe](https://developers.openai.com/api/docs/models/gpt-transcribe): kandidat API/model yang disebut; status dan endpoint dibaca ulang secara live, tanpa memanggil inference.
- [GPT-4.1 mini](https://developers.openai.com/api/docs/models/gpt-4.1-mini), [Whisper hosted](https://developers.openai.com/api/docs/models/whisper-1) dan [panduan transkripsi](https://developers.openai.com/api/docs/guides/speech-to-text): kandidat lama berstatus deprecated; contoh timestamp/subtitle lama bukan janji untuk model pengganti.
- [whisper.cpp CLI](https://raw.githubusercontent.com/ggml-org/whisper.cpp/master/examples/cli/README.md) dan [model files](https://github.com/ggml-org/whisper.cpp/blob/master/models/README.md): opsi output SRT/VTT/JSON dan model `base` multilingual. Ini runtime/model lokal, bukan endpoint deprecated `whisper-1`.

Sumber di setiap kelompok di bawah memperinci pilihan driver, framework, media, native dan model. Membaca dokumentasi memverifikasi kemampuan tertentu, bukan kompatibilitas kombinasi versi, akurasi model, keamanan, performa atau biaya nyata.

### Sumber kategori 3 dan 5

Pembacaan sumber memverifikasi kapabilitas yang disebut, **bukan** seluruh kombinasi atau versi. Seluruh brief kategori3/5 dan batas palette menjadi dasar alokasi. URL dokumentasi bergerak; cocokkan kembali dengan rilis yang akhirnya dikunci. Halaman lama PowerSync `/self-hosting/overview`, `/self-hosting/architecture` dan Turso `/sdk/java` mengembalikan 404; tidak dijadikan bukti dukungan.

- [Room](https://developer.android.com/training/data-storage/room): abstraksi SQLite, query terverifikasi dan jalur migrasi; bukan sync engine. [Compose](https://developer.android.com/jetpack/compose): toolkit UI Kotlin native Android. [WorkManager](https://developer.android.com/develop/background-work/background-tasks/persistent): pekerjaan persisten melintasi restart/reboot, bukan jadwal real-time.
- [Expo SQLite](https://docs.expo.dev/versions/latest/sdk/sqlite/): DB bertahan restart, initialization/migration hooks. [Better Auth Expo](https://www.better-auth.com/docs/integrations/expo): plugin native, SecureStore, callback/deep-link dan cookie authenticated request.
- [Keycloak OIDC](https://www.keycloak.org/securing-apps/oidc-layers): discovery, authorization-code, token dan JWKS; direct password grant bukan pilihan. [Better Auth Drizzle](https://www.better-auth.com/docs/adapters/drizzle): PG/MySQL/SQLite adapters dan schema generation; migrasi bisa dimiliki Drizzle pipeline.
- [Convex Clerk](https://docs.convex.dev/auth/clerk): issuer/applicationID, JWT verification, Expo-compatible React provider dan `getUserIdentity`. [Convex Svelte](https://docs.convex.dev/client/svelte) serta [Svelte authentication](https://docs.convex.dev/client/svelte/authentication): `setupAuth`, refresh/clear token, tidak ada dedicated Clerk Svelte wrapper yang diasumsikan.
- [PowerSync Expo/native](https://docs.powersync.com/client-sdk-references/react-native-and-expo): SQLite adapter, OP-SQLite, native-build caveat, `fetchCredentials` dan `uploadData`. [Self-host configuration](https://docs.powersync.com/configuration/powersync-service/self-hosted-instances): source, bucket PG/Mongo terpisah, JWT/JWKS/audience, sync config; aturan berbagi server tergantung versi PG. [Sync Rules legacy](https://docs.powersync.com/sync/rules/overview): nomenklatur lama, transisi ke Sync Streams.
- [pgx](https://pkg.go.dev/github.com/jackc/pgx/v5): driver PG native Go dan bridge database/sql. [Go MySQL driver](https://pkg.go.dev/github.com/go-sql-driver/mysql): database/sql MySQL. [goose](https://pressly.github.io/goose/): incremental SQL/Go migrations. [River migrations](https://riverqueue.com/docs/migrations): tabel internal, runner dan opsi integrasi dengan migration framework.
- [Turso Go](https://docs.turso.tech/sdk/go/reference): driver dibedakan menurut engine; libsql-client-go untuk libSQL remote. [SQL over HTTP](https://docs.turso.tech/sdk/http/reference): parameter binding, pipeline, baton/transaksi dan timeout; dasar pilihan Ktor504 tanpa mengarang JDBC remote.
- [SQLx](https://docs.rs/sqlx/latest/sqlx/): driver PG/MySQL dan migrate module. [pgJDBC](https://jdbc.postgresql.org/documentation/): JDBC PG. [MySQL Connector/J](https://dev.mysql.com/doc/connector-j/en/): JDBC MySQL. [Flyway database drivers](https://documentation.red-gate.com/flyway/reference/database-driver-reference): referensi PG/MySQL; bukan klaim edisi gratis mendukung seluruh fitur.
- [Ktor database integration](https://ktor.io/docs/server-integrate-database.html): JSON serialization dan integrasi database JVM; query JDBC blocking membutuhkan penanganan dispatcher. Rancangan ini tidak mengadopsi Exposed hanya karena dipakai contoh docs.
- [Drizzle PG](https://orm.drizzle.team/docs/get-started-postgresql) dan [Drizzle Turso](https://orm.drizzle.team/docs/connect-turso): adapter pg/libSQL; contoh versi docs tidak otomatis dipin. [Encore SQLDatabase](https://encore.dev/docs/ts/primitives/databases): SQL migrations dan satu definisi resource. [Encore self-host](https://encore.dev/docs/self-host/configure-infra): konfigurasi koneksi resource eksternal untuk Docker image.
- [DBOS TypeScript](https://docs.dbos.dev/typescript/programming-guide): PostgreSQL system database, workflow/step checkpoints dan recovery; bukan jaminan exactly-once layanan eksternal.
- [Rust image](https://docs.rs/image/latest/image/): encode/decode dan image operations; WASM target masih perlu spike. [IMG.LY background removal](https://github.com/imgly/background-removal-js): browser package dan lisensi AGPL; bukan otomatis persetujuan lisensi.
- [Playwright PDF](https://playwright.dev/docs/api/class-page#page-pdf): PDF buffer dan print CSS. [axe-core](https://github.com/dequelabs/axe-core): audit otomatis/localization, masih memerlukan manual review. [S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html): upload/download terbatas waktu, dapat dipakai berkali-kali sampai kedaluwarsa.
- [Material 3 Compose](https://developer.android.com/develop/ui/compose/designsystems/material3): komponen dan theming native. [Serde](https://serde.rs/): serialization/deserialization, bukan pengganti validasi bisnis. [wasm-bindgen](https://wasm-bindgen.github.io/wasm-bindgen/): binding Rust WASM–JavaScript.
- [PDF.js](https://mozilla.github.io/pdf.js/): parsing/rendering PDF. [Mammoth](https://github.com/mwilliamson/mammoth.js): DOCX raw-text extraction, tanpa sanitization otomatis; pilih raw text escaped, external-file access nonaktif, proses ber-timeout/memory-limit. [pdf-lib](https://pdf-lib.js.org/): pembuatan PDF di browser/Node. [qrcode](https://github.com/soldair/node-qrcode): QR generator browser/server; dukungan runtime bukan bukti maintenance final.

Asumsi yang diputuskan untuk draft: 501 single-peternak tanpa sync; 306 editor Baserow hanya mengelola template; web/mobile selain kontrak offline di atas tetap online-first; backend native Kotlin berjalan di JVM, bukan mengklaim Kotlin/Native server. Dokumen ini tidak mereservasi stack, mengubah registry atau meloloskan gate produk/deploy.
