# Context-Aware ChatOps Alerting & Shift Management System

This project implements an intelligent, context-aware ChatOps system. It integrates server monitoring with active administrator shift schedules to ensure that critical server alerts are routed **only** to the specific administrator currently on duty via Telegram. Additionally, it provides in-chat tools for administrators to manage schedules (swap shifts) and acknowledge incidents, measuring the **Mean Time to Acknowledge (MTTA)**.

---

## 🛠️ System Architecture & Tech Stack

This system is built upon the following pillars:

1.  **Monitoring Layer:** Prometheus & Node Exporter (Metrics collection).
2.  **Visualization & Alerting:** Grafana (Dashboards and alert rule definitions).
3.  **Automation Engine (The Brain):** **n8n** (Handles all logic routing, database interactions, and Telegram API communication).
4.  **Database Layer:** PostgreSQL (Stores admin schedules, user data, and incident logs).
5.  **Interface Layer:** Telegram Bot (User interaction via commands and interactive buttons).

---

## ✨ Key Features

-   ✅ **Context-Aware Alerting:** Routes critical alerts directly to the personal Telegram chat of the administrator currently assigned to the active shift.
-   ✅ **Detailed Notifications:** Telegram messages include specific details: instance IP, alert status, error type, and a descriptive message.
-   ✅ **Interactive Incident Management:** Alerts include an "Acknowledge" button. Clicking this button updates the status, hides the button, and notifies the group that the issue is being handled.
-   ✅ **MTTA Measurement:** Automatically calculates and logs the time difference between the alert occurrence (`waktu_kejadian`) and administrator acknowledgment (`waktu_ditangani`) in PostgreSQL.
-   ✅ **In-Chat Shift Tools:** Administrators can view the current schedule and request/execute dynamic shift swaps directly from Telegram using commands (`/jadwal`, `/tukar`).
-   ✅ **Security:** Includes logic to expire inline keyboard buttons after use to prevent re-execution of stale commands.

---

## 🆘 Emergency Patch: Grafana Alert Recovery

If you accidentally deleted your Grafana alert rules, use this section as a reference to recreate them using the optimized A-B-C expression structure (Query -> Reduce -> Threshold) to ensure labeled data (like Instance IP) is always included in the webhook payload.

### 1. Alert: HostHighCpuLoad (Reference: image_16a4da.jpg)

Based on the deleted rule shown in the screenshot, recreate the alert with these settings:

#### **Step 1: Define query and alert condition**

* **A (Prometheus Query):**
    ```promql
    100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
    ```
    *(This queries the actual CPU usage percentage. **Do not** add `> 80` inside this query box).*

* **B (Expression - Reduce):**
    * **Input:** `A`
    * **Function:** `Last`
    * *(This takes the most recent value from Query A)*.

* **C (Expression - Threshold):**
    * **Input:** `B`
    * **Condition:** `IS ABOVE`
    * **Value:** `80`
    * *(This defines the firing condition: CPU load > 80%).*

* **Set as Alert Condition:** Make sure **C** is set as the alert condition.

#### **Step 2: Set evaluation behavior**

* **Evaluate every:** `1m`
* **Pending period (For):** `0s` (for immediate firing during demos) or `1m` (recommended for production to avoid false positives).

#### **Step 3: Add details (Annotations)**

Crucial for the n8n webhook workflow to receive detailed information:

* **Summary:** `Host High CPU Load on {{ $labels.instance }}`
* **Description:** `CPU load is > 80%. Reported Value: {{ $values.B.Value | printf "%.2f" }}%`

---

## 💾 Database Schema Reference (PostgreSQL)

Before activating n8n workflows, ensure these tables exist in your PostgreSQL database:

```sql
-- Table for storing administrator shift schedules
CREATE TABLE jadwal_shift (
    id SERIAL PRIMARY KEY,
    telegram_id VARCHAR(50) NOT NULL,
    nama_admin VARCHAR(100) NOT NULL,
    hari_shift DATE NOT NULL
);

-- Table for logging incidents and measuring MTTA
CREATE TABLE log_insiden (
    id SERIAL PRIMARY KEY,
    nama_alert VARCHAR(100),
    ip_server VARCHAR(50),
    pesan_error TEXT,
    waktu_kejadian TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    admin_piket VARCHAR(50),
    waktu_ditangani TIMESTAMP, -- Populated when admin clicks 'Acknowledge'
    status_penanganan VARCHAR(20) DEFAULT 'Menunggu Respons'
);
