# Grafophone: Technical Specifications

Detailed hardware, electrical, and data source specifications for the Grafophone sensor-to-synthesizer sequencer. Referenced from [DESIGN.md](DESIGN.md).

---

## Table of Contents

1. [CV/Gate Output Specifications](#cvgate-output-specifications)
2. [DAC Selection](#dac-selection)
3. [Output Stage Circuits](#output-stage-circuits)
4. [Gate/Trigger Circuits](#gatetrigger-circuits)
5. [Voltage References](#voltage-references)
6. [Sensor Input Specifications](#sensor-input-specifications)
7. [ADC Selection](#adc-selection)
8. [Data Source Specifications](#data-source-specifications)
9. [Protection Circuits](#protection-circuits)
10. [Power Supply](#power-supply)
11. [MIDI Electrical Specifications](#midi-electrical-specifications)
12. [Calibration](#calibration)
13. [CI/CD Pipeline Specifications](#cicd-pipeline-specifications)

---

## CV/Gate Output Specifications

### Voltage Standards

| Signal Type | Voltage Range | Resolution | Standard |
|---|---|---|---|
| Pitch CV | 0V to +10V (10 octaves) or 0V to +5V (5 octaves) | 16-bit (0.15 mV/step) | 1V/Oct (Moog/Euro) |
| Gate | 0V (off) / +5V or +10V (on) | Binary | Eurorack: >+2.5V = on |
| Trigger | +5V or +10V pulse, 5-10 ms | Binary | Rising edge, >+2.5V threshold |
| Mod CV (unipolar) | 0V to +5V or 0V to +10V | 12-bit or 16-bit | General purpose |
| Mod CV (bipolar) | -5V to +5V or -2.5V to +2.5V | 12-bit or 16-bit | LFO, pitch bend |
| Clock | +5V pulse, 5-15 ms | Binary | Rising edge |

### Output Impedance

- Target: 1k ohm (Eurorack convention)
- Series output resistor: 470 ohm to 1k ohm for short-circuit protection
- Must drive loads of 100k ohm (standard Eurorack input impedance) without voltage drop

### V/Oct Accuracy Targets

| Quality Level | Pitch Error | Voltage Tolerance | DAC Requirement |
|---|---|---|---|
| Minimum viable | +/- 5 cents | +/- 0.42 mV/V | 12-bit with calibration |
| Good | +/- 2 cents | +/- 0.17 mV/V | 14-bit or calibrated 12-bit |
| Excellent | +/- 1 cent | +/- 0.083 mV/V | 16-bit |
| Sub-cent | < 0.5 cents | < 0.042 mV/V | 16-bit with calibration |

---

## DAC Selection

### Pitch CV DACs (16-bit Required)

| DAC | Bits | Channels | Interface | Speed | Internal Ref | INL (typ) | Temp Coeff | Notes |
|---|---|---|---|---|---|---|---|---|
| **DAC8568** | 16 | 8 | SPI (50 MHz) | Fast | 2.5V | 4 LSB | 2 ppm/C | **Recommended**. Proven in Mutable Instruments Yarns. Easier than AD5668. |
| **AD5668** | 16 | 8 | SPI | Fast | 1.25V / 2.5V | 8 LSB | 5 ppm/C | Same pinout as DAC8568. Reported finicky by some builders. |
| **DAC8565** | 16 | 4 | SPI | Fast | 2.5V | 4 LSB | 2 ppm/C | Quad version. Used in Mutable Instruments Yarns. |

### Modulation CV DACs (12-bit Adequate)

| DAC | Bits | Channels | Interface | Internal Ref | Notes |
|---|---|---|---|---|---|
| **MCP4728** | 12 | 4 | I2C | 2.048V | Simple wiring, 0-4.096V on internal ref. Adequate for mod CV. |
| **MCP4922** | 12 | 2 | SPI | None | Faster than I2C options. Dual channel. |
| **MCP4725** | 12 | 1 | I2C | None | Simplest. Good for single-channel prototyping. |

### Resolution Analysis

| DAC Bits | Steps Over 10V | mV/Step | Cents/Step (1V/Oct) | Steps/Semitone |
|---|---|---|---|---|
| 8-bit | 256 | 39.06 | 46.88 | 0.21 |
| 10-bit | 1,024 | 9.77 | 11.72 | 0.85 |
| 12-bit | 4,096 | 2.44 | 2.93 | 3.41 |
| 14-bit | 16,384 | 0.61 | 0.73 | 13.65 |
| 16-bit | 65,536 | 0.15 | 0.18 | 54.61 |

**Key insight**: 12-bit is theoretically +/- 1.5 cents over 10V, but real-world INL errors on cheap 12-bit DACs can push actual accuracy to +/- 15 cents. 16-bit provides margin for software calibration to achieve sub-cent results.

---

## Output Stage Circuits

### Unipolar 0-10V from 0-2.5V DAC

Non-inverting amplifier with gain of 4:

```
                       R2 (30k)
                 ┌────/\/\/────┐
                 │      C1     │
                 │   ┌──||──┐  │
                 │   │(100pF)│  │
                 │   │       │  │       R_out (470)
                 │  ┌┴───────┴┐ └───────/\/\/──── JACK
                 └──┤-   OUT  ├─────── CV_OUT (0-10V)
                    │ OP-AMP  │
DAC_OUT ────────────┤+        │
                    └─────────┘
                         │
                        R3 (10k)
                         │
                        GND
```

Note: In the non-inverting configuration, the signal enters the `+` input. R2 feeds back from the output to the `-` input, and R3 connects from the `-` input to GND.

- Gain = 1 + R2/R3 = 1 + 30k/10k = 4
- DAC 0V → 0V out, DAC 2.5V → 10V out
- C1 (100 pF): Phase margin compensation, DAC glitch filtering
- R_out (470 ohm): Short-circuit protection, output impedance
- Power op-amp from +/-12V (Eurorack rails) or regulated +/-15V

### Bipolar +/-5V from 0-2.5V DAC

Gain + offset using precision voltage reference:

```
                      R2 (30k)
                ┌────/\/\/────┐
                │      C1     │
DAC_OUT ──R1────┤   ┌──||──┐  │
          (10k) │   │(100pF)│  │
                │  ┌┴───────┴┐ │
                ├──┤-   OUT  ├─┘──── CV_OUT (+/-5V)
                │  │ OP-AMP  │
         R4     │  │         │
V_REF ──/\/\/───┘  └────┬────┘
        (14.3k)         │
                   ┌─────┘
                   │
            V_REF (2.5V) via
            resistor divider
            to non-inv input
```

- Configuration produces +/-5V swing from 0-2.5V DAC input
- For +/-10V: adjust gain to 8x with appropriate offset
- Use precision resistors (0.1% tolerance or better) for accurate offset

### Component Specifications

| Component | Value | Tolerance | Notes |
|---|---|---|---|
| Gain resistors | Per circuit | 0.1% metal film | Critical for V/Oct accuracy |
| Feedback cap (C1) | 100 pF | C0G/NP0 ceramic | Low temperature coefficient |
| Output resistor | 470 ohm - 1k ohm | 1% | Short-circuit protection |
| Bypass caps | 100 nF + 10 uF per rail per op-amp | Ceramic + electrolytic | Decouple power rails |

---

## Gate/Trigger Circuits

### N-Channel MOSFET (10V Gate Output)

```
                                   +12V
                                     │
                                [10k pull-up]
                                     │
                          Drain ─────┴──── GATE_OUT (0V or ~10V)
                                │
                          BS170 / 2N7000
                                │
                          Source ──── GND

GPIO (3.3V) ────[10k]──── Gate
```

- Output is **inverted**: GPIO HIGH → MOSFET on → drain LOW (0V); GPIO LOW → MOSFET off → drain HIGH (~12V via pull-up)
- Inversion handled in software
- MOSFET Vgs threshold must be < 2.5V (BS170: 0.8-3.0V typ, 2N7000: 1.0-3.0V typ)
- Rise/fall time: < 100 ns (much faster than needed for gate/trigger)
- 10k pull-up to +12V gives ~1.2 mA source current (adequate for 100k input impedance loads)

### Logic Buffer (5V Gate Output)

```
GPIO (3.3V) ──── IN ┌────────┐ OUT ──── GATE_OUT (5V)
                     │74HCT125│
              OE ────┤  (5V)  │
              GND     └────────┘
```

- 74HCT125 or 74HCT244 powered from +5V
- TTL-compatible input thresholds: VIL < 0.8V, VIH > 2.0V (accepts 3.3V directly)
- Non-inverting
- Octal versions (74HCT244) provide 8 gate outputs from one IC

### Gate Specifications

| Parameter | Specification |
|---|---|
| OFF voltage | < 0.3V |
| ON voltage (Eurorack) | +5V to +10V |
| ON voltage (Doepfer standard) | +5V |
| Rise time | < 1 ms (< 100 us preferred for triggers) |
| Trigger pulse width | 5-10 ms (configurable) |
| Source current | > 1 mA at rated voltage |
| Short-circuit tolerant | Yes (current limited by pull-up resistor) |

---

## Voltage References

### Recommended References

| Part | Voltage | Initial Accuracy | Temp Coefficient | Package | Use Case |
|---|---|---|---|---|---|
| **REF5025** | 2.5V | +/- 0.05% | 3 ppm/C | SOIC-8 | Primary pitch CV reference |
| **MAX6133A** | 2.5V | +/- 0.06% | 7 ppm/C | SOT-23 | Compact, recommended by AD app notes |
| **LM4040-2.5** | 2.5V | +/- 0.1% | 100 ppm/C | SOT-23 | Budget option, used in Mutable Instruments |
| **REF02** | 5.0V | +/- 0.3% | 8.5 ppm/C | DIP-8 | Easy prototyping |

### Temperature Drift Impact

At 3 ppm/C (REF5025) with 2.5V reference:
- Drift = 3 ppm/C × 2.5V = 7.5 uV/C
- Over a 10V output (4x gain): 30 uV/C
- Musical impact: 0.036 cents/C — negligible
- Over 20C temperature swing: 0.72 cents — inaudible

At 100 ppm/C (LM4040):
- Drift = 100 ppm/C × 2.5V = 250 uV/C
- Over 10V output: 1 mV/C
- Musical impact: 1.2 cents/C
- Over 20C swing: 24 cents — audible, but acceptable for non-pitch CV

**Conclusion**: Use REF5025 or MAX6133A for pitch CV. LM4040 acceptable for modulation CV only.

---

## Sensor Input Specifications

### Recommended Sensors

#### Motion/Gesture

| Sensor | DOF | Interface | Sample Rate | Resolution | Price | Notes |
|---|---|---|---|---|---|---|
| **MPU-6050** | 6 (accel+gyro) | I2C (400 kHz) | Up to 1 kHz | 16-bit | ~$2 | Most common, adequate for gesture. Noisy, needs filtering. |
| **BNO055** | 9 (+ fusion) | I2C | 100 Hz (fusion) | 16-bit | ~$10 | On-chip sensor fusion (quaternions). Simplifies IMU code significantly. |
| **LSM6DSO** | 6 (accel+gyro) | I2C/SPI | Up to 6.66 kHz | 16-bit | ~$5 | Better noise floor than MPU-6050. SPI option for speed. |
| **ICM-42688-P** | 6 | I2C/SPI | Up to 32 kHz | 16-bit | ~$4 | Latest generation, lowest noise, FIFO buffer. |

#### Environmental

| Sensor | Measures | Interface | Sample Rate | Resolution | Notes |
|---|---|---|---|---|---|
| **BME280** | Temp/humidity/pressure | I2C/SPI | 1-100 Hz | 20-bit pressure | Most popular. Good for slow generative parameters. |
| **BME680** | Temp/humidity/pressure/gas | I2C/SPI | 1-10 Hz | 20-bit + gas resistance | Air quality index adds creative mapping option. |
| **SHT40** | Temp/humidity | I2C | 1-10 Hz | 16-bit | High accuracy (+/- 0.2C). |

#### Light

| Sensor | Measures | Interface | Range | Notes |
|---|---|---|---|---|
| **TSL2591** | Visible + IR lux | I2C | 188 uLux to 88 kLux | Very wide range, high sensitivity. |
| **VEML7700** | Ambient light | I2C | 0 to 120 kLux | Simple, wide range. |
| **Phototransistor** | Light intensity (analog) | ADC | Varies | Cheapest, fastest response for light gates. |

#### Distance

| Sensor | Technology | Range | Interface | Update Rate | Notes |
|---|---|---|---|---|---|
| **VL53L0X** | Time-of-flight laser | 30-1200 mm | I2C | 50 Hz | Accurate, unaffected by surface color. |
| **VL53L1X** | Time-of-flight laser | 40-4000 mm | I2C | 50 Hz | Longer range version. |
| **HC-SR04** | Ultrasonic | 20-4000 mm | GPIO (trig/echo) | 40 Hz | Cheap, wide beam, affected by temperature. |
| **Sharp GP2Y0A21** | Infrared | 100-800 mm | Analog | 25 Hz | Analog output, easy to read. |

#### Touch/Pressure

| Sensor | Type | Channels | Interface | Notes |
|---|---|---|---|---|
| **MPR121** | Capacitive touch | 12 | I2C | Configurable sensitivity, electrode proximity detection. |
| **FSR 402** | Force-sensing resistor | 1 (analog) | ADC | Pressure-sensitive. Good for velocity/aftertouch. |
| **Velostat/Eeonyx** | Piezoresistive fabric | Custom | ADC | DIY pressure pads. |
| **ESP32 built-in** | Capacitive touch | 10/14 | GPIO | Free on ESP32/S3, no external hardware. |

#### Audio Input

| Sensor | Type | Interface | Notes |
|---|---|---|---|
| **INMP441** | MEMS digital mic | I2S | High quality, no ADC needed. Good for FFT/envelope. |
| **MAX4466** | Electret + preamp | Analog (ADC) | Adjustable gain, simple. |
| **MAX9814** | Electret + AGC | Analog (ADC) | Auto gain control, hands-off. |

---

## ADC Selection

### For Reading Analog Sensors

| ADC | Bits | Channels | Interface | Max Sample Rate | Notes |
|---|---|---|---|---|---|
| **MCP3208** | 12 | 8 | SPI | 100 kSPS | Proven in Terminal Tedium. Good general purpose. |
| **MCP3008** | 10 | 8 | SPI | 200 kSPS | Cheaper, adequate for most sensors. |
| **ADS1115** | 16 | 4 | I2C | 860 SPS | High resolution, built-in PGA. Slow — best for environmental. |
| **ADS1015** | 12 | 4 | I2C | 3300 SPS | Faster than ADS1115, less resolution. |
| **ADS8688** | 16 | 8 | SPI | 500 kSPS | High speed + high resolution. Expensive. |

### Platform Built-in ADCs

| Platform | ADC Bits | Channels | Notes |
|---|---|---|---|
| Raspberry Pi | None | — | External ADC required |
| ESP32 | 12-bit | 18 | Known nonlinearity; calibrate or use external ADC for precision |
| ESP32-S3 | 12-bit | 20 | Improved over original ESP32 |
| Teensy 4.1 | 12-bit | Multiple | Good performance, fast conversion |
| STM32F4 | 12-bit | 16 | Generally more accurate than ESP32 |

---

## Data Source Specifications

For configuration YAML syntax and adapter architecture, see [DESIGN.md — Data Sources](DESIGN.md#data-sources).

### Supported Source Types

| Source Type | Protocol | Directionality | Typical Latency | Auth Methods |
|---|---|---|---|---|
| **PostgreSQL** | TCP (libpq) | Poll (SQL query) | 1-50 ms | User/pass, SSL, mTLS |
| **MySQL** | TCP | Poll (SQL query) | 1-50 ms | User/pass, SSL |
| **SQLite** | File | Poll (SQL query) | < 1 ms | File permissions (no network auth) |
| **Redis** | TCP (RESP) | Poll (command) or Push (SUBSCRIBE, keyspace) | < 1 ms | Password, ACL, TLS |
| **Prometheus** | HTTP (PromQL) | Poll (query API) | 10-200 ms | Basic auth, Bearer token, mTLS |
| **InfluxDB** | HTTP (Flux/InfluxQL) | Poll (query API) | 10-200 ms | Token, user/pass |
| **Grafana** | HTTP (JSON API) | Poll (dashboard/panel query) | 50-500 ms | API key, Bearer token |
| **OpenTelemetry** | gRPC or HTTP (OTLP) | Push (receiver endpoint) | < 10 ms | mTLS, API key headers |
| **REST API** | HTTP/HTTPS | Poll (GET/POST) | 10-500 ms | API key, OAuth2, Bearer token |
| **GraphQL** | HTTP/HTTPS | Poll (query) | 10-500 ms | API key, Bearer token | (shares REST adapter; sends POST with query body) |
| **WebSocket** | WS/WSS | Push (persistent connection) | < 10 ms | Token in handshake, custom headers |
| **MQTT** | TCP/TLS | Push (subscribe to topic) | < 10 ms | User/pass, TLS, client cert |
| **Kafka** | TCP | Push (consumer group) | 10-100 ms | SASL, TLS |

### Poll Interval Ranges

| Data Source Category | Minimum Interval | Typical Interval | Maximum Interval |
|---|---|---|---|
| Real-time feeds (WS, MQTT) | Event-driven (0 ms) | Event-driven | N/A |
| OTEL receiver | Event-driven (0 ms) | Event-driven | N/A |
| Redis (poll mode) | 100 ms | 500 ms - 1s | 60s |
| Prometheus / InfluxDB | 1s | 5-15s | 5 min |
| PostgreSQL / MySQL | 500 ms | 5-30s | 5 min |
| REST / GraphQL APIs | 1s | 10-60s | 15 min |
| Grafana dashboard API | 5s | 15-60s | 5 min |

### Value Extraction

Each data source adapter extracts a scalar float (or vector of floats) from the response:

| Source | Extraction Method | Example |
|---|---|---|
| Prometheus | PromQL result → `result[n].value[1]` | `avg(rate(http_requests_total[5m]))` → 142.7 |
| PostgreSQL | SQL result → column value from first row | `SELECT avg(response_ms) FROM requests` → 23.5 |
| Redis | Command result → direct value | `LLEN queue` → 847 |
| OTEL metrics | Metric name + label filter → data point value | `system.cpu.utilization{state=user}` → 0.73 |
| OTEL traces | Span filter → duration / count / attribute | Spans where `service.name=api` → avg duration 45ms |
| OTEL logs | Log filter → count per interval / severity level | `severity >= WARN` count in last 10s → 3 |
| REST/GraphQL | JSONPath on response body | `$.data.temperature` → 22.1 |
| WebSocket | JSONPath on each message | `$.price` → 43250.00 |
| MQTT | JSONPath on message payload | `$.sensor.value` → 0.85 |

### Normalization

Raw values are normalized to a 0.0-1.0 (unipolar) or -1.0 to 1.0 (bipolar) float range:

```
normalized = clamp((raw_value - range_min) / (range_max - range_min), 0.0, 1.0)
```

| Mode | Behavior |
|---|---|
| **Fixed range** | User specifies min/max. Values outside range are clamped. |
| **Auto-range** | Learn min/max from observed values over a configurable window. Expands range on new extremes. |
| **Logarithmic** | Apply log scaling before normalization. Useful for values spanning orders of magnitude (request rates, byte counts). |
| **Delta** | Normalize the rate of change rather than the absolute value. Useful for monotonic counters. |
| **Z-score** | Normalize relative to running mean and standard deviation. Useful for anomaly detection → musical surprise. |

### Credential Management

Connection strings, passwords, API keys, and tokens must never be hardcoded in YAML config files. The configuration loader supports environment variable substitution using `${VAR_NAME}` syntax.

| Mechanism | Usage | Example |
|---|---|---|
| **Environment variables** | Primary method. Substituted at config load time. | `connection: "${GRAFOPHONE_POSTGRES_URL}"` |
| **Env file** | For systemd deployments, use `EnvironmentFile=` directive. | `/etc/grafophone/env` with `GRAFOPHONE_POSTGRES_URL=postgres://...` |
| **Secrets manager** | For production. Fetch secrets at startup via SDK. | HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager |
| **File reference** | For TLS certs/keys. Config points to file path. | `tls_cert: "/etc/grafophone/tls/client.crt"` |

**Security requirements:**
- The config loader must refuse to start if an `${VAR_NAME}` reference resolves to an empty string (fail-closed).
- Config files must not be world-readable. Recommended permissions: `0600` owned by the grafophone service user.
- Log output must redact connection strings and tokens. Use `[REDACTED]` for any field containing `password`, `token`, `key`, or `secret`.
- SQL queries from config files are executed as-is (trusted admin input). The database user should have **read-only** permissions. The application does not perform query validation or sanitization.

### Connection Management

| Behavior | Specification |
|---|---|
| Reconnection | Exponential backoff: 1s, 2s, 4s, 8s, max 60s |
| Timeout (poll) | Configurable, default 5s. On timeout: hold last value. |
| Stale data | If no new value within 3x poll interval, mark source as stale. Configurable behavior: hold, zero, or trigger alert gate. |
| Connection pool | Reuse connections for DB sources (pool size configurable, default 2). |
| Health check | Periodic connectivity test. Expose per-source status via UI/API. |

### OTEL Receiver Specification

Grafophone can act as an OpenTelemetry collector endpoint, receiving telemetry pushed from instrumented applications:

| Parameter | Specification |
|---|---|
| Protocol | OTLP/gRPC (port 4317) and OTLP/HTTP (port 4318) |
| Supported signals | Metrics, Traces, Logs |
| Metric types handled | Gauge → CV value. Counter → delta rate → CV. Histogram → percentile → CV. |
| Trace handling | Span start → gate on. Span end → gate off. Duration → CV. Span attributes → mappable values. |
| Log handling | Log event → trigger pulse. Severity → velocity/CV. Body regex match → configurable trigger. |
| Filtering | By service name, metric name, span name, log severity, resource attributes |
| Authentication | mTLS (recommended), API key via header, or none (not recommended — accepts arbitrary telemetry) |
| Buffering | Ring buffer per metric/span stream, same as hardware sensor streams |

### Resource Limits

| Resource | Limit | Notes |
|---|---|---|
| Max concurrent data sources | 32 | Each source runs in its own async task/thread |
| Max poll rate (aggregate) | 100 queries/sec across all sources | Prevent overloading external systems |
| Response body size limit | 1 MB per query response | Discard oversized responses |
| Value history buffer | 4096 samples per source | Same ring buffer as hardware sensors |
| Memory per source (estimated) | ~50-200 KB | Connection state + buffer + config |

---

## Protection Circuits

### Input Protection (Eurorack CV/Audio In)

```
JACK ────┬──── TVS ────┬──── R_series (470-1k) ──── To ADC/Op-Amp Input
         │             │
         │            GND
         │
     Ferrite bead
     (optional, for EMI)
```

- **TVS diode**: Bidirectional, clamp voltage matching the input IC's maximum (3.3V for MCU ADC, +/-12V for op-amp).
- **Series resistor**: 470 ohm to 1k ohm after TVS, limits current to sensitive IC inputs.
- **Schottky steering diodes**: Alternative to TVS — one to Vcc, one to GND, clamps input to within one diode drop of rails.
- All inputs must survive continuous connection to +/-12V without damage.

### Output Protection

- 470 ohm to 1k ohm series resistor at each output jack (already included in output stage design).
- Op-amp inherent current limiting (typically 25-40 mA short-circuit current).
- Output must survive indefinite short to any voltage between power rails.

### Power Supply Protection

```
+12V_EURO ──── D1 (1N5817) ──── C1 (22uF) ──── +12V_LOCAL
-12V_EURO ──── D2 (1N5817) ──── C2 (22uF) ──── -12V_LOCAL
```

- **Reverse polarity protection**: Series Schottky diodes (1N5817) on both rails. ~200 mV forward drop.
- **Bypass capacitors**: 22 uF electrolytic + 100 nF ceramic on each rail, close to power connector.
- **Shrouded header**: Use keyed 10-pin or 16-pin Eurorack power header to prevent reversed insertion.
- **Pin 9-10 unconnected**: On 16-pin headers, leave CV/gate bus pins unconnected (Rossum convention) to avoid damage from reversed 16-to-10 cables.

### ESD Protection

- TVS diodes on all externally-facing connectors (jacks, USB, MIDI DIN).
- Place TVS as close to connector as possible with short, wide ground traces.
- Use low-capacitance variants (0.5-5 pF) for audio/CV signals to preserve bandwidth.
- No resistance before TVS (keep TVS path low-impedance for fast clamping).

---

## Power Supply

### Eurorack Power Budget

| Rail | Available (typical PSU) | Budget for Module |
|---|---|---|
| +12V | 1-2A shared | < 250 mA |
| -12V | 500 mA - 1A shared | < 50 mA (op-amps only) |
| +5V | 500 mA - 1A (if available) | < 500 mA (Pi/MCU) |

### Local Regulation

| Component | Regulator | Input | Output | Notes |
|---|---|---|---|---|
| +5V for MCU | 7805 or LM1117-5.0 | +12V Euro | +5V @ 500 mA | Or use Eurorack +5V rail if available |
| +3.3V for sensors | AMS1117-3.3 or MCP1700 | +5V | +3.3V @ 250 mA | Low dropout, low noise |
| +/-12V for op-amps | Direct from Eurorack | — | — | Add local 100 nF + 10 uF bypass |

### Battery Operation (ESP32)

| Parameter | Specification |
|---|---|
| Battery type | LiPo 3.7V, 850-2000 mAh |
| Regulator | MCP1700-3302E or HT7333 (LDO, low quiescent) |
| Active current (WiFi) | 160-240 mA |
| Active current (BLE only) | 95-130 mA |
| Deep sleep current | 10-150 uA (board dependent) |
| Expected runtime (850 mAh, BLE) | 6-9 hours active |
| Charging IC | TP4056 (USB micro-B, 1A charge) |

---

## MIDI Electrical Specifications

### DIN-5 MIDI Out

```
UART_TX (3.3V) ──── R_base (1k) ──── Base ┐
                                            │ NPN (BC547 / 2N3904)
                                     Emitter┘──── GND
                                            │
                                    Collector┘──── DIN Pin 5
                                                   │
                               +5V ── R1 (220) ────┤
                                                   │
                               +5V ── R2 (220) ──── DIN Pin 4
```

Or use a simple buffer/inverter IC (74HCT14) instead of transistor.

| Parameter | Specification |
|---|---|
| Baud rate | 31250 bps |
| Current loop | 5 mA (drives opto-isolator at receiving device) |
| Source resistors | 220 ohm on pins 4 and 5 |
| Connector | 5-pin DIN (180 degree) |
| Cable length | Up to 15 meters |

### USB MIDI (Gadget Mode)

| Platform | USB Port | Setup |
|---|---|---|
| Pi Zero (2) W | Micro-USB OTG | `dtoverlay=dwc2`, `modprobe g_midi` |
| Pi 4B | USB-C (power port) | `dtoverlay=dwc2`, `libcomposite`. Power via GPIO pins 2/4. |
| ESP32-S3 | Native USB | TinyUSB library, USB MIDI class |
| Teensy 4.x | Native USB | Built-in, zero config. `usb_midi.h` |

### BLE MIDI

| Parameter | Specification |
|---|---|
| Service UUID | 03B80E5A-EDE8-4B33-A751-6CE34EC4C700 |
| Characteristic UUID | 7772E5DB-3868-4112-A1A9-F2669D106BF3 |
| MTU | 23 bytes default (negotiate higher for lower latency) |
| Latency | 7.5-30 ms (connection interval dependent) |
| Range | ~10 meters typical |
| Platform support | ESP32 (native), Pi (BlueZ stack) |

---

## Calibration

### Two-Point Calibration (Minimum)

1. Set DAC to minimum code (0x0000). Measure actual output voltage with multimeter: V_low.
2. Set DAC to maximum code (0xFFFF). Measure actual output voltage: V_high.
3. Compute gain correction: `gain = (V_target_high - V_target_low) / (V_high - V_low)`
4. Compute offset correction: `offset = V_target_low - (gain * V_low)`
5. Apply in software: `dac_code = (target_voltage - offset) / gain * 65535 / V_ref`

Corrects for op-amp gain error and DAC offset, but not for DAC nonlinearity (INL).

### Multi-Point Lookup Table (Recommended)

1. Step DAC through codes at each semitone (or every 100 mV) across the full output range.
2. Measure actual output voltage at each point.
3. Build a correction table in firmware: `correction[i] = expected_code[i] - actual_code_for_target[i]`
4. At runtime, interpolate between table entries for arbitrary target voltages.
5. Store calibration data in non-volatile memory (EEPROM, flash, SD card).

This corrects for DAC INL, op-amp nonlinearity, and resistor tolerance errors. Essential for 12-bit DACs, beneficial for 16-bit.

### Per-VCO Auto-Calibration (Advanced)

1. Patch VCO output back to a frequency counter input on the module (audio input → ADC → FFT or zero-crossing detection).
2. Step through octaves: output 1V, 2V, 3V... and measure the actual VCO frequency at each step.
3. Build a per-VCO correction table that maps desired frequency → required DAC code for that specific oscillator.
4. Compensates for VCO tracking errors, temperature drift, and the entire analog signal chain.

### Calibration Frequency

- Factory calibration (multi-point lookup table) at manufacturing/assembly time
- User re-calibration available via menu/button sequence
- Auto-calibration on power-up (optional, if audio input is available)
- Temperature compensation: re-calibrate if ambient temperature changes significantly (>10C)

---

## CI/CD Pipeline Specifications

### Overview

All CI/CD runs on **GitHub Actions**. The pipeline is defined in `.github/workflows/` with separate workflow files for CI (every push/PR) and Release (tag push).

### Required Go Version

| Dependency | Version | Notes |
|---|---|---|
| **Go** | >= 1.23 | Use `actions/setup-go@v5` with version from `go.mod` |
| **golangci-lint** | >= 1.62 | Installed via `golangci/golangci-lint-action@v6` |
| **GoReleaser** | >= 2.5 | Installed via `goreleaser/goreleaser-action@v6` |
| **govulncheck** | latest | Installed via `go install golang.org/x/vuln/cmd/govulncheck@latest` |

### Stage 1: Lint & Static Analysis

**Workflow file**: `.github/workflows/ci.yml` (job: `lint`)

**Tool: `golangci-lint`** — aggregates 50+ Go linters in a single binary. Configured via `.golangci.yml`.

**Enabled linters (`.golangci.yml`):**

| Linter | Category | What It Catches |
|---|---|---|
| `govet` | Correctness | Printf format strings, struct tags, unreachable code |
| `errcheck` | Correctness | Unchecked error return values |
| `staticcheck` | Correctness | Bugs, simplifications, performance (SA*, S*, ST*, QF* rules) |
| `gosimple` | Simplification | Code that can be simplified |
| `unused` | Dead code | Unused functions, types, variables, constants |
| `ineffassign` | Correctness | Assignments to variables that are never read |
| `typecheck` | Correctness | Type-checking errors |
| `gocritic` | Style/Perf | Opinionated diagnostics (hugeParam, rangeValCopy, etc.) |
| `revive` | Style | Configurable superset of `golint` |
| `gofumpt` | Formatting | Stricter `gofmt` — enforces consistent formatting |
| `misspell` | Docs | Commonly misspelled English words in comments/strings |
| `bodyclose` | Correctness | Unclosed HTTP response bodies |
| `noctx` | Correctness | HTTP requests without context.Context |
| `exhaustive` | Correctness | Non-exhaustive switch statements on sum types |
| `gosec` | Security | Potential security issues (G101-G601 rules) |
| `prealloc` | Performance | Slice declarations that could be preallocated |

**Additional static analysis tools:**

| Tool | Purpose | Invocation |
|---|---|---|
| `go vet ./...` | Official Go static analyzer | Built-in, also run by golangci-lint |
| `govulncheck ./...` | Known vulnerability scanner for Go dependencies | Checks against Go vulnerability database |
| **YAML schema validation** | Validate example configs against JSON Schema | `go test ./internal/config/...` (schema test) |

**golangci-lint configuration (`.golangci.yml`):**

```yaml
run:
  timeout: 5m
  go: "1.23"

linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - gosimple
    - unused
    - ineffassign
    - typecheck
    - gocritic
    - revive
    - gofumpt
    - misspell
    - bodyclose
    - noctx
    - exhaustive
    - gosec
    - prealloc

linters-settings:
  gocritic:
    enabled-tags:
      - diagnostic
      - style
      - performance
  revive:
    rules:
      - name: blank-imports
      - name: context-as-argument
      - name: dot-imports
      - name: error-return
      - name: error-strings
      - name: exported
      - name: increment-decrement
      - name: var-naming
      - name: package-comments
        disabled: true   # allow packages without doc comments during early dev
  gosec:
    excludes:
      - G104   # allow unhandled errors in deferred Close() calls
  exhaustive:
    default-signifies-exhaustive: true

issues:
  exclude-use-default: false
  max-issues-per-linter: 0
  max-same-issues: 0
```

### Stage 2: Unit Tests

**Workflow file**: `.github/workflows/ci.yml` (job: `test`)

**Tools and flags:**

```bash
# Run all unit tests with race detector and coverage
go test -race -coverprofile=coverage.out -covermode=atomic ./...

# Upload coverage to Codecov (or similar)
# Coverage gate: fail if total coverage < 80%

# Fuzz tests (time-limited in CI, unlimited locally)
go test -fuzz=FuzzMapper -fuzztime=30s ./internal/mapper/...
go test -fuzz=FuzzQuantizer -fuzztime=30s ./internal/mapper/...
go test -fuzz=FuzzConfigParse -fuzztime=30s ./internal/config/...
```

**Unit test conventions:**

| Convention | Specification |
|---|---|
| Test file location | `*_test.go` alongside source files |
| Build tag for integration tests | `//go:build integration` — excluded from unit test runs |
| Build tag for acceptance tests | `//go:build acceptance` — excluded from unit test runs |
| Table-driven tests | Required for all functions with multiple input/output cases |
| Test helpers | Use `t.Helper()` for shared setup functions |
| Golden files | Use `testdata/` directories for expected output fixtures |
| Mocks | Use interfaces + hand-written mocks (no code generation frameworks) |
| Benchmarks | `Benchmark*` functions for hot-path code (mapper, quantizer, ring buffer) |

**Coverage requirements:**

| Package | Minimum Coverage |
|---|---|
| `internal/engine` | 85% |
| `internal/mapper` | 90% |
| `internal/buffer` | 90% |
| `internal/config` | 85% |
| `internal/input/datasource` | 80% |
| `internal/output` | 80% |
| **Overall** | **80%** |

**Fuzz targets:**

| Target | What It Fuzzes |
|---|---|
| `FuzzMapper` | Mapping curve functions with arbitrary float64 inputs — catch panics, NaN, Inf |
| `FuzzQuantizer` | Scale quantizer with arbitrary pitch values — ensure output is always in-scale |
| `FuzzConfigParse` | YAML config parser with arbitrary bytes — catch panics, ensure graceful errors |
| `FuzzNormalize` | Normalization function with edge-case ranges — division by zero, negative ranges |

### Stage 3: Build

**Workflow file**: `.github/workflows/ci.yml` (job: `build`)

**Build matrix:**

| GOOS | GOARCH | Target Platform | Artifact Name |
|---|---|---|---|
| `linux` | `amd64` | Dev machines, CI, Docker | `grafophone-linux-amd64` |
| `linux` | `arm64` | Raspberry Pi 3/4/5 (64-bit OS) | `grafophone-linux-arm64` |
| `linux` | `arm` (GOARM=6) | Raspberry Pi Zero (W) / Pi 1 (32-bit) | `grafophone-linux-arm` |
| `darwin` | `arm64` | macOS Apple Silicon (dev) | `grafophone-darwin-arm64` |

**Build command:**

```bash
CGO_ENABLED=0 GOOS=${os} GOARCH=${arch} go build \
  -ldflags "-s -w \
    -X main.version=${GITHUB_REF_NAME:-dev} \
    -X main.commit=$(git rev-parse --short HEAD) \
    -X main.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -o grafophone-${os}-${arch} \
  ./cmd/grafophone/
```

- `CGO_ENABLED=0`: Static binary, no libc dependency. Critical for cross-compilation and Pi deployment.
- `-s -w`: Strip debug symbols and DWARF — reduces binary size ~30%.
- Artifacts uploaded via `actions/upload-artifact@v4` for use in later stages.

### Stage 4: Integration Tests

**Workflow file**: `.github/workflows/ci.yml` (job: `integration`)

**Triggered**: On PRs to `main` and on `main` branch pushes. Skipped on feature branch pushes (too slow for rapid iteration).

**Infrastructure (Docker Compose):**

```yaml
# .github/docker-compose.integration.yml
# NOTE: All credentials below are ephemeral CI-only test fixtures.
# Never use these values in production. See DESIGN.md for credential management guidance.
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: grafophone_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports: ["5432:5432"]
    healthcheck:
      test: pg_isready -U test
      interval: 2s
      retries: 10

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass test
    ports: ["6379:6379"]
    healthcheck:
      test: redis-cli -a test ping
      interval: 2s
      retries: 10

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./testdata/prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]
    healthcheck:
      test: wget -qO- http://localhost:9090/-/healthy
      interval: 2s
      retries: 10

  mosquitto:
    image: eclipse-mosquitto:2
    volumes:
      - ./testdata/mosquitto.conf:/mosquitto/config/mosquitto.conf
    ports: ["1883:1883"]
    healthcheck:
      test: mosquitto_sub -t '$$SYS/broker/uptime' -C 1 -W 2
      interval: 2s
      retries: 10
```

Note: `testdata/mosquitto.conf` should set `allow_anonymous true` for CI use only. In production MQTT deployments, always require authentication.

**Test execution:**

```bash
# Start services
docker compose -f .github/docker-compose.integration.yml up -d --wait

# Run integration tests (all credentials are CI-only ephemeral values)
GRAFOPHONE_TEST_POSTGRES_URL="postgres://test:test@localhost:5432/grafophone_test?sslmode=prefer" \
GRAFOPHONE_TEST_REDIS_URL="redis://:test@localhost:6379" \
GRAFOPHONE_TEST_PROMETHEUS_URL="http://localhost:9090" \
GRAFOPHONE_TEST_MQTT_URL="tcp://localhost:1883" \
go test -race -tags=integration -count=1 -timeout=5m ./...

# Teardown
docker compose -f .github/docker-compose.integration.yml down -v
```

**Integration test scope:**

| Test Suite | What It Validates |
|---|---|
| `datasource/postgres_integration_test.go` | Connect, query, value extraction, reconnection on failure, connection pooling |
| `datasource/redis_integration_test.go` | GET/LLEN polling, SUBSCRIBE push, keyspace notifications, reconnection |
| `datasource/prometheus_integration_test.go` | PromQL instant query, range query, value extraction from result vector |
| `datasource/mqtt_integration_test.go` | Subscribe, receive message, value extraction, QoS 0 and 1 |
| `datasource/websocket_integration_test.go` | Connect to echo server, receive messages, value extraction, reconnection |
| `config/config_integration_test.go` | Load full multi-source config, validate all adapters initialize, health check |

### Stage 5: Acceptance Tests

**Workflow file**: `.github/workflows/ci.yml` (job: `acceptance`)

**Triggered**: On `main` branch only (merge commits). Not on PRs (integration tests provide sufficient confidence for PR review).

**Test execution:**

```bash
go test -race -tags=acceptance -count=1 -timeout=10m ./test/acceptance/...
```

**Acceptance test scope:**

End-to-end tests that exercise the full pipeline from config file through to output, with mocked hardware backends:

| Test Scenario | Description |
|---|---|
| `TestPrometheusToMIDI` | Prometheus query → mapper (linear) → quantizer (C major) → MIDI note on/off. Verify correct MIDI bytes. |
| `TestRedisToCV` | Redis LLEN → mapper (log) → CV output (mock SPI backend). Verify DAC codes match expected voltages. |
| `TestMultiSourceMultiTrack` | 4 data sources → 4 tracks → 4 outputs simultaneously. Verify independence and clock sync. |
| `TestOTELReceiverToGate` | Push OTLP log event → severity filter → gate trigger. Verify gate on/off timing. |
| `TestConfigReload` | Modify YAML config at runtime → verify sources reconnect, mappings update, no output glitch. |
| `TestGracefulShutdown` | Send SIGTERM → verify all connections closed, no panics, clean exit code. |
| `TestClockSync` | External MIDI clock input → verify sequencer follows tempo, outputs on correct beats. |
| `TestRecordPlaybackLoop` | Record 8 steps from data source, stop, play back, verify output matches recording. |

**Mock backends:**

- `internal/output/cv_mock.go`: Records all `WritePitch`/`WriteMod`/`WriteGate` calls with timestamps. Used to verify output correctness without hardware.
- `internal/output/midi_mock.go`: Captures MIDI byte stream. Assertions verify note/CC/clock messages.
- `internal/input/sensor/sensor_mock.go`: Replays recorded sensor data from golden files.

### Stage 6: Release

**Workflow file**: `.github/workflows/release.yml`

**Triggered**: On tag push matching `v*` (e.g., `v0.3.0`, `v1.0.0-rc.1`).

**Tool: GoReleaser** — builds, packages, and publishes release artifacts.

**GoReleaser configuration (`.goreleaser.yml`):**

```yaml
version: 2

project_name: grafophone

before:
  hooks:
    - go mod tidy
    - go generate ./...

builds:
  - id: grafophone
    main: ./cmd/grafophone/
    binary: grafophone
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
    goarch:
      - amd64
      - arm64
      - arm
    goarm:
      - "6"
    ignore:
      - goos: darwin
        goarch: arm
    ldflags:
      - -s -w
      - -X main.version={{.Version}}
      - -X main.commit={{.ShortCommit}}
      - -X main.date={{.Date}}

  - id: grafocal
    main: ./cmd/grafocal/
    binary: grafocal
    env:
      - CGO_ENABLED=0
    goos:
      - linux
    goarch:
      - arm64
      - arm
    goarm:
      - "6"
    ldflags:
      - -s -w
      - -X main.version={{.Version}}

archives:
  - id: grafophone-archive
    builds:
      - grafophone
    name_template: "{{ .ProjectName }}-{{ .Version }}-{{ .Os }}-{{ .Arch }}{{ if .Arm }}v{{ .Arm }}{{ end }}"
    format: tar.gz
    files:
      - LICENSE
      - CHANGELOG.md
      - configs/examples/**

  - id: grafocal-archive
    builds:
      - grafocal
    name_template: "grafocal-{{ .Version }}-{{ .Os }}-{{ .Arch }}{{ if .Arm }}v{{ .Arm }}{{ end }}"
    format: tar.gz

nfpms:
  - id: grafophone-deb
    package_name: grafophone
    builds:
      - grafophone
    vendor: Grafophone
    homepage: https://github.com/leftathome/grafophone
    maintainer: Grafophone Contributors
    description: Sensor-to-synthesizer sequencer
    license: MIT
    formats:
      - deb
    contents:
      - src: configs/default.yml
        dst: /etc/grafophone/config.yml
        type: config
      - src: scripts/grafophone.service
        dst: /lib/systemd/system/grafophone.service
    scripts:
      postinstall: scripts/postinstall.sh

dockers:
  - image_templates:
      - "ghcr.io/leftathome/grafophone:{{ .Version }}-amd64"
    build_flag_templates:
      - "--platform=linux/amd64"
    use: buildx
    dockerfile: Dockerfile
    ids:
      - grafophone

  - image_templates:
      - "ghcr.io/leftathome/grafophone:{{ .Version }}-arm64"
    build_flag_templates:
      - "--platform=linux/arm64"
    use: buildx
    dockerfile: Dockerfile
    ids:
      - grafophone

docker_manifests:
  - name_template: "ghcr.io/leftathome/grafophone:{{ .Version }}"
    image_templates:
      - "ghcr.io/leftathome/grafophone:{{ .Version }}-amd64"
      - "ghcr.io/leftathome/grafophone:{{ .Version }}-arm64"

  - name_template: "ghcr.io/leftathome/grafophone:latest"
    image_templates:
      - "ghcr.io/leftathome/grafophone:{{ .Version }}-amd64"
      - "ghcr.io/leftathome/grafophone:{{ .Version }}-arm64"

checksum:
  name_template: "SHA256SUMS"

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^ci:"
      - "^chore:"
      - "^refactor:"
      - "^test:"
  groups:
    - title: Features
      regexp: "^feat:"
    - title: Bug Fixes
      regexp: "^fix:"
    - title: Performance
      regexp: "^perf:"
    - title: Others
      order: 999
```

**Dockerfile:**

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
COPY grafophone /usr/local/bin/grafophone
USER nonroot:nonroot
EXPOSE 4317 4318
ENTRYPOINT ["grafophone"]
CMD ["--config", "/etc/grafophone/config.yml"]
```

- Uses distroless base image (no shell, no package manager — minimal attack surface).
- Runs as non-root user `nonroot` (UID 65534).
- Ports 4317/4318 are for optional OTEL receiver; not exposed unless configured.
- Config file is mounted at runtime via Docker volume or bind mount.

**systemd unit file (`scripts/grafophone.service`):**

```ini
[Unit]
Description=Grafophone sensor-to-synthesizer sequencer
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=grafophone
Group=grafophone
ExecStart=/usr/bin/grafophone --config /etc/grafophone/config.yml
EnvironmentFile=-/etc/grafophone/env
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**postinstall script (`scripts/postinstall.sh`):**

```bash
#!/bin/sh
# Create service user (no login shell, no home directory)
id -u grafophone >/dev/null 2>&1 || useradd -r -s /usr/sbin/nologin grafophone
# Set config permissions
chmod 0640 /etc/grafophone/config.yml
chown root:grafophone /etc/grafophone/config.yml
# Enable and start service
systemctl daemon-reload
systemctl enable grafophone
```

**Release workflow (`.github/workflows/release.yml`):**

```yaml
name: Release
on:
  push:
    tags: ["v*"]

permissions:
  contents: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: goreleaser/goreleaser-action@v6
        with:
          version: "~> v2"
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### CI Workflow (`.github/workflows/ci.yml`)

```yaml
name: CI
on:
  push:
    branches: [main, "feature/**", "claude/**"]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - uses: golangci/golangci-lint-action@v6
        with:
          version: v1.62
      - name: govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...
      - name: Markdown link check
        uses: gaurav-nelson/github-action-markdown-link-check@v1
        with:
          use-quiet-mode: 'yes'
          config-file: '.markdown-link-check.json'

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - name: Unit tests
        run: go test -race -coverprofile=coverage.out -covermode=atomic ./...
      - name: Check coverage
        run: |
          total=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
          echo "Total coverage: ${total}%"
          if (( $(echo "$total < 80" | bc -l) )); then
            echo "Coverage ${total}% is below 80% threshold"
            exit 1
          fi
      - name: Fuzz tests
        run: |
          go test -fuzz=FuzzMapper -fuzztime=30s ./internal/mapper/...
          go test -fuzz=FuzzQuantizer -fuzztime=30s ./internal/mapper/...
          go test -fuzz=FuzzNormalize -fuzztime=30s ./internal/input/...
          go test -fuzz=FuzzConfigParse -fuzztime=30s ./internal/config/...
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage.out

  build:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      matrix:
        include:
          - goos: linux
            goarch: amd64
          - goos: linux
            goarch: arm64
          - goos: linux
            goarch: arm
            goarm: "6"
          - goos: darwin
            goarch: arm64
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - name: Build
        env:
          CGO_ENABLED: "0"
          GOOS: ${{ matrix.goos }}
          GOARCH: ${{ matrix.goarch }}
          GOARM: ${{ matrix.goarm }}
        run: |
          go build -ldflags "-s -w \
            -X main.version=${GITHUB_REF_NAME:-dev} \
            -X main.commit=$(git rev-parse --short HEAD) \
            -X main.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
            -o grafophone-${{ matrix.goos }}-${{ matrix.goarch }} \
            ./cmd/grafophone/
      - uses: actions/upload-artifact@v4
        with:
          name: grafophone-${{ matrix.goos }}-${{ matrix.goarch }}
          path: grafophone-${{ matrix.goos }}-${{ matrix.goarch }}

  integration:
    runs-on: ubuntu-latest
    needs: [test, build]
    if: github.ref == 'refs/heads/main' || github.event_name == 'pull_request'
    # NOTE: All credentials below are ephemeral CI-only test fixtures.
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: grafophone_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 2s
          --health-timeout 5s
          --health-retries 10
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 2s
          --health-timeout 5s
          --health-retries 10
      prometheus:
        image: prom/prometheus:latest
        ports: ["9090:9090"]
        options: >-
          --health-cmd "wget -qO- http://localhost:9090/-/healthy || exit 1"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
      mosquitto:
        image: eclipse-mosquitto:2
        ports: ["1883:1883"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - name: Integration tests
        env:
          GRAFOPHONE_TEST_POSTGRES_URL: "postgres://test:test@localhost:5432/grafophone_test?sslmode=prefer"
          GRAFOPHONE_TEST_REDIS_URL: "redis://localhost:6379"
          GRAFOPHONE_TEST_PROMETHEUS_URL: "http://localhost:9090"
          GRAFOPHONE_TEST_MQTT_URL: "tcp://localhost:1883"
        run: go test -race -tags=integration -count=1 -timeout=5m ./...

  acceptance:
    runs-on: ubuntu-latest
    needs: integration
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - name: Acceptance tests
        run: go test -race -tags=acceptance -count=1 -timeout=10m ./test/acceptance/...
```

### Commit Message Convention

Follow [Conventional Commits](https://www.conventionalcommits.org/) for changelog generation:

| Prefix | Meaning | Changelog Section |
|---|---|---|
| `feat:` | New feature | Features |
| `fix:` | Bug fix | Bug Fixes |
| `perf:` | Performance improvement | Performance |
| `refactor:` | Code restructuring | (excluded from changelog) |
| `test:` | Test additions/changes | (excluded from changelog) |
| `docs:` | Documentation | (excluded from changelog) |
| `ci:` | CI/CD changes | (excluded from changelog) |
| `chore:` | Maintenance | (excluded from changelog) |
| `BREAKING CHANGE:` | In body/footer | Breaking Changes (bumps MAJOR) |
