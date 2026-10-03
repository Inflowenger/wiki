# Production bible — *Building on FloMorphic* video series

> **Not part of the book.** This folder is production material for filming
> [Part VIII](../../book/08-flomorphic-guide/) as a seven-episode screencast series and a
> companion deck. Nothing here is published as documentation; it exists so a shoot is
> repeatable and a reshoot is cheap.

## The series

Seven episodes, one per chapter. Each is independently useful, and a bad take only costs one
episode.

| Ep | Chapter | Target | Spoken words @150 wpm | Shots | Script state |
| --- | --- | --- | --- | --- | --- |
| 1 | [Install](../../book/08-flomorphic-guide/01-install.md) | 8 min | ~1,200 | 9 | **verbatim** — [ep1-install.md](ep1-install.md) |
| 2 | [Orientation](../../book/08-flomorphic-guide/02-orientation.md) | 20 min | ~3,000 | 24 | **verbatim** — [ep2-orientation.md](ep2-orientation.md) |
| 3 | [Identification](../../book/08-flomorphic-guide/03-identify.md) | 15 min | ~2,250 | 10 | **not written yet** |
| 4 | [Build a plugin](../../book/08-flomorphic-guide/04-build-a-plugin.md) | 18 min | ~2,700 | 11 | **not written yet** |
| 5 | [Ingestion](../../book/08-flomorphic-guide/05-ingest.md) | 25 min | ~3,750 | 16 | **not written yet** |
| 6 | [Decision](../../book/08-flomorphic-guide/06-decide.md) | 25 min | ~3,750 | 15 | **not written yet** |
| 7 | [What you built](../../book/08-flomorphic-guide/07-what-you-built.md) | 8 min | ~1,200 | 7 | **not written yet** |

Total ≈ **2 hours**, ≈ 92 shots, ≈ 17,850 spoken words. **Ep1 and ep2 are written and
filmable (28 min, ~4,200 words, 33 shots). Ep3–7 are planned but unwritten.**

**150 words per minute**, not the conversational 160–180. Technical narration over a UI needs
the viewer's eyes to land before the next sentence starts.

## The demo install

**Hybrid.** One real database so the claims that matter are genuine; everything else seeded by
the flow itself, following the convention in
[`flomorphic-wapp/samples`](https://github.com/FloMorphic/morph-wapp/tree/main/samples) —
*"every sample seeds its own data, so you need no set-up beyond an LLM credential."*

| Source | On camera | Why |
| --- | --- | --- |
| **Postgres** `crm` | **Real**, in `docker compose` | Ep3's Extensions install, ep4's settings profile, and ep6's live contract read are the three places *"it attaches to the system you already run"* has to be true, not mimed |
| MySQL `ticketing` | Seeded by a JS first node | A legacy ticket table adds nothing on screen that Postgres has not already shown |
| ArangoDB `assets` | Seeded by a JS first node | Ep4 builds the *plugin*; a live Arango would only prove the plugin works, which the subscription log already does |
| 380k tickets | **40 representative rows** | Volume is a slide, not a screen. Say the number, show forty |
| Vendor parts API | Seeded | — |

### Why not fully self-seeded

A demo where every source is faked quietly guts
[chapter 3's central claim](../../book/08-flomorphic-guide/03-identify.md#installing-a-plugin-from-the-catalog).
One real Postgres costs one compose file and buys the episode its honesty. Say on camera that
the other sources are seeded for the demo — viewers trust a stated shortcut and distrust a
discovered one.

### What to build

```
production/video-guided-session/
└── northwind-demo/                 ← to be built; flow-cookbook format
    ├── docker-compose.yml          Postgres with the crm schema + 400 customers, 1.2k contracts
    ├── README.md                   setup order, the 3 stores, the LLM profile you must supply
    ├── 00-stores.md                flo_create_* calls for ingest_state, ontology_concept,
    │                               decisions_store, org_knowledge
    ├── ingest-crm.flow.json        the 6 flows as graph patches — NO credentials,
    ├── ingest-tickets.flow.json    settings profile ids left blank, exactly as the
    ├── ingest-assets.flow.json     cookbook convention requires
    ├── ingest-nightly.flow.json
    ├── ingest-webhook.flow.json
    ├── answer-request.flow.json
    └── broken/                     deliberately-broken variants for the teaching shots
        ├── double-run.flow.json        a node with two inbound edges and no join  (ep5)
        ├── silent-rule.flow.json       a Rule that returns undefined on one path  (ep5)
        └── many-scope-router.flow.json a Jev node on $.records[*]                 (ep6)
```

### Reset between takes

The entire install is one SQLite file.

```bash
cd <install dir>/flomorphic
docker compose down
cp ../northwind-demo.db data/flomorphic.db
docker compose up -d
```

Bake `northwind-demo.db` once, at the state each episode *starts* from. Seven snapshots, one
per episode, named `ep1-start.db` … `ep7-start.db` — so a reshoot of ep5 never requires
re-filming ep3.

## Capture settings — lock these before the first take

| Setting | Value | Why |
| --- | --- | --- |
| **Recording** | 2560×1440, downscale to 1080p on export | Canvas node labels are small; capturing at 1440 keeps them crisp after compression |
| **Theme** | **Dark**, fixed | There is a light/dark/system toggle in the header. Pick one and never touch it mid-series. Dark reads better on the canvas and in a dark presentation room |
| **Sidebar** | **Expanded** in ep2, **collapsed** for canvas work from ep5 on | Ep2 is the menu tour, so labels must be visible. After that the 60px rail gives the graph room |
| **Browser** | Chrome, no bookmarks bar, no extensions, no tab strip if possible | Kiosk or a clean profile |
| **Browser zoom** | 100%, never changed | A zoom change mid-series makes the edit look like a different product |
| **Canvas zoom** | Use **Fit** at the start of every graph shot | Reproducible framing, and it is a real button so it costs nothing |
| **Terminal** | 14pt minimum, light-on-dark, prompt shortened to `$` | A long `user@host:~/path$` prompt eats a third of the line |
| **Cursor** | Highlight/halo on, click-ping on | On a 1080p export a bare cursor is invisible |
| **Mouse speed** | Slow. Move, pause, *then* click | The single biggest readability lever in a UI screencast |
| **Audio** | 48kHz mono, −16 LUFS, noise floor below −60dB | — |

## Redaction list — read this before the first take

Three of these are live secrets.

| Never on camera unredacted | Appears in | Handling |
| --- | --- | --- |
| **The Extensions install one-liner** — *"embeds a live credential; treat it like a password"* | ep3, ep4 | Film it, then blur the token span in post. Or show it only in the collapsed card |
| `INFRA_CRED` in `.env.inflow` | ep4 | Truncate on screen: `INFRA_CRED=LS0tLS1CRUdJTiBOQVRTIFVTRVIg…` |
| The platform **API Secret Key** in `flomorphic/.env` | ep1 | Do not `cat` that file on camera. Show the directory tree instead |
| Provider token in a Node Settings profile | ep2, ep3 | The field masks it; do not click reveal |
| Webhook static token + `/hooks/<slug>` URL | ep5 | Use an obviously fake token, `demo-token-not-real` |
| LLM provider API key | ep2 onward | Set it before recording, never open that drawer field |

> **Simplest policy: film the entire series against a throwaway install, then destroy it.**
> Cheaper than frame-by-frame blurring, and it means a missed redaction is harmless.

## Visual assets

### Use the Snapshot button

The editor toolbar's **Snapshot** saves a PNG of the whole graph. That is your slide art — no
window cropping, no cursor, no chrome.

**But watch the aspect ratio.** Existing cookbook snapshots run **4322×756** and
**5056×1262** — 6:1 and 4:1. Unusable as a full slide. For a long flow, either:

- **Video:** a slow left-to-right pan at native resolution. Reads as following the flow, which
  is what the flow does.
- **Slides:** split into 2–3 labelled segments (*retrieve · decide · respond*), one per slide.

### Diagrams

The ASCII topology diagrams in
[the part index](../../book/08-flomorphic-guide/index.md#where-this-sits),
[ep5's orchestrator shape](../../book/08-flomorphic-guide/05-ingest.md#the-shape-and-the-constraint-that-produces-it)
and [ep6's flow](../../book/08-flomorphic-guide/06-decide.md#the-flow) should be redrawn as
real vector diagrams for the deck. They were written to survive in a terminal; on a slide they
should not.

### Naming

```
ep<N>-<shot>-<slug>.png        ep2-07-sidebar-build-group.png
ep<N>-<shot>-<slug>.mp4        ep5-12-nightly-run.mp4
```

Shot numbers match the episode scripts, so an edit decision list writes itself.

## Reference frames

[`screenshots/`](screenshots/) holds 17 frames captured from a live install with Playwright —
see [its README](screenshots/README.md) for provenance and the three chapter corrections they
produced. [`capture.js`](capture.js) reproduces them. They are correct for layout and labels;
the **data in them is real internal project data and must be reshot** against the Northwind
snapshots before anything ships.

## Shot notation used in the episode files

```
### 2.7 — The Build group · canvas · 0:45
**On screen** — what the viewer sees
**Do** — the action to perform, in order
**Say** — verbatim narration (blockquote)
**Watch out** — prep, redaction, or a take that commonly fails
```

Durations are *target*, and they sum to the episode budget. Narration is written to be read
aloud at 150 wpm — if a block runs long against its duration, cut the narration, not the
pause.

## Pre-flight, every shoot day

- [ ] Restore the episode's start snapshot, `docker compose up -d`, confirm the header badge
      reads **Connected**.
- [ ] **Settings → Engine resources** lists at least one resource. If it is empty, nothing in
      ep5 or ep6 will run.
- [ ] LLM provider profile exists and has been used once today (so the first take is not also
      the first API call).
- [ ] Postgres container up; the plugin's ping check passes.
- [ ] Theme dark, zoom 100%, sidebar in the state this episode calls for.
- [ ] Audio level check — 30 seconds of room tone recorded for noise removal.
- [ ] `history -c` and a fresh terminal.

## Open production decisions

| Decision | Status |
| --- | --- |
| Ep3–7 scripts | **not started** — ep1 and ep2 are written verbatim and filmable; ep3–7 follow the same shot notation |
| `northwind-demo/` built | **outstanding** |
| Broken-flow variants built | **outstanding** |
| The deck | **outstanding** |
| Trailer / cold open | not planned; ep4's *"the assistant now knows the plugin exists"* and ep6's park-and-resume are the two candidate payoff shots |
| Captions / transcript | recommended — the narration scripts here are the transcript, so this is nearly free |
