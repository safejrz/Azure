# 🏗️ Architecture Diagrams

## Phase 1 — Foundation (Simulation)

The full Azure pipeline is built and validated using a Python telemetry simulator.
No physical devices required. All data flows and transformations are established here.

```mermaid
flowchart TD
    subgraph SIM["🖥️ Local Machine (Simulator)"]
        PY["Python Telemetry\nSimulator"]
    end

    subgraph AZURE["☁️ Azure"]
        IH["Azure IoT Hub\n(Device ingestion)"]
        EH["Azure Event Hubs\n(Streaming bus)"]
        SA["Azure Stream Analytics\n(Real-time processing)"]
        DL["Azure Data Lake Gen2\n(Raw + processed storage)"]
        KV["Azure Key Vault\n(Secrets)"]
        MON["Azure Monitor\n(Logs & Metrics)"]
    end

    PY -->|"AMQP / HTTPS\n(IoT Hub SDK)"| IH
    IH -->|"Built-in routing"| EH
    EH --> SA
    SA -->|"Processed stream"| DL
    SA -->|"Aggregated windows"| DL
    IH -.->|"Diagnostics"| MON
    SA -.->|"Job metrics"| MON
    KV -.->|"Secrets"| IH
    KV -.->|"Secrets"| SA
```

---

## Phase 2 — Real-Time Alerting Pipeline

Stream Analytics detects anomalies (e.g. power spike, temperature threshold).
Alerts are routed through Service Bus → Logic Apps → notifications.

```mermaid
flowchart TD
    subgraph SIM["🖥️ Simulator / Devices"]
        PY["Telemetry Source\n(Simulator or real)"]
    end

    subgraph AZURE["☁️ Azure"]
        IH["Azure IoT Hub"]
        EH["Azure Event Hubs"]
        SA["Azure Stream Analytics"]
        DL["Azure Data Lake Gen2"]
        CDB["Azure Cosmos DB\n(Latest device state)"]
        SB["Azure Service Bus\n(Alert queue)"]
        LA["Azure Logic Apps\n(Alert workflows)"]
        AI["Application Insights"]
    end

    subgraph NOTIFY["📣 Notifications"]
        EM["Email"]
        MS["Teams / Slack"]
    end

    PY --> IH --> EH --> SA
    SA -->|"All events"| DL
    SA -->|"Latest state"| CDB
    SA -->|"Anomaly detected"| SB
    SB --> LA
    LA --> EM
    LA --> MS
    SA -.-> AI
```

---

## Phase 3 — Real Device Integration (Shelly via MQTT)

Physical Shelly devices are connected via a local MQTT broker (Mosquitto)
that bridges messages into Azure IoT Hub. The Azure pipeline is unchanged.

```mermaid
flowchart TD
    subgraph HOME["🏠 Home Network"]
        SD["Shelly Devices\n(Plug S, 1PM, H&T)"]
        MQ["Mosquitto MQTT Broker\n(Local Raspberry Pi / PC)"]
        GW["IoT Edge Gateway\n(optional, for protocol translation)"]
    end

    subgraph AZURE["☁️ Azure"]
        IH["Azure IoT Hub"]
        EH["Azure Event Hubs"]
        SA["Azure Stream Analytics"]
        DL["Azure Data Lake Gen2"]
        CDB["Azure Cosmos DB"]
        SB["Azure Service Bus"]
        LA["Azure Logic Apps"]
    end

    SD -->|"MQTT publish\n(shellies/+/relay/0)"| MQ
    MQ -->|"MQTT bridge\n(TLS 8883)"| IH
    GW -.->|"Alternative path"| IH
    IH --> EH --> SA
    SA --> DL
    SA --> CDB
    SA -->|"Anomaly alert"| SB --> LA
```

---

## Phase 4 — Full Production Stack

A live web dashboard and scalable microservices backend complete the stack.
API Management provides a secure gateway to all backend services.

```mermaid
flowchart TD
    subgraph HOME["🏠 Home Network"]
        SD["Shelly Devices"]
        MQ["Mosquitto Bridge"]
    end

    subgraph AZURE["☁️ Azure"]
        direction TB
        IH["IoT Hub"]
        EH["Event Hubs"]
        SA["Stream Analytics"]
        DL["Data Lake Gen2"]
        CDB["Cosmos DB"]
        SB["Service Bus"]
        LA["Logic Apps"]
        CA["Container Apps\n(API Backend)"]
        APIM["API Management"]
        SWA["Static Web App\n(Dashboard)"]
        MON["Azure Monitor\n+ App Insights"]
    end

    subgraph USER["👤 User"]
        DASH["Browser Dashboard"]
    end

    SD --> MQ --> IH --> EH --> SA
    SA --> DL & CDB
    SA -->|"Alert"| SB --> LA
    CDB --> CA
    CA --> APIM --> SWA --> DASH
    MON -.->|"Observability"| IH & SA & CA
```

---

## Data Flow Summary

```mermaid
sequenceDiagram
    participant Device as Shelly Device
    participant MQTT as Mosquitto Broker
    participant Hub as IoT Hub
    participant EH as Event Hubs
    participant SA as Stream Analytics
    participant Store as Storage (DL / Cosmos)
    participant Alert as Alert System

    Device->>MQTT: Publish telemetry (every 10s)
    MQTT->>Hub: Bridge MQTT → Azure MQTT
    Hub->>EH: Route telemetry event
    EH->>SA: Stream event
    SA->>Store: Write processed record
    SA-->>Alert: Anomaly detected → trigger alert
    Alert-->>Device: (future: send command back)
```
