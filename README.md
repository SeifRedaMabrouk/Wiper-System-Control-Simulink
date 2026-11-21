# Windshield Wiper Control System in MATLAB/Simulink

A Simulink implementation of a windshield wiper controller with **OFF, AUTO, LOW, and HIGH** operating modes. The system combines manual speed selection with rain-intensity-based automatic control, rain-sensor fault protection, smooth automatic duty-cycle transitions, and pulse-width modulation (PWM) for the motor command.

## Features

- Four operating modes: OFF, AUTO, LOW, and HIGH.
- Automatic adjustment of the PWM duty cycle according to rain intensity.
- Automatic-mode shutdown when a rain-sensor fault is detected.
- Fixed **40% duty cycle** in LOW mode and **70% duty cycle** in HIGH mode.
- Gradual duty-cycle changes in AUTO mode to avoid abrupt speed commands.
- Separate PWM motor command and wiper-active indication.
- A 20-second Simulink test scenario covering mode changes, changing rainfall, and a sensor fault.

## Inputs and Outputs

### Inputs

| Signal | Description | Values |
|---|---|---|
| `ModeOfWiperOperation` | Requested operating mode | `0` = OFF, `1` = AUTO, `2` = LOW, `3` = HIGH |
| `StatusOfRainSensor` | Rain-sensor fault flag | `0` = healthy, `1` = fault |
| `QuantitativeMeasureOfRainIntensity` | Measured rain intensity | 0–100 (assumed simulation scale) |

### Outputs

| Signal | Description |
|---|---|
| `SignalToWiperMotor` / `PWM` | Binary PWM signal (`0` or `1`) sent to the motor-control interface |
| `IndicationSignal` / `WiperActive` | `1` when the commanded duty cycle is greater than zero; otherwise `0` |

## Operating Logic

| Mode | Condition | Commanded PWM duty cycle |
|---|---|---|
| OFF (`0`) | Any sensor status | 0% |
| AUTO (`1`) | Sensor healthy (`0`) | Adjusts with rain intensity, using a gradual transition |
| AUTO (`1`) | Sensor fault (`1`) | 0% immediately |
| LOW (`2`) | Any sensor status | 40% |
| HIGH (`3`) | Any sensor status | 70% |

The rain-sensor fault affects **AUTO mode only**. Manual LOW and HIGH settings remain available regardless of sensor status.

## Control Architecture

```text
 Mode Selection ───────────────────────────────┐
                                               │
 Rain Intensity ──> Rain-to-Duty Conversion ──> Smooth AUTO Control
                                               │
 Rain Sensor Fault ──> AUTO Enable Logic ──────┤
                                               v
                                        Mode Selection
                                         /          \
                                        v            v
                                  Duty Cycle    Wiper Active
                                        |
                                        v
                              PWM Carrier + Comparator
                                        |
                                        v
                                 PWM Motor Signal
```

### 1. Rain-intensity mapping

Rain intensity is limited to the range 0–100. In AUTO mode, the target duty cycle is calculated as:

$$
D_{\text{target}} = 0.007\,\operatorname{clip}(R,0,100)
$$

where `R` is the rain-intensity input and `D_target` is the target duty-cycle fraction. This maps no rain to 0% and maximum modeled rainfall to 70%.

### 2. Smooth AUTO transitions

A discrete feedback loop limits how much the automatic duty cycle can change in one simulation step:

$$
D[k] = D[k-1] + \operatorname{clip}\!\left(D_{\text{target}}[k]-D[k-1],-0.0002,+0.0002\right)
$$

The sample time is **0.001 s**, so the maximum duty-cycle change is **20 percentage points per second**. When AUTO is disabled or a sensor fault occurs, the automatic duty-cycle command and its stored previous value are reset to zero. This allows a smooth restart after the fault clears.

### 3. PWM generation

A repeating carrier counts from `0` to `99` with a 1 ms sample period. The duty-cycle fraction is multiplied by `100`, and a comparator generates the PWM output:

```text
PWM = 1 when Carrier < (100 × DutyCycle)
PWM = 0 otherwise
```

For example, a 40% duty cycle produces 40 HIGH samples and 60 LOW samples per PWM period. With 100 samples per period, the demonstration PWM frequency is **10 Hz** and the duty-cycle resolution is **1 percentage point**.

The 10 Hz frequency is selected for simulation and visualization; it is **not a validated PWM frequency for a physical automotive wiper motor**.

## Simulation Configuration

| Parameter | Value |
|---|---|
| Simulation start time | `0 s` |
| Simulation stop time | `20 s` |
| Solver type | Fixed-step |
| Solver | Discrete (no continuous states) |
| Fixed-step size | `0.001 s` |
| PWM demonstration frequency | `10 Hz` |

### Test scenario

| Time interval | Mode | Rain / sensor test condition | Expected behavior |
|---|---|---|---|
| 0–2 s | OFF | — | 0% duty cycle |
| 2–3 s | AUTO | No rain | 0% duty cycle |
| 3–5 s | AUTO | Rain intensity 20 | Gradual rise toward 14% |
| 5–7 s | AUTO | Rain intensity 70 | Gradual rise toward 49% |
| 7–8 s | AUTO | Rain intensity 30 | Gradual decrease toward 21% |
| 8–9 s | AUTO | Sensor fault = 1 | Immediate shutdown (0%) |
| 9–10 s | AUTO | Sensor fault cleared | Gradual restart from 0% |
| 10–14 s | LOW | Any rain/sensor status | Fixed 40% |
| 14–18 s | HIGH | Any rain/sensor status | Fixed 70% |
| 18–20 s | OFF | — | 0% duty cycle |

The stated AUTO percentages are **target values**. The actual output approaches them over time because of the transition-rate limit.

## How to Run

**Requirements:** MATLAB and Simulink.

### Open the existing model

1. Download or clone the repository.
2. Open the `.slx` model file in MATLAB/Simulink.
3. Run the simulation for 20 seconds.
4. Inspect the PWM, wiper-active, and duty-cycle Scope signals.

### Alternatively, generate the model from the MATLAB script

If the repository includes `WiperSystem_DualMode_Build.m`, run this command from the folder containing the script:

```matlab
run('WiperSystem_DualMode_Build.m');
```

The build script creates `WiperSystem_DualMode.slx`, configures the simulation, and runs the test scenario. Run it in a folder that does not already contain a model with that name, because the script intentionally avoids overwriting an existing model.

## Verification Checklist

- OFF produces no PWM pulses and clears the active indication.
- AUTO follows changes in rain intensity without an abrupt duty-cycle jump.
- A sensor fault forces AUTO PWM to zero.
- Clearing the fault permits AUTO to restart gradually.
- LOW generates a 40% duty cycle.
- HIGH generates a 70% duty cycle.
- The active indication reflects the commanded nonzero duty cycle rather than individual PWM pulses.

## Design Scope

This is a **control-logic simulation**. The rain sensor is modeled using test signals rather than physical sensing hardware. A production vehicle implementation would require appropriate motor-driving electronics, validated PWM frequency, electrical protection, motor feedback, mechanical end-stop/park control, and hardware testing.

## Possible Extensions

- Intermittent wiping with an adjustable delay.
- Improved rain-sensor filtering and sensitivity settings.
- Motor position feedback and automatic park functionality.
- Hardware-oriented PWM and fault-diagnostic features.
