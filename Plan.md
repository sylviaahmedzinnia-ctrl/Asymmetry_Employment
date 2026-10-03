# Asymmetry in U.S. Unemployment Dynamics

**Research question:** Does unemployment increase faster during recessions than it decreases during economic recoveries?

---

## 1. Motivation and Hypothesis

A long-standing observation in macroeconomics is that the unemployment rate behaves
asymmetrically over the business cycle: it spikes sharply when the economy contracts
and drifts down slowly when the economy recovers. Friedman's "plucking model" and the
nonlinearity literature (Neftçi 1984; Sichel 1993) describe this pattern, but it is
straightforward to test directly with public data.

**H0 (null):** The average monthly change in the unemployment rate during contractions
is equal in magnitude to the average monthly change during recoveries.

**H1 (alternative):** The magnitude of the monthly increase during contractions exceeds
the magnitude of the monthly decrease during recoveries — unemployment rises like a
rocket and falls like a feather.

**Expected finding:** unemployment rises faster per month, and for fewer months, than it
falls.

**What would support or contradict the expectation.** "Faster" is tested in two senses,
speed (pp per month) and duration (months), and each gets its own verdict:

- **Supports H1:** in the paired cycle-by-cycle tests, ascent speed exceeds descent speed
  and descent duration exceeds ascent duration, with one-sided p < 0.05; the pooled tests
  agree; and the direction survives the robustness checks in Section 5, above all
  excluding 2020.
- **Contradicts H1:** descent speed equal to or greater than ascent speed (an asymmetry
  ratio at or below 1), or recoveries no longer than rises, across most cycles.
- **Inconclusive:** point estimates in the H1 direction but primary tests that disagree
  about significance. With about 12 cycles, low power makes this a real possibility, and a
  non-rejection would be weak evidence of symmetry rather than evidence against asymmetry.

**Why it matters:** if the asymmetry is real, the labor-market cost of a recession is
not symmetric with the benefit of a recovery of the same length, which bears on how
aggressive stabilization policy should be and on how long "full employment" takes to
restore.

---

## 2. Data

All series are pulled from FRED via the `fredapi` client. The API key is read from the
`FRED_API_KEY` environment variable (stored in a local `.env` file that is git-ignored).
No key is committed to this repository.

| Series ID | Description | Frequency | Role |
|-----------|-------------|-----------|------|
| `UNRATE`  | Civilian unemployment rate, 16+, seasonally adjusted | Monthly | Primary dependent variable |
| `USREC`   | NBER-based recession indicator (1 = contraction) | Monthly | Defines recession/recovery regimes |
| `PAYEMS`  | Total nonfarm payroll employment | Monthly | Robustness: immune to labour-force exit |
| `NROU`    | CBO noncyclical rate of unemployment | Quarterly | Robustness: u\* is not constant over 1948–2026 |

`GDPC1` and `ICSA` were considered and dropped: neither is needed to identify regimes once
`USREC` and `UNRATE`'s own turning points are in hand, and pulling series the analysis does
not use is clutter.

**Sample period:** January 1948 (start of `UNRATE`) through the most recent available
month. Robustness splits are described in Section 5.

**Data vintage.** `UNRATE` is revised — the BLS re-estimates seasonal factors each
January — so a cached CSV can diverge from what FRED currently publishes. Every pull is
stamped in `data/raw/_manifest.csv` with its retrieval date and last observation, so the
vintage behind any reported result is documented.

### Cleaning steps

1. **One monthly grid.** Combine the four series on a month-start `DatetimeIndex`
   (`asfreq("MS")`), first checking that no observation is stamped off the 1st of the month
   and that no date is duplicated, since `asfreq` would silently drop such rows.
2. **Frequency conversion.** `NROU` is quarterly. Linearly interpolate it to monthly
   between observed quarters only, with no extrapolation beyond its span.
3. **Date range and merge.** Trim to the span where both `UNRATE` and `USREC` exist
   (January 1948 onward). `USREC` starts in 1854 and `PAYEMS` in 1939, so `UNRATE` is the
   binding constraint. Cast `USREC` to an integer flag.
4. **Missing values.** Keep any missing month on the grid as `NaN` rather than dropping
   the row. Dropping it would make the next month-over-month change span two months.
   Leading `NaN`s from series that start later are expected and are left in place.
5. **Transformations.** Monthly change in `UNRATE` and its first lag; a centred 3-month
   moving average; percent change in `PAYEMS`; the unemployment gap `UNRATE − NROU` and its
   change.

---

## 3. Definitions

Precise regime definitions drive the entire result, so they are fixed in advance:

- **Contraction (recession) episode.** From the NBER business-cycle peak month to the
  trough month, as encoded by `USREC == 1`.
- **Recovery episode.** From the NBER trough month to the earlier of (a) the next NBER
  peak, or (b) the month in which `UNRATE` first returns to its pre-recession level.
  Both variants are computed; (b) is the primary specification because it measures
  labor-market repair rather than expansion length.
- **Unemployment ascent.** From the cycle's pre-recession `UNRATE` minimum to the
  cycle's `UNRATE` maximum. Unemployment is a lagging indicator, so its turning points
  do not coincide exactly with NBER dates; using `UNRATE`'s own local extrema avoids
  mechanically understating the descent.
- **Descent.** From the `UNRATE` maximum to the month `UNRATE` returns to the prior
  minimum. If it never fully recovers before the next recession, the descent ends at the
  lowest rate reached before unemployment turns up again, which is where the next cycle's
  ascent begins, so no month is counted in two cycles.

Both NBER-dated and `UNRATE`-extrema-dated episodes are reported so the finding is not
an artifact of one dating rule.

---

## 4. Methodology

### Step 1 — Ingest and cache
Pull the series, store raw pulls to `data/raw/` as CSV so the analysis is reproducible
without re-hitting the API. Align everything to a monthly `DatetimeIndex`.

### Step 2 — Episode construction
Build an episode table: one row per business cycle with peak date, trough date,
recovery-complete date, `UNRATE` at each point, and the duration in months of the
ascent and descent phases.

### Step 3 — Core comparison (descriptive)
For each episode compute:

- `ascent_rate`  = (u_max − u_min_pre) / months_of_ascent  → percentage points per month
- `descent_rate` = (u_max − u_min_post) / months_of_descent → percentage points per month
- `asymmetry_ratio` = ascent_rate / descent_rate
- `recovery_lag` = months_of_descent − months_of_ascent

### Step 4 — Formal tests

All tests are **one-sided** in the direction stated in H1, fixed here before any data
were pulled.

1. **Paired test across episodes.** Paired t-test and Wilcoxon signed-rank test, run on
   two distinct claims: **speed** (`ascent_rate` vs `descent_rate`) and **duration**
   (`months_descent` vs `months_ascent`). Both are needed because "faster" has two
   meanings, and a recovery can be only slightly slower per month yet take three times as
   long. Paired because each cycle contributes both values; Wilcoxon because n ≈ 12 and
   normality is not assumed. This is the most conservative and most credible test in the
   study.
2. **Pooled monthly changes.** Welch two-sample t-test and Mann–Whitney U on the
   month-over-month change in `UNRATE`, split by cycle phase. Welch rather than Student's
   because recession months are far more volatile than recovery months.
3. **Skewness test.** The asymmetry implies a right-skewed distribution of ΔUNRATE.
   Test skewness against zero (D'Agostino), which is a dating-rule-free check.
4. **Regression specification.**
   `ΔUNRATE_t = α + β·Rising_t + γ·(ΔUNRATE_{t−1} − mean) + ε_t`
   The lag is **demeaned** so that α is the fitted change for an average recovery month;
   with a raw lag, α is the conditional mean at ΔUNRATE_{t−1} = 0, which is not a
   meaningful quantity. β is the *impact* effect; because γ propagates each shock forward,
   the economically relevant magnitude is the cumulative effect **β/(1−γ)**, which is
   reported alongside it.

   Two error structures: Newey–West HAC (12 lags) for serial correlation, and
   cluster-robust by business cycle. The clustered version has only ~12 clusters, far below
   the 30–50 needed for reliable asymptotics, so it is reported as a discipline on the HAC
   result rather than as the primary inference.

### Step 5 — Visualization
- `UNRATE` time series with NBER recession shading and CBO u\* overlaid.
- Event-study plot: all cycles aligned at the `UNRATE` peak (t = 0), months −24 to +60.
  **Two panels**: raw percentage points, and each cycle normalised by its own peak-to-low
  amplitude. The raw average alone conflates *shape* asymmetry (the claim) with *amplitude*
  differences between deep and shallow cycles; the normalised panel isolates shape.
- Per-episode bar chart of ascent vs descent, in **both** speed and duration.
- Histogram of ΔUNRATE by phase, plotted as **densities not counts** — recoveries last far
  longer than recessions, so raw counts would differ for a reason unrelated to the
  hypothesis.

---

## 5. Robustness Checks

| Check | Reason |
|-------|--------|
| Exclude the 2020 COVID cycle | A 10+ pp spike and unusually fast rebound is an extreme outlier that could drive or mask the result in either direction. |
| Pre-1985 vs post-1985 split | The Great Moderation changed cycle amplitude; "jobless recoveries" (1991, 2001, 2009) may be a post-1985 phenomenon. |
| 3-month moving average of `UNRATE` | Reduces month-to-month sampling noise in the household survey. |
| NBER dating vs `UNRATE`-extrema dating | Confirms the result is not a dating artifact. The least favourable specification for H1. |
| `PAYEMS` growth as the dependent variable, sign-flipped | Confirms the result is not specific to the unemployment-rate definition — a recovery can look fast because discouraged workers left the labour force. |
| Unemployment **gap** (`UNRATE − NROU`) as the change variable | A drifting natural rate would contaminate a comparison of raw levels. |
| Episodes **rebuilt on the gap**, then paired-tested | The stronger form of the same check: redates every turning point relative to u\*, rather than merely redefining the monthly change. |
| Completed recoveries only | In unfinished episodes (unemployment never regained its prior low) the next recession cuts the recovery short. That understates descent duration, which biases the duration test **against** H1, and it truncates both the distance and the time of the descent, so its effect on speed has no fixed sign. This check removes those episodes. |

**Multiple testing.** Around a dozen tests of a single hypothesis are reported. They are
far from independent — most reuse the same months — so Bonferroni is conservative rather
than correct; the threshold is printed for reference. The verdict requires **all four
primary tests** (pooled Welch, pooled Mann–Whitney, paired t, Wilcoxon) to reject, not the
most favourable one. Every test run is reported, including any that fail.

---

## 6. Expected Output

- A tidy episode-level table (`data/processed/episodes.csv`) reporting ascent rate,
  descent rate, ratio, durations, and a `recovered` flag per cycle.
- A data-vintage manifest (`data/raw/_manifest.csv`).
- A results table of all test statistics and p-values
  (`results/tables/results_summary.csv`) and the robustness table
  (`results/tables/robustness.csv`).
- Four figures saved to `results/figures/`.
- A written conclusion stating whether H0 is rejected, the effect size in
  percentage-points-per-month **and in months**, and the main caveats.

**Validation.** Before any test is run, the episode table is checked against known history
(2020 peaks in April 2020; the 2008 cycle peaks in 2009, after the NBER trough) and for
internal coherence (dates ordered, amplitudes positive, descent rates non-negative, labels
unique). Every episode dropped or re-dated by the construction code is printed, never
discarded silently.

---

## 7. Limitations

- **Low power.** With ~12 U.S. cycles since 1948, the paired design can detect only a
  large asymmetry. A null result would be weak evidence of symmetry, not evidence against
  asymmetry. The pooled and regression tests have more observations but overstate their
  effective sample size, since months within one recession are not independent draws.
- **Unfinished recoveries.** Episodes where unemployment never regained its prior low end
  at the lowest rate reached, which understates how long a full recovery would have taken.
  They are flagged in the episode table and removed in a dedicated sensitivity run.
- NBER dates are announced with a lag and are partly judgmental. The analysis avoids
  depending on them by dating turning points on `UNRATE` itself.
- The unemployment rate is affected by labor-force participation, so a "recovery" can
  look faster than it is when discouraged workers exit the labor force. `PAYEMS` partly
  addresses this.
- `NROU` is a CBO model estimate, not data, and is interpolated from quarterly to monthly.
- This is a descriptive/statistical study. It documents asymmetry; it does not identify
  its cause (hiring frictions, search behavior, firing costs, hysteresis).

---

## 8. Project Structure

```
unemployment_recession_analysis/
├── Plan.md              # this file
├── analysis.ipynb       # main analysis notebook
├── requirements.txt     # pinned dependencies
├── .gitignore
├── .env                 # FRED_API_KEY (never committed)
├── data/
│   ├── raw/             # cached FRED pulls
│   └── processed/       # episode tables
└── results/
    ├── figures/
    └── tables/
```

---

## 9. Setup

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows
pip install -r requirements.txt
```

Create a `.env` file in the project root containing:

```
FRED_API_KEY=your_key_here
```

Request a free key at https://fredaccount.stlouisfed.org/apikeys. The notebook loads it
with `python-dotenv`; it is never hard-coded and `.env` is git-ignored.

---

## 10. Workflow Checklist

- [x] Environment set up, `FRED_API_KEY` loads successfully
- [x] Series pulled and cached to `data/raw/`
- [x] Episode table constructed and manually sanity-checked against NBER dates
- [x] Descriptive ascent/descent rates computed
- [x] Formal tests run
- [x] Figures produced
- [x] Robustness checks run
- [x] Conclusion written

---

## References

- Friedman, M. (1993). "The 'Plucking Model' of Business Fluctuations Revisited."
  *Economic Inquiry*, 31(2), 171–177.
- Neftçi, S. N. (1984). "Are Economic Time Series Asymmetric over the Business Cycle?"
  *Journal of Political Economy*, 92(2), 307–328.
- Sichel, D. E. (1993). "Business Cycle Asymmetry: A Deeper Look."
  *Economic Inquiry*, 31(2), 224–236.
- National Bureau of Economic Research, US Business Cycle Expansions and Contractions.
