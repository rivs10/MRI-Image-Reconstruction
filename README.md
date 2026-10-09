# MRI  Partial Fourier Reconstruction

## Overview

This project investigates **Partial Fourier (PF) reconstruction in Magnetic Resonance Imaging (MRI)** by comparing three reconstruction approaches: Zero-Filled Reconstruction, Homodyne Reconstruction, and Projection onto Convex Sets (POCS).

Partial Fourier imaging reduces MRI acquisition time by collecting only a fraction of the required k-space data. Since the missing k-space samples cannot simply be recovered from acquired data without additional assumptions, reconstruction methods are used to estimate the missing information and recover image quality.

The project uses a **simulated multi-coil MRI phantom generated using BART (Berkeley Advanced Reconstruction Toolbox)** to create controlled, reproducible experiments with known fully sampled reference data.

## Objectives

* Understand the effect of Partial Fourier undersampling on MRI reconstruction.
* Simulate 75% Partial Fourier acquisition by retaining 192 of 256 phase-encoding lines.
* Implement and compare Zero-Filled, Homodyne, and POCS-based reconstruction methods.
* Evaluate reconstruction quality using Mean Squared Error (MSE), Root Mean Squared Error (RMSE), and Peak Signal-to-Noise Ratio (PSNR).
* Investigate the influence of sampling direction, phase information, and coil sensitivity maps on reconstruction performance.

## Background

### 1. Partial Fourier MRI

For a complex-valued MRI image, the k-space data approximately satisfy conjugate symmetry when the underlying image has negligible spatially varying phase:

$$
K(-k_x,-k_y)\approx K^*(k_x,k_y)
$$

This property allows some missing k-space information to be estimated from acquired samples. However, phase variations can violate simple conjugate symmetry, making phase estimation and correction important.

In this project, 75% Partial Fourier sampling retains:

* Full matrix size: \(256\times256\)
* Acquired phase-encoding lines: 192
* Missing phase-encoding lines: 64
* Acquisition fraction: 75%

### 2. Zero-Filled Reconstruction

Unacquired k-space samples are set to zero, and the inverse Fourier transform is applied:

$$
I_{\mathrm{ZF}}=\mathcal{F}^{-1}\{K_{\mathrm{PF}}\}
$$

This provides a simple baseline reconstruction. Missing k-space information causes blurring and other reconstruction artifacts.

### 3. Homodyne Reconstruction

Homodyne reconstruction uses the approximate conjugate symmetry of MRI k-space together with a low-resolution phase estimate.

The method generally involves:

1. Zero-filling the missing k-space region.
2. Estimating the image phase from central, low-frequency k-space data.
3. Correcting the phase of the zero-filled reconstruction.
4. Applying an asymmetric k-space weighting function.
5. Taking the appropriate real-valued component to obtain the reconstructed image.

Homodyne reconstruction can reduce Partial Fourier artifacts when the phase estimate is sufficiently accurate. Its performance depends on the acquired fraction, phase variation, and weighting strategy.

### 4. Projection onto Convex Sets (POCS)

POCS iteratively enforces constraints in image and k-space domains.

The general approach is:

1. Initialize the reconstruction using zero-filled k-space.
2. Transform the current image into k-space.
3. Replace the acquired samples with their measured values.
4. Apply an image-domain constraint, such as a phase-consistency constraint.
5. Repeat until the iteration limit or convergence criterion is reached.

For multi-coil data, this project also investigates BART's **POCSENSE** implementation, which incorporates coil sensitivity information.

POCS and POCSENSE are related but not interchangeable: POCSENSE uses sensitivity-encoded reconstruction and should not be treated as identical to every conventional single-image POCS implementation.

## Methodology

### Data generation

The fully sampled multi-coil phantom is generated using BART. The simulated coil images are transformed into k-space using a two-dimensional Fourier transform.

A 75% Partial Fourier mask is then applied along the phase-encoding direction, leaving 25% of the k-space lines unacquired.

### Reconstruction pipeline

```text
BART Multi-Coil Phantom
          |
          v
  Fully Sampled K-Space
          |
          v
  75% Partial Fourier Mask
          |
     _____|______
    |      |     |
    v      v     v
   Zero  Homodyne POCS/
 Filling         POCSENSE
    |      |     |
    v      v     v
  Reconstructed Coil Images
          |
          v
   Magnitude / Coil Combination
          |
          v
  Comparison with Full Reference
          |
          v
   MSE, RMSE and PSNR Evaluation
```

### Software and tools

* **BART:** Phantom generation, Fourier transforms, sensitivity handling, and POCSENSE reconstruction.
* **Python:** K-space manipulation, reconstruction experiments, metric calculation, and visualization.
* **NumPy:** Numerical operations and complex-valued data processing.
* **Matplotlib:** Visualization and comparison of reconstructed images.

## Evaluation Metrics

### Mean Squared Error (MSE)

$$
\mathrm{MSE}=\frac{1}{N}\sum_{i=1}^{N}(I_i-\hat I_i)^2
$$

Measures the average squared difference between the reference and reconstructed images. Lower values indicate less reconstruction error.

### Root Mean Squared Error (RMSE)

$$
\mathrm{RMSE}=\sqrt{\mathrm{MSE}}
$$

Measures reconstruction error in the same intensity units as the image.

### Peak Signal-to-Noise Ratio (PSNR)

$$
\mathrm{PSNR}=10\log_{10}
\left(\frac{I_{\max}^2}{\mathrm{MSE}}\right)
$$

Here, \(I_{\max}\) is the maximum intensity of the reference image. Higher PSNR indicates lower reconstruction error under the same evaluation convention.

All methods should be compared against the same fully sampled reference, using consistent image scaling, coil combination, and evaluation regions.

## Preliminary Results

The following results are from the current **BART phantom experiment**, using 75% Partial Fourier sampling.

| Metric | Zero Filling |      POCSENSE |
| ------ | -----------: | ------------: |
| MSE    | 6,617,177.66 | 54,201,109.31 |
| RMSE   |     2,572.39 |      7,362.14 |
| PSNR   |     36.75 dB |      27.62 dB |

The current experiment shows that zero filling outperforms the tested POCSENSE configuration by approximately 9.13 dB in PSNR.

This is a preliminary result, not evidence that POCS methods are generally inferior. The sensitivity encoding, Partial Fourier mask convention, phase constraints, and reconstruction configuration require further investigation.

**Homodyne results are pending implementation and validation.** They will be added after the method has been run against the same reference data and evaluated with the same metrics.


## Repository Structure

```text
MRI-Partial-Fourier-Reconstruction/
├── README.md
├── scripts/
│   ├── make_pf.py
│   ├── make_two_pf.py
│   ├── metrics.py
│   └── homodyne.py
├── results/
│   ├── phantom_reference.png
│   ├── zf_image.png
│   ├── pocs_image.png
│   └── homodyne_image.png
└── requirements.txt
```

The structure above is the proposed organization. Include files and results as they are implemented; do not commit generated files that are not yet available. Large intermediate k-space arrays can be regenerated from the scripts and generally do not need to be stored in Git.

## Running the BART Experiments

Generate the phantom and fully sampled k-space:

```bash
bart phantom -x 256 -s 4 phantom
bart fft -u 6 phantom full_ksp
```

After creating the Partial Fourier dataset `pf_ksp`, reconstruct using zero filling:

```bash
bart fft -iu 6 pf_ksp zf_coils
bart cabs zf_coils zf_abs
bart rss 8 zf_abs zf_rss
```

Run POCSENSE using the prepared sensitivity maps:

```bash
bart pocsense -i 30 -r 0 pf_ksp sens pocs_ksp
bart fft -iu 6 pocs_ksp pocs_coils
bart cabs pocs_coils pocs_abs
bart rss 8 pocs_abs pocs_rss
```

The POCSENSE output is transformed back into image space before magnitude calculation and coil combination.

**Note:** These commands assume that the input files, dimensions, sampling mask, and sensitivity maps have been prepared consistently. Consult the scripts for the exact conventions used in each experiment.

## Future Work

* Implement Homodyne reconstruction and compare it with Zero Filling and POCSENSE.
* Evaluate different Partial Fourier acquisition fractions, such as 60%, 70%, 75%, and 85%.
* Compare alternative sampling directions and central-k-space coverage.
* Study the effect of phase estimation and coil sensitivity accuracy on reconstruction quality.
* Investigate reconstruction quality using both full-image and region-of-interest metrics.
* Extend the experiment to real multi-coil MRI magnitude and phase data, accounting for the limitations of reconstructed images compared with raw acquisition k-space.

## Disclaimer

This is an educational and experimental MRI reconstruction project. The simulated results are intended for algorithmic comparison and are not validated for clinical diagnostic use.
