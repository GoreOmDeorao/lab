# Prometheus and Grafana — Practical Monitoring Lab

Practical steps and PromQL from Prometheus slides 2–14.

## 1. Install and run Prometheus on Ubuntu

Download the Linux AMD64 tar.gz release from [Prometheus downloads](https://prometheus.io/download/). Then in the directory containing the downloaded archive:

```bash
cd ~/Downloads
tar -xzf prometheus-*.linux-amd64.tar.gz
mv prometheus-*.linux-amd64 prometheus
cd ~/Downloads/prometheus
ls -l
./prometheus
```

Open `http://localhost:9090`. Keep this terminal running.

> The slide shows a version-specific directory name with an ellipsis. The wildcard above avoids hardcoding the version. If the downloaded archive extracts to a different directory name, use that exact extracted name in `mv`.

## 2. Clone and start the Flask bank application

Open a new terminal:

```bash
cd ~
git clone https://github.com/newdelthis/prometheus-app.git
cd ~/prometheus-app/bank-app
sudo apt update
sudo apt install -y python3-venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python app.py
```

Open `http://localhost:5000`. Leave this terminal running.

## 3. Run traffic and inspect application metrics

Open another terminal:

```bash
cd ~/prometheus-app/bank-app
source venv/bin/activate
python traffic.py
```

Open a third terminal and inspect metrics:

```bash
curl -s localhost:5000/metrics | grep '^bank_'
```

## 4. Flask metrics instrumentation fragment from the slides

The slide explicitly labels this section **read, don't type**. It is a fragment, not a complete standalone `app.py`: the Flask app, imports, request-field variables, timer setup, and full `Response` arguments are not included in the supplied slide.

```python
REQUESTS = Counter(
    "bank_http_requests_total",
    "HTTP requests",
    ["method", "endpoint", "status"]
)

LATENCY = Histogram(
    "bank_request_duration_seconds",
    "Request duration",
    ["endpoint"]
)

TOTAL_BALANCE = Gauge("bank_total_balance", "Sum")

@app.after_request
def record_metrics(response):
    LATENCY.labels(endpoint).observe(elapsed)
    REQUESTS.labels(method, endpoint, status).inc()
    return response

@app.route("/metrics")
def metrics():
    return Response(generate_latest(), ...)
```

Do not paste this fragment into a new Python file and expect it to run without the missing imports and variables.

## 5. Add the bank app scrape job

Edit `prometheus.yml` in the Prometheus installation directory. Add this job under the existing top-level `scrape_configs:` list:

```yaml
  - job_name: bank-app
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:5000"]
```

Restart Prometheus from the same extracted directory:

```bash
cd ~/Downloads/prometheus
# Press Ctrl+C in the Prometheus terminal, then:
./prometheus
```

Verify target health:

```bash
curl -s localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"'
```

In the Prometheus UI, open **Status → Target health** and verify that `bank-app` is `UP`.

If the Flask app is on another machine, replace `localhost` in `targets` with that machine's reachable IP and allow access to TCP port 5000.

## 6. PromQL queries

Open `http://localhost:9090`, enter each query in the query field, and select the Graph view.

**Is the app up?**

```promql
up{job="bank-app"}
```

**Requests per second by endpoint:**

```promql
sum by (endpoint) (rate(bank_http_requests_total[1m]))
```

**Share of requests that fail:**

```promql
sum(rate(bank_http_requests_total{status=~"5.."}[1m]))
/
sum(rate(bank_http_requests_total[1m]))
```

**95th percentile response time:**

```promql
histogram_quantile(
  0.95,
  sum by (le) (rate(bank_request_duration_seconds_bucket[1m]))
)
```

**Withdrawals refused in the last five minutes:**

```promql
increase(bank_transactions_total{result="insufficient_funds"}[5m])
```

**Total bank balance:**

```promql
bank_total_balance
```

Allow `traffic.py` to run for a couple of minutes before querying `rate()` so there is enough data.

## 7. Test app and traffic failure behavior

Stop the Flask app with `Ctrl+C` in its terminal. The `up{job="bank-app"}` query should drop to 0 after a scrape attempt.

Restart the app:

```bash
cd ~/prometheus-app/bank-app
source venv/bin/activate
python app.py
```

Stop the traffic generator with `Ctrl+C` in its terminal. Request rate should fall to 0 while the balance gauge remains at its current value.

## 8. Install and connect Grafana

Download and install Grafana using the instructions at [Grafana downloads](https://grafana.com/grafana/download).

Open `http://localhost:3000`. The slide lists the initial username and password as `admin` / `admin`; follow the first-login password-change prompt if shown.

In Grafana:

1. Open **Connections → Add new connection**.
2. Select **Prometheus** and add a Prometheus data source.
3. Set the Prometheus Server URL to `http://localhost:9090`.
4. Click **Save & test**.

## 9. Create a Grafana dashboard

Create a dashboard and add panels with these queries.

**Panel 1 — App status**

```promql
up{job="bank-app"}
```

**Panel 2 — Requests by endpoint**

```promql
sum by (endpoint) (
  rate(bank_http_requests_total[1m])
)
```

**Panel 3 — Failed requests percentage**

```promql
100 *
sum(rate(bank_http_requests_total{status=~"5.."}[1m]))
/
sum(rate(bank_http_requests_total[1m]))
```

**Panel 4 — 95th percentile response time**

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(bank_request_duration_seconds_bucket[1m])
  )
)
```

**Panel 5 — Total bank balance**

```promql
bank_total_balance
```

**Panel 6 — Failed withdrawals**

```promql
increase(
  bank_transactions_total{
    result="insufficient_funds"
  }[5m]
)
```

Source reference: Prometheus slides 2–14.
