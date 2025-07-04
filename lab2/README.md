# **Lab Guide: Data Transformation & Automation**

Integrating Kafka for real-time network event streaming and transforming data into insightful visualizations and alerts with Grafana and Prometheus. We'll be using Kafka events related to telemetry and alarms from anomaly events coming from advanced analytics from Nokia Network PLatform Services as an example.  
At the end of this lab, you should be able to transform data from any sort of source or subscription into useful information to gain better insight into your network state.

---

## Infrastructure Setup

Following are the steps to set up the environment.

Get into the workspace folder for this part of the lab:

```bash
# change into Part 2 directory
cd /workspaces/autocon3-ws-a3/lab2/kafka-alarm-sim
```

---

## **Kafka-Based Alert Simulation & Export to Prometheus**

This part of the lab focuses on simulating **realistic network alerts** using Kafka producers and exposing them as **Prometheus metrics** using a lightweight Python exporter. The goal is to create a reproducible alert simulation and monitoring environment using Docker.

---

### **1. Build Kafka Producer and Exporter Images**

#### Kafka Alarm Producer

This component simulates network anomaly events coming from Nokia NSP analytics and hard threshold indicators, by producing JSON messages to Kafka topics: `rt-anomalies` and `nsp-act-action-event`. These messages mimic real-time telemetry alerts, serving as input for downstream monitoring and visualization tools.

```bash
docker build -t kafka-alarm-producer . -f Dockerfile.kafka-alarm-producer
```

#### Alert Prometheus Exporter

This lightweight Python service acts as a Kafka consumer and translates incoming anomaly messages into Prometheus-compatible metrics. It exposes these metrics on a `/metrics` endpoint, which Prometheus scrapes periodically for monitoring and alerting.

```bash
docker build -t kafka-alert-exporter  . -f Dockerfile.kafka-alert-exporter
```

---

### **2. Start the Kafka + Exporter Stack**

Start the infrastructure using Docker Compose:

```bash
docker compose up -d
```

This will launch the full monitoring and alert simulation stack, including:

- **Zookeeper**  
  A coordination service required by Kafka for managing brokers and distributed state.

- **Kafka Broker**  
  The core messaging system that handles publishing and subscribing to telemetry and anomaly event topics.

- **Kafka Alarm Producer** *(NSP Kafka Message Simulator — not in the scope of this lab)*  
  Simulates network anomaly messages and pushes them into Kafka topics for downstream processing.

- **Kafka Alert Exporter** *(Prometheus endpoint)*  
  Consumes Kafka messages and exposes them as metrics in Prometheus format at `/metrics`.

- **Grafana**  
  Provides interactive dashboards to visualize Prometheus data. Pre-configured with data sources and sample dashboards.

- **Prometheus**  
  Scrapes metrics from the Kafka Alert Exporter and evaluates alerting rules based on real-time data.

- **Alertmanager**  
  Handles alerts generated by Prometheus and manages routing, silencing, and notifications (email, webhook, etc.).

---

### **5. Prometheus Exporter: Kafka Connectivity & Message Inspection**

Once the container stack is up, verify that the Kafka Alert Exporter is properly connected to the Kafka broker and consuming messages as expected.

---

#### ✅ Check Kafka Connectivity

Ensure that the exporter container can reach the Kafka broker and list available topics:

```bash
docker exec -ti kafka-alert-exporter python3 kafka-check.py --bootstrap-servers kafka:9092
```

**Expected output:**

```shell
✅ Kafka server at 'kafka:9092' is accessible over PLAINTEXT.
📜 Available Topics: test-topic, nsp-act-action-event, rt-anomalies
```

This confirms the Kafka service is up and the exporter can communicate with it.

---

#### 📥 Inspect Kafka Messages Being Consumed

To see live messages being processed by the exporter, run:

```bash
docker exec -ti kafka-alert-exporter python3 kafka-msgs.py --bootstrap-servers kafka:9092
```

You should see output like:

```json
Received [rt-anomalies]: {
  "baselineName": "MTCHSD0102M-PIRRSDBW03M - lag-202  - Egress",
  "neId": "10.13.0.12",
  "counter": "transmitted-octets",
  ...
}
Received [nsp-act-action-event]: {
  "data": {
    "ietf-restconf:notification": {
      "nsp-act:action-event": {
        "type": "Threshold-Crossing",
        "neId": "10.13.0.234",
        ...
      }
    }
  }
}
```

These messages come from simulated telemetry and alarm streams. They are parsed by the Kafka Alert Exporter and translated into Prometheus metrics (defined in `alarms.yml`) such as `anomaly_detected` or other configured counters.

#### Check prometheus exporter

kafka-alert-exporter instance is running `kafka-alert-exporter.py` and consumes messages and translates them into Prometheus metrics.

- Accessible at port 8001 (use port mapping in VSCode for access)
- Defined in `alarms.yml`

Sample metric definition:

```yaml
alarms:
  # - topic: 'rt-anomalies'
  #   partition: 0
  #   alertType: 'baseline'
  #   counters:
  #     anomaly_detected:
  #       name: 'anomaly_detected'
  #       description: 'Anomaly detected in real-time telemetry'
  #       labels: ['neId', 'neName', 'counter']
  - topic: 'nsp-act-action-event'
    partition: 0
    alertType: 'Threshold-Crossing'
    counters:
      threshold_violation:
        name: 'threshold_violation'
        description: 'Threshold Crossing Alert'
        labels: ['source.neId', 'source.neName', 'payload.thresholdName']
```

Here’s a fully revised and enhanced version of the **Monitoring and Validation** section, with clear explanations, added context for using VSCode (locally or via Codespaces), and an **exercise** to uncomment and activate a new alarm stream.

---

## **Monitoring and Validation**

After starting the container stack and confirming messages are flowing, follow the steps below to validate the full monitoring pipeline using Prometheus and Grafana.

---

### **📜 View Exporter Logs**

To see if metrics are being registered and labels extracted correctly, tail the Kafka Alert Exporter logs:

```bash
sudo docker logs --tail 10 kafka-alert-exporter
```

You should see debug logs like:

```shell
DEBUG: Prometheus label source_neId = 10.13.0.234
DEBUG: Updating metric threshold_violation with labels {'source_neId': ..., 'payload_thresholdName': ...}
```

---

### **🔍 Access Monitoring UIs via VSCode**

Depending on your environment:

#### 🔧 If you're using **VSCode Dev Containers (locally)**

1. Open the **"Ports"** tab at the bottom panel in VSCode.
2. Locate and click on any of the following ports to **open in your browser**:
   - **`8001`** — Prometheus Exporter (raw `/metrics` endpoint for scraping)
   - **`9095`** — Prometheus UI (query and inspect metric time series)
   - **`3000`** — Grafana (interactive dashboards) **admin/secret**
   - **`9093`** — Alertmanager (view and manage active alerts)

#### ☁️ If you're using **GitHub Codespaces**

1. Click on **"Ports"** in the top menu.
2. Wait for the port list to populate and click **"Open in Browser"** on the relevant entry.
   - Ensure **port 8001** is visible and published (if not, add it manually or ensure the container is running).

---

### 🔗 What Each Port Serves

| Port   | Service              | Description |
|--------|----------------------|-------------|
| `8001` | **Prometheus Exporter** | Exposes raw metrics collected from Kafka messages (e.g., `anomaly_detected`, `threshold_violation`) |
| `9095` | **Prometheus UI**        | Interface to query time series, check targets, and validate alert rules |
| `3000` | **Grafana**              | Visual dashboards (gauges, graphs) based on Prometheus data |
| `9093` | **Alertmanager**         | View firing alerts and check routing logic |

---

### 🧪 Exercise: Enable Anomaly Detection Metrics

By default, only threshold violations are enabled. Let's now enable the anomaly alert stream:

#### **Step 1: Uncomment anomaly detection in `alarms.yml`**

Edit the file:

```bash
code alarms.yml
```

Uncomment the following block:

```yaml
- topic: 'rt-anomalies'
  partition: 0
  alertType: 'baseline'
  counters:
    anomaly_detected:
      name: 'anomaly_detected'
      description: 'Anomaly detected in real-time telemetry'
      labels: ['neId', 'neName', 'counter']
```

#### **Step 2: Restart the exporter**

Kill and restart the exporter to reload the config:

```bash
sudo docker kill kafka-alert-exporter
sudo docker compose up -d kafka-alert-exporter
```

#### **Step 3: Confirm new metrics appear**

Go to Prometheus Exporter (`http://localhost:8001`) and look for:

- `anomaly_detected`
- `anomaly_detected_count_total`

You should now see output like:

```text
anomaly_detected{counter="transmitted-octets",neId="10.13.0.13",neName="10.13.0.13"} 1.0
anomaly_detected_count_total{...} 1.0
```

Also, check the logs:

```bash
sudo docker logs --tail 20 kafka-alert-exporter
```

Look for messages indicating:

- the new topic `rt-anomalies` is being consumed
- metrics like `anomaly_detected` are being updated

## 🚀 Final Step: Publish Your Own Kafka Exporter Image to GitHub Container Registry (GHCR)

In this section, you’ll learn how to fork the repository, push a public Docker image of your Kafka Alert Exporter to GitHub Container Registry (GHCR), and use that image in your `docker-compose.yml` file.

---

### ✅ Step 1: Fork the Repository

1. Navigate to the GitHub repository provided for this lab.
2. Click the **"Fork"** button (top right) to create a personal copy in your GitHub account.
3. Clone your forked repo locally or continue using GitHub Codespaces.

---

### 🐳 Step 2: Set Up GitHub Actions for Docker Image Publishing

In your forked repo, create a file:  
**`.github/workflows/docker-publish.yml`**

Paste the following workflow (edit the image tag if needed):

```yaml
name: Docker Push kafka-alert-exporter image

on:
  push:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}:v1

jobs:
  build_and_publish:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Docker Login
        run: echo "${{ secrets.GHCR_TOKEN }}" | docker login ${{ env.REGISTRY }} --username ${{ github.actor }} --password-stdin

      - name: Docker Build
        working-directory: lab2/kafka-alarm-sim
        run: docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} -f Dockerfile.kafka-alert-exporter .

      - name: Docker Push
        run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
```

---

### 🔐 Step 3: Create a GitHub Token for Publishing

1. Go to **https://github.com/settings/tokens/new**  
2. Create a **Personal access tokens (classic)** with:
   - **Repository access**: your forked repo
   - **Permissions**:
     - `contents: read`
     - `packages: write`
   - Name it something like `GHCR Token`
   - Copy the Token String and keep it safe for use in next step.

3. In your forked repository:
   - Go to **Settings > Secrets and variables > Actions**
   - Create a **new secret**:
     - **Name**: `GHCR_TOKEN`
     - **Value**: *paste the token you just generated*

---

### 🧪 Step 4: Trigger the Build

Push a commit to the `main` branch (e.g., update README or re-push the workflow file).  
The workflow will build the image and push it to:

```text
ghcr.io/<your-username>/<repo-name>:v1
```

You can verify it under your GitHub profile → **Packages**.

---
### 🛠️ Step 5: Change Package to Public

Open the Package Page: GitHub profile → **Packages**.

1. Select the package created in the previous step
2. Go to **Package Settings**:
   On the package page, click on the "Package Settings" button on the right side of the screen.
3. Change Visibility:
   Under the **"Danger Zone"** you should see the visibility settings. Click on Change Visibility
4. Change from **Private** to **Public**.
5. **Confirm** the Change:
   GitHub may ask you to confirm this change. Confirm it if prompted.

---

### 🛠️ Step 6: Update `docker-compose.yml`

Now that you’ve published your image, update the `docker-compose.yml` to use it:

```yaml
  kafka-alert-exporter:
    image: ghcr.io/<your-username>/<repo-name>:v1
    ...
```

Save the file and restart the stack:

```bash
docker compose up -d --force-recreate kafka-alert-exporter
```

---

### 🏁 You're Done!

You now:

- Own the Kafka exporter image in GHCR
- Use CI/CD with GitHub Actions
- Understand how to make Docker services portable and public

This same process applies to **any microservice** you want to publish or share with others. 🎉
=====
I have done the changes.