# Coherence and Asymmetry — Explanation Document

> Source: *Coherence and Asymmetry* (Elektrika, Inc. / Somatics, LLC)
> Purpose: Understanding and re-implementing the output values of Cymatron-family EEG analysis devices

---

## Table of Contents

1. [Overview](#1-overview)
2. [Foundational Concepts](#2-foundational-concepts)
3. [Coherence — Formula](#3-coherence--formula)
4. [Asymmetry — Formula](#4-asymmetry--formula)
5. [Calculation Procedure — A Close Reading of the Paper's Five Steps](#5-calculation-procedure--a-close-reading-of-the-papers-five-steps)
6. [The Critical Pitfall: Why Averaging Order Decides Everything](#6-the-critical-pitfall-why-averaging-order-decides-everything)
7. [Per-Band Analysis Done Right](#7-per-band-analysis-done-right)
8. [Six Implementation Pitfalls You Will Hit](#8-six-implementation-pitfalls-you-will-hit)
9. [Mapping to the Genie Bands Display](#9-mapping-to-the-genie-bands-display)
10. [Result Interpretation Guide](#10-result-interpretation-guide)
11. [Appendix: Glossary](#11-appendix-glossary)

---

## 1. Overview

This paper defines two indices for **quantifying the relationship between the left and right hemispheres of the brain on a per-frequency basis**, derived from two EEG channels (EEG1, EEG2).

| Index | What it measures | Context |
|---|---|---|
| **Coherence** (γ²) | Strength of **phase synchronization** between two channels | "Are the left and right hemispheres moving in the same rhythm?" |
| **Asymmetry** (A) | **Power (intensity) imbalance** between two channels | "Which hemisphere is more active?" |

The two indices carry **independent information**. Examples follow.

- **High Coherence × Low Asymmetry**: both sides moving in the same rhythm at the same intensity (ideal coordination)
- **High Coherence × High Asymmetry**: same rhythm but one side stronger (synchronized yet asymmetric activity)
- **Low Coherence × Low Asymmetry**: independent activity that happens to be of equal intensity
- **Low Coherence × High Asymmetry**: completely unrelated, with one side dominant

For this reason, displaying both indices appears to be standard practice in Cymatron-family devices.

---

## 2. Foundational Concepts

### 2.1 Signals and FFT

The FFT (Fast Fourier Transform) converts a time-domain signal into the frequency domain.

$$x(t) \;\xrightarrow{FFT}\; X(f) = \operatorname{Re}_x(f) + j\,\operatorname{Im}_x(f)$$

- \operatorname{Re}_x(f) : real part (cosine component)
- \operatorname{Im}_x(f) : imaginary part (sine component)
- j : imaginary unit (j^2 = -1)

### 2.2 Magnitude of a Complex Number

$$|\Re + j\Im|^2 = \Re^2 + \Im^2$$

This is what the paper means by "absolute value squared = real² + imag²" at its top.

### 2.3 Auto-power Spectrum (Sxx)

Represents how much energy a signal has at frequency f.

$$S_{xx}(f) = X^*(f) · X(f) = \operatorname{Re}_x^2(f) + \operatorname{Im}_x^2(f)$$

Here X* is the complex conjugate (real part unchanged, imaginary part sign-flipped).

> **Note:** The `Power = Sxx / 2` in the paper is a one-sided spectrum convention — the factor used when collapsing a two-sided spectrum into a one-sided form.

### 2.4 Cross-power Spectrum (Sxy)

Represents the **relationship between two signals at frequency f** as a complex number.

$$S_{xy}(f) = X^*(f) · Y(f)$$

Expanded into real and imaginary parts:

$$S_{xy}(f) = (\operatorname{Re}_x + j\operatorname{Im}_x)^* · (\operatorname{Re}_y + j\operatorname{Im}_y)$$

$$= (\operatorname{Re}_x - j\operatorname{Im}_x) · (\operatorname{Re}_y + j\operatorname{Im}_y)$$

$$= (\operatorname{Re}_x\operatorname{Re}_y + \operatorname{Im}_x\operatorname{Im}_y) + j(\operatorname{Re}_x\operatorname{Im}_y - \operatorname{Im}_x\operatorname{Re}_y)$$

> **Important:** The paper writes `Sxy = (Re_x + jIm_x) * (Re_y - jIm_y)`, which is the form X · Y*. This is the complex conjugate of X* · Y, but since the Coherence calculation squares the result, **either form yields the same final value**.

The squared magnitude of the cross-power spectrum:

$$|S_{xy}(f)|^2 = \operatorname{Re}_{xy}^2 + \operatorname{Im}_{xy}^2$$

This is the numerator of the Coherence formula.

---

## 3. Coherence — Formula

### 3.1 Definition

The paper's definition:

$$\boxed{\;\gamma^2(f) = \frac{|S_{xy}(f)|^2}{S_{xx}(f) · S_{yy}(f)}\;}$$

- Range: 0 ≤ γ² ≤ 1
- 0 = uncorrelated, 1 = fully linearly correlated (matching phase and amplitude)

### 3.2 What It Represents

Expresses **"how much the two signals move together at that frequency"** on a 0–1 scale.

- γ² = 1.0: at this frequency, one signal can fully predict the other
- γ² = 0.5: about half-correlated
- γ² = 0.0: fully independent at this frequency

### 3.3 A Mathematically Important Property

**When computed from a single segment, γ² is always exactly 1.0, at every frequency.**

This is a mathematical identity, not a bug — it follows directly from the definitions:

$$|S_{xy}|^2 = |X^* Y|^2 = |X|^2 · |Y|^2 = S_{xx} · S_{yy}$$

$$\therefore \gamma^2 = \frac{S_{xx} · S_{yy}}{S_{xx} · S_{yy}} = 1$$

**This property is precisely why the paper insists on averaging across multiple segments.**

To obtain a meaningful Coherence value, the **real and imaginary parts of the cross-spectrum must be averaged across multiple independent segments**, with the final calculation performed from those averages (see [Section 6](#6-the-critical-pitfall-why-averaging-order-decides-everything)).

---

## 4. Asymmetry — Formula

### 4.1 Definition

The paper's definition:

$$\boxed{\;A(f) = 2 · \frac{|S_{xx}(f) - S_{yy}(f)|}{S_{xx}(f) + S_{yy}(f)}\;}$$

- Range: 0 ≤ A ≤ 2 (the paper uses 0% – 200% notation)
- 0 = perfectly symmetric power
- 2 = energy entirely on one side

### 4.2 What It Represents

Expresses **"how much energy is biased toward one channel versus the other"**.

| Ratio of Sxx : Syy | A value | % notation |
|---|---|---|
| 1 : 1 (symmetric) | 0.000 | 0% |
| 2 : 1 | 0.667 | 67% |
| 3 : 1 | 1.000 | 100% |
| 9 : 1 | 1.600 | 160% |
| 1 : 0 (one side only) | 2.000 | 200% |

### 4.3 The Reason for the Factor of 2

Without the factor of 2, the maximum value would be 1.0. The paper's definition multiplies by 2 to give:

- 0% = perfectly symmetric
- 100% = moderate imbalance (3:1 power ratio)
- 200% = entirely one-sided

This places Asymmetry on a **scale comparable in digits to Coherence**. With Coherence in 0–1 (0%–100%) and Asymmetry in 0–2 (0%–200%), placing the two in percent form gives a similar visual scale — a deliberate convention to ease side-by-side comparison.

### 4.4 Coherence vs. Asymmetry — Recap

| | Coherence | Asymmetry |
|---|---|---|
| Input | Cross-spectrum Sxy | Auto-spectra only Sxx, Syy |
| Phase information | **Used** (complex) | **Not used** (real powers only) |
| What it measures | Strength of coupling | Magnitude imbalance |

---

## 5. Calculation Procedure — A Close Reading of the Paper's Five Steps

The paper's procedure, restated in modern terms with explanation of why each step matters.

### Step 1. Collect time signals x(t), y(t)

Capture two channels of time-series data — for EEG, from electrodes on the left and right hemispheres. The sampling frequency Fs depends on device settings (typical value: 200 Hz).

### Step 2. Transform to the frequency domain via FFT

> Paper text: *"FFT them into X(t) and Y(t)"*
> This is a typo in the original — it should be X(f), Y(f). FFT output is a function of frequency.

Apply a **window function** to each segment (typically 256 or 512 samples) before the FFT. The window tapers both ends smoothly toward 0 to suppress "spectral leakage" — an artifact arising because the FFT implicitly assumes the signal is periodic.

Common windows: Hann window = 0.5 · (1 - cos(2πi / N))
The original code's window = 1 - cos(2πi / N) (twice the Hann window, divided by a normalization factor of 1.5)

### Step 3. Critical: Average the cross-spectrum keeping real and imaginary parts separate

> Original: *"For coherence, you must use the real and imaginary parts of the cross power spectrum averaged over several segments, not the |Sxy(f)| averaged over those segments. Only after getting the average parts do you do the ℜ² + ℑ² calculation."*

This is the most important instruction in the entire paper.

**Correct order:**

$$\overline{S_{xy}}(f) = \frac{1}{M}\sum_{m=1}^{M} S_{xy}^{(m)}(f)$$

$$|\overline{S_{xy}}(f)|^2 = (\text{Re}\overline{S_{xy}})^2 + (\text{Im}\overline{S_{xy}})^2$$

(First average the complex values, then take the squared magnitude at the end.)

**Wrong order** (must not do this):

$$\frac{1}{M}\sum_{m=1}^{M} |S_{xy}^{(m)}(f)|^2$$

(Taking the magnitude of each segment first, then averaging.)

### Step 4. Average the auto-spectra over the same number of segments

$$\overline{S_{xx}}(f) = \frac{1}{M}\sum_{m=1}^{M} S_{xx}^{(m)}(f)$$

$$\overline{S_{yy}}(f) = \frac{1}{M}\sum_{m=1}^{M} S_{yy}^{(m)}(f)$$

### Step 5. Aggregate by band only at the very end

> Original: *"Note that if you want the coherence for bands, rather than a single bin, you must do the summation at the very end, because the sum of squares (the correct way) is not the same as the square of sums."*

When summarizing multiple frequency bins into a single band (e.g., Alpha 8–13 Hz):

**Correct:** compute the averaged cross- and auto-spectra at each bin, take the squared magnitudes per bin, then aggregate (sum or mean) within the band.

**Wrong:** sum the averaged cross-spectra across bins first, then square. Mathematically distinct from the correct approach (sum of squares vs. square of sums).

---

## 6. The Critical Pitfall: Why Averaging Order Decides Everything

### 6.1 Single-segment Coherence Is Always 1

As shown in [Section 3.3](#33-a-mathematically-important-property), Coherence computed from a single segment is **always 1.0**.

```
Single-segment S_xy = X*·Y
|S_xy|^2 = |X|^2 · |Y|^2 = S_xx · S_yy
γ² = S_xx · S_yy / (S_xx · S_yy) = 1
```

### 6.2 "Coherence of the Average" vs. "Average of the Coherence" — Different Things

A direct numerical comparison on the same data:

| Order | mean | max |
|---|---|---|
| **Wrong**: compute γ² per segment → average | 1.000 | 1.000 |
| **Correct**: accumulate raw cross-spectra → compute γ² at the end | 0.68 | 0.99 |

Inverting the order yields a meaningless **constant 1.0**.

### 6.3 Why the Correct Averaging Yields Meaningful Values

Averaging complex numbers is equivalent to vector averaging.

- **When two signals are genuinely synchronized**: each segment's Sxy points in roughly the same direction, so the average preserves direction and magnitude.
- **When two signals are independent**: the direction of Sxy varies randomly across segments. The vectors cancel out in the average, leaving magnitude near zero.

The auto-spectra Sxx, Syy are always positive real values, free of this directional issue, so they retain their value whether the signals are synchronized or not.

The result:

- Synchronized signals → |avg Sxy|^2 ≈ avg Sxx · avg Syy → γ² ≈ 1
- Independent signals → |avg Sxy|^2 ≈ 0 → γ² ≈ 0

This is what is considered the essence of the **Welch method** — segment averaging that keeps phase information alive.

### 6.4 Welch Method Segment Design

| Parameter | Typical value | Effect |
|---|---|---|
| FFT length n | 256, 512, 1024 | Frequency resolution = Fs / n |
| Hop length | n/2 (50% overlap) | Trade-off between segment independence and noise reduction |
| Number of segments M | 10 – 100 | More segments = better statistical stability (but requires longer data) |

Using overlap increases the effective segment count and reduces noise at the cost of more computation. 50% overlap is the traditional optimum.

---

## 7. Per-Band Analysis Done Right

In EEG analysis, summarizing by physiologically meaningful **frequency bands** is more useful than discussing individual 1 Hz steps.

### 7.1 Standard Band Definitions

| Band | Range | Physiological meaning |
|---|---|---|
| Delta | 0.5 – 4 Hz | Deep sleep, unconsciousness |
| Theta | 4 – 8 Hz | Light sleep, meditation, creativity |
| Alpha | 8 – 13 Hz | Relaxed wakefulness |
| Beta | 13 – 30 Hz | Active wakefulness, focus, thought |
| Gamma | 30 Hz + | Higher cognition, attention |

Cymatron Genie band definitions (confirmed from Image 2):

| Band | Range (Hz) |
|---|---|
| Total | 0.7 – 25.0 |
| Delta | 0.7 – 3.5 |
| Theta | 3.5 – 8.0 |
| Alpha | 8.0 – 13.0 |
| Beta | 13.0 – 25.0 |
| X Band | 4.0 – 7.5 |
| Y Band | 8.5 – 12.0 |

X Band and Y Band are Cymatron-specific sub-bands centered on the Theta and Alpha mid-ranges.

### 7.2 Per-band Coherence Aggregation

Following Step 5 of the paper, compute per bin then aggregate within the band.

```
Pseudocode:
for each bin k in band [Lo, Hi]:
    coh[k] = (avgRe[k]² + avgIm[k]²) / (avgSxx[k] · avgSyy[k])

band_max  = max(coh[k] for k in band)
band_mean = mean(coh[k] for k in band)
```

The official app's `Coh%` column displays either the in-band maximum or in-band mean. The implementation in this project outputs both.

### 7.3 Mean Frequency

The "centroid frequency" within a band can also be computed:

$$f_{\text{mean}} = \frac{\sum_k f_k · \overline{S_{xx}}(f_k)}{\sum_k \overline{S_{xx}}(f_k)}$$

This is the value displayed in the official app's `Mean Frequencies Hz:` row.

---

## 8. Six Implementation Pitfalls You Will Hit

A list of typical bugs encountered during actual implementation of this paper.

### Pitfall ① Computing Coherence per segment

**Symptom**: Result is always 1.000 (100%).

**Cause**: The mathematical identity from [Section 6.1](#61-single-segment-coherence-is-always-1).

**Fix**: Use an accumulator pattern.

```csharp
// NG: Wrong
foreach (segment) {
    coh = ComputeCoherence(segment);
    cohSum += coh;
}
result = cohSum / segCount;

// OK: Correct
foreach (segment) {
    Accumulate(Sxx, Syy, Sxy_re, Sxy_im);  // sum raw spectra
}
result = ComputeCoherenceFromAccumulated();  // compute once at the end
```

### Pitfall ② FFT twiddle table sign bug

**Symptom**: Feeding a pure sinusoid to the FFT spreads energy across many bins instead of producing a single peak at the expected bin.

**Cause**: The cosine table holds only a half period, and modular indexing fails to flip the sign in the second half period.

```csharp
// NG Half-period table + % (MAXSAMP/2)
private int[] cosTbl = new int[MAXSAMP / 2];
int wi = cosTbl[(MAXSAMP/4 + k*delta) % (MAXSAMP/2)];  // sign info lost

// OK Full-period table + % MAXSAMP
private int[] cosTbl = new int[MAXSAMP];
int wi = cosTbl[(MAXSAMP/4 + k*delta) % MAXSAMP];
```

A separate cos and sin table is recommended.

### Pitfall ③ Missing the Nyquist bin

**Symptom**: Power at the highest frequency (Fs/2) is always 0.

**Cause**: Loop terminates at `i < n/2`, so the bin at `i = n/2` is never computed.

```csharp
// NG
for (int i = 0; i < n/2; i++) { ... }

// OK
for (int i = 0; i <= n/2; i++) { ... }  // inclusive upper bound
```

### Pitfall ④ Integer FFT overflow

**Symptom**: FFT output values are abnormally small, and Coherence collapses to ~0.

**Cause**: `int * int` exceeds the 32-bit range and wraps around, destroying the value.

```csharp
// NG int * int overflows
int xr = (real[l] * wr - imag[l] * wi) / ZOOM;

// OK Compute via long
long mr = (long)real[l] * wr - (long)imag[l] * wi;
int xr = (int)(mr / ZOOM);
```

The safest fix: use `double` from the start.

### Pitfall ⑤ Wrong sampling frequency

**Symptom**: Bin indices for band boundaries (e.g., 8–13 Hz) drift, aggregating the wrong frequencies.

**Cause**: Hardcoded Fs that disagrees with the device setting.

**Verification**: Inspect the official app's spectrum display and read off the bin width. Bin width × n should equal Fs.

```
Example: bin width 0.78 Hz, n=256 → Fs = 0.78 × 256 ≈ 200 Hz
```

### Pitfall ⑥ Excessive smoothing

**Symptom**: The calculation logic is correct but peak values are lower than expected.

**Cause**: Post-processing such as a 3-point moving average flattens sharp resonance peaks.

```csharp
// NG The longer the FFT, the more smoothing damages the peaks
cohSmooth[k] = (coh[k-1] + coh[k] + coh[k+1]) / 3.0;

// OK With a long enough FFT, no smoothing is needed
cohSmooth = coh;  // pass through
```

A holdover from the integer-arithmetic era when quantization noise was severe. With double precision and an adequate FFT length, smoothing is unnecessary.

---

## 9. Mapping to the Genie Bands Display

The Cymatron Genie's `Bands EEG 1&2 (accumulated)` screen displays the indices of this paper in the following columns.

```
Frequency Hz   Absolute μV²    Relative %    Asym %    Coh %
              EEG1   EEG2     EEG1   EEG2
Total  0.7-25.0  ...    ...    ...    ...    ...      ...
Delta  0.7- 3.5  ...    ...    ...    ...    ...      ...
Theta  3.5- 8.0  ...    ...    ...    ...    ...      ...
Alpha  8.0-13.0  ...    ...    ...    ...    ...      ...
Beta  13.0-25.0  ...    ...    ...    ...    ...      ...
X Band 4.0- 7.5  ...    ...    ...    ...    ...      ...
Y Band 8.5-12.0  ...    ...    ...    ...    ...      ...
Mean Frequencies Hz:    ...    ...
Segments: Collected-NN   Computed-NN
```

Per-column formulas:

| Column | Formula |
|---|---|
| **Absolute μV²** | sum over band avg Sxx(f_k) (and Syy) |
| **Relative %** | (band power / Total band power) × 100 |
| **Asym %** | In-band mean or max of A(f) × 100 |
| **Coh %** | In-band mean or max of γ²(f) × 100 |
| **Mean Frequencies Hz** | Σ f_k · avg Sxx(f_k) / Σ avg Sxx(f_k) |
| **Collected** | Number of segments fed into the FFT |
| **Computed** | Number of segments effectively used after artifact rejection |

To match `Computed = Collected − segments overlapping ExcludeRange`, the implementation must skip excluded ranges inside the Accumulate loop, driven by the raw data's `ExcludeRange` field.

---

## 10. Result Interpretation Guide

### 10.1 Typical Values in Healthy Resting Wakefulness

| Item | Expected value |
|---|---|
| Alpha Coh % | 70–95% (high synchronization) |
| Alpha Asym % | 0–30% (low asymmetry) |
| Alpha Relative % | 30–60% (Alpha-dominant during wakefulness) |
| Mean Frequency Hz | 9–11 (Alpha centroid) |

### 10.2 Patterns That Warrant Attention

**Delta abnormally large (Relative > 50%)**
- Deep sleep state, OR
- DC drift / artifact contamination, OR
- High-pass filter not applied

**Coh % low across all bands (mean < 30%)**
- Poor electrode contact
- Signal quality degradation on one side
- Genuinely independent left/right activity

**Asym % spike in a specific band only**
- The source generator for that band is lateralized
- Single-side electrode artifact

**MSC@Final outside the Alpha band**
- Alpha-band coherence is weaker than other bands → possible reduced wakefulness or attentional issues

### 10.3 Coherence × Asymmetry Interpretation Matrix

|  | Low Asym (< 30%) | Medium Asym (30–70%) | High Asym (> 70%) |
|---|---|---|---|
| **High Coh (> 70%)** | Ideal bilateral coordination | Synchronized but lateralized activity | Strong synchronized one-sided activity |
| **Medium Coh (30–70%)** | Partial coordination, healthy range | Typical everyday state | Partially synchronized one-sided dominance |
| **Low Coh (< 30%)** | Independent and symmetric | Independent with mild bias | Strong one-sided activity |

---

## 11. Appendix: Glossary

| Term | Definition |
|---|---|
| **FFT** | Fast Fourier Transform. Algorithm decomposing a time signal into frequency components. |
| **bin** | One frequency point in the FFT output. Bin width = Fs/n. |
| **Nyquist frequency** | Fs/2. Frequencies above this cannot be reproduced by sampling theory. |
| **Welch method** | A method for stable spectral estimation that splits a signal into multiple segments, FFTs each, and averages the results. |
| **MSC** | Magnitude Squared Coherence. Refers to γ²(f). |
| **window function** | A weighting function applied to a signal before FFT, tapering the ends toward 0 to suppress leakage. Examples: Hann, Hamming, Blackman. |
| **leakage** | Energy spread away from the true frequency, caused by the FFT's implicit periodic-signal assumption. |
| **Auto-spectrum** | Power spectrum of a single signal (Sxx, Syy). Always real and positive. |
| **Cross-spectrum** | Spectrum between two signals (Sxy). Complex-valued, carrying phase information. |
| **overlap** | Partial reuse of samples between segments in the Welch method. 50% is standard. |
| **artifact** | Non-brain signal contamination — blinks, body movement, EMG, AC mains, etc. |
| **high-pass filter** | A filter cutting low frequencies (such as DC drift). For EEG, a cutoff around 0.5 Hz is common. |

---

## Closing Notes

Despite its brevity, this paper is regarded as a sound document distilling the essence of the Welch method — preserving phase information through statistical averaging — and applying it to two-channel analysis as Coherence and Asymmetry.

Implementation pitfalls, however, demand prior knowledge from the reader, which this document supplements. When implementing, work through the pitfall list in [Section 8](#8-six-implementation-pitfalls-you-will-hit) at minimum.

---

*Document version: 1.0*
