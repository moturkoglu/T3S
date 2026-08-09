# T³S: Think in Thermal Time for Generalizable Crop Mapping from Satellite Image Time Series

<p align="center">
  <b>Official implementation of Thermal Time-based Temporal Sampling (T³S)</b><br>
  A model-agnostic, phenology-aware temporal sampling strategy for satellite image time series.
</p>

<p align="center">
  <a href="https://github.com/moturkoglu/swiss_crop_thermal">Code</a> •
  <a href="https://huggingface.co/datasets/EOA-team/SwissCrop25">Dataset: SwissCrop25</a>
</p>

---

## TL;DR

Crop phenology does not follow the calendar: the same crop can reach the same growth stage weeks earlier or later in different years because of temperature variability.

**T³S replaces calendar-time sampling with thermal-time sampling.** It computes cumulative growing degree days (**cGDD**), partitions the thermal-time axis into equal intervals, and selects the **least-cloudy Sentinel-2 observation** from each interval.

The result is a simple input-level preprocessing step that:

- aligns phenologically comparable observations across years and regions,
- reduces redundant satellite observations,
- requires **no architectural modification**,
- works with different model families,
- improves both **cross-domain accuracy** and **uncertainty calibration**.

On SwissCrop, T³S + U-TAE improves average cross-year accuracy from **71.2% → 77.0%** and reduces ECE from **5.1% → 1.1%**.

---

## Method

Standard satellite image time-series pipelines usually sample observations along calendar time:

```text
January ---------------- April ---------------- July
   |       |       |       |       |       |
                 calendar time
```

T³S instead samples along accumulated thermal time:

```text
0 cGDD -------------- thermal development -------------- max cGDD
  |         |         |         |         |         |
               phenological time
```

For day \(i\), growing degree days are computed as

\[
\mathrm{GDD}_i =
\max\left(
0,
\frac{T_{\max,i} + T_{\min,i}}{2} - T_{\mathrm{base}}
\right),
\]

and accumulated through the season:

\[
\mathrm{cGDD}_d = \sum_{i=1}^{d} \mathrm{GDD}_i.
\]

We use \(T_{\mathrm{base}} = 0^\circ C\). To obtain a sequence of \(T\) observations, the cGDD range is divided into \(T\) equal thermal-time intervals and the least-cloudy Sentinel-2 observation within each interval is retained.

In the paper, we use **T = 24** observations.

### Why sampling instead of encoding?

Thermal Positional Encoding (TPE) injects thermal information into a model's positional representation. T³S acts **before the model**: it uses thermal time to determine *which observations the model sees*.

This makes T³S applicable to architectures where positional encodings are unavailable, fixed, or expensive to modify.

---

## SwissCrop / SwissCrop25

The original **SwissCrop** dataset introduced with T³S contains country-scale Sentinel-2 crop time series for Switzerland from **2021–2023**, paired with crop labels and daily temperature information for phenology-aware modelling.

**SwissCrop is now included in the extended [SwissCrop25 benchmark](https://huggingface.co/datasets/EOA-team/SwissCrop25).**

### Download

👉 **[SwissCrop25 on Hugging Face](https://huggingface.co/datasets/EOA-team/SwissCrop25)**

SwissCrop was designed for realistic cross-year evaluation and contains:

- country-wide Sentinel-2 Level-2A imagery,
- multi-year crop labels,
- daily temperature data,
- strong inter-annual climatic variability,
- long-tailed real-world crop distributions.

Please refer to the Hugging Face dataset page for the latest dataset structure and download instructions.

---

## Results

### Cross-year generalization on SwissCrop

Average over the six train/test year combinations:

| Method | Accuracy ↑ | mIoU ↑ | IoU ↑ | ECE ↓ |
|---|---:|---:|---:|---:|
| U-TAE | 71.2 | 17.3 | 55.3 | 5.1 |
| MC-Dropout | 71.9 | 17.3 | 56.1 | 3.4 |
| Thermal Positional Encoding | 71.3 | 17.1 | 55.5 | 7.7 |
| Deformable Sampling | 75.4 | 20.5 | 60.5 | 6.3 |
| **T³S + U-TAE** | **77.0** | **21.5** | **62.6** | **1.1** |

### Different architectures

| Method | Accuracy ↑ | IoU ↑ | ECE ↓ |
|---|---:|---:|---:|
| U-TAE | 71.2 | 55.3 | 5.1 |
| Galileo | 70.1 | 54.0 | 8.8 |
| **T³S + U-TAE** | **77.0** | **62.6** | **1.1** |
| T³S + Galileo | 74.7 | 59.7 | 5.8 |

T³S also improves cross-region transfer on the **TimeMatch** benchmark using the PSE+LTAE backbone, supporting its model-agnostic design.

---

## Repository

The main training implementation is in:

- [`train_final.py`](./train_final.py) — main U-TAE training pipeline
- [`src/dataset_CH_TempC.py`](./src/dataset_CH_TempC.py) — SwissCrop loader and temporal sampling
- [`inference_semantic_ECE.py`](./inference_semantic_ECE.py) — inference and calibration evaluation
- [`train_galileo.py`](./train_galileo.py) — Galileo experiments
- [`src/galileo_segmentation.py`](./src/galileo_segmentation.py) — Galileo segmentation model

---

## Training

`train_final.py` exposes the switches used for the main sampling variants.

### T³S

```bash
torchrun --nproc_per_node=2 train_final.py \
    --use_temperature_calendar \
    --use_temperature_subsampling \
    --PE v2 \
    --dataset_folder /path/to/swisscrop \
    --res_dir ./storage/results
```

### U-TAE baseline

```bash
torchrun --nproc_per_node=2 train_final.py \
    --no_sliding_subsample \
    --dataset_folder /path/to/swisscrop \
    --res_dir ./storage/results
```

### Deformable / cloud-aware calendar sampling

```bash
torchrun --nproc_per_node=2 train_final.py \
    --dataset_folder /path/to/swisscrop \
    --res_dir ./storage/results
```

### Thermal Positional Encoding baseline

```bash
torchrun --nproc_per_node=2 train_final.py \
    --use_temperature_calendar_no_sliding_subsample \
    --dataset_folder /path/to/swisscrop \
    --res_dir ./storage/results
```

You can change the number of GPUs and standard optimization parameters directly through the command-line arguments in [`train_final.py`](./train_final.py).

---

## Inference

Example for T³S:

```bash
CUDA_VISIBLE_DEVICES=0 python inference_semantic_ECE.py \
    --weight_folder /path/to/checkpoints \
    --result_folder ./results/t3s \
    --PE v2 \
    --use_temperature_calendar \
    --use_temperature_subsampling \
    --truncate_portion 1
```

`--truncate_portion 1` evaluates the complete time series. Values below `1` can be used for early-season evaluation.

---

## Experimental setting

The paper evaluates T³S along three main axes:

1. **Cross-year generalization** on SwissCrop.
2. **Operational robustness**, including early-season prediction and training with only 10% of labels.
3. **Cross-region generalization** on TimeMatch.

The method is evaluated with three architecturally different backbones:

- **U-TAE** — convolutional encoder-decoder with temporal attention,
- **PSE+LTAE** — pixel-set encoder with temporal attention,
- **Galileo** — pretrained Earth observation foundation model.

---

## Citation

If you use T³S or SwissCrop in your research, please cite the paper:

```bibtex
@inproceedings{turkoglu2026t3s,
  title     = {T3S: Think in Thermal Time for Generalizable Crop Mapping from Satellite Image Time Series},
  author    = {Turkoglu, Mehmet Ozgur and Ledain, S{\'e}l{\`e}ne and Lauber, Thomas and Zweidler, Jeffrey and Aasen, Helge},
  year      = {2026}
}
```

The BibTeX entry will be updated with the final proceedings information once available.

For the extended dataset, please also refer to the citation information provided on the **[SwissCrop25 Hugging Face page](https://huggingface.co/datasets/EOA-team/SwissCrop25)**.

---

## Authors

**Mehmet Ozgur Turkoglu**, Sélène Ledain, Thomas Lauber, Jeffrey Zweidler, and Helge Aasen

Earth Observation of Agroecosystems Team, Agroscope, Switzerland  
ETH Zurich, Switzerland

---

## Dataset link

**SwissCrop25:**  
https://huggingface.co/datasets/EOA-team/SwissCrop25

**Repository:**  
https://github.com/moturkoglu/swiss_crop_thermal
