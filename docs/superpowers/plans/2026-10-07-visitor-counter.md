# Visitor Counter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Menambahkan counter pengunjung global (Busuanzi) ke footer `index.html`.

**Architecture:** Muat skrip Busuanzi resmi (`cdn.busuanzi.cc`) via tag `<script defer>` di `<head>`, letakkan dua `<span id="busuanzi_site_uv">` / `<span id="busuanzi_site_pv">` di dalam pill footer, lalu format angka mentah ke locale `id-ID` dengan `MutationObserver` + sembunyikan pill bila API tidak merespons dalam 6 detik.

**Tech Stack:** Vanilla JS inline, Tailwind CDN (kelas yang sudah ada), Busuanzi `3.6.9`. Tanpa build tools, tanpa test framework (lihat `AGENTS.md` — verifikasi manual).

**Spec:** `docs/superpowers/specs/2026-10-07-visitor-counter-design.md`

**Context untuk engineer:** Semua perubahan di SATU file: `index.html` (landing page). Jangan sentuh ketiga file game. Repo tidak punya `package.json`, test, atau lint — jangan menambahkannya. Jangan menambah komentar di kode (kebiasaan repo).

---

### Task 1: Muat skrip Busuanzi di `<head>`

**Files:**
- Modify: `index.html:17-20`

- [ ] **Step 1: Sisipkan tag skrip setelah link Google Fonts**

Konteks saat ini (baris 17-20):

```html
  <script src="https://cdn.tailwindcss.com"></script>
  <link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&family=Fredoka+One&display=swap"
    rel="stylesheet" />
   <base href="https://ametsuramet.github.io/Amet-Simple-Games" />
```

Ubah menjadi:

```html
  <script src="https://cdn.tailwindcss.com"></script>
  <link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&family=Fredoka+One&display=swap"
    rel="stylesheet" />
  <script defer src="https://cdn.busuanzi.cc/busuanzi/3.6.9/busuanzi.min.js"></script>
   <base href="https://ametsuramet.github.io/Amet-Simple-Games" />
```

- [ ] **Step 2: Verifikasi skrip bisa diambil**

Run:
```bash
curl -sSL --max-time 15 "https://cdn.busuanzi.cc/busuanzi/3.6.9/busuanzi.min.js" | head -c 200
```
Expected: diawali `(()=>{if(window.busuanziRequestSent)return;` (bukan halaman HTML/404).

---

### Task 2: Pill counter di footer

**Files:**
- Modify: `index.html` (blok `<footer>`, sekitar baris 345-354)

- [ ] **Step 1: Ganti kolom kiri footer**

Konteks saat ini:

```html
    <div class="mx-auto flex max-w-6xl flex-col items-center justify-between gap-3 text-center sm:flex-row sm:text-left">
      <p class="text-xs text-slate-500 sm:text-sm">
        Dibuat untuk koleksi game pribadi • HTML5 • Statis • Tanpa build
      </p>
      <p class="text-xs text-slate-600 sm:text-sm">
        © <span id="year"></span> Simple Games Collection by AMET SURAMET
      </p>
    </div>
```

Ganti `<p>` pertama dengan wrapper berisi pill + paragraf yang sama:

```html
    <div class="mx-auto flex max-w-6xl flex-col items-center justify-between gap-3 text-center sm:flex-row sm:text-left">
      <div class="flex flex-col items-center gap-2 sm:items-start">
        <span id="visitor-counter"
          class="inline-flex items-center gap-1.5 rounded-full bg-slate-50/10 px-2.5 py-1 text-xs font-semibold text-slate-200 ring-1 ring-inset ring-white/15">
          <span class="h-1.5 w-1.5 rounded-full bg-emerald-400"></span>
          👤 <span id="busuanzi_site_uv">…</span> pengunjung • <span id="busuanzi_site_pv">…</span> kunjungan
        </span>
        <p class="text-xs text-slate-500 sm:text-sm">
          Dibuat untuk koleksi game pribadi • HTML5 • Statis • Tanpa build
        </p>
      </div>
      <p class="text-xs text-slate-600 sm:text-sm">
        © <span id="year"></span> Simple Games Collection by AMET SURAMET
      </p>
    </div>
```

Catatan: ID `busuanzi_site_uv` / `busuanzi_site_pv` wajib persis — skrip Busuanzi mengisi elemen berdasarkan kunci JSON responsnya (`busuanzi_site_uv`, `busuanzi_site_pv`, dll, sudah diverifikasi terhadap API live).

---

### Task 3: Format angka + fallback timeout

**Files:**
- Modify: `index.html` (blok `<script>` sebelum `</body>`, sekitar baris 356-358)

- [ ] **Step 1: Ganti isi blok skrip**

Konteks saat ini:

```html
  <script>
    document.getElementById("year").textContent = new Date().getFullYear();
  </script>
```

Ganti seluruh isi menjadi:

```html
  <script>
    document.getElementById("year").textContent = new Date().getFullYear();

    const counterPill = document.getElementById("visitor-counter");
    const counterNums = [
      document.getElementById("busuanzi_site_uv"),
      document.getElementById("busuanzi_site_pv"),
    ];

    const formatCounter = (el) => {
      const raw = el.textContent.trim();
      if (!/^\d+$/.test(raw)) return false;
      el.textContent = Number(raw).toLocaleString("id-ID");
      return true;
    };

    const observer = new MutationObserver(() => {
      const ok = counterNums.map(formatCounter).every(Boolean);
      if (ok) counterPill.hidden = false;
    });

    counterNums.forEach((el) =>
      observer.observe(el, { childList: true, characterData: true, subtree: true })
    );

    setTimeout(() => {
      if (counterNums.every((el) => el.textContent.trim() === "…"))
        counterPill.hidden = true;
    }, 6000);
  </script>
```

Logika yang harus dipahami engineer (jangan diubah sembarangan):

- `formatCounter` hanya mengubah teks yang murni digit (`/^\d+$/`); placeholder `…` dan angka yang sudah diformat (`1.234`) diabaikan → tidak ada loop tak terbatas dengan `MutationObserver`.
- `counterNums.map(formatCounter)` (bukan `.every(formatCounter)`) agar kedua span tetap diproses walau yang pertama belum terisi (`.every` berhenti di elemen falsy pertama).
- Pill terlihat default dengan `…`; `setTimeout` 6 detik menyembunyikan (`hidden` attribute → Tailwind preflight `display:none !important`) hanya bila keduanya masih `…`; observer menampilkan lagi (`hidden = false`) begitu angka datang — mencakup skenario API datang terlambat setelah timeout.
- Tidak ada animasi baru (sesuai `prefers-reduced-motion` dan spec).

- [ ] **Step 2: Cek sintaks JS**

Run:
```bash
python3 - <<'EOF'
import re, sys
html = open('index.html').read()
m = re.search(r'<script>\n(.*?)\n  </script>', html, re.S)
assert m, 'inline script not found'
open('/tmp/inline.js','w').write(m.group(1))
EOF
node --check /tmp/inline.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`

---

### Task 4: Verifikasi manual di browser

**Files:**
- Test: `index.html` (dibuka di browser)

Repo tidak punya test framework; verifikasi manual. Gunakan tool browser (Playwright) atau `open` di macOS.

- [ ] **Step 1: Jalankan server statis**

Run:
```bash
python3 -m http.server 8000
```
Expected: `Serving HTTP on 0.0.0.0 port 8000 ...`

- [ ] **Step 2: Cek render & pengisian angka**

Buka `http://localhost:8000/index.html`.

Expected:
- Pill `👤 … pengunjung • … kunjungan` tampil di footer (state loading awal).
- Setelah skrip Busuanzi merespons: kedua angka terisi dan terformat `id-ID` (mis. `1`, `2` — server lokal dihitung sebagai site terpisah `localhost`, jadi angkanya kecil; angka produksi GitHub Pages mengikuti setelah deploy).
- Tidak ada error di console (khususnya bukan CORS / 404 pada `cdn.busuanzi.cc`).
- Refresh sekali lagi: PV bertambah, UV tidak berubah (satu hari, IP sama).

- [ ] **Step 3: Cek layout mobile dan desktop**

Resize viewport ke 375×667 lalu 1280×800.

Expected: pill tidak meluber, footer tetap `sm:flex-row` rapi di desktop, stacked & center di mobile.

- [ ] **Step 4: Cek fallback API mati**

Di DevTools → Network → block `cdn.busuanzi.cc/*`, reload halaman.

Expected: pill tampil `…` selama ±6 detik, lalu hilang sepenuhnya (tidak menyisakan placeholder). Lalu unblock + reload: pill tampil kembali berisi angka.

- [ ] **Step 5: Hentikan server**

Run (Ctrl+C di terminal server, atau):
```bash
lsof -ti:8000 | xargs kill 2>/dev/null; true
```

---

### Task 5: Commit

- [ ] **Step 1: Review diff**

Run:
```bash
git diff index.html
```
Expected: hanya bertambah tag skrip di `<head>`, pill di footer, dan blok skrip di akhir `<body>`. Tidak ada perubahan di file game lain.

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "Add global visitor counter to landing page"
```
Expected: commit sukses, `git status` bersih.

---

## Self-Review (sudah dijalankan)

- **Spec coverage:** script tag (Task 1), pill + dua span ID persis spec (Task 2), format `id-ID` via MutationObserver (Task 3), timeout 6 detik + un-hide (Task 3), verifikasi 5 poin spec (Task 4), scope hanya `index.html` (semua task), commit (Task 5).
- **Placeholder scan:** tidak ada TBD/TODO; semua langkah berisi kode/perintah konkret.
- **Konsistensi:** ID `visitor-counter`, `busuanzi_site_uv`, `busuanzi_site_pv` konsisten di Task 2 dan Task 3; nama fungsi `formatCounter` konsisten.
