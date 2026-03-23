# 🏠 Azure Smart Home IoT Pipeline

A real-world cloud learning project that ingests telemetry from home IoT devices (Shelly smart plugs, switches, and sensors) into a full Azure data pipeline — enabling real-time monitoring, anomaly detection, and alerting.

## 🎯 Learning Objectives

- Azure infrastructure provisioning and management
- High-throughput event ingestion and stream processing
- Real-time and batch data pipelines
- Auto-scaling cloud deployments
- Observability, alerting, and cost management

## 🧩 Physical Devices (Phase 2+)

| Device | Data Published |
|---|---|
| Shelly Plug S / Plus | Power (W), Voltage (V), Current (A), ON/OFF state |
| Shelly 1PM | Switch state, energy readings |
| Shelly H&T | Temperature (°C), Humidity (%) |
| Any Shelly | WiFi RSSI, uptime, firmware version |

All Shelly devices publish data over **MQTT** or **HTTP webhooks** — compatible with the Azure ingestion layer via a Mosquitto MQTT bridge.

---

## ☁️ Azure Services

| Service | Tier / SKU | Role | Phase |
|---|---|---|---|
| **Azure IoT Hub** | Free / S1 Standard | Device authentication & telemetry ingestion | 1 |
| **Azure Event Hubs** | Basic / Standard | High-throughput event streaming bus | 1 |
| **Azure Stream Analytics** | Standard Streaming Units | Real-time processing, windowed aggregations, anomaly detection | 1 |
| **Azure Data Lake Storage Gen2** | LRS | Raw and processed telemetry long-term storage | 1 |
| **Azure Cosmos DB** | Serverless | Low-latency storage for latest device state + alerts | 2 |
| **Azure Service Bus** | Standard | Reliable alert message queuing | 2 |
| **Azure Logic Apps** | Consumption | Alert workflows (email/Teams/SMS notifications) | 2 |
| **Azure Monitor** | Built-in | Infrastructure metrics, logs, diagnostic alerts | 1 |
| **Azure Application Insights** | Basic | APM for processing services | 2 |
| **Azure Key Vault** | Standard | Secrets management (IoT Hub keys, connection strings) | 1 |
| **Azure Container Apps** | Consumption | Scalable processing microservices / API backend | 3 |
| **Azure Static Web Apps** | Free | Live dashboard frontend | 3 |
| **Azure API Management** | Developer | REST API gateway for dashboard backend | 3 |

---

## 📁 Repository Structure

```
Azure/
├── README.md                    ← This file
├── architecture.md              ← Architecture diagrams (all phases)
├── implementation-plan.md       ← Phased implementation plan
├── phase-1-foundation/          ← IaC and simulator scripts
├── phase-2-pipeline/            ← Stream Analytics queries, alert config
├── phase-3-integration/         ← Mosquitto bridge config, device setup
└── phase-4-dashboard/           ← Frontend dashboard code
```

---

## 🔗 Related Documents

- [Architecture Diagrams](./architecture.md)
- [Phased Implementation Plan](./implementation-plan.md)
