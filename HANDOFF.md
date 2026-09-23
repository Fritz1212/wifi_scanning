# HANDOFF — Cross-Domain Wi-Fi CSI Perception

Status of this project, what is real, what is not, and the exact next steps.

---

## 1. What exists right now

| Artifact | Path | State |
|---|---|---|
| Manuscript (IEEEtran, two-column) | `paper/main.tex` | complete draft of method + related work + protocol. **No measured results.** |
| Plain-text preview (read without TeX) | `paper/PAPER_PREVIEW.md` | the full draft rendered as markdown |
| Verified bibliography | `paper/references_verified.bib` | 49 entries, every field fetched from Crossref/arXiv/DBLP/OpenAlex |
| Reference audit | `references_report.md` | corrections, the unverifiable item, the added citations |
| Verification report (machine) | `notes/verification.json` | per-reference pass/flag record |
| Abstracts (grounding) | `notes/abstracts.json` | fetched abstracts for Related Work |
| Reference scripts | `scripts/` | re-runnable verification + lint pipeline |
| Experiment runner | **does not exist yet** | see section 4 |

### The honest summary

There are **no experimental results**. Not in this workspace, not in the
conversation history, not in any results directory. The only material supplied was the
reference list. Consequently:

- Every result slot in `main.tex` is a red `[RESULT NEEDED: ...]` placeholder. There are 7.
- Every design choice that is still open is a blue `[DECISION: ...]` marker. There are 10,
  plus 4 `[TODO]` notes.
- No accuracy, F1, or error number anywhere in the manuscript was invented. The only
  numbers in the text are quoted from prior work and attributed to it in the same sentence.
- The claim–experiment map (Table I) is written so that each claim states, in advance, what
  would falsify it. That table is the contract this draft is built around.

---

## 2. The contribution, in one sentence

> Treat the environment in cross-domain Wi-Fi CSI sensing as *retrievable context* rather
> than nuisance: a self-supervised objective factorises a CSI window into a content code
> and a domain code, and a geometry-indexed memory conditions the classifier on the space
> the measurement came from — so a new room is handled by inserting an entry, not by
> retraining.

Method name in the draft is a placeholder: **`\method` = "Relay"**. Rename it with one
`\newcommand` edit in the preamble once you decide.

---

## 3. Open decisions that need you (not me)

These are marked `[DECISION]` in the manuscript. I did not guess them because guessing would
misrepresent your work.

1. **Dataset(s) and split protocol.** The draft says Widar3.0 is the natural choice: its
   domain factors already separate environment / location / orientation, which is exactly
   what claim C1 needs. SignFi, Wiar, CSIDA and XRF55 are the alternatives. Decide before
   running anything, because the split determines the baselines.
2. **Independence mechanism.** HSIC penalty vs. adversarial domain classifier. The draft
   recommends the adversarial variant because it matches Kang et al. (2021), the closest
   prior work on CSI disentanglement, and is easier to tune; HSIC is more stable but adds a
   kernel-bandwidth hyperparameter. Pick one to keep the paper tight.
3. **What the domain descriptor $\rho_m$ actually contains.** This is the load-bearing
   design choice for the whole retrieval idea: is it a geometric signature derived from
   signal statistics, installation metadata entered by the installer, or both? If it needs
   manual entry, the "no extra effort" claim weakens and you must say so.
4. **Target adaptation budget.** 1-shot or 5-shot for the comparison against domain
   adaptation baselines (C3). FewSense reports 5-shot numbers, CrossFi reports 1-shot and
   zero-shot, so match whichever you compare against.
5. **Statistics.** ≥5 seeds; paired bootstrap (10k resamples) for CIs; McNemar's exact test
   for paired method comparisons. Confirm this is what your venue expects.
6. **The name.** "Relay" was chosen because retrieval *relays* the room context to the
   classifier. It is a placeholder.

---

## 4. Experiment plan (runnable, ordered, with falsification built in)

Nothing here is executed. This is the sequence that produces the paper.

### Stage A — Baselines and the naive failure (establish the problem)

```
experiments/
  datasets/        loader per dataset; produce (csi, label, env, location, orientation)
  encoders/        shared backbone used by EVERY method (fair comparison)
run_supervised.py  train on source domains, evaluate on each held-out factor
analyze.py         aggregate over seeds, CIs, McNemar
```

Deliverable: the naive supervised baseline's cross-domain collapse, per factor. This is the
number that motivates the paper and is *missing* from the current draft.

Gate: if the naive baseline does not collapse on your chosen split, the premise is wrong for
that split — change the split before continuing.

### Stage B — Disentanglement (claims C1)

`run_disentangle.py` with the objective in `main.tex` Eq. 4.

Required evidence:
- linear probe of the label from $\zc$ (should be high)
- linear probe of *environment identity* from $\zd$ (should be high) **and** from $\zc$ (should be near chance)
- the same two probes for (a) a joint encoder with no split and (b) an adversarially
  invariant encoder

Gate: if $\zc$ still predicts the environment strongly, the split is not working and C1
fails — report it rather than hiding it.

### Stage C — Retrieval (claims C2, C5)

`run_retrieval.py`, identical weights to Stage B with the memory enabled/disabled.

Required evidence: accuracy with and without the memory; the same comparison against a
self-supervised encoder scaled to equal pretraining data (this is what separates "retrieval
helps" from "more pretraining helps").

Gate: if memory-off matches memory-on, the contribution is zero. That is a publishable
negative result only if you can explain *why*; otherwise pivot.

### Stage D — Adaptation comparison (claim C3)

`run_adaptation_compare.py`: Fidora, DANN-R, Wi-SFDAGR, FewSense, CrossFi under matched
label budgets. Report each baseline's required labels and compute in the table.

### Stage E — Out-of-memory detection (claim C4)

Withhold one environment from the memory entirely; measure flag rate and post-flag accuracy.
Calibration curve if you use the conformal-style branch.

### Stage F — Figures

Vector PDF only (`savefig('fig.pdf')`), Okabe-Ito palette, no title inside the figure,
self-contained captions, grayscale-checked. Figure 1 is currently a placeholder box in the
draft — draw it last, once the pipeline is frozen, so the diagram matches the code.

### Non-negotiables for every stage

- Save after each unit of work; skip completed work on re-run so a crash is recoverable.
- Snapshot the script next to its results (`cp run_x.py results/<exp>/script_snapshot.py`).
- Commit results with a one-line finding in the message.
- Report mean ± std over seeds and the seed control method.

---

## 5. What I would fix in the draft before it goes anywhere

1. **Contribution 3 over-promises.** It currently says we "report" statistical tests for an
   evaluation that has not run. Either run Stage A–C or soften it. Flagged with `[TODO]` in
   the manuscript.
2. **The novelty claim is the weak point.** Retrieval-conditioned adaptation sits between
   two crowded literatures: few-shot prototype matching (FewSense, CrossFi, FewCS) and
   wireless RAG (ENWAR, ENWAR 2.0, Wi-Chat). The distinguishing move is that no prior work
   uses a *spatially-indexed* memory as the adaptation mechanism for a *sensing head*.
   Re-run a targeted search for "retrieval-augmented CSI sensing" and
   "environment-conditioned Wi-Fi sensing" immediately before submitting; this is the claim
   most likely to be scooped. Flagged with `[TODO]`.
3. **One citation cannot be verified** and is excluded from the bibliography:
   Puspitasari (2014). It is marked `[CITATION NEEDED]` in the appendix. Supply a DOI or
   publisher page, or drop it.
4. **Abstracts: 42/45 fetched** (93%). ENWAR 2.0 and WiSwiss were recovered from OpenAlex.
   The three still missing are `Lee2026PCC`, `Yang2021BookChapter` and `Puspitasari2014RSSI`
   — all three are genuinely absent from OpenAlex/Crossref/arXiv (verified individually,
   including by DOI and by title search). None is cited for a factual claim, so there is no
   fabrication risk, but **do not paraphrase their content from the title** — read the PDF
   first.

---

## 6. Reproducing the reference work

```bash
python3 scripts/verify_refs_v2.py      # Crossref/arXiv/DBLP -> references_verified.bib
python3 scripts/resolve_refs.py        # inconsistency probes + candidate lookups
python3 scripts/expand_abstracts.py    # OpenAlex/Crossref/arXiv -> notes/abstracts.json
python3 scripts/add_arcfi.py           # the no-DOI reference (ARC-Fi via arXiv)
python3 scripts/append_added_refs.py   # the added citations
python3 scripts/lint_paper.py          # structural checks before compiling
python3 scripts/make_ref_report.py     # regenerate references_report.md
```

`lint_paper.py` currently reports **0 errors**: all 49 distinct `\cite` keys resolve
against the 49 bibliography entries, all environments balance, math delimiters balance, no
dangling `\ref`, no duplicate labels. The one warning is benign: a few labels
(`sec:intro`, `fig:overview`) are declared but never cross-referenced.

---

## 7. Compiling the PDF

**The container cannot build the PDF, and this was verified, not assumed.** Four
independent routes were tried:

| Route | Result |
|---|---|
| `dnf install texlive-*` | **Fails.** `baseos` repomd.xml checksum mismatch; "All mirrors were tried". Also no `pdflatex`/`latexmk`/`bibtex` afterwards. |
| Tectonic static binary | Binary runs (`/root/bin/musl/tectonic`), but its bundle is `default_bundle_v33.tar` -> **2.88 GB** at ~280 KB/s (~3 h), and every fetch of `latex.ltx` from `data1b.fullyjustified.net` **times out**. No PDF. |
| TeX Live installer (`install-tl`) | **Fails: `env: perl: No such file or directory`.** There is no `perl` in this image, so `install-tl`, `tlmgr`, `fmtutil` and `mktexfmt` are all unusable. Tectonic was the only engine that could have worked, and it cannot reach its bundle. |
| Individual CTAN packages | **Work, and fast (~1.5 MB/s).** All 50 packages `main.tex` needs download in seconds — total 5.7 MB. |

Also relevant: there is **no `xz` binary**, but Python has `lzma`/`tarfile`, so `.tar.xz`
can still be extracted programmatically.

`scripts/stage_texlive.py` implements the only viable remaining route: take the ~1.2 GB
`texlive-<date>-bin.tar.xz` (binaries + basic runtime), extract it with Python's `lzma`,
drop the 50 already-downloaded tlnet packages into `texmf-dist`, and let kpathsea find
everything. It also generates the `pdflatex.fmt` format file directly (since `fmtutil`
needs perl) and then runs the full `pdflatex`/`bibtex`/`pdflatex`/`pdflatex` sequence.

Run it once the binary bundle is down:

```bash
cd /root && curl -sL -o tlbin.tar.xz \
  https://ftp.math.utah.edu/pub/tex/historic/systems/texlive/2024/texlive-20240312-bin.tar.xz
python3 /root/paper-workspace/scripts/stage_texlive.py
```

Or compile on your own machine, which is simpler and needs nothing unusual:

```bash
cd paper && latexmk -pdf main.tex
```

`main.tex` needs only stock `IEEEtran` plus `cite`, `amsmath`, `graphicx`, `booktabs`,
`xcolor`, `url`, `algorithm`, `algpseudocode`. Two packages were **removed** because their
tlnet archive names are unavailable and neither was load-bearing: `subcaption` (unused —
there are no subfigures) and `balance` (its only call was `\balance`, a cosmetic
two-column nicety before the bibliography).

Until a PDF exists, read the draft as `paper/PAPER_PREVIEW.md`.

When you are ready to de-draft it: set `\draftnoticetrue` -> `\draftnoticefalse` to
remove the red banner, fill the 7 `[RESULT NEEDED]` slots, and resolve the 10 `[DECISION]`
and 4 `[TODO]` markers.
