# Emoji Worm Zone Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Membangun `emoji_worm.html` — arcade worm ala WormZones dengan karakter emoji, 6 bot AI, boost, power-up, minimap, lalu mendaftarkannya di `index.html` + `AGENTS.md`.

**Architecture:** Satu file HTML mandiri (vanilla JS + Tailwind CDN, tanpa build). Satu `<canvas>` 2D dengan game loop `requestAnimationFrame` (dt di-clamp 50ms), state machine `menu | playing | gameover`, overlay HTML/Tailwind di atas canvas untuk menu/HUD/game-over. Worm direpresentasikan sebagai array titik trail (head + badan), bot AI memakai state machine `MAKAN | BURU | KABUR`.

**Tech Stack:** HTML5 Canvas 2D, vanilla JS, Tailwind CDN (`https://cdn.tailwindcss.com`), Google Fonts (Nunito + Fredoka One). Tanpa dependensi lain, tanpa panggilan API jaringan.

**Spec:** `docs/superpowers/specs/2026-10-07-emoji-worm-design.md`

**Catatan repo (dari AGENTS.md):** Tidak ada test framework/linter/CI — verifikasi manual via browser (`open <file>` atau `python3 -m http.server 8000`). Jangan tambahkan komentar di kode. Landing page kartu memakai URL absolut `https://ametsuramet.github.io/Amet-Simple-Games/<file>`.

**Git:** Buat branch `feature/emoji-worm` sebelum Task 1 (`git checkout -b feature/emoji-worm`). Merge fast-forward ke `main` + push setelah semua task selesai dan diverifikasi (sama seperti alur visitor-counter yang sudah disetujui). Commit per task memakai gaya repo: imperative, singkat.

---

### Task 1: Skeleton File + Game Loop + Worm Bergerak

**Files:**
- Create: `emoji_worm.html`

- [ ] **Step 1: Buat branch kerja**

Run: `git checkout -b feature/emoji-worm`
Expected: `Switched to a new branch 'feature/emoji-worm'`

- [ ] **Step 2: Tulis file lengkap `emoji_worm.html`**

```html
<!DOCTYPE html>
<html lang="id">

<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>Emoji Worm Zone - Simple Games Collection</title>
  <meta name="description" content="Emoji Worm Zone - arcade worm bertema emoji: makan, tumbuh, kalahkan bot AI." />
  <meta name="theme-color" content="#0f172a" />
  <script src="https://cdn.tailwindcss.com"></script>
  <link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&family=Fredoka+One&display=swap" rel="stylesheet" />
  <style>
    :root {
      color-scheme: dark;
    }

    html,
    body {
      margin: 0;
      height: 100%;
      overflow: hidden;
      background-color: #0f172a;
      touch-action: none;
      -webkit-tap-highlight-color: transparent;
      user-select: none;
      -webkit-user-select: none;
      font-family: "Nunito", system-ui, sans-serif;
      color: #f8fafc;
    }

    #game {
      position: fixed;
      inset: 0;
      display: block;
    }

    .overlay {
      position: fixed;
      inset: 0;
      z-index: 40;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      gap: 1rem;
      padding: 1rem;
      text-align: center;
      background-color: rgba(15, 23, 42, 0.93);
    }

    .title-display {
      font-family: "Fredoka One", cursive;
      letter-spacing: -0.01em;
      text-shadow: 0px 10px 60px rgba(59, 130, 246, 0.35),
        2px 4px 12px rgba(2, 6, 23, 0.45);
    }

    .emoji-pick {
      font-size: 1.75rem;
      width: 3rem;
      height: 3rem;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 0.75rem;
      background-color: rgba(30, 41, 59, 0.85);
      border: 1px solid rgba(255, 255, 255, 0.1);
      cursor: pointer;
    }

    .emoji-pick.sel {
      border-color: #38bdf8;
      box-shadow: 0 0 0 2px #38bdf8;
      background-color: rgba(14, 165, 233, 0.25);
    }

    #joystick {
      position: fixed;
      z-index: 30;
      width: 120px;
      height: 120px;
      border-radius: 9999px;
      background-color: rgba(148, 163, 184, 0.15);
      border: 2px solid rgba(148, 163, 184, 0.45);
      display: none;
      pointer-events: none;
    }

    #stick {
      position: absolute;
      left: 35px;
      top: 35px;
      width: 50px;
      height: 50px;
      border-radius: 9999px;
      background-color: rgba(56, 189, 248, 0.55);
      border: 2px solid rgba(255, 255, 255, 0.65);
    }
  </style>
</head>

<body>
  <canvas id="game"></canvas>

  <div id="hud" class="hidden fixed left-3 top-3 z-20 flex flex-col gap-1 text-sm font-bold">
    <div class="rounded-full bg-slate-950/70 px-3 py-1 ring-1 ring-inset ring-white/15">⭐ <span id="hudScore">0</span></div>
    <div class="rounded-full bg-slate-950/70 px-3 py-1 ring-1 ring-inset ring-white/15">📏 <span id="hudLen">10</span></div>
    <div class="rounded-full bg-slate-950/70 px-3 py-1 ring-1 ring-inset ring-white/15">💀 <span id="hudKills">0</span></div>
  </div>

  <div id="powerhud" class="hidden fixed right-3 top-3 z-20 flex flex-col items-end gap-1 text-sm font-bold"></div>

  <div id="board" class="hidden fixed right-3 z-20 w-36 rounded-xl bg-slate-950/70 p-2 text-xs ring-1 ring-inset ring-white/15" style="bottom: 6.5rem">
    <div class="mb-1 font-bold uppercase tracking-wider text-slate-400">Leaderboard</div>
    <ul id="boardList" class="flex flex-col gap-0.5"></ul>
  </div>

  <button id="boostBtn" class="hidden fixed bottom-6 right-6 z-30 h-16 w-16 rounded-full bg-sky-500/80 text-2xl ring-2 ring-inset ring-sky-300/60 active:scale-95">🚀</button>

  <div id="menu" class="overlay">
    <h1 class="title-display text-4xl text-sky-400 sm:text-5xl">🐛 Emoji Worm Zone</h1>
    <p class="max-w-md text-sm text-slate-300 sm:text-base">
      Makan buah, tumbuh makin panjang, silang jalan lawan sampai mereka menabrak badanmu.
      Hindari border dan kepala worm lain!
    </p>
    <div id="emojiPick" class="grid grid-cols-6 gap-2"></div>
    <button id="startBtn" class="rounded-full bg-sky-500 px-10 py-3 text-lg font-extrabold shadow-lg active:scale-95">Main</button>
    <p class="text-xs text-slate-400">
      WASD / panah + Spasi (boost) di desktop &bull; Joystick + 🚀 di HP
    </p>
  </div>

  <div id="gameover" class="overlay hidden">
    <h1 class="title-display text-4xl text-rose-400 sm:text-5xl">Kalah!</h1>
    <p id="goCause" class="text-slate-300"></p>
    <div class="grid grid-cols-2 gap-2 text-sm">
      <div class="rounded-xl bg-slate-900/80 px-4 py-2 ring-1 ring-inset ring-white/10">⭐ Skor: <span id="goScore">0</span></div>
      <div class="rounded-xl bg-slate-900/80 px-4 py-2 ring-1 ring-inset ring-white/10">📏 Panjang: <span id="goLen">0</span></div>
      <div class="rounded-xl bg-slate-900/80 px-4 py-2 ring-1 ring-inset ring-white/10">💀 Kill: <span id="goKills">0</span></div>
      <div class="rounded-xl bg-slate-900/80 px-4 py-2 ring-1 ring-inset ring-white/10">⏱️ Waktu: <span id="goTime">0</span> d</div>
    </div>
    <div class="flex gap-3">
      <button id="retryBtn" class="rounded-full bg-sky-500 px-6 py-2.5 font-bold shadow-lg active:scale-95">Main Lagi</button>
      <button id="homeBtn" class="rounded-full bg-slate-700 px-6 py-2.5 font-bold ring-1 ring-inset ring-white/15 active:scale-95">Menu</button>
    </div>
  </div>

  <div id="joystick">
    <div id="stick"></div>
  </div>

  <script>
    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");

    const W = 3200;
    const SEG_GAP = 13;
    const HEAD_R = 15;
    const BODY_R = 11;
    const BASE_SPEED = 160;
    const TURN_RATE = 3.2;
    const FOOD_COUNT = 150;
    const BOT_COUNT = 6;
    const MAX_SEGS = 500;
    const BOT_PERMS = ["agresif", "rakus", "penakut", "penakut", "rakus", "agresif"];

    const PICK_EMOJIS = ["🍓", "🍩", "🍉", "🍒", "🍔", "🍑", "🥝", "🍕", "🌮", "🍪", "🧁", "🍭"];
    const FOOD_EMOJIS = ["🍓", "🍌", "🍉", "🍇", "🍊", "🍑", "🍩"];
    const BOT_EMOJIS = ["🍒", "🥝", "🍕", "🌮", "🍪", "🧁", "🍭", "🍇"];
    const POWER_TYPES = ["star", "ghost", "magnet"];
    const POWER_EMOJI = { star: "⭐", ghost: "👻", magnet: "🧲" };
    const EMOJI_COLORS = {
      "🍓": "#f43f5e", "🍩": "#f59e0b", "🍉": "#ef4444", "🍇": "#a855f7",
      "🍒": "#ef4444", "🍔": "#f97316", "🍑": "#fb923c", "🥝": "#84cc16",
      "🍕": "#f97316", "🌮": "#eab308", "🍪": "#d97706", "🧁": "#f472b6",
      "🍭": "#ec4899", "🍌": "#facc15", "🍊": "#fb923c"
    };

    let state = "menu";
    let selected = "🍓";
    let player = null;
    let bots = [];
    let foods = [];
    let powers = [];
    let cam = { x: W / 2, y: W / 2, zoom: 1 };
    let keys = {};
    let boostHeld = false;
    let joy = { active: false, id: null, ox: 0, oy: 0, angle: null };
    let playTime = 0;
    let hudClock = 0;
    let boardClock = 0;
    let powerTimer = 0;

    let dpr = 1;
    let vw = 0;
    let vh = 0;

    const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
    const emojiColor = (e) => EMOJI_COLORS[e] || "#38bdf8";

    function resize() {
      dpr = Math.min(window.devicePixelRatio || 1, 2);
      vw = window.innerWidth;
      vh = window.innerHeight;
      canvas.width = Math.floor(vw * dpr);
      canvas.height = Math.floor(vh * dpr);
      canvas.style.width = vw + "px";
      canvas.style.height = vh + "px";
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }
    window.addEventListener("resize", resize);
    resize();

    function show(id, on) {
      document.getElementById(id).classList.toggle("hidden", !on);
    }

    const emojiPickEl = document.getElementById("emojiPick");
    PICK_EMOJIS.forEach((e) => {
      const b = document.createElement("button");
      b.className = "emoji-pick" + (e === selected ? " sel" : "");
      b.textContent = e;
      b.onclick = () => {
        selected = e;
        Array.from(emojiPickEl.children).forEach((c) => c.classList.remove("sel"));
        b.classList.add("sel");
      };
      emojiPickEl.appendChild(b);
    });

    function makeWorm(x, y, emoji, personality) {
      const a = Math.random() * Math.PI * 2;
      return {
        x, y, angle: a, targetAngle: a,
        vx: 0, vy: 0,
        trail: [{ x, y }],
        segs: 10, score: 0, kills: 0,
        emoji, personality: personality || "",
        isPlayer: false, alive: true,
        boost: false, boostAcc: 0,
        ghost: 0, magnet: 0,
        distAcc: 0, decideT: 0,
        respawnT: 0, cause: ""
      };
    }

    function seedTrail(w) {
      w.trail = [];
      for (let i = 0; i < 10; i++) {
        w.trail.push({
          x: w.x - Math.cos(w.angle) * SEG_GAP * i,
          y: w.y - Math.sin(w.angle) * SEG_GAP * i
        });
      }
    }

    function stepWorm(w, dt) {
      let da = w.targetAngle - w.angle;
      while (da > Math.PI) da -= Math.PI * 2;
      while (da < -Math.PI) da += Math.PI * 2;
      const slow = Math.max(0.7, 1 - w.segs / 3000);
      w.angle += clamp(da, -TURN_RATE * slow * dt, TURN_RATE * slow * dt);
      const sp = BASE_SPEED * slow;
      w.vx = Math.cos(w.angle) * sp;
      w.vy = Math.sin(w.angle) * sp;
      w.x += w.vx * dt;
      w.y += w.vy * dt;
      w.distAcc += sp * dt;
      if (w.distAcc >= SEG_GAP) {
        w.distAcc = 0;
        w.trail.unshift({ x: w.x, y: w.y });
        const cap = Math.min(MAX_SEGS, Math.ceil(w.segs));
        while (w.trail.length > cap) w.trail.pop();
      }
    }

    function readInput() {
      let dx = 0;
      let dy = 0;
      if (keys["KeyA"] || keys["ArrowLeft"]) dx -= 1;
      if (keys["KeyD"] || keys["ArrowRight"]) dx += 1;
      if (keys["KeyW"] || keys["ArrowUp"]) dy -= 1;
      if (keys["KeyS"] || keys["ArrowDown"]) dy += 1;
      if (dx || dy) player.targetAngle = Math.atan2(dy, dx);
      else if (joy.angle !== null) player.targetAngle = joy.angle;
      player.boost = boostHeld;
    }

    window.addEventListener("keydown", (e) => {
      keys[e.code] = true;
      if (e.code.startsWith("Arrow")) e.preventDefault();
    });
    window.addEventListener("keyup", (e) => {
      keys[e.code] = false;
    });

    function startGame() {
      player = makeWorm(W / 2, W / 2, selected, "");
      player.isPlayer = true;
      seedTrail(player);
      bots = [];
      foods = [];
      powers = [];
      playTime = 0;
      hudClock = 0;
      boardClock = 0;
      cam = { x: player.x, y: player.y, zoom: 1 };
      state = "playing";
      show("menu", false);
      show("gameover", false);
      show("hud", true);
      show("powerhud", true);
      show("board", true);
      show("boostBtn", true);
      if (document.activeElement) document.activeElement.blur();
    }

    document.getElementById("startBtn").onclick = startGame;
    document.getElementById("retryBtn").onclick = startGame;
    document.getElementById("homeBtn").onclick = () => {
      state = "menu";
      show("menu", true);
      show("gameover", false);
      show("hud", false);
      show("powerhud", false);
      show("board", false);
      show("boostBtn", false);
    };

    function update(dt) {
      playTime += dt;
      readInput();
      stepWorm(player, dt);
      cam.zoom += (clamp(1 - player.segs / 1250, 0.6, 1) - cam.zoom) * Math.min(1, dt * 3);
      cam.x += (player.x - cam.x) * Math.min(1, dt * 8);
      cam.y += (player.y - cam.y) * Math.min(1, dt * 8);
      cam.x = clamp(cam.x, 0, W);
      cam.y = clamp(cam.y, 0, W);
    }

    function drawGrid() {
      const step = 64;
      const halfW = vw / 2 / cam.zoom;
      const halfH = vh / 2 / cam.zoom;
      const x0 = Math.max(0, Math.floor((cam.x - halfW) / step) * step);
      const x1 = Math.min(W, cam.x + halfW);
      const y0 = Math.max(0, Math.floor((cam.y - halfH) / step) * step);
      const y1 = Math.min(W, cam.y + halfH);
      ctx.fillStyle = "#1e293b";
      for (let x = x0; x <= x1; x += step) {
        for (let y = y0; y <= y1; y += step) {
          ctx.beginPath();
          ctx.arc(x, y, 2, 0, 7);
          ctx.fill();
        }
      }
    }

    function drawBorder() {
      ctx.fillStyle = "rgba(2, 6, 23, 0.55)";
      const m = 400;
      ctx.fillRect(-m, -m, W + m * 2, m);
      ctx.fillRect(-m, W, W + m * 2, m);
      ctx.fillRect(-m, 0, m, W);
      ctx.fillRect(W, 0, m, W);
      ctx.strokeStyle = "#38bdf8";
      ctx.lineWidth = 6;
      ctx.setLineDash([24, 18]);
      ctx.strokeRect(0, 0, W, W);
      ctx.setLineDash([]);
    }

    function drawWorm(w) {
      const alpha = w.ghost > 0 ? 0.5 : 1;
      const n = Math.min(w.trail.length, Math.min(MAX_SEGS, Math.ceil(w.segs)));
      ctx.globalAlpha = alpha;
      for (let i = n - 1; i >= 0; i--) {
        const p = w.trail[i];
        const t = 1 - i / n;
        ctx.beginPath();
        ctx.arc(p.x, p.y, BODY_R * (0.75 + 0.25 * t), 0, 7);
        ctx.fillStyle = emojiColor(w.emoji);
        ctx.fill();
        if (i > 0 && i % 5 === 0) {
          ctx.globalAlpha = alpha * (0.5 + 0.5 * t);
          ctx.font = "13px serif";
          ctx.textAlign = "center";
          ctx.textBaseline = "middle";
          ctx.fillText(w.emoji, p.x, p.y);
          ctx.globalAlpha = alpha;
        }
      }
      ctx.font = "28px serif";
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      ctx.fillText(w.emoji, w.x, w.y);
      ctx.globalAlpha = 1;
    }

    function drawWorms() {
      const all = [player].concat(bots).filter((w) => w && w.alive);
      for (const w of all) drawWorm(w);
    }

    function render() {
      ctx.fillStyle = "#0f172a";
      ctx.fillRect(0, 0, vw, vh);
      ctx.save();
      ctx.translate(vw / 2, vh / 2);
      ctx.scale(cam.zoom, cam.zoom);
      ctx.translate(-cam.x, -cam.y);
      drawGrid();
      drawBorder();
      drawWorms();
      ctx.restore();
    }

    let rafId = 0;
    let lastT = 0;

    function loop(t) {
      rafId = requestAnimationFrame(loop);
      if (!lastT) lastT = t;
      const dt = Math.min((t - lastT) / 1000, 0.05);
      lastT = t;
      if (state === "playing") update(dt);
      render();
    }
    rafId = requestAnimationFrame(loop);

    document.addEventListener("visibilitychange", () => {
      if (document.hidden) {
        cancelAnimationFrame(rafId);
        rafId = 0;
      } else if (!rafId) {
        lastT = 0;
        rafId = requestAnimationFrame(loop);
      }
    });
  </script>
</body>

</html>
```

- [ ] **Step 3: Verifikasi di browser**

Run: `open emoji_worm.html`
Expected: menu tampil dengan judul "🐛 Emoji Worm Zone", grid 12 emoji (2 baris, 🍓 ter-highlight), tombol Main, hint kontrol. Klik emoji lain → highlight pindah. Klik **Main** → overlay hilang; arena (kisi titik gelap + border biru putus-putus + area gelap di luar border) tampil dengan worm 🍓 di tengah yang bergerak lurus; WASD/panah membelokkan worm dengan halus; kamera mengikuti; zoom sedikit menjauh saat... (panjang masih konstan 10 di task ini, zoom ~0.992 — tetap). Tekan tab lalu kembali → tidak ada error console (loop berhenti/lanjut via visibilitychange). Dari menu, HUD/leaderboard/tombol boost tidak terlihat.

- [ ] **Step 4: Commit**

```bash
git add emoji_worm.html
git commit -m "Add Emoji Worm Zone skeleton with core movement"
```

### Task 2: Makanan, Pertumbuhan, HUD

**Files:**
- Modify: `emoji_worm.html`

- [ ] **Step 1: Tambahkan fungsi makanan (sisipkan setelah `seedTrail`)**

```js
    function spawnFood(x, y, val) {
      foods.push({
        x: x !== null ? x : 60 + Math.random() * (W - 120),
        y: y !== null ? y : 60 + Math.random() * (W - 120),
        val: val || 1,
        emoji: FOOD_EMOJIS[Math.floor(Math.random() * FOOD_EMOJIS.length)]
      });
    }

    function dropFood(x, y) {
      spawnFood(
        clamp(x + (Math.random() - 0.5) * 24, 40, W - 40),
        clamp(y + (Math.random() - 0.5) * 24, 40, W - 40),
        0.5
      );
    }

    function eatFood(w) {
      const cap = Math.min(MAX_SEGS, Math.ceil(w.segs));
      const headR = HEAD_R;
      for (let i = 0; i < foods.length; i++) {
        const f = foods[i];
        const dx = f.x - w.x;
        const dy = f.y - w.y;
        if (dx * dx + dy * dy <= (headR + 14) * (headR + 14)) {
          if (w.segs < MAX_SEGS) w.segs = Math.min(MAX_SEGS, w.segs + f.val);
          w.score += f.val >= 1 ? 1 : 0.5;
          const trailCap = Math.min(MAX_SEGS, Math.ceil(w.segs));
          if (w.trail.length < trailCap) w.trail.push({ ...w.trail[w.trail.length - 1] });
          foods.splice(i, 1);
          spawnFood(null, null, 1);
          return true;
        }
      }
      return false;
    }

    function updateHUD() {
      document.getElementById("hudScore").textContent = Math.floor(player.score);
      document.getElementById("hudLen").textContent = Math.ceil(player.segs);
      document.getElementById("hudKills").textContent = player.kills;
    }

    function drawFoods() {
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      for (const f of foods) {
        ctx.font = (f.val >= 1 ? "18px" : "13px") + " serif";
        ctx.fillText(f.emoji, f.x, f.y);
      }
    }
```

- [ ] **Step 2: Ganti `startGame` (isi makanan + HUD)**

Ganti fungsi `startGame` lama dengan:

```js
    function startGame() {
      player = makeWorm(W / 2, W / 2, selected, "");
      player.isPlayer = true;
      seedTrail(player);
      bots = [];
      foods = [];
      powers = [];
      for (let i = 0; i < FOOD_COUNT; i++) spawnFood(null, null, 1);
      playTime = 0;
      hudClock = 0;
      boardClock = 0;
      powerTimer = 8 + Math.random() * 7;
      cam = { x: player.x, y: player.y, zoom: 1 };
      state = "playing";
      show("menu", false);
      show("gameover", false);
      show("hud", true);
      show("powerhud", true);
      show("board", true);
      show("boostBtn", true);
      updateHUD();
      if (document.activeElement) document.activeElement.blur();
    }
```

- [ ] **Step 3: Ganti `update` (makan + HUD periodik)**

Ganti fungsi `update` lama dengan:

```js
    function update(dt) {
      playTime += dt;
      readInput();
      stepWorm(player, dt);
      eatFood(player);
      hudClock += dt;
      if (hudClock >= 0.2) {
        hudClock = 0;
        updateHUD();
      }
      cam.zoom += (clamp(1 - player.segs / 1250, 0.6, 1) - cam.zoom) * Math.min(1, dt * 3);
      cam.x += (player.x - cam.x) * Math.min(1, dt * 8);
      cam.y += (player.y - cam.y) * Math.min(1, dt * 8);
      cam.x = clamp(cam.x, 0, W);
      cam.y = clamp(cam.y, 0, W);
    }
```

- [ ] **Step 4: Ganti `render` (gambar makanan)**

Di dalam `render()`, ganti:

```js
      drawGrid();
      drawBorder();
      drawWorms();
      ctx.restore();
```

menjadi:

```js
      drawGrid();
      drawBorder();
      drawFoods();
      drawWorms();
      ctx.restore();
```

- [ ] **Step 5: Verifikasi di browser**

Run: `open emoji_worm.html` → klik **Main**.
Expected: 150 emoji makanan tersebar di arena; worm bergerak dan makan makanan yang disentuhnya (emoji hilang, spawn baru muncul di lokasi acak); ⭐ Skor naik per makanan, 📏 Panjang naik dari 10 dan terus bertambah; HUD kiri-atas ter-update (score naik saat makan). Jika panjang > 1250/1250..., zoom mundur bertahap. Jangan ada error console.

- [ ] **Step 6: Commit**

```bash
git add emoji_worm.html
git commit -m "Add food, growth and HUD to Emoji Worm Zone"
```

### Task 3: Boost + Kontrol Sentuh (Joystick & Tombol Boost)

**Files:**
- Modify: `emoji_worm.html`

- [ ] **Step 1: Boost di `stepWorm`**

Ganti isi `stepWorm` (dari baris `const slow = ...` sampai akhir fungsi) dengan:

```js
    function stepWorm(w, dt) {
      let da = w.targetAngle - w.angle;
      while (da > Math.PI) da -= Math.PI * 2;
      while (da < -Math.PI) da += Math.PI * 2;
      const slow = Math.max(0.7, 1 - w.segs / 3000);
      w.angle += clamp(da, -TURN_RATE * slow * dt, TURN_RATE * slow * dt);
      const canBoost = w.boost && w.segs > 10;
      const sp = BASE_SPEED * slow * (canBoost ? 2.1 : 1);
      w.vx = Math.cos(w.angle) * sp;
      w.vy = Math.sin(w.angle) * sp;
      w.x += w.vx * dt;
      w.y += w.vy * dt;
      w.distAcc += sp * dt;
      if (canBoost) {
        w.boostAcc += dt;
        while (w.boostAcc >= 0.3) {
          w.boostAcc -= 0.3;
          w.segs -= 1;
          dropFood(w.x, w.y);
          if (w.segs <= 10) {
            w.segs = 10;
            break;
          }
        }
      } else {
        w.boostAcc = 0;
      }
      if (w.distAcc >= SEG_GAP) {
        w.distAcc = 0;
        w.trail.unshift({ x: w.x, y: w.y });
        const cap = Math.min(MAX_SEGS, Math.ceil(w.segs));
        while (w.trail.length > cap) w.trail.pop();
      }
    }
```

- [ ] **Step 2: Spasi sebagai boost (keyboard)**

Ganti listener `keydown` lama dengan:

```js
    window.addEventListener("keydown", (e) => {
      keys[e.code] = true;
      if (e.code.startsWith("Arrow")) e.preventDefault();
      if (e.code === "Space") {
        e.preventDefault();
        boostHeld = true;
      }
    });
    window.addEventListener("keyup", (e) => {
      keys[e.code] = false;
      if (e.code === "Space") boostHeld = false;
    });
    window.addEventListener("blur", () => {
      keys = {};
      boostHeld = false;
    });
```

- [ ] **Step 3: Joystick pointer + tombol boost DOM**

Sisipkan setelah listener keyboard:

```js
    const joyEl = document.getElementById("joystick");
    const stickEl = document.getElementById("stick");
    const boostBtnEl = document.getElementById("boostBtn");

    window.addEventListener("pointerdown", (e) => {
      if (state !== "playing") return;
      if (e.pointerType === "mouse") return;
      if (e.target === boostBtnEl) return;
      if (e.clientX > vw * 0.6) return;
      joy.active = true;
      joy.id = e.pointerId;
      joy.ox = e.clientX;
      joy.oy = e.clientY;
      joy.angle = null;
      joyEl.style.left = e.clientX - 60 + "px";
      joyEl.style.top = e.clientY - 60 + "px";
      joyEl.style.display = "block";
      stickEl.style.left = "35px";
      stickEl.style.top = "35px";
    });

    window.addEventListener("pointermove", (e) => {
      if (!joy.active || e.pointerId !== joy.id) return;
      const dx = e.clientX - joy.ox;
      const dy = e.clientY - joy.oy;
      const d = Math.hypot(dx, dy);
      if (d > 8) joy.angle = Math.atan2(dy, dx);
      const cl = Math.min(d, 40);
      const nx = d ? (dx / d) * cl : 0;
      const ny = d ? (dy / d) * cl : 0;
      stickEl.style.left = 35 + nx + "px";
      stickEl.style.top = 35 + ny + "px";
    });

    function endJoy(e) {
      if (!joy.active || e.pointerId !== joy.id) return;
      joy.active = false;
      joy.id = null;
      joy.angle = null;
      joyEl.style.display = "none";
    }
    window.addEventListener("pointerup", endJoy);
    window.addEventListener("pointercancel", endJoy);

    boostBtnEl.addEventListener("pointerdown", (e) => {
      e.preventDefault();
      boostHeld = true;
    });
    window.addEventListener("pointerup", () => {
      boostHeld = false;
    });
    window.addEventListener("pointercancel", () => {
      boostHeld = false;
    });
```

Catatan: `joy.angle` di-reset ke `null` saat idle supaya worm kembali lurus arah terakhir... tidak — arah terakhir dipertahankan oleh `targetAngle` karena `readInput` hanya mengubah `targetAngle` saat ada input; saat joystick dilepas worm tetap pada `targetAngle` terakhir (benar, sesuai spec: heading berhenti berubah, bukan reset).

- [ ] **Step 4: Verifikasi di browser**

Run: `open emoji_worm.html` → **Main**.
Expected (desktop): tekan **Spasi** → worm melaju ~2× lebih cepat, makanan jatuh di belakangnya tiap ~0.3s, panjang berkurang; lepas Spasi → normal kembali; panjang tidak pernah turun di bawah 10 saat boost. Panah/keyboard tetap bekerja. Tab out → `keys` di-reset (tidak ada tombol nyangkut).
Expected (mobile / DevTools device mode): sentuh sisi kiri layar (≤60% lebar) → joystick mengambang muncul di titik sentuh, stick mengikuti, worm belok; sentuh kanan (di luar tombol 🚀) → joystick tidak muncul; tahan 🚀 → boost aktif, lepas → berhenti. Jangan ada error console.

- [ ] **Step 5: Commit**

```bash
git add emoji_worm.html
git commit -m "Add boost and touch controls to Emoji Worm Zone"
```

### Task 4: Tabrakan, Kematian, Game Over (Aturan WormZones Autentik)

**Files:**
- Modify: `emoji_worm.html`

- [ ] **Step 1: Tambahkan fungsi kematian & game over (sisipkan setelah `updateHUD`)**

```js
    function killWorm(w, killer, cause, credit) {
      if (!w.alive) return;
      w.alive = false;
      w.cause = cause;
      w.respawnT = 5;
      if (cause !== "border") {
        const cap = Math.min(w.trail.length, Math.min(MAX_SEGS, Math.ceil(w.segs)));
        for (let i = 0; i < cap; i++) {
          spawnFood(w.trail[i].x, w.trail[i].y, 1);
        }
      }
      if (killer && credit && killer !== w && killer.alive) {
        killer.kills += 1;
        killer.score += 5 + Math.floor(w.segs / 10);
      }
      if (w.isPlayer) endGame(cause, killer);
    }

    function endGame(cause, killer) {
      if (state !== "playing") return;
      state = "gameover";
      document.getElementById("goCause").textContent =
        cause === "border"
          ? "Kamu menabrak border arena!"
          : cause === "headon"
            ? "Kepalamu bertabrakan dengan worm lain!"
            : killer
              ? "Kepalamu menabrak badan " + killer.emoji + "!"
              : "Kamu menabrak badan sendiri!";
      document.getElementById("goScore").textContent = Math.floor(player.score);
      document.getElementById("goLen").textContent = Math.ceil(player.segs);
      document.getElementById("goKills").textContent = player.kills;
      document.getElementById("goTime").textContent = playTime.toFixed(1);
      show("gameover", true);
      show("hud", false);
      show("powerhud", false);
      show("board", false);
      show("boostBtn", false);
      boostHeld = false;
      joy.active = false;
      joy.angle = null;
      document.getElementById("joystick").style.display = "none";
    }
```

Aturan credit (sesuai spec): kepala menabrak **badan** orang lain → pemilik kepala mati, **pemilik badan** di-credit (`killer` = pemilik badan, `credit = true`). Head-on ≤10% beda panjang → keduanya mati **tanpa** credit. Head-on >10% → yang lebih kecil mati, survivor (pemilik kepala yang menang) di-credit. Border → mati tanpa drop makanan & tanpa credit. Worm ghost menembus semua badan, tapi **head-on dan border tetap mematikan** untuk ghost (ghost tidak di-skip di loop head-on; ghost hanya skip deteksi badan).

- [ ] **Step 2: Tambahkan `checkCollisions` (sisipkan setelah `endGame`)**

```js
    function checkCollisions() {
      const all = [player].concat(bots).filter((w) => w && w.alive);
      const dead = new Map();
      for (const w of all) {
        if (w.x < 0 || w.y < 0 || w.x > W || w.y > W) {
          dead.set(w, { cause: "border", killer: null, credit: false });
          continue;
        }
        let headOn = false;
        for (const o of all) {
          if (o === w) continue;
          if (Math.hypot(o.x - w.x, o.y - w.y) <= HEAD_R * 2) {
            headOn = true;
            break;
          }
        }
        if (headOn) {
          dead.set(w, { cause: "headon", killer: null, credit: false });
          continue;
        }
        if (w.ghost > 0) continue;
        const hitR = HEAD_R + BODY_R - 6;
        for (const o of all) {
          if (o === w || o.ghost > 0) continue;
          const cap = Math.min(o.trail.length, Math.min(MAX_SEGS, Math.ceil(o.segs)));
          for (let i = 2; i < cap; i++) {
            const p = o.trail[i];
            const dx = p.x - w.x;
            const dy = p.y - w.y;
            if (dx * dx + dy * dy <= hitR * hitR) {
              dead.set(w, { cause: "body", killer: o, credit: true });
              break;
            }
          }
          if (dead.has(w)) break;
        }
      }
      const headOnDead = [...dead.entries()].filter(([, i]) => i.cause === "headon");
      for (const [w, info] of headOnDead) {
        const rivals = headOnDead
          .map(([o]) => o)
          .filter((o) => o !== w && Math.hypot(o.x - w.x, o.y - w.y) <= HEAD_R * 2);
        if (rivals.length === 0) {
          dead.set(w, { cause: "body", killer: null, credit: false });
          continue;
        }
        const biggest = rivals.reduce((a, b) => (b.segs > a.segs ? b : a));
        const ratio = Math.min(w.segs, biggest.segs) / Math.max(w.segs, biggest.segs);
        if (ratio > 0.9) {
          dead.set(w, { cause: "headon", killer: null, credit: false });
        } else if (w.segs < biggest.segs) {
          dead.set(w, { cause: "headon", killer: biggest, credit: true });
        } else {
          for (const o of rivals) {
            if (o.segs < w.segs) {
              dead.set(o, { cause: "headon", killer: w, credit: true });
            }
          }
          dead.delete(w);
        }
      }
      for (const [w, info] of dead) {
        killWorm(w, info.killer, info.cause, info.credit);
      }
    }
```

- [ ] **Step 3: Ganti `update` (panggil tabrakan + guard game-over)**

Ganti fungsi `update` lama dengan:

```js
    function update(dt) {
      playTime += dt;
      readInput();
      stepWorm(player, dt);
      for (const b of bots) {
        if (b.alive) stepWorm(b, dt);
      }
      eatFood(player);
      checkCollisions();
      if (state !== "playing") return;
      hudClock += dt;
      if (hudClock >= 0.2) {
        hudClock = 0;
        updateHUD();
      }
      cam.zoom += (clamp(1 - player.segs / 1250, 0.6, 1) - cam.zoom) * Math.min(1, dt * 3);
      cam.x += (player.x - cam.x) * Math.min(1, dt * 8);
      cam.y += (player.y - cam.y) * Math.min(1, dt * 8);
      cam.x = clamp(cam.x, 0, W);
      cam.y = clamp(cam.y, 0, W);
    }
```

Catatan: loop bot (`stepWorm(b, dt)`) baru ada AI-nya di Task 5 — di task ini `bots` masih kosong jadi aman.

- [ ] **Step 4: Verifikasi di browser**

Run: `open emoji_worm.html` → **Main**.
Expected:
- Dorong worm ke border → game over, pesan "Kamu menabrak border arena!", statistik tampil (Skor/Panjang/Kill/Waktu), **tidak ada** makanan baru berceceran di border (border = tanpa drop).
- Karena bot belum ada (Task 5), uji kematian tabrakan via console: jalankan `bots.push(makeWorm(player.x + 300, player.y, "🍕", "rakus")); bots[0].targetAngle = Math.atan2(player.y - bots[0].y, player.x - bots[0].x); seedTrail(bots[0]);` lalu dorong kepalamu ke trail bot → mati, makanan berceceran sepanjang trail bot, pesan "Kepalamu menabrak badan 🍕!".
- Uji sebaliknya: jalankan `player.segs = 100; player.alive` tetap; buat bot menabrak trail-mu dari console (`bots.push(makeWorm(player.x - 300, player.y, "🌮", "rakus")); bots[0].targetAngle = 0; seedTrail(bots[0]);`) → bot mati, **💀 Kill kamu naik**, skor +5.
- **Main Lagi** → reset penuh (skor 0, panjang 10, makanan 150). **Menu** → kembali ke overlay menu.
- Jangan ada error console.

- [ ] **Step 5: Commit**

```bash
git add emoji_worm.html
git commit -m "Add collision, death and game over rules"
```

### Task 5: Bot AI, Leaderboard, Minimap

**Files:**
- Modify: `emoji_worm.html`

- [ ] **Step 1: Tambahkan util & AI bot (sisipkan setelah `eatFood`)**

```js
    function nearAny(x, y, r) {
      for (const f of foods) {
        const dx = f.x - x;
        const dy = f.y - y;
        if (dx * dx + dy * dy <= r * r) return true;
      }
      for (const p of powers) {
        const dx = p.x - x;
        const dy = p.y - y;
        if (dx * dx + dy * dy <= r * r) return true;
      }
      return false;
    }

    function findSpot() {
      for (let i = 0; i < 40; i++) {
        const x = 400 + Math.random() * (W - 800);
        const y = 400 + Math.random() * (W - 800);
        if (!nearAny(x, y, 420)) return { x, y };
      }
      return { x: W / 2, y: W / 2 };
    }

    function botThink(w, dt) {
      if (!w.alive) return;
      w.decideT -= dt;
      const margin = 260;
      if (w.x < margin || w.y < margin || w.x > W - margin || w.y > W - margin) {
        w.targetAngle = Math.atan2(W / 2 - w.y, W / 2 - w.x);
        w.decideT = 0.4;
        return;
      }
      if (w.decideT > 0) return;
      w.decideT = 0.3 + Math.random() * 0.2;
      const ps = w.personality;
      if (ps === "penakut") {
        const threat = Math.hypot(w.x - player.x, w.y - player.y);
        if (player.alive && threat < 300 && player.segs > w.segs * 0.9) {
          w.targetAngle = Math.atan2(w.y - player.y, w.x - player.x);
          w.boost = w.segs > 40;
          return;
        }
        w.boost = false;
      }
      if (ps === "agresif" && player.alive) {
        const d = Math.hypot(w.x - player.x, w.y - player.y);
        if (d < 420 && player.segs < w.segs) {
          w.targetAngle = Math.atan2(player.y - w.y, player.x - w.x);
          w.boost = d > 160 && w.segs > 40;
          return;
        }
      }
      w.boost = false;
      let best = null;
      let bestD = 520 * 520;
      for (const f of foods) {
        const dx = f.x - w.x;
        const dy = f.y - w.y;
        const d2 = dx * dx + dy * dy;
        if (d2 < bestD) {
          bestD = d2;
          best = f;
        }
      }
      if (best) {
        w.targetAngle = Math.atan2(best.y - w.y, best.x - w.x);
        if (ps === "rakus" && bestD < 220 * 220) w.boost = w.segs > 60;
      } else {
        w.targetAngle = Math.random() * Math.PI * 2;
      }
    }
```

- [ ] **Step 2: Leaderboard & minimap (sisipkan setelah `drawFoods`)**

```js
    function updateBoard() {
      const all = [player].concat(bots).filter((w) => w && w.alive);
      all.sort((a, b) => b.segs - a.segs);
      const list = document.getElementById("boardList");
      list.innerHTML = "";
      for (let i = 0; i < Math.min(5, all.length); i++) {
        const w = all[i];
        const li = document.createElement("li");
        li.className =
          "flex items-center justify-between gap-1 rounded px-1 py-0.5" +
          (w.isPlayer ? " bg-sky-500/25" : "");
        const name = document.createElement("span");
        name.className = "truncate";
        name.textContent = (i + 1) + ". " + w.emoji + (w.isPlayer ? " Kamu" : "");
        const val = document.createElement("span");
        val.className = "tabular-nums";
        val.textContent = Math.ceil(w.segs);
        li.appendChild(name);
        li.appendChild(val);
        list.appendChild(li);
      }
    }

    function drawMinimap() {
      const size = 140;
      const x0 = 12;
      const y0 = vh - size - 12;
      ctx.fillStyle = "rgba(2, 6, 23, 0.7)";
      ctx.fillRect(x0, y0, size, size);
      ctx.strokeStyle = "rgba(56, 189, 248, 0.6)";
      ctx.lineWidth = 2;
      ctx.strokeRect(x0, y0, size, size);
      const sx = size / W;
      for (const f of foods) {
        ctx.fillStyle = "#facc15";
        ctx.fillRect(x0 + f.x * sx - 1, y0 + f.y * sx - 1, 2, 2);
      }
      const all = [player].concat(bots).filter((w) => w && w.alive);
      for (const w of all) {
        ctx.fillStyle = w.isPlayer ? "#f8fafc" : emojiColor(w.emoji);
        ctx.beginPath();
        ctx.arc(x0 + w.x * sx, y0 + w.y * sx, w.isPlayer ? 5 : 3.5, 0, 7);
        ctx.fill();
      }
      ctx.fillStyle = "rgba(148, 163, 184, 0.9)";
      ctx.font = "10px Nunito, sans-serif";
      ctx.textAlign = "left";
      ctx.textBaseline = "top";
      ctx.fillText("Peta", x0 + 5, y0 + 5);
    }
```

- [ ] **Step 3: Ganti `startGame` (spawn bot + board reset)**

Ganti `startGame` lama dengan (perubahan: pool bot emoji, spawn 6 bot, `updateBoard()` awal):

```js
    function startGame() {
      player = makeWorm(W / 2, W / 2, selected, "");
      player.isPlayer = true;
      seedTrail(player);
      bots = [];
      foods = [];
      powers = [];
      for (let i = 0; i < FOOD_COUNT; i++) spawnFood(null, null, 1);
      const pool = BOT_EMOJIS.filter((e) => e !== selected);
      for (let i = 0; i < BOT_COUNT; i++) {
        const spot = findSpot();
        const b = makeWorm(spot.x, spot.y, pool[i % pool.length], BOT_PERMS[i]);
        seedTrail(b);
        bots.push(b);
      }
      playTime = 0;
      hudClock = 0;
      boardClock = 0;
      powerTimer = 8 + Math.random() * 7;
      cam = { x: player.x, y: player.y, zoom: 1 };
      state = "playing";
      show("menu", false);
      show("gameover", false);
      show("hud", true);
      show("powerhud", true);
      show("board", true);
      show("boostBtn", true);
      updateHUD();
      updateBoard();
      if (document.activeElement) document.activeElement.blur();
    }
```

Catatan: `findSpot` memakai `foods`/`powers` — makanan sudah diisi sebelum loop spawn bot, jadi urutan ini wajib (makanan dulu, bot kemudian).

- [ ] **Step 4: Ganti `update` (AI + respawn + leaderboard periodik)**

Ganti `update` lama dengan:

```js
    function update(dt) {
      playTime += dt;
      readInput();
      stepWorm(player, dt);
      for (const b of bots) {
        if (b.alive) {
          botThink(b, dt);
          stepWorm(b, dt);
          eatFood(b);
        } else {
          b.respawnT -= dt;
          if (b.respawnT <= 0) {
            const spot = findSpot();
            const nb = makeWorm(spot.x, spot.y, b.emoji, b.personality);
            seedTrail(nb);
            Object.assign(b, nb);
          }
        }
      }
      eatFood(player);
      checkCollisions();
      if (state !== "playing") return;
      hudClock += dt;
      if (hudClock >= 0.2) {
        hudClock = 0;
        updateHUD();
      }
      boardClock += dt;
      if (boardClock >= 0.5) {
        boardClock = 0;
        updateBoard();
      }
      cam.zoom += (clamp(1 - player.segs / 1250, 0.6, 1) - cam.zoom) * Math.min(1, dt * 3);
      cam.x += (player.x - cam.x) * Math.min(1, dt * 8);
      cam.y += (player.y - cam.y) * Math.min(1, dt * 8);
      cam.x = clamp(cam.x, 0, W);
      cam.y = clamp(cam.y, 0, W);
    }
```

Catatan respawn: objek bot dipakai ulang via `Object.assign` supaya referensi di array tetap stabil; `makeWorm` mereset `respawnT`/`cause`/`segs`/`kills` (kill bot hilang saat respawn — sesuai spec: statistik bot per-kehidupan). Kematian bot yang memberi credit sudah ditangani `killWorm` di Task 4.

- [ ] **Step 5: Ganti `render` (gambar minimap setelah restore)**

Ganti `render` lama dengan:

```js
    function render() {
      ctx.fillStyle = "#0f172a";
      ctx.fillRect(0, 0, vw, vh);
      ctx.save();
      ctx.translate(vw / 2, vh / 2);
      ctx.scale(cam.zoom, cam.zoom);
      ctx.translate(-cam.x, -cam.y);
      drawGrid();
      drawBorder();
      drawFoods();
      drawWorms();
      ctx.restore();
      drawMinimap();
    }
```

- [ ] **Step 6: Verifikasi di browser**

Run: `open emoji_worm.html` → **Main**.
Expected: 6 bot emoji (≠ emoji pemain) muncul di posisi tersebar dan bergerak sesuai kepribadiannya — 🍕/🍒 **agresif** mengejar saat kamu dekat & lebih kecil, 🧁/🍇 **rakus** mengejar makanan (kadang boost), 🌮/🍪 **penakut** kabur saat kamu dekat & lebih besar; semua menghindari border. Leaderboard kanan-bawah (di atas tombol 🚀) menampilkan 5 worm terpanjang, baris pemain disorot biru, update tiap 0.5s. Minimap 140×140 kiri-bawah: titik makanan kuning, titik worm (putih untukmu), label "Peta". Bot yang kamu bunuh hilang 5 detik lalu muncul lagi di tempat baru dgn 10 segmen. Makanan bot jatuh saat boost. Leaderboard & skor bertambah saat kamu bunuh bot (💀 naik, +skor). Jangan ada error console.

- [ ] **Step 7: Commit**

```bash
git add emoji_worm.html
git commit -m "Add bot AI, leaderboard and minimap"
```

### Task 6: Power-Up (⭐ 👻 🧲)

**Files:**
- Modify: `emoji_worm.html`

- [ ] **Step 1: Tambahkan fungsi power-up (sisipkan setelah `eatFood`)**

```js
    function spawnPower() {
      powers.push({
        x: 200 + Math.random() * (W - 400),
        y: 200 + Math.random() * (W - 400),
        type: POWER_TYPES[Math.floor(Math.random() * POWER_TYPES.length)]
      });
    }

    function eatPower(w) {
      for (let i = 0; i < powers.length; i++) {
        const p = powers[i];
        const dx = p.x - w.x;
        const dy = p.y - w.y;
        if (dx * dx + dy * dy <= (HEAD_R + 16) * (HEAD_R + 16)) {
          if (p.type === "star") {
            w.segs = Math.min(MAX_SEGS, w.segs + 8);
            w.score += 10;
            const cap = Math.min(MAX_SEGS, Math.ceil(w.segs));
            while (w.trail.length < cap) w.trail.push({ ...w.trail[w.trail.length - 1] });
          } else if (p.type === "ghost") {
            w.ghost = 5;
          } else {
            w.magnet = 5;
          }
          powers.splice(i, 1);
          return;
        }
      }
    }

    function magnetPull(w, dt) {
      if (w.magnet <= 0) return;
      const r2 = 160 * 160;
      for (const f of foods) {
        const dx = w.x - f.x;
        const dy = w.y - f.y;
        const d2 = dx * dx + dy * dy;
        if (d2 > 0 && d2 <= r2) {
          const d = Math.sqrt(d2);
          const step = Math.min(300 * dt, d);
          f.x += (dx / d) * step;
          f.y += (dy / d) * step;
        }
      }
    }

    function updatePowerHUD() {
      const el = document.getElementById("powerhud");
      const items = [];
      if (player.ghost > 0) items.push("👻 " + player.ghost.toFixed(1) + "s");
      if (player.magnet > 0) items.push("🧲 " + player.magnet.toFixed(1) + "s");
      const html = items
        .map((t) => '<div class="rounded-full bg-slate-950/70 px-3 py-1 ring-1 ring-inset ring-white/15">' + t + "</div>")
        .join("");
      if (el.dataset.h !== html) {
        el.dataset.h = html;
        el.innerHTML = html;
      }
    }

    function drawPowers() {
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      const pulse = 1 + Math.sin(playTime * 5) * 0.1;
      for (const p of powers) {
        ctx.font = 22 * pulse + "px serif";
        ctx.fillText(POWER_EMOJI[p.type], p.x, p.y);
      }
    }
```

- [ ] **Step 2: Ganti `update` (timer power-up, aktifkan magnet & durasi)**

Ganti `update` lama dengan:

```js
    function update(dt) {
      playTime += dt;
      readInput();
      powerTimer -= dt;
      if (powerTimer <= 0) {
        powerTimer = 8 + Math.random() * 7;
        if (powers.length < 4) spawnPower();
      }
      if (player.ghost > 0) player.ghost = Math.max(0, player.ghost - dt);
      if (player.magnet > 0) player.magnet = Math.max(0, player.magnet - dt);
      magnetPull(player, dt);
      stepWorm(player, dt);
      for (const b of bots) {
        if (b.alive) {
          if (b.ghost > 0) b.ghost = Math.max(0, b.ghost - dt);
          if (b.magnet > 0) b.magnet = Math.max(0, b.magnet - dt);
          magnetPull(b, dt);
          eatPower(b);
          botThink(b, dt);
          stepWorm(b, dt);
          eatFood(b);
        } else {
          b.respawnT -= dt;
          if (b.respawnT <= 0) {
            const spot = findSpot();
            const nb = makeWorm(spot.x, spot.y, b.emoji, b.personality);
            seedTrail(nb);
            Object.assign(b, nb);
          }
        }
      }
      eatFood(player);
      eatPower(player);
      checkCollisions();
      if (state !== "playing") return;
      hudClock += dt;
      if (hudClock >= 0.2) {
        hudClock = 0;
        updateHUD();
        updatePowerHUD();
      }
      boardClock += dt;
      if (boardClock >= 0.5) {
        boardClock = 0;
        updateBoard();
      }
      cam.zoom += (clamp(1 - player.segs / 1250, 0.6, 1) - cam.zoom) * Math.min(1, dt * 3);
      cam.x += (player.x - cam.x) * Math.min(1, dt * 8);
      cam.y += (player.y - cam.y) * Math.min(1, dt * 8);
      cam.x = clamp(cam.x, 0, W);
      cam.y = clamp(cam.y, 0, W);
    }
```

Catatan: `magnetPull(b, dt)` tanpa syarat aman karena fungsi langsung return bila `magnet <= 0`; `eatFood(b)` juga menangani bot. Power-up pemain (`eatPower(player)`) dipanggil sebelum `checkCollisions` supaya 👻 aktif lebih dulu saat tabrakan frame itu.

- [ ] **Step 3: Ganti `render` (gambar power-up setelah makanan)**

Ganti `render` lama dengan:

```js
    function render() {
      ctx.fillStyle = "#0f172a";
      ctx.fillRect(0, 0, vw, vh);
      ctx.save();
      ctx.translate(vw / 2, vh / 2);
      ctx.scale(cam.zoom, cam.zoom);
      ctx.translate(-cam.x, -cam.y);
      drawGrid();
      drawBorder();
      drawFoods();
      drawPowers();
      drawWorms();
      ctx.restore();
      drawMinimap();
    }
```

- [ ] **Step 4: Verifikasi di browser**

Run: `open emoji_worm.html` → **Main**.
Expected: maksimal 4 power-up (⭐/👻/🧲) berdenyut muncul tiap 8–15 detik; ambil ⭐ → panjang +8, skor +10; ambil 👻 → worm semi-transparan 5 detik, menembus badan bot (kepala bot tidak membunuhmu, tapi head-on & border tetap mati), ikon 👻 + hitung mundur di HUD kanan-atas; ambil 🧲 → makanan dalam radius 160px tersedot ke arahmu selama 5 detik (300px/s), ikon 🧲 + hitung mundur; ikon hilang saat durasi habis. Bot juga bisa ambil power-up. Jangan ada error console.

- [ ] **Step 5: Commit**

```bash
git add emoji_worm.html
git commit -m "Add power-ups to Emoji Worm Zone"
```

### Task 7: Integrasi Landing Page + AGENTS.md

**Files:**
- Modify: `index.html`
- Modify: `AGENTS.md`

- [ ] **Step 1: Grid 2×2 di `index.html`**

Ganti `lg:grid-cols-3` pada baris grid (~baris 217) menjadi `lg:grid-cols-2`:

```html
      <div class="grid gap-4 sm:grid-cols-2 sm:gap-5 lg:grid-cols-2 lg:gap-6">
```

- [ ] **Step 2: Tambah kartu ke-4**

Sisipkan tepat setelah `</article>` penutup Kartu 3 (sebelum `      </div>` penutup grid). Salin struktur kartu yang ada (anchor + judul + deskripsi + meta), dengan:

- `href="https://ametsuramet.github.io/Amet-Simple-Games/emoji_worm.html"`
- emoji ikon: `🐛`
- judul: `Emoji Worm Zone`
- deskripsi: `Arcade worm bertema emoji — makan buah, tumbuh panjang, saling silang jalan dengan 6 bot AI. Boost, power-up, dan minimap!`
- meta kiri: `Arcade` / meta kanan: `44` (urutan keempat, sesuai pola nomor kartu eksisting)
- `style="animation-delay: 360ms"`

- [ ] **Step 3: Edit AGENTS.md (4 titik)**

1. Baris overview: `three self-contained browser games` → `four self-contained browser games`
2. Baris `The repo contains three vanilla JS` → `The repo contains four vanilla JS`
3. Tambah bullet setelah baris `onet_emoji.html`:
   `- emoji_worm.html - Emoji Worm Zone arcade game with bot AI, boost, and power-ups`
4. Baris `alongside the existing three` → `alongside the existing four`

- [ ] **Step 4: Verifikasi di browser**

Run: `python3 -m http.server 8000` → buka `http://localhost:8000/`.
Expected: grid landing page kini 2 kolom di layar besar (4 kartu tersusun 2×2); kartu ke-4 "Emoji Worm Zone" (🐛) tampil dengan animasi delay terakhir; klik → `emoji_worm.html` termuat (file lokal via server); menu game tampil. Klik 3 kartu lama → tetap berfungsi. Jangan ada error console.

- [ ] **Step 5: Commit**

```bash
git add index.html AGENTS.md
git commit -m "Register Emoji Worm Zone in landing page and AGENTS"
```

### Task 8: Verifikasi Akhir Seluruh Spec

**Files:**
- Modify: `emoji_worm.html` (bila ada perbaikan)

- [ ] **Step 1: Checklist manual spec (10 poin)**

Buka game via `python3 -m http.server 8000` (uji juga mode device/mobile di DevTools):

1. Menu: pilih emoji → worm & pemain memakai emoji pilihan; bot ≠ emoji pemain.
2. Makan: 150 makanan, respawn penuh saat dimakan, skor/panjang naik, zoom mundur bertahap (>1250 seg → min 0.6).
3. Boost: Spasi/🚀 → ×2.1, −1 seg per 0.3s, makanan jatuh, min 10 seg; joystick hanya kiri 60% & non-mouse; tombol 🚀 tidak memicu joystick.
4. Kematian: badan-ditabrak (kill ke pemilik badan), head-on ≤10% (dua-duanya, tanpa credit), head-on >10% (yang kecil mati, survivor credit), border (tanpa drop), ghost lolos badan tapi mati head-on/border — sesuai aturan.
5. Kill reward: 💀 +1, skor +5 + floor(segs/10); makanan drop 1 per segmen korban.
6. Bot: 6 bot 3 kepribadian (agresif/rakus/penakut), respawn 5 detik dgn 10 segmen, hindari border margin 260px.
7. Leaderboard: top-5, baris pemain disorot, update 0.5s.
8. Minimap: 140×140 kiri-bawah, makanan kuning, worm putih/punya warna.
9. Power-up: maks 4, spawn 8–15s, ⭐ +8 seg +10 skor, 👻 5s, 🧲 5s radius 160px pull 300px/s, HUD kanan-atas.
10. Alur: Main → game over → Main Lagi (reset penuh) → Menu; visibilitychange pause/resume tanpa error; layout 2×2 landing page OK; tanpa error console.

- [ ] **Step 2: Commit perbaikan (jika ada)**

```bash
git add emoji_worm.html
git commit -m "Fix issues found in final Emoji Worm Zone verification"
```

(Lewati commit ini bila tidak ada perbaikan.)

- [ ] **Step 3: Merge & push**

```bash
git checkout main
git merge feature/emoji-worm
git push origin main
git branch -d feature/emoji-worm
```

Expected: fast-forward merge, push sukses (`github.com/ametsuramet/Amet-Simple-Games`), branch terhapus. Game live di `https://ametsuramet.github.io/Amet-Simple-Games/emoji_worm.html` (GitHub Pages).
