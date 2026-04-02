I pulled the actual dashboard JSON and cross-referenced it with what IBM MQ + MQ Appliance HA/DR health monitoring actually requires. I’m going to break this into 3 parts:

⸻

🧠 1. What you should be monitoring (HA/DR health model)

From IBM MQ + Prometheus integration, the core health signals are:
	•	Queue manager health
	•	Channel health (RUNNING vs not)
	•	Queue depth / backlog
	•	Throughput (put/get rates)
	•	Resource pressure (CPU, disk)
	•	Connections / handles
	•	Replication / HA state (critical for appliance)  ￼

👉 The problem: most exporters (including yours) don’t expose HA/DR state directly
→ You have to infer health from indirect signals.

⸻

🔍 2. What your dashboard HAS vs MISSING

I analyzed the JSON (fitchos/mqa-prometheus dashboard). Here’s the real breakdown:

⸻

✅ Metrics you DO have (good coverage)

🟢 Queue Manager Activity
	•	ibmmq_qmgr_mqput_total
	•	ibmmq_qmgr_mqget_total
	•	ibmmq_qmgr_mqput1_total

✔ Gives:
	•	Throughput
	•	Dead/stalled qmgr detection

⸻

🟢 Queue Depth / Backlog
	•	ibmmq_queue_current_depth
	•	ibmmq_queue_max_depth

✔ Critical for:
	•	Backpressure detection
	•	App issues

⸻

🟢 Channel Status
	•	ibmmq_channel_status
	•	ibmmq_channel_messages

✔ This is HUGE:
	•	You can detect broken DR/HA links
	•	Status mapping (RUNNING vs STOPPED)  ￼

⸻

🟢 Connections
	•	ibmmq_qmgr_connections

✔ Helps identify:
	•	Maxhandle pressure
	•	Client disconnect events

⸻

🟢 Resource Metrics (partial)
	•	CPU (if node exporter present)
	•	Some disk usage (depending on config)

✔ Needed for:
	•	appliance stress detection

⸻

❌ Metrics you are MISSING (this is the real issue)

These are the ones you NEED for HA/DR health but are NOT present:

⸻

🔴 HA STATE (CRITICAL GAP)

Missing:
	•	active / standby role
	•	sync status
	•	partitioned state

👉 There is NO metric like:
	•	mq_ha_role
	•	mq_ha_sync_status
	•	mq_ha_partitioned

🚨 This is why you can’t directly detect:
	•	split brain
	•	partitioned qmgr
	•	failover health

⸻

🔴 DR STATE (CRITICAL GAP)

Missing:
	•	replication lag
	•	DR link status

👉 No:
	•	mq_dr_replication_status
	•	mq_dr_lag_seconds

⸻

🔴 QMGR STATUS (binary up/down)

Missing:
	•	qmgr_running_state

You’re inferring from activity instead of knowing definitively.

⸻

🔴 LOG / ERROR SIGNALS

Missing:
	•	FDC / error logs
	•	AMQ errors

⸻

🔴 DISK CRITICAL THRESHOLDS (Appliance-specific)

You likely don’t have:
	•	log filesystem usage
	•	linear log saturation

⸻

⚠️ Bottom line on your current dashboard

👉 It is good for performance monitoring
👉 It is NOT sufficient for HA/DR health monitoring

⸻

🏗️ 3. Build the dashboard you ACTUALLY want

You asked for:

table of qmgrs by quads (HA/DR pairs), worst health first

Here’s exactly how to do it.

⸻

🧩 Step 1 — Define a HEALTH SCORE

You need a computed metric like:

health_score =
(
  channel_down * 5 +
  queue_depth_ratio * 3 +
  no_activity * 4 +
  high_cpu * 2
)


⸻

Example components:

🚨 Channel health

max by(qmgr) (ibmmq_channel_status != 3)

(3 = RUNNING)

⸻

🚨 Queue pressure

max by(qmgr) (
  ibmmq_queue_current_depth / ibmmq_queue_max_depth
)


⸻

🚨 No activity (dead qmgr)

rate(ibmmq_qmgr_mqput_total[5m]) == 0


⸻

🚨 Connection drop

ibmmq_qmgr_connections == 0


⸻

🧩 Step 2 — Group by QUAD (this is key for your topology)

You MUST label your metrics like:

qmgr="QM1"
site="Texas"
ha_group="A"
dr_group="1"
quad="Texas-A"

Then:

max by(quad, qmgr) (health_score)


⸻

🧩 Step 3 — Build the TABLE panel

Columns:

Quad	QMGR	Health Score	Channel Status	Queue Pressure	Activity


Sort:

ORDER BY health_score DESC

👉 Worst at top = exactly what you want

⸻

🧩 Step 4 — Add HA/DR awareness (workaround)

Since exporter doesn’t expose HA:

Option A (recommended)

Scrape appliance REST:

/ibmmq/rest/v2/admin/status/ha

Create custom metric:

mq_ha_role{qmgr="QM1"} = 1 (primary) / 0 (secondary)
mq_ha_sync = 1/0
mq_partitioned = 1/0


⸻

Option B (infer HA indirectly)
	•	If BOTH nodes active → split brain
	•	If NO channels running → failover issue

⸻

🧭 4. What your final dashboard should look like

🔴 TOP PANEL (Table)
	•	QMgr by quad
	•	Sorted by worst health

⸻

🟡 SECOND PANEL
	•	HA status (custom metrics)

⸻

🟢 THIRD PANEL
	•	Channel matrix (RUNNING vs NOT)

⸻

🔵 FOURTH PANEL
	•	Queue pressure heatmap

⸻

💡 Real talk (based on your environment)

Given your:
	•	Texas HA pair
	•	Virginia HA pair
	•	DR between them

👉 Your current dashboard will NOT catch partitioned qmgr

You need:
	1.	Custom HA metrics (REST scrape)
	2.	Health scoring logic
	3.	Quad grouping labels
