# T³S: Think in Thermal Time for Generalizable Crop Mapping from Satellite Image Time Series

Official repository for **T³S (Thermal Time-based Temporal Sampling)**, a simple, model-agnostic approach for improving the generalization of crop mapping models across years and regions.

> **Code release:** implementation and reproducibility scripts will be added to this repository.

## TL;DR

Crop development does not strictly follow calendar time. Depending on temperature, the same crop can reach the same phenological stage substantially earlier or later across different years.

**T³S replaces calendar-time sampling with thermal-time sampling.** Satellite observations are re-indexed using cumulative growing degree days (**cGDD**), so that observations corresponding to similar crop development stages are more consistently aligned across years.

This simple input-level strategy:

- improves cross-year and cross-region generalization,
- substantially improves uncertainty calibration,
- remains effective in low-data and early-season settings,
- requires no architectural modification, and
- can be applied to different model families.

## Method

T³S uses accumulated temperature as a proxy for crop development.

For day \(i\), growing degree days are computed as

\[
\mathrm{GDD}_i =
\max\left(
0,
\frac{T_{\max,i} + T_{\min,i}}{2} - T_{\mathrm{base}}
\right),
\]

with cumulative growing degree days

\[
\mathrm{cGDD}_d = \sum_{i=1}^{d} \mathrm{GDD}_i.
\]

Instead of sampling satellite observations uniformly in calendar time, T³S samples them uniformly along the thermal-time axis. Within each thermal interval, the least-cloudy Sentinel-2 observation is selected.

The central idea is simple:

**calendar time tells us when an observation was acquired; thermal time better reflects where the crop is in its development.**

## Results

On the SwissCrop cross-year benchmark, T³S consistently improves both predictive performance and uncertainty calibration.

| Method | Accuracy ↑ | mIoU ↑ | IoU ↑ | ECE ↓ |
|---|---:|---:|---:|---:|
| U-TAE | 71.2 | 17.3 | 55.3 | 5.1 |
| MC-Dropout | 71.9 | 17.3 | 56.1 | 3.4 |
| Thermal Positional Encoding | 71.3 | 17.1 | 55.5 | 7.7 |
| Deformable Sampling | 75.4 | 20.5 | 60.5 | 6.3 |
| **T³S + U-TAE** | **77.0** | **21.5** | **62.6** | **1.1** |

T³S also improves performance when applied to a pretrained Earth observation foundation model and improves cross-region transfer on the TimeMatch benchmark.

## SwissCrop Dataset

The paper introduces **SwissCrop**, a country-scale, multi-year Sentinel-2 crop mapping dataset for Switzerland paired with daily temperature data.

**SwissCrop is distributed as part of the extended [SwissCrop25 benchmark](https://huggingface.co/datasets/EOA-team/SwissCrop25).**

### Dataset

👉 **[SwissCrop25 on Hugging Face](https://huggingface.co/datasets/EOA-team/SwissCrop25)**

SwissCrop contains data used for the cross-year experiments in the T³S paper and is included within the broader SwissCrop25 benchmark.

If you use the **SwissCrop subset introduced with T³S**, please cite **both the T³S paper and the SwissCrop25 dataset paper**.

## Paper

**T³S: Think in Thermal Time for Generalizable Crop Mapping from Satellite Image Time Series**

Mehmet Ozgur Turkoglu, Sélène Ledain, Thomas Lauber, Jeffrey Zweidler, and Helge Aasen

Earth Observation of Agroecosystems Team, Agroscope, Switzerland  
ETH Zurich, Switzerland

Paper link will be added here when the proceedings version is publicly available.

## Code

The implementation used in the paper will be released in this repository.

**Repository:** https://github.com/moturkoglu/T3S

## Citation

If you use T³S, please cite:

```bibtex
@article{turkoglu2025t3s,
  title         = {{$T^{3}S$: Think in Thermal Time for Generalizable Crop Mapping from Satellite Image Time Series}},
  author        = {Turkoglu, Mehmet Ozgur and Ledain, Selene and Zweidler, Jeffrey and Lauber, Thomas and Aasen, Helge},
  journal       = {arXiv preprint arXiv:2506.12885},
  year          = {2025},
  eprint        = {2506.12885},
  archivePrefix = {arXiv},
  primaryClass  = {cs.CV},
  doi           = {10.48550/arXiv.2506.12885}
}
```

The citation will be updated with the final proceedings information once available.

If you use the **SwissCrop data**, please also cite the **SwissCrop25 dataset paper**. See the citation information on the [SwissCrop25 Hugging Face page](https://huggingface.co/datasets/EOA-team/SwissCrop25).
