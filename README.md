# Shadowing Sidepanel — UI Templates (v3)

Six hand-crafted UI templates for the Chrome **side panel** of the language-shadowing browser extension.
Live showcase: **https://local-de-coach.github.io/sidepanel-templates/**

## v3 update (this revision)

- 🎤 **Live word tracker (all six)** — while you shadow, every word lights up as you say it: spoken words
  dim to gray, the current word gets a **themed chip** (yellow marker in 03 — like the Duolingo reference,
  iOS blue in 09, Spotify green in 04, M3 purple in 06, Upwork green in 01, brutalist sticker in 08),
  synced with the sentence progress bar.
  → Full feature breakdown + **backend contract with examples**: see *🎯 Word Tracker* at the bottom of this file.
- 03 · **no-scroll layout** — the ring is compacted and Daily goal / streak row / `UP NEXT · LONG SENTENCES`
  moved off the main flow into a **"Quests & up next" sheet**, opened from the 💎 XP button in the top bar.
  Main view now fits any sidepanel height without scrolling (`min-height:100vh` + flexible spacer).
- 09 · **compact header + two-line progress** — removed `‹ Library`; the nav is now
  `🌐 EN · Shadowing · ⋯` (settings behind the three-dot). Progress is two lines:
  **line 1** = position of the sentence inside the video (blue segment on the video track),
  **line 2** = real-time sentence progress with a green **"you" fill** that trails the audio while you speak,
  plus a live `YOU · 00:44.1` timecode.

## v2 update

- ✂️ Reduced from 10 → **6 templates** (kept `01 · 03 · 04 · 06 · 08 · 09`; retired `02 · 05 · 07 · 10`).
- 🌍 **Transcript languages page** — a button in every header opens a full language switcher *inside the same
  side panel* (manual / ASR badges, quality dots, sentence counts, "re-prepare" note).
- ⏱ **Sentence-based training** — the training unit is the **whole long sentence** (no word-by-word karaoke):
  per-sentence progress bar, sentence loop & auto-pause, full-sentence recording and scoring, up-next queue.
- ⋮ **Settings on demand** — Settings are hidden by default and open **only** from the ⋮ (three-dot) menu.
  Training unit is locked to `Sentence ✓` (`Word` is marked OFF).
- 🖼 **HTML only** — no screenshots/images; the showcase uses live iframe previews.

## Templates

| # | File | Style | Signature sections |
|---|------|-------|--------------------|
| 01 | `01-clarity-pro.html` | Upwork-inspired light SaaS | Sentence card + progress, transport, record CTA, stats, up-next queue |
| 03 | `03-streak-quest.html` | Duolingo-style gamified | XP ring, streak/hearts, mascot tip, 3D record button, daily goal |
| 04 | `04-midnight-audio.html` | Spotify-style dark console | Cover + waveform + A-B loop, rec level bars, take score, tab nav |
| 06 | `06-material-you.html` | Material Design 3 | Segmented tabs, tonal sentence card, M3 switches, extended FAB |
| 08 | `08-pop-brutal.html` | Neo-brutalism | Sticker header, block progress, chunky buttons, sentence report |
| 09 | `09-cupertino.html` | iOS 17 | Large title, segmented control, voice-memo recorder, grouped lists |

## Shared interaction spec (all six)

- `openLangs()` / `pickLang()` — in-panel transcript-languages page; header chip reflects the selection.
- Sentence stage with per-sentence progress (`00:41.2 → 00:46.8`), loop ×3, auto-pause.
- `openSet()` — settings overlay bound to the ⋮ button only; backdrop click closes.
- Toast feedback on language switch; toggles are clickable in the preview.

## Using them in the extension

Each file is fully self-contained (inline CSS + vanilla JS, Google Fonts link only) — drop one into the
MV3 side panel (`chrome.sidePanel`), or lift the overlay pattern (`#langs`, `#settings`) into your own UI.
All strings/pixel values are design-intent placeholders ready to be wired to the backend
(`POST /v1/prepare` → `/v1/lesson` → `/v1/score`).

---

# 🎯 Word Tracker — how it works (backend contract + examples)

The **word tracker** is the line of words in the sentence card that reacts while you shadow —
the feature you saw in the Duolingo reference image: every word you have already said dims to
gray, the word you are saying **right now** gets a themed chip, and after scoring each word
keeps a green / amber / orange tint showing how well you said it. All six templates render it
with the same logic and the same DOM — only colors change per theme.

This section is the complete feature spec: UI states, the two data modes, the exact frontend
wiring code, **what your backend must return** (real JSON from the Shadowing Engine API, schema 2),
worked examples, and the optional additive fields if you want to go further.

## 1. TL;DR — what your backend must provide

| Tracker capability | Driven by | Status in your API today |
|---|---|---|
| Word chip moves while the **native clip** plays (karaoke) | `GET /v1/lesson` → `words[].start` / `words[].end` | ✅ already in schema 2 |
| Spoken words dim + sentence progress bar | same clock (`audio.currentTime`) | ✅ client-side, no data needed |
| "Your voice" fill while recording (09 · line 2) | recording stopwatch mapped onto the native word timeline | ✅ client-side estimate, no data needed |
| Per-word feedback **after** scoring (`good / weak / miss`) | `POST /v1/score` → `words[].score` | ✅ already in schema 2 |
| Hover tips on weak words | `words[].phones[].tip` | ✅ already in schema 2 |
| Drill chips (worst IPA first) | `focus[]` | ✅ already in schema 2 |
| Real per-word timings of **your** audio (replay / inspector) | `words[].you_start` / `you_end` | 🟡 optional additive field (§5) |
| "User skipped this word" flag | `words[].matched` | 🟡 optional additive field (§5) |
| Live ASR position of the learner's voice **during** recording | WebSocket stream | 🟡 optional, future (§6) |

**Bottom line for your backend:** the Shadowing Engine as documented (`api.md`, schema 2)
**already returns everything the word tracker needs** for its two main modes — live karaoke and
post-attempt word feedback. No breaking change is required. The only backend work is optional
additive fields, and your API convention already allows that (*"Unknown/optional fields may be
added in minor versions — ignore what you don't know"*).

There is exactly **one hard contract rule** to verify in your backend:

> **`lesson.words[]` must be exactly the whitespace tokenization of `lesson.text` — same count,
> same order, one entry per token.**
> The frontend renders `text.trim().split(/\s+/)` and aligns everything **by index, never by
> string matching** (see §4). If your aligner ever normalizes tokens (e.g. text says `43` but
> the aligner speaks `forty-three`), keep the display form `43` in `words[].w` and encode the
> spoken form only inside `phones[]`.

## 2. UI anatomy and states

The tracker is one container whose text is replaced by word spans on load:

```html
<!-- before -->
<div class="sent">I've been meaning to ask you about that new coffee place downtown.</div>

<!-- after init JS runs -->
<div class="sent">
  <span class="w done">I've</span> <span class="w done">been</span> …
  <span class="w cur">coffee</span> <span class="w">place</span> <span class="w">downtown.</span>
</div>
```

### 2.1 Live states (while listening / recording)

| Class | Meaning | Set when | Look in 01 (Clarity Pro) |
|---|---|---|---|
| *(none)* | upcoming word | index > cursor | normal ink color |
| `.done` | already spoken | index < cursor | dimmed gray `#9fb0a0` |
| `.cur` | current word | index == cursor | green chip, white text, soft glow |

### 2.2 Report states (painted after `POST /v1/score`)

Thresholds deliberately mirror the backend band table (`excellent ≥ 85`, `good ≥ 70`,
`almost ≥ 50`, `try_again < 50`) so UI and scores stay consistent:

| Class | Score | Backend band | Recommended look (per theme) |
|---|---|---|---|
| `.good` | ≥ 70 | `excellent` / `good` | theme green text, faint green tint |
| `.weak` | 50 – 69 | `almost` | amber text + soft amber background; `title` tooltip from `phones[].tip` |
| `.miss` | < 50 | `try_again` | soft orange text + tint — **never red/shame**, matches band guidance |

### 2.3 Themed chip colors (the `.cur` chip) per template

| Template | Chip background | Chip text | Extra |
|---|---|---|---|
| 01 · Clarity Pro | `--green` `#14a800` | white | glow `rgba(20,168,0,.4)` |
| 03 · Streak Quest | Duolingo yellow `#ffc800` | `#3c3c3c` | matches the reference image |
| 04 · Midnight Audio | Spotify green `#1db954` | near-black | glow |
| 06 · Material You | M3 purple tonal | on-primary | rounded M3 chip |
| 08 · Pop Brutal | yellow | black | 2px hard border + offset shadow |
| 09 · Cupertino | iOS blue `#007aff` | white | soft blue glow |

Shared report-mode CSS to drop into any template (adjust the three hex pairs from the table style):

```css
.sent .w.good{ color:#14a800 }
.sent .w.weak{ color:#c77800; background:rgba(255,200,0,.14) }
.sent .w.miss{ color:#c94f42; background:rgba(255,92,92,.12) }
```

## 3. The two data modes

```
prepare (once per clip)                 score (once per attempt)
POST /v1/prepare                        POST /v1/score  (lesson_id + audio)
      │ 202 {job_id, poll}                    │ 200 ScoreReport
      ▼ poll                                  ▼
GET lesson ──► words[].start/end ──► MODE A: live tracking (clock-driven)
                                              │
                                              ▼
                                    MODE B: report (repaint by score)
```

**Mode A — live tracking (clock-driven).** The source of truth is `lesson.words[]`, produced by
forced alignment on the **native clip clock** (seconds from clip start). While the native audio
plays, the cursor is the word whose `[start, end)` contains `audio.currentTime`. While *you*
record (auto-pause mode: native is stopped), the tracker maps the recording stopwatch onto the
native timeline — legitimate because shadowing is text-dependent: the cursor simply assumes you
keep native pace, which is exactly the skill being trained.

**Mode B — attempt report (score-driven).** After `POST /v1/score` returns the `ScoreReport`,
each word is repainted by `words[].score` (`.good` / `.weak` / `.miss`), weak phones get a hover
`tip`, and `focus[]` renders as drill chips. Mode A colors are replaced by Mode B colors until
the next attempt.

Both modes are **stateless paints from data** — no accumulated UI state, which is why seek /
A-B loop / replay work for free.

## 4. Wiring the templates to the real backend (drop-in JS)

The six templates currently drive the tracker with a **mock clock** (`setInterval` 430 ms, one
word per tick, starting at index 9 = "coffee"; in 09 the "you" fill is simulated as
`cursor − 1.4` words and the `YOU · 00:44.1` label from `T0 + (T1−T0)·y/12`). It is clearly
marked in every file — search for `v3 · live word tracker`. Replace that IIFE with the code
below; the CSS does not change.

### 4.1 Render tokens and attach lesson timings

```js
const sentEl = document.querySelector('.sent');
let tokens = [];                                  // [{el, start, end}]
let nativeDur = 0;

function renderTokens(text){                      // tokenization = whitespace split. nothing else.
  sentEl.innerHTML = text.trim().split(/\s+/)
    .map(w => '<span class="w">' + w + '</span>').join(' ');
  tokens = [...sentEl.querySelectorAll('.w')].map(el => ({el}));
}

function attachLesson(lesson){                    // words[] aligns 1:1 with tokens by index
  nativeDur = lesson.native_duration_s;
  lesson.words.forEach((w, k) => {
    if (tokens[k]) { tokens[k].start = w.start; tokens[k].end = w.end; }
  });
}
```

### 4.2 Clock → word index (binary search)

```js
// returns the index of the word containing t; if t falls in a gap between
// words it keeps the previous word highlighted; -1 = before the first word.
function wordIndexAt(t){
  let lo = 0, hi = tokens.length - 1, ans = -1;
  while (lo <= hi){
    const mid = (lo + hi) >> 1;
    if (t < tokens[mid].start) hi = mid - 1;
    else if (t >= tokens[mid].end) { ans = mid; lo = mid + 1; }
    else return mid;                              // start ≤ t < end → exact hit
  }
  return ans;
}

function paintCursor(i){                          // stateless paint — safe on seek/loop
  tokens.forEach((w, k) => {
    w.el.classList.toggle('done', k < i);
    w.el.classList.toggle('cur', k === i);
  });
  sprog.style.width = Math.max(0, Math.round((i + 1) / tokens.length * 100)) + '%';
}
```

### 4.3 Mode A — karaoke while the native clip plays

```js
let cur = -1;
function tick(){                                  // rAF ≈ 60 fps; timeupdate only fires ~4 Hz
  const i = wordIndexAt(audioEl.currentTime);
  if (i !== cur){ cur = i; paintCursor(i); }
  requestAnimationFrame(tick);
}
```

For 09's two-line progress, the real values replace the mock numbers directly:

```js
// line 1 — sentence position inside the VIDEO (mock: left 19.4% = 41.2 / 212 s)
vseg.style.left = ((lesson.source.start + audioEl.currentTime) / videoDuration * 100) + '%';

// line 2 — blue fill = native sentence progress; green "you" fill while recording:
//   youT = clipStart + (clipEnd − clipStart) × elapsedRec / nativeDur
//   label: 'YOU · ' + fmt(youT)      → replaces the mock 'YOU · 00:44.0'
```

### 4.4 Mode A while *you* record (auto-pause: native clock is stopped)

```js
let recStart = 0, rafId = 0;
function recTick(){
  const elapsed = (performance.now() - recStart) / 1000;   // seconds since record start
  paintCursor(wordIndexAt(elapsed));                       // assume native pace
  youFill.style.width = Math.min(100, elapsed / nativeDur * 100) + '%';
  rafId = requestAnimationFrame(recTick);
}
// start: recStart = performance.now(); recTick();
// stop:  cancelAnimationFrame(rafId);  then send the blob to POST /v1/score (§4.6)
```

Optional refinement: after the first take you know the learner's pace
(`report.pace.ratio`, e.g. 1.09) — scale the cursor with it:
`paintCursor(wordIndexAt(elapsed / report.pace.ratio))`.

### 4.5 Mode B — repaint after scoring

```js
function applyScore(report){
  if (report.verdict !== 'scored'){                 // 'retry' = couldn't hear you (api.md §6)
    toast("Couldn't hear that — go again");
    return;                                         // keep live colors, never show zeros
  }
  if (report.words.length !== tokens.length) return; // re-render from lesson.text first (§7)
  report.words.forEach((w, k) => {
    const el = tokens[k].el;
    el.classList.remove('cur', 'done');
    el.classList.add(w.score >= 70 ? 'good' : w.score >= 50 ? 'weak' : 'miss');
    const tips = (w.phones || []).filter(p => p.tip).map(p => p.p + ': ' + p.tip).join('\n');
    if (tips) el.title = tips;                      // hover hint, e.g. "œ: round the vowel"
  });
  focusEl.innerHTML = (report.focus || [])
    .map(p => '<i class="fchip">' + p + '</i>').join('');
}
```

### 4.6 Putting it together — the two API calls

```js
const api = 'http://localhost:8000';                // your Shadowing Engine base URL
const sleep = ms => new Promise(r => setTimeout(r, ms));
const getJSON  = (u, o) => fetch(api + u, o).then(r => r.json());
const postJSON = (u, b) => fetch(api + u, {
  method: 'POST', headers: {'Content-Type': 'application/json'},
  body: JSON.stringify(b)
}).then(r => r.json());

// 1) once per clip — mirrors api.md §5.1–5.3 incl. the cache-hit shortcut
async function prepare(body){
  const j = await postJSON('/v1/prepare', body);    // {url, start, end, lang?, text?}
  if (j.status === 'done') return getJSON(j.poll);  // cache hit → poll IS the lesson URL
  for (let i = 0; i < 60; i++){
    const job = await getJSON(j.poll);
    if (job.status === 'done')   return job.lesson;
    if (job.status === 'failed') throw new Error(job.error_code);
    await sleep(1000);
  }
  throw new Error('prepare timeout');
}

// 2) once per attempt — mirrors api.md §5.4
async function scoreAttempt(lesson, blob){
  const fd = new FormData();
  fd.append('lesson_id', lesson.id);
  fd.append('audio', blob, 'attempt.webm');         // MediaRecorder default is fine
  const r = await fetch(api + '/v1/score', { method: 'POST', body: fd });
  if (r.status === 422) return { verdict: 'retry' }; // alignment_failed — normal, show retry
  return r.json();
}

// flow:  const lesson = await prepare({url, start, end, lang});
//        renderTokens(lesson.text); attachLesson(lesson); requestAnimationFrame(tick);
//        … record …  applyScore(await scoreAttempt(lesson, blob));
```

> Auth & CORS: if `API_KEY` is set on the server, add
> `headers: {'Authorization': 'Bearer …'}` to every call; declare
> `host_permissions` for the backend origin in the MV3 manifest (or add the
> `chrome-extension://…` origin to `CORS_ORIGINS`).

## 5. Optional backend upgrades (additive fields — no schema bump)

Forced alignment already computes *where in the learner's audio* each reference word landed —
the backend just doesn't emit it yet. Adding these three optional fields to `ScoreReport.words[]`
unlocks the richest version of the tracker without breaking anything (§4 Conventions: unknown
fields are ignored by old clients):

```jsonc
"words": [
  { "w": "coffee", "score": 58,
    "matched": true,                 // false = aligner found no region → user skipped it
    "you_start": 3.78, "you_end": 4.41,   // WHERE in the attempt audio you said it (seconds)
    "phones": [ {"p": "ɔ", "score": 49, "tip": "open the vowel"} ] }
]
```

| Field | Type | Meaning | UI it unlocks |
|---|---|---|---|
| `you_start`, `you_end` | float \| null | word position on the **attempt** clock (0 = first sample of the recording) | replay "your voice" with karaoke; per-word A/B compare (native vs you); real `YOU ·` timeline in 09 after scoring |
| `matched` | bool | `false` → word not found in your audio | `.miss` + "skipped?" hint — distinguishes *said badly* from *not said* (score is `null` then) |
| `score` stays 0–100 int; `null` only when `matched:false` | | | |

## 6. Optional future: live learner position while recording (WebSocket)

Everything above needs **no streaming**. Only if you later want the green "you" fill / word chip
to follow the learner's **actual voice** in real time (instead of the stopwatch estimate of §4.4)
would you add a WebSocket endpoint. Proposed minimal contract — build this last:

```
WS /v1/ws/track                    (same auth + CORS rules as REST)

client → {"type":"start","lesson_id":"les_9c31e2a7"}
client → (binary audio chunks, audio/webm, ~250 ms)
server → {"type":"position","word_index":9,"t_attempt":3.60}     // ~4 Hz, incremental align
server → {"type":"final","report":{ …full ScoreReport incl. you_start/you_end… }}
server → {"type":"error","code":"alignment_failed"}
```

## 7. Worked example end-to-end (the sentence live in the templates)

Sentence card: *"I've been meaning to ask you about that new coffee place downtown."* — 12
tokens, clip `00:41.2 → 00:46.8` (5.6 s), so **"coffee" is index 9** (why the mock starts there).

**Step 1 — lesson (what `GET /v1/lesson` returns, abridged but real shape):**

```jsonc
{
  "id": "les_7d20aa41", "schema": 2, "lang": "en",
  "source": {"provider": "youtube", "video_id": "…", "start": 41.2, "end": 46.8},
  "text": "I've been meaning to ask you about that new coffee place downtown.",
  "native_clip": "clips/les_7d20aa41.wav", "native_duration_s": 5.6,
  "words": [
    {"w": "I've",     "start": 0.04, "end": 0.32, "phones": [{"p": "aɪ", "start": 0.04, "end": 0.22}, {"p": "v", "start": 0.22, "end": 0.32}]},
    {"w": "been",     "start": 0.32, "end": 0.58, "phones": [{"p": "b", "start": 0.32, "end": 0.40}, {"p": "ɪn", "start": 0.40, "end": 0.58}]},
    {"w": "meaning",  "start": 0.58, "end": 1.14, "phones": [{"p": "m", "start": 0.58, "end": 0.68}, {"p": "iː", "start": 0.68, "end": 0.92}, {"p": "nɪŋ", "start": 0.92, "end": 1.14}]},
    {"w": "to",       "start": 1.14, "end": 1.32, "phones": [{"p": "tʊ", "start": 1.14, "end": 1.32}]},
    {"w": "ask",      "start": 1.32, "end": 1.66, "phones": [{"p": "æ", "start": 1.32, "end": 1.50}, {"p": "sk", "start": 1.50, "end": 1.66}]},
    {"w": "you",      "start": 1.66, "end": 1.90, "phones": [{"p": "juː", "start": 1.66, "end": 1.90}]},
    {"w": "about",    "start": 1.90, "end": 2.36, "phones": [{"p": "ə", "start": 1.90, "end": 2.02}, {"p": "baʊt", "start": 2.02, "end": 2.36}]},
    {"w": "that",     "start": 2.36, "end": 2.62, "phones": [{"p": "ð", "start": 2.36, "end": 2.44}, {"p": "æt", "start": 2.44, "end": 2.62}]},
    {"w": "new",      "start": 2.62, "end": 3.10, "phones": [{"p": "n", "start": 2.62, "end": 2.74}, {"p": "juː", "start": 2.74, "end": 3.10}]},
    {"w": "coffee",   "start": 3.42, "end": 3.95, "phones": [{"p": "k", "start": 3.42, "end": 3.51}, {"p": "ɔ", "start": 3.51, "end": 3.68}, {"p": "f", "start": 3.68, "end": 3.80}, {"p": "iː", "start": 3.80, "end": 3.95}]},
    {"w": "place",    "start": 3.95, "end": 4.34, "phones": [{"p": "p", "start": 3.95, "end": 4.03}, {"p": "leɪs", "start": 4.03, "end": 4.34}]},
    {"w": "downtown", "start": 4.34, "end": 5.52, "phones": [{"p": "d", "start": 4.34, "end": 4.42}, {"p": "aʊn", "start": 4.42, "end": 4.80}, {"p": "taʊn", "start": 4.80, "end": 5.52}]}
  ],
  "calibration": {"mu": -0.87, "sigma": 0.52, "n": 34}
}
```

**Step 2 — live snapshot at `audio.currentTime = 3.60 s`** → `wordIndexAt(3.6)` = **9**:

| token | index | class | user sees (01) |
|---|---|---|---|
| I've … new | 0 – 8 | `.done` | dimmed gray |
| coffee | 9 | `.cur` | green chip, white text |
| place, downtown. | 10 – 11 | *(none)* | normal ink |

**Step 3 — the learner records; `POST /v1/score` returns (abridged):**

```jsonc
{
  "lesson_id": "les_7d20aa41",
  "overall": 84, "band": "good", "verdict": "scored",
  "pace": {"native_s": 5.6, "you_s": 6.1, "ratio": 1.09, "score": 88},
  "rhythm": 79,
  "words": [
    {"w": "I've", "score": 93, "phones": [{"p": "aɪ", "score": 95, "tip": null}, {"p": "v", "score": 90, "tip": null}]},
    {"w": "been", "score": 91, "phones": [{"p": "b", "score": 92, "tip": null}, {"p": "ɪn", "score": 90, "tip": null}]},
    {"w": "meaning", "score": 88, "phones": [{"p": "m", "score": 90, "tip": null}, {"p": "iː", "score": 84, "tip": null}, {"p": "nɪŋ", "score": 89, "tip": null}]},
    {"w": "to", "score": 95, "phones": [{"p": "tʊ", "score": 95, "tip": null}]},
    {"w": "ask", "score": 90, "phones": [{"p": "æ", "score": 88, "tip": null}, {"p": "sk", "score": 92, "tip": null}]},
    {"w": "you", "score": 92, "phones": [{"p": "juː", "score": 92, "tip": null}]},
    {"w": "about", "score": 87, "phones": [{"p": "ə", "score": 86, "tip": null}, {"p": "baʊt", "score": 88, "tip": null}]},
    {"w": "that", "score": 89, "phones": [{"p": "ð", "score": 85, "tip": null}, {"p": "æt", "score": 92, "tip": null}]},
    {"w": "new", "score": 86, "phones": [{"p": "n", "score": 90, "tip": null}, {"p": "juː", "score": 83, "tip": null}]},
    {"w": "coffee", "score": 58, "phones": [{"p": "k", "score": 84, "tip": null}, {"p": "ɔ", "score": 49, "tip": "open the vowel — 'caw', not 'cow'"}, {"p": "f", "score": 70, "tip": null}, {"p": "iː", "score": 62, "tip": "hold it longer"}]},
    {"w": "place", "score": 88, "phones": [{"p": "p", "score": 91, "tip": null}, {"p": "leɪs", "score": 86, "tip": null}]},
    {"w": "downtown", "score": 71, "phones": [{"p": "d", "score": 78, "tip": null}, {"p": "aʊn", "score": 66, "tip": "glide fully into 'u'"}, {"p": "taʊn", "score": 70, "tip": null}]}
  ],
  "focus": ["ɔ", "aʊ"],
  "timing_ms": {"total": 1480, "vad": 62, "align": 910, "gop": 420, "dtw": 88}
}
```

**Step 4 — `applyScore()` repaints (thresholds ≥ 70 `.good`, 50–69 `.weak`, < 50 `.miss`):**

| token | score | class | UI |
|---|---|---|---|
| I've … new (0–8) | 86 – 95 | `.good` | green text |
| coffee (9) | 58 | `.weak` | amber tint, hover tip *"ɔ: open the vowel — 'caw', not 'cow'"* |
| place (10) | 88 | `.good` | green text |
| downtown (11) | 71 | `.good` | green text |
| drill chips | — | `focus[]` | `ɔ` `aʊ` chips under the card, worst-first |

`overall 84 → band "good"` → the header pill/CTA shows the encourage state; `pace.ratio 1.09`
feeds back into §4.4's cursor scaling for the next take.

## 8. Rules and edge cases

1. **Index alignment, never string matching.** The frontend compares nothing textual — it only
   zips `lesson.words[k]` with `span[k]`. Punctuation stays attached (`downtown.` is one token),
   apostrophes/contractions stay one token (`I've`), German compounds stay one token
   (`Kaffeevollautomat`), separable verbs keep their written form. That's why umlauts/`ß`
   need no special handling: the backend guarantees `words[]` = whitespace split of `text`
   (the one hard rule from §1).
2. **Numbers & normalization trap.** If `text` contains `43`, the aligner must still emit one
   word `{"w":"43"}` (not `"forty-three"`), or the 1:1 index contract breaks. If you need the
   spoken form, put it in `phones[]` (e.g. `"p":"fɔːti θriː"`).
3. **Token count mismatch** (user edited the text and re-prepared with `force:true`): always
   re-render tokens **from the new `lesson.text`** — never keep spans from an older lesson.
   `applyScore()` bails out if lengths differ (§4.5).
4. **`verdict:"retry"` / HTTP 422** (couldn't hear you): keep the last live colors, toast
   "go again", never paint zeros — same philosophy as the band table.
5. **Silence gaps between words**: `wordIndexAt` keeps the previous word `.cur` during a gap —
   words light up only while the clock is inside `[start, end)`.
6. **Seek / A-B loop / replay**: the paint is a pure function of `currentTime`, so all transport
   features work with zero extra code (the cursor jumps back with the audio).
7. **Pace mismatch**: if the learner is slower (`pace.ratio > 1.15`), the estimated "you" fill
   can outrun the sentence — clamp it to 100% (§4.4 does), or scale by the last ratio.
8. **Multiple takes**: repainting on every report is correct; optionally keep the **best score
   per word across takes** (one `Math.max` merge loop) so the card improves visually as the
   learner drills.
9. **`text_confidence < 0.95`** (text came from ASR captions): offer the "is this the sentence?"
   edit **before** training — the tracker visually commits the user to token boundaries, so wrong
   text = wrong tracking (api.md §8.3 flow: edit → re-prepare with corrected `text` + `force:true`).

## 9. Backend integration checklist

- ☑ **Nothing required to change** for the two main modes — schema 2 already ships
  `lesson.words[].start/end` and `score.words[].score` / `phones[].tip` / `focus[]`.
- ☑ **Verify** the one hard rule: `words[]` = whitespace tokenization of `text`, 1:1 by index,
  display form preserved (numbers, punctuation, contractions).
- ☑ Serve `DATA_DIR/clips/` statically so the sidepanel can play the native clip it karaoke's over.
- ☐ Optional: add `words[].you_start / you_end` (§5) → real you-replay + 09 timeline after scoring.
- ☐ Optional: add `words[].matched` (§5) → skipped-word detection.
- ☐ Optional, last: `WS /v1/ws/track` (§6) → live learner-voice position during recording.
- ☑ MV3 wiring: `host_permissions` for the backend origin (or `CORS_ORIGINS`), bearer header
  only if `API_KEY` is set.
