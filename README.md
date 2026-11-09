# OEMRITemporalUpsampling

Temporal upsampling of dynamic oxygen-enhanced (OE) lung MRI with time-conditioned, flow-based video frame interpolation.

## Motivation

Oxygen-enhanced lung MRI maps regional ventilation without a contrast agent: the subject breathes room air, then 100 % oxygen, then air again, and the paramagnetic oxygen shortens T2\* and lowers the lung signal by a few percent. The clinical protocol acquires a dual-echo T2\*-sensitive image series at about 1.5 s per frame, from which percent signal enhancement (PSE), the change in R2\* and wash-in time constants are derived.

1.5 s per frame is slow compared with the physiology. Quiet breathing (about 0.3 Hz) sits at the Nyquist limit of the protocol, so respiratory motion is aliased into the signal curves and the maps. Faster acquisition is possible (58 ms per frame on the same scanner) but costs coverage and SNR, so clinical protocols stay slow.

This project asks whether a learned video frame interpolation network can synthesise the frames between the acquired ones (26x, from 1.54 s to 58 ms) in a way that recovers the breathing dynamics while leaving the oxygen measurements unchanged.

## Approach

- **Training data with real high-rate ground truth.** A single-slice fast acquisition (2000 frames at 58 ms, both echoes) is sub-sampled by 26 to mimic the clinical frame rate. The network receives two frames 1.5 s apart and the relative time of the target, and is trained to reconstruct the real frame at that time.
- **Flow-based, time-conditioned interpolation.** A U-Net encoder predicts two displacement fields (from the target time to each input frame) and a blending mask. Both inputs are backward-warped to the target time, blended, and corrected by a refinement U-Net. Because the target time is an input, any position in the gap can be synthesised.
- **One-echo and two-echo variants.** The two-echo version processes both echoes jointly with a shared motion estimate, so the echo ratio, and hence R2\*, can be preserved.
- **OE-aware loss terms.** On top of reconstruction and warping losses: a lung-masked term, a temporal-consistency term that supervises the change between two synthesised frames of the same gap, an enhancement term on the lung-mean signal, and an echo-ratio term for the two-echo model.
- **Two levels of evaluation.** On the fast scan, against real 58 ms frames: lung-region error, fidelity of the lung signal curve, how much of the respiratory oscillation is recovered, pixel-wise temporal variability, and the R2\* curve. On a real clinical OE scan (no high-rate ground truth): whether PSE and the oxygen-induced R2\* change computed from the upsampled sequence agree with those from the acquired frames.

## What this repository contains

| File | Content |
|---|---|
| `oe_lung_temporal_upsampling.ipynb` | The project notebook: configuration, data loading and lung masks, models (linear interpolation, U-Net, flow-based one- and two-echo networks), loss functions with explanations, training loop, evaluation (image metrics, dense 26x reconstruction, breathing-spectrum analysis, PSE and ΔR2\* consistency checks), experiment grid with a results registry, and result figures. Executed outputs are included. |
| `data_timing_analysis.ipynb` | Analysis of the acquisitions: DICOM timing and sequence parameters, gas-phase check of the clinical scan, lung-signal and R2\* curves of the fast scans. |
| `data/` | Placeholder for the image data (not included, see below). |

## Data

The notebooks expect registered Analyze (`.hdr`/`.img`) image series under `data/`:

- `data/0601_reg/reg/IM-01-01-*` and `IM-01-02-*`: fast scan, echo 1 and echo 2 (2000 frames each, 96 x 96).
- `data/TV3_20220208/reg/IM-0{slice}-0{echo}-*`: clinical OE scan, 340 frames per slice and echo.

The data are not part of this repository.

## Running

Python 3.9 with PyTorch 2.x (CUDA), NumPy, SciPy, scikit-image, nibabel, pandas, matplotlib and Jupyter. Set `SMOKE = True` in the first code cell for a quick pipeline check (2-epoch models), `SMOKE = False` for the full experiments. Trained checkpoints are written to `checkpoints/` and reused on later runs; all metrics are collected in `results/registry.csv`.

## Status

Work in progress. Results so far are from one subject and one training slice; conclusions about generalisation need further data. The clinical-scan checks are consistency checks (upsampled versus acquired frames), not accuracy measurements, because no high-rate ground truth exists for an oxygen challenge yet.

## Acknowledgements

Developed in collaboration with Mina Kim (UCL Centre for Medical Image Computing), whose dual-echo OE-MRI protocol and data this work builds on.

## License

MIT, see `LICENSE`.
