# Shadowing Sidepanel — UI Templates (v2)

Six hand-crafted UI templates for the Chrome **side panel** of the language-shadowing browser extension.
Live showcase: **https://local-de-coach.github.io/sidepanel-templates/**

## v2 update (this revision)

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
