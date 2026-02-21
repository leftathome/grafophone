# Grafophone: Sensor-to-Synthesizer Sequencer

## Vision

Grafophone transforms real-world data streams into musical sequences for analog and digital synthesizers. It ingests time-series data from two classes of sources:

1. **Hardware sensors** — IMUs, environmental sensors, touch surfaces, distance sensors, microphones (connected via I2C/SPI/ADC/BLE)
2. **Software data sources** — databases (Postgres, Redis), observability platforms (Grafana, Prometheus), OpenTelemetry endpoints, REST/GraphQL APIs, and arbitrary time-series feeds

Data from either source class is mapped to musical parameters and output as CV/Gate signals, MIDI, OSC, and/or VST plugin automation.

The system bridges the physical and digital worlds and the modular synthesizer — turning server metrics into melodies, gesture into rhythm, infrastructure alerts into triggers, and environmental flux into evolving sonic textures.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Sensor Input Layer](#sensor-input-layer)
3. [Sequencer Engine](#sequencer-engine)
4. [Output Layer](#output-layer)
5. [Hardware Platform](#hardware-platform)
6. [Software Architecture](#software-architecture)
7. [Open Source References](#open-source-references)
8. [CI/CD Pipeline](#cicd-pipeline)
9. [Build Artifacts & Release Process](#build-artifacts--release-process)
10. [Development Phases](#development-phases)

For detailed technical specifications, see [SPECS.md](SPECS.md).

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   HARDWARE SENSORS                       │
│                                                         │
│  IMU ─┐  Env Sensors ─┐  Touch ─┐  Analog ─┐  BLE ─┐  │
│       │               │         │           │       │  │
│       ▼               ▼         ▼           ▼       ▼  │
│  ┌──────────────────────────────────────────────────┐  │
│  │           I2C / SPI / ADC / Wireless              │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────────┐
│                   DATA SOURCES                    │      │
│                                                   ▼      │
│  Postgres ─┐  Prometheus ─┐  OTEL ─┐  REST ─┐  Redis ─┐│
│            │              │        │        │          ││
│            ▼              ▼        ▼        ▼          ▼│
│  ┌──────────────────────────────────────────────────┐   │
│  │      Polled / Subscribed / Streamed Queries       │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                   SEQUENCER ENGINE                       │
│                                                          │
│  Time Series Buffer ──▶ Mapping/Quantizer ──▶ Sequencer │
│                                                          │
│  Features:                                               │
│  - Record / playback / loop sensor streams               │
│  - Configurable mapping curves (linear, log, S-curve)    │
│  - Scale quantization (chromatic, major, pentatonic...)  │
│  - Clock sync (internal, external, MIDI clock)           │
│  - Euclidean rhythm generation from sensor density       │
│  - Parameter locks per step                              │
│  - Probability / humanization per step                   │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                    OUTPUT LAYER                           │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │  CV/Gate  │  │   MIDI   │  │   OSC    │  │  VST    │ │
│  │ (analog)  │  │ (USB/DIN)│  │ (network)│  │ (plugin)│ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
│                                                          │
│  CV: 16-bit DAC, 1V/Oct, +/-5V or 0-10V                │
│  Gate: 0V / +5V (or +10V), <1ms rise time              │
│  MIDI: USB gadget mode + DIN-5                          │
│  OSC: WiFi/Ethernet for software synth control          │
│  VST: CLAP/VST3 plugin for DAW integration              │
└──────────────────────────────────────────────────────────┘
```

---

## Sensor Input Layer

The input layer has two sub-systems: **hardware sensors** (physical devices) and **data sources** (software/network queries). Both produce normalized time-series streams that feed identically into the sequencer engine.

### Hardware Sensors

| Category | Sensors | Data Characteristics | Musical Application |
|---|---|---|---|
| **Motion/Gesture** | MPU-6050, BNO055, LSM6DSO | 6/9-DOF, 100-1000 Hz | Pitch bend, filter sweeps, tilt-to-note |
| **Environment** | BME280, BME680, SHT40 | Temp/humidity/pressure, 1-10 Hz | Slow evolving drones, generative seeds |
| **Light** | TSL2591, VEML7700, phototransistors | Lux/spectral, 1-100 Hz | Brightness-to-velocity, light gates |
| **Distance** | VL53L0X, HC-SR04, Sharp IR | mm-to-m range, 10-50 Hz | Theremin-like pitch, proximity triggers |
| **Touch** | MPR121, capacitive pads, FSR | 12-channel touch, pressure | Note triggers, pressure-to-modulation |
| **Flex/Strain** | Flex sensors, load cells + HX711 | Resistance/weight change | Bow pressure, physical modeling input |
| **Sound** | INMP441 (I2S), MAX4466 (analog) | Audio envelope, FFT peaks | Audio-reactive sequencing, vocoder CV |
| **Magnetic** | Hall effect (SS49E), magnetometer | Field strength/direction | Proximity detection, compass mapping |

### Data Sources

Software-defined inputs that are periodically queried, subscribed to, or streamed. Each data source is configured with a query/endpoint, a poll interval (or push/subscription mode), and a value extractor that produces a scalar or vector time series.

| Category | Sources | Query Method | Musical Application |
|---|---|---|---|
| **Relational DB** | PostgreSQL, MySQL, SQLite | SQL query on poll interval | Map row counts, aggregates, column values to CV. Database activity → rhythmic density. |
| **Key-Value / Cache** | Redis, Memcached | Key subscribe (SUBSCRIBE/keyspace notifications) or poll | Real-time counters → velocity. Cache hit rates → filter cutoff. |
| **Time-Series DB** | Prometheus, InfluxDB, TimescaleDB | PromQL / Flux query on interval | CPU load → pitch. Request rate → tempo. Error rate → chaos parameter. |
| **Observability** | Grafana (API), Datadog, New Relic | Dashboard/panel JSON API poll | Any Grafana panel data source as a musical input. Alert state → gate trigger. |
| **OpenTelemetry** | OTLP gRPC/HTTP receiver | Push (OTLP receiver endpoint) | Trace spans → note events. Metric streams → CV. Log severity → velocity. |
| **REST / GraphQL** | Any HTTP API | GET/POST on poll interval | Weather APIs → environmental parameters. Stock prices → pitch sequences. |
| **Message Queue** | MQTT, RabbitMQ, Kafka | Subscribe/consume | IoT device telemetry → sensor streams. Event-driven triggers. |
| **WebSocket** | Any WS endpoint | Persistent connection | Real-time feeds (crypto prices, chat activity, game state). |

#### Data Source Configuration

Each data source is defined by:

```yaml
sources:
  - name: "cpu_load"
    type: prometheus
    endpoint: "${PROMETHEUS_URL}"          # e.g. https://prometheus:9090
    query: "avg(rate(node_cpu_seconds_total{mode!='idle'}[1m]))"
    poll_interval: 1s
    value_path: "result[0].value[1]"      # JSONPath to extract scalar
    range: [0.0, 1.0]                      # expected value range for normalization

  - name: "active_users"
    type: postgres
    connection: "${GRAFOPHONE_POSTGRES_URL}" # e.g. postgresql://user:pass@host/db
    query: "SELECT count(*) FROM sessions WHERE active = true"
    poll_interval: 5s
    range: [0, 10000]

  - name: "error_events"
    type: otel_receiver
    protocol: grpc
    port: 4317
    filter: "severity >= ERROR"
    mode: push                              # event-driven, not polled
    # Each matching event produces a trigger/gate

  - name: "redis_queue_depth"
    type: redis
    connection: "${GRAFOPHONE_REDIS_URL}"   # e.g. redis://:password@host:6379
    command: "LLEN work_queue"
    poll_interval: 500ms
    range: [0, 1000]

  - name: "btc_price"
    type: websocket
    url: "wss://stream.binance.com/ws/btcusdt@trade"
    value_path: "p"                         # price field from JSON message
    range: [20000, 100000]
```

**Credential management:** Connection strings and API keys must never be hardcoded in config files. Use environment variable substitution (`${VAR_NAME}`) in YAML configs. The config loader expands environment variables at startup. For production deployments, inject secrets via a secrets manager (e.g., HashiCorp Vault, systemd `EnvironmentFile`, Kubernetes secrets).

#### Data Source Adapters

Each data source type has an adapter that handles:
- **Connection management**: Connect, reconnect on failure, connection pooling
- **Authentication**: Credentials, API keys, mTLS, OAuth tokens
- **Query execution**: Timed polling or subscription/push handling
- **Value extraction**: JSONPath, XPath, or regex to pull scalar/vector values from responses
- **Normalization**: Map raw values to 0.0-1.0 float based on configured range (with auto-range learning option)
- **Error handling**: Timeout, connection loss → hold last value, output zero, or trigger an alert gate

#### Event-Driven vs Polled Sources

- **Polled sources** (Prometheus, Postgres, REST): Query on a configurable interval. Produces a steady stream of values like a slow-updating sensor.
- **Push/subscribed sources** (OTEL receiver, Redis SUBSCRIBE, WebSocket, MQTT): Events arrive asynchronously. Each event can produce a gate trigger and/or update a CV value.
- **Hybrid**: Some sources support both. E.g., Prometheus can be polled, but Alertmanager can push webhook alerts.

See [SPECS.md — Data Source Specifications](SPECS.md#data-source-specifications) for protocol details, authentication methods, poll interval ranges, and resource limits.

### Sensor Bus Architecture (Hardware)

- **I2C bus** (400 kHz Fast Mode): IMU, environmental, light, ToF sensors. Use TCA9548A 8-channel I2C multiplexer for address conflicts (up to 8 multiplexers for 64 downstream channels).
- **SPI bus** (10+ MHz): High-speed ADCs (MCP3208), DACs, displays. One CS line per device.
- **Analog inputs**: Via external ADC (MCP3208 for 8-channel 12-bit, ADS1115 for 16-bit). Flex sensors, FSRs, phototransistors, analog microphones.
- **BLE/WiFi**: ESP32 or Pi for receiving wireless sensor data. ESP-NOW for low-latency connectionless streaming.

### Input Conditioning

Applies to both hardware sensors and data sources:

- All inputs should be filtered (moving average or low-pass) to remove noise/jitter before mapping.
- Configurable sample/poll rates per input (1 Hz for environment/slow DB queries, 100+ Hz for gesture sensors).
- Sensor fusion (complementary or Madgwick filter) for IMU data to get stable orientation.
- Calibration routines: auto-range detection, min/max learning, zero-offset correction.
- **Dead-band**: Ignore changes smaller than a configurable threshold (prevents jitter on noisy sources).
- **Slew limiting**: Smooth sudden jumps in data source values to produce musically useful glides rather than harsh steps.

---

## Sequencer Engine

### Core Concepts

The sequencer engine treats sensor data as **time-series streams** that are recorded, transformed, and played back as musical events.

**Data Flow:**
```
Sensor Stream ──▶ Ring Buffer ──▶ Mapper ──▶ Quantizer ──▶ Step Sequence ──▶ Output
                      │                                          │
                      ▼                                          ▼
                  Record/Loop                              Clock/Transport
```

### Time Series Capture

- **Ring buffer** per sensor channel, configurable depth (e.g., 256-4096 samples).
- **Record modes**: Free-running, clock-synced, trigger-start.
- **Playback modes**: Forward, reverse, ping-pong, random, interpolated.
- **Time stretching**: Decouple capture rate from playback rate. A 10-second gesture can drive a 1-bar or 64-bar sequence.

### Mapping Engine

Sensor values are mapped to musical parameters through configurable **mapping slots**:

```
[Source: IMU_accel_x] ──▶ [Curve: logarithmic] ──▶ [Range: C2-C6] ──▶ [Dest: CV_pitch_1]
[Source: light_lux]   ──▶ [Curve: S-curve]      ──▶ [Range: 0-127]  ──▶ [Dest: MIDI_CC_1]
[Source: distance_mm]  ──▶ [Curve: linear]       ──▶ [Range: 0-10V]  ──▶ [Dest: CV_mod_1]
```

**Mapping curves**: Linear, logarithmic, exponential, S-curve (sigmoid), step/quantized, custom breakpoint tables.

**Scaling modes**:
- Absolute: sensor range maps directly to output range
- Relative/delta: sensor changes add/subtract from current value
- Threshold/gate: sensor crossing a threshold triggers a gate
- Window: only a sub-range of sensor input produces output

### Quantization

- **Scale-aware quantization**: Constrain pitch CV/MIDI notes to a musical scale.
- Built-in scales: chromatic, major, minor, pentatonic, whole tone, blues, dorian, phrygian, lydian, mixolydian, locrian, harmonic minor, melodic minor, hungarian minor, japanese (in-sen, hirajoshi), arabic, custom user scales.
- **Root note** and **octave range** configurable per output.
- **Quantize strength**: 0% = free pitch, 100% = hard quantize, in-between = attracted toward scale tones.

### Rhythm and Timing

- **Clock sources**: Internal BPM (20-300), external clock input (rising edge), MIDI clock, tap tempo.
- **Clock division/multiplication**: 1/1 to 1/64, triplets, dotted.
- **Euclidean rhythm generator**: Sensor density value drives the fill parameter of a Euclidean pattern. Maps naturally from e.g. light level → rhythmic density.
- **Swing/shuffle**: Adjustable per track (50-75%).
- **Probability per step**: 0-100% chance of firing. Sensor data can modulate probability in real time.
- **Ratcheting**: Repeat a step N times within its time slot (1x, 2x, 3x, 4x).

### Sequencer Modes

1. **Live**: Sensor data passes through mapping/quantization in real time to outputs. No recording.
2. **Record**: Capture sensor stream to ring buffer. Continues to output in real time.
3. **Playback**: Loop the recorded buffer as a repeating sequence. Optionally quantize to step grid.
4. **Overdub**: Layer new sensor data on top of existing recorded sequence.
5. **Generative**: Use sensor data as seeds for algorithmic patterns (Euclidean rhythms, Markov chains, cellular automata, Turing machine-style shift registers).

### Multi-Track

- Minimum 4 independent tracks, each with its own sensor source, mapping, quantization, and output destination.
- Tracks can share a clock or run at independent divisions.
- Per-track sequence length (polymetric sequencing).
- Parameter locks: override any parameter on a per-step basis (inspired by Elektron sequencers).

---

## Output Layer

### CV/Gate (Analog)

**Voltage Standards:**
- **1V/Oct pitch CV**: 1 volt per octave, the dominant standard. 0V = configurable reference note (commonly C1 or C2, varies by manufacturer). 5V range = 5 octaves.
- **Gate**: 0V (off) / +5V or +10V (on). Active duration configurable. Rise time < 1 ms.
- **Trigger**: 5-10 ms pulse at gate voltage. For clocks, resets, drum triggers.
- **Modulation CV**: 0-5V, 0-10V, or +/-5V depending on destination. For filter cutoff, VCA level, waveshaping.

**Target channel count:**
- 4x pitch CV (16-bit DAC, calibrated to 1V/Oct)
- 4x gate/trigger (level-shifted to 5V or 10V)
- 4x modulation CV (12-bit or 16-bit DAC)
- 1x clock output
- 1x reset output

See [SPECS.md](SPECS.md) for DAC selection, output stage circuits, and calibration details.

### MIDI

- **USB MIDI**: Device mode (appears as MIDI controller to host computer or hardware). Pi 4 / Pi Zero gadget mode or Teensy native USB.
- **DIN-5 MIDI**: Traditional 5-pin MIDI out via UART + transistor/buffer driver circuit.
- **MIDI messages**: Note On/Off, CC (continuous controllers), pitch bend, clock, start/stop, program change.
- **MPE (MIDI Polyphonic Expression)**: Per-note pitch bend, pressure, and slide for expressive control of MPE-capable synths.
- **BLE MIDI**: Wireless MIDI over Bluetooth Low Energy (ESP32 native support).

### OSC (Open Sound Control)

- UDP-based protocol over WiFi/Ethernet.
- Configurable OSC address patterns (e.g., `/grafophone/track1/pitch`).
- Bundles for sample-accurate timing of multiple parameter changes.
- Use cases: SuperCollider, Max/MSP, Pure Data, TouchOSC, VCV Rack.

### VST/CLAP Plugin

A companion DAW plugin that receives sensor data from the hardware and exposes it as automatable parameters:

**Plugin Frameworks (ranked):**
1. **CLAP** (CLever Audio Plug-in API): Modern, open-source, extensible. Best for new development. Supports per-note modulation natively.
2. **VST3** (Steinberg): Widest DAW compatibility. Use JUCE or iPlug2 framework.
3. **JUCE** (framework): Cross-platform, builds VST3/AU/CLAP/AAX from one codebase. Industry standard. GPLv3 or commercial license.
4. **iPlug2** (framework): Lightweight alternative to JUCE, builds VST2/VST3/AU/AAX/CLAP. Liberal license (WDL).

**Plugin Architecture:**
```
Hardware (USB/BLE/WiFi) ──▶ Plugin receives sensor stream
                                    │
                            ┌───────┴───────┐
                            │  Mapping UI   │
                            │  (in-plugin)  │
                            └───────┬───────┘
                                    │
                            DAW automation lanes
                            MIDI CC output
                            Per-note expression
```

**Communication Protocol (Hardware ↔ Plugin):**
- USB serial or USB MIDI SysEx for wired connection
- OSC over network for wireless
- BLE MIDI for wireless without WiFi infrastructure
- Custom binary protocol for highest throughput (sensor data at 100+ Hz)

### MIDI-to-CV Bridge (Software)

For users without dedicated hardware, the sequencer engine can run as software that outputs MIDI, paired with a MIDI-to-CV converter:
- Commercial: Expert Sleepers FH-2, Endorphin.es Shuttle Control, Mutant Brain
- DIY: Arduino/Teensy MIDI-to-CV (open source projects available)
- Software: VCV Rack as a bridge (MIDI input → virtual CV)

---

## Hardware Platform

### Recommended Configurations

#### Configuration A: Raspberry Pi (Full-Featured)

**Best for**: Complex mapping, multiple sensor types, display UI, WiFi/BLE, VST plugin bridge.

| Component | Choice | Rationale |
|---|---|---|
| **Processor** | Raspberry Pi 4B (4GB) | Best real-time Linux performance, USB gadget mode, proven in Eurorack |
| **CV DAC (pitch)** | DAC8568 (16-bit, 8-ch, SPI) | Proven in Mutable Instruments Yarns, sub-cent pitch accuracy |
| **CV DAC (mod)** | MCP4728 (12-bit, 4-ch, I2C) | Adequate for modulation, simple wiring |
| **ADC (sensors)** | MCP3208 (12-bit, 8-ch, SPI) | Fast enough for gesture sensors, proven in Terminal Tedium |
| **Gate output** | N-ch MOSFET (BS170) + 10k pull-up to +12V | Clean 10V gates, software-inverted |
| **Op-amp** | OPA4171 (quad, precision) | Low offset, rail-to-rail, powers from +/-12V |
| **Voltage ref** | REF5025 (2.5V, 3 ppm/C) | Stable pitch CV reference |
| **OS** | Patchbox OS (PREEMPT_RT) | Pre-configured for audio, low latency |
| **Language** | Go (single static binary, `periph.io` for SPI/I2C) | Goroutines for concurrent data sources, zero deployment deps |

#### Configuration B: ESP32 (Compact/Wireless)

**Best for**: Portable, battery-powered, wireless sensor hub, BLE MIDI.

| Component | Choice | Rationale |
|---|---|---|
| **Processor** | ESP32-S3 | Dual core, BLE 5.0, native USB, no legacy BT power draw |
| **CV DAC (pitch)** | MCP4728 (12-bit, 4-ch, I2C) or DAC8568 (16-bit, SPI) | 12-bit adequate for 5-octave range with calibration; 16-bit for precision |
| **ADC (sensors)** | Built-in 12-bit + MCP3208 external | Internal ADC for non-critical, external for precision |
| **Gate output** | 74HCT244 (5V) or MOSFET (10V) | Simple level shifting |
| **Framework** | ESP-IDF (production) or Arduino (prototyping) | Full FreeRTOS control for real-time |
| **Wireless** | BLE for sensor nodes, ESP-NOW for low-latency | Dual-mode wireless |

#### Configuration C: Teensy 4.1 (Performance)

**Best for**: Maximum real-time determinism, USB MIDI, Ornament & Crime ecosystem.

| Component | Choice | Rationale |
|---|---|---|
| **Processor** | Teensy 4.1 (600 MHz Cortex-M7) | Fastest MCU option, native USB MIDI, Audio Library |
| **CV DAC** | DAC8568 (16-bit, 8-ch, SPI) | Same as Ornament & Crime |
| **Eurorack I/O** | Teensy Eurorack shield | 14 in / 16 out, purpose-built |
| **Language** | C++ (Arduino framework) | Audio Library integration, deterministic timing |

### Form Factor Options

1. **Eurorack module** (target: 16-20 HP): Powered from Eurorack bus (+/-12V, +5V). Direct CV/Gate jacks. Display + encoder UI.
2. **Desktop box**: USB-powered or wall adapter. MIDI/CV/Gate jacks. Larger display. WiFi/BLE antenna.
3. **Portable/wearable**: ESP32 battery-powered sensor node that transmits wirelessly to a Eurorack receiver module.

---

## Software Architecture

### Language Choice

**Primary language: Go** for the core application (sequencer engine, data source adapters, sensor drivers, CV/MIDI output, CLI/web UI).

| Concern | Why Go |
|---|---|
| **Concurrency** | Goroutines map naturally to 32+ concurrent data source pollers/subscribers, each running independently |
| **Observability ecosystem** | OTEL SDK, Prometheus client, gRPC — all Go-native, first-class support |
| **Database drivers** | `database/sql` + `pgx` (Postgres), `go-redis`, `sarama` (Kafka), `paho` (MQTT) — mature, well-tested |
| **Hardware I/O** | `periph.io` for SPI/I2C/GPIO on Raspberry Pi; `go-midi` for MIDI |
| **Cross-compilation** | `CGO_ENABLED=0 GOOS=linux GOARCH=arm64` produces a static binary for Pi — no runtime dependencies |
| **Deployment** | Single binary. Copy to Pi, run. No virtualenv, no node_modules, no JVM. |
| **Testing** | Built-in `testing` package, race detector (`-race`), benchmarks, fuzzing |
| **Static analysis** | `golangci-lint` aggregates 50+ linters in one tool |

**Exceptions (separate codebases):**
- **ESP32/Teensy firmware**: C++ (ESP-IDF / Arduino framework). Communicates with Go core via USB serial or BLE.
- **VST/CLAP plugin**: C++ (JUCE or iPlug2). Communicates with Go core via OSC or USB MIDI.

### Go Module Structure (Planned)

```
grafophone/
├── cmd/
│   ├── grafophone/          # main CLI binary
│   │   └── main.go
│   └── grafocal/            # standalone calibration tool
│       └── main.go
├── internal/
│   ├── engine/              # sequencer engine (clock, transport, tracks)
│   ├── mapper/              # mapping curves, quantizer, scale library
│   ├── buffer/              # ring buffer for time-series capture
│   ├── input/               # InputSource interface + implementations
│   │   ├── source.go        # interface definition
│   │   ├── sensor/          # hardware sensor drivers (I2C, SPI, ADC)
│   │   │   ├── imu.go
│   │   │   ├── env.go
│   │   │   └── ...
│   │   └── datasource/      # software data source adapters
│   │       ├── prometheus.go
│   │       ├── postgres.go
│   │       ├── redis.go
│   │       ├── otel.go
│   │       ├── websocket.go
│   │       ├── mqtt.go
│   │       ├── rest.go
│   │       └── ...
│   ├── output/              # Output interface + implementations
│   │   ├── cv.go            # CV/Gate via SPI/I2C DAC
│   │   ├── midi.go          # USB MIDI, DIN MIDI, BLE MIDI
│   │   └── osc.go           # OSC over UDP
│   ├── config/              # YAML config parsing, validation
│   ├── calibration/         # DAC calibration routines, lookup tables
│   └── ui/                  # Web UI (embedded), display driver
├── firmware/                # ESP32/Teensy C++ firmware (separate build)
│   ├── esp32/
│   └── teensy/
├── plugin/                  # VST/CLAP C++ plugin (separate build)
├── configs/                 # example YAML configurations
├── scripts/                 # build, release, flash scripts
├── .github/
│   └── workflows/           # CI/CD pipeline definitions
├── .golangci.yml            # linter configuration
├── go.mod
├── go.sum
├── Makefile
├── DESIGN.md
├── SPECS.md
└── CHANGELOG.md
```

### Key Go Dependencies

| Package | Purpose |
|---|---|
| `periph.io/x/conn/v3` | SPI, I2C, GPIO abstraction for Pi hardware |
| `periph.io/x/host/v3` | Host driver initialization |
| `github.com/jackc/pgx/v5` | PostgreSQL driver (fast, full-featured) |
| `github.com/redis/go-redis/v9` | Redis client with Pub/Sub support |
| `github.com/prometheus/client_golang` | Prometheus metrics exposition |
| `github.com/prometheus/common/model` | Prometheus query result types |
| `go.opentelemetry.io/collector` | OTEL collector receiver components |
| `go.opentelemetry.io/otel` | OTEL SDK for internal instrumentation |
| `google.golang.org/grpc` | gRPC for OTEL OTLP receiver |
| `github.com/gorilla/websocket` | WebSocket client |
| `github.com/eclipse/paho.mqtt.golang` | MQTT client |
| `github.com/IBM/sarama` | Kafka consumer |
| `gitlab.com/gomidi/midi/v2` | MIDI message encoding/decoding |
| `gopkg.in/yaml.v3` | YAML config parsing |
| `github.com/hypebeast/go-osc` | OSC message encoding/sending |

### Module Decomposition

```
┌─────────────────────────────────────────────┐
│              cmd/grafophone                  │
│                                              │
│  ┌─────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ Web UI  │ │  Config  │ │  Calibration │ │
│  │ (embed) │ │  (YAML)  │ │   (grafocal)  │ │
│  └────┬────┘ └────┬─────┘ └──────┬───────┘ │
│       │           │              │          │
│  ┌────▼───────────▼──────────────▼────────┐ │
│  │    internal/engine (Sequencer Engine)  │ │
│  │                                        │ │
│  │  ┌──────────┐ ┌────────┐ ┌─────────┐  │ │
│  │  │ buffer/  │ │ mapper/│ │ engine/ │  │ │
│  │  │ Ring Buf │ │ Curves │ │ Clock   │  │ │
│  │  │          │ │ Quant  │ │ Tracks  │  │ │
│  │  └──────────┘ └────────┘ └─────────┘  │ │
│  └────────────────────────────────────────┘ │
│       │                           │         │
│  ┌────▼──────────┐        ┌───────▼──────┐  │
│  │ internal/input│        │internal/output│  │
│  │               │        │              │  │
│  │ sensor/  data-│        │ cv.go        │  │
│  │          source/       │ midi.go      │  │
│  │               │        │ osc.go       │  │
│  └───────────────┘        └──────────────┘  │
└─────────────────────────────────────────────┘
```

### Hardware Abstraction Layer (HAL)

Hardware sensors and data sources implement the same `InputSource` interface. Output targets implement `CVOutput`, `MIDIOutput`, or `OSCOutput`.

```go
// internal/input/source.go

// InputSource is the unified interface for all input streams —
// hardware sensors and software data sources alike.
type InputSource interface {
    // Read returns the current normalized value (0.0 to 1.0 unipolar,
    // or -1.0 to 1.0 bipolar).
    Read() float64

    // Subscribe registers a callback for event-driven sources
    // (OTEL, WebSocket, MQTT, Redis SUBSCRIBE). For polled sources,
    // the callback fires after each poll.
    Subscribe(func(value float64, ts time.Time))

    // Configure sets sample/poll rate and filtering parameters.
    Configure(cfg SourceConfig) error

    // Calibrate runs auto-range learning or zero-offset correction.
    Calibrate(ctx context.Context) error

    // Status returns the current connection state.
    Status() ConnectionState

    // Close releases resources (connections, file handles, goroutines).
    Close() error
}

type ConnectionState int

const (
    Connected ConnectionState = iota
    Disconnected
    Stale    // no new data within 3x poll interval
    Error
)

type SourceConfig struct {
    SampleRate  float64       // Hz (hardware) or polls/sec (data source)
    FilterType  FilterType    // MovingAvg, LowPass, None
    FilterParam float64       // window size or cutoff frequency
    DeadBand    float64       // ignore changes smaller than this
    SlewLimit   float64       // max change per second (0 = unlimited)
    RangeMin    float64       // for normalization
    RangeMax    float64       // for normalization
    AutoRange   bool          // learn min/max from observed values
}
```

```go
// internal/output/cv.go

type CVOutput interface {
    WritePitch(channel int, voltage float64) error  // V/Oct calibrated
    WriteMod(channel int, voltage float64) error    // raw voltage
    WriteGate(channel int, on bool) error           // on/off
    Close() error
}
```

```go
// internal/output/midi.go

type MIDIOutput interface {
    NoteOn(channel, note, velocity uint8) error
    NoteOff(channel, note uint8) error
    CC(channel, number, value uint8) error
    PitchBend(channel uint8, value int16) error
    Clock() error
    Start() error
    Stop() error
    Close() error
}
```

```go
// internal/output/osc.go

type OSCOutput interface {
    Send(address string, args ...any) error  // e.g. "/grafophone/track1/pitch", 0.75
    Bundle(messages []OSCMessage) error       // atomic multi-message send
    Close() error
}
```

### Real-Time Constraints

| Task | Update Rate | Latency Budget | Priority |
|---|---|---|---|
| Gate output (trigger) | Event-driven | < 1 ms | Highest (dedicated goroutine, `SCHED_FIFO`) |
| Pitch CV update | Per step (1-1000 Hz) | < 1 ms | High |
| Mod CV update | 1-10 kHz | < 5 ms | High |
| Sensor reading | 1-1000 Hz (per type) | < 10 ms | Medium |
| MIDI output | Event-driven | < 3 ms | High |
| Data source poll | 0.1-100 Hz | < 100 ms | Medium |
| Display / Web UI | 10-30 Hz | < 50 ms | Low |
| Wireless comms | Async | < 100 ms | Low |

**Go-specific real-time notes:**
- Pin the CV output goroutine to an OS thread with `runtime.LockOSThread()` and set `SCHED_FIFO` via syscall for lowest jitter.
- Use `GOGC=off` or `debug.SetGCPercent(-1)` in the hot path, with manual `runtime.GC()` calls during idle periods (between sequences) to avoid GC pauses during playback.
- Pre-allocate all buffers. The sequencer engine hot path should be zero-allocation after initialization.

### Data Formats

**Internal representation:**
- Sensor/data source values: `float64`, normalized 0.0 to 1.0 (or -1.0 to 1.0 for bipolar).
- Pitch: MIDI note number (`float64`, e.g., 60.0 = C4, 60.5 = C4 + 50 cents).
- Velocity/modulation: 0.0 to 1.0 `float64` internally, scaled to output format at the driver level.
- Time: Tick-based (PPQ = 96 or 480 pulses per quarter note). `int64` tick counter.

**Preset/patch storage:** YAML for human-editable configuration files. JSON for machine interchange (web UI ↔ backend).

---

## Open Source References

### Essential Projects to Study

| Project | Platform | Relevance | GitHub |
|---|---|---|---|
| **EuroPi** | Pi Pico | MicroPython Eurorack, CV output design | Allen-Synthesis/EuroPi |
| **Ornament & Crime** | Teensy 3.2/4.1 | 16-bit quad CV, quantizers, sequencers | eh2k/squares-and-circles |
| **Mutable Instruments** | STM32 | Gold standard for CV/Gate firmware, DSP | pichenettes/eurorack |
| **CTAG Strämpler** | ESP32 | Dual-core audio/CV synthesis | ctag-fh-kiel/ctag-straempler |
| **Terminal Tedium** | RPi | DC-coupled Pi audio HAT for Eurorack | mxmxmx/terminal_tedium |
| **Zynthian** | RPi | Full Pi synth platform, PREEMPT_RT | zynthian/zynthian-sys |
| **Marcel Licence ESP32 Synth** | ESP32 | ESP32 audio synth reference | marcel-licence/esp32_basic_synth |
| **MiniDexed** | RPi (bare-metal) | Circle framework, USB MIDI gadget | probonopd/MiniDexed |
| **Monome Teletype** | ARM | Algorithmic sequencing, I2C ecosystem | monome/teletype |

### Key Libraries

**Go (core application):**
- **periph.io** — SPI/I2C/GPIO hardware abstraction for Raspberry Pi
- **pgx** — PostgreSQL driver (high performance, extended protocol)
- **go-redis** — Redis client with Pub/Sub and Streams
- **go-osc** — OSC message encoding/sending
- **gomidi** — MIDI message encoding/decoding
- **sarama** — Apache Kafka consumer/producer
- **paho.mqtt.golang** — MQTT 3.1.1/5.0 client
- **gorilla/websocket** — WebSocket client
- **go.opentelemetry.io/collector** — OTEL collector receiver components
- **go.opentelemetry.io/otel** — OTEL SDK for internal observability

**C++ (firmware / plugin):**
- **ESP-IDF** — Official ESP32 development framework (FreeRTOS)
- **Teensy Audio Library** — Graphical DSP block patching for Teensy
- **JUCE** — Cross-platform audio plugin framework (VST3/AU/CLAP)
- **iPlug2** — Lightweight plugin framework (liberal license)
- **DaisySP** — DSP library for Daisy/STM32

**Infrastructure:**
- **Patchbox OS** — Audio-optimized Linux distribution for Raspberry Pi
- **Circle** — Bare-metal C++ framework for Raspberry Pi (if needed)

---

## CI/CD Pipeline

The project uses **GitHub Actions** for all CI/CD. The pipeline enforces code quality, correctness, and produces release artifacts automatically.

For detailed tool configuration and pipeline specifications, see [SPECS.md — CI/CD Pipeline Specifications](SPECS.md#cicd-pipeline-specifications).

### Pipeline Stages

```
Push / PR
    │
    ▼
┌─────────────────────────────────────────────┐
│  Stage 1: Lint & Static Analysis            │
│                                             │
│  golangci-lint run (50+ linters)            │
│  go vet ./...                               │
│  govulncheck ./...                          │
│  YAML config schema validation              │
│  Markdown link check (DESIGN.md, SPECS.md)  │
└──────────────────┬──────────────────────────┘
                   │ pass
                   ▼
┌─────────────────────────────────────────────┐
│  Stage 2: Unit Tests                        │
│                                             │
│  go test -race -coverprofile=coverage.out ./...│
│  Coverage gate: >= 80%                      │
│  Fuzz tests: go test -fuzz (time-limited)   │
└──────────────────┬──────────────────────────┘
                   │ pass
                   ▼
┌─────────────────────────────────────────────┐
│  Stage 3: Build                             │
│                                             │
│  Build matrix:                              │
│    linux/amd64  (dev/CI)                    │
│    linux/arm64  (Raspberry Pi)              │
│    linux/arm    (Pi Zero)                   │
│    darwin/arm64 (macOS, dev)                │
│                                             │
│  go build -ldflags (embed version, commit)  │
│  Upload artifacts                           │
└──────────────────┬──────────────────────────┘
                   │ pass
                   ▼
┌─────────────────────────────────────────────┐
│  Stage 4: Integration Tests                 │
│                                             │
│  Docker Compose services:                   │
│    Postgres, Redis, Prometheus, MQTT broker │
│  go test -tags=integration ./...            │
│  Test data source adapters against real     │
│  services (not mocks)                       │
└──────────────────┬──────────────────────────┘
                   │ pass
                   ▼
┌─────────────────────────────────────────────┐
│  Stage 5: Acceptance Tests                  │
│                                             │
│  End-to-end scenarios:                      │
│    Config → Source → Engine → Output        │
│  YAML config validation test suite          │
│  MIDI output byte-level verification        │
│  OSC output message verification            │
│  CV output value-level verification (mock   │
│    SPI/I2C backend)                         │
└──────────────────┬──────────────────────────┘
                   │ pass (main branch only)
                   ▼
┌─────────────────────────────────────────────┐
│  Stage 6: Release (tag push only)           │
│                                             │
│  goreleaser: build, package, publish        │
│  GitHub Release with checksums              │
│  Container image → ghcr.io                  │
│  CHANGELOG.md auto-generation               │
└─────────────────────────────────────────────┘
```

### Branch Rules

| Branch | Triggers | Required Checks |
|---|---|---|
| `main` | Push, PR | All stages (1-5). PRs require passing checks + 1 review. |
| `feature/**` | Push, PR to main | Stages 1-4. Integration tests run on PR, not every push. |
| `v*` (tags) | Tag push | Full pipeline + Stage 6 (release). |
| `claude/*` | Push | Stages 1-3 (lint, test, build). |

---

## Build Artifacts & Release Process

### Artifacts

| Artifact | Format | Target | Contents |
|---|---|---|---|
| `grafophone-linux-amd64` | Binary (tar.gz) | Dev machines, servers | CLI binary + example configs |
| `grafophone-linux-arm64` | Binary (tar.gz) | Raspberry Pi 3/4/5 (64-bit) | CLI binary + example configs |
| `grafophone-linux-arm` | Binary (tar.gz) | Raspberry Pi Zero (W) / Pi 1 (32-bit) | CLI binary + example configs |
| `grafophone-darwin-arm64` | Binary (tar.gz) | macOS (Apple Silicon) dev | CLI binary + example configs |
| `grafophone-<version>.deb` | Debian package | Pi (apt install) | Binary + systemd unit + default config |
| `ghcr.io/leftathome/grafophone` | OCI container image | Docker/Podman on Pi or server | Multi-arch (amd64, arm64) |
| `grafocal-linux-arm64` | Binary (tar.gz) | Pi 3/4/5 | Standalone calibration tool (linux-only: requires hardware access) |
| `grafocal-linux-arm` | Binary (tar.gz) | Pi Zero (W) / Pi 1 | Standalone calibration tool |

### Versioning

**Semantic Versioning 2.0.0** (`MAJOR.MINOR.PATCH`):

| Version Component | Incremented When |
|---|---|
| **MAJOR** | Breaking changes to config schema, HAL interfaces, or CLI flags |
| **MINOR** | New data source adapters, new mapping modes, new output types, new sequencer features |
| **PATCH** | Bug fixes, calibration improvements, documentation, dependency updates |

**Pre-release tags**: `v0.1.0-alpha.1`, `v0.2.0-beta.1`, `v1.0.0-rc.1`

The project starts at `v0.1.0`. Semantic versioning commitments (backwards compatibility guarantees) begin at `v1.0.0`.

### Version Embedding

Version, commit SHA, and build date are embedded at build time via `-ldflags`:

```bash
go build -ldflags "-X main.version=v0.3.1 -X main.commit=$(git rev-parse HEAD) -X main.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

`grafophone --version` outputs: `grafophone v0.3.1 (commit abc1234, built 2026-03-15T10:30:00Z)`

### Release Process

1. **Development** happens on `feature/*` or `claude/*` branches, merged to `main` via PR.
2. **Changelog** entries are added to `CHANGELOG.md` under an `## [Unreleased]` section using [Keep a Changelog](https://keepachangelog.com/) format.
3. **Release** is triggered by pushing a semver tag:
   ```bash
   git tag -a v0.3.0 -m "Release v0.3.0: Add Prometheus and Redis data sources"
   git push origin v0.3.0
   ```
4. **GoReleaser** (via GitHub Actions) automatically:
   - Builds cross-compiled binaries for all target platforms
   - Creates `.tar.gz` archives with binary + example configs + LICENSE
   - Builds `.deb` packages for Raspberry Pi
   - Builds and pushes multi-arch container images to `ghcr.io`
   - Creates a GitHub Release with changelog, checksums (`SHA256SUMS`), and attached artifacts
   - Updates the `CHANGELOG.md` `[Unreleased]` section to the new version heading
5. **Container images** are tagged with both the semver tag and `latest`.

### GoReleaser Configuration

The project uses [GoReleaser](https://goreleaser.com/) for reproducible, cross-platform release builds. Configuration lives in `.goreleaser.yml` at the repository root.

---

## Development Phases

### Phase 1: Foundation (`v0.1.0`)
- Go module scaffolding, CI pipeline (lint + test + build)
- `InputSource` and `CVOutput` interfaces
- Ring buffer implementation with tests
- Single data source adapter (Prometheus) → single MIDI output
- YAML config loading and validation
- Unit test coverage >= 80%

### Phase 2: Multi-Source Sequencer (`v0.2.0`)
- Postgres, Redis, REST, WebSocket data source adapters
- Mapping engine with configurable curves (linear, log, S-curve)
- Scale quantizer with built-in scale library
- Internal clock with division/multiplication
- 4-track sequencer with record/playback/loop
- MIDI output (USB) + OSC output
- Integration tests against real services (Docker Compose)

### Phase 3: Hardware I/O (`v0.3.0`)
- `periph.io` SPI/I2C drivers for DAC8568, MCP4728
- CV/Gate output on Raspberry Pi
- Hardware sensor drivers (IMU, environmental, distance)
- DAC calibration routines (two-point + multi-point lookup table)
- `.deb` package for Pi deployment
- Acceptance test suite

### Phase 4: Full Platform (`v0.4.0` → `v1.0.0`)
- OTEL receiver (gRPC + HTTP)
- MQTT, Kafka data source adapters
- Euclidean rhythm generation
- Parameter locks, probability, ratcheting
- Multi-track polymetric sequencing
- Preset save/load
- Web UI for configuration and monitoring
- BLE MIDI output
- MIDI clock sync (send and receive)

### Phase 5: Ecosystem (`v1.x`)
- CLAP/VST3 companion plugin (C++, separate repo/build)
- ESP32 firmware for wireless sensor nodes (C++, separate repo/build)
- Per-VCO auto-calibration
- MPE MIDI support
- Grafana plugin for bidirectional integration

---

## License

TBD — Consider GPLv3 (like Mutable Instruments) or MIT for maximum community adoption.
