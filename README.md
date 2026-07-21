# Automated Differential ODMR Mapping of Zeeman-Split NV Ensembles

Automated optically detected magnetic resonance (ODMR) measurements of a **single microdiamond containing an ensemble of nitrogen-vacancy (NV) centers**. The system combines optical excitation and fluorescence readout with programmable microwave sweeps, differential microwave-on/off acquisition, and motorized permanent-magnet positioning.

This project was developed by **Ercihan Kaya** at the **ZHAW School of Engineering** as part of the Master of Science in Engineering program, supervised by **Dr. Wolf Wüster**.

## Project status

The current implementation demonstrates:

- Automated ODMR frequency sweeps around the NV zero-field splitting near 2.870 GHz.
- Differential microwave-on/off fluorescence acquisition.
- Motorized magnet-position scans using a Thorlabs KDC101 / T-Cube DC Servo stage.
- Per-position Lorentzian fitting of up to four symmetric resonance pairs.
- Position-resolved two-dimensional ODMR map generation.
- Machine-readable CSV, JSON, and pickle outputs with diagnostic plots.

The reported long-duration scan was interrupted by a laser failure. Post-processing reconstructed the reported map from **24 valid spectra out of 27 recorded candidate spectra**.

## Scientific scope

The experiment addresses one physical microdiamond, but the optical signal is an **ensemble, multi-photon fluorescence signal** from many NV centers inside that particle. A microdiamond may contain all four crystallographic NV orientations, with two spin transitions per orientation, so the ODMR spectrum can contain up to eight resonance lines.

The differential contrast is calculated as

```text
contrast [%] = 100 × (V_on - V_off) / V_off
```

With this sign convention, ODMR resonances appear as negative contrast features.

For the symmetric Lorentzian-pair model, an effective NV-axis magnetic-field projection is estimated from the fitted half-splitting `d`:

```text
B_parallel [mT] = 1000 × d [GHz] / 28
```

This value is an effective field projection along an NV axis. Absolute magnetic-field calibration still requires a Hall-sensor measurement or a calibrated stage-position-to-field map.

## Hardware

| Function | Component |
|---|---|
| Optical excitation | 532 nm laser, mirrors, multimode fiber, collimation and focusing optics |
| Sample | Single microdiamond containing an NV ensemble |
| Microwave delivery | Microstrip PCB and RF cable |
| Microwave source | Windfreak SynthHD Mini |
| Optical filtering | Green-blocking / red-transmitting filter path |
| Photodetection | FEMTO OE-200-SI/FS photoreceiver |
| Data acquisition | NI USB-6251 with BNC-2120 interface |
| Magnetic bias | Permanent magnet on translation stage |
| Stage controller | Thorlabs KDC101 / T-Cube DC Servo |

## Software requirements

The measurement software is intended for a Windows laboratory computer with:

- Python 3 (the exact tested version is not specified in the current documentation).
- Thorlabs Kinesis installed.
- NI-DAQmx drivers installed and the NI device visible to the system.
- USB serial access to the Windfreak SynthHD Mini.

Python dependencies used by the acquisition script:

```bash
pip install pythonnet pyserial numpy pandas matplotlib scipy nidaqmx
```

Dependency versions are not pinned in the current work.

## Default dense-scan parameters

| Parameter | Default value |
|---|---:|
| Center frequency | 2.870 GHz |
| Sweep range | 2.770–2.970 GHz |
| Frequency points | 201 |
| Nominal frequency step | 1 MHz |
| RF set point | 15 dBm |
| RF switching | Mute/unmute with PLL kept active |
| Magnet positions | 31 positions from 0 to 30 mm |
| Position step | 1 mm |
| Stage settling time | 10 s |
| On/off modulation frequency | 0.5 Hz |
| Microwave-on duration | 1 s |
| Microwave-off duration | 1 s |
| Repetitions per frequency | 20 cycles |
| Integration per frequency | 40 s |
| DAQ channel | `Dev1/ai0` |
| DAQ mode | Differential |
| DAQ input range | ±0.5 V |
| DAQ sampling rate | 20 ksample/s |

## Configuration

Review the user-parameter section near the top of the acquisition script before each run.

### Stage and magnet

```python
STAGE_ENABLED = True
SERIAL_NO = "83833290"
KINESIS_PATH = r"C:\Program Files\Thorlabs\Kinesis"
MAGNET_POSITIONS_MM = np.linspace(0.0, 30.0, 31).tolist()
HOME_STAGE_BEFORE_SEQUENCE = True
MAGNET_SETTLE_S = 10.0
STAGE_MIN_MM = 0.0
STAGE_MAX_MM = 50.0
```

Verify the controller serial number, mechanical travel limits, stage orientation, homing behavior, velocity, acceleration, and the safe magnet-to-sample distance before enabling movement.

### Microwave source

```python
WINDFREAK_PORT = None # Auto-detect, or set COM port manually
CENTER_FREQ_GHZ = 2.870
SPAN_GHZ = 0.200
RF_POWER_DBM = 15.0
RF_SWITCH_MODE = "mute"
```

The configured generator power is not the same as the microwave power delivered at the microdiamond. Cable loss, impedance matching, PCB geometry, and local coupling must be calibrated separately.

### NI-DAQ acquisition

```python
DAQ_DEVICE = "Dev1"
DAQ_SAMPLE_RATE_HZ = 20_000
DAQ_INPUT_RANGE_V = (-0.5, 0.5)
N_FREQ_POINTS = 201
N_CYCLES_PER_FREQ = 20
MODULATION_FREQ_HZ = 0.5
```

Confirm the physical channel assignment and input range with NI Measurement & Automation Explorer before running the full experiment.

## Running the acquisition

1. Confirm laser alignment and that the filtered red/orange fluorescence is visible at the selected microdiamond.
2. Verify that the RF output, microwave PCB, photoreceiver, NI-DAQ, and stage are connected.
3. Close vendor monitoring tools that may reserve the USB devices.
4. Check all configuration constants in the acquisition script.
5. Start with a short test scan using fewer positions, frequency points, and cycles.
6. Run the dense acquisition:

```powershell
python code\odmr_windfreak_kdc101_stage_dense_2d_map.py
```

During the magnet-settling wait, pressing `x` requests a controlled stop before the next measurement. `Ctrl+C` is also handled and attempts to stop the stage safely, mute the RF output, and close device connections.

## Measurement sequence

For each programmed magnet position, the script:

1. Mutes the microwave output.
2. Moves the stage to the absolute target coordinate.
3. Waits for mechanical and magnetic settling.
4. Sweeps all programmed microwave frequencies.
5. Waits for PLL lock at each point.
6. Alternates microwave-on and microwave-off acquisition blocks.
7. Averages the repeated measurements.
8. Computes differential voltage and normalized contrast.
9. Fits four symmetric Lorentzian resonance pairs.
10. Saves per-position data, metadata, and plots.
11. Builds aggregate comparison products and a final 2D ODMR map.

## Output data

The default output directory is timestamped:

```text
Data/odmr_kdc101_magnet_sweep_<YYYYMMDD_HHMMSS>/
```

Typical per-position outputs include:

```text
*_odmr_metadata.json
*_odmr_differential_spectrum.csv
*_odmr_differential_cycles.csv
*_lorentz_fit_parameters.csv
*_measurement_result.pkl
*_odmr_on_off.png
*_odmr_diff.png
*_odmr_contrast.png
*_odmr_lorentz_fit.png
*_odmr_drift_check.png
```

Typical aggregate outputs include:

```text
magnet_stage_positions.csv
windfreak_stage_sweep_summary.csv
windfreak_stage_sweep_results.pkl
windfreak_stage_sweep_contrast_comparison.png
windfreak_stage_sweep_diff_comparison.png
windfreak_stage_sweep_fit_Bfield_vs_position.png
windfreak_stage_sweep_odmr_2d_map.png
```

## Interpreting the results

- A negative feature in the contrast spectrum indicates microwave-induced fluorescence reduction.
- Resonance positions moving with magnet-stage position are the experimental signature of Zeeman splitting.
- Multiple branches are expected from the four possible NV orientations and two transitions per orientation.
- The stage coordinate is a reproducible control variable, not an absolute magnetic-field value.
- The fitted `B_parallel` values are model-dependent effective NV-axis projections.
- Display interpolation in the 2D map smooths the rendered image but does not alter the saved raw data.

## Troubleshooting

### USB device is unavailable at startup

The observed initialization failures were consistent with another process reserving a USB-controlled device.

Recommended recovery procedure:

1. Stop the Python measurement process.
2. Close NI Device Monitor and other vendor utilities that may hold the DAQ, stage, or serial device.
3. Disconnect the affected USB device and, where applicable, remove its power.
4. Wait briefly, then reconnect power and USB.
5. Verify the device in its vendor software.
6. Close the vendor software before restarting the Python script.

### Kinesis DLLs cannot be loaded

Confirm that:

- Thorlabs Kinesis is installed.
- `KINESIS_PATH` points to the actual installation directory.
- Python and the installed Kinesis components use compatible architectures.
- `pythonnet` is installed in the active environment.

### NI-DAQ initialization fails

Confirm that:

- NI-DAQmx is installed.
- The device name matches `DAQ_DEVICE`.
- `ai0` is available and wired correctly.
- No other process has reserved the device.
- The expected voltage remains within the configured ±0.5 V range.

### Lorentzian fit does not converge

Possible causes include low ODMR contrast, optical drift, an overly broad sweep, resonance overlap, insufficient frequency resolution, or unsuitable initial bounds. Keep the raw spectrum even when fitting fails and review the data before changing the model.

## Known limitations and next steps

- Absolute magnetic-field calibration is not yet integrated.
- The current field axis is the mechanical stage position.
- RF power at the microstrip has not been fully calibrated.
- Laser power and FEMTO gain should be logged explicitly.
- Guard intervals after RF switching may improve measurement robustness.
- Repeated spectra are required for statistical uncertainty estimates.
- Narrower or adaptive sweeps could reduce acquisition time.
- The four-pair symmetric Lorentzian model is a diagnostic approximation.
- Photon-counting hardware is a future extension for subsequent quantum-random-number experiments.

## Safety

This experiment involves laser radiation, microwave RF energy, strong permanent magnets, moving mechanical hardware, and sensitive electronics.

- Follow the laboratory laser-safety classification and eyewear requirements.
- Enclose or terminate RF paths appropriately and avoid touching energized RF hardware.
- Keep magnetic-sensitive objects, tools, storage media, and medical implants away from the permanent magnet.
- Define and test stage travel limits before unattended movement.
- Provide an accessible emergency stop and do not leave a new configuration unattended.
- Power down equipment before changing optical, RF, or DAQ wiring.

## Acknowledgements

The work was supervised by Dr. Wolf Wüster. Its experimental structure and physical framing build on the modular NV-center setup reported by Stegemann et al.
