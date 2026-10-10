# Research: which psychoacoustic masking models are tractable in real time?

Resolves issue #3. Scope: the model that computes the per-bin **Masking Threshold** beneath which the **Residual** is shaped. Vocabulary per `GLOSSARY.md`.

Evidence is tagged throughout:

- **[A] verified from a primary source** — a spec, a paper, or a file I fetched and read (source code, `LICENSE`/`COPYING`).
- **[B] secondary claim** — review article, tutorial, mailing list, patent prose.
- **[C] my own inference** — reasoning from [A]/[B], not stated anywhere I found.

---

## 1. Headline recommendation

**Build on the Vorbis psychoacoustic model (`libvorbis/lib/psy.c`), BSD-3-Clause.** It is the only full-featured, production-hardened, per-bin masking model I found whose licence is MIT-compatible, and its cost is strictly O(N) per frame with no iterative solve. Everything else is either licence-hostile (LGPL/GPL/Fraunhofer), not a per-bin masking model at all (Opus), or an order of magnitude more expensive (Moore/Glasberg).

Decision-relevant caveat, which is the honest part of this ticket: **no model in this list predicts the audibility of a clipping Residual.** They all predict the audibility of *quantisation noise* in a *codec*, which is a different signal with a different statistical structure. Section 6 sets out where that gap bites, and why **Transparent** has to be settled by the Pass Gate rather than by the model's own number.

---

## 2. Comparison table

| Model | Inputs | Outputs | Cost per block | Open implementation | Licence |
|---|---|---|---|---|---|
| MPEG-1 psy model 1 | FFT power spectrum (512 pt L1 / 1024 pt L2) **[B]**; sample rate | SMR per coder sub-band | O(N log N) FFT + tonal/non-tonal peak picking + per-component spreading; O(K·M) threshold summation | ISO 11172-5 reference code; LAME (model 1 largely vestigial) | ISO ref code: non-free. LAME: **LGPL v2** **[A]** |
| MPEG-1 psy model 2 | 1024-pt FFT magnitude + phase, plus 2 previous frames for prediction | SMR per partition → per sub-band | O(N log N) + spreading matrix; see §4 | LAME `psymodel.c` (heavily modified descendant) | **LGPL v2** **[A]** |
| 3GPP TS 26.403 / AAC-style PE model | MDCT or FFT band energies, per scalefactor band | Per-band threshold + Perceptual Entropy, drives block switch | O(N log N) + O(B) one-pass spreading (B ≈ 49 bands) | FFmpeg `libavcodec/aacpsy.c` | **LGPL v2.1+** **[A]** |
| Vorbis psy model | Log FFT magnitude + log MDCT magnitude, per bin | Per-bin log Masking Threshold (tone + noise, mixed) | O(N) after the FFT, see §4 | `xiph/vorbis` `lib/psy.c` (1213 lines) | **BSD-3-Clause** **[A]** |
| Opus / CELT | Per-band log energies; time-domain envelope | *Not a Masking Threshold.* SMR-derived spread weights; transient flag | Trivial, O(B) | `xiph/opus` `celt/celt_encoder.c` | **BSD-3-Clause** **[A]** |
| Moore/Glasberg ERB (1997; ISO 532-2) | Spectrum → ERB-spaced excitation pattern; level-dependent filter shapes | Excitation, specific loudness, partial loudness; thresholds derivable | ~100+ ERB filters, level-dependent shapes; see §4 | Cambridge binaries (no source); AMT `glasberg2002`/`moore2016`; torch-amt; Loudness Toolbox; SQAT | AMT: **GPL v3** **[A]**. torch-amt: **GPL-3.0-or-later** **[A]**. SQAT: **GPL v3** **[A]**. Loudness Toolbox: **unconfirmed** |
| ISO 226 equal-loudness contours | Frequency, phon level | Absolute threshold / ELLC curve. *Not a masking model* | Table lookup, negligible | Curves republished widely; ATH table already baked into Vorbis `psy.c` **[A]** | ISO 226:2023 is **paywalled**; the data is not a software licence question |
| Learned / neural masking | Waveform or spectrogram | Learned threshold or perceptual loss weighting | GPU-scale inference per frame | Research code only; nothing I could confirm as a released, licensed threshold model **[A, negative finding]** | n/a |

---

## 3. What each one actually gives you

### MPEG-1 psychoacoustic models 1 and 2

**[B]** Model 1 identifies tonal components as local spectral maxima meeting a prominence criterion, lumps the remainder of each critical band into a single non-tonal masker at the band's energy centroid, decimates maskers below the absolute threshold, applies a single spreading function per masker, and sums. Model 2 drops the tonal/non-tonal split in favour of a continuous *tonality* measure derived from the **unpredictability** of each bin's magnitude and phase against a linear prediction from the two preceding frames; tonality then interpolates the signal-to-noise requirement between a tonal and a noise-masker value. Model 2 is the one that was actually used in practice (Layer III, and the ancestor of every encoder psy model since).

**[A]** The most useful consequence for us: model 2's tonality measure needs **two frames of history**, so it has state and it reacts to transients with a one-to-two-frame lag by construction. **[C]** On an attack, the predictor fails, unpredictability spikes, tonality drops toward "noise-like", and the threshold is raised — which is the *wrong* direction if what we are trying to hide is a Residual burst concentrated exactly at that attack. This is a structural argument against model 2 for our use case, independent of its cost.

**[B]** A 2006 `auditory-list` thread has a practitioner saying flatly of MPEG psy model 2, "I think it is too complex", and that it "has some really unnecessary lookaheads". Anecdote, not measurement, but it matches the structure.

**Licence [A]:** `lameproject/lame` `COPYING` is the **GNU Library General Public License, Version 2, June 1991** — verbatim from the file. That is a real problem for an MIT VST3: LGPL v2's relinking requirement is widely regarded as unsatisfiable for a statically-linked, closed-or-permissive plugin binary distributed through commercial channels, and even dynamic linking obliges you to ship the relink toolchain. **[C]** Treat LAME's `psymodel.c` as *read-only reference* — do not copy code, do not link.

### 3GPP TS 26.403 / the AAC lineage — what the modern codec world actually uses

**[A]** FFmpeg's AAC encoder psy model (`libavcodec/aacpsy.c`, Konstantin Shishkov, 2008) is an implementation of the 3GPP TS 26.403 model, and the file's constants are explicit about the model's shape:

```c
#define PSY_3GPP_THR_SPREAD_HI   1.5f // spreading factor for low-to-hi threshold spreading  (15 dB/Bark)
#define PSY_3GPP_THR_SPREAD_LOW  3.0f // spreading factor for hi-to-low threshold spreading  (30 dB/Bark)
```

So: **one-pass, two-sided, fixed-slope spreading on a Bark scale — 15 dB/Bark upward, 30 dB/Bark downward.** No matrix multiply, no iteration. **[C]** That is the single most important structural fact in this ticket: the industry converged on a *recursive two-pass spread with fixed slopes*, i.e. O(B) in the number of bands, precisely because the textbook O(B²) spreading-matrix formulation was not worth its cost.

**[A]** TS 26.403 ("Enhanced aacPlus general audio codec; Encoder Specification AAC part") and its ANSI-C reference code in TS 26.410 are freely downloadable from `3gpp.org/ftp/Specs/archive/26_series/` — I confirmed 17 versions of 26.403 are listed with no login or paywall. **[B]** The 26.410 cover material carries a disclaimer that the document "has not been subject to any approval process by the 3GPP Organizational Partners and shall not be implemented", and 3GPP's own terms are not an OSI licence. **[C]** The *spec text* is therefore usable as a specification to implement from — which is the licence-clean route — while FFmpeg's code (**LGPL v2.1+ [A]**, from `LICENSE.md` and the `aacpsy.c` header) is again reference-only.

**[A]** Note also that the published 26.403 model grew a "spreading function" lookup table big enough that a 2004 DAFx paper on AAC encoder optimisation reports the table "requires 6614 words (at sampling rate 44100 Hz) without optimization" **[B]**. Memory, not cycles, is the binding constraint in that implementation.

### Vorbis — the licence-clean full model

**[A]** `xiph/vorbis` `lib/psy.c`. The header comment is itself a finding:

> `function: psychoacoustics not including preecho`

The model has two independent halves, which is exactly the decomposition we want:

- **`_vp_tonemask()`** — seeds a tone-masking curve from the log FFT. Primes a working vector with the per-octave-line peak, then for each peak calls `seed_curve()`, which selects a precomputed masking curve by **level** (`choice=(int)((amp+dBoffset-P_LEVEL_0)*.1f)`, i.e. a 10 dB-quantised level index into `P_LEVELS` curves), and max-combines it into the seed buffer. The absolute threshold (`p->ath`) is laid in first and floats below the local spectral max rather than the global one. **Level-dependent spreading by table lookup, max-combined — no summation, no solve.**
- **`_vp_noisemask()`** — computes a noise-floor estimate via `bark_noise_hybridmp()`, which I read in full: it builds five running prefix sums (`N, X, XX, Y, XY`) in one forward pass, then for each bin does a constant-time windowed weighted linear regression by differencing the prefix sums at the bin's precomputed Bark window edges (`p->bark[i]` packs `lo` and `hi` into one long). Called twice, plus two O(N) difference passes.
- **`_vp_offset_and_mix()`** — max-combines tone and noise thresholds with per-band offsets, giving the final per-bin log Masking Threshold.

**[A] Cost: after the FFT, the whole model is a small constant number of O(N) passes with table lookups. No iteration, no per-block matrix, no solve.** The only non-linear-time thing in the file is `seed_chase()`, which is a single-pass stack-based convex chase — still O(N) amortised.

**[A] Licence:** `xiph/vorbis` `COPYING` is three-clause BSD ("Redistribution and use in source and binary forms... Neither the name of the Xiph.org Foundation nor the names of its contributors may be used to endorse or promote products derived from this software without specific prior written permission."). `psy.c`'s own header confirms "GOVERNED BY A BSD-STYLE SOURCE LICENSE INCLUDED WITH THIS SOURCE IN 'COPYING'". **MIT-compatible: you may copy the code into an MIT project, retaining the Xiph notice.**

**[C]** Two caveats. First, the model is *tuned* against Vorbis's MDCT sizes, its bitrate ladder, and the AoTuV patches (visible in `_vp_offset_and_mix`, commented `/* AoTuV */`); its offsets are not neutral psychoacoustics and will need retuning for our block size and for a Residual rather than quantisation noise. Second, `P_LEVELS` level-quantisation at 10 dB granularity is coarse — fine for choosing a quantiser, possibly audible as threshold stepping if we drive gain continuously from it.

### Opus / CELT — not a masking model, but the best temporal-masking numbers in any open source

**[A]** I grepped `celt/celt_encoder.c` (3163 lines). There is **no masking-threshold model**. What there is:

1. A comment-documented temporal masking pair in the transient detector:
   - `/* Forward masking: 6.7 dB/ms. */` → `forward_decay = QCONST16(.0625f,15)` (and 3.3 dB/ms via `.03125f` for low bitrates)
   - `/* Backward masking: 13.9 dB/ms. */` → `mem0 = tmp[i] + 0.875f*mem0; tmp[i] = 0.125f*mem0;`
   These are two one-pole smoothers run forward (post-echo threshold) and backward (pre-echo threshold) over a half-rate energy envelope, and the transient decision is `is_transient = mask_metric>200`, described in the code as "a bitrate-normalized temporal noise-to-mask ratio".
2. A deliberately crude frequency spread, labelled in the source: `/* Compute a really simple masking model to avoid taking into account completely masked bands when computing the spreading decision. */` — `mask[i] = bandLogE[i]-noise_floor[i]`, then `mask[i] = MAXG(mask[i], mask[i-1] - GCONST(2.f))` upward and `MAXG(mask[i], mask[i+1] - GCONST(3.f))` downward, i.e. **2 dB and 3 dB per band**, with the comment "Mask is never more than 72 dB below the peak and never below the noise floor."

**[C]** Opus is the clearest evidence that a state-of-the-art codec decided a full per-bin masking threshold was not worth computing. That is an argument about *codecs*, not about us — we are not spending bits, we are deciding how much Residual we can hide, so we need the threshold itself. But the two temporal-masking slopes are directly reusable, come under BSD-3-Clause **[A]**, and are the thing Vorbis's `psy.c` explicitly does *not* provide.

### Moore/Glasberg ERB models

**[B]** Moore, Glasberg & Baer (1997), *A Model for the Prediction of Thresholds, Loudness, and Partial Loudness*, JAES 45(4):224–240; standardised as the Moore–Glasberg method in **ISO 532-2**. Chain: spectrum → excitation pattern via level-dependent ERB-scaled auditory filters → specific loudness via a compressive transform of the form `Ls = k[(GE+A)^α − A^α]` → integrate over ERB number for loudness; *partial loudness* in the presence of a masker is what would give us a threshold.

**[C]** Cost is the problem. The excitation pattern is computed over ~100+ ERB-spaced channels whose filter *shapes depend on level*, so the weights are not a fixed precomputable matrix — which is exactly the structure that turns into either a per-block iterative/level-dependent recomputation or a large interpolated table. **[B]** A patent on loudness processing states the complexity "can be decreased using an approximation" without quantifying it, and a published MATLAB package exists specifically titled around "techniques for the rapid calculation of the excitation pattern", which is itself evidence that the direct computation is too slow to be casual about.

**[C] Concretely, on cost:** at 192 kHz the per-block budget shrinks by 4.35× relative to 44.1 kHz for the same block size in samples (or the block count rises 4.35× for the same block length in ms). An O(N log N) FFT plus O(N) threshold passes (Vorbis, 3GPP) scales benignly. A model whose per-block work is ~100 level-dependent filter evaluations each touching a wide spectral support is **O(E·N)** with E ≈ 100 — two orders of magnitude more arithmetic than the Vorbis path, per block, per channel. For a stereo plugin at 192 kHz, with Stereo Link requiring at least a combined threshold across channels, **[C] I would not plan on a direct Moore/Glasberg excitation pattern per block.** If any per-block *iterative* solve is involved (partial loudness inversion to recover a threshold, rather than a forward prediction), that should be treated as disqualifying for the 192 kHz case until measured.

**Licence [A]:** every open implementation I could check is copyleft.
- Auditory Modeling Toolbox: "The AMT is free software released under the GNU GPL version 3 (some models differently licensed)" — quoted from `amtoolbox.org`. It is the home of `glasberg2002` and `moore2016`.
- `torch-amt` (PyTorch port of AMT, which is where the `moore2016` and `glasberg2002` code actually lives in a modern form): PyPI metadata reports `license: GPL-3.0-or-later`, classifier `License :: OSI Approved :: GNU General Public License v3 or later (GPLv3+)`.
- SQAT (Sound Quality Analysis Toolbox): `LICENSE` is the **GNU General Public License Version 3**. Covers ISO 532-1 (Zwicker), not confirmed for 532-2.
- Cambridge (Moore's group) distribute **Windows/DOS executables, not source** **[B]** — no licence to inherit, and no code to use.
- **Loudness Toolbox (Genesis Acoustics): I could not confirm the licence.** A 2010 announcement says it is "available for free on its website" and lists "Moore et al model for time-varying sounds" **[B]**; "free of charge" is not a licence grant. **This is an open item.**

**[C]** Net: Moore/Glasberg is the most perceptually defensible model on this list and the least available to an MIT project. The route, if we want it, is implement-from-paper, which is legal but is weeks of work and needs its own validation against published excitation patterns.

### ISO 226

**[A]/[C]** ISO 226 is equal-loudness-level contours — an **absolute** threshold/loudness mapping for pure tones in a free field, binaural, listeners aged 18–25. It is **not** a masking model and cannot produce a Masking Threshold. Its role is the floor term: the masking threshold must never fall below absolute audibility. Vorbis already carries an `ATH` table and lays it in before tone masking (`logmask[i]=p->ath[i]+att`) **[A]**, with the notable refinement that it floats the ATH relative to the *local* spectral max, clamped by `ath_maxatt`.

**[B]** ISO 226:2023 differs from :2003 only slightly: the foreword cites alignment with ISO 389-7 on the 0-phon data and "correction of systematic errors that lead to minor changes in the entire data up to 0,6 dB". **[C]** 0.6 dB is below anything that will decide an ABX, so the edition question is irrelevant to us; and since ISO 226 is paywalled, using Vorbis's BSD-licensed ATH table is both cheaper and licence-clean.

### Learned / neural masking models

**[A] Negative finding.** I searched for a released, licensed, learned masking-threshold model and did not find one. What exists: differentiable *codec emulators* used as perceptual losses (MIT Media Lab / Spotify, 2020 **[B]**); DNNs that set per-band quantiser step sizes inside a transform coder **[B]**; MP3net, which computes MP3-style Bark-band thresholds inside a GAN's loss **[B]**; and a patent on using a masking threshold to reweight a network's error function **[B]**. A 2024 survey, *Neural Speech and Audio Coding* (arXiv 2408.06954) **[B]**, treats psychoacoustic models as perceptual *loss functions*, not as runtime threshold predictors.

**[C]** Verdict for this project: not relevant yet. Even setting licensing aside, a per-block neural threshold predictor inside a VST3 adds an inference dependency, is hard to make sample-rate-agnostic across 44.1–192 kHz, and gains us nothing we can validate. Latency is unlimited here so *latency* is not the objection — **cost per block and non-determinism are.** Revisit only if someone releases a permissively-licensed threshold predictor with published validation against masking data.

---

## 4. Cost, concretely

**[C]** throughout this section unless marked, reasoning from the source I read.

Let N = bins per block, B = number of Bark/ERB bands (~25 Bark partitions, ~49 AAC scalefactor bands, ~100+ ERB channels), E = excitation channels.

| Model | Per-block complexity after the FFT | Iterative solve? | 192 kHz verdict |
|---|---|---|---|
| Vorbis psy | **O(N)** — ~6 linear passes with table lookups **[A, read the source]** | No | Fine. Cheapest real model available. |
| 3GPP / AAC PE | **O(B)** two-pass recursive spread + O(N) band energies | No (the *encoder's* two-loop rate search is iterative, but that is quantiser bit allocation, not the psy model — we do not need it) | Fine |
| MPEG-1 model 2 | **O(N)** prediction + **O(K·B)** spreading, K maskers | No, but 2-frame state and lookahead | Workable; structurally wrong for transients (§3) |
| MPEG-1 model 1 | O(N) peak picking + O(K·B) summation, K maskers | No | Workable but obsolete |
| Moore/Glasberg | **O(E·N)** with level-dependent filter shapes, E ≈ 100+ | Possibly, for partial-loudness inversion — unconfirmed | **Do not plan on it.** Measure before committing. |
| Opus temporal masking | O(N) two one-pole passes **[A, read the source]** | No | Trivially fine |
| Neural | GPU-scale inference | n/a | Not viable as specified |

The useful framing: **the FFT dominates.** Vorbis-class and 3GPP-class models are a small multiple of the FFT you are already paying for in an STFT-based device. Moore/Glasberg is not — it is the FFT plus a second, larger computation. That is the dividing line, and it is the answer to the ticket's "say so with numbers or complexity classes".

---

## 5. Simultaneous vs temporal masking

**[A]** The models above are, with one exception, **simultaneous (frequency) masking only**:
- Vorbis `psy.c` says so in its own header: "psychoacoustics not including preecho".
- Opus's CELT is the exception and handles temporal masking *separately* from frequency, with the two slopes quoted in §3: **forward (post-) masking 6.7 dB/ms, backward (pre-) masking 13.9 dB/ms**, implemented as one-pole forward and backward envelope smoothers.
- The AAC/3GPP model handles time indirectly, via Perceptual Entropy driving a long/short **block switch**, not via a temporal term in the threshold.

**[B]** Published durations: post-masking persists roughly 100–200 ms; pre-masking covers only about 20 ms before the masker. Forward masking is much stronger than backward masking, which is why **pre-echo is audible and post-echo generally is not**.

**[C] This asymmetry is the central design constraint for us.** Our mechanism is clipping, whose Residual is an impulsive burst localised at the waveform peak — i.e. at an attack. If we shape that Residual in the frequency domain with an STFT whose window is longer than ~20 ms, the shaped Residual is smeared across the whole window, including the ~20 ms *before* the transient where backward masking is weak. That is pre-echo, and it is listed in the Glossary as the characteristic **Artifact** of spectral processing. A simultaneous-only masking model will score that configuration as inaudible and it will fail the Pass Gate. The practical consequence: **window length is not a free parameter to be chosen for frequency resolution; it is bounded above by the pre-masking window.** Latency being unlimited does not rescue this — unlimited latency buys lookahead, not temporal resolution.

**[C]** So the architecture this research points to is a *pair*: a Vorbis-derived per-bin simultaneous threshold, times a CELT-derived temporal envelope gate using the 6.7 / 13.9 dB/ms slopes, both BSD-3-Clause. Neither alone is sufficient.

---

## 6. Where these models are known to mispredict on real music

All **[B]** unless marked. These are the things that will decide the Pass Gate.

1. **Stationarity.** Masking formulations assume stationary maskers and additive masking. Music is non-stationary and dense; additive masking underestimates perceptual interaction in dense mixtures. **[C]** Our worst case — a loud transient-rich master — is the case the models are weakest on.
2. **Transients specifically.** Pre-masking is short (~20 ms) and models predict it badly; a frequency-domain model cannot localise noise in time at all. **[C]** And MPEG model 2's tonality measure actively *raises* the threshold at an attack (§3), which is the wrong sign.
3. **Binaural masking level difference (BMLD).** Every model here is monaural. BMLD can lower the real masking threshold by **up to 15 dB** relative to monaural masker/probe data; the effect is strongest on headphones and most severe for near-monophonic content. **[C] This is the single largest quantified misprediction on the list, and it lands squarely on us:** we are stereo, with **Stereo Link** on by default, so our Residual will be highly correlated between channels while the programme material is not. A correlated Residual under a decorrelated masker is precisely the BMLD-vulnerable configuration. A model-derived threshold could be 15 dB optimistic for a headphone listener. Codecs mitigate this with M/S so the coding noise sits "behind" the masker, and one patent describes bolting an SNR floor onto the psy model at the cost of capacity at low frequencies. **[C] Expect to need a BMLD safety margin, and expect it to cost Reduction Depth.** Flag for the ABX protocol: test on headphones, and include near-mono material in the **Reference Set**.
4. **Inter-listener variability.** Critical-band definitions and spreading functions vary between listeners; a mastering engineer running the ABX is likely at the sensitive tail.
5. **The authors' own position.** Herre & Dick's tutorial review (*Applied Sciences* 9(14):2854, 2019, DOI 10.3390/app9142854) states that commonly used models "only represent a small subset of the known properties and abilities of the human auditory system", while arguing that such models "do not need to be perfect to satisfy their purpose". **[C]** Their purpose is bit allocation under a rate constraint, where being 3 dB wrong costs some bits. Our purpose is a transparency claim, where being 3 dB wrong costs the Pass Gate. **The models' own authors do not claim the accuracy we are asking of them.**
6. **Codec-noise vs Residual mismatch.** **[C]** Every tuning constant in `psy.c` and `aacpsy.c` was fitted so that *quantisation noise* at that threshold was inaudible. Quantisation noise is broadband, roughly stationary within a block, and uncorrelated with the signal. A clipping Residual is none of those — it is impulsive, harmonically related to the signal, and phase-locked to it. Harmonic distortion correlated with the masker is a different detection task from uncorrelated noise, and I found **no** source establishing that a codec-tuned threshold transfers to it. **This is the biggest unvalidated assumption in the whole approach, and it is not resolvable from the literature.**

**[C] Direct answer to the ticket's acceptance-gate question:** the model most likely to "score well on paper and fail on programme material" is **any of them used as an absolute guarantee**. The failure modes are known and signed: BMLD makes the threshold optimistic by up to 15 dB on headphones; pre-echo makes it optimistic on transients; Residual-vs-quantisation-noise mismatch makes it optimistic for correlated distortion. **All three errors point the same way — the model will say "inaudible" when it is audible.** The design should therefore treat the Masking Threshold as a *shaping target with a margin*, calibrated by ABX, never as a proof of transparency. That is consistent with the Glossary's definition of **Transparent** as a claim that must be tested.

---

## 7. Licence summary (each verified by reading the file)

| Project | File read | Licence, exactly |
|---|---|---|
| `xiph/vorbis` | `COPYING`, `lib/psy.c` header | BSD 3-clause ("Copyright (c) 2002-2020 Xiph.org Foundation") — **usable** |
| `xiph/opus` | `COPYING` | BSD 3-clause ("Copyright 2001-2023 Xiph.Org, Skype Limited, Octasic, …") — **usable** |
| `lameproject/lame` | `COPYING` | **GNU Library General Public License, Version 2, June 1991** (LGPL v2) — reference only |
| `knik0/faac` (libfaac) | `COPYING` | **GNU Lesser General Public License Version 2.1, February 1999** — reference only |
| `mstorsjo/fdk-aac` (libfdk-aac) | `NOTICE` | **"Software License for The Fraunhofer FDK AAC Codec Library for Android"** — bespoke, non-OSI; the same file states patent licences "may be obtained through Via Licensing … or through the respective patent owners individually". **Avoid: licence *and* patents.** |
| `FFmpeg/FFmpeg` | `LICENSE.md`, `libavcodec/aacpsy.c` header | **LGPL v2.1+** (GPL v2+ for optional parts not used by default); `aacpsy.c` header confirms LGPL v2.1+ — reference only |
| `Eomys/MoSQITo` | `LICENSE` | **Apache License 2.0** — **usable**, but it is ISO 532-1 loudness/sharpness, not a masking model |
| `ggrecow/SQAT` | `LICENSE` | **GNU GPL Version 3** — unusable |
| Auditory Modeling Toolbox | `amtoolbox.org` | **GNU GPL version 3** ("some models differently licensed") — unusable |
| `torch-amt` | PyPI metadata | **GPL-3.0-or-later** — unusable |
| 3GPP TS 26.403 / 26.410 | 3GPP FTP listing | Freely downloadable spec + ANSI-C; **not an OSI licence**, and 26.410 carries a "shall not be implemented" disclaimer. Use the **spec text** as a specification to implement from. |
| ISO 226:2023, ISO 532-2 | ANSI webstore preview | **Paywalled standards.** |
| Loudness Toolbox (Genesis) | — | **Could not confirm.** Open item. |

**[C]** Pattern worth internalising: **the two Xiph codecs are the only permissively-licensed psychoacoustics in the open-source audio world.** Every other open implementation of a masking model is LGPL, GPL, or Fraunhofer-bespoke. If the project is MIT, the practical choice set is Xiph code plus anything implemented from a spec or paper.

---

## 8. Recommendation and next steps

**Recommend:** Vorbis `psy.c` (BSD-3-Clause) as the simultaneous-masking core, retuned for our block size and Residual statistics, **plus** CELT's temporal masking slopes (BSD-3-Clause) as a separate time-domain gate. Implement the 3GPP/AAC two-sided fixed-slope Bark spread (15/30 dB/Bark) from the freely-available spec as an alternative threshold path to A/B against Vorbis — it is cheap, well-documented, and licence-clean if written from the spec rather than copied from FFmpeg.

**Reject for now:** Moore/Glasberg (cost at 192 kHz, and every implementation is GPL — would have to be written from the paper); MPEG-1 model 2 (LGPL, and its tonality measure mispredicts the transient case we care most about); neural models (nothing released and licensed); ISO 226 as a masking model (it is not one — use Vorbis's ATH table for the absolute floor).

**Open items I could not confirm:**
1. The Loudness Toolbox (Genesis Acoustics) licence. "Free" was announced; terms unverified.
2. Whether Moore/Glasberg partial-loudness inversion requires a per-block *iterative* solve. If it does, that is decisive for 192 kHz. Unresolved from the literature; would need the 1997 paper's equations.
3. Herre & Dick's full text. The MDPI and FAU copies both returned 403 to automated fetching; I have only quoted the abstract-level statements that search surfaced. Worth a human download — it is the best single overview and it is open access.
4. No published figure for MPEG-1 model 2's operation count exists that I could find. Would have to be tallied from the ISO/IEC 11172-3 Annex D pseudocode.
5. **Nothing in the literature validates a codec-tuned masking threshold against a correlated, impulsive clipping Residual.** This is not a gap in my search; I believe it is a gap in the field. It should become an early empirical experiment rather than a reading task.

---

## Sources

Primary (fetched and read): [xiph/vorbis COPYING](https://raw.githubusercontent.com/xiph/vorbis/master/COPYING) · [xiph/vorbis lib/psy.c](https://raw.githubusercontent.com/xiph/vorbis/master/lib/psy.c) · [xiph/opus COPYING](https://raw.githubusercontent.com/xiph/opus/main/COPYING) · [xiph/opus celt/celt_encoder.c](https://raw.githubusercontent.com/xiph/opus/main/celt/celt_encoder.c) · [lame COPYING](https://raw.githubusercontent.com/lameproject/lame/master/COPYING) · [faac COPYING](https://raw.githubusercontent.com/knik0/faac/master/COPYING) · [fdk-aac NOTICE](https://raw.githubusercontent.com/mstorsjo/fdk-aac/master/NOTICE) · [FFmpeg LICENSE.md](https://raw.githubusercontent.com/FFmpeg/FFmpeg/master/LICENSE.md) · [FFmpeg libavcodec/aacpsy.c](https://raw.githubusercontent.com/FFmpeg/FFmpeg/master/libavcodec/aacpsy.c) · [MoSQITo LICENSE](https://raw.githubusercontent.com/Eomys/MoSQITo/master/LICENSE) · [SQAT LICENSE](https://raw.githubusercontent.com/ggrecow/SQAT/master/LICENSE) · [AMT licence statement](https://www.amtoolbox.org/) · [torch-amt PyPI metadata](https://pypi.org/pypi/torch-amt/json) · [3GPP TS 26.403 archive](https://www.3gpp.org/ftp/Specs/archive/26_series/26.403/)

Secondary: [Herre & Dick, Psychoacoustic Models for Perceptual Audio Coding — A Tutorial Review, Appl. Sci. 9(14):2854, 2019](https://www.mdpi.com/2076-3417/9/14/2854) · [Painter & Spanias, Perceptual Coding of Digital Audio](http://www.mp3-tech.org/programmer/docs/audiopaper1.pdf) · [AES audio coding tutorial, AudioLabs Erlangen](https://www.audiolabs-erlangen.de/content/resources/aesCodingTutorial/media/tutorial/aes.pdf) · [DAFx-04: AAC encoder spreading-function optimisation](https://dafx.de/paper-archive/2004/P_335.PDF) · [Implementing Loudness Models in MATLAB, DAFx](https://dafx.de/paper-archive/2004/P_177.PDF) · [auditory-list: complexity of psychoacoustic models, 2006](https://auditory.org/postings/2006/73.html) · [auditory-list: Loudness Toolbox announcement, 2010](https://auditory.org/postings/2010/140.html) · [Cambridge hearing group software](https://www.psychol.cam.ac.uk/hearing/auditory-demonstrations-and-useful-software-1) · [Suzuki, Takeshima & Kurakata on ISO 226:2023, Acoust. Sci. Tech. 45(1)](https://doi.org/10.1250/ast.e23.66) · [ISO 226:2023 preview](https://webstore.ansi.org/preview-pages/ISO/preview_ISO+226-2023.pdf) · [Sottek et al., Forum Acusticum 2023](https://dael.euracoustics.org/confs/fa2023/data/articles/000585.pdf) · [Neural Speech and Audio Coding survey, arXiv 2408.06954](https://arxiv.org/pdf/2408.06954) · [MP3net, arXiv 2101.04785](https://arxiv.org/pdf/2101.04785) · [torch-amt moore2016 source](https://torch-amt.readthedocs.io/en/stable/_modules/torch_amt/models/moore2016.html) · [3GPP TS 26.410 (reference C code)](https://www.freecalypso.org/pub/GSM/3GPP/archive/26_series/26.410/26410-b00.pdf) · [Wikipedia: Pre-echo](https://en.wikipedia.org/wiki/Pre-echo)
