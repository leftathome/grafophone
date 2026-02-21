# Grafophone Implementation Plan

Tasks ordered by MVP priority: **remote data sources → OSC/MIDI** first, then DAW plugin, then hardware last. Each task is a single PR-sized unit of work with red-to-green TDD and code review before merge.

**Task status:** `[ ]` not started · `[~]` in progress · `[x]` complete

---

## Milestone 1: Foundation

_Produces: A Go binary that reads a YAML config, connects to Prometheus, and sends normalized values via OSC._

Ref: [DESIGN.md — Software Architecture](DESIGN.md#software-architecture), [SPECS.md — CI/CD Pipeline](SPECS.md#cicd-pipeline-specifications)

### 1.1 Go module and project skeleton
`[ ]` Initialize `go.mod`, create directory tree (`cmd/grafophone/`, `internal/{engine,mapper,buffer,input,output,config}/`), add empty `main.go` with `--version` flag, `Makefile` with `lint`, `test`, `build` targets, `.golangci.yml` from SPECS.md.

**Exit criteria:** `make lint`, `make test`, `make build` all exit 0. `./grafophone --version` prints `grafophone dev (commit <hash>, built <date>)`.

### 1.2 InputSource interface and core types
`[ ]` Define `InputSource` interface, `SourceConfig`, `ConnectionState`, `FilterType` in `internal/input/source.go`. Define `OutputConfig` types in `internal/output/output.go`.

**Exit criteria:** Types compile. Hand-written mocks implement all interfaces and pass compile-time assertions (`var _ InputSource = (*MockSource)(nil)`).

### 1.3 Ring buffer
`[ ]` Implement ring buffer in `internal/buffer/ring.go` with configurable depth (power-of-2, 256–4096).

**Exit criteria:** Table-driven tests cover: write/read, wrap-around, len/cap, forward playback, reverse playback, ping-pong playback, reset. Benchmark for hot-path write.

### 1.4 Fixed-range normalization
`[ ]` Implement `Normalize(raw, min, max float64) float64` in `internal/input/normalize.go`. Clamp to 0.0–1.0.

**Exit criteria:** Table-driven tests for: mid-range, at-bounds, below-min, above-max, min==max (returns 0.0), negative range. Fuzz target `FuzzNormalize`.

### 1.5 Structured logging with redaction
`[ ]` Set up `slog` structured logger in `internal/log/`. Add redaction: any field containing `password`, `token`, `key`, or `secret` is replaced with `[REDACTED]` in output.

**Exit criteria:** Tests verify: log output is structured JSON, sensitive fields are redacted, non-sensitive fields pass through. Test that `connection: "postgres://user:s3cret@host"` is redacted.

### 1.6 YAML config parsing
`[ ]` Implement config struct definitions and YAML deserialization in `internal/config/config.go`. Parse both `sources:` and `outputs:` sections. Fuzz target `FuzzConfigParse`.

**Exit criteria:** Tests parse example YAML from DESIGN.md into Go structs. Tests for: valid config round-trips, malformed YAML returns error, unknown fields are rejected.

### 1.7 Config env-var expansion and validation
`[ ]` Add `${ENV_VAR}` substitution to config loader. Reject empty env vars (fail-closed). Validate required fields per source type.

**Exit criteria:** Tests for: env var expanded correctly, empty env var returns error, missing required field returns error (e.g. Prometheus source without `endpoint`), extra env var syntax in non-string fields returns error.

### 1.8 Output interfaces
`[ ]` Define `MIDIOutput` interface (NoteOn, NoteOff, CC, PitchBend, Clock, Start, Stop) and `OSCOutput` interface (Send, SendBundle, Close) in `internal/output/`. Verify method signatures match DESIGN.md.

**Exit criteria:** Interfaces compile. Mock implementations (`midi_mock.go`, `osc_mock.go`) satisfy the interfaces and record all calls for test assertions.

### 1.9 OSC output
`[ ]` Implement `OSCOutput` using `hypebeast/go-osc` in `internal/output/osc.go`. Send float values to configurable UDP address/port with configurable OSC address pattern.

**Exit criteria:** Unit test starts a local UDP listener, sends OSC messages via the output, reads and decodes them, asserts address pattern and float value match. Test covers both `Send` and `SendBundle`.

### 1.10 Prometheus adapter
`[ ]` Implement Prometheus data source in `internal/input/datasource/prometheus.go`. Poll PromQL instant query endpoint, extract scalar value from response JSON, normalize via fixed-range.

**Exit criteria:** Unit tests use `httptest.Server` returning canned Prometheus API JSON. Tests cover: successful query + value extraction, normalization, HTTP error → hold last value, timeout → hold last value. Adapter satisfies `InputSource` interface.

### 1.11 Reconnection/backoff utility
`[ ]` Implement shared exponential backoff in `internal/input/backoff.go`. Backoff sequence: 1s, 2s, 4s, 8s, max 60s. Reset on successful connection.

**Exit criteria:** Tests verify: backoff sequence is correct, max cap is respected, reset returns to 1s. Used by Prometheus adapter (and all future adapters).

### 1.12 CLI entry point — poll loop wiring
`[ ]` Wire up `cmd/grafophone/main.go`: parse `--config` flag, load config, construct source(s) and output(s), run poll loop. Basic context-cancellation shutdown on SIGINT/SIGTERM (comprehensive shutdown deferred to 5.4).

**Exit criteria:** In-process test with mock source + mock output: config loads, poll loop runs, mock output receives normalized values. Context cancellation stops the loop cleanly.

### 1.13 Integration test — binary end-to-end
`[ ]` End-to-end test: build binary, start it with a test config pointing at an `httptest.Server` (Prometheus) and a local UDP OSC listener. Assert OSC messages arrive with correct values.

**Exit criteria:** Test starts binary as subprocess, waits for first OSC message, verifies value, sends SIGINT, verifies clean exit (exit code 0).

### 1.14 CI pipeline
`[ ]` Add `.github/workflows/ci.yml` with lint, test, and build jobs per SPECS.md. Add `.github/workflows/release.yml` stub. Validate with `actionlint`.

**Exit criteria:** `actionlint` passes locally. `.golangci.yml` matches SPECS.md config exactly. Push to `claude/*` branch triggers lint + test + build; all three jobs pass.

### 1.15 Example config files
`[ ]` Create `configs/default.yml` (minimal working config) and `configs/examples/` directory with example configs for each source type documented in DESIGN.md.

**Exit criteria:** `configs/default.yml` parses without error. Each example config in `configs/examples/` parses without error (tested via config loader unit tests using golden files).

---

## Milestone 2: Mapping and Sequencer Core

_Produces: A sequencer that maps source values through curves and scales, and outputs MIDI and OSC on an internal clock._

Ref: [DESIGN.md — Sequencer Engine](DESIGN.md#sequencer-engine)

### 2.1 Mapping curves
`[ ]` Implement pure mapping functions in `internal/mapper/curves.go`: linear, logarithmic, exponential, S-curve (sigmoid), step/quantized. Each takes a 0.0–1.0 input and returns 0.0–1.0 output.

**Exit criteria:** Table-driven tests for each curve: 0.0→0.0, 1.0→1.0, midpoint behavior, monotonicity (output[i] <= output[i+1] for ascending input). Fuzz target `FuzzMapper` asserts no NaN/Inf and output stays in [0.0, 1.0].

### 2.2 Core mapper — range scaling and curve application
`[ ]` Implement `Mapper` in `internal/mapper/mapper.go`: configurable source range, destination range, curve selection. Maps source range → 0–1 → curve → destination range.

**Exit criteria:** Tests cover: identity mapping (same ranges, linear curve), range expansion (0–1 → 0–127), range inversion (1–0 input), each curve type applied correctly.

### 2.3 Mapper filters — dead-band and slew limiting
`[ ]` Add dead-band filter (ignore changes smaller than threshold) and slew limiter (cap rate of change per step) to `Mapper`.

**Exit criteria:** Dead-band test: change of 0.001 with threshold 0.01 produces no output change. Slew test: jump from 0.0 to 1.0 with slew=0.1 takes 10 steps to reach target.

### 2.4 Input conditioning filters
`[ ]` Implement moving-average and low-pass filters in `internal/input/filter.go`. Configurable via `FilterType` and `FilterParam` from `SourceConfig`.

**Exit criteria:** Moving-average test: window=4, inputs [1,2,3,4] → output 2.5. Low-pass test: step input converges to target within expected time constant. Both filters handle startup (fewer samples than window) gracefully.

### 2.5 Scale library
`[ ]` Define built-in scales in `internal/mapper/scales.go`: chromatic, major, natural minor, pentatonic major/minor, whole tone, blues, dorian, phrygian, lydian, mixolydian, locrian, harmonic minor, melodic minor. Support custom scales as `[]int` (semitone offsets from root).

**Exit criteria:** Each built-in scale has a test asserting correct interval pattern (e.g. major = [0,2,4,5,7,9,11]). Custom scale test with arbitrary intervals.

### 2.6 Scale quantizer
`[ ]` Implement pitch quantizer in `internal/mapper/quantizer.go`. Input: float64 pitch (MIDI note number). Output: nearest in-scale pitch. Configurable root note, octave range, quantize strength (0–100%).

**Exit criteria:** Table-driven tests for: chromatic (passthrough), C major (61.0/C#→60.0/C or 62.0/D), pentatonic, custom scale. Strength 0% = passthrough, 100% = hard snap, 50% = halfway. Fuzz target `FuzzQuantizer` asserts output is always a valid in-scale note.

### 2.7 USB MIDI output
`[ ]` Implement `MIDIOutput` in `internal/output/midi.go`. Support NoteOn, NoteOff, CC, PitchBend, Clock, Start, Stop. Use `gomidi/midi` or `portmidi` binding.

**Exit criteria:** Unit tests verify correct MIDI byte encoding: NoteOn ch0 note 60 vel 100 → `[0x90, 0x3C, 0x64]`, CC ch0 cc1 val 64 → `[0xB0, 0x01, 0x40]`, PitchBend center → `[0xE0, 0x00, 0x40]`. Mock MIDI port records all sent bytes.

### 2.8 Internal clock
`[ ]` Implement clock in `internal/engine/clock.go`. Configurable BPM (20–300), PPQ resolution (96 default), clock division/multiplication, start/stop/reset.

**Exit criteria:** Tests verify: tick callback fires at expected rate (120 BPM / 96 PPQ = tick every ~5.2ms, tolerance ±1%), division by 4 produces quarter-note steps, start/stop/reset state transitions are correct, stop halts callbacks.

### 2.9 Single-track sequencer — live mode
`[ ]` Implement `Track` and `Engine` in `internal/engine/`. Live mode: each clock step reads source → applies filter → mapper → quantizer → sends to output.

**Exit criteria:** Test with mock source (returns known sequence), mock output: assert output receives correctly mapped and quantized values at each step in order.

### 2.10 Record and playback mode
`[ ]` Add record and playback modes to `Track`. Record: capture mapped values into ring buffer. Playback: loop buffer contents as repeating sequence. Support forward, reverse, ping-pong direction.

**Exit criteria:** Test: record 8 steps of known values, switch to playback, assert output matches recorded sequence for 2+ complete loops. Test reverse outputs values in reversed order. Test ping-pong outputs forward-then-backward.

### 2.11 Multi-track sequencer
`[ ]` Support 4+ independent tracks in `Engine`, each with own source, mapper, quantizer, output, and sequence length.

**Exit criteria:** Test: 4 tracks with different mock sources and outputs, different sequence lengths. Assert each track outputs independently. Polymetric test: track A = 3 steps, track B = 4 steps — after 12 steps, both have cycled an integer number of times.

### 2.12 Advanced normalization modes
`[ ]` Add auto-range (learn min/max from observed values), logarithmic, delta (rate of change), and z-score normalization modes to `internal/input/normalize.go`.

**Exit criteria:** Auto-range test: first N values expand range, subsequent values normalize within learned range. Log test: input spanning 3 orders of magnitude maps evenly. Delta test: monotonic counter input produces rate. Z-score test: values near mean → ~0.5, outliers → near 0.0 or 1.0.

---

## Milestone 3: Data Source Adapters

_Produces: Full suite of remote data source adapters, each with unit and integration tests._

Ref: [DESIGN.md — Data Sources](DESIGN.md#data-sources), [SPECS.md — Data Source Specifications](SPECS.md#data-source-specifications)

### 3.1 Integration test infrastructure
`[ ]` Add `.github/docker-compose.integration.yml` per SPECS.md (Postgres, Redis, Prometheus, Mosquitto). Add `make integration` target. Wire integration job into CI pipeline.

**Exit criteria:** `docker compose -f .github/docker-compose.integration.yml up -d --wait` starts all services and health checks pass. `make integration` runs `go test -tags=integration ./...` and exits 0 (no integration tests exist yet, so it's a no-op pass).

### 3.2 REST/HTTP adapter
`[ ]` Implement in `internal/input/datasource/rest.go`. GET or POST on poll interval, JSONPath value extraction, normalization. Uses shared backoff for reconnection.

**Exit criteria:** Unit tests with `httptest.Server`. Tests for: GET with JSONPath extraction (nested object, array index), POST with body, poll interval respected, HTTP 4xx/5xx → hold last value + backoff, timeout → hold last value.

### 3.3 WebSocket adapter
`[ ]` Implement in `internal/input/datasource/websocket.go`. Persistent connection, JSONPath value extraction per message, reconnection with shared backoff utility.

**Exit criteria:** Unit tests with `httptest` WebSocket server. Tests for: connect + receive message + extract value, server closes connection → reconnect with backoff, Subscribe callback fires on each new value.

### 3.4 PostgreSQL adapter
`[ ]` Implement in `internal/input/datasource/postgres.go` using `pgx`. SQL query on poll interval, extract first-row scalar value, normalize.

**Exit criteria:** Unit tests with interface-based mock. Integration test (`//go:build integration`) against real Postgres from Docker Compose: create table, insert rows, query, verify extracted + normalized value.

### 3.5 Redis adapter
`[ ]` Implement in `internal/input/datasource/redis.go` using `go-redis`. Two modes: poll (GET/LLEN/etc.) and push (SUBSCRIBE). Reconnection via shared backoff.

**Exit criteria:** Unit tests with mock. Integration test against real Redis: SET key + poll GET verifies value, LPUSH + LLEN verifies count, PUBLISH + SUBSCRIBE verifies push delivery.

### 3.6 MQTT adapter
`[ ]` Implement in `internal/input/datasource/mqtt.go` using `paho.mqtt.golang`. Subscribe to topic, JSONPath value extraction, QoS 0 and 1.

**Exit criteria:** Unit tests with mock. Integration test against Mosquitto: publish JSON to topic, verify adapter receives and extracts correct value via JSONPath.

### 3.7 OTEL receiver — transport and metrics
`[ ]` Implement OTLP/gRPC receiver in `internal/input/datasource/otel.go`. Accept metrics on port 4317. Extract gauge values → normalized float. Filter by service name and metric name.

**Exit criteria:** Unit test: construct OTLP ExportMetricsServiceRequest protobuf with a gauge data point, push to receiver, verify extracted and normalized value.

### 3.8 OTEL receiver — traces
`[ ]` Add trace handling to OTEL receiver. Span start → gate on, span end → gate off. Span duration → CV value. Filter by service name and span name.

**Exit criteria:** Unit test: push span with known start/end times, verify gate on at start, gate off at end, duration maps to expected CV value.

### 3.9 OTEL receiver — logs
`[ ]` Add log handling to OTEL receiver. Log event → trigger pulse. Severity level → CV value. Filter by severity and body regex.

**Exit criteria:** Unit test: push log record with severity ERROR, verify trigger fires. Push log with severity INFO when filter is `>= WARN`, verify trigger does not fire.

### 3.10 Staleness detection
`[ ]` Implement per-source staleness tracking. Mark source as `Stale` when no new value arrives within 3× poll interval (configurable). Configurable behavior: hold last value, decay to zero, or fire alert gate.

**Exit criteria:** Test: source stops updating, after 3× interval source state becomes `Stale`. Test hold: output unchanged. Test decay: output ramps toward 0.0. Test alert gate: gate output fires.

---

## Milestone 4: Advanced Sequencer Features

_Produces: Generative and expressive sequencing — Euclidean rhythms, probability, parameter locks._

Ref: [DESIGN.md — Rhythm and Timing](DESIGN.md#rhythm-and-timing)

### 4.1 Euclidean rhythm generator
`[ ]` Implement Bjorklund's algorithm in `internal/engine/euclidean.go`. Inputs: steps (int), pulses (int). Output: boolean pattern (always starting with a pulse on step 0). Sensor value can drive the pulses parameter.

**Exit criteria:** Table-driven tests for known patterns with step-0-aligned convention: E(3,8), E(5,8), E(5,13), E(1,4), E(0,8)=[00000000], E(8,8)=[11111111]. Test: pattern length always equals steps, pulse count always equals pulses.

### 4.2 Probability per step
`[ ]` Add per-step probability (0–100%) to `Track`. On each step, fire with given probability, else skip (gate off, no note).

**Exit criteria:** Test with deterministic seeded RNG: probability=100% fires every step, probability=0% fires none, probability=50% with known seed produces exact expected sequence.

### 4.3 Ratcheting
`[ ]` Add per-step ratchet count (1×–4×) to `Track`. Subdivides the step into N equal triggers within the step duration.

**Exit criteria:** Test: ratchet=1 fires once per step. Ratchet=4 fires 4 triggers per step. Verify sub-trigger timing is evenly spaced (±1% of expected interval). Total duration equals one step.

### 4.4 Swing/shuffle
`[ ]` Add swing parameter (50–75%) to clock. Even-numbered steps (0-indexed: steps 1, 3, 5...) are delayed by the swing amount.

**Exit criteria:** Swing=50% → zero delay (straight time), tolerance ±0.1ms. Swing=67% → even steps delayed by 1/3 of step duration, ±1%. Swing=75% (max) → even steps delayed by 1/2 of step duration. Odd steps unaffected in all cases.

### 4.5 Parameter locks
`[ ]` Add per-step parameter overrides to `Track`. Any mapping parameter (curve type, destination range, quantize scale) can be overridden on individual steps. Unlocked steps use the track default.

**Exit criteria:** Test: default mapping is linear 0–127. Step 3 has lock: log curve, range 0–64. Verify step 3 output uses lock, steps 0/1/2/4+ use default. Test removing a lock restores default.

### 4.6 Noise/mutation layer
`[ ]` Implement in `internal/input/mutation.go`. Wraps an `InputSource`, applies configurable jitter (random), drift (random walk), or Perlin-style smooth noise around the last-known source value. Auto-activates when underlying source is stale.

**Exit criteria:** Test: source returns 0.5, mutation amplitude=0.1 produces values in [0.4, 0.6] over 100 samples. Test: source goes stale (3× poll interval), mutation auto-activates. Test: amplitude=0 passes source value through unchanged.

---

## Milestone 5: Production Polish

_Produces: A production-ready service with BLE MIDI, clock sync, config reload, acceptance tests, and packaging._

Ref: [DESIGN.md — Output Layer](DESIGN.md#output-layer), [SPECS.md — MIDI Electrical Specifications](SPECS.md#midi-electrical-specifications)

### 5.1 BLE MIDI output
`[ ]` Implement BLE MIDI in `internal/output/ble_midi.go`. Encode MIDI messages per BLE MIDI spec with timestamps. Advertise service UUID `03B80E5A-EDE8-4B33-A751-6CE34EC4C700`.

**Exit criteria:** Unit tests verify BLE MIDI packet encoding matches spec: correct service UUID, characteristic UUID `7772E5DB-3868-4112-A1A9-F2669D106BF3`, timestamp header bytes, MIDI payload. Integration test on Linux with BlueZ (if available, otherwise skip with `t.Skip`).

### 5.2 MIDI clock sync — send
`[ ]` Add MIDI clock output to `Engine`. Send 24 MIDI Clock messages per quarter note, plus Start/Stop/Continue on transport state changes.

**Exit criteria:** Test: at 120 BPM, verify 48 clock messages per second (24 PPQ × 2 beats/sec). Start message sent on play, Stop on stop, Continue on resume.

### 5.3 MIDI clock sync — receive
`[ ]` Add external MIDI clock input to `Engine`. Tempo follower: measure interval between incoming Clock messages, derive BPM, sync internal sequencer clock.

**Exit criteria:** Test: feed 24 clock messages at 120 BPM intervals, verify derived BPM = 120 ±1. Test tempo change from 120→140 BPM is tracked within 1 beat.

### 5.4 Graceful shutdown
`[ ]` Comprehensive resource cleanup on SIGINT/SIGTERM: close all data source connections, send MIDI all-notes-off, flush output buffers, release MIDI/BLE/OSC ports, exit 0.

**Exit criteria:** Test: start engine with mock sources and outputs, send SIGTERM, verify every source's `Close()` called, every output's `Close()` called, no goroutine leaks (via `goleak`), exit code 0.

### 5.5 Config reload
`[ ]` Handle SIGHUP: re-read YAML config, diff against running config, add/remove/reconfigure sources and outputs without stopping the clock or interrupting unaffected tracks.

**Exit criteria:** Test: start with config A (1 Prometheus source), send SIGHUP with config B (add a second REST source). Verify: second source starts producing values, first source has no output gap > 1 poll interval during reload.

### 5.6 Preset save/load
`[ ]` Save current engine state to YAML file. State includes: track count, per-track source binding, mapper config (curve, source/dest ranges), quantizer config (scale, root, strength), recorded sequence data, clock BPM and division. Load preset restores all state.

**Exit criteria:** Test: configure engine with 2 tracks (specific curves, scales, 8-step recorded sequences, BPM=140), save preset, reset engine to defaults, load preset, verify all parameters match saved values.

### 5.7 Status API
`[ ]` HTTP server with `/api/status` endpoint returning JSON: per-source connection state, current value, staleness; per-track mode, current step, sequence length; clock BPM and running state.

**Exit criteria:** Test with `httptest`: start engine with mock sources, GET `/api/status`, assert JSON contains source states and track info with correct values.

### 5.8 Embedded web frontend
`[ ]` Static HTML/JS UI served via Go `embed` at `/`. Renders status data from `/api/status`. Shows source connection indicators, live value bars, track state.

**Exit criteria:** Test: GET `/` returns HTTP 200 with `Content-Type: text/html` and body contains expected `<script>` and `<div>` elements.

### 5.9 Acceptance test suite
`[ ]` Implement end-to-end acceptance tests per SPECS.md in `test/acceptance/`: `TestPrometheusToMIDI`, `TestMultiSourceMultiTrack`, `TestConfigReload`, `TestGracefulShutdown`, `TestClockSync`, `TestRecordPlaybackLoop`. Uses mock output backends (`midi_mock.go`, `osc_mock.go`).

**Exit criteria:** All 6 acceptance tests pass with `go test -tags=acceptance ./test/acceptance/...`. Tests are wired into CI on `main` branch only.

### 5.10 GoReleaser config and Dockerfile
`[ ]` Create `.goreleaser.yml` and `Dockerfile` per SPECS.md. Distroless base image, non-root user, OTEL ports exposed.

**Exit criteria:** `goreleaser check` validates config. `goreleaser build --snapshot --clean` produces binaries for linux/amd64, linux/arm64, linux/arm, darwin/arm64. Docker build succeeds.

---

## Milestone 6: DAW Plugin

_Produces: A CLAP/VST3 plugin that receives OSC from Grafophone and exposes values as DAW-automatable parameters._

Ref: [DESIGN.md — VST/CLAP Plugin](DESIGN.md#vstclap-plugin)

### 6.1 CLAP plugin skeleton
`[ ]` C++ CLAP plugin that loads in a host, exposes 16 automatable float parameters (0.0–1.0), does no audio processing.

**Exit criteria:** `clap-validator` reports zero errors for plugin descriptor. Plugin loads in Reaper or similar CLAP host. 16 parameters visible in automation lanes.

### 6.2 OSC receiver in plugin
`[ ]` Plugin listens on configurable UDP port for OSC messages. Maps incoming `/grafophone/param/N` messages to plugin parameter N.

**Exit criteria:** Unit test (C++): send OSC message to plugin's UDP port, verify parameter value updates. Verify thread-safe parameter update from network thread to audio thread.

### 6.3 Plugin UI — state model and rendering
`[ ]` Minimal UI: connection status indicator, per-parameter current value display, OSC port configuration. Testable state model separate from rendering.

**Exit criteria:** Unit test: state model correctly reflects connected/disconnected/timeout states. State model updates parameter display values on incoming OSC. Manual verification: UI renders in DAW.

### 6.4 VST3 build
`[ ]` Build the same plugin as VST3 (via dual-target CMake or JUCE wrapper) for broader DAW compatibility.

**Exit criteria:** Plugin loads in a VST3 host. `vstvalidator` reports no errors. Parameters and OSC receiver behave identically to CLAP version.

---

## Milestone 7: Hardware I/O

_Produces: CV/Gate output on Raspberry Pi, hardware sensor input, DAC calibration, DIN-5 MIDI._

Ref: [DESIGN.md — Hardware Platform](DESIGN.md#hardware-platform), [SPECS.md — CV/Gate Output](SPECS.md#cvgate-output-specifications), [SPECS.md — DAC Selection](SPECS.md#dac-selection), [SPECS.md — Sensor Input](SPECS.md#sensor-input-specifications)

### 7.1 periph.io SPI/I2C initialization
`[ ]` Set up `periph.io` host drivers in `internal/output/hal.go`. Open SPI and I2C buses. Detect connected devices.

**Exit criteria:** Unit tests with mock SPI/I2C buses. Integration test on Pi (`//go:build integration`): detect bus, list connected device addresses.

### 7.2 DAC8568 pitch CV driver
`[ ]` Implement 16-bit SPI DAC driver in `internal/output/dac8568.go`. Write voltage to any of 8 channels. Enable internal 2.5V reference. Gain stage produces 0–10V.

**Exit criteria:** Unit tests: verify SPI command bytes for 0V → code 0x0000, 5V → code 0x7FFF, 10V → code 0xFFFF. Integration test on Pi: measure output with multimeter (manual).

### 7.3 MCP4728 mod CV driver
`[ ]` Implement 12-bit I2C DAC driver in `internal/output/mcp4728.go`. Write voltage to any of 4 channels.

**Exit criteria:** Unit tests: verify I2C command bytes for known voltage targets. Integration test on Pi: measure output voltage (manual).

### 7.4 Gate/trigger output
`[ ]` Implement gate output via GPIO in `internal/output/gate.go`. On/off state for gates, configurable pulse width (5–15ms) for triggers.

**Exit criteria:** Unit tests: verify GPIO set/clear calls and timing. Trigger test: pulse width within ±1ms of configured duration.

### 7.5 CV output composite interface
`[ ]` Wire DAC8568 + MCP4728 + gate GPIO into a `CVOutput` implementation. Pitch channels → DAC8568, mod channels → MCP4728, gate channels → GPIO.

**Exit criteria:** Unit tests with mock DACs: `WritePitch(0, 3.0)` sends correct code to DAC8568 channel 0. `WriteMod(0, 0.5)` sends correct code to MCP4728 channel 0. `WriteGate(0, true)` sets GPIO high.

### 7.6 Two-point DAC calibration
`[ ]` Implement in `internal/calibration/twopoint.go`. Given two measured voltages (V_low, V_high) at DAC codes 0x0000 and 0xFFFF, compute gain + offset correction applied at runtime.

**Exit criteria:** Test: V_low=0.01V, V_high=9.98V (targets 0V and 10V) → corrected DAC code for 5.0V target is within 1 LSB of expected.

### 7.7 Multi-point lookup table calibration
`[ ]` Implement in `internal/calibration/lookup.go`. Build correction table from measured voltages at semitone intervals. Interpolate between table entries at runtime. Store/load table from file.

**Exit criteria:** Test: given correction table with known errors at 12 points, interpolated correction for intermediate voltages is within 0.5 LSB. Test: save table to file, load from file, results identical.

### 7.8 Calibration tool CLI
`[ ]` Implement `cmd/grafocal/main.go`. Step through DAC codes, prompt user for measured voltage at each step, build and save calibration lookup table.

**Exit criteria:** Test with simulated stdin input: user enters 12 voltage readings, calibration table is written to expected file path in correct format. Table loads correctly via 7.7.

### 7.9 IMU sensor driver (MPU-6050)
`[ ]` Implement in `internal/input/sensor/imu.go`. Read 3-axis accelerometer + 3-axis gyroscope via I2C. Normalize each axis to -1.0–1.0. Apply complementary filter for stable orientation.

**Exit criteria:** Unit tests with mock I2C: verify correct register read sequence, verify raw-to-normalized conversion. Test complementary filter: noisy input converges to stable orientation angle.

### 7.10 Environmental sensor driver (BME280)
`[ ]` Implement in `internal/input/sensor/env.go`. Read temperature, humidity, pressure via I2C. Apply BME280 compensation formulas. Normalize each to 0.0–1.0 based on configurable range.

**Exit criteria:** Unit tests with mock I2C: verify compensation formula matches BME280 datasheet examples, verify normalization.

### 7.11 Distance sensor driver (VL53L0X)
`[ ]` Implement in `internal/input/sensor/distance.go`. Read time-of-flight distance via I2C. Normalize to 0.0–1.0 based on configured min/max range.

**Exit criteria:** Unit tests with mock I2C: verify register sequence, verify normalization. Out-of-range readings (closer than min, farther than max) clamp to 0.0 or 1.0 respectively.

### 7.12 DIN-5 MIDI output
`[ ]` Implement traditional MIDI over UART in `internal/output/din_midi.go`. 31250 baud serial output via Pi GPIO UART. Same `MIDIOutput` interface as USB MIDI.

**Exit criteria:** Unit tests: verify byte encoding matches USB MIDI tests (same interface, same bytes). Integration test on Pi: loopback UART to MIDI input, verify received messages match sent messages.

### 7.13 systemd unit and postinstall script
`[ ]` Create `scripts/grafophone.service` and `scripts/postinstall.sh` per SPECS.md. Service user creation, config file permissions, systemd enable.

**Exit criteria:** `systemd-analyze verify scripts/grafophone.service` passes. `shellcheck scripts/postinstall.sh` passes.

### 7.14 .deb packaging via GoReleaser nfpm
`[ ]` Add nfpm section to `.goreleaser.yml` per SPECS.md. Package includes binary, default config, systemd unit, postinstall script.

**Exit criteria:** `goreleaser build --snapshot --clean` produces `.deb`. Package installs on Debian/Ubuntu test container: creates `grafophone` user, installs service unit, places config at `/etc/grafophone/config.yml`.
