What this is
A monthly paid-media performance report (Meta + Google Ads,school-admissions) delivered as a single self-contained HTMLfile that behaves like a Google Slides deck: 14 fixed 1920×1080 slides,keyboard navigation, filmstrip, grid overview, and print-to-PDF that outputsexactly one full-bleed slide per page.

The defining feature: the entire deck renders from one data object. Everyfigure, label, list and sentence is data-driven, so next month's report is adata update, not a code update. Three update routes, all functional:

Google Sheets fetch (primary — see contract below)
JSON file import/export
In-browser Edit mode (click any value; lists accept add/remove)
Hard rules for any change you make
Never hardcode a number, label, or sentence into a slide template. Everydisplayed value must come from DATA via the helper functions below. This isthe core contract of the project.
One file. No build step, no server, no external assets except GoogleFonts + Lucide CDN. Keep it that way.
Slides are fixed 1920×1080. Content clips, never reflows. New contentmust fit inside the padding (84px top, 96px sides, ~120px bottom clearancefor the footer).
Computed values are never stored — they live in calc() only.
If you change the data schema in a breaking way, bump LS_KEY(citizen-report-v1 → v2) or stale localStorage will break rendering.
After editing list-rendering code, check the ADDERS registry (below) stillhas an entry for every "+ Add" button path.
Runtime flow (read this before touching anything)
DEFAULT_DATA  ──load()──▶  DATA  (localStorage if present, else defaults)      │      ├── calc(DATA)      → derived figures (totals, %, counts, earliest date…)      ├── buildSlides(DATA, calc) → [{t: title, h: html, dark: bool}, …]      ├── renderAll()     → deck HTML + filmstrip + grid thumbnails      ├── playSlide()     → stagger-reveal animation on slide entry      └── fit()           → scales #viewport so 1920×1080 fits the window
renderAll(animate) re-renders the whole deck and is called after everydata mutation (edit, import, sheet fetch). That's intentional — state lives inDATA, never in the DOM. Slides are stateless templates.

Data schema (complete)
{  // meta / identity — cover, footers, topbar  brand, docType, period, month, year, channels,  coverTitle, agency,           // cover slide only  dek,                          // cover intro (legacy; cover v2 doesn't render it)  planTagline, thanks, closingNote,  spend: {    dek,                        // chapter 01 subtitle    google, meta,               // AED, plain numbers — drive the channel split bar    googleParts: [{label, value}],   // breakdown; % scales within the group    metaParts:   [{label, value}],  },  stats: {    meta:   {dailySpend, leadCapture, whatsapp, cpl, cplDir},    google: {dailySpend, leadCapture, whatsapp, cpl, cplDir},  },  // cplDir: "above" renders "> 500 AED", "below" renders "< 500 AED"  metaChannel: {    tagline, analysis,          // chapter head + lead paragraph    facilitiesRange, facilitiesNote,   // big callout on language slide    campaigns: [{date, type, name}],   // date = "DD.MM.YY" plain text (string-sorted                                       // for "earliest launch"; never let Sheets                                       // reformat as a locale date)    languageSplit: {      english: {leads, cpl, cplDir, note},      arabic:  {leads, cpl, cplDir, note},    },    assets: [{name, hook, leads, ctr, cpl}],   // array order = display rank    aud: {      top: {items: [string], leads},      low: {items: [string], leads},    },    summary: [string],    actions: [{title, body}],  },  googleChannel: {    tagline, analysis, pull,    // pull = italic pull-quote    campaigns,                  // NUMBER, not array (count only)    structure,                  // string, e.g. "5 Search · 1 PMAX"    insights: [{k, v}],         // k = bold lead-in, v = sentence    assetNote: {label, value},  // sitelinks callout    actions: [{title, body}],  },}
Computed in calc() — do not duplicate in DATA
total spend, channel %, blended cost/engagement, lead+traffic+other campaigncounts (regex /lead/i, /traffic/i, else "other"), earliest launch date,max hook rate, % deltas, largest breakdown line, takeaway sentences.

Editable-element system
Templates render values through helpers that attach data-path (dotted pathinto DATA):

edT('spend.meta') — editable text
edN('spend.meta') — editable number (with animated .cnt counter)
roN(v) — read-only number (computed values)
cpl('stats.meta.cpl','stats.meta.cplDir') — the ">/< AED" chip with anabove/below toggle button
Edit mode flow: click .ed[data-path] → inline input/textarea → blur commitsvia commitValue() (coerces to number if the original was a number) →setPath() → persist() → renderAll(false). Lists add/remove viaADDERS registry (path → factory returning a new element) and[data-rm="path"][data-idx] buttons. Any new editable list needs an ADDERSentry or the "+ Add" button breaks.

Slide inventory (14)
#	Template fn	Content
01	slCover	Dark branded cover (black bg, orange dot-square logo SVG, centered title, Period/Channels, "IN GOOD COMPANY" bottom-right). Detected in CSS via .slide:has(.cv-root).
02	slContents	TOC — slide ranges are hardcoded here; update when adding/removing slides
03	slSpend	Ch.01 head, channel split bar, per-channel breakdown tables
04	slScore	Ch.01 scorecard, Meta vs Google side-by-side + takeaway
05	slMetaOv	Ch.02 head, lead paragraph, stat band
06	slMetaCamps	Campaign table + lead/traffic/other distribution bar
07	slMetaLang	Facilities callout + EN/AR split panel
08	slAssets	Ranked asset cards with hook-rate bars
09	slAud	Top / under-performing audiences
10	slMetaSum	Report summary bullets
11	slGogOv	Ch.03 head, lead paragraph, pull quote, stat band
12	slGogIns	Numbered insights + sitelinks callout
13	slPlan	Ch.04, Meta vs Google action columns
14	slThanks	Dark closing slide
Headers: ch(no,title,dek,tag) for chapter-opening slides, eh(no,sec,title,note)for continuation slides. Footer is appended automatically by foot(i,n) —never put it in a template.

Deck mechanics
go(i) — clamp + syncChrome() (counter, filmstrip, progress bar) + playSlide()
playSlide() — removes .static, staggers .rv/.grow reveals(~60ms/element), animates .cnt counters. Grid thumbnails force-finalizevia CSS overrides instead.
fit() — scales #viewport to the stage rect; called on resize + load.
Keyboard: ← → Space PgUp/PgDn Home End; G toggles grid; Esc closesoverlays. Guarded while an input/textarea is focused.
Print: @page {size:1920px 1080px; margin:0} + print CSS unmounts allchrome, prints every slide sequentially. finalizeCounts() runs onbeforeprint so counters show final values. User must enableBackground graphics in the print dialog — that's in the human docs.
Google Sheets contract (primary data source)
Fetches each tab as CSV via the gviz endpoint:https://docs.google.com/spreadsheets/d/{ID}/gviz/tq?tqx=out:csv&sheet={Tab}(requires link-sharing "Anyone with the link — Viewer"; domain-only fails CORS).

Nine tabs, names case-sensitive, row 1 always the header row:

Tab	Columns	Maps to
Config	key, value	dotted-path keys (spend.google, stats.meta.cplDir…) via setPathOn; autoType coerces "21,450"→21450
SpendLines	channel, label, value	spend.{google,meta}Parts
MetaCampaigns	date, type, name	metaChannel.campaigns
MetaAssets	name, hook, leads, ctr, cpl	metaChannel.assets
LanguageSplit	language, leads, cpl, cplDir, note	languageSplit.{english,arabic}
Audiences	group(top/low), audience	aud.top.items / aud.low.items
MetaSummary	note	metaChannel.summary
GoogleInsights	k, v	googleChannel.insights
Actions	channel(meta/google), title, body	actions per channel
sheetToData() builds from blankData() (a zero-filled skeleton) then fillsfrom tabs — any new schema field must be added to blankData() too, andideally to the Apps Script generator (createReportSheet, kept in/scripts or docs — regenerates the whole spreadsheet with sample data).Missing tabs are tolerated (toast lists them); missing Config rows fall backto zeros/defaults. The sheet URL is remembered in localStorage keycitizen-report-sheet-url.

google sheet - https://docs.google.com/spreadsheets/d/1Cwf8B_QZyuz5Wvw8B3vF3R5OrmSgLZ8p2pu7IEChp8M/edit?usp=sharing

Design tokens
--paper:#F3EFE4  --paper2:#E9E2D0  --ink:#1C1913  --ink2:#6C6452--line:#D9D1BE   --line2:#BCB29A   --meta:#C13A12 (Meta accent / progress /                                                  takeaway — vermillion)--brand-orange:#FF6B00            (cover logo square only; approximated —                                   client may supply exact hex)Fonts: Fraunces (display serif, uses font-variation-settings SOFT/WONK),Archivo (sans), Spline Sans Mono (all figures — never set numbers in the serif)Icons: Lucide via CDN, initialized with lucide.createIcons() after every render
Channel color language is Meta = vermillion, Google = ink — used in swatchsquares, bars, and the split stack. Don't introduce new accent colors.

Common tasks
Add a data field → ① DEFAULT_DATA ② blankData() in sheetToData ③generator script's Config array (or a tab) ④ render with edT/edN ⑤ if thefield must survive old localStorage, guard in the template:if(!d.coverTitle)d.coverTitle='…' (see slCover).

Add a slide → ① new slX() template fn ② push into buildSlides()③ update hardcoded slide ranges in slContents ④ keep 14→N consistent(foot(), counter, grid all derive from array length automatically).

Restyle → tokens in :root; slide canvas is .slide (padding 84/96/120);:has() cover selector needs modern browsers (Chrome 105+, FF 121+).

Debug a blank/broken deck → check localStorage for stale schema (Resetbutton clears it), check calc() for division-by-zero on empty lists(||1 guards exist — preserve them when editing).

Gotchas
September has 30 days — sample period reads "1 – 30 Sep 2026" on purpose.
googleChannel.campaigns is a count, unlike Meta's campaign array.
Campaign type matching is regex-based (/lead/i, /traffic/i) — "General"counts as "other" and renders a dashed tag.
Edit mode re-renders without animation (renderAll(false)) — that's fine;never try to preserve DOM animation state.
The cover's dek field is no longer rendered (cover was redesigned) butkept in schema for JSON-export compatibility.
localStorage is the persistence layer; file:// protocol works everywhereexcept some Safari private-mode cases — Edit-mode saves silently fail there(wrapped in try/catch already).
Two notes on using it:

