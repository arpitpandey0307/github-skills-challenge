# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!

## Task Results

### Task 1: Continuous Integration

**Result:** Python test and coverage workflows were added under
`.github/workflows/`. The tests run successfully on `main`; the initial coverage
report showed 58% coverage and identified opportunities for additional tests.

### Task 2: Analyse Logs and Metrics

**Result:** The operational data was reviewed and documented below. Metrics are
response time, CPU, and memory; log information is the log level and message; normal
records occur before `10:05` and after `10:06`, while the timeout records are unusual.

### Task 3: Identify Anomalies

**Result:** The detector processed 10 records and identified anomalies at `10:05` and
`10:06`, including the corresponding abnormal metric values and timeout log messages.

### Task 4: Verify the AIOps Event Flow

**Result:** The producer published the two detected anomaly events to the shared
`service-events` topic, the consumer received both events, and the downstream AIOps
pipeline returned them in `events_consumed`.

### Task 5: Investigate and Correct the Workflow

**Result:** The anomaly detector was the affected component. It checked only for
`WARNING` logs, so the dataset's `ERROR` timeout records were detected by metrics but
their concerning log information was omitted. The detector now recognizes both
`ERROR` and `WARNING` log levels as `Error log detected`. The pipeline was rerun and
reported 10 records processed, 2 anomalies detected, 2 events consumed, and the log
reason on both anomaly events. The shared-topic correction from Task 4 remains in
place, so the complete event flow continues to work.

### Task 6: Execute the End-to-End Pipeline

**Result:** The complete workflow was executed successfully. Ten operational records
were processed, two anomalous observations were detected, two `ANOMALY` events were
generated and published to the `service-events` topic, and both events were consumed
and returned by the downstream AIOps pipeline.

## AIOps Assessment

This repository monitors a synthetic `payment-service`. The service processes payment
requests, and its operational data records response time, CPU utilization, memory
utilization, log level, and a message at one-minute intervals.

### Repository Components

- **Operational data:** [data/service_data.json](data/service_data.json) contains the
	recorded service observations.
- **Metrics and logs:** `response_time_ms`, `cpu_percent`, and `memory_percent` are
	numeric metrics. `log_level` and `message` provide log information, while `timestamp`
	identifies when each observation occurred.
- **Anomaly detection:** [src/anomaly_detector.py](src/anomaly_detector.py) compares
	response time, CPU, and memory values with configured thresholds and creates anomaly
	records when a threshold is exceeded.
- **Event production:** [src/event_producer.py](src/event_producer.py) publishes
	detected anomaly events.
- **Event topics:** [src/event_topic.py](src/event_topic.py) provides the in-memory
	topics that store published events.
- **Event consumption:** [src/event_consumer.py](src/event_consumer.py) reads events
	from a topic.
- **Final AIOps processing:** [src/aiops_pipeline.py](src/aiops_pipeline.py) loads the
	operational data, runs anomaly detection, publishes detected events, and returns the
	processing results.

### Operational Problem

The operational problem is identifying degraded or failing payment requests quickly.
The data includes normal successful requests as well as timeout events with elevated
response time and, in one observation, high CPU and memory utilization. AIOps is used
in this assessment to combine service metrics and log information, detect unusual
behavior automatically, and produce events that can be consumed by downstream
monitoring or response processes.

### Log and Metric Observations

- **Metrics:** `response_time_ms`, `cpu_percent`, and `memory_percent` are numeric
	measurements of request latency and resource utilization. `service` identifies the
	service producing each record, but is not itself a metric.
- **Log information:** `log_level` and `message` describe the outcome or condition
	reported by the service. The normal records use `INFO` with a successful-processing
	message; the unusual records use `ERROR` with timeout messages.
- **Timestamps:** `timestamp` values use the `2026-09-20T10:MM:00` ISO-style format.
	They identify when each observation occurred, advance at one-minute intervals from
	`10:00` through `10:09`, and do not include a timezone or UTC offset.
- **Normal behaviour:** Records from `10:00` through `10:04` and from `10:07` through
	`10:09` show stable operation. Response times range from `120` to `150 ms`, CPU from
	`42%` to `50%`, and memory from `51%` to `57%`; all have `INFO` logs indicating
	successful payment processing.
- **Unusual behaviour:** The `10:05` observation has a `610 ms` response time and an
	`ERROR` message reporting a payment-service timeout. At `10:06`, response time is
	`640 ms`, CPU is `94%`, and memory is `91%`, with an `ERROR` message reporting a
	database connection timeout. These exceed the detector thresholds of `500 ms` for
	response time and `80%` for CPU and memory.

### Detection Report

Running [src/aiops_pipeline.py](src/aiops_pipeline.py) with
[data/service_data.json](data/service_data.json) processes all 10 records and produces
two anomaly events:

| Timestamp | Log information | Metric values | Detection reasons |
| --- | --- | --- | --- |
| `2026-09-20T10:05:00` | `ERROR`: Payment service timeout | `610 ms`, CPU `75%`, memory `70%` | High response time |
| `2026-09-20T10:06:00` | `ERROR`: Database connection timeout | `640 ms`, CPU `94%`, memory `91%` | High response time; high CPU utilization; high memory utilization |

The remaining eight observations are treated as normal, and no normal event is
incorrectly flagged by the configured metric or log-level checks. After the Task 5
correction, both timeout records include `Error log detected` in their reported
reasons.

The pipeline publishes the detected events to the `service-events` topic, and the
consumer reads those same events from that topic. The verified run therefore reports
two events consumed, preserving the anomaly timestamps, source records, and detection
reasons, including the concerning `ERROR` log reason after the Task 5 correction. A
possible improvement is to include the original log level and message directly in the
detected event reasons or output.

### Event Flow Verification

The complete event-processing flow is:

1. **Anomaly detector:** identifies an abnormal observation and creates an `ANOMALY`
	event containing the timestamp, service, reasons, and source record.
2. **Producer:** receives that event and publishes it to the `service-events` topic.
3. **Topic:** stores the event in the in-memory message list.
4. **Consumer:** reads the event from the same topic.
5. **AIOps pipeline:** returns the consumed events as `events_consumed` for downstream
	processing and reporting.

Running the pipeline against the operational data processed 10 records, detected 2
anomalies, and consumed 2 events. The consumed events correspond to `10:05` and
`10:06`, confirming that both abnormal observations reached the downstream result.

### End-to-End Execution Record

The verified Task 6 flow was:

`data/service_data.json` → `AnomalyDetector` → `ANOMALY` event → `EventProducer` →
`service-events` topic → `EventConsumer` → `events_consumed` AIOps output.

Execution output:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2
2026-09-20T10:05:00 ANOMALY payment-service
	High response time, Error log detected
2026-09-20T10:06:00 ANOMALY payment-service
	High response time, High CPU utilization, High memory utilization, Error log detected
```

The final output represents the payment-service issues at `10:05` and `10:06`,
including the elevated response time, resource utilization, and concerning error logs.

### Reproduce the Demonstration

From the repository root, use Python 3.13 or a compatible Python 3 version:

1. Create and activate a virtual environment:

	```bash
	python -m venv .venv/aiops
	source .venv/aiops/bin/activate
	```

2. Install the project and test dependencies:

	```bash
	python -m pip install --upgrade pip
	python -m pip install -r requirements.txt
	python -m pip install pytest==8.4.1 coverage pytest-cov
	```

3. Run the complete test suite:

	```bash
	python -m pytest -q
	```

4. Execute the end-to-end AIOps pipeline. The `PYTHONPATH` setting is required by
	the existing sibling-module imports in `src/aiops_pipeline.py`:

	```bash
	PYTHONPATH=src python src/aiops_pipeline.py
	```

5. Confirm the expected result: 10 records processed, 2 anomalies detected, and 2
	events consumed. The anomaly timestamps should be `10:05` and `10:06`, with the
	response-time, CPU, memory, and `Error log detected` reasons shown above.

The corrections made during the assessment are preserved in the existing architecture:
the producer and consumer share the `service-events` topic, and the detector treats
both `ERROR` and `WARNING` as concerning log levels.


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

