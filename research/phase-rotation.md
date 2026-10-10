# Research 06 — Phase Rotation: achievable Reduction Depth and audibility

Ticket: issue #6. Date: 2026-10-10.

Evidence is tagged throughout:
**[P]** verified from a primary source (peer-reviewed paper, patent, manufacturer's own technical paper),
**[C]** community/forum claim or vendor marketing,
**[I]** my own inference or calculation.

---

## 1. Headline answer

**Phase Rotation realistically buys 1–3.5 dB of Reduction Depth, and only on material whose waveform is asymmetric or transient-dominated.** The best single published figure for a music-like source is **3.5 dB on an isolated bass drum**; the best for speech is **3–4 dB of peak-to-average reduction**, claimed by Orban. On already-symmetric or already-compressed material it delivers **0 dB, and can make the peak worse**.

That overlaps the project's 3–6 dB transparency target only at the bottom. Phase Rotation alone cannot hit the target; it is a pre-stage that shaves the first dB or two off so the Clipping stage has less to distort.

Two load-bearing caveats that the published numbers hide:

1. **Almost all published numbers are measured on isolated one-shot transients** (a bass drum sample, a snare sample, a hi-hat, castanets, gunshots, beeps) — not on full mixes. No primary source gives a figure for Phase Rotation applied to a mastered stereo mix. **[P/I]**
2. **The parameter settings that buy the most reduction are the ones with the most group delay**, and they exceed published group-delay audibility thresholds. See §5.

---

## 2. The mechanism: asymmetry, verified

Orban and Foti's own 2001 AES paper is the primary source, and it is explicit:

> "The phase rotator is a chain of allpass filters (typically four poles, all at 200Hz) whose group delay is very non-constant as a function of frequency. Many voice waveforms (particularly male voices) exhibit as much as 6dB asymmetry. The phase rotator makes voice waveforms more symmetrical and can sometimes reduce the peak-to-average ratio of voice by 3-4dB."
> — R. Orban and F. Foti, "What Happens to My Recording When It's Played on the Radio?", AES 111th Convention (2001 Nov.), paper 5469. Freely hosted copy: https://totalsonic.net/manuals/Radioprocessing_Bob_Orban_2001.pdf **[P]**

So the developer's expectation is confirmed by the canonical source: **male voice up to 6 dB asymmetry, 3–4 dB peak-to-average reduction.** Note the unit — Orban says *peak-to-average ratio*, not Sample Peak or True Peak. **[P]**

An independent engineering treatment corroborates the mechanism and adds a measured asymmetry figure for a synthetic speech model:

- James L. Tonne, "All-Pass Networks in a Speech Chain", *QEX* Nov/Dec 2017, pp. 30–34 (ARRL). https://www.arrl.org/files/file/QEX_Next_Issue/NovDec2017/Tonne.pdf **[P]**
  - Builds an LTspice speech simulator with "about 10 dB of asymmetry … and no dc component", 200 Hz fundamental.
  - Confirms the degree of asymmetry "is highly dependent on the voice of the individual" — i.e. per-source, not a fixed constant.
  - Shows a high-pass filter is *not* a substitute: "the peak-to-peak voltage value has actually increased."
  - Documents the historical analogue precedent: the **Kahn Communications SymmetraPeak** (c. 1959), a lumped-element allpass network built for exactly this, and an op-amp 8-pole allpass by Gary Blau W3AM whose "phase shift … goes from near zero degrees at very low audio through near 1440 degrees at extremely high audio frequencies."

### 2.1 The failure mode — this is the important one

Tonne documents, and simulates, the case the marketing never mentions:

> "If the applied waveform is symmetrical top-to-bottom in the first place — it has no even-order harmonic content — then the all-pass can actually increase the signal peak-to-peak amplitude. … This aspect of the all-pass is usually glossed over." **[P]**

This is independently confirmed in the audio literature: the Orban-style rotator (fc = 200 Hz, pole radius r = 0.80) applied to a bass drum **increased** its maximum value by **0.83 dB**. **[P]** (Välimäki et al. 2022, §2.2 — full cite below.)

**Design consequence:** a fixed-parameter Phase Rotation stage is not safe. Every published method includes a per-frame **bypass** when no reduction is available — Lynch, Parker, and the Aalto group all say so explicitly ("a bypass mode must still be implemented, implying no processing, for signals that cannot be reduced"). **[P]** The project already plans the stage as bypassable; this says the bypass must also be *automatic and per-frame*, not just a user switch.

---

## 3. Topologies used in practice

| Topology | Source | Detail |
|---|---|---|
| 4 cascaded 2nd-order allpass sections, all poles at 200 Hz | Orban & Foti 2001 **[P]** | Pole radius not disclosed. Välimäki et al. found r = 0.80 "yielded a good result" by manual search. |
| Same, with pole frequency and radius searched per frame | Välimäki et al. 2022 ("extended phase rotator") **[P]** | fc ∈ [40, 200] Hz, r ∈ [0.6, 0.98], 5 frequencies × 40 radii = 200 candidate parameter sets. 32 operations/sample. Real-time viable. |
| 3 cascaded Schroeder allpasses, delay + feedback gain searched | Parker & Välimäki 2013, IEEE SPL 20(7):669–672 **[P]** | 100 parameter settings. 12 ops/sample. |
| Same, millions of settings, GPU | Belloch et al., EUSIPCO 2014, pp. 890–894 **[P]** | >10⁷ candidates. Offline only. |
| Single sparse Schroeder allpass, delay synced to autocorrelation peaks, gain by gradient descent | Schlecht, Fierro, Välimäki, Backman, ICASSP 2022, pp. 1006–1010; patented as **US 11881829 B2** (AAC Technologies, granted 2024-01-23) **[P]** | H(z) = (g + z⁻ᵐ)/(1 + g·z⁻ᵐ). m up to ~300 samples (≈6.8 ms @ 44.1 kHz), g ≈ ±0.6. 3 MUL + 5 ADD per sample per iteration. |
| Wideband linear chirp FIR ("quadratic phase dispersion") | Lynch, JAES 36(3):147–152 (1988) **[P]** | Not an allpass; an FIR chirp. See §4. |
| Lumped-element analogue allpass | Kahn SymmetraPeak, c. 1959 **[P]** via Tonne | Historical precedent. |

Note the two families differ in character. The Orban-style **low-frequency 2nd-order cascade** disperses energy within a cycle or two of the bass. The **Schroeder allpass** family uses a bulk delay of up to ~7 ms and works by moving the peak's energy onto a nearby autocorrelation peak — a different, longer-timescale mechanism.

---

## 4. The numbers, by method and by material

### 4.1 Published per-source results — all isolated one-shots

Välimäki, Fierro, Schlecht, Backman, **"Audio Peak Reduction Using Ultra-Short Chirps"**, *JAES* 70(6):485–494 (2022 June), DOI 10.17743/jaes.2022.0011. Open access: https://acris.aalto.fi/ws/portalfiles/portal/86964673/Audio_Peak_Reduction_Using_Ultra_Short_Chirps.pdf **[P]**

Peak reduction in dB (their Table 3):

| Method | Bass drum | Snare | Hi-hat | Piano | Mallet |
|---|---|---|---|---|---|
| Parker [allpass, 100 candidates] | 0.5 | 2.3 | 1.4 | 1.3 | 1.2 |
| Belloch [GPU, >10⁷ candidates, offline] | 2.6 | **3.2** | **2.5** | **2.3** | **2.7** |
| Linear chirp (whitened, ≤4 ms) | 0.2 | 2.0 | 1.3 | 0.5 | 1.4 |
| **Extended phase rotator** | **3.5** | 2.7 | 1.3 | 1.6 | 1.3 |

Best parameters found (their Table 2) — note bass drum and snare both want **40 Hz, r = 0.98**; hi-hat wants 200 Hz, r = 0.80; mallet 160 Hz, r = 0.65. **[P]**

The ICASSP companion paper gives a second, partly different set on the same five sources (grid search over the Schroeder allpass is the stronger method there): Bass 2.1, Snare 4.9, Hi-hat 3.6, Piano 3.1, Mallet 4.3 dB. **[P]**

### 4.2 Distribution over a 105-sound corpus

Both Aalto papers evaluate on the same 105-sample set of "drums, knocks, speech excerpts, shots, and other impulsive sounds", deliberately including "pre-compressed audio and sounds presenting a small crest factor, which are typically hard to reduce". **[P]**

| Method | Share of samples reaching the stated Reduction Depth |
|---|---|
| Extended phase rotator (200 candidates, real-time) | ≥1 dB on **43%**; ≥3 dB on **>10%** |
| Whitened linear chirp (≤4 ms) | ≥1 dB on only **~33%** |
| Schroeder allpass, exhaustive grid search (offline ground truth) | ≥2.5 dB on **>50%** |
| SyncAPF (real-time) | ≥1 dB on **>65%** |

Paper's own summary: **"The practical peak reduction obtained with the proposed methods spans from 0 to about 3.5 dB."** **[P]**

So on an impulsive-sound corpus, more than half the samples get **under 1 dB** from a real-time Orban-style rotator. **[I]** That is the realistic expectation for a bypassable stage, and it is well short of the 3–6 dB Reduction Depth target on its own.

### 4.3 Other reference points

- Lynch 1988: **1.3 dB** average on voice from the dispersion filter alone; **7.7 dB** when combined with a nonlinear DRC. **[P]** The 7.7 dB figure is often quoted loose — it is *not* phase processing alone.
- Theoretical ceiling for an impulse: the 4-pole/200 Hz rotator's own impulse response peaks at ≈0.63, i.e. **≈4 dB** down. A perfect unit impulse is the easiest possible case. On a real drum track the same filter gave **≈1.6 dB**. (Universal Audio, "Allpass Filters" technical blog, https://www.uaudio.com/blogs/ua/allpass-filters) **[P]** — manufacturer technical writing, numbers from their own figures.
- Välimäki et al. show the same effect as a symmetry change: a mallet sample's positive/negative peak ratio went from **1.12 to 1.02** while the peak dropped 1.6 dB; a hi-hat went to a 1.01 ratio for 1.3 dB. **[P]**
- **BiteRotator** (4bite), a commercial adaptive phase-rotation plugin for voice: claims "several dB" on "sufficiently asymmetric material", adaptive mode 764 ms latency, fixed mode 13.6 ms, mono voice/dialogue only, explicitly "no meaningful benefit on already-balanced waveforms", with internal guards against increasing the peak. No dB figure, no independent verification in the thread. **[C]**

---

## 5. Frequency band: the expectation is confirmed, with a sting

### 5.1 Low frequencies do yield the most — but not exclusively

Confirmed: the two sources whose energy is low and asymmetric (bass drum, snare) both optimise to the **lowest** pole frequency tested, 40 Hz, at the **highest** pole radius, 0.98. The hi-hat optimises to 200 Hz and a modest r = 0.80; the mallet to 160 Hz, r = 0.65. **[P]** The paper also reports that for the hi-hat and mallet the reduction "is mainly or solely dependent on the pole radius" (heat-map stripes), while for the bass drum "the optimal reduction concentrates in a low-frequency region near the unit circle". **[P]**

**So: LF-asymmetric content yields most, as expected — but the band that pays is a function of where the transient's energy is, not a universal 200 Hz.** A fixed band is leaving reduction on the table for HF transients. **[I]**

### 5.2 Why band-limiting is the right call — the audibility thresholds favour LF

This is the strongest argument for the developer's band-limiting proposal, and it is well sourced. Group-delay audibility thresholds **rise steeply towards low frequencies**:

| Study | Centre frequency → threshold (peak group delay, ms) |
|---|---|
| Jensen & Møller (via Liski et al.) | 250 Hz → **4.7**; 500 Hz → 2.3; 800 Hz → 1.5; 1200 Hz → 1.6; 2000 Hz → 1.6 |
| Blauert & Laws (via Liski et al.) | 500 Hz → **3.2**; 1 kHz → 2.1; 2 kHz → **0.9**; 4 kHz → 1.3; 8 kHz → 1.9 |
| Liski, Mäkivirta, Välimäki, "Audibility of Group-Delay Equalization", *IEEE/ACM TASLP* 29:2189–2201 (2021), open access https://acris.aalto.fi/ws/portalfiles/portal/66449704/Audibility_of_Group_Delay_Equalization.pdf | Treats **500 Hz – 4 kHz as "the most sensitive frequency range"** and deliberately cancels LF group delay to isolate it. Smallest mean ABX thresholds: **−0.56 ms** (negative) and **+0.64 ms** (positive), both with a pink impulse. Negative group-delay variation is more audible than positive. |

All **[P]**. Three further findings from Liski et al. that matter here:

- Thresholds are **"highly dependent on the signal"**. Unit impulse and pink impulse are the critical cases; a synthetic hi-hat decaying 60 dB in 80 ms *hides* a positive group-delay variation; a recorded castanet is harder to hear than the synthetic signals. **[P]**
- Their 2021 thresholds are **smaller than previous studies'** — the literature has been tightening, not loosening.
- Negative group delay (energy arriving *earlier*) is worse than positive. A causal minimum-phase-ish allpass gives positive delay, which is the favourable direction. **[I]**

### 5.3 My own calculation — the configuration that buys 3.5 dB is over threshold

I computed the group delay of the two configurations the Aalto paper names, for four cascaded 2nd-order allpass sections at 44.1 kHz. (Script: `scratchpad/gd.py`. Sanity check: my fc = 200 Hz, r = 0.80 case gives 1.61 ms at DC, matching the paper's stated "about 1.6 ms near 0 Hz" — so the method is right.) **[I]**

| Frequency | Orban-style: fc = 200 Hz, r = 0.80 | Extended: fc = 40 Hz, r = 0.98 (the 3.5 dB bass-drum setting) | Middle ground: fc = 40 Hz, r = 0.90 |
|---|---|---|---|
| 40 Hz | 1.61 ms | **15.8 ms** | 3.43 ms |
| 100 Hz | 1.60 ms | **12.2 ms** | 3.38 ms |
| 200 Hz | 1.58 ms | **6.3 ms** | 3.21 ms |
| 400 Hz | 1.52 ms | **2.05 ms** | 2.67 ms |
| 800 Hz | 1.29 ms | 0.55 ms | 1.59 ms |
| 2 kHz | 0.63 ms | 0.09 ms | 0.42 ms |

Reading this against §5.2 **[I]**:

- The **classic Orban configuration is comfortably inside every published threshold** — 1.6 ms at LF against a 4.7 ms threshold at 250 Hz, and only 0.63 ms at 2 kHz against a 0.9 ms threshold. It is a conservative, defensible setting. That is consistent with it having survived decades of broadcast use.
- The **high-r extended configuration that buys 3.5 dB is not** — 6.3 ms at 200 Hz against a ~4.7 ms threshold at 250 Hz, and 2.05 ms at 400 Hz where the threshold is ~1.5–2.3 ms. This is almost certainly why the paper's authors report that above r ≈ 0.98 "the phase rotator produces a disturbing change in the signal timbre". **[P]** for the quote, **[I]** for the connection.
- **A 2nd-order allpass is not band-limited.** Its group delay has a long low-frequency plateau and only rolls off *above* the pole. The fc = 40 Hz / r = 0.98 case still puts 2 ms at 400 Hz and 0.55 ms at 800 Hz. **If the project wants a genuinely band-limited stage it needs an explicit crossover (process the low band, pass the rest through a matched delay), not just a low pole frequency.** That is a real architectural decision, and latency is unlimited, so a linear-phase crossover is affordable. **[I]**
- The pole radius, not the pole frequency, is the audibility knob. Mapping Reduction Depth onto r means audibility degrades with depth — which fits "graceful degradation beyond 3–6 dB", but means the stage cannot be assumed Transparent at its maximum setting. **[I]**

---

## 6. Audibility evidence

### 6.1 What the primary sources actually say

**Orban and Foti concede a cost on music, in their own paper.** This is the single most useful audibility datapoint, and it is usually left out of forum summaries:

> "Because this processing is linear (it adds no new frequencies to the spectrum, so it doesn't sound raspy or fuzzy) it's the closest thing to a 'free lunch' that one gets in the world of transmission processing. There are a few prices to pay. In the good old days when source material wasn't grossly clipped, the main price was **a very subtle reduction in transparency and definition in music**. This was widely accepted as a valid trade-off to achieve greatly reduced speech distortion, because the phase rotator's effects on music are unlikely to be heard on typical consumer radios, like car radios, boomboxes, 'Walkman'-style portables, and table radios." **[P]**

Read that carefully. The canonical source says the effect on music **is** a reduction in transparency and definition, and justifies it by the *playback system* — car radios and boomboxes — not by inaudibility. **That justification does not transfer to a mastering plugin judged by blind ABX on monitors.** **[I]** This is the most consequential finding in the ticket.

Orban and Foti also describe the second-order consequence relevant to this project: a rotator *downstream* of a clipper redistributes the flat tops so they

> "end up looking like little smooth sections of the waveform where all the detail is missing — a bit like a scar from a severe burn. … they no longer help reduce the peak-to-average ratio of the waveform. Instead, their only effect is to add unnecessary grungy distortion." **[P]**

**Design consequence:** Phase Rotation must sit **before** the Clipping stage, never after. If it runs after, it un-does the flat-topping that made the ceiling hold, raising the peak back up while keeping the harmonic distortion. **[P/I]** Tonne independently says the same for analogue chains: "Placement of the all-pass block should always be prior to the point where clipping occurs or where it might occur." **[P]**

### 6.2 Formal listening tests — what exists and what does not

**No published formal listening test evaluates the audibility of an Orban-style phase rotator itself.** I searched AES, IEEE, JAES, DAFx and the patent literature and found none. What exists is adjacent and bounds the problem:

- **Välimäki et al. 2022, webMUSHRA listening test** — 16 experienced listeners, Sennheiser HD650, soundproof booth at Aalto; 2 of 16 screened out. Stimuli: unit impulse, pink impulse, synthetic hi-hat, recorded castanets, filtered with 2/4/8/12/20 ms chirps, rated for "chirpiness". Results: 2 ms "barely distinguishable from the reference" (MOS 4–5); 4 ms MOS 3–4 on the impulses but still high on hi-hat and castanets; 8 ms MOS 2–3; 12 and 20 ms MOS 1–2. Conclusion: cap the dispersion at **4 ms**. **[P]**
  - This tests the *chirp* method, not the rotator. But it is the same perceptual quantity — dispersion length — and it gives a hard number: **≈2 ms of dispersion is near-transparent, 4 ms is the practical limit, 8 ms is clearly audible.** **[I]**
  - It also confirms **masking is material-dependent in the project's favour**: "the audibility of the chirp effect is higher for impulsive sounds, whereas the phase dispersion is partially masked for noisy or reverberant transients, such as the synthetic hi-hat and castanet sounds." **[P]** A dense mix will mask more than a solo click.
- **Liski et al. 2021 ABX thresholds** — see §5.2. The directly applicable, properly blinded numbers. **[P]**
- Universal Audio's technical blog states the rule of thumb: smearing "on a time scale under 5–10 ms is unlikely to cause significant perceptual artifacts", and "with short impulse responses, allpass filters can modify phase in a transparent way", while "with longer impulse responses, they can create audible effects while preserving spectral balance". **[P]** (manufacturer technical writing, no test data behind it — treat as expert opinion.)
- **US 11881829 B2 reports no listening test at all.** Its audibility claim ("inaudibly smear the energy") is unverified. **[P]**

### 6.3 Community/forum consensus — labelled as such

The forum record is thin, divided, and mostly unquantified. **[C]** throughout:

- Phase rotation is **near-universal but near-unknown** in the mastering community. Parker Dinkins (MasterDigital) on the ARSCLIST mailing list: "Few audio engineers have ever heard of phase rotation", and "virtually all audio you hear on the radio undergoes this process."
- **Favourable:** KVR posters report liking it "on acoustic drum buses and especially acoustic kick" and on acoustic bass, and note it "can noticeably reduce peak-to-peak swing" while acting "like frequency-dependent predelay".
- **Unfavourable, from a broadcast-processor developer:** the Stereo Tool developer's forum position is that it is "really bad to use phase rotation when it isn't needed" and that the effect must be switchable — he later shipped it as a switchable feature anyway.
- **Scope caveat, repeated:** a KVR poster notes it is "only really suited for voice" because of glottal pulse asymmetry, "and that results vary by voice."
- I could **not** find a mastering-forum thread in which engineers report ABX results, or in which anyone claims it ruins a master. The "mastering engineers divide on this" framing in the ticket is real but the division is expressed as preference, not as test outcomes. **[C]**

### 6.4 Verdict

**Not Transparent by assertion; plausibly Transparent at conservative settings; not Transparent at the settings that buy the most dB.** Specifically **[I]**:

- At ≈1.6 ms of LF group delay (classic Orban configuration), every published threshold is cleared by a factor of 2–3, and the expected Reduction Depth is **1–2 dB** on favourable material and 0 on the rest.
- At the ≈6–16 ms settings needed for 3–3.5 dB, published thresholds are **exceeded**, the paper's own authors report "a disturbing change in the signal timbre", and this must be assumed audible until the project's Pass Gate says otherwise.
- The project's Pass Gate is the right instrument and this stage genuinely needs it. Build the stage, parameterise r as the depth control, and ABX at each depth. The critical Reference Set items are **solo-ish impulsive material** (unit-impulse-like clicks, dry kick, dry snare, close-miked percussion) and **solo male voice**; dense, reverberant, noisy material will mask the artifact and will not discriminate. **[P]** for the masking finding, **[I]** for the Reference Set implication.

---

## 7. True Peak vs Sample Peak

**No primary source addresses this.** Every published peak-reduction result above is a **Sample Peak** figure; Orban's is a peak-to-average figure. I found nothing measuring Phase Rotation against ITU-R BS.1770 True Peak. **[P: absence]**

My analysis **[I]**:

1. **In principle it should work equally well on True Peak.** An allpass network is an LTI operation with a flat magnitude response. It acts on the band-limited continuous-time waveform the DAC reconstructs just as it acts on the samples; there is no mechanism by which it would favour the sample grid. Unlike Clipping — which creates new high-frequency energy, aliases, and therefore new inter-sample peaks — Phase Rotation adds no new frequencies, so it cannot manufacture inter-sample overshoot. This is a genuine advantage over Clipping and worth stating in the design rationale.
2. **But it will only reduce True Peak if you optimise against True Peak.** Every published method is a *search*: it tries N parameter sets and keeps the one minimising the measured peak. If the objective function is `max|x[n]|`, the search will happily find a parameter set that lowers Sample Peak by 3 dB while leaving True Peak unchanged — or raising it, since the sample-grid alignment of the peak is exactly what phase rotation shuffles. This is a concrete, avoidable bug.
3. **Actionable:** make the search objective an **oversampled True Peak** measurement (4× minimum, per BS.1770-4), not `max|x[n]|`. Latency is unlimited, so the cost is acceptable. Then also measure what the gap does: I would expect Phase Rotation to *reduce the Sample Peak–True Peak gap*, because making the waveform smoother and more symmetric around its peak reduces inter-sample overshoot — but that is a hypothesis to test, not a finding.
4. **Open question worth a measurement:** does the Reduction Depth available in True Peak terms differ from the Sample Peak figures above? Nobody has published this. It is a cheap experiment on the Reference Set and would be a genuinely novel number.

---

## 8. Per-bin spectral phase manipulation — prior art

### 8.1 Does the exact thing exist?

**I found no prior art for per-bin STFT phase optimisation aimed at audio peak reduction.** Searched: Google Patents, USPTO full-text, AES E-Library, IEEE, DAFx archive. The absence is not proof — this is a diffuse area with imprecise vocabulary — but it is a reasonably thorough negative. **[P: absence]**

What exists is adjacent, in three clusters:

**(a) Per-bin spectral phase manipulation in audio, for other purposes — exists, and is evaluated.**
- **C. J. Keyes and A. Tan, "Design and Evaluation of a Spectral Phase Rotation Algorithm for Upmixing to 3D Audio", JAES 68(11):856–864 (2020)**, DOI 10.17743/jaes.2020.0055. **[P]** This is literally per-bin spectral phase rotation on music, with listening tests — reporting increased spaciousness and liveliness and significant listener preference. It is *not* peak reduction, and the perceptual change it produces is exactly the kind a mastering plugin must avoid: a change in spatial imagery. **This is the closest prior art and the clearest warning.**
- Dolby's decorrelation patent family (US 9520135, 9640188, 9691404, 9715882, 10269364, 10796706, 11308969) adds a randomised phase shift per STFT bin, held constant per bin while the scale factor updates per frame, with the amount "controlled by a Decorrelation Scale Factor in a manner that minimises audible signal warbling artifacts", plus a separate transient-handling mode. **[P]** The artifact vocabulary — warbling, block-rate artifacts — is the honest account of what per-bin phase does.
- G. S. Kendall, "The Decorrelation of Audio Signals and Its Impact on Spatial Imagery", *Computer Music Journal* 19(4):71–87 (1995) **[P]** — the foundational treatment; phase decorrelation changes spatial imagery, which is the project's Stereo Link concern.
- Välimäki, Abel & Smith, "Spectral Delay Filters", JAES 57(7/8) (2009) **[P]** — frequency-dependent delay as a deliberate *effect*. Worth reading as a map of where this becomes audible.

**(b) Audio peak reduction via phase — exists, but allpass-based, not per-bin.** All of §3. US 11881829 B2 is live and worth a glance for freedom-to-operate, though its claims are specific to the sparse-allpass + autocorrelation-sync + gradient-descent method and do not read on a per-bin STFT approach. **[P]** (Not legal advice.)

**(c) RF/OFDM PAPR reduction — the formal version of the problem.** See §8.2.

### 8.2 Does the OFDM PAPR literature transfer?

Partly, and the transfer is informative about the ceiling rather than the method. **[P]** for the figures, **[I]** for the transfer argument.

**Selected Mapping (SLM)** is exactly the project's hypothetical: multiply the subcarriers by one of U candidate phase vectors, transmit whichever gives the lowest peak. Published gains at CCDF 10⁻³ with U = 16 candidates: **5.4 dB at 64 subcarriers, 4.4 at 128, 4.3 at 256, 4.1 at 512, 3.9 dB at 1024**. Gains shrink as the number of subcarriers grows and grow with the number of candidates, with diminishing returns. Refined phase-sequence designs report ~1.2–2.4 dB over conventional SLM. The relevant patent for the audio-adjacent version is **US 8509345**, "Crest factor reduction with phase optimization" (expired per Google Patents), which selects a phase offset per carrier by trial-and-keep-the-lowest-peak. **[P]**

**Why the numbers do not transfer directly [I]:**

1. **The bin-count trend runs the wrong way.** SLM's gain *falls* as subcarriers increase — 5.4 dB at 64 bins down to 3.9 dB at 1024. A mastering STFT is 2048–8192 bins. Extrapolating the trend puts the available gain at or below the 3.5 dB that a four-pole analogue-style allpass already delivers, for vastly more computation. The band-limited approach is not obviously leaving anything on the table.
2. **OFDM has no listener.** SLM sends *side information* telling the receiver which phase vector was used, so the receiver undoes the rotation exactly. The audio case has no receiver: **the phase change is permanent and the listener hears it.** This is the fundamental disanalogy and it is fatal to a naive port.
3. **OFDM subcarriers are independent symbols; audio bins are not.** The partials of a note have a phase relationship that defines the waveform's shape and the transient's attack. Per-bin rotation breaks that relationship. This is why Dolby's decorrelators need a Decorrelation Scale Factor and a transient mode, and why Dorran, Coyle & Lawlor (DAFx 2004) had to invent horizontal-phase-coherence constraints for phase-vocoder time-scaling. **[P]** for those sources.
4. **Phase coherence across frames is an additional constraint OFDM does not have.** Overlap-add of frames with independently chosen phases produces frame-rate modulation — warbling. The ICASSP authors flag the same issue even for their single-filter method: "continuity between frames requires additional attention to avoid signal discontinuities. A necessary trade-off is between frame size, lookahead, and perceptual quality." **[P]**

**What it buys beyond the band-limited approach [I]:** on the evidence, **little, at high artifact risk.** The honest statement is:

- There is **no prior art**, so it is novel — but novelty here is probably because it is a bad trade, not because nobody thought of it.
- The quantity the extended phase rotator already optimises per frame (pole frequency × radius, 200 candidates) is a two-parameter, smoothly-varying, *provably spectrally flat and provably inter-frame continuous* subspace of the full per-bin phase space. Per-bin optimisation gains access to the rest of that space, but every extra degree of freedom is a degree of freedom to break transient coherence.
- A defensible middle path, if the project wants to explore it: optimise phase **per band, not per bin** — a small number of group-delay offsets over a handful of bands, which is a frequency-dependent allpass (and is exactly the extension the ICASSP paper names as future work: "the proposed method could be extended to frequency-dependent APF"). **[P]** for the quote. That keeps the coherence guarantees while widening the search.
- Where per-bin *could* legitimately win: using the project's existing **Masking Threshold** model to constrain the phase change per bin to a perceptually-weighted budget, rather than a hard dispersion-length limit. That is genuinely unexplored and sits naturally alongside the residual-shaping work. But it is a research project, not a 1.0 feature. **[I]**

---

## 9. Recommendations for the stage as scoped

Taking the settled decisions (hard True Peak ceiling, 3–6 dB Reduction Depth target, unlimited latency, opt-in bypassable band-limited stage, off by default):

1. **Keep it opt-in and off by default.** Correct call, and the evidence supports it more strongly than expected — Orban's own paper concedes a transparency cost on music. **[P]**
2. **Place it strictly before the Clipping stage.** Orban & Foti and Tonne both say a rotator after a clipper converts flat tops into pure distortion with no peak benefit. **[P]**
3. **Make the bypass automatic and per-frame, not just a user switch.** An allpass *increases* the peak on symmetric material — verified both in simulation (Tonne) and in measurement (+0.83 dB on a bass drum). Every published method includes a bypass-if-no-gain rule. **[P]**
4. **Optimise against oversampled True Peak, not `max|x[n]|`.** Otherwise the search can buy Sample Peak reduction that the True Peak ceiling never sees. **[I]** — the cheapest high-value finding here.
5. **Implement band-limiting with an explicit crossover**, not merely a low pole frequency. A 2nd-order allpass at 40 Hz still puts 2 ms of group delay at 400 Hz. Latency is unlimited, so a linear-phase crossover is affordable. **[I]**
6. **Parameterise depth as pole radius, and cap it.** r ≤ 0.9 keeps LF group delay near 3.4 ms, inside Jensen & Møller's 4.7 ms threshold at 250 Hz; r = 0.98 does not. Expect ~1–2 dB at the safe setting. **[I]**
7. **Set expectations in the UI.** On more than half of impulsive material a real-time rotator delivers under 1 dB. The stage should meter what it actually achieved per frame, or users will assume it is broken. **[I]**
8. **Pass Gate stimuli must include solo impulsive material and solo male voice.** Dense/reverberant material masks the artifact and will not discriminate. **[P]** for the masking result.
9. **Do not pursue per-bin phase optimisation for 1.0.** No prior art, no evidence of headroom beyond the allpass approach, and the OFDM analogy breaks on the point that matters (no receiver to undo the rotation). Revisit only as masking-constrained per-*band* group delay. **[I]**

---

## 10. What I could not confirm

- **The Orban & Foti AES paper itself is paywalled.** I used a freely-hosted copy at totalsonic.net, dated June 2001 versus the AES convention date of November 2001. The text matches the AES abstract and the quotations circulating from the Optimod manuals, but I have not diffed it against the AES version. The "4 poles at 200 Hz / 6 dB asymmetry / 3–4 dB" passage should be treated as high-confidence but formally unverified against the AES copy.
- **Orban's pole radius is still undisclosed.** Välimäki et al. say so explicitly and had to find r = 0.80 by manual search. No Optimod manual I could reach states it.
- **I could not reach an Optimod manual directly.** Gearspace and the ProSoundWeb R/E/P forum both return HTTP 403 to automated fetches; the Optimod-FM 8600 manual is not on an open URL I found. The frequently-quoted manual passage (sections 2-9 to 2-13) appears to be the same text as the 2001 paper, which I do have.
- **No measured asymmetry statistics for a corpus of real programme material.** Orban's "as much as 6 dB" for male voice and Tonne's "about 10 dB" for a synthetic model are the only figures I found. Nobody has published a distribution of waveform asymmetry over, say, 100 commercial tracks. This would be a useful and cheap thing for the project to measure on its own Reference Set — it directly predicts how much the stage can deliver.
- **No figure for Phase Rotation applied to a full mix.** Every number in §4 is an isolated one-shot. The 3.5 dB bass-drum figure is a solo kick sample, not a kick inside a master. The expected value on a dense mix is almost certainly lower, and unpublished.
- **Nothing on True Peak specifically.** §7 is entirely my own reasoning.
- **No blinded listening test of a phase rotator.** §6.2's numbers are from adjacent experiments (chirp dispersion, group-delay equalisation). The direct test does not exist in the literature.
- **Forum evidence is weak and I could not strengthen it.** The two most promising threads (Gearspace "Phase rotator", R/E/P) are 403-blocked to fetching, so some of the forum material in §6.3 reaches me via search-engine summaries of those pages rather than the pages themselves. Treat §6.3 as indicative only.
- **Keyes & Tan (2020) full text is paywalled.** I have the abstract and the bibliographic record; I have not read their per-bin mechanics or the detail of their listening tests. Worth buying if per-bin is ever revisited — it is the nearest prior art.

---

## Sources

**Primary — papers**
- R. Orban and F. Foti, "What Happens to My Recording When It's Played on the Radio?", AES 111th Convention, paper 5469 (2001 Nov.). Free copy: https://totalsonic.net/manuals/Radioprocessing_Bob_Orban_2001.pdf — AES record: https://aes2.org/publications/elibrary-page/?id=9812
- R. Orban, F. Foti and B. Katz, in *Mastering Audio: The Art and the Science*, pp. 271–278 (Focal Press, 2002) — cited by Välimäki et al. as the second Orban/Foti source; not read.
- V. Välimäki, L. Fierro, S. J. Schlecht, J. Backman, "Audio Peak Reduction Using Ultra-Short Chirps", *JAES* 70(6):485–494 (2022). DOI 10.17743/jaes.2022.0011. Open access: https://acris.aalto.fi/ws/portalfiles/portal/86964673/Audio_Peak_Reduction_Using_Ultra_Short_Chirps.pdf
- S. J. Schlecht, L. Fierro, V. Välimäki, J. Backman, "Audio Peak Reduction Using a Synced Allpass Filter", ICASSP 2022, pp. 1006–1010. DOI 10.1109/ICASSP43922.2022.9747877. Open access: https://aaltodoc.aalto.fi/items/50773735-c5fe-47f0-80e0-305e09402ff4
- J. Liski, A. Mäkivirta, V. Välimäki, "Audibility of Group-Delay Equalization", *IEEE/ACM TASLP* 29:2189–2201 (2021). Open access: https://acris.aalto.fi/ws/portalfiles/portal/66449704/Audibility_of_Group_Delay_Equalization.pdf
- J. T. Lynch, "Reduction of Peak/rms Ratio of Speech by Amplitude Compression and Quadratic Phase Dispersion", *JAES* 36(3):147–152 (1988) — via Välimäki et al.; not read directly.
- J. Parker and V. Välimäki, "Linear Dynamic Range Reduction of Musical Audio Using an Allpass Filter Chain", *IEEE SPL* 20(7):669–672 (2013). DOI 10.1109/LSP.2013.2263136
- J. A. Belloch, J. Parker, L. Savioja, A. Gonzalez, V. Välimäki, "Dynamic Range Reduction of Audio Signals Using Multiple Allpass Filters on a GPU Accelerator", EUSIPCO 2014, pp. 890–894.
- C. J. Keyes and A. Tan, "Design and Evaluation of a Spectral Phase Rotation Algorithm for Upmixing to 3D Audio", *JAES* 68(11):856–864 (2020). DOI 10.17743/jaes.2020.0055 — paywalled; abstract only.
- G. S. Kendall, "The Decorrelation of Audio Signals and Its Impact on Spatial Imagery", *Computer Music Journal* 19(4):71–87 (1995).
- D. Dorran, R. Lawlor, E. Coyle, DAFx 2004: https://dafx.de/paper-archive/2004/P_083.PDF
- J. L. Tonne (W4ENE), "All-Pass Networks in a Speech Chain", *QEX* Nov/Dec 2017, pp. 30–34: https://www.arrl.org/files/file/QEX_Next_Issue/NovDec2017/Tonne.pdf

**Primary — patents**
- US 11881829 B2, "Method for audio peak reduction using all-pass filter", AAC Technologies, granted 2024-01-23: https://patents.google.com/patent/US11881829B2/en
- US 8509345, "Crest factor reduction with phase optimization": https://patents.google.com/patent/US8509345
- Dolby decorrelation family: US 9520135, 9640188, 9691404, 9715882, 10269364, 10796706, 11308969

**Manufacturer technical writing**
- Universal Audio, "Allpass Filters": https://www.uaudio.com/blogs/ua/allpass-filters
- Sound On Sound glossary, phase rotator: https://www.soundonsound.com/node/4923678

**Community / vendor claims [C]**
- ARSCLIST, Parker Dinkins (2007-01): https://cool.culturalheritage.org/byform/mailing-lists/arsclist/2007/01/msg00128.html
- KVR Audio, "Phase rotation for speech": https://kvraudio.com/forum/viewtopic.php?p=7199390
- KVR Audio, BiteRotator announcement thread: https://www.kvraudio.com/forum/viewtopic.php?p=9274973
- Stereo Tool forum, developer on switchability: https://www.forums.stereotool.com/viewtopic.php?p=4414
- R/E/P Community thread quoting the Optimod manual (403 to fetch): https://repforums.prosoundweb.com/index.php/topic,4162.0.html

**Own analysis [I]**
- Group-delay computation script: `scratchpad/gd.py` (validated against the 1.6 ms figure in Välimäki et al. Fig. 8).
