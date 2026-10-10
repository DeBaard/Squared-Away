# Prior art: how do existing clippers control true peak?

Research note for issue #2. Researched 2026-10-10.

Vocabulary is per `GLOSSARY.md`. In particular: **Peak Ceiling** is a guarantee about output;
**Clipping** and **Spectral Magnitude Limiting** are mechanisms; only Clipping (a time-domain
mechanism) enforces a Peak Ceiling intrinsically. The whole finding of this note turns on that
distinction, so the glossary's framing is load-bearing rather than decorative.

## Evidence classes

Every claim below carries one of these tags:

- **[V]** Verified from a primary source — the vendor's own manual/documentation, a patent, a
  standards document, or a peer-reviewed/archived paper.
- **[M]** Marketing copy — vendor product page or press release, not documentation. Describes
  intent and positioning, not mechanism.
- **[C]** Community claim — forum post, review, user report. Attributed but unverified.
- **[I]** My own inference, reasoning from the above or from DSP first principles. Not sourced.

---

## 1. Headline answer

**Per-bin independent clipping for Peak Ceiling enforcement is neither unexplored nor
"tried and abandoned". It is a third thing: the mechanism exists and is patented, but
nobody has ever claimed it as a Peak Ceiling mechanism, because it mathematically cannot be
one. What *is* genuinely unexplored is per-bin spectral work as a *pre-conditioner* feeding a
time-domain guarantee stage — and the market has started moving into exactly that space in the
last few years.**

Three separate findings support this:

1. **The mechanism is patented, and the patent does not claim a peak guarantee.** US 9,210,506
   B1, "FFT bin based signal limiting" (Audyssey Laboratories, filed 2011, granted 2015), claims
   precisely Spectral Magnitude Limiting: window → FFT → magnitude/phase → limit the magnitude →
   inverse FFT → overlap-add → drive a transducer. Its stated purpose is raising *perceived
   loudness* without harmonic distortion and protecting a transducer. It makes **no time-domain
   peak claim**, and the specification does not analyse overlap-add peak behaviour at all. **[V]**

2. **The best-informed designers put the guarantee stage after the inverse transform.**
   US 11,496,099 B2 (Mimi Hearing Technologies, 2020/2022) describes an STFT processing chain and
   then explicitly places the limiter on the *time-domain* samples after the inverse transform,
   "on a hop-by-hop basis with the need of only one hop look-ahead". The patent does not argue
   *why* — it simply never considers doing it in the frequency domain. **[V]**

3. **Every shipping product that has a documented hard ceiling enforces it in the time domain,
   with no exceptions found.** Spectral and multiband machinery, where present, decides *where*
   reduction goes; a wideband time-domain stage decides *that the ceiling holds*. See §3.

### Why Spectral Magnitude Limiting cannot enforce a Peak Ceiling

This is the "establish *why*" the ticket asks for. The argument is short, and it is arithmetic
rather than empirical — which is why no amount of prototyping will rescue the approach.

- A sample value is the sum of all bins at that instant, with their phases. Bounding each
  `|X[k]|` does not bound the sum. The only bound available from bin magnitudes alone is the
  triangle inequality, `|x[n]| ≤ (1/N)·Σ|X[k]|`, which is attained when every bin aligns in
  phase. To *guarantee* a ceiling from magnitudes you would have to hold the **sum** of all bin
  magnitudes under the ceiling — which for broadband programme material means reduction depths
  far beyond anything musically usable. **[I]**
- This matches the glossary's own definition: peak level is produced by phase alignment across
  bins, and Spectral Magnitude Limiting leaves phase alone. **[V]**
- Overlap-add compounds it. The output at a sample is the windowed sum of several overlapping
  frames; a per-frame magnitude bound does not survive the sum. **[I]**
- The telecoms literature has the same failure mode under the name **peak regrowth**, and has
  documented it for 20+ years. Armstrong, "Peak-to-average power reduction for OFDM by repeated
  clipping and frequency domain filtering", *Electronics Letters* 38(5):246-247, 2002
  (DOI 10.1049/el:20020175), is the canonical reference: clip in the time domain, filter in the
  frequency domain, and the filtering *puts the peaks back*. The entire literature that follows
  it exists because a single pass does not converge — the standard remedy is to **iterate**
  clip-and-filter, at two FFTs per iteration, and even then the result is statistical, not a
  guarantee. **[V]**
- Consequence for this project: an iterative STFT scheme can reduce the *probability* of
  overshoot but cannot produce a hard guarantee in bounded time. A hard TRUE-peak ceiling — a
  settled project decision — therefore requires a final time-domain stage. **[I]**

### The honest caveat on "hard"

A TRUE-peak ceiling is itself only ever an *approximation*, because true peak is measured by
oversampled detection. ITU-R BS.1770 specifies -12.04 dB pre-attenuation, ≥4x oversampling, a
low-pass, absolute value, and +12.04 dB restoration; it states that higher oversampling ratios
are preferred. Residual under-read is real and frequency-dependent, approaching worst case for
tones near Nyquist. **[V, via Essentia's implementation docs and FabFilter's forum statement;
the ITU PDF itself returned 404 at the canonical URL and I could not read it directly — see §6.]**
FabFilter's own position is that the BS.1770-4 example detection filter "is generally considered
to be sub-optimal, and can give quite significant under- and overreads". **[V — vendor
statement, FabFilter forum]** So "hard TRUE-peak ceiling" in practice means "hard against a
chosen detector at a chosen oversampling factor", and the project should name the detector it
guarantees against. **[I]**

---

## 2. Is there a shipping spectral peak-control product?

Yes — one, and it is the most important commercial data point in this note.

### Leapwing LimitOne (2024/2025)

Marketed verbatim as "A spectral approach to limiting and clipping". **[M]** Architecture, from
the official manual (`Manual-LimitOne.pdf`, manual version 1A) and product page:

- **Pufferfish** — "A frequency domain processor" whose purpose is "an even distribution across
  the whole frequency spectrum for a given Drive and Ceiling level". Also described as "a fully
  automated adaptive multiband dynamics controller". **[V]**
- **Hedgehog** — "A time domain processor whose main purpose is to shape transients", using an
  adaptive control system sharing the same Drive and Ceiling parameters. **[V]**
- "Ceiling is the absolute maximum level that is allowed at the output." **[V]**
- A `Clipping` parameter "controls the balance between Pufferfish and Hedgehog" and at 100%
  gives hard clipping characteristics. **[V]**
- Spectral display: "Frequencies that get attenuated, light up in orange to red. **Frequencies
  that get boosted, light up in yellow to white.**" **[V]**

Three readings of this, in decreasing confidence:

- The frequency-domain stage **boosts as well as attenuates**. It is therefore not Spectral
  Magnitude Limiting at all — it is spectral *redistribution*. That is consistent with
  "even distribution of energy", and it is a different idea from per-bin clipping. **[I,
  from a [V] manual quote]**
- The ceiling is enforced by the time-domain half of the hybrid, not the frequency-domain half.
  The manual's architecture — frequency domain for distribution, time domain for transients and
  for the hard edge — is exactly the split this note recommends. **[I]**
- **The manual never uses the words "true peak", "inter-sample peak", or "oversampling", and
  gives no latency figure.** The only true-peak claim anywhere is a Bob Katz testimonial on the
  product page saying it "nails true peak values with no overshoot" — that is an endorsement
  quote, not documentation. **[M]** So LimitOne's actual true-peak behaviour is **unverified**,
  and it is a direct measurement target for the comparison bench.

LimitOne is the clearest evidence that the spectral direction is live commercial territory rather
than a dead end — *and* that the ceiling in it is not spectral.

---

## 3. The comparison bench, product by product

Columns: detection path / reduction mechanism / spectral or per-band claims.

### FabFilter Pro-L 2 — the best-documented ceiling in the bench

- **Detection:** True Peak Limiting is an explicit mode. Metering "fully complies with the ITU-R
  BS.1770 and EBU R128 standards". Oversampling is user-selectable; 4x recommended. **[V]**
- **Mechanism:** Lookahead brickwall limiting, **two-stage** for the ceiling. Stage one: "The
  limiting process will detect true peaks that already exist in the input signal, and reduce them
  appropriately." Stage two: "Any true peak overshoot introduced by the ultra-fast limiting
  process itself, will be smoothly attenuated afterwards, ensuring the final true peak output
  doesn't exceed the current maximum true peak Output Level." **[V]**
- **The guarantee is stated as a guarantee:** "To guarantee that FabFilter Pro-L 2's output
  doesn't exceed a certain maximum true peak level, enable the True Peak Limiting option." **[V]**
- Also documents the honest version: better to *prevent* overshoot than to catch it. "In most
  cases, choosing 4x oversampling in combination with a minimum lookahead time of 0.1 ms, keeps
  inter-sample peaks within a range of 0.1 dB." **[V]** Extra latency from true-peak limiting:
  about 5 ms. **[V]**
- **Spectral/per-band:** none. Wideband. **[V]**

**This two-stage pattern is the single most reusable finding in the bench.** A mechanism that
reduces peaks *generates* new true peaks; a second, slower, dedicated stage cleans up after it.
Any Squared Away architecture needs the equivalent. **[I]**

### TDR Limiter 6 GE — the most instructive architecture

Six modules, four reorderable (Compressor, Peak Limiter, HF Limiter, Clipper), then a fixed
Output stage and Meter. **[V]**

- **Output (Protection Limiter):** the ceiling stage, fixed last, with "a precise, purely
  technical function". Two ceiling modes: "The PCM Ceiling purely limits the PCM sample values"
  versus "The True Peak ceiling on the other hand limits an approximation (very safe and
  accurate) of the true waveform." **[V]** Note "approximation" — the vendor is explicit that
  true peak is estimated, not exact.
- **Clipper:** hard clipping, modes B. Wall / Open / LF Clipper, plus a **`Separation`** control:
  "Adjusts the band separation in percent", described as "a continuous transition between
  wideband and multiband clipping control". **[V]** So **multiband clipping ships today**, and
  is continuously blendable against wideband.
- **Peak Limiter:** lookahead brickwall. "Limiter 6 uses a small lookahead to anticipate
  overloads before they happen." Multi Band option: "An advanced, cross-linked multiband
  structure essentially combines the advantages of band split limiting"; the manual stresses
  "this is more than just a crossover with 3 limiters". A `B. Wall` cross-band link is described
  as guaranteeing brickwall behaviour. **[V]**
- **HF Limiter:** splits off the HF band and "dynamically subtracting it from the original".
  **[V]** A frequency-selective mechanism, but for tone, not for the ceiling.
- **Global `Processing Quality`** (Eco / Precise / Insane / Low Latency) affects "inter-sample
  accuracy and amount of aliasing by-products" — not tied to a named per-module oversampling
  factor. **[V]**
- Latency reported by a user as 6 samples (Low Latency, "much more aliasing") or 360 samples
  otherwise. **[C]** A user also reports true peak overshooting the ceiling by 0.1 dB as measured
  by Youlean. **[C]** Both worth re-measuring on the bench rather than trusting.

**Reading:** TDR separates *tonal* band-selectivity (HF Limiter, multiband Clipper, multiband
Peak Limiter) from the *ceiling guarantee* (a fixed, wideband, final, time-domain Output stage).
Band-splitting is used for how reduction is distributed; it is never trusted with the ceiling.
**[I]**

### iZotope Ozone Maximizer — the closest thing to spectral peak control in a major product

- **IRC IV** "shapes the spectrum to reduce pumping and distortion". When the signal exceeds the
  threshold, "the IRC IV algorithm limits frequency bands that contribute most to these peaks",
  using "dozens of psychoacoustically spaced bands". "Limiting only the bands that are producing
  the peaks is meant to reduce intermodulation between different signal components." "When no
  limiting is necessary, the spectrum will be unaltered." Most CPU-intensive mode, high latency.
  **[V]**
- iZotope's own background article: spectral shaping analyses "across 32 Mel-spaced frequency
  bands", processed individually and — notably — "without applying crossovers". It describes the
  technique's origin as limiting that "could detect frequency bands which are producing the most
  peaks" and apply limiting "only in those areas and only when peaks were present". First shipped
  as IRC IV in Ozone 7. **[V — vendor technical article; sits between [V] and [M], treat the
  mechanism claims as vendor-asserted]**
- **Documented contradiction:** the same Maximizer documentation states "it is not a multiband
  effect", while IRC IV is described in band terms. iZotope does not reconcile this. **[V]**
  Most plausible resolution: the bands drive *detection and gain allocation* inside one limiter,
  not a split into separately processed output paths. **[I]** The "without applying crossovers"
  phrasing points the same way, and is consistent with an STFT or complex-filterbank
  implementation rather than a crossover tree. **[I]**
- **Detection/ceiling:** a separate `True Peak` switch — when enabled the limiter accounts for
  "the levels of each digital sample and the levels of the analog signal" after D/A. `Ceiling`
  "set[s] the maximum output level of the Maximizer". **[V]** No oversampling factor published.
- `Transient Emphasis` shapes transients *before* limiting. **[V]**

**Reading:** IRC IV is per-band gain allocation for *artifact reduction*, bolted onto a limiter
whose ceiling is still enforced wideband and in the time domain. This is the prior art closest to
Squared Away's founding idea, and it stops exactly where the arithmetic says it must. **[I]**

### iZotope Neutron 5 Clipper

Multiband clipping across up to 3 bands with crossover auto-learn; 4x oversampling; mid/side and
transient/sustain channel modes. **[M — iZotope feature page, not manual]** No documented
true-peak mode found. Listed here because it is a second shipping multiband clipper.

### Kazrog KClip 3

- **Mechanism:** oversampled clipping, 8 named curves (Smooth, Crisp, Tube, Tape, Germanium,
  Silicon, Broken Speaker, Guitar Amp). Oversampling "Adjustable (2x to 32x)", with separate
  online/offline render settings. **[M — product page]**
- **Multiband mode:** "use up to 4 different clipping modes split between 4 customizable
  bandwidth regions". **[M]** A third shipping multiband clipper.
- **Ceiling:** the manual describes a final `Ceiling` stage that "applies a final stage of 100%
  hard clipping at the session sample rate with **no oversampling**", and warns "do NOT rely on
  the Ceiling to clip substantial dBs, or you will incur audible aliasing". **[V — manual, via
  search snippet; I could not read the PDF directly, see §6]**
- **True peak:** the product page does not mention true peak or inter-sample peaks at all. **[V,
  negative]** A KVR reply states it is a clipper, not a limiter, and suggests adding an ISP
  limiter after it. **[C]** One review asserts a "proper true-peak-aware output stage"; this
  contradicts the manual's own Ceiling description and I would not rely on it. **[C, doubtful]**

**Reading:** KClip 3 is the Worth-Using Gate reference and it does **not** offer a documented
TRUE-peak ceiling. Its non-oversampled final Ceiling is a sample-peak catch, not a true-peak
guarantee. If Squared Away ships a real TRUE-peak guarantee it is doing something KClip does not
claim to do — which matters for how the Worth-Using Gate is framed: the comparison is on
*audible quality at matched Reduction Depth*, not on the guarantee. **[I]**

### SIR Audio Tools StandardCLIP

- **Chain:** oversample → clip → filter above source Nyquist → downsample. **[V]**
- Oversampling up to 256x (v1.1 added 64/128/256); minimum-phase (no added latency) or
  linear-phase (adds latency, preserves phase) filters, with adjustable quality/cutoff, kernel
  size or filter order. Larger kernels bring "more latency, CPU consumption and potential
  pre-ringing artifacts". Offline factor settable separately; online must be ≥2x. **[V]**
- **Ceiling:** post-oversampling, and the manual is explicit about the cost — the ceiling "should
  never be lower than the computed clip level", because it "occurs after the oversampling stage
  and will create aliasing then". **[V]** Same structural compromise as KClip's Ceiling.
- **True peak:** the manual **never uses the terms "true peak" or "inter-sample peak"**. It does
  acknowledge the phenomenon: "Due to the oversampling process, some signal peaks can appear
  above the computed clip level." **[V]** An `Output Gain Assistant` sets output gain by
  observation so no output sample exceeds the ceiling. **[V]**
- **Spectral/per-band:** none. The only frequency-selective element is the oversampling
  band-limiting filter. **[V]**
- User-reported latency 64 samples with oversampling. **[C]**

### Softube / Weiss MM-1

- Based on DS1-MK3 technology. Chain ends in a "brick-wall limiter"; `Limiter Gain` is "the gain
  after the initial processing that drives the brick-wall limiter"; `Amount` at 0% means "no
  processing applied before the brick-wall limiter". Output Trim targets given in **dBFS**
  (-0.1 / -0.3 / -1). Styles: Transparent, Loud, Punch, Wide, Deess; Wide and Deess use M/S.
  **[V]**
- **True peak, oversampling, lookahead, latency, bands: all absent from the manual.** The
  dBFS-denominated trim targets suggest a sample-peak ceiling rather than a true-peak one. **[I]**
- A review's "1:1 code-ported from DS1-MK3" claim is unverified. **[C]**

### Brainworx bx_limiter True Peak

- "Selective Oversampling" (SOS) — oversampling applied "only where it actually improves the
  signal" — plus an "Advanced Look-Ahead" design. Styles: Modern, Classic. `XL` is a saturation
  stage, not a limiting mode. **[M — product/press copy only]**
- **No manual found.** Despite "True Peak" being in the product name, I found no documentation of
  the detection filter, oversampling factor, or ceiling guarantee. **[V, negative]**
- TMT (US Patent 10,725,727) belongs to the console emulations, not to this limiter. **[I, from
  [M] sources]**
- A user reports `XL` at 100% reaching very loud levels but "really crushes the sound". **[C]**

### sonible smart:limit

- Measures "True Peak [dBTP] - using high-resolution oversampling"; factor unpublished. **[M]**
- `Style` dial moves "between either a soft or hard limiting style"; plus saturation, a `Balance`
  control for spectral polish, and `Bass Control`. Balance and Bass are described as tone
  controls, **not** as limiting bands. **[M]**
- Content-aware auto-parametrisation with genre profiles; "Instant Impact Prediction" readouts.
  **[C — review]**
- **No multiband or spectral limiting claim found in any source.** **[V, negative]**

### A.O.M. Invisible Limiter G3 — the algorithmic outlier worth a closer look

Not on the bench list, but it belongs there. A.O.M. describe the limiting algorithm as
**minimising the difference between the original and the limited signals** — an explicit
optimisation framing rather than an envelope-follower framing. G3 brings "significant updates
for the fundamental limiting algorithm and oversampling engine", aimed at reducing "narrow stereo
image, signal smearing, spatial orientation change and groove alternation". **[M — vendor
copy/press]** I did not retrieve the manual, so the detection path and oversampling factor are
unverified. **Recommend adding it to the Reference Set comparison**: it is the one commercial
product whose stated objective function resembles Squared Away's. **[I]**

### Bench summary table

| Product | True-peak detection | Ceiling enforced where | Per-band / spectral machinery | Ceiling is spectral? |
|---|---|---|---|---|
| FabFilter Pro-L 2 | Yes, BS.1770-compliant, selectable OS **[V]** | Time domain, 2-stage **[V]** | None **[V]** | No |
| TDR Limiter 6 GE | Yes, "approximation" **[V]** | Time domain, fixed final stage **[V]** | Multiband clipper (`Separation`), multiband peak limiter, HF limiter **[V]** | No |
| Ozone Maximizer | Yes, switchable **[V]** | Time domain, wideband **[V]** | IRC IV: dozens of Mel bands, no crossovers **[V]** | No |
| Neutron 5 Clipper | Not documented | Time domain **[I]** | 3-band clipping, 4x OS **[M]** | No |
| KClip 3 | **Not claimed** **[V, neg]** | Final non-oversampled hard clip **[V]** | 4-band multiband clipping **[M]** | No |
| StandardCLIP | **Terms never used** **[V, neg]** | Post-OS ceiling, aliasing-prone **[V]** | None **[V]** | No |
| Weiss MM-1 | Not documented; trim in dBFS **[V, neg]** | Time-domain brickwall **[V]** | None documented **[V]** | No |
| bx_limiter True Peak | Named, not documented **[M]** | Lookahead limiting **[M]** | None **[V, neg]** | No |
| smart:limit | Yes, factor unpublished **[M]** | Time domain **[I]** | None **[V, neg]** | No |
| Leapwing LimitOne | **Not mentioned in manual** **[V, neg]** | Time-domain half of hybrid **[I]** | Pufferfish: frequency-domain multiband, boosts *and* cuts **[V]** | No |
| A.O.M. Invisible Limiter G3 | Not verified | Time domain **[I]** | None documented | No |

**Zero of eleven enforce the ceiling spectrally. Four of eleven use band-split or spectral
machinery to decide where reduction goes.**

---

## 4. Academic and patent landscape

- **US 9,210,506 B1 — "FFT bin based signal limiting"** (Audyssey Laboratories; inventors Nawfal
  and Balamurali; filed 2011-09-12, granted 2015-12-08; Google lists Sound United LLC as current
  assignee with an accuracy caveat). Claim 1: window → FFT → magnitude/phase → limit magnitude →
  back to complex → inverse FFT → assemble → drive a transducer, with limiting recited as
  "selecting a limit value and setting all values above the limit value to the limit value".
  Claim 2 adds a Hanning window at 75% overlap, "limiting the magnitude of the FFT bin data using
  soft smoothing", and an overlap-add accumulator emitting N/4 samples per frame. ~512-point FFT
  in the described DSP implementation. Motivation: existing filter-bank and hard-clipping limiters
  have "very limited resolution" and add noise; the goal is raised perceived loudness without
  harmonic distortion. **Domain: loudspeaker playback (music, TV, film, cell phones).**
  Notes an internal inconsistency — the summary says "distortion is reduced because no hard
  limiting occurs" while claim 1 recites hard limiting. **No time-domain peak guarantee is
  claimed, and peak regrowth / overlap-add overshoot is never discussed.** **[V]**

  *This is the single most important citation in this note.* The founding idea is not novel as a
  mechanism — it was patented 15 years ago — and the patent's own framing (loudness and
  transducer protection, not a ceiling) is the tell.

- **US 11,496,099 B2 — "Systems and methods for limiter functions"** (Mimi Hearing Technologies;
  Hohnerlein and Clark; filed 2020-07-28, granted 2022-11-08). STFT chain with overlap-add; the
  limiter "can be applied to the time domain samples", "on a hop-by-hop basis with the need of
  only one hop look-ahead", adding no further delay. Does not state that the frequency domain
  *cannot* do it — it just doesn't try. **[V]**

- **Armstrong (2002)**, *Electronics Letters* 38(5):246-247, DOI 10.1049/el:20020175 — repeated
  clipping and frequency-domain filtering for OFDM PAPR. Establishes **peak regrowth**: filtering
  a clipped signal in the frequency domain reintroduces peaks above the clip level, so the
  process must be iterated, at two FFTs per iteration. The method's acknowledged drawback is
  complexity. **[V — bibliographic details and mechanism verified via multiple citing papers; I
  did not read the original letter, see §6]**

- **Jeannerot, de Koeijer, Martínez-Nuevo, Møller, Dyreby, Prandoni (2022)**, "Increasing loudness
  in audio signals: a perceptually motivated approach to preserve audio quality",
  arXiv:2202.08183 (eess.SP / eess.AS; carries an IEEE copyright notice). **The most relevant
  paper found, and the clearest template for Squared Away's architecture.** It formulates exactly
  this project's problem: formulation (3) minimises a psychoacoustically weighted error subject
  to `‖x‖∞ ≤ λ`. Crucially:
  - **The optimisation variable is the time-domain signal `x`.** **[V]**
  - The perceptual constraint is applied in the frequency domain: a **diagonal** matrix `P`
    "weighs the frequency components" of the error "based on the psychoacoustic masking
    properties" of the original. **[V]**
  - The peak constraint is an `ℓ∞` constraint on the time-domain signal, not a per-bin magnitude
    cap. **[V]**
  - The crest-factor problem is noted as nonconvex; they use a convex surrogate. **[V]**
  - Reported: better loudness-vs-quality trade-off than classical compression and clipping; they
    state subjective listening tests are still needed. **[V]**
  - Observed behaviour: low detectability mainly modifies 100-200 Hz content; at higher
    detectability "reducing the absolute peak value dominates and tends to produce a line
    spectrum". **[V]**

  **This is the architecture the arithmetic forces, done properly: peak constrained in the time
  domain, audibility evaluated per bin against a masking model.** It maps directly onto the
  glossary's **Residual** / **Masking Threshold** pair. **[I]**

- **US 4,208,548 (1980) and US 4,241,266** — masking-controlled clipping. The input is "coupled
  to a plurality of filters each covering a different critical band" whose responses "approximate
  psychoacoustic masking curves"; a feedback loop uses the audibility estimate to set the
  clipper's input level, holding clipper distortion "at a maximum non-audible level". Aimed at
  broadcast receivers with HF rolloff and pre-emphasis. **[V]** **Shaping the Residual beneath a
  Masking Threshold is ~46-year-old prior art.** The novelty available to Squared Away is not the
  idea but the resolution, the model, and the control law.

- **US 6,205,225** — argues the lower sideband produced by peak limiting is more audible than the
  upper, because strong components mask upward more readily than downward. **[V]** Directly
  relevant to shaping clipping Residual: the asymmetry means symmetric treatment of intermodulation
  products is wrong. **[I]**

- **US 4,495,643** — Hilbert-transform peak limiter behaving as an RF clipper below ~4 kHz and an
  audio clipper above, so LF material produces no harmonic distortion. **[V]** A frequency-split
  clipper, and a reminder that frequency-dependent clipping strategies are long-explored.

- **McCormack and Välimäki (2017)**, "FFT-Based Dynamic Range Compression", *Proc. 14th Sound and
  Music Computing Conference*, Espoo; Zenodo DOI 10.5281/zenodo.1401877. Generalised
  time-frequency dynamic range control with an FFT implementation; examples are
  frequency-dependent ducking for automatic mixing and high-resolution multiband compression.
  **[V — abstract/metadata]** A claimed 133 frequency bands comes from an AI-generated summary on
  Academia.edu and is **not verified**. **[C]** **Note the scope: compression and ducking, not
  peak ceilings.** The academic FFT-dynamics line of work does not attempt peak control either.

---

## 5. What this means for Squared Away

1. **Drop per-bin clipping as the ceiling mechanism.** Not because it was tried and failed
   commercially, but because the bound from bin magnitudes to sample peaks is loose by a factor
   that makes a guarantee unusable. The glossary already encodes this; the research confirms it
   against a patent that implements the mechanism and conspicuously declines to claim a peak
   guarantee. **[I, resting on [V]]**

2. **Keep per-bin work, repositioned.** Two legitimate roles, both with prior art and both with
   room left:
   - **Pre-conditioner** — redistribute energy across bins so the time-domain stage has less to
     do. This is what LimitOne's Pufferfish does, and it boosts as well as cuts. **[V]**
   - **Audibility criterion** — compute the per-bin Masking Threshold and shape the Residual to
     sit beneath it. This is arXiv:2202.08183's `P` matrix, and US 4,208,548's critical-band
     feedback loop. **[V]**

3. **Adopt the two-stage ceiling.** Pro-L 2's documented pattern — reduce pre-existing true
   peaks, then clean up the overshoot your own fast reduction created — is the right shape, and
   the warning generalises: *any* peak-reduction mechanism generates new true peaks. With latency
   unlimited, Squared Away can afford a more generous version than Pro-L 2's ~5 ms. **[I]**

4. **Exploit unlimited latency and Phase Rotation, which the market has not.** No product in the
   bench documents Phase Rotation for peak control, and every one of them is latency-constrained
   in a way Squared Away is not. Phase Rotation is information-preserving by the glossary's own
   definition, and peak level is a phase-alignment phenomenon — so the lever the arithmetic
   identifies as the *cause* of peaks is the one nobody is pulling. This looks like the real
   opening, more than per-bin magnitude work does. **[I]**

5. **Reframe the Worth-Using Gate.** KClip3 does not document a TRUE-peak ceiling at all; its
   final Ceiling is a non-oversampled hard clip the manual warns against leaning on. So the gate
   is correctly about *audible quality at matched Reduction Depth*, not about the guarantee —
   where Squared Away would be ahead by default. Worth stating explicitly so the gate is not
   accidentally won on a technicality. **[I]**

6. **Name the detector the guarantee is against.** Since true peak is inherently an oversampled
   estimate with frequency-dependent under-read, "hard TRUE-peak ceiling" needs a stated detector
   and oversampling factor to be a testable claim. TDR calls its own true-peak ceiling an
   "approximation"; that is the honest register. **[I]**

7. **Measurement targets for the bench.** Highest-value unknowns, all resolvable by measurement
   rather than reading: (a) LimitOne's actual true-peak overshoot — undocumented, with only a
   testimonial behind it; (b) TDR's reported 0.1 dB overshoot with true peak enabled **[C]**;
   (c) KClip 3's true-peak behaviour, since it claims none; (d) A.O.M. Invisible Limiter G3,
   whose stated objective function most resembles this project's.

---

## 6. What I could not verify

Listed so the gaps are not mistaken for findings.

- **ITU-R BS.1770-4 itself.** The canonical ITU PDF URL returned **HTTP 404**. The pre-attenuation
  (-12.04 dB), ≥4x oversampling, filter chain, and "higher over-sampling ratios are preferred"
  preference are reported via Essentia's implementation documentation and FabFilter's forum, both
  of which cite the standard. The specific under-read figure of 0.688 dB for a tone near half the
  sampling rate comes from a secondary article and is **unverified against the standard's tables**.
  **Someone should read Annex 2 of the actual recommendation before any guarantee is published.**
- **KClip 3 manual.** The PDF's text layer could not be extracted (Dr.Explain-generated, subset
  fonts). The Ceiling and oversampling quotes come from search snippets of that manual — primary
  in origin but not read in context by me. Kazrog's product page does not mention true peak at
  all, which is a reliable negative.
- **bx_limiter True Peak documentation.** No manual located, by search or on the product page.
  Everything about it here is marketing copy.
- **sonible smart:limit official manual.** Not retrieved; all claims are retailer listings or a
  review.
- **Weiss DS1-MK3 manual.** Not consulted. It may document the limiter algorithm the MM-1 inherits.
- **Armstrong (2002) original letter.** Not retrieved. Bibliographic details and the peak-regrowth
  mechanism are consistent across several citing papers, so I rate the mechanism verified and the
  exact iteration counts and figures unread.
- **McCormack and Välimäki (2017) full text.** Only abstract and metadata. The 133-band figure is
  an unverified AI-generated summary.
- **AES E-Library.** Not searched directly (paywalled, no access from here). AES Convention Paper
  9041 (Ian Dash, on true-peak meter error) surfaced only as a second-hand quotation. A targeted
  AES search for "perceptual limiter", "masking-based clipping", "psychoacoustic peak limiting" is
  the most likely place to find anything this note has missed.
- **Ozone's IRC IV implementation.** Whether the "dozens of psychoacoustically spaced bands,
  without crossovers" is an STFT or a complex filterbank is my inference, not documented.
- **The KVR discussion of FFT-based dynamics** (thread 531986) is explicitly speculative — the
  participants label their own answers as guesses, and the thread contains no confirmed developer
  statement. I have cited it only as evidence that the question is asked and not well answered in
  public, not as support for any mechanism claim.

---

## Sources

Primary — patents and standards:
- US 9,210,506 B1, FFT bin based signal limiting — https://patents.google.com/patent/US9210506B1/en
- US 11,496,099 B2, Systems and methods for limiter functions — https://patents.google.com/patent/US11496099B2/en
- US 4,208,548, Apparatus and method for peak-limiting audio frequency signals — https://patents.google.com/patent/US4208548A/en
- US 4,241,266, Peak-limiting apparatus for audio signal
- US 6,205,225, Lower sideband modulation distortion cancellation
- US 4,495,643, Audio peak limiter using Hilbert transforms
- ITU-R BS.1770-4 (**not retrievable — 404**); true-peak chain via https://essentia.upf.edu/reference/std_TruePeakDetector.html

Primary — papers:
- arXiv:2202.08183, Jeannerot et al., Increasing loudness in audio signals — https://arxiv.org/abs/2202.08183
- Armstrong, Electronics Letters 38(5):246-247, 2002, DOI 10.1049/el:20020175
- McCormack and Välimäki, FFT-Based Dynamic Range Compression, SMC 2017 — https://zenodo.org/record/1401877

Primary — vendor documentation:
- FabFilter Pro-L 2 help: true peak limiting — https://www.fabfilter.com/help/pro-l/using/truepeaklimiting
- FabFilter Pro-L 2 help: oversampling, output options — https://www.fabfilter.com/help/pro-l/using/oversampling
- TDR Limiter 6 GE manual — https://docs.tokyodawn.net/limiter-6-ge-manual/
- iZotope Ozone 9 Maximizer documentation — https://downloads.izotope.com/docs/ozone9/en/maximizer/index.html
- iZotope, What Is Spectral Shaping? — https://www.izotope.com/en/learn/what-is-spectral-shaping.html
- SIR Audio Tools StandardCLIP manual — https://siraudiotools.com/StandardCLIP_manual.php
- Leapwing LimitOne manual — https://s3-eu-west-1.amazonaws.com/leapwing-files/manuals/Manual-LimitOne.pdf
- Softube Weiss MM-1 manual — https://www.softube.com/manuals/weiss-mm-1-mastering-maximizer
- Kazrog KClip 3 manual (text layer unextractable) — https://m.barryrudolph.com/recall/manuals/kazrog_kclip_3.pdf

Marketing / product pages:
- Leapwing LimitOne — https://www.leapwingaudio.com/product/limitone/
- Kazrog KClip 3 — https://kazrog.com/products/kclip-3
- Plugin Alliance bx_limiter True Peak — https://www.plugin-alliance.com/en/products/bx_limiter_true_peak.html
- iZotope Neutron Clipper — https://www.izotope.com/en/products/neutron/features/clipper
- iZotope Ozone 11 features — https://www.izotope.com/en/products/ozone/features

Community:
- KVR, How do "spectral" (FFT-based) dynamics processors work? — https://www.kvraudio.com/forum/viewtopic.php?t=531986
- KVR, Kazrog KClip 3 - True Peak Limiter? — https://www.kvraudio.com/forum/viewtopic.php?p=8848840
- Gearspace, Leapwing presents LimitOne — https://gearspace.com/board/new-product-alert-2-older-threads/1435458-leapwing-presents-limitone-spectral-approach-limiting-clipping-3.html
