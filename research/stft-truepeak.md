# Research: STFT artifact mechanics and BS.1770 True Peak detection requirements

Resolves research ticket #5. Vocabulary per `GLOSSARY.md` (True Peak, Sample Peak, Bin, Artifact, Peak Ceiling, Spectral Magnitude Limiting).

**Confidence legend** used throughout:
- **[A] Verified primary** — read in the owning document (ITU-R recommendation text, EBU recommendation text, peer-reviewed paper, vendor's own documentation).
- **[B] Secondary** — a write-up about a primary source, or a vendor/forum claim not independently checked.
- **[C] Inference** — my own derivation or arithmetic from [A] facts. Flagged inline.

A note on sourcing: I obtained the ITU-R BS.1770-5 and BS.1770-4 PDFs directly from `itu.int` and extracted their text, so every ITU quote below is from the recommendation itself, not a summary of it. Same for EBU Tech 3341, EBU R 128, and Painter & Spanias (Proc. IEEE 2000).

---

## Part 1 — STFT artifact mechanics

### 1.1 The central distinction: a modifying STFT is not an analysis STFT

This is the point the ticket flagged, and it is the correct one to lead with.

**Perfect reconstruction (no modification) requires COLA.** A window `w` gives constant overlap-add at hop size `R` if and only if its transform is zero at all harmonics of the frame rate `2π/R`. Smith states this "gives necessary and sufficient conditions for perfect reconstruction in overlap-add FFT processors." **[A]** (Smith, *Spectral Audio Signal Processing*, "Dual of Constant Overlap-Add".)

**That guarantee is void the moment you modify.** Smith is explicit: "when the short-time spectrum is being modified, these conditions no longer apply," so "a stronger COLA constraint is preferable." **[A]**

**Strong COLA** requires the window transform to be bandlimited, `W(ω) = 0` for `|ω| ≥ π/R`, consistent with downsampling by `R`. Smith: "Strong COLA implies weak COLA, but it cannot be achieved exactly by finite-duration window functions." **[A]**

So the project cannot have strong COLA exactly. It can only approach it, and the approach costs window length. This is a structural limit, not an implementation detail.

### 1.2 The precise mechanism by which per-bin gains break reconstruction

Two distinct failure modes, which are often conflated. Keep them separate.

**Failure mode A — circular convolution / time-domain aliasing (within a frame).**

Multiplying a frame's Bins by a gain vector is multiplication in the DFT domain, which is *circular* convolution in time, not linear convolution. The gain vector has an implied impulse response; if that response is longer than the zero-padding available, its tail wraps around the frame and adds back onto the frame's own start.

The governing condition, from Smith's OLA material: `N ≥ M + L − 1` to avoid time aliasing, where `M` is the frame (window) length, `L` is the maximum length of the FIR filter applied to each frame, and `N` is the FFT size. Smith: "If we don't add enough zeros, some of our convolution terms 'wrap around' and add back upon others." For simple LTI filtering with a rectangular window he recommends `M = N − L + 1`. **[A]** (Smith, "Overlap-Add (OLA) STFT Processing".)

Consequence for this project **[C]**: a per-bin gain vector that varies *abruptly across Bins* has a long effective impulse response, and therefore a large `L`. A smooth gain curve has a short one. So **smoothing the gain across Bins is not cosmetic — it is what keeps `L` small enough for `N ≥ M + L − 1` to hold.** If the project uses the whole FFT frame as the window (`M = N`, no zero-padding), then `L` must be 1, i.e. no modification at all, for strict freedom from aliasing. Any real per-bin gain therefore produces *some* time-domain aliasing; the design question is how much and whether it is audible.

Mitigating note, from the DAFx literature **[B]**: when the zero-padding conditions are not met, the time-domain aliasing from circular convolution is usually inaudible if a sufficiently tapered analysis and synthesis window are used. This is the escape hatch real implementations rely on, and it is a claim to verify by listening, not to assume (cf. the `Transparent` entry in `GLOSSARY.md`).

**Failure mode B — gains differing between adjacent frames (across the overlap).**

COLA says the shifted windows sum to a constant. If frame `m` is scaled by `g_m` and frame `m+1` by `g_{m+1}`, the overlap region no longer sums to a constant but to a window-weighted blend of the two gains. Within a single hop the output gain therefore ramps from `g_m` to `g_{m+1}` with a shape set by the window, not by any gain law the designer chose. Two consequences **[C]**:

1. The effective gain trajectory is low-pass filtered by the window's overlap shape. The project does not get to specify its own attack/release independently of window and hop — the window imposes a floor on how fast the applied Reduction Depth can change.
2. Where the two frames' gains differ sharply, the blend is an amplitude modulation at the frame rate `fs/hop`. Broadband, frame-synchronous gain jitter is audible as a buzz or roughness at the frame rate.

The literature's name for the per-Bin version of this is **musical noise**. Breithaupt, Gerkmann & Martin describe the mechanism exactly: isolated Bins that escape attenuation due to estimation error become "narrow spectral peaks of single outliers [that] appear only for a duration of a single frame." **[A]** That one-frame lifetime is the signature — in `GLOSSARY.md` terms this is an Artifact of spectral processing, distinct from clipping's harmonic distortion.

### 1.3 Pre-echo: mechanism and the concrete window-length relationship

**Mechanism.** Painter & Spanias, verbatim: "Pre-echoes occur when a signal with a sharp attack begins near the end of a transform block immediately following a region of low energy." And the crucial step: "time-frequency uncertainty dictates that the inverse transform will spread quantization distortion evenly in time throughout the reconstructed block. This results in unmasked distortion throughout the low-energy region preceding in time the signal attack at the decoder." **[A]** (Painter & Spanias, "Perceptual Coding of Digital Audio", *Proc. IEEE* 88(4), April 2000.)

Their figure 14 caption makes the scale concrete: the castanets pre-echo example is at a **2048-point block size**. **[A]**

Translated to this project **[C]**: substitute "gain reduction applied to the whole frame" for "quantization noise". The logic is identical and the ticket's framing is correct. A Bin's gain is constant over the entire window. If the window contains a transient, the reduction that transient provoked is applied to *every sample in the window*, including the quiet samples before the attack. The pre-transient portion is ducked for no reason the listener can attribute to anything — which is the pre-echo (for a reduction it is heard as a pre-dip or "suck-in" rather than an added noise burst, but the time-spreading mechanism is the same).

**The concrete duration relationship.** The Fraunhofer/AudioLabs Erlangen AES coding tutorial states that the quantization noise "may now spread out in time over the complete block" and that it can precede the attack by **up to a full block**. **[A]**

So: **pre-echo duration ≈ window length, not hop length.** Overlap does not reduce it. Doubling the FFT size doubles the pre-echo duration. This is the single most important number for the FFT-size fog item.

**The audibility budget.** Painter & Spanias, verbatim: "premasking tends to last only about 5 ms". Their figure 7 caption: "Pre-masking occurs prior to masker onset and lasts only a few milliseconds". **[A]** They add the design rule directly: "temporal premasking is possible only if the transform block size is sufficiently small (minimal coder delay)." **[A]**

### 1.4 The resulting tension, in numbers

Derived arithmetic **[C]**, using the ~5 ms premasking budget from §1.3 **[A]**:

| FFT size `N` | Window duration @ 48 kHz | Bin spacing @ 48 kHz | `N` to hold same duration @ 192 kHz |
|---|---|---|---|
| 256 | 5.33 ms | 187.5 Hz | 1024 |
| 512 | 10.67 ms | 93.8 Hz | 2048 |
| 1024 | 21.33 ms | 46.9 Hz | 4096 |
| 2048 | 42.67 ms | 23.4 Hz | 8192 |
| 4096 | 85.33 ms | 11.7 Hz | 16384 |
| 8192 | 170.67 ms | 5.9 Hz | 32768 |

Largest window whose pre-echo fits inside the ~5 ms premasking budget, per sample rate **[C]**:

| Sample rate | Max `N` for ≤5 ms window | Resulting Bin spacing |
|---|---|---|
| 44.1 kHz | 220 | 200 Hz |
| 48 kHz | 240 | 200 Hz |
| 88.2 kHz | 441 | 200 Hz |
| 96 kHz | 480 | 200 Hz |
| 176.4 kHz | 882 | 200 Hz |
| 192 kHz | 960 | 200 Hz |

The settled decision to scale FFT size with sample rate to hold time/frequency resolution constant means Bin spacing is invariant at 200 Hz across the whole 44.1–192 kHz range — the constraint is a single number, which is convenient.

**The uncomfortable conclusion [C]:** a window short enough for pre-echo to be hidden by premasking gives ~200 Hz Bin spacing. That is coarser than the critical bandwidth below roughly 500 Hz, so Spectral Magnitude Limiting at that resolution cannot separate a fundamental from its neighbours in the bass — exactly where peak energy lives in mastering material. Conversely, a 2048-point window at 48 kHz spreads pre-echo over 42.7 ms, about **8× longer than premasking covers**. There is no FFT size that is simultaneously fine enough in frequency for per-Bin work and short enough in time for pre-echo to be inaudible by premasking alone. **Any viable design must therefore either (a) use a mitigation from §1.5, or (b) keep the per-frame gain change small enough that the pre-echo sits beneath the Masking Threshold on its own merits** — which is the project's existing residual-shaping strategy and is the more promising route given the 3–6 dB Reduction Depth target.

Note that at 3–6 dB Reduction Depth the pre-echo is a 3–6 dB dip, not a full-scale noise burst, so (b) is not hopeless. But it must be tested at the Pass Gate, not assumed.

### 1.5 Mitigations, with what each costs

**Window switching (the AAC approach).** Painter & Spanias: window switching changes the analysis block length from "long" duration (e.g. 25 ms) during stationary segments to "short" duration (e.g. 4 ms) when transients are detected. **[A]** For AAC specifically, verbatim: "the AAC filter bank resolution is signal adaptive. Stationary signals are analyzed with a 2048-point window, while transients are analyzed with a block of eight 256-point windows… The maximum frequency resolution is therefore 23 Hz for a 48 kHz sample rate, and the maximum time resolution is 2.6 milliseconds." **[A]** (2.6 ms is the 128-sample hop of a 256-sample window at 50% overlap, not the window itself — worth noting so the figure is not misread. **[C]**)

Costs, all from Painter & Spanias **[A]**:
- Transition windows are mandatory: "To satisfy PR constraints, window switching typically requires transition windows between the long and short blocks. Even when suitable transition windows satisfy the PR constraints, they do so at the expense of poor time [and frequency localisation]."
- "increased coder delay" — irrelevant here, latency is unlimited.
- "undesirable latency for closely spaced transients (e.g., long-start-short-stop-start-short)" — the switching state machine cannot keep up with dense transients.
- "impractical overusage of short windows for 'pitched' signals". They note pitched material can have impulsive bursts only 8 ms apart, "much less than the typical analysis block length."

**A switching-specific hazard for a *modifying* STFT.** This is the sharpest finding on the mitigation side and it is easy to miss. The DAFx-11 multi-resolution paper states plainly: "It is not guaranteed that an optimal decision before the spectrum modification remains optimal during the modification." **[A]** (Dual-resolution phase estimation, DAFx-11.) The transient detector must choose the window *before* the gains are known, but the right window depends on what the gains turn out to be. A coder can afford this because it only needs the window to be good for quantisation; a modifier cannot, because the modification itself changes which resolution was correct. The same paper notes that with switching, algorithms "do not get the whole spectrogram for processing. Instead, they get only the frames the transient detector has allocated to them." **[A]**

**Parallel multi-resolution (the Lukin/Todd approach).** Rather than switching, run several window lengths in parallel and decide per frame afterwards which performed best. **[A]** (Described in the DAFx-11 paper above; originates in A. Lukin and J. Todd, "Adaptive time-frequency resolution for analysis and processing of audio," AES 120th Convention.) This sidesteps the decide-before-you-know problem because the decision is made after modification. Costs: N× the compute, and the paper warns that naive mixing of parallel results "can lead to phase cancellations," so they instead "perform a strong decision for every frame which time-frequency resolution is most appropriate." **[A]** Given unlimited latency and a mastering (not live) use case, this is the better-matched family for this project. **[C]**

**Gain smoothing — and it is genuinely two-sided.** Breithaupt et al. give the mechanism for why smoothing works: "smoothing them along time prevents single coefficients of the filter function from changing excessively and independently of their neighboring bins, thus suppressing musical noise." **[A]** But they also quantify the trade directly:
- Too little smoothing: "a lower value of the smoothing constant α leads to an increased amount of these artifacts"; "with lower values of α the histogram has heavier tails caused by more outliers… Correspondingly, the amount of musical noise perceivable in the filtered signal increases." **[A]**
- Too much smoothing: their own parameters "result in a slight noise shaping at the end of words… which makes the speech sound slightly reverberant." **[A]** This is temporal smearing — the other Artifact named in `GLOSSARY.md`.
- Onsets are the thing smoothing destroys: "Speech onsets and the spectral envelope of fricatives and plosives must not be distorted by the smoothing procedure. Therefore, the smoothing is not applied to low cepstral coefficients." **[A]**

Their fix is **cepstral** smoothing — smooth only the higher cepstral coefficients (fine spectral structure), leave the low ones (spectral envelope) and the pitch-related ones alone. **[A]** The insight transfers: smoothing in the cepstral domain lets you smooth *fine* structure across Bins without smearing the envelope, which is what blunt across-Bin averaging would do.

So to answer the ticket's question directly: **both** across-frames and across-Bins smoothing are standard, they address different failure modes (§1.2 mode B and mode A respectively), and neither is free. Across-frames smoothing costs transient definition and, pushed far, adds reverberance. Across-Bins smoothing costs spectral selectivity — which is the whole point of Spectral Magnitude Limiting, so it is the more expensive of the two here. Cepstral smoothing is the literature's answer to getting across-Bins smoothing cheaper. **[A]/[C]**

**Temporal Noise Shaping (TNS).** Worth knowing as the fourth family. Apply linear prediction *across frequency* rather than across time; the convolution in the spectral domain corresponds to multiplication in time, so the LP synthesis filter's envelope shapes the added noise to follow the signal's own temporal envelope. **[A]** (Painter & Spanias.) It is the one technique that attacks pre-echo without shortening the window, which given the §1.4 deadlock makes it the most interesting unexplored option for this project. **[C]** It is also the most involved to implement, and I did not research its interaction with a magnitude-only modification (TNS in AAC operates on MDCT coefficients that carry sign, not on magnitudes) — flagged as an open question.

### 1.6 What window shape and overlap actually buy

- **Window length** sets pre-echo duration (≈ the full window) and Bin spacing. The dominant parameter. **[A]**
- **Window shape** sets sidelobe leakage, hence how much a loud Bin contaminates its neighbours' gain decisions, and it sets the overlap blend shape from §1.2 mode B. Painter & Spanias note AAC additionally switches between two MDCT window *shapes* to "optimize filter bank frequency selectivity." **[A]**
- **Overlap/hop** sets the frame rate, hence how fast gains may change, and determines which COLA condition is satisfiable. It does **not** reduce pre-echo. **[A]/[C]** Higher overlap gives finer gain-trajectory time resolution and a higher modulation frequency for mode-B artifacts (pushing the buzz up out of the most sensitive range), at linear compute cost. **[C]**

---

## Part 2 — True Peak detection per BS.1770

### 2.1 BS.1770-4 vs BS.1770-5: which to cite

**BS.1770-5 exists and is current, dated November 2023.** BS.1770-4 is dated October 2015. **[A]**

**Annex 2 (the True Peak algorithm) is substantively unchanged between -4 and -5.** I verified this by extracting and comparing both documents' Annex 2 text directly: identical prose, identical stage list, identical "order 48, 4-phase, FIR interpolating" description, identical coefficient values, and an identical under-read table (0.554 / 0.688 at 4×). The one difference I found is editorial: "**Appendix** 1 to Annex 2" in -4 was renamed "**Attachment** 1 to Annex 2" in -5. **[A]**

Caveat on method **[A]**: my text extractor dropped the negative-signed coefficient rows from the -4 PDF (a font-encoding artifact), so I confirmed the positive-tap values byte-for-byte across both revisions and all surrounding prose, but could not byte-compare the negative taps. Everything verifiable matched, so I am confident the tables are the same, but this is the one place I stopped short of proof.

**Recommendation:** cite **BS.1770-5** going forward and note it supersedes -4. The project's existing `GLOSSARY.md` entry for True Peak says "measured per ITU-R BS.1770-4"; that should be updated to -5. Nothing about the measurement changes, so no design consequence — it is a citation-hygiene fix. (Note that -4 → -5 did change other things outside Annex 2, chiefly around loudness for advanced sound systems; I did not audit those, as they do not bear on this ticket.)

### 2.2 What Annex 2 actually requires

Verbatim from BS.1770-5 Annex 2 **[A]**. The five stages:

1. Attenuate: 12.04 dB attenuation
2. 4× over-sampling
3. Low-pass filter
4. Absolute: Absolute value
5. Conversion to dB TP

Key provisions, quoted:

- **The attenuation is optional in floating point.** "The first step consists of imposing an attenuation of 12.04 dB (2-bit shift). The purpose of this step is to provide headroom for the subsequent signal processing that could employ integer arithmetic. **This step is not necessary if the calculations are performed in floating point.**" So a float implementation skips stages 1 and the compensation entirely.

- **4× is specified relative to 48 kHz, and higher is preferred.** "The 4× over-sampling filter increases the sampling rate of the signal from 48 kHz to 192 kHz… **Higher sampling rates and over-sampling ratios are preferred** (see Attachment 1 to this Annex). Incoming signals that are at higher sampling rates require proportionately less over-sampling (e.g. for an incoming signal at 96 kHz sample rate a 2× over-sampling would be sufficient.)"

- **The filter coefficients are illustrative, not mandatory.** This is the single most-misreported fact about BS.1770 and the exact wording matters: "**One set of filter coefficients** (for the order 48, 4-phase, FIR interpolating) **that would satisfy the requirements would be as follows**:" — followed by the 4×12 coefficient table (phase 0 begins 0.0017089843750, peak tap 0.9721679687500). So: a 48-tap, 4-phase polyphase FIR is the *example*, not the specification. Any filter meeting the requirements is compliant. **[A]**

- **The only quasi-normative accuracy hook is an absolute rate, not a ratio.** "Meters that follow these guidelines, and that **use an oversampled sampling rate of at least 192 kHz**, should indicate the result in the units of dB TP." **[A]**

- **BS.1770 itself states no numerical tolerance.** I grepped both revisions for tolerance and accuracy language in Annex 2 and there is none. Annex 2 opens by calling itself "an algorithm for **estimation** of true-peak level" (emphasis mine — the recommendation's own word). The tolerance comes from EBU, not ITU (§2.3). **[A]**

**Consequence for this project [C]:** because the hook is "at least 192 kHz" in absolute terms rather than "4×", the required oversampling factor falls as the input rate rises. At 192 kHz input, the letter of the recommendation requires **no oversampling at all**. That is a real and slightly surprising saving for the project's high-rate modes — and §2.4 shows it is also defensible, not just a loophole, because what bounds the error is the absolute oversampled rate relative to content bandwidth.

### 2.3 The tolerance lives in EBU Tech 3341, not in BS.1770

EBU Tech 3341 §2.6, verbatim: "**In addition to the specifications of ITU-R BS.1770 Annex 2**, the true-peak measure implemented in an [EBU Mode meter shall comply with the test signals in Table] 1 (signals 15-23)… The **total measurement error of the true-peak level must comply with these tests, including any pass-band ripple** [of the up-sampling filter, as described in] ITU-R BS.1770, Appendix 1 to Annex 2." **[A]**

(Square brackets mark short spans my text extraction lost; the surrounding text is verbatim and the sense is unambiguous.)

**The tolerance is `+0.2 / −0.4 dBTP`**, applied to test cases 15–23. **[A]** Note it is asymmetric: a meter may over-read by 0.2 dB but may under-read by up to 0.4 dB. And explicitly, the interpolation filter's pass-band ripple counts against this budget — the under-read bound of §2.4 is not the whole error.

The test signals themselves (all `+0.2/−0.4 dBTP`) **[A]**:
- #15–#18: stereo sine at `fs/4`, `fs/4`, `fs/6`, `fs/8`, at phases 0°, 45°, 60°, 67.5°, amplitude 0.50 FFS → expect −6.0 dBTP
- #19: `fs/4`, amplitude 1.41 FFS, phase 45° → expect **+3.0** dBTP (i.e. a compliant meter must read *above* full scale)
- #20–#23: a `fs/6` sine containing a single period of `fs/4` at amplitude 1.00, "synthesized at `4*fs` (for example, 192 kHz), and then lowpass (anti-aliasing) filtered and downsampled to `fs`" with **0, 1, 2 and 3 sample offsets** at the `4*fs` rate → expect 0.0 dBTP. These four are the ones that specifically probe whether the meter finds a peak sitting at an awkward sub-sample phase.

**EBU R 128** sets the production limit: "−1 dBTP (dB True Peak) during production (linear audio), measured with a meter compliant with ITU-R BS.1770 [and EBU Tech 3341]." **[A]**

**A load-bearing gap I found [C].** The highest-frequency Tech 3341 true-peak test tone is at `fs/4`. Using the ITU's own under-read formula (§2.4), a 4× meter's worst-case under-read at `f_norm = 0.25` is **0.169 dB** — comfortably inside the −0.4 dB tolerance. Solving for where 4× exactly reaches −0.4 dB gives `f_norm = 0.383`, i.e. **18.4 kHz at 48 kHz**. So:

> **Tech 3341 conformance at 4× certifies nothing about content between ~0.383·fs and 0.5·fs.** A 4× meter can pass every Tech 3341 true-peak test and still under-read by up to 0.688 dB on content near Nyquist.

For a mastering tool this matters: cymbals, distorted guitar, and anything heavily limited carry real energy up to 20 kHz+ at 44.1/48 kHz. "BS.1770-compliant" and "Tech 3341-conformant" therefore do **not** mean "will not overshoot."

### 2.4 Measure versus guarantee — the heart of the ticket

The ticket asks whether 4× suffices to *guarantee* a ceiling or only to *measure* one. **The answer is unambiguous: 4× does not guarantee, and no finite oversampling factor guarantees. But the ITU gives a worst-case bound, which can be converted into a guarantee by backing the ceiling off by that bound.** This is the actionable result of this half of the ticket.

**The ITU's own bound.** Attachment 1 to Annex 2 (informative) derives it. The reasoning, verbatim in part: consider "what under-read will result from a pair of samples at the over-sampled rate occurring symmetrically either side of the peak of a sinusoid at our maximum [normalized frequency]", with "(2 in denominator since a peak can be missed by a maximum of half the over-sampling period)". **[A]**

The resulting formula, as printed (with the glyphs my extraction lost reconstructed and then **verified numerically**, see below):

```
maximum under-read (dB) = −20 · log10( cos( π · f_norm / n ) )
```

where `n` is the oversampling ratio and `f_norm` the highest frequency to be metered as a fraction of the sampling frequency.

The ITU's printed table, "which probably covers the range of interest" **[A]**:

| Over-sampling ratio | Under-read (dB), `f_norm` = 0.45 | Under-read (dB), `f_norm` = 0.5 |
|---|---|---|
| 4 | 0.554 | 0.688 |
| 8 | 0.136 | 0.169 |
| 10 | 0.087 | 0.108 |
| 12 | 0.060 | 0.075 |
| 14 | 0.044 | 0.055 |
| 16 | 0.034 | 0.042 |
| 32 | 0.008 | 0.010 |

I reimplemented the formula and it reproduces all fourteen printed values exactly (0.554, 0.688, 0.136, 0.169, 0.087, 0.108, 0.060, 0.075, 0.044, 0.055, 0.034, 0.042, 0.008, 0.010). **[C, verifying [A]]** The formula is therefore safe to use directly in the project rather than interpolating the table.

**So, concretely:**

- **4× is a *measurement* method with a ~0.69 dB worst-case under-read** at `f_norm = 0.5`. It is sufficient to measure to Tech 3341 tolerance *for the frequencies Tech 3341 tests* (§2.3), and not sufficient to bound error near Nyquist.
- **To turn it into a guarantee, subtract the bound.** With 4× detection and a Peak Ceiling target of 0 dBTP, setting the internal ceiling to **−0.7 dBTP** guarantees the true ceiling is met for worst-case near-Nyquist content. At 8×, the back-off is 0.17 dB; at 16×, 0.05 dB. This is the clean engineering answer and it makes the cost of each factor legible: **the price of 4× is 0.5 dB of headroom you must give away; the price of 16× is 0.05 dB of headroom plus 4× the oversampling compute.** **[C, from [A]]**
- Add the interpolation filter's pass-band ripple on top, which Tech 3341 explicitly counts as part of total error. **[A]**

**Caveats on the bound [C].** The ITU derivation is for a sinusoid at `f_norm` sampled symmetrically about its peak. It is a worst case for a single tone, and I have not found a proof that it bounds arbitrary band-limited signals. I would treat it as a well-founded engineering bound rather than a theorem — adequate for choosing a back-off, not adequate for claiming a mathematical Peak Ceiling. **A true mathematical guarantee requires a time-domain mechanism**, which is consistent with what `GLOSSARY.md` already says: Clipping "is the only one in this project that enforces a peak ceiling intrinsically." Nothing in this research contradicts that; it quantifies how close oversampled detection can get.

### 2.5 Sample-rate scaling of the oversampling factor

Derived from the ITU formula **[C]**, under-read in dB for content up to 22 kHz:

| Input rate | ×1 | ×2 | ×4 | ×8 | ×16 | ×32 |
|---|---|---|---|---|---|---|
| 44.1 kHz | 48.97 | 2.995 | 0.684 | 0.168 | 0.042 | 0.010 |
| 48 kHz | 17.69 | 2.477 | 0.575 | 0.141 | 0.035 | 0.009 |
| 88.2 kHz | 2.995 | 0.684 | 0.168 | 0.042 | 0.010 | 0.003 |
| 96 kHz | 2.477 | 0.575 | 0.141 | 0.035 | 0.009 | 0.002 |
| 176.4 kHz | 0.684 | 0.168 | 0.042 | 0.010 | 0.003 | 0.001 |
| 192 kHz | 0.575 | 0.141 | 0.035 | 0.009 | 0.002 | 0.001 |

The table makes the invariant obvious and confirms the ITU's "proportionately less over-sampling" remark: **what bounds the error is the absolute oversampled rate, not the ratio.** The diagonals are equal. So the project should parameterise oversampling as a *target oversampled rate*, not a fixed factor — exactly parallel to the settled decision to scale FFT size with sample rate. **[C]**

Required oversampled rate to bound under-read for 22 kHz content **[C]**:

| Target bound | Required oversampled rate | Equivalent factor at 48 kHz |
|---|---|---|
| 0.10 dB | ≥ 456 kHz | 9.5× |
| 0.05 dB | ≥ 645 kHz | 13.4× |
| 0.01 dB | ≥ 1441 kHz | 30× |

Given **latency is unlimited** and this is a mastering tool, the compute argument for staying at 4× is weak. A target of ~768 kHz (16× at 48 kHz, 4× at 192 kHz) buys a ~0.04 dB bound, which is small enough that the back-off is negligible and the Peak Ceiling can sit at −0.1 dBTP rather than −0.7. **[C]**

### 2.6 What commercial True Peak limiters actually do

**FabFilter Pro-L 2** is the best-documented, and its own help pages are primary **[A]**:
- Purpose: "To guarantee that FabFilter Pro-L 2's output doesn't exceed a certain maximum true peak level."
- The recommended configuration and its *measured residual*: "choosing 4x oversampling in combination with a minimum lookahead time of 0.1 ms" **"keeps inter-sample peaks within a range of 0.1 dB."**
- Two-stage structure: "The limiting process will detect true peaks that already exist in the input signal, and reduce them appropriately," and "Any true peak overshoot introduced by the ultra-fast limiting process itself, will be smoothly attenuated afterwards."
- Cost: "about 5 ms" extra latency.
- On oversampling generally: 4× or 8× for normal use; "16x and 32x options may result in even higher audio quality, but for most systems this is simply too taxing to run in real-time" — advised for offline rendering. **[A]**
- Historically honest about the failure: in Pro-L 1, "enabling oversampling could generate output levels that were slightly higher than the ceiling set by the Output Level parameter"; fixed in Pro-L 2. **[A]**

**The important reading of the 0.1 dB figure [C]:** FabFilter achieves ~0.1 dB residual at 4× — better than the 0.69 dB worst-case bound of §2.4 — and the mechanism is the **second stage**, not the oversampling. They detect, limit, then re-attenuate whatever the limiter itself overshot. This is the architectural lesson for this project: **a bounded Peak Ceiling comes from a corrective output stage, not from making detection finer.** It also matches §2.4's conclusion that detection alone cannot guarantee. A spectral stage followed by a time-domain corrective stage is the structure that actually delivers a ceiling — and `GLOSSARY.md` already anticipates this in noting that Spectral Magnitude Limiting "makes *no* guarantee about the resulting waveform's peak."

**iZotope Ozone Maximizer** **[B]** — True Peak mode "oversample[s] the audio, reading it again and placing additional sample points between the existing ones"; iZotope does not publish the factor. They note the inter-sample gap "can exceed its corresponding digital signal's peak level by more than 3 dB", and caution that "different True Peak meters have different oversampling algorithms and may produce slightly different level readings." One published review measured a −1 dBFS ceiling holding true peaks at about −0.9 dBFS (single uncontrolled test).

**EBU Tech 3344** **[B]** (I could not extract this PDF's text, so these are from secondary reporting and should be confirmed before being relied on) — notes that overshoot can exist between a 4×-oversampled True Peak measurement and the actual signal; cites AES TD1008 as recommending a true-peak limiter that anticipates peak level *after* the playback device's DAC and reconstruction filter; and warns that codec and sample-rate-converter overshoots "can be additive". If true, the last point matters for the project's output-stage design: the ceiling that survives downstream encoding is not the ceiling measured at the plugin output.

**Common practice** **[B]** — the widely-repeated mastering advice is 4× or more oversampling *plus* a ceiling set below target (typically 0.2–0.5 dB below). That is the same back-off strategy §2.4 derives from first principles, arrived at empirically.

---

## Answers to the ticket, condensed

**Part 1**
1. Window **length** is the dominant parameter: pre-echo lasts up to one full window and is unaffected by overlap. **[A]**
2. COLA is necessary and sufficient for perfect reconstruction *only without modification*; once you modify, those conditions "no longer apply" and strong COLA is preferable — but strong COLA is unachievable with any finite window. **[A]**
3. Per-Bin gains break reconstruction two ways: circular-convolution time aliasing within a frame, bounded by `N ≥ M + L − 1` (so across-Bin gain smoothness controls `L`); and gain mismatch across the overlap, which makes the window dictate the gain trajectory and produces frame-rate modulation and one-frame-long musical noise. **[A]/[C]**
4. Gain smoothing across frames *and* across Bins are both standard and address the two different modes. Across-frames costs transient definition and adds reverberance when overdone; across-Bins costs the spectral selectivity that is the point of the whole mechanism. Cepstral smoothing is the literature's way to get across-Bin smoothing without smearing the envelope. **[A]**
5. There is no FFT size that is both short enough for premasking (~5 ms, ~200 Hz Bins) and fine enough in frequency for per-Bin work. The route through is residual shaping beneath the Masking Threshold, or parallel multi-resolution, or TNS — **not** window size alone. **[C]**
6. AAC-style window switching has a specific hazard for a modifier: the resolution decision must be made before the gains are known, and "it is not guaranteed that an optimal decision before the spectrum modification remains optimal during the modification." Parallel multi-resolution (Lukin/Todd) avoids this and suits unlimited latency. **[A]**

**Part 2**
1. BS.1770 requires: 12.04 dB attenuation (optional in float) → 4× oversampling → low-pass → absolute value → dB TP. Coefficients are an **example** ("One set of filter coefficients… that would satisfy the requirements"), not a specification. **[A]**
2. BS.1770 states **no tolerance** and calls itself an "estimation" algorithm. The `+0.2/−0.4 dBTP` tolerance is EBU Tech 3341's, and it explicitly includes the interpolation filter's pass-band ripple. **[A]**
3. **4× measures; it does not guarantee.** Worst-case under-read is 0.688 dB at `f_norm = 0.5`, per the ITU's own informative table, whose formula I verified reproduces all printed values. **[A]/[C]**
4. A guarantee is obtained by backing the ceiling off by the bound: −0.7 dB at 4×, −0.17 dB at 8×, −0.05 dB at 16×. **[C]**
5. The cost of going beyond 4× is compute only, and with unlimited latency that argument is weak. Parameterise as a **target oversampled rate** (~768 kHz) rather than a fixed factor, mirroring the FFT-size decision. **[C]**
6. Tech 3341's highest true-peak test tone is `fs/4`, so **4× conformance certifies nothing above ~0.383·fs** (18.4 kHz at 48 kHz). "BS.1770-compliant" ≠ "will not overshoot." **[C]**
7. FabFilter Pro-L 2 holds inter-sample peaks "within a range of 0.1 dB" at 4× — achieved by a **second corrective stage** that re-attenuates the limiter's own overshoot, not by finer detection. That is the architecture that actually produces a Peak Ceiling. **[A]/[C]**
8. BS.1770-**5** (Nov 2023) is current; Annex 2 is substantively identical to -4, the only change I found being "Appendix 1" → "Attachment 1". Update the `GLOSSARY.md` True Peak entry's citation. **[A]**

---

## Could not confirm / open questions

1. **EBU Tech 3344** — PDF text would not extract; the AES TD1008 reference, the additive-overshoot claim, and the post-DAC limiting recommendation are **[B]** only. Worth confirming directly, because the additive-overshoot point affects output-stage design.
2. **Marina Bosi's IEEE perceptual-coding tutorial** — PDF would not extract (newer PDF object streams). Not load-bearing; Painter & Spanias covered the same ground primarily.
3. **ISO/IEC 14496-3 (the AAC standard itself)** is paywalled. The 2048/256 figures come from Painter & Spanias **[A]**, which is authoritative but still one step from the standard.
4. **Ozone's oversampling factor** is not published. **[B]** only.
5. **The ITU under-read bound's generality** — derived for a sinusoid; I found no proof it bounds arbitrary band-limited signals. Treat as an engineering bound, not a theorem. **[C]**
6. **BS.1770-4 negative coefficient rows** could not be byte-compared against -5 due to a font-encoding artifact in my extractor; everything else in Annex 2 matched exactly.
7. **TNS applied to magnitude-only modification** — not researched. AAC's TNS operates on signed MDCT coefficients, not magnitudes, so whether the technique transfers to a magnitude-domain modifier is genuinely open and, given §1.4's deadlock, is probably the highest-value follow-up question.
8. **Pre-echo as a gain *reduction* rather than added noise** — the entire pre-echo literature is about quantisation noise in coders. The mechanism transfers cleanly **[C]**, but I found no source studying the audibility of a pre-transient *dip* of 3–6 dB specifically. Audibility at the project's Reduction Depth target is an empirical question for the Reference Set, not a settled fact.

## Sources

Primary, read directly:
- [Recommendation ITU-R BS.1770-5 (11/2023)](https://www.itu.int/dms_pubrec/itu-r/rec/bs/R-REC-BS.1770-5-202311-I!!PDF-E.pdf) — Annex 2 and Attachment 1 to Annex 2
- [Recommendation ITU-R BS.1770-4 (10/2015)](https://www.itu.int/dms_pubrec/itu-r/rec/bs/R-REC-BS.1770-4-201510-S!!PDF-E.pdf) — Annex 2 and Appendix 1 to Annex 2
- [EBU Tech 3341](https://tech.ebu.ch/docs/tech/tech3341.pdf) — §2.6 and Table 1 test cases 15–23
- [EBU R 128](https://tech.ebu.ch/docs/r/r128.pdf) — −1 dBTP production limit
- [Painter & Spanias, "Perceptual Coding of Digital Audio", Proc. IEEE 88(4), Apr 2000](http://www.mp3-tech.org/programmer/docs/audiopaper1.pdf) — pre-echo, premasking, pre-echo control strategies, AAC filter bank
- [Breithaupt, Gerkmann & Martin, "Cepstral Smoothing of Spectral Filter Gains for Speech Enhancement Without Musical Noise", IEEE SPL 2007](https://uol.de/f/6/dept/mediphysik/ag/speech/download/paper/breithaupt_gerkmann_martin_cepstralSmoothing_spl2007.pdf)
- [Multiresolution STFT phase estimation, DAFx-11](http://recherche.ircam.fr/pub/dafx11/Papers/09_e.pdf) — window-switching drawbacks for modification; Lukin/Todd reference
- [AudioLabs Erlangen, AES coding tutorial: Pre-Echo Distortion](https://www.audiolabs-erlangen.de/content/resources/aesCodingTutorial/preecho.html) and [tutorial paper](https://www.audiolabs-erlangen.de/content/resources/aesCodingTutorial/media/tutorial/aes.pdf)
- [J. O. Smith, Spectral Audio Signal Processing: "Dual of Constant Overlap-Add"](https://www.dsprelated.com/freebooks/sasp/Dual_Constant_Overlap_Add.html)
- [J. O. Smith, "Overlap-Add (OLA) STFT Processing"](https://www.dsprelated.com/freebooks/sasp/Overlap_Add_OLA_STFT_Processing.html)
- [FabFilter Pro-L 2: True peak limiting](https://www.fabfilter.com/help/pro-l/using/truepeaklimiting) and [Oversampling](https://www.fabfilter.com/help/pro-l/using/oversampling)

Secondary:
- [EBU Tech 3344](https://tech.ebu.ch/docs/tech/tech3344v1_1.pdf) (could not extract text; claims are secondary)
- [iZotope Ozone Maximizer documentation](http://help.izotope.com/docs/ozone/pages/modules_loudness_maximizer.htm)
- A. Lukin and J. Todd, "Adaptive time-frequency resolution for analysis and processing of audio," AES 120th Convention (cited via DAFx-11; not read directly)
