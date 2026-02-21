# Grafophone: Sensor-to-Synthesizer Sequencer

## Vision

Grafophone transforms real-world data streams into musical sequences for analog and digital synthesizers. It ingests time-series data from two classes of sources:

1. **Hardware sensors** — IMUs, environmental sensors, touch surfaces, distance sensors, microphones (connected via I2C/SPI/ADC/BLE)
2. **Software data sources** — databases (Postgres, Redis), observability platforms (Grafana, Prometheus), OpenTelemetry endpoints, REST/GraphQL APIs, and arbitrary time-series feeds

Data from either source class is mapped to musical parameters and output as CV/Gate signals, MIDI, and/or VST plugin automation.

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
    endpoint: "http://prometheus:9090"
    query: "avg(rate(node_cpu_seconds_total{mode!='idle'}[1m]))"
    poll_interval: 1s
    value_path: "result[0].value[1]"    # JSONPath to extract scalar
    range: [0.0, 1.0]                    # expected value range for normalization

  - name: "active_users"
    type: postgres
    connection: "postgresql://user:pass@host/db"
    query: "SELECT count(*) FROM sessions WHERE active = true"
    poll_interval: 5s
    range: [0, 10000]

  - name: "error_events"
    type: otel_receiver
    protocol: grpc
    port: 4317
    filter: "severity >= ERROR"
    mode: push                            # event-driven, not polled
    # Each matching event produces a trigger/gate

  - name: "redis_queue_depth"
    type: redis
    connection: "redis://localhost:6379"
    command: "LLEN work_queue"
    poll_interval: 500ms
    range: [0, 1000]

  - name: "btc_price"
    type: websocket
    url: "wss://stream.binance.com/ws/btcusdt@trade"
    value_path: "p"                       # price field from JSON message
    range: [20000, 100000]
```

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

### Sensor Bus Architecture (Hardware)

- **I2C bus** (400 kHz Fast Mode): IMU, environmental, light, ToF sensors. Use TCA9548A multiplexer for address conflicts (up to 64 buses).
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
- **1V/Oct pitch CV**: 1 volt per octave, the dominant standard. 0V = C0 or configurable reference. 5V range = 5 octaves.
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
- **DIN-5 MIDI**: Traditional 5-pin MIDI out via UART + optocoupler circuit.
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
1. **CLAP** (CLever Audio Plugin): Modern, open-source, extensible. Best for new development. Supports per-note modulation natively.
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
| **Language** | Python (UI/mapping) + C (real-time CV loop) | Best of both worlds |

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

### Module Decomposition

```
┌─────────────────────────────────────────────┐
│                 Application                  │
│                                              │
│  ┌─────────┐ ┌──────────┐ ┌──────────────┐ │
│  │   UI    │ │  Preset  │ │  Calibration │ │
│  │ Manager │ │  Storage │ │   Routines   │ │
│  └────┬────┘ └────┬─────┘ └──────┬───────┘ │
│       │           │              │          │
│  ┌────▼───────────▼──────────────▼────────┐ │
│  │          Sequencer Engine              │ │
│  │                                        │ │
│  │  ┌──────────┐ ┌────────┐ ┌─────────┐  │ │
│  │  │ Recorder │ │ Mapper │ │ Clocked │  │ │
│  │  │ /Buffer  │ │        │ │Sequencer│  │ │
│  │  └──────────┘ └────────┘ └─────────┘  │ │
│  └────────────────────────────────────────┘ │
│       │                           │         │
│  ┌────▼──────┐            ┌───────▼──────┐  │
│  │  Sensor   │            │   Output     │  │
│  │  Drivers  │            │   Drivers    │  │
│  │ (HAL)     │            │   (HAL)      │  │
│  └───────────┘            └──────────────┘  │
└─────────────────────────────────────────────┘
```

### Hardware Abstraction Layer (HAL)

Abstract all input reading and output writing behind interfaces so the same sequencer engine runs on Pi, ESP32, Teensy, or in software (VST plugin). Hardware sensors and data sources implement the same `InputSource` interface:

```
InputSource:                            # unified interface for all inputs
  read() -> float                       # normalized 0.0-1.0 (or -1.0 to 1.0)
  subscribe(callback)                   # for event-driven sources (OTEL, WS, MQTT)
  configure(sample_rate, filter)        # poll interval or sample rate
  calibrate()                           # auto-range learning
  status() -> ConnectionState           # connected / error / stale

HardwareSensor(InputSource):            # I2C/SPI/ADC sensor implementation
  bus_address, channel, ...

DataSource(InputSource):                # network/DB data source implementation
  endpoint, query, auth, value_path, ...

CVOutput:
  write_pitch(channel, voltage)         # V/Oct calibrated
  write_mod(channel, voltage)           # raw voltage
  write_gate(channel, state)            # on/off

MIDIOutput:
  note_on(channel, note, velocity)
  note_off(channel, note)
  cc(channel, number, value)
  clock()
```

### Real-Time Constraints

| Task | Update Rate | Latency Budget | Priority |
|---|---|---|---|
| Gate output (trigger) | Event-driven | < 1 ms | Highest (ISR) |
| Pitch CV update | Per step (1-1000 Hz) | < 1 ms | High |
| Mod CV update | 1-10 kHz | < 5 ms | High |
| Sensor reading | 1-1000 Hz (per type) | < 10 ms | Medium |
| MIDI output | Event-driven | < 3 ms | High |
| Display refresh | 10-30 Hz | < 50 ms | Low |
| Wireless comms | Async | < 100 ms | Low |

### Data Formats

**Internal representation:**
- Sensor values: 32-bit float, normalized 0.0 to 1.0 (or -1.0 to 1.0 for bipolar).
- Pitch: MIDI note number (float, e.g., 60.0 = C4, 60.5 = C4 + 50 cents).
- Velocity/modulation: 0.0 to 1.0 float internally, scaled to output format at the driver level.
- Time: Tick-based (PPQ = 96 or 480 pulses per quarter note).

**Preset/patch storage:** JSON or MessagePack for sensor mappings, scale selections, clock settings, sequence data.

---

## Open Source References

### Essential Projects to Study

| Project | Platform | Relevance | GitHub |
|---|---|---|---|
| **EuroPi** | Pi Pico | MicroPython Eurorack, CV output design | Allen-Synthesis/EuroPi |
| **Ornament & Crime** | Teensy 3.2/4.1 | 16-bit quad CV, quantizers, sequencers | eh2k/squares-and-circles |
| **Mutable Instruments** | STM32 | Gold standard for CV/Gate firmware, DSP | pichenettes/eurorack |
| **CTAG Strammer** | ESP32 | Dual-core audio/CV synthesis | ctag-fh-kiel/ctag-straempler |
| **Terminal Tedium** | RPi | DC-coupled Pi audio HAT for Eurorack | mxmxmx/terminal_tedium |
| **Zynthian** | RPi | Full Pi synth platform, PREEMPT_RT | zynthian/zynthian-sys |
| **Marcel Licence ESP32 Synth** | ESP32 | ESP32 audio synth reference | marcel-licence/esp32_basic_synth |
| **MiniDexed** | RPi (bare-metal) | Circle framework, USB MIDI gadget | probonopd/MiniDexed |
| **Monome Teletype** | ARM | Algorithmic sequencing, I2C ecosystem | monome/teletype |

### Key Libraries

- **JUCE** — Cross-platform audio plugin framework (VST3/AU/CLAP)
- **iPlug2** — Lightweight plugin framework (liberal license)
- **clap-juce-extensions** — CLAP format support for JUCE projects
- **DaisySP** — DSP library for Daisy/STM32 (oscillators, filters, effects)
- **Teensy Audio Library** — Graphical DSP block patching for Teensy
- **Circle** — Bare-metal C++ framework for Raspberry Pi
- **Patchbox OS** — Audio-optimized Linux distribution for Raspberry Pi

---

## Development Phases

### Phase 1: Proof of Concept
- Single sensor (IMU) → single CV output (pitch)
- Hardcoded linear mapping, chromatic quantization
- ESP32 + MCP4728 + op-amp output stage
- Validate V/Oct accuracy with tuner

### Phase 2: Multi-Channel Sequencer
- 4 sensor inputs → 4 CV + 4 gate outputs
- Configurable mapping engine with multiple curves
- Scale quantization with selectable scales
- Internal clock with division/multiplication
- Record/playback/loop of sensor streams
- MIDI output (USB)

### Phase 3: Full Platform
- Raspberry Pi version with display UI
- Euclidean rhythm generation
- Parameter locks, probability, ratcheting
- Multi-track polymetric sequencing
- Preset save/load
- WiFi/BLE wireless sensor support
- MIDI clock sync (send and receive)

### Phase 4: Software Integration
- CLAP/VST3 companion plugin
- OSC output for software synths
- Web-based configuration interface
- Per-VCO auto-calibration
- MPE MIDI support

---

## License

TBD — Consider GPLv3 (like Mutable Instruments) or MIT for maximum community adoption.
