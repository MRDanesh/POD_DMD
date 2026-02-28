# Cylinder Wake Flow Prediction with Dynamic Mode Decomposition (DMD)

The section demonstrates **flow-field prediction** for the 2D cylinder wake dataset (`cylinder_nektar_wake.mat`) using **Dynamic Mode Decomposition (DMD)**.  
The code trains a rank-4 DMD model on a portion of the time snapshots, then **rolls out** the dynamics to predict the remaining (test) snapshots.

---

## Dataset

The MATLAB file contains:
- `X_star ∈ R^{N×2}`: spatial coordinates of grid points
- `t ∈ R^{nt×1}`: time vector
- `U_star ∈ R^{N×2×nt}`: velocity snapshots  
  - `u = U_star[:,0,:] ∈ R^{N×nt}` (x-velocity)
  - `v = U_star[:,1,:] ∈ R^{N×nt}` (y-velocity)
- `p_star ∈ R^{N×nt}`: pressure snapshots (not used here)

In this project we build the snapshot matrix from the x-velocity:
$$
X = [x_1, x_2, \dots, x_{nt}] \in \mathbb{R}^{N \times nt},
$$
where $$x_k \in \mathbb{R}^N$$ is the field at time $t_k$

---

## Train/Test Split

We use a **contiguous** split in time:
- Training: first 70% of snapshots
- Test: remaining 30%

Let $X_{\mathrm{tr}}$ and $X_{\mathrm{te}}$ denote training and test snapshot matrices.

We subtract the **training mean field**:
$$
\bar{x} = \frac{1}{n_{\mathrm{tr}}}\sum_{k \in \mathrm{train}} x_k,
$$
$$
X_{\mathrm{tr},c} = X_{\mathrm{tr}} - \bar{x}\mathbf{1}^T,
\qquad
X_{\mathrm{te},c} = X_{\mathrm{te}} - \bar{x}\mathbf{1}^T.
$$

---

## Dynamic Mode Decomposition (DMD): Mathematical Formulation

DMD seeks a linear time-advance model:
$$
x_{k+1} \approx A x_k,
$$
where $$A \in \mathbb{R}^{N\times N}$$ is an (unknown) linear operator that best maps snapshots forward in time.

### 1) Build snapshot pairs
From the mean-subtracted training data $X_{\mathrm{tr},c}$:
$$
X_1 = [x_1, x_2, \dots, x_{m-1}],
\qquad
X_2 = [x_2, x_3, \dots, x_m],
$$
so that ideally
$$
X_2 \approx A X_1.
$$

### 2) Low-rank projection via SVD
Compute the (economy) SVD of $X_1$:
$$
X_1 = U \Sigma V^T.
$$
Truncate to rank $r$ (here $r=4$):
$$
X_1 \approx U_r \Sigma_r V_r^T.
$$

### 3) Reduced operator
Project the dynamics onto the subspace spanned by $U_r$:
$$
\tilde{A} = U_r^T A U_r \approx U_r^T X_2 V_r \Sigma_r^{-1}
\in \mathbb{R}^{r\times r}.
$$

### 4) Eigen-decomposition and DMD modes
Compute eigenpairs of the reduced operator:
$$
\tilde{A} W = W \Lambda,
$$
where $$\Lambda = \mathrm{diag}(\lambda_1,\dots,\lambda_r)$$ are DMD eigenvalues.

The **exact DMD modes** are:
$$
\Phi = X_2 V_r \Sigma_r^{-1} W
\in \mathbb{R}^{N \times r}.
$$

### 5) Forecasting (discrete-time)
Given an initial state $x_0$ (we use the **first test snapshot**), compute modal amplitudes:
$$
b = \Phi^{\dagger} x_0,
$$
and predict:
$$
\hat{x}_k = \Phi \Lambda^{k} b,
\qquad k = 0,1,2,\dots
$$
Finally, add the mean field back:
$$
\hat{x}_k^{(\mathrm{full})} = \bar{x} + \hat{x}_k.
$$

---

## What the Code Does

1. Loads the cylinder wake dataset and extracts $u(x,t)$.
2. Splits the snapshots into training and test time windows (70/30).
3. Mean-subtracts using the training mean field.
4. Constructs $X_1, X_2$ from training data.
5. Computes rank-4 DMD (SVD → reduced operator → eigenvalues/modes).
6. Fits initial modal amplitudes using the first test snapshot.
7. Predicts the entire test window using $\hat{x}_k = \Phi \Lambda^k b$.
8. Adds back the mean field to obtain the full predicted flow.

---

