# 🔍 Risk Analysis Service – Technical Documentation

## ✅ Overview

The **Risk Analysis Service** is the third core microservice in our smart logistics platform. While the Delay Prediction Service estimates whether a shipment will arrive late, the Risk Analysis Service answers a deeper question:

> *"How serious is this delay? Should the business be worried or take action?"*

It evaluates the **severity** and **impact** of predicted delays by analyzing external factors such as location risk, vendor reliability, and delay duration. It then assigns a risk level (LOW, MEDIUM, HIGH) and sends this data forward for alerts or dashboards.

---

## 🧠 Business Use Case

Imagine two shipments are delayed:

* One is delayed by 3 hours but is on a reliable route with no issues.
* Another is delayed by 6 hours, moving through a flood-prone zone with an unreliable courier.

The **Risk Analysis Service** helps distinguish between these two and flags the second one as **HIGH RISK**.

> 📌 This ensures logistics managers focus on the most urgent problems first.

---

## 🏗 Architecture & Flow

```plaintext
┌────────────────────────────┐
│ Kafka Topic: shipment.predicted │
└────────────┬───────────────┘
             ▼
┌────────────────────────────────────┐
│ Risk Analysis Service              │
│                                    │
│ 1. Read predicted delay            │
│ 2. Evaluate external risks         │
│ 3. Compute risk level              │
│ 4. Emit risk event                 │
└────────────┬───────────────────────┘
             ▼
┌────────────────────────────────────┐
│ Kafka Topic: shipment.risk        │
└────────────────────────────────────┘
```

---

## 🎯 Responsibilities

* Listen to Kafka topic `shipment.predicted`
* Analyze delay severity and combine with external data
* Determine risk level (LOW / MEDIUM / HIGH)
* Publish `ShipmentRiskAssessedEvent` to Kafka topic `shipment.risk`

---

## 📥 Input Kafka Message Format (`shipment.predicted`)

```json
{
  "shipmentId": "SHIP1234",
  "predictedETA": "2025-06-28T18:00:00Z",
  "delayInMinutes": 180,
  "isDelayed": true
}
```

---

## 📤 Output Kafka Message Format (`shipment.risk`)

```json
{
  "shipmentId": "SHIP1234",
  "riskLevel": "HIGH",
  "reason": "Delay > 3 hours + route risk score 0.8 + unreliable vendor"
}
```

---

## ⚙️ Technologies Used

| Component           | Description                          |
| ------------------- | ------------------------------------ |
| Spring Boot         | Core service framework               |
| Kafka Consumer      | Reads predicted shipment delays      |
| Kafka Producer      | Sends out risk-assessed events       |
| Redis/DB (Optional) | Stores risk scores per region/vendor |
| Java 21             | Language of implementation           |

---

## 🔧 Class Design

### 1. `ShipmentPredictedDelayEvent`

```java
public class ShipmentPredictedDelayEvent {
    private String shipmentId;
    private Instant predictedETA;
    private int delayInMinutes;
    private boolean isDelayed;
    // Getters and Setters
}
```

### 2. `ShipmentRiskAssessedEvent`

```java
public class ShipmentRiskAssessedEvent {
    private String shipmentId;
    private String riskLevel; // LOW, MEDIUM, HIGH
    private String reason;
    // Getters and Setters
}
```

### 3. `RiskAnalysisService`

```java
@Service
public class RiskAnalysisService {
    public ShipmentRiskAssessedEvent assessRisk(ShipmentPredictedDelayEvent event) {
        String riskLevel;
        String reason;

        int delay = event.getDelayInMinutes();
        double routeRiskScore = 0.8; // Placeholder, in reality: fetch from DB/Redis
        boolean unreliableVendor = true;

        if (delay > 180 && routeRiskScore > 0.7 && unreliableVendor) {
            riskLevel = "HIGH";
            reason = "Delay > 3 hours + route risk score 0.8 + unreliable vendor";
        } else if (delay > 60) {
            riskLevel = "MEDIUM";
            reason = "Moderate delay with average risk factors";
        } else {
            riskLevel = "LOW";
            reason = "Minor delay or low-risk route";
        }

        return new ShipmentRiskAssessedEvent(
            event.getShipmentId(),
            riskLevel,
            reason
        );
    }
}
```

### 4. `KafkaListeners`

```java
@Component
public class RiskEventConsumer {
    @Autowired
    private RiskAnalysisService analysisService;
    @Autowired
    private KafkaTemplate<String, ShipmentRiskAssessedEvent> kafkaTemplate;

    @KafkaListener(topics = "shipment.predicted", groupId = "risk-analyzer")
    public void consume(ShipmentPredictedDelayEvent event) {
        ShipmentRiskAssessedEvent risk = analysisService.assessRisk(event);
        kafkaTemplate.send("shipment.risk", risk);
    }
}
```

---

## 🧪 Future Enhancements

* Use historical data to compute real route/vendor scores
* Connect to external APIs (weather, traffic, customs alerts)
* Use Google Maps or other routing APIs to estimate ETA when historical data is missing
* Plug into ML-based risk scoring models
* Cache common region/vendor risks in Redis

---

## 🔐 Security & Monitoring

* Secured Kafka access via ACLs
* Spring Actuator endpoints for metrics
* Prometheus + Grafana integration planned

---

## 🚀 Summary

The Risk Analysis Service enhances logistics intelligence by turning delay data into actionable insights. It helps businesses focus on shipments that are not just delayed, but **risky**, making it a key component of proactive supply chain optimization.

---

Ready for development ✅
