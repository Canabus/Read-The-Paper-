# Read the Paper — audit fixes, 2026-11-26

Build: artifact `contract-reader` v29 (was v27). `app.html` in this repo is a
byte-identical copy of the v29 CVM build
(sha256 `9079bee2ef32f1942be432f010da342f112f5c2628c810e8e0035fbf6ef023a5`).

Every claim below was verified against the actual v27 code before anything
was changed. Outside reviews were treated as audit material only. Per Nate's
instruction (2026-11-26), outside AIs' product-direction recommendations
were NOT adopted — no narrowing to the contract reader, no Books-mode
changes, no missing-clauses module, no exact-ask feature, no pricing
changes, no front-door redesign, no modes removed. Only verified bugs and
honesty defects were fixed.

## 1. Source location in packet mode — claim VERIFIED, fixed

Claims checked: (a) `pageOf()` returned the last `[Page N]` marker anywhere
before the passage, ignoring `[Source: X]` boundaries — a TXT/OCR passage
after a 3-page PDF was stamped "p. 3". TRUE. (b) Page labels never included
a document name. TRUE. (c) `buildSourceViewer`/`buildLocIndex` expected
`[Page n] [Source: x]` together, but the packet writes `[Source: x]` on its
own line, so body paragraphs got no source and the fallback credited the
first file containing the first 50 characters. TRUE in effect.

Changed:
- Page lookup is segment-aware (`findNeedleIndex` / `segmentFor`): page
  markers count only inside the passage's own `[Source: X]` segment. A
  passage whose segment has no page marker now reads
  **"location not available"** (the old "page unavailable" string is gone
  everywhere) instead of a borrowed page.
- All 16 citation spans now carry the document name in packet mode
  ("p. 2 · from contract.pdf" / "location not available · from notes.txt");
  single-document mode still reads "· from your document" and PDF
  `[Page N]` locations work exactly as before.
- `buildSourceViewer` carries the source forward (a marker-only line sets
  the source and is not stored as body text; the carried page resets when
  the source changes), so `buildLocIndex` populates document + page.
- `sourceFileFor` fallback credits a file only when exactly one packet file
  contains the probe text; shared boilerplate is credited to neither.
- Extra find during verification, fixed in scope: the sentence splitter
  breaks `[Source: notes.txt]` at the "." in the file name; `segmentFor`
  now takes the last `[Source:]` marker starting at or before the passage,
  so that fragment resolves to the right file.

Not changed: `packetCombinedText()` format (the carry-forward fix made a
format change unnecessary).

Known limit (Item B, duplicate sentences — claim PARTIALLY VERIFIED): the
passages-to-inspect list carries extraction-time occurrence and tracks each
copy of a repeated sentence separately, and segment-aware lookup respects
that. Other sections use `occFor()`, which resolves to the first
occurrence; it was deliberately NOT made to increment (a sentence quoted in
several sections but occurring once would then hunt for copies that don't
exist). Known limit, in plain words: when the exact same sentence appears
more than once in a document, some sections point at the first copy's page;
the passages-to-inspect list tracks each copy separately.

## 2. Backup restore validation — claim VERIFIED, fixed

Claims checked: validation was container-shape only (`storeShapeOk`);
`payload.version` was written on export but never checked; the import file
input was never reset; a partial backup mixed restored data with data
already on the device, and stores empty at export were skipped, not
cleared. All TRUE. There is no `confirm()` pattern anywhere in the app, so
none was added.

Changed:
- Deep validation before writing (`cleanStore`): bills entries that aren't
  plain objects are dropped; a kept bill's missing/non-object `notices` is
  replaced with the blank-notices default so the tracker can't crash on
  `bill.notices[n.key]`; non-object wrong-finding report entries dropped;
  Who/Where keeps its object check and drops non-string values and
  non-object manual entries. Drops are counted and reported in the status
  message ("N damaged entries were left out — everything else restored.").
- Version gate: a backup with a missing, non-numeric, or > 1 version is
  refused with a plain newer-version message and nothing changes.
- The import input is reset after every finished attempt (success, refusal,
  unreadable file, reader error), so picking the same file again works.
- Clean replace: all three store keys are snapshotted; keys present in the
  backup are written (cleaned), keys absent from the backup — including
  stores that were empty/null at export — are removed, so the device
  matches the backup instead of mixing. Rollback on a write error restores
  all three keys. The existing parse / kind-check / validate-before-write /
  rollback skeleton was kept.

## 3. Packet order / FILE_HASHES — claim VERIFIED, fixed

Claims checked: `FILE_HASHES.push` ran inside the async
`arrayBuffer().then()` chain, so hash order scrambled against the order
files were added; `handleFiles()` never reset `FILE_HASHES` (only Clear
and Wipe did), so stale hashes rode into the next reading's fingerprint
line; the packet header said "in the order you selected them," which
overclaims (a browser picker may hand files over alphabetically). All TRUE.

Changed:
- A hash slot is reserved synchronously for each file, in the order files
  were added, and filled when its digest resolves — order can't scramble.
- `handleFiles()` resets `FILE_HASHES` when a batch starts with an empty
  packet (a new packet); adding files to an existing packet keeps the
  earlier hashes, because those files are still in the packet.
- The fingerprint line skips unfinished slots and prints each file's name
  with its first-16 hex, so a fingerprint traces to its file.
- Packet header copy now reads "in the order they were added."

Not changed: no reorder control was added (not required this pass).

## 4. Dates and .ics export — claims VERIFIED, fixed

Claims checked: no `DTSTAMP` in the .ics (RFC 5545 requires it); UID was
the list index, so re-importing duplicated events; a date the user marked
"Not a deadline" still exported, titled "Reminder you confirmed — Not a
deadline"; "March 5th, 2026" and "5 March 2027" were never detected;
"25/12/2026" matched the numeric pattern but failed `new Date()` and was
silently dropped. All TRUE.

Changed:
- One shared date pattern (`DATE_PATTERN_SRC` / `makeDateRe` /
  `parseDateText`) at all three detection sites (gap check, obligations,
  timeline). It now also matches ordinal forms ("March 5th, 2026",
  "March 5th 2026") and day-first month-name forms ("5 March 2027",
  "5th March 2027"); ordinal suffixes are stripped for parsing only — the
  label shown stays exactly as found in the document.
- .ics: `DTSTAMP` (UTC) added to every VEVENT. UID is now a content hash —
  `rtp-<hash>@readthepaper` over date + label + meaning + sentence — so
  re-importing the same export dedupes instead of duplicating, and two
  different dates never share a UID.
- A date whose meaning is set to "Not a deadline / not established" no
  longer exports at all. If every ticked date is skipped for that reason,
  nothing is exported and a toast says plainly that no calendar file was
  made.

Deliberately NOT changed: numeric-date order. `5/3/2026` is still read as
May 3 (US month-first). The only exception: when US parsing is impossible
and the first number is > 12 and the second is ≤ 12 (e.g. `25/12/2026` →
December 25, 2026), the date is read day-first instead of being dropped —
that case is unambiguous. Ambiguous numeric dates are never swapped.

## 5. "Language to inspect" count — claim VERIFIED, narrow fix only

Claim checked: "at any time" is in the power-phrase list, so harmless
wording like "you may cancel at any time" inflated the count. TRUE.

Changed: only the headline passage count (`powerSents`). A sentence whose
ONLY power-phrase hit is "at any time" and which matches a you-side
cancellation pattern ("you may cancel…", "the customer may terminate this
plan at any time") is no longer counted. The phrase list and the
phrase-match tally are untouched, and their-side "at any time" sentences
still count.

Not changed: the YOUR side / THEIR side section was not redesigned. A
review's suggestion that the tally still feels like a scoreboard is noted
and was NOT acted on — that's a product call for Nate, not a bug.

## 6. Packet Remove left stale findings — claim VERIFIED, fixed

Claim checked (third review): the Remove handler only filtered the packet
and re-rendered the list; a removed file's findings stayed on screen.
TRUE.

Changed: removing a packet file also drops its `FILE_HASHES` slot. If a
reading is on screen, it is cleared (results, trust results, and export
bar hidden; reading state reset; paste box set to the remaining packet
text) with the toast: "Removed <name>. The reading on screen was cleared —
run the packet again to get findings for what's left." No silent re-run.
With no reading displayed, Remove behaves as before (fixture-verified: no
toast in that case).

## C. Builder wrapper / tunnel shim — claim PARTIALLY VERIFIED, documented, bundle NOT altered

Claim checked: the shipped `app.html` is the CVM build, and its prelude
overrides fetch and XHR. Read directly in the shipped file:

- TRUE: `installTunnel()` runs unconditionally and replaces `window.fetch`,
  `XMLHttpRequest`, `WebSocket`, `EventSource`, `navigator.sendBeacon`,
  and `URL`, and rewrites URL attributes in authored HTML/CSS.
- NOT TRUE that it proxies network calls: `tunnelFetch` and `TunnelXHR`
  delegate every external URL (any absolute scheme, protocol-relative,
  `data:`, `blob:`) straight to the native fetch/XHR with the original
  arguments. Only same-origin requests — URLs under the page's own base
  directory — are routed through a `MessagePort` to the hosting shell,
  which is how the Muse sandbox serves its own asset files.
- The app's own code makes no fetch/XHR calls for user documents; files
  are read locally via FileReader / pdf.js / Tesseract in the page. No
  document content passes through the tunnel. The "nothing leaves your
  phone" copy remains literally true for user documents.
- Standalone caveats, stated plainly: on GitHub Pages there is no host, so
  the port never resolves — a same-origin fetch through the shim would
  hang rather than fall back (the app issues none). The shim's Worker
  wrapper only accepts shim-registered blob URLs, a compatibility risk for
  CDN-spawned workers (pdf.js points its worker at a CDN URL; pdf.js has a
  main-thread fallback). The file contains no service-worker registration
  at all, so PWA install limits are not caused by the shim.
- Decision: the artifact system produces only the CVM build, and the
  bundle was NOT hacked apart. The builder confirmed the build ships the
  standard prelude unchanged and offers no standalone no-tunnel variant
  switch. Whether the privacy copy needs any adjustment on this basis is
  the parent's/Nate's call; no copy was changed in this pass.

## Verification

Builder ran a Node fixture harness against the actual shipped functions
(66 assertions, all passing; `node --check` clean), with fully fictional
fixtures only: packet location (PDF clause → correct page + file name; TXT
clause → "location not available" + its file name, never the PDF's page;
single-PDF mode unregressed; duplicate sentences p. 1 / p. 2 in the
occurrence-aware path), corrupted-backup restore (bad entries dropped and
reported, valid bill's notices repaired, version-99 backup refused, input
reset and re-trigger verified, absent keys removed), .ics export (DTSTAMP
present, stable content-hash UIDs, not-a-deadline date produces no event),
date detection (all new forms found; `5/3/2026` still May 3), Fix 5
cancellation sentences count 0 while their-side sentences still count, and
Fix 6 remove-with-reading clears + toasts while remove-without-reading
does not. Grep checks: zero "page unavailable" left, zero `confirm(` added,
exactly 3 shared date-pattern call sites, packet copy reads "in the order
they were added."

Note on the artifact's public muse.ai link: the v29 build is complete, but
the shared link still serves the previously approved version until a new
share is approved. The GitHub Pages copy below is the updated one.
