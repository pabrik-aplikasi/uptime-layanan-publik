# Uptime situs layanan publik

Project `311-uptime-layanan-publik` · kategori 3 · dokumentasi perencanaan.

Masalah: Tidak ada data publik soal ketersediaan situs pemerintah.

**Belum ada implementasi aplikasi.** Stack utama dan alternatif sudah dipetakan dan direview sebagai rencana, bukan kombinasi versi yang telah lulus spike/e2e. Pembuatan repository ini tidak meloloskan gate wave, GO/NO-GO, stack final, desain atau deploy.

## Mulai membaca

- [Ide asli dan batas budget/platform](docs/idea.md)
- [Stack utama, alternatif, alasan dan kontrak](docs/stack.md)
- [Auth, data/migrasi, runtime, operasional dan gate](docs/stack-support.md)
- [Pembahasan, keputusan dan triage](docs/decisions.md)
- [Status dan kelanjutan](docs/progress.md)
- [Sumber dan provenance](docs/sources.md)

Stack utama: **Go · Gin · Astro + React islands · TimescaleDB**. Masuk ranking triage; belum dipilih untuk wave implementasi.

Repository ini akan menjadi monorepo. Folder aplikasi, dependency, workflow CI dan konfigurasi deploy belum dibuat; tidak ada perintah build/test yang sudah diverifikasi.
