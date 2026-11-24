# atomtronic-rotational-sensor
# Double-target BEC rotation sensor

Reproduction of Adeniji et al. 2024 (arXiv:2411.06585), up to Figs. 6 and 7, using a 2D rotating-frame Gross-Pitaevskii simulation in numpy/scipy.

## What the simulation does

Two ring-shaped traps (the "double target") hold a sodium BEC. We find the ground state, imprint a winding number of 1 on the top ring (the atoms flow around it), and then raise and lower a barrier between the rings (Fig. 4 schedule: 25 ms wait, 100 ms ramp up, 100 ms hold, 100 ms ramp down). If the barrier is low enough, or the frame rotates fast enough, the flow moves from the top ring to the bottom ring. We count the winding number of each ring at the end. The speed at which the flow moves is the rotation signal.

## Setup

```bash
cd atomtronic
python3 -m venv .venv
source .venv/bin/activate
python -m pip install numpy scipy matplotlib
```

Always run scripts with `python ...` inside the activated venv (the prompt shows `(.venv)`).

## Files

| File | Purpose |
| --- | --- |
| `dtbec.py` | Core module: parameters, potential, ground state, phase imprint, rotation, time evolution, winding number |
| `test_basic.py` | Sanity test: rotation direction, ground state, imprint, 25 ms settle. Saves `init_state.npy` |
| `stage1.py` | Stage 1: potential, ground state, imprinted state. Saves `init_state.npy` and 2 PNGs |
| `stage2_barrier.py` | Stage 2: barrier envelope and geometry |
| `stage3_winding.py`, `stage3_plot.py` | Stage 3: three full barrier cycles (Ub 47 or 48 nK, with and without Ω) and winding vs time |
| `stage4_run.py`, `stage4_plots.py` | Stage 4: 5 ms free expansion, the readout images |
| `fig6_scan.py`, `plot_fig6.py` | Stage 5: Fig. 6, final winding vs Ub |
| `fig7_scan.py`, `plot_fig7.py` | Stage 6: Fig. 7, final winding vs Ω for several Ub |

Run each stage's scripts in order. `init_state.npy` (from Stage 1) is needed by every later stage.

## Running the stages

```bash
python test_basic.py
python stage1_plots.py
python stage2_barrier.py
python stage3_winding.py && python stage3_plot.py
python stage4_run.py && python stage4_plots.py

# Stage 5 (Fig. 6): example coarse scan, then plot
python fig6_scan.py --start 40 --stop 60 --step 1 --out fig6_a.csv
python plot_fig6.py fig6_a.csv

# Stage 6 (Fig. 7)
python fig7_scan.py --ubs 46.9,46.7,46.5,46.3,46.1,45.9 \
    --omegas 0,0.02,0.05,0.08,0.1,0.15,0.2,0.3,0.4,0.6,1.0 --out fig7_coarse.csv
python plot_fig7.py fig7_coarse.csv
```
> p.s. haven't got the output yet, still trying!

## Key physics and numerics

- **Equation:** 2D GPE in the rotating frame (paper Eq. A7), Strang split-step Fourier. The −Ω·Lz term is applied as an exact 3-shear FFT rotation each time step.
- **Atoms:** ²³Na, N = 166,667. Ring radius 22.5 µm, ring width 5 µm, target spacing 10 µm, V0 = 209.2 nK, barrier width 11.25 µm.
- **Assumed values (not given in the paper):** a_s = 2.75 nm, ω_z = 2π × 500 Hz.
- **Grid:** 256 × 256 over 150 µm (dx = 0.586 µm). The healing length is about 0.45 µm, so the vortex core is barely resolved.
- **Time step:** dt = 1e-5 s. Do not go above this. A test with dt = 1e-4 gave nonsense windings.
- **Winding number:** wrapped phase differences around a circle of radius Rr on each ring centre. Only the final value at 300 ms counts. During the barrier hold the traces are noisy because the density at the neck is near zero.
- **Readout:** switch off all potentials and expand for 5 ms. Flow gives a dark vortex-core dot at the target centre. No flow gives a bright centre with concentric rings. The core score is min density within 1 µm of the centre divided by the mean within 5 µm (flow about 0.07–0.19, no flow about 1.2–1.4).

## Sign convention (important)

Ω > 0 means the frame rotates counter-clockwise. With the e^(+iφ) imprint on the top ring, only Ω > 0 moves the flow to the bottom ring. Ω < 0 does nothing. The effect is one-sided. The paper's wording ("flow opposite to frame rotation") does not match what works here, so trust the sign tested in `test_basic.py`.

## Results so far

| Quantity | This reproduction | Paper |
| --- | --- | --- |
| Ub,c (onset of transfer, Fig. 6) | 47.0–47.25 nK (dt = 1e-5) | 45.8 nK |
| Fig. 6: transfer range | transfer up to 53 nK, none from 54 nK | see paper |
| Stage 3 final windings (top, bottom) | Ub 47, Ω 0: (1, 0); Ub 47, Ω 0.1: (0, 1); Ub 48, Ω 0: (0, 1) | n/a |

Ub,c depends on numerics (Julia gave 45–46 nK, numpy gives about 47.1 nK), so choose the Fig. 7 values relative to your own Ub,c, not the paper's absolute numbers.

Fig. 7 test runs (final bottom-ring winding, 1 = transferred):

| Ub (nK) | Result |
| --- | --- |
| 47.0 | transfers at Ω about 0.1 |
| 46.3 | no transfer at 0.1, transfer at 0.2 and 0.3 |
| 45.8, 45.3 | no transfer up to Ω = 0.3 |
| 44.7 | no transfer up to Ω = 1.0 |
| 43.1, 41.6 | no transfer up to Ω = 2.5 |

Ω_c rises as Ub drops (the paper's trend), but much more steeply than the paper's 0.06–0.24 rad/s over about 5 nK. The useful Ub window is only about 1 nK wide here. The full Fig. 7 scan is still to be filled in.

## Known differences from the paper

- Absolute Ub,c and Ω_c do not match (numerics-sensitive, see above).
- Fig. 6 details after the onset (the re-entrant windows) differ from the paper.
- Eq. B4's Heaviside step looks like a typo in the paper. It is implemented as a half-plane split at y = ydt.
- Possible next checks: a finer grid (384 or 512), a smaller dt, applying Ω during the settle step, off-axis rotation.

## Scale of the signal (for reference)

The Sagnac phase is about 1.15 rad per (rad/s) per ring. The paper's Ω_c of 0.06–0.24 rad/s is roughly 1000–3000 times the Earth's rotation rate.

## Next steps (not started)

2 × 2 array (paper Fig. 9), CNN winding-number classifier trained on GPE-generated expansion images, and an optional miniaturization discussion.
