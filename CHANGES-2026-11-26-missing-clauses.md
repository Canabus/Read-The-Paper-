# Read the Paper — missing-clauses module v1, 2026-11-26

Build: artifact `contract-reader` v31 (module built in v30, one correction in
v31). `app.html` in this repo is a byte-identical copy of the v31 CVM build
(sha256 `e2a1d69f9db2324d16fea45f82ef5794e9f204065bfb667e3b0a93ffe2fd8426`).
Base was v29 (commit bcfe5bf); no v29 fix was regressed — the builders'
harnesses re-ran the v29 fixture paths (segment-aware locations,
"location not available" ×8 in the shipped file, restore, hashes, .ics,
Remove) and the only source edits are the ones listed below.

Built to the binding scope:
`missing-clauses-scope-2026-11-26.md` (goal files), with Nate's approved
defaults: section name **"What isn't in here"**, FULL checklists, the
one-line question per listed item ships in v1.

## What was built

- A new sub-section, "What isn't in here", inside the existing results
  Section 5 ("Gaps, language & dates") — a `#missingOut` block in the
  results DOM, rendered by `renderMissingClauses` from the analyze path
  only. Trust mode never calls it; Books mode untouched. No new mode, no
  new screen, no front-door change, no pricing change.
- Full checklists keyed by the document type the app already detects:
  - General (every agreement): governing law, notices, assignment limits,
    amendment, termination, dispute path.
  - Loan / credit: right to cure before acceleration, cure period length,
    liability cap, prepayment terms, rate cap.
  - Lease: right to cure before eviction, entry notice, renewal terms,
    deposit return deadline.
  - Bill / invoice: dispute window, how to dispute, late-fee cap.
- Three states only:
  - **Found** — not listed in the section; the passage lives in the normal
    findings with quote and location.
  - **Not found** — listed under "Not found in what you gave it." The
    section never asserts the full legal document lacks a clause.
  - **Couldn't check** — listed separately, never merged with or counted
    as not found. Triggers: fragment/short text, any packet file's OCR
    average confidence under 70%, a packet file with no readable text, or
    an unidentified document type (general list only, and the section says
    plainly the type wasn't identified).
- Every listed item (both lists, after the v31 correction) carries: the
  clause name in plain words, one plain sentence on what the clause would
  have done, and the one-line question to ask in writing. No legal advice,
  no sign/don't-sign steering.
- No score, no tally: the rendered section contains no count of missing
  items anywhere (harness-checked: no digit + "not found"/"missing"
  pattern), and the header carries the line that this is "a checklist of
  clauses documents like this commonly state — not every clause a document
  could have."
- Found verdicts trace through the v29 segment-aware machinery
  (`findNeedleIndex`/`segmentFor`/`docOf`): a clause is credited only to
  the packet file it actually sits in — never a borrowed location.
- Exports: because the section sits inside the results section,
  `buildExportMarkdown`/`buildCaseHtml` include it verbatim with the same
  wording/framing, tied to the reading's fingerprint line at the top of
  the results like the rest of the export.

## Correction made during this run (v31)

The v30 build's own attestation flagged that Couldn't-check items rendered
without their ask line (the fictional bill fixture's "late-fee ask present"
check failed). Fixed in v31 with a single-line change: the Couldn't-check
list now renders `miss-ask` exactly like the Not-found list. Re-run: the
fragment fixture's 11 Couldn't-check items now render 11 ask lines
(previously 0).

## Fixture results (fully fictional fixtures only)

Run by the builders in Node harnesses against the shipped functions
(40 checks in the v30 harness, all passing; v31 harness re-ran the set):

1. **Loan WITH cure clause** ("thirty (30) days after written notice to
   cure any default before Lender may accelerate", plus the other
   general/loan clauses present): type detected "loan / credit"; cure and
   cure-length NOT listed; Couldn't-check empty. PASS.
2. **Loan WITHOUT cure language**: right-to-cure listed under Not found
   with the scope's exact would-have-done sentence and ask-in-writing
   line; cure-length follows it; Couldn't-check empty. PASS.
3. **Partial / low-OCR text**: a ~90-character fragment → nothing under
   Not found, general items under Couldn't check with a stated reason; the
   full loan text as a one-file packet whose note carries "average OCR
   confidence 52%" → nothing under Not found, cure under Couldn't check,
   reason cites the OCR confidence. PASS.
4. **Packet, clause in file 2** (loan-main.txt without cure + rider.txt
   with it): cure/cure-length found, not listed; `docOf` on the cure
   sentence returns "rider.txt", not file 1. PASS.
- Extra render checks: lease fixture lists Entry notice and Deposit
  return deadline as Not found with ask lines; bill fixture lists Dispute
  window, How to dispute, and Late-fee cap (late-fee ask present after the
  v31 correction); unidentified-type agreement prints the plain
  type-wasn't-identified note and evaluates the general list only.

## Deliberately NOT done

- Exact-ask module (send-ready negotiation sentences beyond the one-line
  question) — still parked, not part of this build.
- No narrowing of the app, no Books-mode changes, no pricing changes, no
  new modes, no front-door changes, no reorder control.
- The muse.ai public link was NOT re-shared: it still serves the
  previously approved version. Publishing v31 there needs Nate's approval
  via a new `artifact.share`.
- Known limit carried over from v29, unchanged: when the exact same
  sentence appears more than once in a document, some sections point at
  the first copy's page; the passages-to-inspect list tracks each copy
  separately.
