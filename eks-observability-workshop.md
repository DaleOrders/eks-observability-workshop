# EKS Observability Workshop

Estimated time: 1.5 hours

This hands-on workshop teaches you how to build a simple and complete observability stack on Amazon EKS using:
- Prometheus for metrics collection
- Grafana for dashboards and visualization
- OpenTelemetry for trace instrumentation
- Jaeger for distributed tracing

The workshop uses a tiny Python app that exposes Prometheus metrics and exports OpenTelemetry traces. This keeps the demo simple while illustrating how each observability layer works.

---

## 1. Workshop overview

### What you'll learn

- Create an Amazon EKS cluster
- Deploy Prometheus for metrics collection
- Deploy Grafana for metric visualization
- Deploy Jaeger and OpenTelemetry Collector for tracing
- Deploy a simple instrumented application
- Generate traffic, inspect metrics, and view traces
- Clean up all resources when done

### Why this matters

Observability helps you understand your system's behavior from the outside. In this workshop:
- Prometheus collects numeric metrics
- Grafana turns metrics into dashboards
- OpenTelemetry instruments requests and exports traces
- Jaeger stores and visualizes traces

---

## 2. Understanding the observability stack

This section explains what each tool does and how it fits into the observability story.

### Prometheus: Metrics collection

**What it is:** Prometheus is a time-series database and metrics collection tool.

**What it does:**
- Prometheus regularly polls (scrapes) HTTP endpoints to read metrics
- It stores metrics as time series: a metric name, labels, and a value at each point in time
- It allows you to query metrics using PromQL, a query language for time-series data
- Common metrics include request counts, latency, CPU usage, memory usage, and error rates

**Example metrics:**
- `http_requests_total` — total number of HTTP requests
- `http_request_duration_seconds` — time taken to serve requests
- `kubernetes_pod_cpu_usage` — CPU usage per pod

**How it's configured in this workshop:**
- Install Prometheus using Helm
- Create a `prometheus.yml` configuration that defines scrape targets
- Point Prometheus at the demo app's `/metrics` endpoint so it can scrape metrics
- Point Prometheus at the OpenTelemetry Collector's metrics endpoint
- Prometheus runs inside the cluster and stores metrics in memory

**Key file in this workshop:**
- `prometheus-values.yaml` defines the scrape interval (15 seconds) and target endpoints

---

### Grafana: Visualization and dashboards

**What it is:** Grafana is a visualization platform that turns metrics into dashboards.

**What it does:**
- Grafana connects to a data source such as Prometheus
- It reads time-series metrics from Prometheus
- It renders metrics as graphs, gauges, heatmaps, and tables
- You can create custom dashboards by adding panels, each powered by a PromQL query
- Grafana also supports alerting based on metric thresholds

**Example dashboard:**
- A graph of `http_requests_total` over time shows request volume
- A gauge of the latest CPU usage value shows current load
- A heatmap of `http_request_duration_seconds_bucket` shows latency distribution

**How it's configured in this workshop:**
- Install Grafana using Helm
- Set `adminPassword` to `"grafana123"` for the admin user
- Configure Prometheus as the default data source so Grafana can query metrics
- Point Grafana to `http://prometheus-server.monitoring.svc.cluster.local` (the Prometheus service in the cluster)

**Key file in this workshop:**
- `grafana-values.yaml` sets the Prometheus data source and admin credentials

---

### OpenTelemetry: Instrumentation and trace export

**What it is:** OpenTelemetry is a vendor-neutral standard for generating and exporting observability data.

**What it does:**
- OpenTelemetry libraries instrument your code to create spans (trace records)
- A span represents a unit of work: a function call, an HTTP request, a database query
- Spans have attributes such as start time, duration, status, and labels
- OpenTelemetry can auto-instrument popular frameworks like Flask, Django, and FastAPI
- OpenTelemetry exports spans to a backend via the OTLP protocol (OpenTelemetry Protocol)

**Example span:**
- An HTTP request span records: method, path, status code, duration, and service name
- A nested database query span records: SQL query, database, duration

**How it's configured in this workshop:**
- The demo app imports OpenTelemetry libraries: `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-http`, `opentelemetry-instrumentation-flask`
- It creates a `TracerProvider` and attaches a `BatchSpanProcessor` that sends spans to the collector
- It calls `FlaskInstrumentor().instrument_app(app)` to auto-instrument Flask
- It manually creates spans using `tracer.start_as_current_span()` for custom code
- Spans are exported to `http://otel-collector.tracing.svc.cluster.local:4318/v1/traces`

**Key code in this workshop:**
```python
tracer_provider = TracerProvider(resource=resource)
otlp_exporter = OTLPSpanExporter(endpoint="http://otel-collector.tracing.svc.cluster.local:4318/v1/traces")
tracer_provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
trace.set_tracer_provider(tracer_provider)
```

---

### Jaeger: Distributed tracing storage and visualization

**What it is:** Jaeger is a distributed tracing backend that stores and visualizes traces.

**What it does:**
- Jaeger receives spans from the OpenTelemetry Collector
- It stores spans in memory (or a persistent backend like Elasticsearch or Cassandra)
- The Jaeger UI allows you to search for traces by service, operation, or tags
- It visualizes traces as waterfall diagrams showing the timeline of spans
- Jaeger calculates service dependencies and shows critical path analysis

**Example trace visualization:**
- A user request comes in at time T0
- Flask auto-instrumentation creates a span from T0 to T1
- Within that span, your custom code creates a nested span from T0.5 to T0.8
- The waterfall shows both spans and their durations side by side

**How it's configured in this workshop:**
- Deploy Jaeger as an all-in-one container using the image `jaegertracing/all-in-one:1.44`
- This includes the collector, storage, and query UI all in one pod
- Expose the Jaeger query UI on port `16686` so you can browse traces
- The OpenTelemetry Collector forwards spans to Jaeger on port `14250` (gRPC)

**Key file in this workshop:**
- `jaeger-all-in-one.yaml` defines the Jaeger service and deployment
- `otel-collector.yaml` configures the collector to export traces to `jaeger.tracing.svc.cluster.local:14250`

---

### How they work together

1. **Demo app** generates metrics and traces
2. **Prometheus** scrapes `/metrics` and stores metric data
3. **Grafana** queries Prometheus and displays dashboards
4. **OpenTelemetry Collector** receives spans via OTLP and forwards them to Jaeger
5. **Jaeger** stores spans and provides a UI to search and visualize traces

This separation of concerns allows you to:
- Swap out Prometheus for another metrics backend without changing the app
- Add a second observability vendor alongside the first
- Scale each component independently

---

## 3. Prerequisites

### Required tools

- AWS account with permissions to create EKS, IAM, VPC, and managed node groups
- `aws` CLI installed and configured (`aws configure`)
- `kubectl`
- `eksctl`
- `helm`
- `git`

### Optional tools

- `jq`
- A browser for Grafana, Prometheus, and Jaeger

### Verify your environment

```bash
aws sts get-caller-identity
kubectl version --client
eksctl version
helm version
```

If any command fails, fix the local setup before continuing.

---

## 4. Create the EKS cluster

### Step 1: Set variables

These variables are referenced by every subsequent command. `CLUSTER_NAME` identifies your cluster. `AWS_REGION=us-west-2` (Oregon) supports all EKS features used here. `NODE_TYPE=t3.medium` gives each worker 2 vCPUs and 4 GB RAM — enough to run all workshop components. `NODE_COUNT=2` spreads pods across two nodes. Change any value before running, but keep it consistent throughout.

```bash
export CLUSTER_NAME=observability-workshop
export AWS_REGION=us-west-2
export NODE_TYPE=t3.medium
export NODE_COUNT=2
```

### Step 2: Create the cluster

Creates a VPC, subnets, EKS control plane, and managed worker nodes in a single command. `eksctl` uses CloudFormation behind the scenes, so the entire cluster can be torn down cleanly later. The `--managed` flag means AWS handles node patching and replacement. This takes 10–15 minutes — do not interrupt it or you may leave orphaned AWS resources. When it finishes, `~/.kube/config` is updated automatically.

```bash
eksctl create cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --node-type "$NODE_TYPE" \
  --nodes "$NODE_COUNT" \
  --managed
```

This operation can take 10–15 minutes.

### Step 3: Verify cluster access

Confirm `kubectl` is talking to the right cluster before deploying anything. Both nodes should show `Ready`; `kubectl get namespaces` confirms the control plane is fully initialized. If nodes show `NotReady`, wait two minutes and retry. If `kubectl` can't reach the API server, run `aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"` to refresh your config.

```bash
kubectl get nodes
kubectl get namespaces
```

Expected result:
- `kubectl get nodes` shows one or more worker nodes in `Ready` state
- `kubectl get namespaces` shows standard namespaces such as `default`, `kube-system`, and `kube-public`

If the cluster is not ready, wait a few minutes and retry the commands.

---

## 5. Install Prometheus

Prometheus collects metrics by scraping endpoints exposed by your application and Kubernetes components.

### Step 1: Create the monitoring namespace

Creates a dedicated namespace for Prometheus and Grafana, separate from the application. This prevents name collisions and simplifies cleanup — deleting the namespace removes everything inside it. Helm requires the namespace to exist before installing into it.

```bash
kubectl create namespace monitoring
```

### Step 2: Add the Prometheus Helm repository

Registers the Prometheus community chart repository under the alias `prometheus-community` and fetches the latest chart list. Run `helm repo update` whenever you add a new repository to avoid installing stale versions.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Step 3: Create a Prometheus values file

Overrides chart defaults without modifying the chart itself. This file disables Alertmanager (not needed here), exposes Prometheus via a `LoadBalancer`, and defines two scrape jobs — `demo-app` and `otel-collector` — polling every 15 seconds. Targets use Kubernetes internal DNS names that resolve automatically inside the cluster.

```bash
cat <<'EOF' > prometheus-values.yaml
alertmanager:
  enabled: false
server:
  service:
    type: LoadBalancer
serverFiles:
  prometheus.yml:
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'demo-app'
        metrics_path: /metrics
        static_configs:
          - targets: ['demo-app.app.svc.cluster.local:80']
      - job_name: 'otel-collector'
        metrics_path: /metrics
        static_configs:
          - targets: ['otel-collector.tracing.svc.cluster.local:4318']
EOF
```

### Step 4: Install Prometheus

Deploys Prometheus into the `monitoring` namespace using your values file. The release name `prometheus` is how Helm tracks this installation — use it to upgrade or uninstall later. If something goes wrong, run `helm uninstall prometheus -n monitoring` before retrying. Allow one to two minutes for the pod to start after the command completes.

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  -f prometheus-values.yaml
```

### Step 5: Confirm Prometheus

Wait for `prometheus-server` to show `Running` with `1/1` ready before continuing. The `EXTERNAL-IP` column may show `<pending>` for one to three minutes while AWS provisions the load balancer. If the pod stays in `Pending` beyond five minutes, run `kubectl describe pod -n monitoring <pod-name>` and check the Events section for the cause.

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Expected result:
- `prometheus-server` pod is running
- `prometheus-server` service is present

If the LoadBalancer service does not yet have an external IP, wait a few minutes and rerun `kubectl get svc -n monitoring`.

### Step 6: Access Prometheus

Tunnels your local port `9090` to the Prometheus Service without needing a public IP. This terminal is occupied while the tunnel runs — open a new one for subsequent steps. At `http://localhost:9090`, check Status → Targets to confirm the demo app and OTel Collector are being scraped successfully.

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
```

Then open `http://localhost:9090` in your browser.

### Why this step matters

Prometheus is the central metrics collector. It regularly polls endpoints such as `/metrics`, stores the results, and makes them queryable.

---

## 6. Install Grafana

Grafana is the visualization layer. It reads Prometheus metrics and turns them into dashboards.

### Step 1: Add the Grafana Helm repository

Register the Grafana Helm charts.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Step 2: Create a Grafana values file

This file configures Grafana to use Prometheus as its default data source.

```bash
cat <<'EOF' > grafana-values.yaml
service:
  type: LoadBalancer
persistence:
  enabled: false
adminPassword: "grafana123"
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus-server.monitoring.svc.cluster.local
EOF
```

### Step 3: Install Grafana

Deploy Grafana into the same `monitoring` namespace.

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  -f grafana-values.yaml
```

### Step 4: Confirm Grafana

Check that the Grafana pod and service are running.

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring | grep grafana
```

Expected result:
- `grafana` pod is running
- `grafana` service exists

### Step 5: Access Grafana

If the service has an external IP, open it in the browser. Otherwise, use port forwarding.

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Open `http://localhost:3000`, then log in with:
- Username: `admin`
- Password: `grafana123`

### Why this step matters

Grafana allows you to build dashboards and explore metric trends visually. It is the most useful tool for sharing observability data across teams.

---

## 7. Deploy a simple instrumented application

This demo app is intentionally small to illustrate observability concepts clearly.
It exposes a Prometheus metrics endpoint and exports OpenTelemetry traces.

### Step 1: Create the application namespace

Create a separate namespace for the demo app.

```bash
kubectl create namespace app
```

### Step 2: Create the demo app manifest

This manifest creates three resources:
- a `ConfigMap` containing the Python app code
- a `Deployment` that runs the app in two replicas
- a `Service` that exposes the app internally

```bash
cat <<'EOF' > demo-app.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-app-code
  namespace: app
data:
  server.py: |
    from flask import Flask, request
    import random
    import time
    from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
    from opentelemetry import trace
    from opentelemetry.sdk.resources import Resource
    from opentelemetry.sdk.trace import TracerProvider
    from opentelemetry.sdk.trace.export import BatchSpanProcessor
    from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
    from opentelemetry.instrumentation.flask import FlaskInstrumentor

    app = Flask(__name__)
    resource = Resource(attributes={"service.name": "demo-app"})
    tracer_provider = TracerProvider(resource=resource)
    otlp_exporter = OTLPSpanExporter(endpoint="http://otel-collector.tracing.svc.cluster.local:4318/v1/traces")
    tracer_provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
    trace.set_tracer_provider(tracer_provider)
    tracer = trace.get_tracer(__name__)

    REQUEST_COUNT = Counter(
      "demo_http_requests_total",
      "Total HTTP requests",
      ["method", "endpoint", "status"],
    )
    REQUEST_LATENCY = Histogram(
      "demo_http_request_duration_seconds",
      "HTTP request latency in seconds",
      ["endpoint"],
    )

    FlaskInstrumentor().instrument_app(app)

    @app.route("/")
    def home():
      start = time.time()
      with tracer.start_as_current_span("home-handler"):
        time.sleep(random.uniform(0.05, 0.25))
        response = "Hello from the observability demo!"
      duration = time.time() - start
      REQUEST_COUNT.labels(method=request.method, endpoint=request.path, status="200").inc()
      REQUEST_LATENCY.labels(endpoint=request.path).observe(duration)
      return response

    @app.route("/metrics")
    def metrics():
      return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}

    if __name__ == "__main__":
      app.run(host="0.0.0.0", port=5000)
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo-app
          image: python:3.11-slim
          command: ["/bin/sh", "-c"]
          args:
            - |
              pip install flask prometheus_client opentelemetry-sdk opentelemetry-exporter-otlp-proto-http opentelemetry-instrumentation-flask && \
              python /app/server.py
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: app-code
              mountPath: /app
      volumes:
        - name: app-code
          configMap:
            name: demo-app-code
---

apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: app
spec:
  selector:
    app: demo-app
  ports:
    - port: 80
      targetPort: 5000
      protocol: TCP
  type: ClusterIP
EOF
```

### Step 3: Deploy the demo app

Apply the manifest to create the app resources.

```bash
kubectl apply -f demo-app.yaml
```

### Step 4: Confirm the app

Check that the deployment and service exist.

```bash
kubectl get pods -n app
kubectl get svc -n app
```

Expected result:
- two `demo-app` pods in `Running` state
- one `demo-app` service of type `ClusterIP`

If pods are not ready, inspect the logs or pod status:

```bash
kubectl describe pod -n app <pod-name>
kubectl logs -n app <pod-name>
```

### Why this step matters

The demo app is the source of observable data. It generates metrics and traces for Prometheus and Jaeger to collect.

---

## 8. Install Jaeger and OpenTelemetry Collector

These components receive and store tracing data from the demo app.

### Step 1: Create the tracing namespace

Create a separate namespace for tracing components.

```bash
kubectl create namespace tracing
```

### Step 2: Deploy Jaeger

Jaeger all-in-one includes the query UI, collector, and storage in a single process.

```bash
cat <<'EOF' > jaeger-all-in-one.yaml
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: tracing
spec:
  selector:
    app: jaeger
  ports:
    - name: query
      port: 16686
      targetPort: 16686
    - name: collector-grpc
      port: 14250
      targetPort: 14250
    - name: collector-http
      port: 14268
      targetPort: 14268
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: tracing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.44
          ports:
            - containerPort: 16686
            - containerPort: 14250
            - containerPort: 14268
EOF
```

```bash
kubectl apply -f jaeger-all-in-one.yaml
```

### Step 3: Deploy the OpenTelemetry Collector

This collector receives OTLP traces from the app and exports them to Jaeger.

```bash
cat <<'EOF' > otel-collector.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: tracing
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          http:
    processors:
      batch:
    exporters:
      jaeger:
        endpoint: jaeger.tracing.svc.cluster.local:14250
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [jaeger]
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: tracing
spec:
  selector:
    app: otel-collector
  ports:
    - name: otlp-http
      port: 4318
      targetPort: 4318
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: tracing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector:0.86.0
          command: ["otelcol"]
          args: ["--config=/conf/otel-collector-config.yaml"]
          ports:
            - containerPort: 4318
          volumeMounts:
            - name: config
              mountPath: /conf
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
            items:
              - key: otel-collector-config.yaml
                path: otel-collector-config.yaml
EOF
```

```bash
kubectl apply -f otel-collector.yaml
```

### Step 4: Confirm tracing components

Verify that Jaeger and the collector are running.

```bash
kubectl get pods -n tracing
kubectl get svc -n tracing
```

Expected result:
- a `jaeger` pod in `Running` state
- an `otel-collector` pod in `Running` state
- `jaeger-query` and `otel-collector` services present

### Why this step matters

The collector decouples the application from the tracing backend. The app sends traces to the collector, and the collector forwards them to Jaeger.

---

## 9. Generate traffic and inspect observability data

This section validates that the system is collecting real observability data.

### Step 1: Port-forward the demo app

Open one terminal and run:

```bash
kubectl port-forward -n app svc/demo-app 8080:80
```

This makes the app available at `http://localhost:8080`.

### Step 2: Generate traffic

Open a second terminal and send requests to the app. Each request will create metrics and traces.

```bash
for i in {1..10}; do curl -s http://localhost:8080/ > /dev/null; sleep 0.5; done
curl -s http://localhost:8080/metrics | head -n 20
```

Expected result:
- the app returns a simple greeting
- the `/metrics` page shows Prometheus counters and histograms

### Step 3: Open Prometheus

If Prometheus is not directly accessible, port-forward it:

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
```

Then open `http://localhost:9090` in your browser.

Run these queries to confirm collected metrics:
- `demo_http_requests_total`
- `rate(demo_http_requests_total[1m])`
- `histogram_quantile(0.95, sum(rate(demo_http_request_duration_seconds_bucket[1m])) by (le))`

Each query returns data points derived from the app's `/metrics` output.

### Step 4: Open Grafana

Port-forward Grafana if needed:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Open `http://localhost:3000`, log in with `admin / grafana123`, and verify that Prometheus is configured as the data source.

Create a simple panel using the query `demo_http_requests_total` to see request volume over time.

### Step 5: Open Jaeger

Port-forward Jaeger query UI:

```bash
kubectl port-forward -n tracing svc/jaeger-query 16686:16686
```

Open `http://localhost:16686` and search for service `demo-app`.

Inspect a trace to see:
- the request span created by the app
- how long the request took
- the trace context propagated by OpenTelemetry

### Why this step matters

This is the actual validation step. It proves that:
- Prometheus can read metrics from the app
- Grafana can visualize those metrics
- OpenTelemetry can capture traces
- Jaeger can display distributed traces

---

## 10. Clean up all resources

### Step 1: Delete namespaces and temporary files

```bash
kubectl delete namespace app monitoring tracing || true
rm -f demo-app.yaml jaeger-all-in-one.yaml otel-collector.yaml prometheus-values.yaml grafana-values.yaml
```

### Step 2: Delete the EKS cluster

```bash
eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
```

### Step 3: Confirm cleanup

```bash
kubectl get namespaces
aws eks list-clusters --region "$AWS_REGION"
```

This cleans the cluster, worker nodes, load balancers, and AWS resources created by the workshop.

---

## 11. Recap of key concepts

- Prometheus scrapes `/metrics` endpoints and stores time-series data.
- Grafana reads Prometheus and turns metrics into dashboards.
- OpenTelemetry instruments your application and exports spans.
- The OpenTelemetry Collector receives trace data and forwards it to Jaeger.
- Jaeger stores traces and visualizes request timing.

These tools together help you understand application performance, latency, and service behavior.

---

## 12. Timing guide

- Cluster creation: 15 minutes
- Prometheus + Grafana setup: 15 minutes
- Jaeger + OpenTelemetry Collector: 15 minutes
- Demo app deployment: 10 minutes
- Traffic and exploration: 25 minutes
- Cleanup: 10 minutes

Estimated total: 1.5 hours
