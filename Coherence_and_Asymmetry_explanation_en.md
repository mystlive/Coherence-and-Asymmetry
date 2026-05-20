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

| Index              | What it measures                                           | Context                                                         |
| ------------------ | ---------------------------------------------------------- | --------------------------------------------------------------- |
| **Coherence** (γ²) | Strength of **phase synchronization** between two channels | "Are the left and right hemispheres moving in the same rhythm?" |
| **Asymmetry** (A)  | **Power (intensity) imbalance** between two channels       | "Which hemisphere is more active?"                              |

The two indices carry **independent information**. Examples follow.

- **High Coherence × Low Asymmetry**: both sides moving in the same rhythm at the same intensity (ideal coordination)
- **High Coherence × High Asymmetry**: same rhythm but one side stronger (synchronized yet asymmetric activity)
- **Low Coherence × Low Asymmetry**: independent activity that happens to be of equal intensity
- **Low Coherence × High Asymmetry**: completely unrelated, with one side dominant

For this reason, displaying both indices is standard practice in Cymatron-family devices.

---

## 2. Foundational Concepts

### 2.1 Signals and FFT

The FFT (Fast Fourier Transform) converts a time-domain signal into the frequency domain.

$$x(t) \xrightarrow{FFT} X(f) = \operatorname{Re}_x(f) + j\,\operatorname{Im}_x(f)$$

- $\operatorname{Re}_x(f)$ : real part (cosine component)
- $\operatorname{Im}_x(f)$ : imaginary part (sine component)
- $j$ : imaginary unit ($j^2 = -1$)

### 2.2 Magnitude of a Complex Number

$$|\Re + j\Im|^2 = \Re^2 + \Im^2$$

This is what the paper means by "absolute value squared = real² + imag²" at its top.

> **Note on the paper's notation:** The original paper reads `|ℜ + jℑ| = ℜ² + ℑ²`, but this appears to be a typo with the square missing on the left-hand side (the equation does not hold if the LHS is the magnitude itself while the RHS is the sum of squares). The correct form is `|ℜ + jℑ|² = ℜ² + ℑ²`. All subsequent equations in the paper (Sxx, |Sxy|², etc.) use the squared form consistently, so there is no internal contradiction.

### 2.3 Auto-power Spectrum (Sxx)

Represents how much energy a signal has at frequency f.

$$S_{xx}(f) = X^*(f) \cdot X(f) = \operatorname{Re}_x^2(f) + \operatorname{Im}_x^2(f)$$

Here $X^*$ is the complex conjugate (real part unchanged, imaginary part sign-flipped).

> **Memo:** The `Power = Sxx / 2` in the paper is the factor that converts squared amplitude |X(f)|² = ℜ² + ℑ² into physical mean power (the time-domain RMS² equivalent). It derives from the fact that a sinusoid A·cos(2πf₀t) has mean power A²/2. This is **distinct** from one-sided spectrum conversion (which multiplies non-DC, non-Nyquist bins by 2) — do not confuse the two. Note that for Coherence and Asymmetry calculations, this factor cancels between numerator and denominator, so applying /2 or not does not affect the result.

### 2.4 Cross-power Spectrum (Sxy)

Represents the **relationship between two signals at frequency f** as a complex number.

$$S_{xy}(f) = X^*(f) \cdot Y(f)$$

Expanded into real and imaginary parts:

$$S_{xy}(f) = (\operatorname{Re}_x - j\operatorname{Im}_x) \cdot (\operatorname{Re}_y + j\operatorname{Im}_y)$$

$$= (\operatorname{Re}_x\operatorname{Re}_y + \operatorname{Im}_x\operatorname{Im}_y) + j(\operatorname{Re}_x\operatorname{Im}_y - \operatorname{Im}_x\operatorname{Re}_y)$$

> **Important: Convention difference**  
> The paper writes `Sxy = X · Y*` (Y conjugated), but this document follows the convention prevalent in academic literature (Bendat & Piersol, etc.): `Sxy = X* · Y`. The two are complex conjugates of each other:
> - `|Sxy|²` is **identical** in both conventions → **Coherence values are the same**
> - The cross-phase `arg(Sxy)` has its **sign flipped** → the interpretation of phase lead/lag is reversed
>
> For Coherence and Asymmetry calculations in this paper, either convention yields the same result. However, if you reuse the cross-spectrum's phase information for other purposes (e.g., time-delay estimation), clearly document which convention your implementation uses.

The squared magnitude of the cross-power spectrum:

$$|S_{xy}(f)|^2 = \operatorname{Re}_{xy}^2 + \operatorname{Im}_{xy}^2$$

This is the numerator of the Coherence formula.

---

## 3. Coherence — Formula

### 3.1 Definition

The paper's definition:

$$\boxed{\;\gamma^2(f) = \frac{|S_{xy}(f)|^2}{S_{xx}(f) \cdot S_{yy}(f)}\;}$$

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

$$|S_{xy}|^2 = |X^* Y|^2 = |X|^2 \cdot |Y|^2 = S_{xx} \cdot S_{yy}$$

$$\therefore \gamma^2 = \frac{S_{xx} \cdot S_{yy}}{S_{xx} \cdot S_{yy}} = 1$$

**This property is precisely why the paper insists on averaging across multiple segments.**

To obtain a meaningful Coherence value, the **real and imaginary parts of the cross-spectrum must be averaged across multiple independent segments**, with the final calculation performed from those averages (see [Section 6](#6-the-critical-pitfall-why-averaging-order-decides-everything)).

---

## 4. Asymmetry — Formula

### 4.1 Definition

The paper's definition:

$$\boxed{\;A(f) = 2 \cdot \frac{|S_{xx}(f) - S_{yy}(f)|}{S_{xx}(f) + S_{yy}(f)}\;}$$

- Range: 0 ≤ A ≤ 2 (the paper uses 0% – 200% notation)
- 0 = perfectly symmetric power
- 2 = energy entirely on one side

### 4.2 What It Represents

Expresses **"how much energy is biased toward one channel versus the other"**.

| Ratio of Sxx : Syy    | A value | % notation |
| --------------------- | ------- | ---------- |
| 1 : 1 (symmetric)     | 0.000   | 0%         |
| 2 : 1                 | 0.667   | 67%        |
| 3 : 1                 | 1.000   | 100%       |
| 9 : 1                 | 1.600   | 160%       |
| 1 : 0 (one side only) | 2.000   | 200%       |

### 4.3 The Reason for the Factor of 2

Without the factor of 2, the range of `|Sxx − Syy|/(Sxx + Syy)` is 0–1 (perfectly symmetric to completely one-sided). The paper multiplies by 2 to extend the range to 0–2 (0%–200%), but the rationale for this factor is **not stated in the paper**. Possible reasons:

- **Intuitive thresholds**: with the factor of 2, a "3:1 power ratio" lands exactly at 1.0 (100%), giving the readable rule "above 100% = strong imbalance"
- **Improved visibility of small asymmetries**: a 0–2 range has wider dynamic range than 0–1, making graph display and threshold detection easier

Note that since Coherence (0–1) and Asymmetry (0–2) have different ranges, placing them side-by-side in percentage form does **not** mean they share the same scale.

### 4.4 Coherence vs. Asymmetry — Recap

|                   | Coherence            | Asymmetry                       |
| ----------------- | -------------------- | ------------------------------- |
| Input             | Cross-spectrum Sxy   | Auto-spectra only Sxx, Syy      |
| Phase information | **Used** (complex)   | **Not used** (real powers only) |
| What it measures  | Strength of coupling | Magnitude imbalance             |

---

## 5. Calculation Procedure — A Close Reading of the Paper's Five Steps

The paper's procedure, restated in modern terms with explanation of why each step matters.

### Step 1. Collect time signals x(t), y(t)

Capture two channels of time-series data — for EEG, from electrodes on the left and right hemispheres.

The sampling frequency Fs is configurable on Cymatron-family devices, but **the manufacturer (Elektrika Inc.) and standard practice use Fs = 200 Hz**. All subsequent descriptions in this document (FFT length, bin width, band bin ranges, etc.) assume **Fs = 200 Hz**.

> **Note:** If you operate the device at a different Fs, the following parameters scale accordingly:
> - Bin width = (Fs / 2) / 128 (divide by FFT length 128 after decimation)
> - Bin numbers corresponding to each band
> - Effective Nyquist frequency = Fs / 4 (Nyquist after the 1/2 decimation)
>
> Always verify the Fs setting when reconciling device output with custom calculations.

### Step 2. Transform to the frequency domain via FFT

> Paper text: *"FFT them into X(t) and Y(t)"*  
> This is a typo in the original — it should be X(f), Y(f).

The FFT (Fast Fourier Transform) converts a time-domain signal into the frequency domain. Split the signal into fixed-length "segments," apply a window function to each, then perform the FFT.

The paper does not specify segment length or window function, but Cymatron-family devices use the following implementation:

**Cymatron-family Device Specifications** (confirmed by Elektrika Inc.):

| Item | Value |
|---|---|
| Sampling frequency Fs | 200 Hz (standard operation) |
| Samples per segment | 256 (1.28 seconds) |
| Decimation | 256 → 128 samples (1/2 decimation) |
| FFT length | 128 points |
| Bin width | 0.78 Hz (= 100 / 128) |
| Used bins | 1–32 (DC bin 0 excluded, approx. 0.78–25 Hz) |
| Window function | `1 − cos(2πn/127)`, n = 0..127 |
| Overlap | None |
| Zero-padding | None |
| Normalization | None |
| Processing order | DC removal → Decimation → Cosine window → FFT |

The window function `1 − cos(2πn/127)` is the standard Hann window `0.5·(1 − cos(2πn/(N−1)))` multiplied by 2 in amplitude (N = 128; the denominator 127 matches the conventional Hann definition). The window is applied for leakage suppression, but according to the manufacturer, no normalization is performed.

**Note on decimation**

After acquiring 256 samples (1.28 seconds at 200 Hz), the signal is decimated to 128 samples (1/2 downsampling) before FFT. This yields:

- Effective Nyquist frequency = 50 Hz (200 Hz / 2 / 2)
- Bin width = 0.78 Hz (≈ 100 Hz / 128)
- Bins 1–32 cover approximately 0.78–25 Hz (consistent with the Total band 0.7–25 Hz)

> **Note:** Whether the decimation is simple downsampling or includes an anti-aliasing filter has not been explicitly confirmed by the manufacturer; this requires inspection of the implementation source. If it is simple downsampling, components above 25 Hz may alias.

### Step 3. Critical: Average the cross-spectrum keeping real and imaginary parts separate

> Original: *"For coherence, you must use the real and imaginary parts of the cross power spectrum averaged over several segments, not the |Sxy(f)| averaged over those segments. Only after getting the average parts do you do the ℜ² + ℑ² calculation."*

This is the most important instruction in the entire paper.

**Correct order:**

$$\overline{S_{xy}}(f) = \frac{1}{M}\sum_{m=1}^{M} S_{xy}^{(m)}(f)$$

$$|\overline{S_{xy}}(f)|^2 = (\operatorname{Re}\overline{S_{xy}})^2 + (\operatorname{Im}\overline{S_{xy}})^2$$

(First average the complex values, then take the squared magnitude at the end.)

**Wrong order** (must not do this):

$$\frac{1}{M}\sum_{m=1}^{M} |S_{xy}^{(m)}(f)|^2$$

(Taking the magnitude of each segment first, then averaging.)

### Step 4. Average the auto-spectra over the same number of segments

$$\overline{S_{xx}}(f) = \frac{1}{M}\sum_{m=1}^{M} S_{xx}^{(m)}(f)$$

$$\overline{S_{yy}}(f) = \frac{1}{M}\sum_{m=1}^{M} S_{yy}^{(m)}(f)$$

### Step 5. Aggregate by band only at the very end

> Original: *"Note that if you want the coherence for bands, rather than a single bin, you must do the summation at the very end, because the sum of squares (the correct way) is not the same as the square of sums (which is what you'll get if you sum up bins beforehand)."*

When computing Coherence over a band (e.g., Alpha 8–13 Hz), the paper instructs "do the summation at the very end." The "sum of squares ≠ square of sums" comparison refers to the processing order of the real and imaginary parts of the averaged cross-spectrum `S̄xy(k) = S̄xy_re(k) + j·S̄xy_im(k)`.

**Correct order (sum of squares)** — take squared magnitudes per bin, then aggregate within the band:

$$\sum_{k \in \text{band}} \bigl[\, \overline{S_{xy,\text{re}}}(k)^2 + \overline{S_{xy,\text{im}}}(k)^2 \,\bigr]$$

**Wrong order (square of sums)** — sum real and imaginary parts across bins first, then square:

$$\Bigl[\,\sum_{k \in \text{band}} \overline{S_{xy,\text{re}}}(k)\,\Bigr]^2 + \Bigl[\,\sum_{k \in \text{band}} \overline{S_{xy,\text{im}}}(k)\,\Bigr]^2$$

These yield mathematically different results. Bins with random phase tend to cancel each other in the sum, so the wrong order leads to an unduly small numerator, producing artificially low band Coherence values.

There are multiple conventions for how to "aggregate" band Coherence (e.g., compute γ²(k) per bin then take the in-band mean or max). The paper does not prescribe a specific aggregation method, but **the principle of "do not sum real/imaginary parts across bins before squaring"** is common to all approaches. See Section 7.2 for specific aggregation methods.

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

| Order | Result |
|---|---|
| **Wrong**: compute γ² per segment → average | Always 1.000 (mathematical identity, see 6.1) |
| **Correct**: accumulate raw cross-spectra → compute γ² at the end | Value depends on signal synchrony (0–1) |

Inverting the order yields a meaningless **constant 1.0**.

> **Note:** The actual value on the "correct" side depends on signal synchrony, noise level, and segment count. It approaches 1 for perfectly synchronized signals and 0 for independent noise sources. See Section 6.3 for details.

### 6.3 Why the Correct Averaging Yields Meaningful Values

Averaging complex numbers is equivalent to vector averaging.

- **When two signals are genuinely synchronized**: each segment's Sxy points in roughly the same direction, so the average preserves direction and magnitude.
- **When two signals are independent**: the direction of Sxy varies randomly across segments. The vectors cancel out in the average, leaving magnitude near zero.

The auto-spectra Sxx, Syy are always positive real values, free of this directional issue, so they retain their value whether the signals are synchronized or not.

The result:

- Synchronized signals → |avg Sxy|² ≈ avg Sxx · avg Syy → γ² ≈ 1
- Independent signals → |avg Sxy|² ≈ 0 → γ² ≈ 0

This is the essence of the **Welch method** — segment averaging that keeps phase information alive.

### 6.4 Welch Method Segment Design

**General Welch method parameters**

| Parameter            | Typical value     | Effect                                                                  |
| -------------------- | ----------------- | ----------------------------------------------------------------------- |
| FFT length n         | 256, 512, 1024    | Frequency resolution = Fs / n                                           |
| Hop length           | n/2 (50% overlap) | Trade-off between segment independence and noise reduction              |
| Number of segments M | 10 – 100          | More segments = better statistical stability (but requires longer data) |

In general practice, 50% overlap with many segments is the traditional approach to balance statistical stability with noise reduction.

**Cymatron-family device implementation**

However, Cymatron-family devices adopt a different design (confirmed by Elektrika Inc.):

| Item | Device specification | Difference from general practice |
|---|---|---|
| FFT length | 128 points (256 acquired → 128 after decimation) | Shorter |
| Overlap | None | No 50% overlap |
| MSC segment count | Most recent 8 segments (fixed) | Fixed, smaller count |

The choice not to use overlap likely reflects a design priority for "sustained" metrics like MSC: each segment should represent an independent time interval. Including temporally overlapping segments would blur the meaning of "sustained."

---

## 7. Per-Band Analysis Done Right

In EEG analysis, summarizing by physiologically meaningful **frequency bands** is more useful than discussing individual 1 Hz steps.

### 7.1 Standard Band Definitions

| Band  | Range      | Physiological meaning               |
| ----- | ---------- | ----------------------------------- |
| Delta | 0.5 – 4 Hz | Deep sleep, unconsciousness         |
| Theta | 4 – 8 Hz   | Light sleep, meditation, creativity |
| Alpha | 8 – 13 Hz  | Relaxed wakefulness                 |
| Beta  | 13 – 30 Hz | Active wakefulness, focus, thought  |
| Gamma | 30 Hz +    | Higher cognition, attention         |

Cymatron Genie band definitions:

| Band   | Range (Hz)  |
| ------ | ----------- |
| Total  | 0.7 – 25.0  |
| Delta  | 0.7 – 3.5   |
| Theta  | 3.5 – 8.0   |
| Alpha  | 8.0 – 13.0  |
| Beta   | 13.0 – 25.0 |
| X Band | 4.0 – 7.5   |
| Y Band | 8.5 – 12.0  |

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

$$f_{\text{mean}} = \frac{\sum_k f_k \cdot \overline{S_{xx}}(f_k)}{\sum_k \overline{S_{xx}}(f_k)}$$

This is the value displayed in the official app's `Mean Frequencies Hz:` row.

### 7.4 Maximum Sustained Coherence (MSC)

**Definition (Cymatron device manual)**

> "The maximum average coherence value in a 5-second segment during the seizure period."

Note: the MSC here is **Maximum Sustained Coherence**, which differs in meaning from the general signal-processing term MSC (Magnitude Squared Coherence = γ²(f)).

**Device implementation specification** (confirmed by Elektrika Inc.)

1. The device maintains the **most recent 8 segments** of post-treatment data
2. MSC is computed only when both conditions are met:
   - A seizure is detected
   - At least 8 segments of post-treatment data have been collected
3. Calculation procedure:
   - Average Sxx and Syy across the 8 segments to obtain mean auto-power
   - Average the real and imaginary parts of cross-power Sxy across the 8 segments separately, then compute |Sxy|² at the end
   - Compute γ²(f) from the above (consistent with the paper's Step 3 principle)

**On the "5 seconds" (not yet confirmed)**

The relationship between the manual's "5-second segment" and the internal implementation's "8 segments" (1.28 s × 8 = 10.24 s) **has not been clarified** from the available documentation or manufacturer responses. Considerations:

- At 200 Hz sampling, 5 seconds = 1000 samples → not an integer multiple of 1.28 s (256 samples)
- 1.28 s × 4 segments = 5.12 s would align naturally, but the manual states "5 seconds"
- How "most recent 8 segments" connects to "5-second segment" requires further clarification

Keep this uncertainty in mind when implementing, and whenever possible, cross-check device output against your own calculations.

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

Separate cos and sin tables are recommended.

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

### Pitfall ⑤ Wrong sampling frequency or decimation

**Symptom**: Bin indices for band boundaries (e.g., 8–13 Hz) drift, aggregating the wrong frequencies.

**Cause**: Incorrect Fs value, or missing/extra decimation before FFT.

**Correct relationships for Cymatron-family devices**:

```
Acquisition sampling rate Fs       = 200 Hz
Samples per segment (acquired)     = 256
                ↓ 1/2 decimation
FFT input samples                  = 128
Effective sampling rate            = 100 Hz
Bin width                          = 100 / 128 ≈ 0.78 Hz
Effective Nyquist frequency        = 50 Hz
Used bins                          = 1–32 (DC excluded, approx. 0.78–25 Hz)
```

**Verification**: Inspect the official app's spectrum display, read off the bin width, and check it matches the relationships above.

Note that if you skip decimation and run a 256-point FFT, the bin width is also approximately 0.78 Hz (= 200/256), so the band bin range appears to match at first glance. However, the following differences will cause discrepancies with the device:

- Effective Nyquist frequency (100 Hz without decimation vs 50 Hz with decimation)
- Treatment of components above 25 Hz, depending on whether an anti-aliasing filter is present
- Number of samples to which the window function is applied (128 vs 256)

To reconcile your output numerically with the device, you must match the device's data-point count and processing pipeline exactly.

### Pitfall ⑥ Excessive smoothing (general note, not from paper or device spec)

> **Note**: This item is not prescribed by the paper or the manufacturer; it is a general implementation heuristic. Unlike Pitfalls ①–⑤, it does not constitute a violation of the paper's instructions.

**Symptom**: The calculation logic is correct but peak values are lower than expected.

**Cause**: Post-processing such as a 3-point moving average may be flattening sharp resonance peaks.

```csharp
// Example: smoothing by averaging adjacent bins
cohSmooth[k] = (coh[k-1] + coh[k] + coh[k+1]) / 3.0;
```

The Cymatron-family device fixes the FFT length at 128, giving a bin width of 0.78 Hz, which is fine enough for EEG analysis that additional smoothing is usually unnecessary. When reconciling your output against the device, it is unclear whether the device performs smoothing internally; the recommended approach is to implement **without smoothing** first and check whether the results match.

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

| Column                  | Formula                                                      |
| ----------------------- | ------------------------------------------------------------ |
| **Absolute μV²**        | sum over band avg Sxx(f\_k) (and Syy)                        |
| **Relative %**          | (band power / Total band power) × 100                        |
| **Asym %**              | In-band mean or max of A(f) × 100                            |
| **Coh %**               | In-band mean or max of γ²(f) × 100                           |
| **Mean Frequencies Hz** | Σ f\_k · avg Sxx(f\_k) / Σ avg Sxx(f\_k)                     |
| **Collected**           | Number of segments fed into the FFT                          |
| **Computed**            | Number of segments effectively used after exclusion          |

> **Note on the Collected vs Computed difference**  
> When the values in `Segments: Collected-NN Computed-NN` differ on the Bands display, the difference represents segments excluded from analysis for some reason (artifact rejection, trimming of waste regions during attachment/removal, etc.). The specific exclusion logic varies by device and implementation, and is outside the scope of this paper.

---

## 10. Result Interpretation Guide

### 10.1 Typical Values in Healthy Resting Wakefulness

| Item              | Expected value                             |
| ----------------- | ------------------------------------------ |
| Alpha Coh %       | 70–95% (high synchronization)              |
| Alpha Asym %      | 0–30% (low asymmetry)                      |
| Alpha Relative %  | 30–60% (Alpha-dominant during wakefulness) |
| Mean Frequency Hz | 9–11 (Alpha centroid)                      |

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

### 10.3 Coherence × Asymmetry Interpretation Matrix

|                         | Low Asym (< 30%)                    | Medium Asym (30–70%)                  | High Asym (> 70%)                          |
| ----------------------- | ----------------------------------- | ------------------------------------- | ------------------------------------------ |
| **High Coh (> 70%)**    | Ideal bilateral coordination        | Synchronized but lateralized activity | Strong synchronized one-sided activity     |
| **Medium Coh (30–70%)** | Partial coordination, healthy range | Typical everyday state                | Partially synchronized one-sided dominance |
| **Low Coh (< 30%)**     | Independent and symmetric           | Independent with mild bias            | Strong one-sided activity                  |

---

## 11. Appendix: Glossary

| Term                  | Definition                                                                                                                              |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **FFT**               | Fast Fourier Transform. Algorithm decomposing a time signal into frequency components.                                                  |
| **bin**               | One frequency point in the FFT output. Bin width = Fs/n.                                                                                |
| **Nyquist frequency** | Fs/2. Frequencies above this cannot be reproduced by sampling theory.                                                                   |
| **Welch method**      | A method for stable spectral estimation that splits a signal into multiple segments, FFTs each, and averages the results.               |
| **MSC**               | **Maximum Sustained Coherence**. An index used by Cymatron-family devices: the maximum sustained coherence value during a seizure period. Note that this differs in meaning from the general signal-processing usage of MSC = Magnitude Squared Coherence (an alias for γ²(f)). See Section 7.4 for details. |
| **window function**   | A weighting function applied to a signal before FFT, tapering the ends toward 0 to suppress leakage. Examples: Hann, Hamming, Blackman. |
| **leakage**           | Energy spread away from the true frequency, caused by the FFT's implicit periodic-signal assumption.                                    |
| **Auto-spectrum**     | Power spectrum of a single signal (Sxx, Syy). Always real and positive.                                                                 |
| **Cross-spectrum**    | Spectrum between two signals (Sxy). Complex-valued, carrying phase information.                                                         |
| **overlap**           | Partial reuse of samples between segments in the Welch method. 50% is standard.                                                         |
| **artifact**          | Non-brain signal contamination — blinks, body movement, EMG, AC mains, etc.                                                             |
| **high-pass filter**  | A filter cutting low frequencies (such as DC drift). For EEG, a cutoff around 0.5 Hz is common.                                         |

---

## Closing Notes

Despite its brevity, this paper is a sound document distilling the essence of the Welch method — preserving phase information through statistical averaging — and applying it to two-channel analysis as Coherence and Asymmetry.

Implementation pitfalls, however, demand prior knowledge from the reader, which this document supplements. When implementing, work through the pitfall list in [Section 8](#8-six-implementation-pitfalls-you-will-hit) at minimum.

---

*Document version: 2.0*
