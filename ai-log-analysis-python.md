# AI-Powered Log Analysis in Python: Catch Errors Before Users Do

Application logs are a goldmine of signal buried under noise. A typical production system generates thousands of log lines per minute, but most engineers only look at them after something breaks. This tutorial shows you how to build an AI-powered log analysis system in Python that detects anomalies, classifies errors, and surfaces issues before they become incidents.

We'll use local LLMs via Ollama — no API keys, no data leaving your infrastructure.

## What We're Building

A log analysis pipeline that:
1. Tails log files in real time
2. Classifies each log line by severity and type using an LLM
3. Detects anomaly patterns (error spikes, new error types, rate changes)
4. Generates a human-readable incident summary

## Prerequisites

- Python 3.10+
- Ollama installed with `llama3.2:3b` or similar small model
- A log file to analyze (we'll generate sample data)

```bash
ollama pull llama3.2:3b
pip install ollama watchdog rich
```

`requirements.txt`:
```
ollama>=0.1.7
watchdog>=4.0.0
rich>=13.0.0
```

## Step 1: Log Parser

```python
# log_parser.py
import re
from dataclasses import dataclass
from datetime import datetime
from enum import Enum


class Severity(str, Enum):
    DEBUG = "DEBUG"
    INFO = "INFO"
    WARNING = "WARNING"
    ERROR = "ERROR"
    CRITICAL = "CRITICAL"
    UNKNOWN = "UNKNOWN"


# Common log formats
PATTERNS = [
    # Standard: 2024-01-15 10:23:45,123 ERROR module: message
    re.compile(
        r'(?P<ts>\d{4}-\d{2}-\d{2}[\sT]\d{2}:\d{2}:\d{2}[.,]\d+)\s+'
        r'(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)\s+'
        r'(?P<source>[\w.]+):\s*(?P<message>.+)'
    ),
    # nginx/apache: [timestamp] [level] message
    re.compile(
        r'\[(?P<ts>[^\]]+)\]\s+\[(?P<level>debug|info|notice|warn|error|crit|alert|emerg)\]\s+(?P<message>.+)',
        re.IGNORECASE,
    ),
    # JSON logs
    re.compile(r'\{.*"level":\s*"(?P<level>[^"]+)".*"message":\s*"(?P<message>[^"]+)".*\}'),
]


@dataclass
class LogEntry:
    raw: str
    timestamp: datetime | None
    severity: Severity
    source: str
    message: str

    def is_error(self) -> bool:
        return self.severity in (Severity.ERROR, Severity.CRITICAL)


def parse_line(line: str) -> LogEntry:
    line = line.strip()
    for pattern in PATTERNS:
        m = pattern.search(line)
        if m:
            groups = m.groupdict()
            level_str = groups.get("level", "UNKNOWN").upper()
            # Normalize nginx levels
            level_map = {"NOTICE": "INFO", "WARN": "WARNING", "CRIT": "CRITICAL",
                        "ALERT": "CRITICAL", "EMERG": "CRITICAL", "DEBUG": "DEBUG"}
            level_str = level_map.get(level_str, level_str)
            try:
                severity = Severity(level_str)
            except ValueError:
                severity = Severity.UNKNOWN

            ts = None
            if ts_str := groups.get("ts"):
                for fmt in ("%Y-%m-%d %H:%M:%S,%f", "%Y-%m-%dT%H:%M:%S.%f", "%d/%b/%Y:%H:%M:%S"):
                    try:
                        ts = datetime.strptime(ts_str[:26], fmt)
                        break
                    except ValueError:
                        continue

            return LogEntry(
                raw=line,
                timestamp=ts,
                severity=severity,
                source=groups.get("source", "unknown"),
                message=groups.get("message", line),
            )

    return LogEntry(raw=line, timestamp=None, severity=Severity.UNKNOWN,
                   source="unknown", message=line)
```

## Step 2: AI Classifier

```python
# classifier.py
import ollama
import json

MODEL = "llama3.2:3b"

CLASSIFY_PROMPT = """Analyze this log message and respond with JSON only.

Log: {message}

Respond with this exact JSON structure:
{{
  "category": "auth|database|network|memory|disk|application|security|unknown",
  "is_actionable": true/false,
  "root_cause_hint": "brief hint or null",
  "priority": 1-5
}}

priority: 1=critical fix now, 5=ignore. is_actionable=true if a developer should look at this."""


def classify_error(message: str) -> dict:
    """Classify a log message using local LLM."""
    try:
        response = ollama.generate(
            model=MODEL,
            prompt=CLASSIFY_PROMPT.format(message=message[:500]),
            options={"temperature": 0.1, "num_predict": 150},
        )
        text = response["response"].strip()
        # Extract JSON from response
        start = text.find("{")
        end = text.rfind("}") + 1
        if start >= 0 and end > start:
            return json.loads(text[start:end])
    except Exception:
        pass
    return {"category": "unknown", "is_actionable": False,
            "root_cause_hint": None, "priority": 5}


SUMMARIZE_PROMPT = """You are a senior SRE. Analyze these recent errors and write a 3-sentence incident summary.
Focus on: what's broken, likely cause, recommended action.

Errors:
{errors}

Write the incident summary:"""


def summarize_errors(errors: list[str]) -> str:
    """Generate incident summary from a batch of error messages."""
    if not errors:
        return "No errors detected."
    error_text = "\n".join(f"- {e[:200]}" for e in errors[:20])
    try:
        response = ollama.generate(
            model=MODEL,
            prompt=SUMMARIZE_PROMPT.format(errors=error_text),
            options={"temperature": 0.3, "num_predict": 300},
        )
        return response["response"].strip()
    except Exception as e:
        return f"Summary unavailable: {e}"
```

## Step 3: Anomaly Detector

```python
# anomaly.py
from collections import deque, Counter
from datetime import datetime, timedelta
from dataclasses import dataclass, field


@dataclass
class AnomalyAlert:
    type: str
    message: str
    severity: str  # "warning" | "critical"
    timestamp: datetime = field(default_factory=datetime.now)


class AnomalyDetector:
    def __init__(self, window_seconds: int = 60, error_rate_threshold: int = 10):
        self.window = window_seconds
        self.threshold = error_rate_threshold
        self.recent_errors: deque = deque()
        self.seen_error_types: set = set()
        self.error_counts: Counter = Counter()

    def _prune_old(self):
        cutoff = datetime.now() - timedelta(seconds=self.window)
        while self.recent_errors and self.recent_errors[0][0] < cutoff:
            self.recent_errors.popleft()

    def record_error(self, message: str, category: str) -> list[AnomalyAlert]:
        alerts = []
        now = datetime.now()
        self.recent_errors.append((now, message, category))
        self._prune_old()

        # Alert 1: error rate spike
        if len(self.recent_errors) >= self.threshold:
            alerts.append(AnomalyAlert(
                type="error_spike",
                message=f"{len(self.recent_errors)} errors in last {self.window}s "
                       f"(threshold: {self.threshold})",
                severity="critical",
            ))

        # Alert 2: new error type never seen before
        error_sig = category + ":" + message[:50]
        if error_sig not in self.seen_error_types:
            self.seen_error_types.add(error_sig)
            if len(self.seen_error_types) > 5:  # Don't alert on first 5 (startup)
                alerts.append(AnomalyAlert(
                    type="new_error_type",
                    message=f"New error pattern detected: [{category}] {message[:80]}",
                    severity="warning",
                ))

        # Alert 3: specific category dominance
        self.error_counts[category] += 1
        total = sum(self.error_counts.values())
        if total > 10 and self.error_counts[category] / total > 0.8:
            alerts.append(AnomalyAlert(
                type="category_dominance",
                message=f"{category} errors are {self.error_counts[category]/total:.0%} "
                       f"of all errors",
                severity="warning",
            ))

        return alerts
```

## Step 4: Real-Time Pipeline

```python
# pipeline.py
import time
import sys
from pathlib import Path
from rich.console import Console
from rich.table import Table
from rich.live import Live
from log_parser import parse_line, Severity
from classifier import classify_error, summarize_errors
from anomaly import AnomalyDetector

console = Console()
detector = AnomalyDetector(window_seconds=60, error_rate_threshold=5)
error_buffer: list[str] = []


def process_line(line: str):
    entry = parse_line(line)
    if not entry.is_error():
        return

    classification = classify_error(entry.message)
    alerts = detector.record_error(entry.message, classification["category"])
    error_buffer.append(entry.message)

    # Print classified error
    priority_color = {1: "red", 2: "red", 3: "yellow", 4: "blue", 5: "dim"}
    color = priority_color.get(classification["priority"], "white")
    console.print(
        f"[{color}][P{classification['priority']}][/{color}] "
        f"[bold]{classification['category']}[/bold] — {entry.message[:100]}"
    )

    if classification["is_actionable"] and classification["root_cause_hint"]:
        console.print(f"         Hint: {classification['root_cause_hint']}", style="italic dim")

    for alert in alerts:
        style = "bold red" if alert.severity == "critical" else "bold yellow"
        console.print(f"\n[{style}]⚠ ANOMALY: {alert.type}[/{style}]")
        console.print(f"  {alert.message}")

        # Generate AI summary on critical anomalies
        if alert.severity == "critical" and len(error_buffer) >= 3:
            console.print("\n[bold cyan]AI Incident Summary:[/bold cyan]")
            summary = summarize_errors(error_buffer[-20:])
            console.print(summary)
            error_buffer.clear()


def tail_file(log_path: str):
    """Tail a log file and process new lines."""
    path = Path(log_path)
    console.print(f"[green]Monitoring:[/green] {path}")

    with open(path) as f:
        f.seek(0, 2)  # Seek to end
        while True:
            line = f.readline()
            if line:
                process_line(line)
            else:
                time.sleep(0.1)


def analyze_file(log_path: str):
    """Analyze an entire log file and produce a report."""
    path = Path(log_path)
    errors = []
    with open(path) as f:
        for line in f:
            entry = parse_line(line)
            if entry.is_error():
                errors.append(entry.message)

    console.print(f"\n[bold]Log Analysis Report: {path.name}[/bold]")
    console.print(f"Total errors found: {len(errors)}\n")

    if errors:
        console.print("[bold cyan]AI Incident Summary:[/bold cyan]")
        summary = summarize_errors(errors)
        console.print(summary)

    return errors


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python pipeline.py <logfile> [--tail]")
        sys.exit(1)

    log_file = sys.argv[1]
    if "--tail" in sys.argv:
        tail_file(log_file)
    else:
        analyze_file(log_file)
```

## Testing It

Generate sample logs:

```bash
python -c "
import random, time, datetime
errors = [
    'ERROR database: Connection timeout after 30s host=db-primary:5432',
    'ERROR auth: Invalid JWT token user_id=4821 ip=192.168.1.45',
    'ERROR memory: OOMKiller invoked pid=2847 process=worker',
    'ERROR network: DNS resolution failed for api.internal',
    'INFO request: GET /health 200 12ms',
    'INFO request: POST /api/users 201 45ms',
]
for i in range(50):
    ts = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S,%f')[:-3]
    line = random.choice(errors)
    print(f'{ts} {line}')
    time.sleep(0.05)
" > sample.log

# Analyze the file
python pipeline.py sample.log

# Or tail live
python pipeline.py sample.log --tail
```

## What the AI Output Looks Like

```
[P1] database — Connection timeout after 30s host=db-primary:5432
         Hint: Database primary may be overloaded or unreachable

⚠ ANOMALY: error_spike
  8 errors in last 60s (threshold: 5)

AI Incident Summary:
The system is experiencing a database connectivity failure with connection timeouts
to db-primary:5432, compounded by authentication failures suggesting downstream
services cannot reach the auth service. Immediate action: check database primary
health, verify network path to db-primary, and inspect load balancer logs for
connection pool exhaustion.
```

## Conclusion

You now have a pipeline that classifies errors by category and priority, detects rate spikes and new error patterns, and generates incident summaries — all running locally with no external dependencies.

The AI layer adds judgment that regex can't: it understands that "connection refused" and "ECONNREFUSED 127.0.0.1:5432" are the same problem, that P1 database errors need immediate attention, and can synthesize 20 error lines into one actionable incident report.

Wire it into your alerting system by adding a Slack webhook call when `severity == "critical"`, or run it as a sidecar alongside any service that writes structured logs.
