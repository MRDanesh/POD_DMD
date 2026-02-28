

# Cylinder Wake Prediction with POD + Two-Frequency Harmonic Model (4 Modes)

This section implements a simple, interpretable reduced-order prediction model for the 2D cylinder wake dataset using:

1) **Proper Orthogonal Decomposition (POD)** to reduce the flow snapshots to the first **4 dominant modes**, and  
2) a **two-frequency harmonic regression** model to forecast the POD coefficients:
- one frequency for the pair $(a_1, a_2)$
- another frequency for the pair $(a_3, a_4)$

The predicted coefficients are then compared against the ground truth coefficients on a test set.

---

## Snapshot Matrix

We use the x-velocity snapshots $u(\mathbf{x}, t)$ and form the snapshot matrix:

$$
X = [x_1, x_2, \dots, x_{n_t}] \in \mathbb{R}^{N \times n_t},
$$

where:
- $N$ is the number of spatial points,
- $n_t$ is the number of time snapshots,
- $x_k \in \mathbb{R}^N$ is the vectorized field at time $t_k$.

In the code, `X = u`.

---

## Train/Test Split

We split the snapshots in time (contiguous split):

- Train: first 70% of snapshots  
- Test: remaining 30%

Let $X_{\mathrm{tr}}$ and $X_{\mathrm{te}}$ denote train and test matrices.

---

## POD on Training Data

### Mean subtraction (training mean)

Compute the training mean field:

$$
\bar{x} = \frac{1}{n_{\mathrm{tr}}} \sum_{k \in \mathrm{train}} x_k \in \mathbb{R}^{N},
$$

and subtract it from training snapshots:

$$
X_{\mathrm{tr},c} = X_{\mathrm{tr}} - \bar{x}\mathbf{1}^T.
$$

### SVD (POD)

Compute the economy SVD:

$$
X_{\mathrm{tr},c} = U \Sigma V^T.
$$

The POD modes are the left singular vectors. We keep the first $r=4$ modes:

$$
\Phi_r = U_{(:,1:r)} \in \mathbb{R}^{N \times 4}.
$$

### POD coefficients (projection)

For any snapshot $x_k$, the POD coefficient vector $a_k \in \mathbb{R}^4$ is:

$$
a_k = \Phi_r^T (x_k - \bar{x}).
$$

Stacking over time gives the coefficient matrices:

- Training coefficients:

$$
A_{\mathrm{tr}} = \Phi_r^T (X_{\mathrm{tr}} - \bar{x}\mathbf{1}^T) \in \mathbb{R}^{4 \times n_{\mathrm{tr}}}.
$$

- Test coefficients:

$$
A_{\mathrm{te}} = \Phi_r^T (X_{\mathrm{te}} - \bar{x}\mathbf{1}^T) \in \mathbb{R}^{4 \times n_{\mathrm{te}}}.
$$

---

## Frequency Estimation from POD Coefficient Pairs

For cylinder wake, the dominant dynamics often appear as oscillatory pairs in POD space.  
We treat $(a_1,a_2)$ as one oscillator and $(a_3,a_4)$ as another (often a harmonic).

### Phase of a coefficient pair

For the first pair:

$$
\theta_{12}(t) = \mathrm{unwrap}\left(\arctan2(a_2(t), a_1(t))\right),
$$

and the instantaneous angular frequency is approximated by:

$$
\omega_{12}(t) \approx \frac{d\theta_{12}}{dt}.
$$

We estimate a constant frequency by averaging over the training window:

$$
\omega_{12} = \mathrm{mean}\left(\omega_{12}(t)\right).
$$

Similarly for the second pair:

$$
\theta_{34}(t) = \mathrm{unwrap}\left(\arctan2(a_4(t), a_3(t))\right),
$$
$$
\omega_{34}(t) \approx \frac{d\theta_{34}}{dt},
$$
$$
\omega_{34} = \mathrm{mean}\left(\omega_{34}(t)\right).
$$

---

## Harmonic Regression Model for Each Coefficient

Instead of evolving the coefficients via a coupled ODE, we fit each coefficient with a sinusoidal curve at its pair’s frequency.

For any coefficient $a_i(t)$, we fit:

$$
a_i(t) \approx b_i + A_i \cos(\omega t) + B_i \sin(\omega t),
$$

where:
- $b_i$ is a constant offset,
- $A_i, B_i$ control amplitude and phase,
- $\omega$ is either $\omega_{12}$ (for $i=1,2$) or $\omega_{34}$ (for $i=3,4$).

### Least-squares fitting

Given sampled times $\{t_k\}$, define the design matrix:

$$
M =
\begin{bmatrix}
1 & \cos(\omega t_1) & \sin(\omega t_1) \\
1 & \cos(\omega t_2) & \sin(\omega t_2) \\
\vdots & \vdots & \vdots \\
1 & \cos(\omega t_n) & \sin(\omega t_n)
\end{bmatrix}.
$$

We fit each coefficient with least squares:

min_{b,A,B} || M [b, A, B]^T - y ||_2^2

where y = [a_i(t1), a_i(t2), ..., a_i(tn)]^T
and M = [1, cos(ω t), sin(ω t)] stacked over all times.

---

## Prediction on the Test Set

Once parameters $(b_i, A_i, B_i)$ are fitted on training data, we evaluate the sinusoids on test times.

For $(a_1,a_2)$ using $\omega_{12}$:

$$
\hat{a}_1(t) = b_1 + A_1 \cos(\omega_{12} t) + B_1 \sin(\omega_{12} t),
$$

$$
\hat{a}_2(t) = b_2 + A_2 \cos(\omega_{12} t) + B_2 \sin(\omega_{12} t).
$$

For $(a_3,a_4)$ using $\omega_{34}$:

$$
\hat{a}_3(t) = b_3 + A_3 \cos(\omega_{34} t) + B_3 \sin(\omega_{34} t),
$$

$$
\hat{a}_4(t) = b_4 + A_4 \cos(\omega_{34} t) + B_4 \sin(\omega_{34} t).
$$

The predicted coefficient matrix is:

$$
\hat{A}_{\mathrm{te}} =
\begin{bmatrix}
\hat{a}_1(t_1) & \cdots & \hat{a}_1(t_{n_{\mathrm{te}}}) \\
\hat{a}_2(t_1) & \cdots & \hat{a}_2(t_{n_{\mathrm{te}}}) \\
\hat{a}_3(t_1) & \cdots & \hat{a}_3(t_{n_{\mathrm{te}}}) \\
\hat{a}_4(t_1) & \cdots & \hat{a}_4(t_{n_{\mathrm{te}}})
\end{bmatrix}.
$$

---
