# Residual treatment in masking-constrained peak reduction

Research for issue #4. Vocabulary per `GLOSSARY.md`: Residual, Masking Threshold, Transparent, Peak Ceiling, Clipping, Spectral Magnitude Limiting, Reduction Depth, Phase Rotation.

Evidence grades used throughout:
- **(a)** verified from a primary source (source code, paper text, patent claims, spec)
- **(b)** secondary claim (abstract, review article, forum, vendor copy)
- **(c)** my own inference

---

## Verdict

**The space is not unexplored, and the core idea the project hopes to invent already ships in FFmpeg.**

The originating hope — "iteratively clip, take the Residual, hold it beneath the Masking Threshold, repeat" — exists as a working, published algorithm. It is called the **psychoacoustic clipper**: `apsyclip` in FFmpeg's libavfilter, the "Psychoacoustic Clipper" in Calf Studio Gear, both derived from Jason Jang's `ShapingClipper`. I read the FFmpeg C source line by line; it is structurally the exact algorithm described in the ticket **(a)**.

So the apparent gap is largely filled by work under terminology this project has not been using: *perception-based clipping* (Defraene et al., academic), *psychoacoustic clipper* (FFmpeg/Calf, practitioner), *iterative clipping and filtering* (the RF/PAPR name for the same iteration structure).

**But three real gaps survive, and all three are exactly where this project's settled decisions give it an advantage:**

1. **No published method delivers a true Peak Ceiling.** Every implementation found constrains *sample* peaks, inside a window, with no oversampled detection. Defraene's constraint is an affine box on output samples **(a)**; `apsyclip` measures peaks on the windowed frame and then does overlap-add afterwards, which can perturb the result **(a)**. The project's goal is a hard **true**-peak guarantee. Nobody has published a masking-constrained peak reduction with a true-peak constraint **(c)**.
2. **Everyone is latency-constrained; this project is not.** Defraene chose frame/solver parameters to hit an 8.7 ms real-time deadline **(a)**. `apsyclip` uses a 256–1024-point FFT with 4× overlap — at 192 kHz that is a 1024-point window, about 5.3 ms, with roughly 187 Hz per Bin, which is poor frequency resolution for low-frequency masking **(a, c)**. With unlimited latency the project can use far longer frames, a proper psychoacoustic model, and a solver run to convergence. This is the single largest unexploited axis **(c)**.
3. **Masking models in use are crude.** `apsyclip`'s Masking Threshold is a spreading function applied to the magnitude spectrum, divided by a fixed 14–20 dB margin curve. It has no absolute threshold of hearing, no tonal/non-tonal masker discrimination, and no temporal masking **(a)**.

There is also a **hard theoretical ceiling** on the whole approach that the ticket should know about, from the Gerzon–Craven noise shaping theorem (below). Shaping the Residual cannot reduce its total power — only redistribute it, at a net cost in total power. That bounds how much any residual-treatment scheme can win **(b, c)**.

---

## 1. The central prior art: `apsyclip` / ShapingClipper

**Core idea.** Per frame: compute a Masking Threshold from the input spectrum; clip; take the Residual; project the Residual's spectrum so no Bin exceeds the Masking Threshold; add it back; repeat. When the peak target and the masking constraint conflict, the algorithm *relaxes the masking constraint* progressively rather than giving up peak reduction.

**Verified from source** (`libavfilter/af_apsyclip.c`, FFmpeg master, read in full) **(a)**:

- `calculate_mask_curve()`: `mask_curve = spread(|X|) / margin_curve`. The spreading function is `exp(-|log(j/i)| * 40)` upward and `exp(-|log(j/i)| * 80)` downward in log-bin distance, normalised, with two spread functions cached per octave as an optimisation.
- `margin_curve` is linearly interpolated in dB from the points
  `{0,14} {125,14} {250,16} {500,18} {1000,20} {2000,20} {4000,20} {8000,17} {16000,14} {20000,-10}`.
  So the allowed Residual sits **14–20 dB below** the spread signal spectrum through the audible range. Notably at 20 kHz the margin goes to **−10 dB**, i.e. the Residual is permitted *10 dB above* the signal there — a deliberate dumping of distortion into the inaudible top octave. That is the "spectral redistribution of the residual" branch of the ticket, already implemented, crudely **(a, c)**.
- `clip_to_window()` accumulates into `clipping_delta` (the Residual) against a *window-scaled* limit, with a `delta_boost` of 2 applied in the last third of iterations to converge on the peak target faster.
- `limit_clip_spectrum()` is the masking projection: per Bin, `rel = |R_i| * 2 / mask_curve_i`, and if `rel > 1`, divide that Bin by `rel`. This is a magnitude-only radial projection; **the Residual's phase is untouched** — a notable degree of freedom nobody exploits **(a, c)**.
- The adaptive relaxation: each iteration multiplies `mask_curve` by a `mask_curve_shift` of at least 1 dB, scaled by the `adaptive` parameter (default 0.5), growing toward the peak overshoot ratio when the peak target is being missed.
- `config_input()`: `fft_size` = 256 / 512 / 1024 for ≤50 k / ≤100 k / >100 kHz; `overlap = fft_size/4`; `num_psy_bins` is halved twice at high rates because the masking calculation is O(n²). Output is Hann-windowed overlap-add divided by 1.5.
- FFmpeg options **(a)**: `level_in`, `level_out`, `clip` (default 0 dBFS), `diff` (audition the Residual alone — directly useful for this project's listening work), `adaptive` (default 0.5), `iterations` (1–20, default 10), `level`.

**Is it a Peak Ceiling?** No. **(a, c)** The peak is measured on the windowed frame before overlap-add, and the masking constraint is relaxed rather than the peak constraint when they conflict — so it *tends* toward the clip level without guaranteeing it, and the final overlap-add is outside the loop. It is a sample-peak-seeking algorithm, not a true-peak guarantee. Any use here needs a final true-peak backstop stage.

**Licence — important.** **(a)**
- `jcj83429/ShapingClipper` on GitHub has **no LICENSE file** (confirmed via the GitHub API: `license: None`). Default copyright, all rights reserved. **Not usable.** It also bundles `pffft`. 1 star, last pushed 2024-06-08.
- The FFmpeg port `af_apsyclip.c` is **LGPL-2.1-or-later**, copyright Jason Jang 2014–2021 and Paul B Mahol 2021.
- Calf Studio Gear is **LGPL-2.1**.

For an MIT project, none of these can be copied or relicensed. The *algorithm* can be reimplemented from its published description — and the README/source describe it plainly — but the code cannot be lifted. Flagging as the ticket asked: **no permissive implementation of this algorithm exists**; everything is LGPL or unlicensed.

**No listening evidence found.** I could not find a single blind test, ABX result, or even a substantive forum discussion of `apsyclip`'s audible quality — not on Hydrogenaudio, not in the repo's issues **(a, negative result)**. This is a genuine opening: the algorithm is published and shipping but, as far as I can establish, has never been evaluated against the project's Pass Gate or Worth-Using Gate.

---

## 2. Perception-based clipping (Defraene et al.) — the academic formulation

Read in full from the KU Leuven archived PDFs (open access).

**Defraene, van Waterschoot, Ferreau, Diehl, Moonen, "Real-Time Perception-Based Clipping of Audio Signals Using Convex Optimization," IEEE TASLP 20(10), Dec 2012, 2657–2671.** DOI 10.1109/TASL.2012.2210875. Archived: `ftp.esat.kuleuven.be/pub/stadius/bdefraen/reports/11-127.pdf`

**Core idea (a).** Formulate Clipping itself as a constrained convex optimization per frame: minimise a Masking-Threshold-weighted L2 norm of the Residual in the frequency domain, subject to an affine box constraint holding every output sample between the clipping levels. Because the objective is quadratic and the constraints affine, each frame is a quadratic program.

Verified details **(a)**:
- Perceptual weights: `w_k` exponentially decreasing in the global Masking Threshold, `w_k = 10^(-c * M_k)`-style with compression parameter `c` found to work best in **0.04–0.06**. The authors note the summed terms "resemble distortion-to-masker power ratios."
- Masking Threshold from the **ISO/IEC 11172-3 MPEG-1 Layer 1 psychoacoustic model 1**: 512-point FFT, SPL normalisation, tonal/non-tonal masker identification, masker decimation (discard below absolute threshold of hearing; merge pairs within 0.5 Bark), individual thresholds, power-additive combination with the absolute threshold of hearing. Substantially more rigorous than `apsyclip`'s spreading function **(a, c)**.
- Three structure-exploiting solvers; method 3 is a projected gradient method with optimal linear convergence, O(√(1/ε)) algorithmic complexity versus method 2's O(1/ε).
- Frame-based with trapezoidal window and overlap-add. Real-time budget stated as **8.7 ms** per frame at 44.1 kHz; about 200 iterations fit. Solution accuracy ε = 10⁻⁴ was experimentally sufficient — quality saturates above that.
- Evaluation: 24 excerpts (music + speech, 16-bit mono 44.1 kHz), PEAQ ODG and Rnonlin, clipping factors {0.70, 0.80, 0.85, 0.90, 0.95, 0.97, 0.98, 0.99}. Perception-based clipping scored significantly higher than hard and soft clipping at every clipping factor; one-tailed paired t-tests at α = 0.05 reject the null for clipping factors ≥ 0.80 (PEAQ).

**Relevance to the 3–6 dB Reduction Depth target (c).** A clipping factor of 0.70 is about 3.1 dB of Reduction Depth, 0.90 is about 0.9 dB. So Defraene's *deepest* tested condition sits at the *shallow* end of this project's transparency target, and most of his evaluation is at depths under 1 dB. The 3–6 dB region this project cares about is essentially unexplored by the academic literature **(c)**. `apsyclip` goes deeper in practice but has no published evaluation at all.

**Objective-only evaluation.** PEAQ and Rnonlin, no listening test in the 2012 paper. The group did run subjective tests later, but on the loudspeaker precompensation work, not on clipping (JASA Express Lett. 140(1), 2016) **(a)**.

**Related, same group:**
- Defraene et al., ICASSP 2011, "A fast projected gradient optimization method for real-time perception-based clipping of audio signals," 333–336. Archived: `.../reports/10-244.pdf`
- Defraene et al., "Declipping of audio signals using perceptual compressed sensing," IEEE TASLP 21(12), Dec 2013, 2627–2637. The declipping counterpart: sparse recovery with a perceptual weighting, "perceptual compressed sensing." Archived: `.../reports/13-35.pdf` **(a)**

**No code published** for any of these **(a)** — the publications page lists no repositories, and I found none.

**A third formulation worth noting.** arXiv 2202.08183, "Increasing Loudness in Audio Signals: a perceptually motivated approach to preserve audio quality" (Feb 2022, eess.SP) **(b)** — fetched the HTML, which lacks a full author list. It gives two convex problems:
- **A:** minimise `‖x‖∞` subject to detectability `‖P_x₀ W (x − x₀)‖₂ ≤ c` — control quality, let the peak fall where it may.
- **B:** minimise detectability subject to `‖x‖∞ ≤ λ` — control the peak, minimise audible damage. This is Defraene's formulation restated.

Its masking model is a Gammatone-filterbank *detectability* measure (van de Par, Taal), calibrated so D = 1 is just-detectable. Formulation A is the interesting one for this project because it makes Transparency the constraint and Reduction Depth the outcome, which is the inverse of how limiters are normally built **(c)**. Caveats: evaluated only on 16 kick-drum clips downsampled to **1 kHz**, objective metrics only, no solver named, no code. Dataset at Zenodo DOI 10.5281/zenodo.5828375.

---

## 3. The RF/PAPR hypothesis — tested

The ticket's specific hypothesis was that PAPR reduction solves formally the same problem. **Partly confirmed, and already cashed in.**

**What transfers (and has already transferred).** *Iterative clipping and filtering* (ICF) is the standard PAPR workhorse: clip in time, filter in frequency, repeat **(b)**. That is structurally identical to `apsyclip`'s loop with the band-limiting filter replaced by a masking projection **(c)**. So the transfer the ticket hypothesised has in effect already happened — ICF's structure is what the psychoacoustic clipper is. The PAPR literature's useful contribution is the *theory of the iteration*, not a new mechanism:

- ICF converges slowly — "8 to 16 iterations" classically; an optimised per-iteration filter chosen by convex optimization reaches target in 1–2 **(b)**, Wang & Luo, "Optimized Iterative Clipping and Filtering for PAPR Reduction of OFDM Signals," IEEE Trans. Commun. This maps onto choosing the per-iteration masking relaxation optimally instead of `apsyclip`'s heuristic `mask_curve_shift` **(c)**. **This is a concrete, publishable improvement to an existing algorithm.**
- ICF reads cleanly as **alternating projections / POCS**: clipping is the Euclidean projection onto the amplitude-bounded set, ideal filtering is an orthogonal projection onto the band-limited subspace **(c — this framing is mine; I found no source that states it, and the search explicitly reported none)**. The caution matters: the project's two sets — "true peak ≤ ceiling" and "Residual beneath Masking Threshold" — **may have empty intersection** at 3–6 dB Reduction Depth. Alternating projections then cycles rather than converging, which is precisely why `apsyclip` must relax the masking constraint. Any formulation must decide *in advance* which constraint is hard. For this project the answer is settled: the Peak Ceiling is the guarantee, so masking is the soft objective **(c)**.
- **ADMM** is used for PAPR in mixed-numerology OFDM (arXiv 1911.01569) **(b)**.

**What does not transfer (a, b).**
- **Tone reservation** needs spare bandwidth whose content the receiver discards. Music has no receiver and no spare bandwidth — except, interestingly, the ultrasonic band above 20 kHz, which is exactly what `apsyclip`'s −10 dB margin at 20 kHz exploits. That is the audio analogue of tone reservation, and it is already in the shipping code **(c)**.
- **Tone injection** enlarges the constellation and raises symbol power; it relies on a discrete constellation that audio does not have **(b)**.
- **Selected mapping** transmits one of M alternative encodings plus side information identifying which. There is no decoder in a mastering chain **(b)**. Dead end.
- PAPR work optimises bit error rate, not audibility, and none of it carries a masking model **(b)**. The optimisation target does not transfer; only the iteration structure does.

**One cross-over paper exists but is not useful:** Al-Safadi & Al-Naffouri, "Peak Reduction and Clipping Mitigation by Compressive Sensing," arXiv 1101.4335 (2011), IEEE TSP DOI 10.1109/TSP.2012.2193396. Despite the promising title it is pure OFDM: a sparse clipper at the transmitter and compressed-sensing recovery of the clipped coefficients at the *receiver*, about 4.5 dB PAPR reduction with 20% reserved tones. No perceptual model, and the receiver-side recovery has no analogue here **(a, from the abstract page)**.

**Verdict on the hypothesis:** the field does solve a formally similar problem, its iteration theory is worth borrowing, and the specific techniques built on the receiver/constellation structure do not transfer. The transfer that *did* make sense has already been made by `apsyclip` without citing the PAPR literature **(c)**.

---

## 4. Noise shaping and error feedback applied to the Residual

The ticket asked specifically about applying noise shaping / error feedback to clipping distortion rather than quantization error. **This branch has a known, serious obstacle and a hard theoretical limit.**

**The idea.** Treat the clipper as a quantizer: the Residual is the "error," feed it back through a filter so the total error acquires a chosen noise transfer function, and use a psychoacoustically shaped target. The telescoping identity holds for any memoryless nonlinearity and any error magnitude, so published noise transfer functions drop in **(b)**.

**The obstacle: error windup (b).** A quantizer's error is bounded by half an LSB. A clipper's Residual is **unbounded**. Feeding an unbounded error through a feedback filter winds up. The comp.dsp discussion of the same problem from the requantizer side reports that clamping the sum of signal and feedback to the available range produces "horrid results with error windup," while clamping the input instead costs dynamic range, worsening with higher-order shaping. Hardware noise-shaping quantizer patents handle it by clamping how much of the limiter error enters the next shaping stage (e.g. US7411527) **(b)**. This is a real, known failure mode, not a hypothetical.

**Community status.** The only direct treatment I found of noise-shaping the Residual of a clipper is a KVR Audio DSP forum thread ("Smoother clippers if clipping residue is treated via noise-shaping"). **It is explicitly labelled by its own author as AI-generated and never tested** **(a — I am reporting the disclosure, not the content)**. It should not be cited as evidence of anything. I found no peer-reviewed or tested treatment. So this branch is genuinely under-explored — but the windup problem explains why.

**The hard limit: the Gerzon–Craven noise shaping theorem (b).** Gerzon & Craven, AES preprint 2822 (1989) — I could not obtain the full text, so this is secondary. The theorem constrains the integral of the log magnitude of the noise transfer function to zero. Consequences, which bound this project's entire premise **(b, c)**:

- **Shaped error always carries more total power than unshaped error.** Any reduction in one band is paid for by amplification elsewhere, on a logarithmic scale. The optimum is piecewise constant: a fixed reduction in the passband, a fixed amplification in the stopbands.
- Therefore pushing the Residual beneath the Masking Threshold **increases the total Residual energy**. Transparency is bought with more distortion, not less — placed where it is masked. This is the correct mental model for the whole project and it sets a ceiling on achievable Reduction Depth at fixed Transparency **(c)**.
- The psychoacoustically optimal curve is signal-dependent, so coefficients must be recomputed continuously **(b)**.

**A scepticism datapoint worth keeping (b).** A critical listening study on high-order dithering concluded "it is thus not possible to 'psycho acoustically optimize' an algorithm, contrary to the claims of some manufacturers and researchers," finding outcomes depended heavily on playback use and listening environment. Relevant caution for the Pass Gate: a masking model validated at one playback level may not hold at another. Masking Thresholds are level-dependent; a master is auditioned at many levels **(c)**.

Related primary literature if this branch is pursued: Lipshitz/Vanderkooy/Wannamaker on psychoacoustically optimal noise shaping; De Koning & Verhelst, "On psychoacoustic noise shaping for audio requantization" (ICASSP 2003); "Noise shaping filter design for minimally audible signal requantization" (IEEE 2001).

---

## 5. Declipping read backwards

What optimal reconstruction tells you about what Clipping destroys.

**The key quantitative finding (b).** From the large-scale declipping evaluation (Záviška, Rajmic, Ozerov, Rencker, "Sparsity-based audio declipping methods: overview, new algorithms, and large-scale evaluation," IEEE/ACM TASLP): degradation becomes **imperceptible above roughly 30 dB SDR**, and the authors recommend SDR over clipping threshold as the measure of severity. "High degradation" in their benchmark is 1 dB input SDR, "mild" is 10 dB.

**Read backwards (c).** If ~30 dB Residual-to-signal ratio is the audibility boundary for *unshaped* clipping distortion, that is the budget a Transparent processor must beat *without* any masking exploitation. `apsyclip`'s 14–20 dB margin below the spread spectrum is a much looser constraint than a flat 30 dB SDR — the whole bet of this project is that masking buys back the difference. That reframes the 30 dB figure as the baseline to beat, and makes it a cheap, immediately useful sanity metric for the Reference Set **(c)**.

**What Clipping actually destroys (b, c).** Declipping works because of the **consistency constraint**: clipped samples are *known* to have exceeded the threshold, so the clipped region carries one-sided information rather than none. Recovery quality degrades with the fraction of clipped samples and with how much sparsity survives. So Clipping destroys the *sign-and-magnitude detail of the peaks* while preserving a one-sided bound and the sparse time-frequency structure of everything else. Implication for this project: a Residual confined to masked Bins is, by construction, *harder to declip away* — which is the correct goal, since the Residual should be perceptually absent rather than removable **(c)**.

**Permissively licensed code exists here (b, licence worth re-verifying before use).** SPADE (SParse Audio DEclipper, Kitić et al., INRIA) MATLAB code and toolbox are stated to be **BSD-3-Clause**, which is MIT-compatible. `spade.inria.fr`. Also: A-SPADE/S-SPADE ADMM derivations in arXiv 1809.09847 (paper text CC BY-NC-SA, which does not cover code), SPAIN (arXiv 1810.13137), an open-source C port "SpadePort" (licence unstated — check), and the "social sparsity" declipper (Siedenburg/Kowalski/Dörfler, ICASSP 2014) whose code has **no published licence**. Also useful: Defraene's perceptual compressed sensing declipper (above) is the one that already fuses the masking model with sparse recovery.

The ADMM machinery in SPADE is directly reusable as the solver skeleton for a masking-constrained projection problem, since the constraint sets are analogous **(c)**.

---

## 6. Patent landscape — flagging, not advice

I am not a lawyer; none of this is a legal opinion and claim construction needs a professional.

**US9225310B1, "Audio limiter system and method"** — the one to take seriously **(a, from the claims)**. Inventor Alexey Lukin; originally **iZotope**, now listed as **Native Instruments USA**. Priority 2012-11-08, granted 2015-12-29, listed as **active with adjusted expiry 2034-06-27**, maintenance fees paid through the 8th year (2023).

Claim 1 requires, in combination: apply a psychoacoustic model to get masking thresholds across critical bands; **generate multiple different amplitude envelopes** for the portion; for each, apply it, form a distortion signal by difference, compute per-critical-band distortion power, compare against the masking thresholds to produce a distortion score; select one based on its score. Dependent claim 3 adds the "below a pre-selected threshold" criterion.

**Why an `apsyclip`-style design is likely outside it (c).** The claim's distinguishing limitation is *multiple candidate amplitude envelopes* scored and selected among. An iterative Residual-projection clipper generates no candidate envelopes and performs no selection — it refines a single Residual. "Iterative" and "FFT" are not limitations in the claim, so they neither help nor hurt. This looks like a genuine structural difference rather than a workaround, but it is exactly the kind of question that needs a patent attorney before release.

**Older masking-controlled clipping art, long expired, useful as prior art (b).** US4208548 (estimate whether clipper distortion would be masked, feed that back to control drive into the clipper) and US4241266 (subtract clipper output from input, compare the difference against the input in a plurality of critical bands, drive a variable gain stage). The *idea* of masking-controlled clipping is therefore ~45 years old and firmly in the public domain. Orban's US5737434 criticises masking-based drive control for causing "unnatural" loudness changes because it reduces drive as a function of peak-to-average ratio — a design caution worth heeding **(b)**.

**US11881829B2** (AAC Acoustic Technologies, filed 2022-04-28, granted 2024-01-23, expiry listed 2042) claims **Phase Rotation** for peak reduction: a single allpass `H(z) = (g + z^-m)/(1 + g·z^-m)` whose delay `m` and gain `g` are chosen to minimise the maximum absolute output, `m` synced to a dominant transient peak in the autocorrelation, `g` refined by about three gradient-descent steps with a linear-programming step size **(a, from the claims)**. Its background mentions "iteratively clipping the input signal while recovering the magnitude spectrum" as *related art* — not claimed, no details. Relevant because Phase Rotation is in this project's glossary and is information-preserving; this patent covers one narrow parameterisation of it, not the concept.

---

## Three most promising techniques, ranked

### 1. True-peak-constrained iterative Residual projection with a long frame
**Best value. Do this one.**

**Core idea.** Take the `apsyclip` iteration — clip, project the Residual beneath the Masking Threshold, repeat — and change the three things the project's settled decisions let it change: make the peak constraint a **true**-peak constraint by evaluating it on an oversampled signal inside the loop; use a long frame (unlimited latency) for real low-frequency masking resolution; and replace the spreading-function margin with a proper psychoacoustic model. Keep the Peak Ceiling hard and the Masking Threshold soft, relaxing masking when the two conflict, since the intersection may be empty at 3–6 dB Reduction Depth.

**Implementation / licence.** Algorithm fully documented and verified at source level, but **no permissive implementation exists**: FFmpeg's is LGPL-2.1+, Calf's is LGPL-2.1, ShapingClipper is unlicensed. Must be reimplemented from the published description for MIT. `apsyclip`'s `diff` option is immediately useful for auditioning the Residual during development even if no code is reused.

**Prototype cost.** Low. **2–4 days** for an offline Python/NumPy version: STFT, spreading-function Masking Threshold, the clip-project loop, oversampled true-peak check. The loop is about 60 lines; the masking model is the bulk of it. Validate against `apsyclip` as a reference by running the same file through FFmpeg. A further **3–5 days** to swap in an ISO/IEC 11172-3-style model and sweep frame length. Real-time C++ is a later concern and no longer latency-bound.

**Main risk (c).** The true-peak constraint may fight the frame structure: overlap-add after the loop can reintroduce overshoot, so the true-peak guarantee probably has to come from a final time-domain backstop stage rather than from the optimization itself. Expect the honest architecture to be "masking-constrained reduction gets you to within a hair of the ceiling, then a conventional true-peak clipper guarantees it," with the Residual of that last stage being small enough not to matter.

### 2. Residual phase as a free variable
**The clearest genuinely unexplored idea found.**

**Core idea.** Every implementation examined projects the Residual's Bin *magnitudes* onto the Masking Threshold and leaves the phases untouched — verified in `limit_clip_spectrum()`, which divides real and imaginary parts by the same scalar **(a)**. But the time-domain peak is produced by phase alignment across Bins (the project's own glossary says this under Spectral Magnitude Limiting). So Residual phase is a degree of freedom that is currently wasted: choosing it to *minimise the resulting true peak* could buy Reduction Depth at zero cost in Masking Threshold violation, since magnitudes are unchanged. This is the Phase Rotation insight applied to the Residual rather than to the signal.

**Implementation / licence.** None found, in audio or in PAPR. The nearest neighbours are allpass decorrelation (DAFx 2016 "Signal Decorrelation using Perceptually Informed Allpass Filters"; DAFx 2017 velvet-noise decorrelator) and the DAFx 2023 immersive-audio warning that allpass phase response "is also likely to impair transients in the signal mix through phase dispersion" **(b)** — a real caution, since transients are exactly where peak reduction acts.

**Prototype cost.** Low to moderate, and it rides on prototype 1. **1–2 days** to add a phase-optimisation step to an existing loop (even a greedy or random-restart search answers the question "is there anything here?"). **1–2 weeks** for a principled formulation, since it is non-convex. Cheap to falsify, which is the right property for the most speculative idea on the list.

**Main risk (c).** Phase manipulation of the Residual may trade harmonic distortion for temporal smearing — swapping one Artifact class for another, per the glossary's note that spectral processing produces smearing and pre-echo. It may also simply not help much; a quick empirical bound is worth more than theory here.

### 3. Optimal per-iteration constraint relaxation, borrowed from PAPR
**Smallest, most certain win. Good second task.**

**Core idea.** `apsyclip` relaxes the Masking Threshold by a heuristic each iteration — at least 1 dB, scaled by `adaptive`, growing toward the peak overshoot ratio, with an ad hoc `delta_boost` of 2 in the final third **(a)**. The PAPR literature showed that replacing ICF's fixed filter with a per-iteration filter chosen by convex optimization cuts iterations from 8–16 to 1–2 **(b)**. The same substitution applies: choose each iteration's relaxation to minimise total masking violation subject to hitting the peak target, instead of guessing.

**Implementation / licence.** The audio version does not exist. The PAPR formulation is published (Wang & Luo, IEEE Trans. Commun.); no code found. SPADE's ADMM skeleton (BSD-3-Clause, MIT-compatible) is reusable as solver scaffolding.

**Prototype cost.** Low, **2–3 days** on top of prototype 1, since it is a modification to one function. With unlimited latency, iteration count is cheap, so the payoff is better *quality at fixed Reduction Depth* rather than speed — a different and better bargain than the one the PAPR authors were making.

**Honest caveat (c).** This is incremental and its ceiling is bounded by Gerzon–Craven. It will not make the project novel on its own; it makes an existing algorithm better and is a reliable use of a few days.

### Not recommended
**Noise shaping / error feedback on the Residual.** The unbounded-error windup problem is real and documented **(b)**, the only direct audio treatment found is a self-declared untested AI-generated forum post, and Gerzon–Craven caps the payoff. The frequency-domain projection in prototype 1 achieves the same end — a Residual shaped to the Masking Threshold — without the stability problem. Revisit only if prototype 1's frame structure proves unworkable.

**Tone injection / selected mapping from PAPR.** Both depend on a decoder that a mastering chain does not have **(b)**. Dead end; the ticket's hypothesis is answered no for these specifically.

---

## What I could not confirm

- **Defraene's figures 5 and 7** (the PEAQ/Rnonlin curves) are images; I verified the statistical claims and the t-test outcomes from the text but have no numeric ODG values except the per-excerpt table at clipping factors 0.99 and 0.90, which I did not transcribe. The *direction* and significance are solid **(a)**; the magnitudes are not in hand.
- **Gerzon & Craven AES preprint 2822 (1989)** — full text not obtained; the noise shaping theorem statement above is **(b)**, from multiple consistent secondary restatements. Given how much weight the verdict places on it, worth buying from the AES e-library.
- **arXiv 2202.08183's authors, venue and solver.** The HTML lacks a full author list, names no journal despite an IEEE copyright notice, and never states which solver it uses for either convex problem. Its evaluation at 1 kHz on kick drums is too narrow to lean on.
- **SPADE's BSD-3-Clause licence** is stated on the INRIA project pages **(b)**; I did not fetch an actual LICENSE file. Verify before using any of it. SpadePort's licence is unstated.
- **No AES paper found** combining a masking-constrained clipper with a mastering context. The AES e-library is not fully searchable without a subscription, so I cannot claim it does not exist — this is a negative result with limited confidence. Vickers, "The loudness war: Background, speculation, and recommendations" (AES Convention 129, 2010) appeared only in reference lists.
- **No listening test of `apsyclip` or Calf's Psychoacoustic Clipper** found anywhere **(a, negative)**. Searched Hydrogenaudio and the repo. Absence of evidence rather than evidence of absence, but I looked specifically and found nothing.
- **Patent status** of US9225310B1 beyond the 8th-year fee (2023); the 12th-year fee is not yet due. Current in-force status should be checked directly with the USPTO, and the whole patent question needs an attorney, not me.
- **The POCS/alternating-projections framing of ICF** in section 3 is my own **(c)**. I searched for a source stating it and the search explicitly reported finding none. Treat the empty-intersection warning as a hypothesis to test numerically, not an established result — though it is cheap to test and the `apsyclip` source's relaxation heuristic is strong circumstantial evidence that its author hit exactly this problem.

---

## Sources

Primary, read directly:
- `libavfilter/af_apsyclip.c`, FFmpeg master — https://raw.githubusercontent.com/FFmpeg/FFmpeg/master/libavfilter/af_apsyclip.c (LGPL-2.1+)
- FFmpeg filter docs, `apsyclip` — https://ffmpeg.org/ffmpeg-filters.html
- ShapingClipper — https://github.com/jcj83429/ShapingClipper (no licence; confirmed via GitHub API)
- Defraene et al., IEEE TASLP 2012 — https://ftp.esat.kuleuven.be/pub/stadius/bdefraen/reports/11-127.pdf · DOI 10.1109/TASL.2012.2210875 · sound examples: https://ftp.esat.kuleuven.be/pub/stadius/vanwaterschoot/abstracts/11-127.html
- Defraene et al., ICASSP 2011 — https://ftp.esat.kuleuven.be/pub/stadius/bdefraen/reports/10-244.pdf
- Defraene et al., perceptual compressed sensing declipping, IEEE TASLP 2013 — https://ftp.esat.kuleuven.be/pub/stadius/bdefraen/reports/13-35.pdf
- US9225310B1 — https://patents.google.com/patent/US9225310B1/en
- US11881829B2 — https://patents.google.com/patent/US11881829B2/en
- arXiv 2202.08183 — https://arxiv.org/html/2202.08183
- arXiv 1101.4335 — https://arxiv.org/abs/1101.4335
- van Waterschoot publication list — https://tvanwate.github.io/publications/

Secondary:
- SPADE project — https://spade.inria.fr/ · toolboxes: https://perso.ens-lyon.fr/remi.gribonval/spade-evaluation-paper-toolboxes/
- S-SPADE ADMM derivation — https://arxiv.org/html/1809.09847v1 · SPAIN — https://arxiv.org/html/1810.13137
- Psychoacoustically motivated declipping, weighted L1 — https://arxiv.org/pdf/1905.00628
- Optimized ICF for PAPR — https://web.xidian.edu.cn/home/ychwang/files/20111209_164923.pdf · ICF/ADMM mixed-numerology — https://arxiv.org/pdf/1911.01569
- Calf Studio Gear — https://github.com/calf-studio-gear/calf (LGPL-2.1)
- DAFx 2016 perceptually informed allpass decorrelation — https://ccrma.stanford.edu/~kermit/website/papers/decorrelation_DAFx2016.pdf · DAFx 2023 — https://www.dafx.de/paper-archive/2023/DAFx23_paper_3.pdf · DAFx 2017 velvet noise — http://www.dafx17.eca.ed.ac.uk/papers/DAFx17_paper_96.pdf
- Orban US5737434 · US4208548 · US4241266 · US7411527
- KVR thread on noise-shaping clipping residue (self-declared AI-generated and untested) — https://www.kvraudio.com/forum/viewtopic.php?p=9301762
