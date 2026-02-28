# Reduced-Order Modeling for Cylinder Wake: POD-Based and DMD Prediction

This repository is a compact project to demonstrate **reduced-order modeling (ROM)** and **data-driven flow prediction** on a canonical fluid dynamics benchmark: the **2D cylinder wake**. It contains two parallel pipelines:

- **POD-based ROM** with a simple, interpretable coefficient predictor (harmonic regression in POD space)
- **DMD-based ROM** for linear spectral forecasting in snapshot space

Both approaches use the **same train/test split** so their prediction performance can be compared fairly.

---

## Repository Structure

```text
.
├── data/
│   └── cylinder_nektar_wake.mat
├── POD/
│   └── README_POD.md
├── DMD/
│   └── README_DMD.md
└── README.md   (this file)