# Multiwavelength Temporal Variability & Bayesian Broadband SED Modeling Pipeline
**Candidate:** Joel Owusu Boateng Kwakye  
 

This notebook contains a unified computational pipeline developed for the multiwavelength temporal and spectral analysis of the intermediate synchrotron-peaked (ISP) blazar **AO 0235+164**. The codebase processes and models quasi-simultaneous data captured over an 8.5-year baseline (2008 to mid-2016) by the *Neil Gehrels Swift Observatory* (XRT/UVOT) and the *Fermi Gamma-ray Space Telescope* (LAT).

The architecture showcases advanced data engineering, frequency-domain time-series analysis, and statistical parameter estimation frameworks built using Python's scientific and astronomical software ecosystem (`pandas`, `numpy`, `matplotlib`, `astropy`, `emcee`, and `corner`).

---

##  Pipeline Architecture & Core Modules

### Module 1: Multiwavelength Temporal Alignment & Extinction Calibration
* **Data Synthesis:** Standardizes disparate time formats from three independent tracking instruments into standard Modified Julian Dates (MJD) using mission-specific reference offsets (e.g., Swift's $T_0 = 242031772.07373$ baseline configuration).
* **Detection Profiling:** Dynamically segregates statistically significant observations from non-detection upper limits across all instruments using strict signal-to-noise and Test Statistic thresholds ($\text{TS} \ge 4$ for *Fermi*-LAT and $\sigma \ge 3$ for Swift-XRT/UVOT).
* **State Visualization:** Employs advanced graphical layers (`axvspan` and `annotate`) to highlight complex, multi-day transient flaring structures (**F1–F4**) to evaluate co-spatial emission correlations within the relativistic jet plasma.

### Module 2: Irregularly Sampled Time-Series Analysis (Lomb-Scargle)
* **The Frequency Challenge:** Orbital visibility constraints, solar exclusion configurations, and satellite downtime introduce non-uniform sampling gaps into space-based datasets. This renders traditional Fast Fourier Transforms (FFTs) prone to severe spectral leakage artifacts.
* **The Solution:** Implements a normalized **Lomb-Scargle Periodogram** using a high-density, 10,000-step linear frequency mesh bounding physical variations between 0.5 and 3.0 years.
* **QPO Extraction:** Isolates the absolute peak spectral power during the source's designated quiescent state ($55151 \le \text{MJD} \le 56731$), resolving a clear dominant Quasi-Periodic Oscillation (QPO) at approximately **1.24 years**.

### Module 3: Bayesian Parameter Estimation (Markov Chain Monte Carlo)
* **The Physical Model:** Constructs a 6-parameter dual log-parabolic likelihood model tracking the low-energy synchrotron continuum and the high-energy Inverse Compton (IC) scattering peak.
* **Linear-Space Superposition:** Because individual components are mathematically defined in logarithmic flux spaces, the code dynamically maps them back to linear space ($10^{\log_{10}(\nu F_\nu)}$) to perform a physically valid summation before transforming the total continuum back to log space for statistical fitting.
* **Astrophysical Priors & Data Exclusion:** Imposes hard uniform prior bounds targeting an ISP blazar profile. Crucially, the script features a directive to filter out soft X-ray data points during key active states (such as the 2008 major flare) where separate bulk Compton scattering from cold electron populations distorts the non-thermal primary continuum.
* **Posterior Mapping:** Uses the affine-invariant ensemble sampler `emcee` over 5,000 steps with 64 parallel walkers. It discards the initial "burn-in" phase to generate high-resolution corner plots that map out 1D/2D parameter covariances and isolate precise asymmetric $1\sigma$ errors (16th, 50th, and 84th percentiles).

### Module 4: State-Dependent SED Evolution & Analytical Anchoring
* **Anti-Smearing Methodology:** Rather than blending multi-year data points into a single time-smeared spectrum, this module tracks true structural changes by calculating the statistical median flux for each instrument across discrete chronological epochs (the 2008 Major Flare, the 2009–2014 Quiescent State, and the 2014–2015 Flares).
* **Analytical Inversion:** Imports the baseline shape parameters (curvatures $b$ and peak frequencies $\nu_p$) derived from the MCMC chains. It analytically back-calculates the shifting peak flux parameters ($S_{\text{sync}}$ and $S_{\text{IC}}$) required to force the theoretical model curves exactly through the median empirical data points, revealing an order-of-magnitude scaling in the baseline synchrotron peak flux.


