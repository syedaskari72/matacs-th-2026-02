# Write-up: MATACS-TH-2026-02

**Core idea:** AI only reads each proposal and copies out facts, with quotes as evidence. Plain code checks the rules, does the maths, ranks and logs everything. A person makes every final decision. The flow is in `diagram/process-flow.png`.

## 1. AI vs. rules

**AI (one step):** extracting facts and evidence quotes from each proposal. Proposals are free-form PDFs, and reading messy text is what AI does well.

**Rules/code (everything else):** upload checks, text extraction, loading admin criteria, validating the AI output, mandatory pass/fail checks, hard limits (budget, 16 weeks), scoring, ranking, tie-breaks and reason text.

**Why the line is here:** AI turns text into facts; rules turn facts into decisions. The AI never sees the weights, never scores and never ranks, so a persuasive or manipulative proposal has nothing to influence. One rule drives the outcomes: an explicit failure is *ineligible*; missing, unclear or suspicious information is *flagged* for a human, never guessed.

## 2. Consistency and explainability

- Scoring is deterministic code: fixed formulas on extracted numbers. Each proposal is scored independently, so adding or removing a bid never changes another bid's score.
- Extraction runs at temperature 0 with a strict JSON schema and is cached by file hash + prompt version + model version. Ranking the same files twice reuses the cache, so the result is identical. A fresh AI re-run could differ slightly, so extraction runs twice and any field that disagrees is flagged.
- The reviewer sees a score per criterion, the quoted evidence with page number for every fact, and a one-line reason generated from a template that reads the scores, so words cannot contradict numbers.

## 3. Human in the loop

- **Review screen:** ranked, flagged and ineligible lists, each fact linked to its PDF page. The reviewer can correct a value, clear a flag, request clarification from a bidder, or override a position or exclusion. Every change needs a typed reason and triggers instant rules-only re-scoring.
- **Verification:** for finalists, the reviewer checks the actual licence, WSIB and insurance documents. The tool confirms they are *stated*, not *true*.
- **Award:** a person decides. The tool never awards.
- **Audit log (append-only):** uploader, file hashes, criteria version, prompt version, model and settings, raw AI output, validated fields, flags, scores, each human change (who, when, old to new, reason) and the final decision.

## 4. Changing criteria

- Criteria are versioned. An edit creates a new version (who, when, why logged). Old versions are never altered, and every score records the version it used.
- Scored proposals are marked "scored under v2, criteria now v3" and are not silently changed. The reviewer clicks Re-score. Extracted facts are stored separately from scoring, so new weights need only a rules re-run, with no AI cost. If a new criterion needs a fact not yet extracted, only that field is re-extracted.
- The new run is saved beside the old one, showing rank changes and eligibility changes (for example from a new mandatory criterion). Weights must total 100 or the save is blocked. A finalised award stays locked; changing it needs a new decision with a reason.

## 5. Risks and safeguards

- **Wrong extraction (misread PDF, OCR error):** each quote must exist in the source text (string match); schema and sanity checks; double extraction; failures flagged; the reviewer sees the PDF page.
- **Made-up facts:** "null if not stated" rule; no quote means no value; gaps are flagged, never filled (see D).
- **Manipulation / prompt injection:** proposal text is treated as untrusted data; the AI has no scoring power; any instruction aimed at the AI is logged and ignored, and the proposal is held for human review (see C).
- **Bias** (favouring large, polished or well-known firms): scoring uses only extracted facts and fixed formulas, not writing quality, and is blind to bidder name. Tool results and human decisions are compared periodically.
- **Over-trust in the ranking:** it is advisory; evidence is one click away.
- **Cost:** one AI call per proposal (two for the double check), cached, with page limits. Re-scoring is free.
- **Confidentiality:** a provider that does not train on inputs, role-based access, retention limits.

## Sample set result

| | Exp 30 | Price 25 | Time 15 | Warranty 15 | Team 15 | Total |
|---|---|---|---|---|---|---|
| A | 30.0 | 10.6 | 7.5 | 12.5 | 15.0 | **75.6** |
| E | 24.0 | 8.4 | 5.6 | 13.5 | 15.0 | **66.5** |

- **A (1st) and E (2nd):** both pass every mandatory check. A leads on experience, price and speed; E wins slightly on warranty.
- **B, ineligible:** the electrical work is subcontracted and the licence is "to be confirmed", so the bidder does not hold it. Being cheapest and fastest cannot offset a mandatory failure.
- **C, flagged:** contains a hidden instruction aimed at the reviewing AI. It is ignored and the bid is held for human integrity review. Judged on legitimate content only, it would score about 58, below A and E.
- **D, flagged:** no completion time is given, so the 16-week requirement and timeline criterion cannot be assessed. I would request clarification rather than guess. Its final position would be 3rd at 16 weeks, 2nd at 12 weeks, and 1st only under about 10 weeks.

## Assumptions

- "Since 2021" counts as inside the five-year window.
- Scoring anchors are my choices, admin-editable: full marks at 10 qualifying installs; price 20% under the ceiling (zero at ceiling); 8 weeks (zero at 16); warranties of 25 years (panel) and 15 years (workmanship); team = four-item checklist (named lead, crew size, safety plan, monitoring plan).
- Under other reasonable anchors A still ranks above E.
