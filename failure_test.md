================================================================================
PATRONI & ETCD CLUSTER FAILURE TESTING & BEHAVIORAL ANALYSIS LOG
================================================================================

1. QUORUM LOSS & READ-ONLY DEGRADATION (2-NODE SCENARIO)
--------------------------------------------------------------------------------
- Test Setup: Cluster reduced to 2 active nodes.
- Failure Injected: Powered off Node 2 while Node 1 was running as Leader.
- Observed Behavior:
  * Node 1 lost the etcd quorum (DCS key update failed).
  * Patroni automatically demoted/safeguarded the Leader to prevent split-brain.
  * Node 1 degraded its local PostgreSQL engine into READ-ONLY mode despite 
    technically holding the leader role prior to quorum loss.
- Conclusion: Confirmed Patroni's strict DCS lease contract—PostgreSQL will not
  accept writes without active distributed consensus lock verification.

2. ETCD STORAGE QUOTA & DATABASE LOCKOUT RISK
--------------------------------------------------------------------------------
- Impact: High
- Mechanism: etcd enforces a strict storage quota (default 2GB). If keyspace 
  revisions grow without compacting, etcd hits its quota alarm and switches 
  to a read-only state.
- Observed Behavior:
  * When etcd triggers a quota alarm, Patroni cannot extend leader leases 
    or write state updates.
  * Patroni degrades the PostgreSQL database engine to READ-ONLY mode.
- Operational Remediation:
  * Regularly monitor etcd space usage (`etcdctl endpoint status`).
  * Implement automated compaction and defragmentation jobs (`etcdctl compact` 
    and `etcdctl defrag`).

3. HIGH-AVAILABILITY FAILOVER VALIDATION (3-NODE QUORUM)
--------------------------------------------------------------------------------
- Test Setup: Healthy 3-node cluster running Patroni + etcd DCS.
- Manual Switchover: Executed `patronictl switchover`. Traffic seamlessly 
  handed off to the target standby with 0 data loss.
- Automatic Failover: Simulated hard VM crash by abruptly turning off the 
  active primary instance.
- Observed Behavior:
  * etcd TTL expired for the primary node lease.
  * Healthy standbys held quorum, evaluated timeline/LSN positions, and 
    promoted the most up-to-date node.
  * HAProxy updated health check routing (port 15432 write endpoint) to 
    the new Leader in < 5 seconds.

4. TIMELINE DIVERGENCE & REINITIALIZATION RECOVERY
--------------------------------------------------------------------------------
- Issue: Standby node experienced LSN/timeline divergence from the primary 
  timeline following a cascading restart event.
- Root Cause: Node attempted recovery on an orphaned timeline branch.
- Remediation Executed:
  * Issued `patronictl -c /etc/patroni/patroni.yml reinit <cluster-name> <node-name>`
  * Patroni safely wiped the divergent local cluster state, re-cloned the 
    base database from the active Leader via pg_basebackup, and reconciled 
    the timeline tracking.

5. PAUSE / RESUME MAINTENANCE MODE STUCK STATES & WORKAROUND
--------------------------------------------------------------------------------
- Observed Anomaly: Executing `patronictl resume` occasionally failed to clear 
  the `pause` flag from the DCS, leaving Patroni stuck in maintenance mode.
- Impact: Auto-failover functionality remains silently disabled while in pause state.
- Root Cause: Intermittent API latency / race condition when clearing DCS key.
- Safe Mitigation / Fix:
  * Avoid relying solely on standard `patronictl resume` during rapid changes.
  * Explicitly remove the flag by enforcing the edit-config update:

    sudo patronictl -c /etc/patroni/patroni.yml edit-config -s pause=false --force

  * Operational Verification: Always verify that `is_paused` evaluates to 
    `false` in `patronictl list` before leaving maintenance windows, otherwise 
    automatic failovers will not trigger during outages.
================================================================================
