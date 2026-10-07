# Design: Counter Pengunjung Global (index.html)

**Date:** 2026-10-07
**Status:** Approved
**Scope:** `index.html` only

## Goal

Menampilkan counter pengunjung global (semua pengunjung, lintas perangkat) di landing page `index.html` koleksi game.

## Constraints

- Situs statis di GitHub Pages, tanpa backend, tanpa build tools (lihat `AGENTS.md`).
- Tanpa signup/token layanan.
- Self-contained: perubahan hanya di `index.html`, inline CSS/JS mengikuti gaya yang ada.
- Tidak ada tambahan library selain CDN yang sudah ada.

## Chosen approach: Busuanzi resmi

Layanan: `cdn.busuanzi.cc` (docs: busuanzi.cc), versi `3.6.9`.

Alasan: tanpa signup, endpoint mengirim `access-control-allow-origin: *` (diverifikasi langsung), mendukung UV + PV, integrasi dua baris.

Alternatif yang ditolak:

- **LibreCounter** (`librecounter.org`) — open source dan tepercaya, tetapi endpoint `siteStats` tidak mengirim header CORS (diverifikasi), sehingga angka tidak dapat dibaca JS dari GitHub Pages; hanya bisa ditampilkan sebagai gambar embed yang tidak cocok dengan tipografi halaman.
- **`bsz.saop.cc`** — API JSON rapi dan CORS OK, tetapi proyek hobby per-orangan dengan umur layanan tidak pasti.
- **CounterAPI** — v1 sudah `410 Gone` (diverifikasi); v2 wajib daftar akun/token.

Trade-off yang diterima: Busuanzi adalah layanan Tiongkok dengan riwayat pernah 502 sesekali. Karenanya desain menyediakan fallback (lihat di bawah).

## Behavior

1. `index.html` memuat `<script defer src="https://cdn.busuanzi.cc/busuanzi/3.6.9/busuanzi.min.js">` di `<head>`.
2. Script Busuanzi mengirim POST `{url, referrer}` ke `cdn.busuanzi.cc/api.php` dan mengisi elemen dengan ID yang cocok:
   - `busuanzi_site_uv` → jumlah pengunjung unik (UV)
   - `busuanzi_site_pv` → total kunjungan halaman (PV)
3. Hanya `index.html` yang memuat script, sehingga PV = kunjungan halaman landing; ketiga file game tidak dihitung (keputusan pengguna).

## UI

- Satu pill di **footer** `index.html`, sisi kiri, memakai gaya badge yang sudah ada:
  `rounded-full bg-slate-50/10 ring-1 ring-inset ring-white/15 text-xs text-slate-300`
  dengan titik hijau berkedip (seperti badge "Gratis & Lokal").
- Isi pill: `👤 <UV> pengunjung • <PV> kunjungan`.
- Nilai awal span: `…`.
- Angka diformat `toLocaleString('id-ID')` (pemisah titik) oleh skrip kecil di halaman: `MutationObserver` pada kedua span, mengubah teks mentah dari Busuanzi menjadi format lokal. Skrip mengabaikan teks yang bukan angka murni agar tidak merusak placeholder.
- Menghormati `prefers-reduced-motion` yang sudah ada (tidak ada animasi baru yang ditambahkan).

## Error handling

- Jika API lambat/mati: span tetap berisi `…`.
- Timeout 6 detik: jika isi masih `…`, sembunyikan seluruh pill (`hidden`) agar UI tidak menampilkan placeholder rusak.
- Jika fetch gagal (offline), skrip Busuanzi melempar ke `console.error`; timeout tetap berlaku sehingga pill menghilang.

## Testing / verifikasi manual

Repo tidak punya test framework (lihat `AGENTS.md`), sehingga verifikasi manual:

1. `python3 -m http.server 8000` → buka `http://localhost:8000/index.html`.
2. Pill tampil di footer; UV dan PV terisi dan terformat `1.234` (id-ID).
3. Cek viewport mobile (~375px) dan desktop: pill tidak melanggar layout footer.
4. Fallback: di DevTools, block request ke `cdn.busuanzi.cc` → reload → pill hilang setelah ~6 detik, tidak menyisakan `…`.
5. Refresh berulang: PV bertambah, UV stabil pada hari yang sama.

## Out of scope

- Counter di halaman game (`emoji_mahjong.html`, `emoji_solitaire.html`, `onet_emoji.html`).
- Statistik harian/grafik, cookie banner (Busuanzi tidak memakai cookie browser untuk penghitungan UV berbasis IP).
- Perubahan analytics lain (GitHub Insights tetap dipakai untuk pemilik repo).
