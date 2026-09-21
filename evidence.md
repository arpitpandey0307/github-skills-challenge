# AIOps Evidence

This document records the terminal evidence for the completed AIOps workflow.
The commands are included so each result can be reproduced locally.

## 1. Operational Data, Logs, and Metrics

Command:

```bash
python -m json.tool data/service_data.json
```

The output showed 10 `payment-service` records containing:

- `timestamp`
- `response_time_ms`
- `cpu_percent`
- `memory_percent`
- `log_level`
- `message`

The records from `10:00` through `10:04` and `10:07` through `10:09` contain `INFO`
messages and stable metrics. The records at `10:05` and `10:06` contain `ERROR`
messages and elevated metrics.

## 2. Anomaly Detection Results

Command:

```bash
PYTHONPATH=src python - <<'PY'
import json
from anomaly_detector import AnomalyDetector

with open("data/service_data.json", encoding="utf-8") as file:
    records = json.load(file)

detector = AnomalyDetector()
for record in records:
    result = detector.detect(record)
    print(record["timestamp"], result or "NORMAL")
PY
```

Result summary:

```text
10:00 NORMAL
10:01 NORMAL
10:02 NORMAL
10:03 NORMAL
10:04 NORMAL
10:05 ANOMALY: High response time, Error log detected
10:06 ANOMALY: High response time, High CPU utilization, High memory utilization, Error log detected
10:07 NORMAL
10:08 NORMAL
10:09 NORMAL
```

## 3. Event Generation and Producer -> Topic -> Consumer Flow

Command:

```bash
PYTHONPATH=src python - <<'PY'
import json
from anomaly_detector import AnomalyDetector
from event_producer import EventProducer
from event_consumer import EventConsumer
from event_topic import EventTopic

with open("data/service_data.json", encoding="utf-8") as file:
    record = json.load(file)[5]

event = AnomalyDetector().detect(record)
print("ANOMALY EVENT:", event)
topic = EventTopic("service-events")
producer = EventProducer(topic)
print("PRODUCER PUBLISHED:", producer.publish(event))
print("TOPIC MESSAGES:", topic.get_messages())
consumer = EventConsumer(topic)
print("CONSUMER RECEIVED:", consumer.consume())
PY
```

Result:

```text
ANOMALY EVENT: created for 2026-09-20T10:05:00
PRODUCER PUBLISHED: True
TOPIC MESSAGES: contains 1 ANOMALY event
CONSUMER RECEIVED: contains the same 1 ANOMALY event
```

This verifies that the detector creates the event, the producer publishes it, the
`service-events` topic stores it, and the consumer receives it.

## 4. Final AIOps Output

Command:

```bash
PYTHONPATH=src python src/aiops_pipeline.py
```

Output:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2

Service: payment-service
Timestamp: 2026-09-20T10:05:00
Type: ANOMALY
Reasons: High response time, Error log detected

Service: payment-service
Timestamp: 2026-09-20T10:06:00
Type: ANOMALY
Reasons: High response time, High CPU utilization, High memory utilization, Error log detected
```

## 5. Validation and Test Execution

Command:

```bash
python -m pytest --cov=src --verbose
coverage report --fail-under=90
```

Result:

```text
15 passed
TOTAL coverage: 96%
Coverage failure threshold: passed (90%)
```

The validation confirms that the operational data is processed, anomalies are
identified, events are generated and transported, consumers receive the events, and
the final AIOps workflow completes successfully.
