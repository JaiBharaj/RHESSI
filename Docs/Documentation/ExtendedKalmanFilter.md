# Extended Kalman Filter (EKF)

This module defines the `ExtendedKalmanFilter` class for nonlinear state estimation with support for different integration strategies. It also includes Jacobian computation functions for polar and spherical dynamics.
- `ExtendedKalmanFilter`: Implements an Extended Kalman Filter (EKF) with pluggable numerical integrators for general nonlinear systems.

---

```python
def __init__(self, f_dynamics, f_jacobian, H, Q, R, x0, P0, integrator):
  ...
```
- **Parameters**:
  - `f_dynamics`: Dynamics function, `f(x)`, computing `dx/dt`.
  - `f_jacobian`: Jacobian function, `df/dx`.
  - `H`: Measurement matrix.
  - `Q`: Process noise covariance matrix.
  - `R`: Measurement noise covariance matrix.
  - `x0`: Initial state.
  - `P0`: Initial state covariance.
  - `integrator`: Object providing `step(f, x, dt)` and `transition_matrix(x, dt)`.

---

```python
def predict(self, dt):
  elf.x = self.integrator.step(self.f, self.x, dt)
  F_disc = self.integrator.transition_matrix(self.x, dt)
  self.P = F_disc @ self.P @ F_disc.T + self.Q
  return self.x, self.P
```
- **Purpose**: Time-update (prediction) step over a time increment `dt`.
- **Parameters**:
    - `dt`: Time increment in seconds.
- **Implementation**:
    - Propagate the state: `x ← integrator.step(f, x, dt)`.
    - Compute transition matrix: `F_disc ← integrator.transition_matrix(x, dt).
    - Propagate covariance: `P ← F_disc · P · F_discᵀ + Q`.

---

 ```python
def update(self, z, eps=1e-8):
  ...
  return self.x, self.P
```
- **Purpose**: Incorporates measurement `z` to update state estimate and reduce uncertainty.
- **Implementation**:
  - Compute innovation: `y = z - H @ x`
  - Compute Kalman gain using `S` and solve system.
  - Update state and covariance using Joseph form.

---

```python
def predict_trajectory(self, times, measurements, x0, P0):
    ...
```
- **Purpose**: Propagates the satellite's state and covariance over a given time sequence using the Extended Kalman Filter (EKF), applying prediction and (optionally) update steps based on available measurements.
- **Parameters**:
  - `times` (`array-like of float`): Sequence of time stamps [t_0, t_1, ..., t_n] at which the filter should step through.
  - `measurements` (`array-like of ndarray`): Sequence of measurement vectors. Each element corresponds to a time in `times`. If a measurement is missing at a step, it should contain `NaN` values.
  - `x0` (`ndarray`): Initial state vector of the system.
  - `P0` (`ndarray`): Initial state covariance matrix.
- **Implementation**:
  * Initializes the internal state `x` and covariance `P` with `x0`, `P0`.
  * For each time step:
    1. Calculates the time increment `dt`. For the first step, `dt = 1e-3` to allow Jacobian estimation.
    2. Propagates the state forward using `self.predict(dt)`.
    3. If the current measurement `z` is available (no NaNs), performs a correction via `self.update(z)`.
    4. Records:
       * Radial position `x[0]`
       * Angular position `x[2]` (θ)
       * Variances `P[0,0]` and `P[2,2]`
       * A boolean flag indicating whether a measurement was used

---

```python
def get_trajectory(self):
  return np.array([self.pred_trajectory['t'], 
  self.pred_trajectory['r'], self.pred_trajectory['theta']]).T
```
- **Purpose**: Simulates the trajectory over given `times`, applying predictions and updates.

---

```python
def get_uncertainty(self):
  return np.array([self.uncertainty['r_var'], 
  self.uncertainty['theta_var'], self.uncertainty['measured']]).T
```
- **Purpose**: Retrieve uncertainty information recorded during trajectory prediction.

---

```python
def crash(self, N=100, dt=1.0, max_steps=10000):
    ...
    return crash_angles
```
- **Purpose**: Performs a Monte Carlo simulation to predict the distribution of crash impact angles (θ) based on the current estimated state and uncertainty.
- **Parameters**:
  * `N` (`int`, default=`100`): Number of Monte Carlo samples (i.e., simulated trajectories).
  * `dt` (`float`, default=`1.0`): Time step used in the numerical integration (in seconds).
  * `max_steps` (`int`, default=`10000`): Maximum number of integration steps allowed per trajectory before termination.
- **Implementation**:
  * For each sample, a new initial state is drawn from a multivariate normal distribution using the current mean state `self.x` and covariance `self.P`.
  * The state is numerically integrated forward in time using the system dynamics.
  * Integration continues until either:
    * The radial position `r` becomes less than or equal to Earth's radius (i.e., the satellite crashes).
    * The number of steps exceeds `max_steps`.
  * If a crash occurs, the angular position `θ` at impact is recorded.
  * The process is repeated `N` times to generate a distribution of crash angles.

---

```python
def crash3D(self, N=100, dt=1.0, max_steps=10000):
    ...
    return crash_angles
```
* **Purpose**: Performs a Monte Carlo simulation in 3D to estimate the satellite's crash location in terms of both polar angle (θ) and azimuthal angle (φ).
* **Parameters**:
  * `N` (`int`, default=`100`):
    Number of Monte Carlo samples.
  * `dt` (`float`, default=`1.0`):
    Integration time step (in seconds).
  * `max_steps` (`int`, default=`10000`):
    Maximum number of time steps allowed for each sample before stopping.
* **Implementation**:
  * For each sample:
    * Draw a new state from the current mean and covariance.
    * Propagate the sample forward in time using the system's integrator.
    * Continue until the satellite crashes (i.e., radius ≤ Earth’s radius), or the `max_steps` limit is reached.
  * If crash is detected:
    * Record the polar (θ) and azimuthal (φ) angles.
    * Print the crash information including impact angle and time steps taken.
* **Returns**: Array of tuples `(θ, φ)` from all successful crash simulations.

---

```python
def crash3D_with_thrust(self, delta_v, h_thrust, 
                        N=100, dt=1.0, max_steps=10000):
    ...
    return crash_angles
```
* **Purpose**: Performs a Monte Carlo simulation like `crash3D`, but introduces a mid-course thrust impulse (`delta_v`) at a specified altitude to alter the trajectory before impact.
* **Parameters**:
  * `delta_v` (`float`):
    Magnitude of the velocity impulse applied (in m/s).
  * `h_thrust` (`float`):
    Altitude threshold (in meters) at which the thrust maneuver is applied.
  * `N` (`int`, default=`100`):
    Number of Monte Carlo simulations to run.
  * `dt` (`float`, default=`1.0`):
    Integration time step.
  * `max_steps` (`int`, default=`10000`):
    Maximum number of integration steps per sample.
* **Implementation**:
  * Samples initial state from the EKF distribution.
  * Propagates forward until:
    * Altitude falls below `h_thrust` (i.e., `r ≤ R_earth + h_thrust`).
    * When triggered, applies thrust in the direction of the velocity vector.
  * Integration continues until crash or max steps reached.
  * Stores and reports `(θ, φ)` at crash location.
* **Returns**: Array of `(θ, φ)` tuples from simulations where the satellite crashed after the thrust maneuver.

---

```python
def compute_F_analytic(x, CD, A, m, GM, rho_func):
    ...
    return F
```
* **Purpose**: Computes the Jacobian matrix `F = ∂f/∂x` for satellite motion in polar coordinates, including the effects of atmospheric drag.
* **Parameters**:
  * `x` (`ndarray`, shape `(4,)`): State vector `[r, ṙ, θ, θ̇]`
  * `CD` (`float`): Drag coefficient.
  * `A` (`float`): Cross-sectional area of the satellite (in m²).
  * `m` (`float`): Mass of the satellite (in kg).
  * `GM` (`float`): Standard gravitational parameter (G × Mₑ).
  * `rho_func` (`Callable`): Function to compute atmospheric density given radius: `rho_func(r)`.
* **Implementation**:
  * Computes velocity and its partial derivatives.
  * Assembles the Jacobian matrix for the 4-dimensional polar system.
  * Includes drag terms through velocity dependencies in radial and angular components.
* **Returns**: The Jacobian matrix `∂f/∂x` evaluated at `x`.

---

```python
def compute_F_spherical(x, CD, A, m, GM, rho_func):
    ...
    return F
```

* **Purpose**: Computes the full Jacobian matrix `F = ∂f/∂x` for satellite dynamics in spherical coordinates, considering 3D motion and atmospheric drag.
* **Parameters**:
  * `x` (`ndarray`, shape `(6,)`): State vector `[r, ṙ, θ, θ̇, λ, λ̇]`
  * `CD` (`float`): Drag coefficient.
  * `A` (`float`): Cross-sectional area (m²).
  * `m` (`float`): Mass of the satellite (kg).
  * `GM` (`float`): Standard gravitational parameter.
  * `rho_func` (`Callable`): Function to compute atmospheric density from altitude: `rho_func(r - Rₑ)`.

* **Implementation**:
  * Computes total velocity magnitude and its derivatives with respect to each state variable.
  * Constructs 6×6 Jacobian using:
    * Newton’s laws in radial, polar, and azimuthal directions.
    * Trigonometric relationships (e.g., sinθ, cosθ, tanθ).
    * Drag components scaled by velocity derivatives.
  * Ensures safe division by small `sin²θ` using a lower-bound threshold.
* **Returns**: Full Jacobian matrix of the spherical dynamic model evaluated at state `x`.
