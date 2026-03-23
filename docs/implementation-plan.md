# 📋 Phased Implementation Plan

## Overview

| Phase | Name | Goal | Duration (est.) |
|---|---|---|---|
| 1 | Foundation | Azure core infra + telemetry simulator | 1–2 weeks |
| 2 | Real-Time Pipeline | Stream processing + anomaly detection + alerts | 1–2 weeks |
| 3 | Device Integration | Connect real Shelly devices via MQTT bridge | 1 week |
| 4 | Dashboard & API | Live web dashboard + REST backend | 1–2 weeks |
| 5 | Production Hardening | Scaling, observability, cost controls | 1 week |

---

## Phase 1 — Foundation

**Goal**: Provision core Azure resources and validate the ingestion pipeline with simulated data.

### Tasks

- [ ] Create Azure subscription (or use existing free trial)
- [ ] Create a **Resource Group**: `rg-smarthome-iot`
- [ ] Provision **Azure Key Vault**: store all connection strings and secrets here
- [ ] Provision **Azure IoT Hub** (Free tier: up to 8,000 msgs/day)
  - Register simulated devices (e.g. `sim-plug-1`, `sim-sensor-1`)
- [ ] Provision **Azure Event Hubs** namespace + hub (`telemetry-hub`)
  - Configure IoT Hub built-in routing → Event Hub
- [ ] Provision **Azure Data Lake Storage Gen2**
  - Create containers: `raw/`, `processed/`
- [ ] Provision **Azure Stream Analytics** job
  - Input: Event Hub
  - Output: Data Lake (raw passthrough to start)
  - Query: simple SELECT * passthrough first
- [ ] Write **Python Telemetry Simulator**
  - Simulate 5–10 devices posting every 10 seconds
  - Payload schema: `{ device_id, timestamp, power_w, voltage_v, temperature_c, humidity_pct, state }`
  - Use Azure IoT Hub Device SDK (`azure-iot-device`)
- [ ] Enable **Azure Monitor** diagnostic logs for IoT Hub and Stream Analytics
- [ ] Validate: messages flow from simulator → IoT Hub → Event Hub → Stream Analytics → Data Lake

### Deliverables
- Terraform or ARM/Bicep templates for all Phase 1 resources
- Python simulator script
- Stream Analytics passthrough query
- Verified data in Data Lake storage

### Tools & SDKs
- Azure CLI or Portal
- Python + `azure-iot-device` SDK
- Azure Stream Analytics query language (SQL-like)
- Bicep / Terraform (Infrastructure as Code)

---

## Phase 2 — Real-Time Pipeline

**Goal**: Add meaningful stream processing: windowed aggregations, anomaly detection, and alert routing.

### Tasks

- [ ] Extend Stream Analytics query:
  - 5-minute tumbling window: avg/min/max per device
  - Spike detection: if power_w > threshold → flag as anomaly
  - Output to Data Lake (processed/) AND Cosmos DB (latest state)
- [ ] Provision **Azure Cosmos DB** (Serverless)
  - Container: `device-state` (partition key: `device_id`)
- [ ] Provision **Azure Service Bus** namespace + queue (`alert-queue`)
- [ ] Connect Stream Analytics anomaly output → Service Bus queue
- [ ] Provision **Azure Logic Apps** workflow:
  - Trigger: new message in Service Bus queue
  - Action: send email or Teams notification with alert details
- [ ] Add **Application Insights** to any processing components
- [ ] Write tests: inject known anomaly events from simulator; verify alert received

### Deliverables
- Extended Stream Analytics SAQL queries
- Cosmos DB schema and index policy
- Logic Apps alert workflow
- Alert triggered + notification received (screenshot/log)

---

## Phase 3 — Real Device Integration

**Goal**: Replace the simulator with real Shelly device telemetry via MQTT bridge.

### Prerequisite
- A local machine (laptop, Raspberry Pi, or home server) always-on on the home network
- Shelly devices on the same WiFi network

### Tasks

- [ ] Install **Mosquitto MQTT broker** on local machine
  - `sudo apt install mosquitto mosquitto-clients`
- [ ] Configure Shelly devices to publish to local Mosquitto
  - Shelly → Settings → MQTT → Enable, set broker IP + port 1883
  - Verify: `mosquitto_sub -t "shellies/#" -v`
- [ ] Switch IoT Hub to use **X.509 certificate authentication** (recommended for production)
  - Or use SAS token per-device for simplicity
- [ ] Configure **Mosquitto → Azure IoT Hub bridge**
  - Edit `/etc/mosquitto/conf.d/azure-bridge.conf`
  - Bridge: local topic `shellies/#` → Azure IoT Hub MQTT endpoint (port 8883, TLS)
  - Map Shelly MQTT topics to IoT Hub device message format
- [ ] Write a **topic mapper / normalizer** (small Python service or Node.js)
  - Translates Shelly MQTT payloads to the unified pipeline schema
  - Handles: `shellies/shellyplug-s-{mac}/relay/0/power` → `{ device_id, power_w, ... }`
- [ ] Validate: real device telemetry appears in Data Lake and Cosmos DB
- [ ] Decommission the simulator (or keep as fallback)

### Shelly MQTT Topic Reference

| Shelly Topic | Data | Mapped Field |
|---|---|---|
| `shellies/{id}/relay/0/power` | Float (Watts) | `power_w` |
| `shellies/{id}/relay/0/energy` | Int (Wh) | `energy_wh` |
| `shellies/{id}/relay/0` | `on`/`off` | `state` |
| `shellies/{id}/temperature` | Float (°C) | `temperature_c` |
| `shellies/{id}/humidity` | Float (%) | `humidity_pct` |

### Deliverables
- Mosquitto bridge config file
- Topic normalizer script
- Real device data visible in Azure pipeline (screenshot)

---

## Phase 4 — Dashboard & API

**Goal**: Build a live dashboard showing device states, power consumption, and alerts.

### Tasks

- [ ] Provision **Azure Container Apps** environment
  - Deploy a REST API backend (Python FastAPI or Node.js)
  - Reads from Cosmos DB (latest device state)
  - Reads from Data Lake (historical queries)
- [ ] Provision **Azure API Management** (Developer tier)
  - Expose backend API securely
  - Add rate limiting and API key auth
- [ ] Provision **Azure Static Web Apps** (Free tier)
  - Build a dashboard (React or vanilla HTML/JS)
  - Pages:
    - Device list with live state (power, temp, on/off)
    - Power consumption chart (24h, 7d)
    - Alert log
- [ ] Set up CI/CD with **GitHub Actions**
  - Auto-deploy Static Web App on push to `main`
  - Auto-deploy Container App on push

### Deliverables
- Live dashboard URL
- GitHub Actions workflow files
- API documentation (OpenAPI spec)

---

## Phase 5 — Production Hardening

**Goal**: Make the system resilient, observable, and cost-controlled.

### Tasks

- [ ] Configure **auto-scaling** for Container Apps (scale on HTTP requests or queue depth)
- [ ] Set **Azure Monitor alerts**:
  - IoT Hub message quota > 80%
  - Stream Analytics input events drop to 0 (dead pipeline)
  - Cosmos DB RU consumption spike
- [ ] Configure **Azure Cost Management** budgets and alerts
- [ ] Implement **Data Lake lifecycle policy** (move old data to cool/archive tier)
- [ ] Add **Azure Defender for IoT** (optional, for device security insights)
- [ ] Document runbooks: how to restart pipeline, add new device, rotate secrets

### Deliverables
- Auto-scaling policy configs
- Monitor alert rules
- Cost budget alert (e.g. alert if > $10/month)
- Final architecture diagram (as-built)

---

## Estimated Monthly Cost (Low-Usage Home Lab)

| Service | Tier | Est. Cost/month |
|---|---|---|
| IoT Hub | Free (8K msgs/day) | $0 |
| Event Hubs | Basic (1 TU) | ~$9 |
| Stream Analytics | 1 Standard SU | ~$80 |
| Data Lake Storage | LRS, ~10GB | ~$0.20 |
| Cosmos DB | Serverless, light use | ~$1–3 |
| Service Bus | Standard | ~$0.05 |
| Logic Apps | Consumption | ~$0.10 |
| Container Apps | Consumption | ~$0–5 |
| Static Web Apps | Free | $0 |
| Key Vault | Standard | ~$0.03 |
| **Total** | | **~$95–100/month** |

> ⚠️ **Stream Analytics is the largest cost driver.** For learning, you can pause the job when not in use, which stops billing for Streaming Units.

> 💡 **Azure free trial** gives $200 in credits for 30 days — enough to run the full stack through Phase 2 at no cost.
