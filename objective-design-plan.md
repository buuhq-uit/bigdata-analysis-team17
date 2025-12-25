# Smart Farm Big Data & Cloud Project (MQTT → Kafka → InfluxDB → Grafana)
*(Multi-farm, multi-zone, real-time monitoring + anomaly detection)*

# Project name:
AgriStream: Real-time IoT Data Streaming for Smart Agriculture

## 1 Goal & Scope

### 1.1 Problem statement
Smart farms (greenhouse / open-field) need real-time monitoring of environmental and device health metrics (air/soil + pump). Data must be stored as time series for dashboards, historical analysis, and alerting/anomaly detection.

### 1.2 Objectives
- Collect telemetry from sensors (ESP32) and operational events (pump start/stop/trip).
- Stream data reliably to the cloud using MQTT and Kafka.
- Store telemetry/events into a time-series database (InfluxDB).
- Visualize and monitor via Grafana dashboards.
- Implement rule-based anomaly detection and ML as future work.

### 1.3 Scope (multi-farm, multi-zone)
- Multiple farms: `farmId` (e.g., F1, F2...)
- Each farm has multiple zones: `zoneId` (e.g., Z1, Z2...)
- Each zone may have multiple devices: `deviceId` (esp32-01, esp32-02...)

Out of scope (optional future):
- Full device provisioning & certificate-based identity
- Predictive ML models, advanced optimization schedules

## 2 Architecture Overview

```text
+------------------------------ EDGE LAYER (FARMS/ZONES) ------------------------------+
|                                                                                      |
|  Farm F1                                                                             |
|   +-------------------+          +-------------------+                               |
|   | Zone Z1           |          | Zone Z2           |                               |
|   | ESP32-F1Z1-01     |          | ESP32-F1Z2-01     |                               |
|   | sensors:          |          | sensors:          |                               |
|   | - air temp/hum    |          | - air temp/hum    |                               |
|   | - soil moisture   |          | - soil moisture   |                               |
|   | - pump current    |          | - pump current    |                               |
|   | - pump temp       |          | - pump temp       |                               |
|   | - (water flow)    |          | - (water flow)    |                               |
|   +---------+---------+          +---------+---------+                               |
|             \                         /                                              |
|              \ MQTT publish          / MQTT publish                                   |
|               \ telemetry + events  /                                                 |
|                v                   v                                                  |
+--------------------------------------------------------------------------------------+
                                   |
                                   v
+--------------------------- CLOUD INGRESS LAYER ---------------------------+
| MQTT Broker (Mosquitto)                                                   |
| Topics:                                                                   |
|  - farm/{farmId}/zone/{zoneId}/telemetry                                  |
|  - farm/{farmId}/zone/{zoneId}/event                                      |
|  - farm/{farmId}/device/{deviceId}/status                                 |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
+---------------------------- STREAMING LAYER ------------------------------+
| MQTT -> Kafka Bridge / Connector                                           |
|  - subscribe MQTT topics                                                   |
|  - transform/validate                                                      |
|  - produce to Kafka                                                        |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
+------------------------------- KAFKA CLUSTER -----------------------------+
| Topics:                                                                    |
|  - iot.telemetry.raw   (key = deviceId, partitions)                        |
|  - iot.event.raw       (key = deviceId, partitions)                        |
+-------------------+------------------------+------------------------------+
                    |                        |
                    v                        v
        +--------------------+   +--------------------+
        | Telemetry Writer   |   | Event Writer       |
        | (Kafka Consumer)   |   | (Kafka Consumer)   |
        | -> Influx telemetry|   | -> Influx events   |
        +----------+---------+   +----------+---------+
                   \                 /
                    \               /
                     v             v
                 +-----------------------------------+
                 | InfluxDB (bucket: smartfarm)      |
                 | measurements: telemetry, events    |
                 +------------------+----------------+
                                    |
                                    v
+------------------------------- VISUALIZATION -----------------------------+
| Grafana Dashboards                                                         |
|  - Global overview (multi-farm)                                            |
|  - Farm detail (multi-zone)                                                |
|  - Device health                                                           |
|  - Alerts & events                                                         |
+----------------------------------------------------------------------------+

```

## 3. Domain Model (Multi-farm / Multi-zone)

### 3.1 Entities
- **Farm**: `farmId`, location, owner, crop type (optional)
- **Zone**: `zoneId`, greenhouse section / field block
- **Device**: `deviceId`, firmwareVersion, sensor set
- **Telemetry**: time-series numeric values
- **Event**: discrete operational occurrences

### 3.2 Recommended identifiers
- `farmId`: `F1`, `F2`, ... (or UUID)
- `zoneId`: `Z1`, `Z2`, ...
- `deviceId`: `esp32-F1Z1-01` (structured) or UUID

## 4. Sensors (Smart Farm) — Concrete & Practical Set

### 4.1 Environmental (air)
(s-1). **air_temperature** (°C)  
   - Purpose: detect heat stress; control fan/vent
(s-2). **air_humidity** (%)  
   - Purpose: prevent mold; control misting/vent

*(Optional) light_lux for advanced crop management.*

### 4.2 Soil & irrigation
(s-3). **soil_moisture** (%) or ADC raw  
   - Purpose: irrigation control (start/stop watering)
(s-4). *(Optional)* **soil_temperature** (°C)  
   - Purpose: root health insights

### 4.3 Pump / actuator health
(s-5). **pump_current** (A)  
   - Purpose: detect overload/blocked pump
(s-6). **pump_temperature** (°C)  
   - Purpose: overheating protection
(s-7). *(Optional but “pro”)* **water_flow** (L/min)  
   - Purpose: detect clogged pipe / empty tank (pump ON but flow = 0)

### 4.4 Suggested sampling frequency
- Telemetry: every **5–10 seconds** (stable demo, not too noisy)
- Critical device metrics (pump_current/flow): **1–2 seconds** when pump is ON
- Events: immediate (on change)

## 5. Events (Operational & Device Lifecycle)

### 5.1 Actuator events (pump, valve, fan, mister)
- `pump.start`
- `pump.stop`
- `pump.auto_stop` (system decides)
- `pump.overcurrent_trip` (threshold exceeded)
- `pump.overtemp_trip`
- `valve.open`, `valve.close`
- `fan.on`, `fan.off`
- `mister.on`, `mister.off`

### 5.2 Device lifecycle events
- `device.boot`
- `device.restart`
- `device.online`, `device.offline`
- `firmware.update`

### 5.3 Severity levels
- `info`, `warning`, `critical`

## 6. MQTT Topics & Payload Standard (Multi-farm, Multi-zone)

### 6.1 MQTT topic design
farm/{farmId}/zone/{zoneId}/telemetry
farm/{farmId}/zone/{zoneId}/event
farm/{farmId}/device/{deviceId}/status

### 6.2 Telemetry JSON payload (example)
```json
{
  "ts": 1735100000000,
  "farmId": "F1",
  "zoneId": "Z1",
  "deviceId": "esp32-F1Z1-01",
  "air_temperature": 31.2,
  "air_humidity": 62.5,
  "soil_moisture": 41.0,
  "pump_current": 1.35,
  "pump_temperature": 54.2,
  "water_flow": 3.8
}

### 6.3 Event JSON payload (example)
{
  "ts": 1735100005000,
  "farmId": "F1",
  "zoneId": "Z1",
  "deviceId": "esp32-F1Z1-01",
  "event_type": "pump.overcurrent_trip",
  "severity": "critical",
  "reason": "current=2.4A > threshold=2.0A"
}

## 7. Kafka Topics, Partitioning & Consumer Design
### 7.1 Kafka topics

iot.telemetry.raw

iot.event.raw

### 7.2 Partition key (important for ordering)

Use deviceId as the message key → ordering per device

Optionally: farmId-zoneId-deviceId if needed

### 7.3 Consumers

TelemetryWriter: Kafka → InfluxDB (measurement: telemetry)

EventWriter: Kafka → InfluxDB (measurement: events)

RuleEngine: Kafka → evaluate rules → write alerts/events back

## 8. InfluxDB Data Model (Bucket / Measurements / Tags / Fields)
### 8.1 Bucket

Bucket: smartfarm

### 8.2 Measurements

- telemetry

Tags (indexed): farmId, zoneId, deviceId

Fields (numeric):

  air_temperature, air_humidity, soil_moisture

  pump_current, pump_temperature, water_flow (optional)

Timestamp: from ts

- events

Tags: farmId, zoneId, deviceId, event_type, severity

Fields:

 value = 1 (to mark event occurrence)

 (optional) detail_code numeric, avoid heavy string analytics

### 8.3 Tag/Field rule of thumb

Tags: things you filter/group by often (farmId/zoneId/deviceId/event_type)

Fields: actual values changing over time (temperature/current)

### 8.4 Retention & downsampling (professional touch)

Raw telemetry retention: 7–30 days

Downsample to 1-minute avg: keep 6–12 months

Events: keep longer (audit trail)

## 9. Flux Query Examples (InfluxDB v2) — Ready for Grafana

Replace bucket name/time range as needed.

### 9.1 Air temperature for Farm F1, Zone Z1

```flux
from(bucket: "smartfarm")
  |> range(start: -6h)
  |> filter(fn: (r) => r._measurement == "telemetry")
  |> filter(fn: (r) => r.farmId == "F1" and r.zoneId == "Z1")
  |> filter(fn: (r) => r._field == "air_temperature")
  |> aggregateWindow(every: 1m, fn: mean, createEmpty: false)
```

### 9.2 Soil moisture per zone (compare zones inside a farm)

```flux
from(bucket: "smartfarm")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "telemetry")
  |> filter(fn: (r) => r.farmId == "F1")
  |> filter(fn: (r) => r._field == "soil_moisture")
  |> group(columns: ["zoneId"])
  |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)
```

### 9.3 Event timeline (pump trips)
```flux
from(bucket: "smartfarm")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "events")
  |> filter(fn: (r) => r.farmId == "F1")
  |> filter(fn: (r) => r.event_type == "pump.overcurrent_trip")
  |> keep(columns: ["_time","farmId","zoneId","deviceId","event_type","severity"]
  ```

## 10. Grafana Dashboards (Panels) — Multi-farm Professional Layout

### 10.1 Dashboard A: Global Overview (All farms)

Variables (dropdown)

- $farmId (All or pick one)

- $zoneId (All or pick one)

- $deviceId (optional)

Panels

- Stat: #Online devices (last seen)

- Time series: Avg air_temperature per farm

- Time series: Avg soil_moisture per farm

- Table: Latest critical events (last 24h)

### 10.2 Dashboard B: Farm Detail (per farm)

Panels

- Air temp/humidity by zone (multi-series)

- Soil moisture by zone

- Pump current by device (only pump devices)

- Heatmap or bar: watering cycles/day by zone

- Event annotations: pump trips on the charts

### 10.3 Dashboard C: Device Health (per device)

Panels

- pump_current + water_flow overlay

- pump_temperature

- Event timeline (start/stop/trip)

- Uptime / restart count (from events)

## 11 Rule-based Anomaly Detection (Concrete Rules for Smart Farm)
### 11.1 Dry soil (zone-level)

IF soil_moisture < 30% for 5 minutes → warning

IF soil_moisture < 20% for 2 minutes → critical

### 11.2 Pump overcurrent (device-level)

IF pump_current > 2.0A for 10 seconds → emit pump.overcurrent_trip + stop pump

### 11.3 Pump overtemperature

IF pump_temperature > 70°C for 10 seconds → emit pump.overtemp_trip + stop pump

### 11.4 Pump ON but no irrigation effect (smart rule)

IF pump is ON for 2 minutes

AND soil_moisture does not increase at least +2%
→ emit irrigation.ineffective (suspect: empty tank / clogged pipe / sensor fault)

### 11.5 Flow anomaly (if water_flow exists)

IF pump is ON AND water_flow == 0 for 10 seconds
→ emit pump.flow_zero_trip + stop pump

## 12. Demo Scenario (Easy + Impressive)

- Select Farm F1, Zone Z1 in Grafana

- Soil moisture drops below 30% → dashboard shows warning

- System emits event pump.start

- Soil moisture rises gradually → successful irrigation

- Simulate fault: increase pump_current > 2.0A

- System emits pump.overcurrent_trip, pump stops

- Grafana shows event in table + chart annotation

## 13. Future Work (ML & Optimization)

### 13.1 Add ML anomaly detection:

z-score, Isolation Forest for sensor anomalies

predict irrigation needs based on weather + history

### 13.1 Add control optimization:

scheduling irrigation windows

reduce energy consumption

### 13.1 Add device management:

provisioning, TLS certs, OTA firmware updates

## 14. Deliverables Checklist (for grading)

 - Architecture diagram (ASCII in report, plus optional slide)

 - MQTT topic & JSON payload standard

 - Kafka topics + partitioning rationale

 - InfluxDB schema (measurement/tags/fields)

 - Grafana dashboard screenshots

 - Rule-based anomaly detection demo

 - Future work section (ML)