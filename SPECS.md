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
DAC_OUT ──────┤   │       │  ├──── CV_OUT (0-10V)
              │   │       │  │
              │  ┌┴───────┴┐ │      R_out (470)
              └──┤-   OUT  ├─┘──────/\/\/──── JACK
                 │ OP-AMP  │
          ┌──────┤+        │
          │      └─────────┘
          │
         GND
          │
         R3 (10k)
          │
         GND
```

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
GPIO (3.3V) ────[10k]──── Gate ┐
                                │ BS170 / 2N7000
                          Source┘──── GND
                                │
                          Drain ┘────┬──── GATE_OUT (0V or ~10V)
                                     │
                                [10k pull-up]
                                     │
                                   +12V
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

### Supported Source Types

| Source Type | Protocol | Directionality | Typical Latency | Auth Methods |
|---|---|---|---|---|
| **PostgreSQL** | TCP (libpq) | Poll (SQL query) | 1-50 ms | User/pass, SSL, mTLS |
| **MySQL** | TCP | Poll (SQL query) | 1-50 ms | User/pass, SSL |
| **Redis** | TCP (RESP) | Poll (command) or Push (SUBSCRIBE, keyspace) | < 1 ms | Password, ACL, TLS |
| **Prometheus** | HTTP (PromQL) | Poll (query API) | 10-200 ms | Basic auth, Bearer token, mTLS |
| **InfluxDB** | HTTP (Flux/InfluxQL) | Poll (query API) | 10-200 ms | Token, user/pass |
| **Grafana** | HTTP (JSON API) | Poll (dashboard/panel query) | 50-500 ms | API key, Bearer token |
| **OpenTelemetry** | gRPC or HTTP (OTLP) | Push (receiver endpoint) | < 10 ms | mTLS, API key headers |
| **REST API** | HTTP/HTTPS | Poll (GET/POST) | 10-500 ms | API key, OAuth2, Bearer token |
| **GraphQL** | HTTP/HTTPS | Poll (query) | 10-500 ms | API key, Bearer token |
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
| Current loop | 5 mA through opto-isolator at receiver |
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
