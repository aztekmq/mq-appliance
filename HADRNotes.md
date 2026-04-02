A partitioned queue manager on an IBM MQ Appliance HA/DR setup is one of those “don’t guess—be surgical” situations. If you handle it wrong, you can make recovery harder or even lose data consistency.

Let’s walk through what’s actually happening and the safest way to fix it.

⸻

🧠 What “partitioned QMGR” means

In an HA pair, MQ appliances rely on:
	•	shared state + replication
	•	heartbeat between appliances

A partitioned queue manager typically means:
	•	Both appliances think they should NOT take over (split-brain prevention), OR
	•	One side lost sync and MQ blocked failover to protect data

Common causes:
	•	Network break between HA nodes (heartbeat failure)
	•	DR link interference or latency
	•	Storage/replication inconsistency
	•	Appliance reboot during sync

⸻

🔍 Step 1 — Check current state (DO THIS FIRST)

On both appliances, run:

show qmgr <QMGR_NAME>

and:

show ha-status

Look for:
	•	Partitioned: yes
	•	Which node is primary / secondary
	•	Sync status (in-sync / out-of-sync)

Also check logs:

show log mq
show log system


⸻

⚠️ CRITICAL RULE

Before fixing anything:

👉 You must decide which appliance has the “correct” data

Typically:
	•	The one that was last active primary
	•	The one still accessible to applications
	•	The one NOT rebooted or isolated

If unsure → stop and verify. This is where people cause data loss.

⸻

🛠️ Fix Scenario 1: One node is clearly correct (most common)

Step A — Disable HA on the BAD node

On the appliance you want to discard:

configure terminal
no ha enable

(or depending on version)

ha disable


⸻

Step B — Reset its state

delete ha-state

(or)

clear ha-replication

(exact command varies slightly by MQ version, but goal = wipe HA sync state)

⸻

Step C — Rejoin HA

ha enable

Then monitor:

show ha-status

Wait until:
	•	status = in-sync
	•	no longer partitioned

⸻

🛠️ Fix Scenario 2: Both nodes stuck / neither primary

This is trickier.

Step A — Force one node to become primary

On the node you trust:

force ha take-over

(or)

ha force-primary


⸻

Step B — Disable HA on the other node

Same as above:

ha disable


⸻

Step C — Rebuild secondary

Re-enable HA so it resyncs cleanly.

⸻

🛠️ Fix Scenario 3: DR interference (HA/DR setups like yours)

Since you’re running HA + DR (4 appliances):

👉 Partitioning can be caused by:
	•	DR replication lag confusing HA quorum
	•	Incorrect routing between Texas ↔ Virginia

Check:

show dr-status

If DR is out of sync:
	•	Fix HA first (local pair)
	•	Then re-establish DR

⸻

🔧 Additional commands you’ll likely need

Show everything quickly:

show qmgr
show ha-status
show dr-status
show mq-status

Restart qmgr (only if needed):

restart qmgr <QMGR_NAME>


⸻

🚨 Things NOT to do
	•	❌ Don’t force both nodes active (split brain)
	•	❌ Don’t restart both appliances blindly
	•	❌ Don’t rejoin HA without clearing bad state
	•	❌ Don’t assume DR is irrelevant (it often is the cause)

⸻

🧭 Real-world guidance (based on your setup)

Given your environment:
	•	Texas HA pair
	•	Virginia HA pair
	•	DR between them

👉 Most likely root cause:
	•	Temporary network partition between HA nodes OR
	•	DR link causing confusion during failover

Best practice fix path:
	1.	Stabilize local HA pair first
	2.	Ensure it is in-sync and clean
	3.	Then reintroduce DR sync

⸻

🧠 Pro Tip (this saves people constantly)

If this happens repeatedly:
	•	Check latency between HA nodes (must be very low)
	•	Ensure heartbeat interfaces are isolated
	•	Verify no routing asymmetry between appliances
	•	Confirm no firewall drops on HA ports
  
