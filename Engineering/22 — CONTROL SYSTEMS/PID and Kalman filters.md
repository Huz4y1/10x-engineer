---
tags: [control, pid, kalman, robotics, reference]
---

# PID and Kalman filters

The two algorithms that make physical machines behave. Section: [[22 — CONTROL SYSTEMS]]

---

## Part 1 — PID control

### The idea in plain English

You want the motor at 100 rpm. It's at 80. What do you do?

- **Push harder the further you are from target** → that's **P**
- **If you've been below target for a while, push harder still** → that's **I**
- **If you're approaching fast, ease off so you don't overshoot** → that's **D**

Add all three together. That's a PID controller, and it runs most of the physical world.

### The formula

$$u(t) = K_p e(t) + K_i \int_0^t e(\tau)\,d\tau + K_d \frac{de(t)}{dt}$$

where $e(t) = \text{setpoint} - \text{measured}$.

### The code

```python
class PID:
    def __init__(self, kp, ki, kd, out_min=-1.0, out_max=1.0, i_limit=1.0):
        self.kp, self.ki, self.kd = kp, ki, kd
        self.out_min, self.out_max, self.i_limit = out_min, out_max, i_limit
        self.integral = 0.0
        self.prev_error = 0.0

    def update(self, setpoint, measured, dt):
        error = setpoint - measured

        p = self.kp * error

        self.integral += error * dt
        self.integral = max(-self.i_limit, min(self.i_limit, self.integral))  # anti-windup
        i = self.ki * self.integral

        d = self.kd * (error - self.prev_error) / dt if dt > 0 else 0.0
        self.prev_error = error

        return max(self.out_min, min(self.out_max, p + i + d))                # clamp output
```

### What each term does

| Term | Fixes | Too much causes |
|---|---|---|
| **P** | Slowness — bigger error, bigger push | Oscillation |
| **I** | **Steady-state error** — never quite reaching target | Overshoot, slow oscillation |
| **D** | Overshoot — damps the approach | Noise amplification, jitter |

> **Why you need I:** with P alone, as the error shrinks the push shrinks, so it settles *just below* target forever. Gravity, friction, or a constant load will always leave a gap. The integral accumulates that persistent gap until it's corrected.

### Tuning, in order

1. **Set `Ki = Kd = 0`.** Raise `Kp` until it oscillates steadily. **Halve it.**
2. **Raise `Ki`** until steady-state error disappears. Too much → slow oscillation.
3. **Raise `Kd`** to damp overshoot. Too much → it reacts to sensor noise.

**Ziegler-Nichols** as a starting point — find $K_u$ (the gain at sustained oscillation) and $T_u$ (its period):

| Controller | $K_p$ | $K_i$ | $K_d$ |
|---|---|---|---|
| P | $0.5 K_u$ | — | — |
| PI | $0.45 K_u$ | $0.54 K_u/T_u$ | — |
| **PID** | $0.6 K_u$ | $1.2 K_u/T_u$ | $0.075 K_u T_u$ |

Treat it as a first guess, not an answer.

### The three bugs everyone hits

**1. Integral windup.** The actuator saturates (motor already at 100%), the error persists, the integral keeps growing. When the load finally clears, that huge accumulated term slams the output and you overshoot wildly.

> **Fix: clamp the integral** (as in the code above), or stop accumulating while the output is saturated. **This is not optional** — it's the single most common PID failure in real hardware.

**2. Derivative kick.** A step change in setpoint makes $de/dt$ enormous for one sample, producing a violent spike.

> **Fix: differentiate the *measurement*, not the error.**
> ```python
> d = -self.kd * (measured - self.prev_measured) / dt
> ```

**3. Noise in the derivative.** `D` amplifies sensor noise. Low-pass filter the measurement first ([[Noise and Filtering]]), or keep `Kd` small.

### Cascade control

Real vehicles nest loops — each loop's output is the next loop's setpoint:

```
Position → [PID] → velocity setpoint → [PID] → attitude setpoint → [PID] → rate setpoint → [PID] → motors
   50 Hz                 100 Hz                     250 Hz                    1000 Hz
```

> **Tune from the inside out.** A badly tuned rate loop makes every outer loop impossible to tune, because they're all reacting to its misbehaviour ([[Project 010 — Advanced Aerospace Intelligent System]]).

### Discrete-time reality

Everything above assumes continuous time. Real controllers run at a fixed rate.

- **Sample rate should be ~10-20× the system's natural frequency.** Too slow and you can't control it; too fast wastes CPU and amplifies noise.
- **`dt` must be consistent.** Variable timing changes your effective gains. Use a hardware timer, not `time.time()` in a loop that does other work ([[Non-blocking timing]]).

---

## Part 2 — Kalman filters

### The idea in plain English

You have two sources telling you where you are, and both are wrong.

- **Encoders** say "you moved 2.1m." Smooth and fast, but wheels slip, so error **accumulates forever**.
- **GPS** says "you're at position X." Doesn't drift, but noisy and slow.

A Kalman filter combines them — trusting each **in proportion to how certain it is** — and produces a better estimate than either alone.

> **The core insight: track not just your best guess, but how uncertain you are about it.** When a confident measurement arrives, trust it. When an uncertain one arrives, mostly ignore it. That's the whole algorithm.

### The two steps

**Predict** — use your model of how things move:

$$\hat{x}_k^- = F \hat{x}_{k-1} + B u_k \quad\quad P_k^- = F P_{k-1} F^T + Q$$

*Uncertainty always grows during prediction* (that `+ Q`).

**Update** — a measurement arrives:

$$K_k = P_k^- H^T (H P_k^- H^T + R)^{-1}$$
$$\hat{x}_k = \hat{x}_k^- + K_k(z_k - H\hat{x}_k^-) \quad\quad P_k = (I - K_k H)P_k^-$$

*Uncertainty always shrinks during update.*

| Symbol | Meaning |
|---|---|
| $x$ | State estimate (position, velocity...) |
| $P$ | **Uncertainty** (covariance) of that estimate |
| $F$ | How the state evolves (motion model) |
| $Q$ | **Process noise** — how wrong your model is |
| $z$ | The measurement |
| $H$ | Maps state → what the sensor measures |
| $R$ | **Measurement noise** — how wrong your sensor is |
| $K$ | **Kalman gain** — how much to trust this measurement |

### The Kalman gain is the whole thing

$$K = \frac{\text{my uncertainty}}{\text{my uncertainty} + \text{sensor uncertainty}}$$

- Sensor very noisy ($R$ large) → $K \to 0$ → **ignore it**, trust the prediction
- Sensor very good ($R$ small) → $K \to 1$ → **trust it**, discard the prediction

It's a weighted average that reweights itself every step.

### 1-D example — you can implement this today

```python
class Kalman1D:
    def __init__(self, x0, p0, q, r):
        self.x, self.p, self.q, self.r = x0, p0, q, r

    def predict(self, u=0.0, dt=1.0):
        self.x += u * dt          # motion model
        self.p += self.q          # uncertainty GROWS
        return self.x

    def update(self, z):
        k = self.p / (self.p + self.r)      # Kalman gain
        self.x += k * (z - self.x)          # correct toward the measurement
        self.p *= (1 - k)                   # uncertainty SHRINKS
        return self.x

kf = Kalman1D(x0=0.0, p0=1.0, q=0.01, r=0.5)
for measurement in sensor_stream:
    kf.predict(u=commanded_velocity, dt=0.02)
    estimate = kf.update(measurement)
```

> **Q and R are the tuning knobs, and they're a ratio.** Large `Q` = "my model is bad, trust sensors." Large `R` = "my sensors are noisy, trust the model." Getting the *ratio* right matters far more than the absolute values.

### The variants

| Filter | For |
|---|---|
| **Kalman (KF)** | Linear systems, Gaussian noise |
| **Extended KF (EKF)** | Non-linear — linearises at each step. **The robotics default.** |
| **Unscented KF (UKF)** | Strongly non-linear — samples instead of linearising |
| **Particle filter** | Very non-linear, multi-modal (could be in several places) |

> Robots use the **EKF** because rotation is non-linear. `robot_localization` in ROS 2 gives you one configured for IMU + odometry + GPS ([[21 — ROBOTICS]]).

### Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `Q` too small | Filter ignores reality, diverges | Increase `Q` |
| `R` too small | Estimate is as noisy as the raw sensor | Increase `R` |
| Wrong initial `P` | Slow convergence, or wild early jumps | Start `P` large if unsure |
| Uneven `dt` | Wrong prediction step | Measure actual `dt` |
| Sensors on different clocks | Fusing stale data as fresh | Timestamp everything at source |
| Forgetting to predict between updates | Estimate lags reality | Always predict, then update |

---

## Where these appear

| Algorithm | Appears in |
|---|---|
| **PID** | Motor speed, drone attitude, temperature, cruise control |
| **Cascade PID** | Flight controllers ([[Project 010 — Advanced Aerospace Intelligent System]]) |
| **Kalman** | Robot localisation, IMU fusion, GPS smoothing, sensor calibration |
| **EKF** | ROS 2 `robot_localization` ([[Project 009 — Autonomous Robotics System]]) |

> **Both are older, cheaper and more provable than reinforcement learning.** Learn these before reaching for RL ([[12 — REINFORCEMENT LEARNING]]) — most "we need ML for control" turns out to be a badly tuned PID.

## Related

[[22 — CONTROL SYSTEMS]] · [[21 — ROBOTICS]] · [[Feedback and PID control]] · [[Noise and Filtering]] · [[20 — EMBEDDED]] · [[Mathematics reference]] · [[23 — AEROSPACE]]
