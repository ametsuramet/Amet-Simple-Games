# Design: Emoji Worm Zone

**Date:** 2026-10-07
**Status:** Approved (design sections + collision rule)
**Scope:** New file `emoji_worm.html`; kartu baru + perubahan grid di `index.html`; update daftar game di `AGENTS.md`.

## Goal

Game keempat koleksi: "Worm Zones"-like arcade di mana pemain mengendalikan worm untuk makan, tumbuh, dan membunuh bot-bot komputer — dengan karakter emoji buah/makanan, mengikuti tema wajib repo (single HTML file, vanilla JS + Tailwind CDN, emoji tanpa aset gambar, mobile-friendly).

## Constraints (dari AGENTS.md)

- Satu file HTML mandiri: inline CSS + JS, tanpa module/build tooling.
- Hanya CDN yang sudah dipakai repo: Tailwind (`cdn.tailwindcss.com`) + Google Fonts.
- Emoji = seluruh visual karakter/makanan (tanpa file gambar).
- Mobile-first: viewport meta, `touch-action: none` di area game, `user-select: none`, `-webkit-tap-highlight-color: transparent`.
- Tanpa komentar di kode. Tanpa audio.
- Mekanik inti (disetujui): makan & tumbuh, bunuh lawan, boost memakan panjang, power-up khusus.

## Collision rules (disetujui: ala WormZones asli)

1. **Kepala menabrak badan worm lain → pemilik kepala MATI.** (Bukan sebaliknya: menyilang jalan lawan adalah cara membunuh — kepala lawan yang menabrak badanmu.)
2. **Kematian karena silang:** korban meledak jadi makanan (1 makanan per segmen, emoji = karakter korban). Pemilik badan yang disilang dapat `kill` + skor (5 + panjang korban/10). Makanan bisa dimakan siapa pun.
3. **Head-on (kepala vs kepala):** yang panjangnya lebih kecil mati; beda panjang ≤10% → keduanya mati.
4. **Border arena:** kepala menyentuh border → mati (tanpa jatuh makanan).
5. **Badan sendiri diperbolehkan ditembus** (tidak ada self-death), seperti WormZones asli.

## Arena, kamera, fisika

- Arena 3200×3200 px, border terlihat (garis putus-putus neon), latar kisi titik gelap sesuai tema halaman.
- Kamera mengikuti pemain (lerp), zoom menyesuaikan panjang: skala 1.0 → 0.6 saat sangat panjang, di-clamp agar view tidak jauh keluar border.
- Kecepatan dasar 160 px/s, turn-rate dibatasi ±3.2 rad/s; worm sangat panjang sedikit lebih lambat & kurang responsif (clamp kecepatan minimal 70%).
- Segmen: kepala r15, badan r11, jarak antar titik ~13px. Representasi: array titik path (head ditambahkan saat bergerak, titik terbuang saat panjang berkurang).

## Makanan, tumbuh, boost, power-up

- **Makanan biasa:** ~150 tersebar, konstan (respawn saat dimakan). Buah/makanan emoji: 🍓🍌🍉🍇🍊🍑🍩🍪. Makan → +1 segmen, +1 skor.
- **Boost:** tahan tombol → kecepatan ×2.1, −1 segmen/0.3 detik, makanan senilai setengah dijatuhkan di belakang. Boost hanya aktif bila panjang >10 segmen; saat ≤10 segmen tombol boost tidak berpengaruh.
- **Power-up** spawn tiap 8–15 detik, maks 4 hidup bersamaan:
  - ⭐ +8 segmen, +10 skor.
  - 👻 Ghost 5 detik: menembus badan worm lain (border & head-on tetap mematikan).
  - 🧲 Magnet 5 detik: menarik makanan dalam radius 160px.
- Efek power-up ditampilkan sebagai ikon + sisa waktu di HUD; durasi pakai detik nyata (akumulasi `dt`).

## Karakter & pilih worm

- Menu start menampilkan grid 12 emoji pilihan: 🍓 🍩 🍉 🍒 🍔 🍑 🥝 🍕 🌮 🍪 🧁 🍭.
- Kepala worm = emoji terpilih ukuran penuh; badan = lingkaran berwarna sesuai karakter dengan emoji sama tiap 4 segmen (opacity menurun ke ekor).
- Bot memakai emoji lain dari daftar yang sama (tidak bentrok dengan pemain).

## AI bot

- 6 bot aktif (rentang 5–7), respawn tiap 5 detik bila mati (mulai 10 segmen) agar arena selalu ramai.
- State machine, keputusan dihitung tiap 0.4 detik (bukan tiap frame): `MAKAN` (ke makanan terdekat) → `BURU` (menyilang worm lebih kecil yang lewat) → `KABUR` (menjauh dari badan worm lebih besar & border).
- Personalitas: 2 agresif (sering BURU, mau boost mengejar), 2 rakus (prioritas makan, boost ke makanan jauh), 2 penakut (jarang BURU, kabur saat ada yang besar <250px).
- Bot hidup di batas arena: state KABUR dipakai juga saat mendekati border, dan bot memutuskan arah balik sebelum kena border (tidak sengaja mati di border ≤10% dari kematian bot).

## Kontrol

- **Mobile:** joystick sentuh mengambang (muncul di titik sentuh, area kiri layar) + tombol boost bulat kanan-bawah (`touch-action: none`).
- **Desktop:** WASD/panah = arah, tahan `Spasi` = boost.
- Arah input = target angle; worm berbelok menuju target dengan turn-rate dibatasi (bukan snap).

## UI (overlay HTML/Tailwind di atas 1 canvas)

- **Menu:** judul, grid pilih emoji (default 🍓), tombol **Main**, aturan main 3 baris.
- **HUD:** top-left: skor, panjang (segmen), kill; top-right: ikon power-up aktif + sisa waktu; bottom-right: minimap 140×140 (dot pemain = hijau, bot = abu, border arena); atas minimap: mini leaderboard 5 besar (termasuk pemain) berdasarkan panjang.
- **Game Over:** "Kalah!", penyebab kematian (badan worm X / border / head-on), statistik: skor akhir, panjang, kill, waktu bertahan; tombol **Main Lagi** + **Menu**.
- Semua state (`menu` | `playing` | `gameover`) dikelola sebagai overlay div; canvas hanya menggambar arena.

## Performa & error handling

- Satu `<canvas>` 2D, satu `requestAnimationFrame`, `dt` di-clamp ≤50ms (anti lompat setelah tab background).
- Loop berhenti otomatis saat tab tersembunyi (`visibilitychange` → pause), lanjut saat kembali.
- Tidak ada panggilan API jaringan apa pun (game sepenuhnya offline setelah halaman termuat).
- Skala objek: 7 worm × ~100 segmen + ~150 makanan → cek tabrakan brute-force (≤ ~10k operasi/frame) masih aman tanpa spatial grid; bila melebihi, batasi panjang maks worm (500 segmen).

## Integrasi

- `emoji_worm.html` dihubungkan dari kartu ke-4 di `index.html` (URL absolut `https://ametsuramet.github.io/Amet-Simple-Games/emoji_worm.html`, konsisten kartu lain); grid diubah menjadi 2×2 di desktop (`lg:grid-cols-2`).
- `AGENTS.md`: daftar game (Overview) ditambah `emoji_worm.html`, "three" → "four".

## Verification (manual — repo tanpa test framework)

1. Buka via `python3 -m http.server 8000` (juga untuk emulasi mobile 375×667 + touch).
2. Menu: pilih emoji berbeda → worm pemain memakai emoji itu.
3. Desktop: WASD/panah belok halus, Spasi boost memangkas panjang & menjatuhkan makanan.
4. Touch: joystick muncul di titik sentuh, arah benar; tombol boost bereaksi; tidak ada scroll/zoom halaman (touch-action).
5. Bunuh bot: silang jalur bot → bot meledak jadi makanan, kill +1, skor naik.
6. Mati: kepala ke badan bot / border / head-on → layar Game Over dengan penyebab & statistik benar; Main Lagi & Menu berfungsi.
7. Power-up: ⭐ bertambah panjang, 👻 menembus badan, 🧲 menarik makanan; ikon HUD hilang saat durasi habis.
8. Bot: tetap ada ~6 bot (respawn), tidak mati massal di border, memburu/makan/kabur terlihat.
9. Performa: FPS stabil (rata-rata ≥55 selama 60 detik main) di Chrome desktop & emulasi mobile.
10. Kartu ke-4 tampil 2×2 di `index.html`, link membuka game; `AGENTS.md` ter-update.

## Out of scope

- Multiplayer online, akun, leaderboard global, penyimpanan skor (localStorage tidak dipakai).
- Audio/musik, efek partikel kompleks, iklan.
- Pilihan jumlah bot/difficulty di UI (dikonstantakan: 6 bot campuran).
- Game modes lain (Infinity/Treasure seperti WormZones asli).
