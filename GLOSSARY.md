# Squared Away

A spectral peak-control plugin for mastering: it holds the output beneath a true-peak ceiling while reducing dynamic range, and tries to do so without audible artifacts.

## Language

### The goal and the mechanisms

The central distinction in this project. A ceiling is a *guarantee about output*; clipping and spectral magnitude limiting are *mechanisms*, and only one of them can make that guarantee. Conflating the three is the known dead end here.

**Peak Ceiling**:
A guarantee that the output never exceeds a set level. A property of the signal leaving the plugin, independent of how it was achieved.
_Avoid_: Limit, ceiling (bare), threshold

**True Peak**:
The peak level of the reconstructed analogue waveform, including peaks that fall between samples, measured per ITU-R BS.1770-5 (current; Annex 2 substantively identical to -4) with oversampled detection.
_Avoid_: dBTP (that is the unit, not the concept), intersample peak

**Sample Peak**:
The largest absolute sample value in the signal. Always less than or equal to the true peak, and not what a DAC reconstructs.

**Clipping**:
Truncating a waveform where it exceeds a threshold, accepting the harmonic distortion this creates. A time-domain mechanism, and the only one in this project that enforces a peak ceiling intrinsically.
_Avoid_: Limiting, saturation, distortion

**Spectral Magnitude Limiting**:
Reducing the magnitude of individual frequency bins that exceed a threshold. Reduces spectral peaks but makes *no* guarantee about the resulting waveform's peak, because peak level is produced by phase alignment across bins.
_Avoid_: Spectral clipping, per-bin clipping, FFT clipping

**Phase Rotation**:
Reducing peak level by changing phase relationships rather than removing energy, typically with an allpass network over a limited frequency band. Information-preserving: the magnitude spectrum is unchanged and nothing is distorted.
_Avoid_: Phase shift, allpass filtering, phase linearization

### What processing leaves behind

**Residual**:
The difference between input and output — the signal a mechanism removed. For clipping this *is* the distortion, which makes it the object of treatment rather than a by-product to discard.
_Avoid_: Error signal, difference signal, clipped-off signal, distortion

**Masking Threshold**:
The per-bin level below which a signal component is inaudible in the presence of the rest of the programme, computed by a psychoacoustic model. The criterion the residual is shaped to sit beneath.
_Avoid_: Masking curve, audibility threshold, hearing threshold

**Transparent**:
Indistinguishable from the unprocessed signal in blind ABX at matched loudness. A claim that must be tested, never asserted from design intent.
_Avoid_: Clean, inaudible, artifact-free, lossless

**Artifact**:
An audible consequence of processing that is not the intended gain reduction. Distinct per mechanism: clipping produces harmonic distortion, while spectral processing produces temporal smearing and pre-echo.

### Measurement and judgement

**Reduction Depth**:
How much peak reduction is being applied, in dB. The project's transparency target is 3-6 dB, with graceful degradation beyond.
_Avoid_: Gain reduction (ambiguous with compression), GR, drive, amount

**Reference Set**:
The fixed collection of programme material on which every iteration is judged. Fixed so that results across iterations remain comparable.
_Avoid_: Test tracks, material, corpus

**Pass Gate**:
Blind ABX against the unprocessed signal at matched loudness. Determines whether the plugin is transparent.

**Worth-Using Gate**:
Blind ABX against KClip3 at matched reduction depth. Determines whether the plugin is better than what already exists.

### Structure

**Bin**:
One frequency component of an STFT frame.

**Stereo Link**:
The degree to which reduction applied to one channel is also applied to the other. Linked by default, because independent per-channel reduction shifts the stereo image.
_Avoid_: Channel coupling, stereo mode
