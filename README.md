# Grafophone

Grafophone transforms real-world data streams into musical sequences for analog and digital synthesizers. It ingests time-series data from two classes of sources:

- **Hardware sensors** — IMUs, environmental sensors, touch surfaces, distance sensors (I2C/SPI/ADC/BLE)
- **Software data sources** — Prometheus, PostgreSQL, Redis, REST/GraphQL APIs, WebSockets, MQTT, OpenTelemetry endpoints

Data is mapped to musical parameters and output as **MIDI** (USB and BLE), **OSC**, and **CV/Gate** signals.

```
  Prometheus ─┐                              ┌─ OSC
  PostgreSQL ─┤                              ├─ USB MIDI
       Redis ─┤  ┌────────────────────────┐  ├─ BLE MIDI
    REST API ─┼──│    Sequencer Engine     │──┤
   WebSocket ─┤  │  map · quantize · step  │  ├─ CV/Gate
        MQTT ─┤  └────────────────────────┘  ├─ CLAP/VST3
        OTEL ─┘                              └─ DIN-5 MIDI
```

## Status

**Pre-alpha.** Design and specification phase. No runnable code yet.

See [TODO.md](TODO.md) for the implementation plan.

## Documentation

| Document | Contents |
|---|---|
| [DESIGN.md](DESIGN.md) | Architecture, sensor input layer, sequencer engine, output layer, hardware platform, software design |
| [SPECS.md](SPECS.md) | CV/Gate voltage standards, DAC selection, circuits, data source protocols, MIDI electrical specs, CI/CD pipeline |
| [TODO.md](TODO.md) | Implementation plan — 7 milestones, ~60 tasks with exit criteria |
| [CHANGELOG.md](CHANGELOG.md) | Release history |
| [AGENTS.md](AGENTS.md) | Contributor guidelines for AI-assisted development |

## MVP Roadmap

1. **Go service** — Remote data sources (Prometheus, REST, etc.) to OSC and USB/BLE MIDI output
2. **DAW plugin** — CLAP/VST3 companion that receives OSC and exposes values as automatable parameters
3. **Hardware** — Raspberry Pi with sensor input, CV/Gate output, and DAC calibration

## Architecture

The system has four layers:

**Input** — Data source adapters poll or subscribe to external systems and normalize values to 0.0–1.0 floats. Each source runs in its own goroutine with configurable poll interval, reconnection backoff, and staleness detection.

**Mapping** — Curves (linear, log, exponential, sigmoid) scale normalized values to musical ranges. A pitch quantizer snaps values to configurable scales (major, minor, pentatonic, custom). Dead-band filtering and slew limiting smooth the output.

**Sequencer** — An internal clock (20–300 BPM, 96 PPQ) drives up to 4 independent tracks. Each track can run in live mode (passthrough), or record and loop sequences. Euclidean rhythms, per-step probability, ratcheting, swing, and parameter locks add generative variation.

**Output** — MIDI (USB, BLE, DIN-5), OSC over UDP, and CV/Gate via SPI/I2C DACs on Raspberry Pi.

## Configuration

Grafophone is configured via YAML. Environment variables are expanded with `${VAR}` syntax:

```yaml
sources:
  - name: request_rate
    type: prometheus
    endpoint: "${PROMETHEUS_URL}"
    query: 'rate(http_requests_total[5m])'
    poll_interval: 5s
    range: { min: 0, max: 1000 }

outputs:
  - name: synth
    type: osc
    address: "127.0.0.1:9000"
    path: "/grafophone/pitch"
```

## Building

Requires Go >= 1.23.

```sh
make build       # build binary
make test        # run unit tests
make lint        # run golangci-lint
make integration # run integration tests (requires Docker)
```

## License

TBD
