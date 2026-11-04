## Purpose

Render static circular-array variants without MATLAB, ITA Toolbox or RAVEN by computing image-source room impulse responses with Pyroomacoustics and synthesizing the multichannel recording in Python, while producing the same artifacts and attempt evidence that the existing static render pipeline validates, indexes and resumes.

## ADDED Requirements

### Requirement: Isolated worker process with the render entrypoint contract
Each Pyroomacoustics render attempt SHALL run as a separate operating-system process launched by the orchestrator through the existing per-variant render dispatch. The worker SHALL read only the attempt's immutable runtime-manifest snapshot and its attempt context: attempt identifier, variant identifier, manifest SHA-256, completion path, failure path and optional diagnostic events path. It SHALL write the WAV and metadata sidecar at `render.output_wav_path` and `render.output_metadata_path`.

The worker SHALL exit with code 0 only after every inventoried output and, when a completion path is provided, the completion record have been published. Any unhandled error SHALL produce a non-zero exit code, so the existing attempt, acceptance and retry machinery observes the same process semantics as for MATLAB. The orchestrator SHALL NOT render Pyroomacoustics variants inside its own process.

#### Scenario: Successful worker attempt
- **WHEN** the orchestrator dispatches a Pyroomacoustics array variant with a fresh attempt
- **THEN** a child process reads the attempt snapshot, writes the N-channel WAV, the sidecar and the completion record, and exits with code 0; the attempt record shows `process_status: succeeded`

#### Scenario: Worker crash
- **WHEN** the worker raises an unexpected exception during impulse-response computation
- **THEN** the process exits non-zero, no completion record is published, and the attempt is classified by the existing retry policy as for a MATLAB process failure

#### Scenario: Snapshot is the only input
- **WHEN** the runtime manifest file is modified after the attempt snapshot was taken
- **THEN** the worker renders from the snapshot and its completion record carries the snapshot's SHA-256

### Requirement: Existing MATLAB-process obligations apply to the worker
Where `matlab-process-retries`, `matlab-render-diagnostics` and `pipeline-progress` assign an obligation to the MATLAB process or entrypoint, the Pyroomacoustics worker SHALL fulfil the same obligation with the same record formats. Specifically:
- the worker SHALL publish the versioned deterministic-failure report with category `invalid_input` when its input manifest fails to load or validate;
- it SHALL emit the structured diagnostic events for the `initialization`, `simulation` and `artifact_writing` stages, the `artifact_written` events and the `entrypoint_completed` event;
- the orchestrator SHALL record per-invocation identity, command, timing, exit code and console log for worker invocations;
- `execution.matlab_process_retry`, the batch circuit breaker, persistent retry chains, remnant isolation and `reconcile-static` SHALL govern worker invocations unchanged;
- terminal progress SHALL report worker activity from its events.

Configuration keys, summary keys and directory names that contain `matlab` SHALL keep their current names.

#### Scenario: Invalid snapshot
- **WHEN** the worker's snapshot lacks the `circular_array` descriptor for an array variant
- **THEN** the worker publishes a `failure.json` with category `invalid_input` linked to the attempt and manifest hash, exits non-zero, and the attempt is classified as a deterministic failure that is not retried

#### Scenario: Retry budget shared with MATLAB semantics
- **WHEN** a worker invocation fails with a non-zero exit, observed termination and no accepted render
- **THEN** it consumes one invocation of the chain configured by `execution.matlab_process_retry` and counts toward the consecutive-failure circuit breaker

#### Scenario: Diagnostic event stream
- **WHEN** a worker attempt runs with diagnostics enabled
- **THEN** its events file contains, in sequence order, `stage_started`/`stage_completed` for `initialization`, `simulation` (with steps `pyroomacoustics_rir` and `audio_postprocessing`) and `artifact_writing`, `artifact_written` events for the WAV and the metadata, and finally `entrypoint_completed`

### Requirement: Explicit reversible coordinate transform
The backend SHALL convert public coordinates `(x, y, z)`, which have `y` vertical and are in meters, to backend coordinates by the rotation `(x, -z, y)` followed by a per-room translation `t = (-min x, max z, 0)` over the room footprint, so `(X, Y, Z) = (x - min x, max z - z, y)`. Backend coordinates SHALL be converted back with the inverse, `(x, y, z) = (X + min x, Z, max z - Y)`. The rotation SHALL be proper (determinant +1), the transform SHALL be rigid (no scaling), and the translation SHALL have no vertical component. The translation exists because the backend's rectangular room primitive is anchored at the backend origin; it does not change any distance or propagation time. The rotation matrix and the translation SHALL be recorded in render metadata.

All manifest coordinates, array descriptors and sidecar fields defined before this change SHALL remain in their existing frames. This includes the legacy source-pose representation in `summary.sources`. Backend-frame coordinates SHALL appear only in clearly labeled backend metadata.

#### Scenario: Round trip
- **WHEN** any finite public point is transformed to the backend frame of a room and back
- **THEN** the result equals the original within 1e-9 m

#### Scenario: Room anchored at the backend origin
- **WHEN** a shoebox room with a normalized public footprint is transformed
- **THEN** every transformed footprint vertex lies in the backend box `[0, length] x [0, width]`, and every in-room public point maps inside the backend room

#### Scenario: Vertical axis
- **WHEN** a microphone is at public height `y = 1.5`
- **THEN** its backend `Z` coordinate is 1.5 and the floor is at backend `Z = 0`

#### Scenario: Public metadata unchanged
- **WHEN** a Pyroomacoustics array variant is rendered
- **THEN** `receiver.coordinate_frame` is `public_xyz_y_up`, and the microphone positions in `receiver.circular_array` equal those of the runtime manifest

### Requirement: Room geometry construction and surface mapping
The backend SHALL construct the room from the manifest's `room.geometry`: the footprint vertices `[x, z]`, `height_m` and `wall_ids`. Shoebox rooms SHALL be built as rectangular boxes. Trapezoid and L-shape rooms SHALL be built by extruding the transformed footprint from the floor to `height_m`.

Footprint winding SHALL be normalized to the orientation the backend requires after transformation. Every wall SHALL receive the acoustic surface of the wall connecting the same two physical vertices in the public footprint. The floor and ceiling SHALL receive the `floor` and `ceiling` surfaces.

Any geometry type not supported before this change SHALL be rejected as invalid input.

#### Scenario: Winding reversal
- **WHEN** a counter-clockwise public footprint is transformed with `Y = -z`
- **THEN** the backend polygon is reordered to the required orientation, and the backend footprint area equals the public footprint area within 1e-9 m²

#### Scenario: Wall material identity
- **WHEN** an L-shaped room assigns a distinct material to each of its six walls
- **THEN** for every backend wall, the inverse-transformed midpoint of its footprint edge equals the midpoint of the public edge of the same `wall_id`, within 1e-9 m

#### Scenario: Unknown geometry type
- **WHEN** a snapshot contains a geometry type other than shoebox, trapezoid or L-shape
- **THEN** the worker publishes an `invalid_input` failure report

### Requirement: Explicit material conversion without equivalence claims
For each surface, the backend SHALL derive octave-band energy-absorption coefficients from the manifest's 31-band `effective_absorption`. It SHALL use the existing center-frequency mapping, which takes the third-octave band at each octave center, at the ten octave centers 31.5 Hz to 16 kHz. This is the mapping used by the Sabine estimator. Scattering coefficients SHALL be derived the same way from `effective_scattering`.

Render metadata SHALL record, per surface, the converted octave values and whether scattering influenced the simulation. Under ISM-only simulation, scattering SHALL be reported as not applied. Metadata and documentation SHALL NOT state that Pyroomacoustics and RAVEN produce acoustically equivalent rooms.

#### Scenario: Octave mapping
- **WHEN** a surface's 31-band absorption vector has index 17 (1000 Hz) equal to 0.12
- **THEN** the backend's 1000 Hz absorption coefficient for that surface is 0.12

#### Scenario: Scattering under ISM
- **WHEN** an ISM-only Pyroomacoustics variant is rendered
- **THEN** the metadata lists each surface's converted scattering values, with `scattering_applied: false`

#### Scenario: Out-of-range coefficient
- **WHEN** a converted absorption coefficient is outside [0, 1] or non-finite
- **THEN** the worker publishes an `invalid_input` failure report instead of clamping it

### Requirement: Image-source-only simulation with explicit parameters
The backend SHALL compute room impulse responses with the Image Source Method only. It SHALL use `pyroomacoustics.max_order` for shoebox rooms and `max_order_non_shoebox` for extruded rooms, the configured air-absorption flag, and the configured temperature and humidity when present (otherwise the backend's default propagation physics). Ray tracing and randomized image sources SHALL be disabled.

Render metadata SHALL record the effective ISM orders, the air-absorption flag, temperature, humidity, the effective speed of sound, the RIR high-pass filter settings and the fractional-delay length.

#### Scenario: Anechoic configuration
- **WHEN** `max_order` is 0 and the room is a shoebox
- **THEN** each microphone's impulse response contains only the direct-path contribution of each source

#### Scenario: Ray tracing never enabled
- **WHEN** any Pyroomacoustics variant is rendered
- **THEN** the metadata records `ray_tracing: false` and `randomized_image_sources: false`

### Requirement: Circular-array receiver rendering
For the `circular_array` variant, the backend SHALL place one omnidirectional microphone at each `receiver.circular_array.microphones[k].position_m` of the snapshot, after coordinate transformation, in increasing `microphone_index` order. It SHALL compute one impulse response per (source, microphone) pair. WAV channel `channel_index` SHALL carry microphone `channel_index - 1`.

The backend SHALL NOT recompute microphone positions from radius or yaw. It SHALL NOT align, normalize or beamform channels individually.

#### Scenario: Three-microphone array
- **WHEN** a snapshot describes a three-microphone array with radius 0.05 m
- **THEN** the backend renders three microphones at the three descriptor positions, and the WAV has three channels in descriptor order

#### Scenario: Channel ordering by arrival
- **WHEN** an anechoic scene places a single source 1 m from the array center along the public direction of microphone k
- **THEN** channel k+1 has the earliest direct-path arrival of all channels

### Requirement: Python signal synthesis reproducing the static array semantics
The backend SHALL synthesize each variant from per-source impulse responses with the following steps, in this order, at the render sample rate:
1. Load the prepared source WAV. Resample it with a polyphase filter when its rate differs from the render rate.
2. Downmix a multichannel source to mono by the arithmetic mean across channels.
3. Peak-normalize to 0.99, leaving an all-zero signal unchanged.
4. Apply `gain_db` as an amplitude factor `10^(gain_db/20)`.
5. Fully linearly convolve the source with each microphone response, giving `len_source + len_rir - 1` samples.
6. When `render.trim_reverb_tail` is true, drop `len_rir - 1` trailing samples, where `len_rir` is the source's longest microphone response.
7. Place the source at `max(0, round_half_away_from_zero(start_time_s * fs))` samples.
8. Sum all sources into a buffer whose length is the maximum end sample.
9. When `target_duration_s` is present, truncate or zero-pad to `max(1, round_half_away_from_zero(target_duration_s * fs))` samples.
10. If the mix peak exceeds 0.999, scale the mix to a peak of 0.999.

Sidecar source summaries SHALL record the original channel count and the `arithmetic_mean` or `none` downmix policy.

#### Scenario: Half-sample rounding
- **WHEN** `start_time_s * fs` equals 22050.5
- **THEN** the source starts at sample index 22051, not at the banker's-rounded 22050

#### Scenario: Stereo source
- **WHEN** a stereo source is rendered through the array
- **THEN** the mean of its two channels feeds every microphone response, and the sidecar records `original_num_channels: 2` and `downmix_policy: arithmetic_mean`

#### Scenario: Linear mixing
- **WHEN** two sources are rendered individually and together with identical parameters, and no peak guard or noise is triggered
- **THEN** the combined output equals the sum of the individual outputs within 1e-6 absolute

#### Scenario: Target duration
- **WHEN** `target_duration_s` is 2.0 at 44100 Hz
- **THEN** the WAV has exactly 88200 samples per channel

### Requirement: Background noise with existing semantics
When the snapshot enables background noise, the backend SHALL apply every configured layer after the first peak guard, following the existing semantics:
- **Seed:** each layer's seed SHALL be its explicit `seed`, otherwise `effective_seed + layer_index - 1`, otherwise `layer_index` (1-based).
- **Colored noise:** generated by spectral shaping of seeded Gaussian white noise with exponent 0 (white), -1 (pink) or -2 (brown), then zero-meaned and normalized to unit RMS.
- **Audio-file noise:** looped or trimmed to the output length.
- **Channels:** mono noise replicated across channels, an N-channel noise file applied channel to channel, and any other channel count rejected as invalid input.
- **Scaling:** each layer scaled independently against the pooled RMS of the pre-noise mix to its `snr_db`; a silent mix or a silent layer skips that layer with `silent_signal` or `silent_noise`.
- **Second peak guard:** after the layers are summed, a guard at 0.999 SHALL be applied and recorded.

Noise metadata SHALL contain the same top-level and per-layer keys as the existing sidecar. The backend's built-in noise and SNR facilities SHALL NOT be used. Bit-identical noise samples with the MATLAB renderer SHALL NOT be required.

#### Scenario: Pink noise at 10 dB SNR
- **WHEN** a mono pink layer with `snr_db: 10` is applied to an eight-channel mix
- **THEN** the measured ratio between the pooled mix RMS and the pooled added-noise RMS is 10 dB ± 0.01 dB before the second guard, and the layer metadata records `channel_mode: mono_replicated`

#### Scenario: Reproducible noise
- **WHEN** the same snapshot is rendered twice
- **THEN** the noise samples are identical in both outputs

#### Scenario: Incompatible noise channels
- **WHEN** a two-channel noise file is configured for an eight-channel array
- **THEN** the worker fails with an error naming the channel mismatch, and no WAV is accepted

### Requirement: WAV export
The backend SHALL write the output as a RIFF WAV file with N channels for N microphones, at the render sample rate, encoded as 16-bit PCM (the format produced by the existing MATLAB writer). It SHALL create missing parent directories, and SHALL write all samples before the metadata and completion record are published.

#### Scenario: Output format
- **WHEN** an eight-microphone variant completes
- **THEN** the WAV decodes fully as 8-channel, 44100 Hz, 16-bit PCM, with the sample count recorded in `summary.output_num_samples`

### Requirement: Sidecar contract compatible with existing validation
The sidecar SHALL contain every field that existing render validation requires for an array variant, with the existing semantics:
- `schema_version` copied from the manifest, `scene_id`, `job_id`, `scene_type`, `sample_rate_hz`, `output_type`, `project_name` and the output paths;
- `effective_seed` and `seed_source`;
- `receiver.circular_array` and `receiver.coordinate_frame`;
- in `summary`: the per-source summaries with the legacy pose representation, `output_num_samples`, `output_num_channels`, `n_sources`, `background_noise`, `room.reverberation` and `batch`.

In addition:
- `summary.render_backend` SHALL be `Pyroomacoustics`;
- a top-level `backend` object SHALL record the backend name, the package version, the simulation parameters, the coordinate transform, the converted materials, the reported RIR timing and the per-pair T30 values.

#### Scenario: Existing validator acceptance
- **WHEN** a Pyroomacoustics array variant is validated with the existing metadata, source, array, seed and noise checks
- **THEN** those checks pass without any exception specific to Pyroomacoustics, other than the backend-declared T30 descriptor and sample-rate set

#### Scenario: Backend block
- **WHEN** a Pyroomacoustics sidecar is read
- **THEN** `backend.name` is `pyroomacoustics`, `backend.version` equals the installed package version, and `backend.simulation.method` is `ism`

### Requirement: Backend-independent T30 measurement
The backend SHALL measure T30 from its own impulse responses at the ten octave centers 31.5 Hz to 16 kHz. It SHALL use octave-band filtering, Schroeder backward integration and a least-squares fit of the decay between -5 dB and -35 dB, extrapolated to 60 dB.

A (source, microphone) band whose decay does not reach -35 dB SHALL be invalid. A band's reported value SHALL be the arithmetic mean over all (source, microphone) pairs, and SHALL be invalid if any pair is invalid.

The reverberation block SHALL use `metric: T30`, `estimated_from: rir`, `method: schroeder_backward_integration_v1`, `band_resolution: octave`, `frequency_mapping: center_frequency`, and the same `band_frequencies_hz`, `valid_band_count`, `status` and `mean_t30_s` semantics as the existing block. Invalid bands SHALL be reported as null.

#### Scenario: Synthetic exponential decay
- **WHEN** the measurement is applied to a synthetic impulse response whose envelope decays with a known T60 of 0.6 s in every band
- **THEN** the measured T30 is 0.6 s ± 5% for bands from 125 Hz to 16 kHz, and ± 10% for 31.5 Hz and 63 Hz

#### Scenario: Truncated decay
- **WHEN** an impulse response's band decay reaches only -20 dB before it ends
- **THEN** that band is null, `status` is `unavailable`, and the existing canonical T30 gate rejects the scene and applies its existing resampling policy

#### Scenario: More absorption, shorter decay
- **WHEN** the same shoebox is rendered with uniform absorption 0.1 and then 0.4
- **THEN** every valid band's T30 is lower for absorption 0.4

### Requirement: Physically consistent propagation and reported timing
Render metadata SHALL report the global delay that the backend's fractional-delay filtering introduces into every impulse response, in samples. Under ISM, the direct-path peak of a source at distance `d` from a microphone SHALL occur at `d / c * fs` plus the reported global delay, within ±1 sample, where `c` is the recorded speed of sound.

For two microphones, the arrival-time difference SHALL match the geometric difference within ±1 sample. In an anechoic room, the direct-path energy ratio between distances `d1` and `d2` SHALL be `(d2/d1)^2` within ±5%.

#### Scenario: Direct-path arrival
- **WHEN** an anechoic shoebox renders one source 2.0 m from one microphone at 44100 Hz
- **THEN** the impulse-response peak index equals `round(2.0 / c * 44100) + global_delay` ± 1

#### Scenario: Inter-microphone delay
- **WHEN** an anechoic scene renders one source at different distances from two array microphones
- **THEN** the cross-correlation lag between the two channels equals the geometric delay difference in samples ± 1

### Requirement: Deterministic rendering
Two renders of identical snapshots, with the same backend package version and the same platform, SHALL produce byte-identical WAV files and sidecars that are equal as JSON values. The backend SHALL seed its package-level random state from the effective seed before simulation, and SHALL record that it did.

#### Scenario: Repeat render
- **WHEN** the same snapshot is rendered twice in separate worker processes
- **THEN** the two WAV files have equal SHA-256 hashes, and the two sidecars are equal JSON documents

### Requirement: Supported dependency range and version provenance
The backend SHALL support Pyroomacoustics only within the version range declared by the orchestrator's optional `pyroomacoustics` dependency extra. Installing the orchestrator without that extra SHALL keep the RAVEN backend fully usable. The installed backend version SHALL be recorded in every Pyroomacoustics sidecar.

#### Scenario: RAVEN-only installation
- **WHEN** the orchestrator is installed without the `pyroomacoustics` extra and runs a RAVEN experiment
- **THEN** no Pyroomacoustics import is attempted and the experiment behaves as before this change

### Requirement: Legacy circular-array angular convention preserved
The backend SHALL render the microphone positions exactly as described by the existing circular-array contract, including its angular convention, which places microphone 0 toward public +Z for receiver yaw +90°. A known risk SHALL be recorded and characterized by tests, not corrected here: the microphone ring's angular direction is opposite to the yaw/azimuth convention used for receiver view vectors and `fixed_position` source placement. Correcting that convention SHALL require a separate, explicitly authorized specification change.

#### Scenario: Characterized yaw behavior
- **WHEN** a four-microphone array with yaw +90° is rendered
- **THEN** microphone 0 is at public offset `[0, 0, +r]` from the center, while a `fixed_position` source at azimuth 0° with receiver yaw +90° lies toward public -Z
