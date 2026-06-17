This project implements a physics-based ultrasound simulation pipeline utilizing the native UltraRay API.
It is designed as a high-fidelity, high-performance substitute for standard PyMUST simulations by substituting geometric and acoustic approximations with precise optical and acoustic ray-tracing mechanics.
Because the underlying scene management system Resolves spatial geometries and structural configurations via relative pathways, the entire simulation workflow is designed to execute directly out of the root ~/UltraRay/ workspace directory.
The pipeline initializes by configuring path variable overrides and establishing a dedicated computation environment.
To achieve high-efficiency data rendering across multiple channels, the system sets the vectorization and rendering backend using Mitsuba 3 configured to its standalone LLVM CPU variant (llvm_mono), which automatically registers the necessary ultra_ray plugins. Following environment declaration, systemic hardware and media parameters are mapped inside an execution namespace. These mirror a physical linear array probe setup (resembling standard L11-5V configuration metrics), defining a 128-element transducer, specific angular opening apertures, a 50 MHz high-frequency data acquisition sampling rate, 25 discrete plane wave steering angles, and custom medium propagation factors such as an explicit speed of sound ($1540.0 \text{ m/s}$) and attenuation scaling variables.Once parameter blocks are established, the system leverages a core structural layer (SceneLoader) to parse and construct the simulation's physical boundary environment. It ingests target object geometries—such as standardized cylindrical mesh boundaries—and overlays the corresponding structural transducer array file onto the scene layout. This spatial data integration handles the setup for transmit beamforming. Instead of using simplified mathematical grid approximations found in legacy MATLAB toolboxes, the system computes plane wave steering delays directly relative to the synthetic boundaries of the loaded object coordinates, preparing the integrator to perform complex Monte Carlo ray evaluations across more than 100,000 spatial samples.The final segment of the notebook handles system verification and array diagnostics before finalizing data generation. It uses specialized inspection queries to loop through active sensor properties, confirming essential operational criteria like axial resolution, film matrix crop sizes ($128 \times 1$), and maximum scanning distances. It also includes diagnostic handles to track individual elements and normal vector orientations. This ensures the primary acoustic propagation component aligns seamlessly along the depth axis and does not experience spatial matrix skews, paving the way for the successful generation of clean Radio Frequency (RF) and Delay-and-Sum (DAS) beamformed B-mode images.
## PyMUST vs. UltraRay — Simulation Parameter Comparison

Both notebooks simulate the same convex-array probe imaging the same `cylinder.obj` phantom, but through two fundamentally different forward models: PyMUST is a deterministic point-scatterer wave simulator, while UltraRay is a Monte Carlo ray/path tracer (Mitsuba-based). Parameters describing the physical acquisition were deliberately matched for a fair comparison; parameters specific to *how* each method computes the result necessarily differ, since they aren't the same kind of knob.

### Transducer geometry

| Parameter | PyMUST | UltraRay | Same? | Notes |
|---|---|---|---|---|
| Elements (lateral) | 128 | 128 | ✅ Same | |
| Array radius | 30 mm (convex) | 30 mm (convex) | ✅ Same | |
| Opening angle | 70° | 70° | ✅ Same | Element pitch derived from this: 70° / 127 gaps × 30 mm ≈ 0.289 mm |
| Element height (elevation) | 4 mm | 4 mm | ✅ Same | |
| Elevation elements | n/a (2-D simulator) | 1 | ⚠️ Different paradigm | UltraRay ray-traces in full 3-D; PyMUST only ever simulates the single imaging plane |

### Acquisition / pulse

| Parameter | PyMUST | UltraRay | Same? | Notes |
|---|---|---|---|---|
| Centre frequency | 5 MHz | 5 MHz | ✅ Same | |
| Sampling frequency | 50 MHz | 50 MHz | ✅ Same | |
| Pulse length | 3 cycles | 3 cycles | ✅ Same | |
| Speed of sound | 1540 m/s | 1540 m/s | ✅ Same | |
| Attenuation coefficient | 0.1 | 0.1 | ✅ Same (value) | Same number is used by both, but it feeds a wave-equation attenuation term in PyMUST vs. a ray/path attenuation term in UltraRay — not guaranteed to be physically identical despite the matching value |
| Fractional bandwidth | 70% | — | ⚠️ Not directly comparable | PyMUST needs an explicit transducer bandwidth for its pulse-echo response; UltraRay has no equivalent exposed parameter, so 70% was a representative placeholder with no UltraRay value to match |

### Transmit sequence (plane-wave compounding)

| Parameter | PyMUST | UltraRay | Same? | Notes |
|---|---|---|---|---|
| Number of steered transmits | 25 | 25 | ✅ Same | |
| Steering angle range | -30° to +30° | -30° to +30° | ✅ Same | |
| Transmit apodization | Hann taper across the aperture | Not exposed in the config (none confirmed) | ❓ Unconfirmed | Added on the PyMUST side specifically to suppress near-field diffraction fringes from firing a hard-edged, coarse-pitch aperture; whether UltraRay's renderer needs/uses an equivalent isn't visible from its argument list |
| Compounding weights across angles | Hann-weighted | Not shown (handled inside `beamform()`) | ❓ Unconfirmed | |

### Forward simulation method

| Parameter | PyMUST | UltraRay | Same? | Notes |
|---|---|---|---|---|
| Underlying physics | Deterministic linear-acoustics wave simulation (`simus`) | Stochastic Monte Carlo ray/path tracing (Mitsuba) | ❌ Fundamentally different | This is the core methodological difference the whole comparison is built around |
| Object representation | Discrete point-scatterer cloud sampled from the mesh's cross-section | Continuous 3-D triangle mesh, intersected directly by rays | ❌ Different | PyMUST never sees the mesh surface itself, only points sampled off it |
| Rays / samples per pixel | n/a | 100,000 | ❌ No PyMUST equivalent | Pure ray-tracing concept |
| Max ray bounce depth | n/a | 10 | ❌ No PyMUST equivalent | |
| Max ray travel distance | n/a | 0.20 m | ❌ No PyMUST equivalent | |
| Temporal binning filter | n/a | `box`, std 2.0 | ❌ No PyMUST equivalent | How UltraRay bins traced ray energy into time samples; PyMUST's RF comes directly from its analytic model |

### Beamforming / reconstruction

| Parameter | PyMUST | UltraRay | Same? | Notes |
|---|---|---|---|---|
| Beamforming method | Delay-and-sum (`dasmtx`) | Delay-and-sum (`ultra_ray` beamformer) | ✅ Same algorithm | Same principle, different implementations |
| Receive f-number | 2.5 | 28 | ❌ Different (intentional) | UltraRay's high f-number (narrow receive aperture) suppresses Monte Carlo ray-tracing noise; PyMUST's much lower f-number is instead tuned to control sidelobe/grating energy from an array pitch coarser than λ/2 — different problems, hence different values |
| Reconstruction grid resolution | 420 × 320 px, manually set window | ~3426 × 4845 px, generated internally | ❌ Different | Not matched; UltraRay's grid is considerably denser and auto-sized |

### Display

| Parameter | PyMUST | UltraRay | Same? | Notes |
|---|---|---|---|---|
| Dynamic range | 60 dB | 60 dB | ✅ Same | |
| Colormap | Grayscale | Grayscale | ✅ Same | |
| Depth orientation | Top = shallow | Top = shallow | ✅ Same | |
