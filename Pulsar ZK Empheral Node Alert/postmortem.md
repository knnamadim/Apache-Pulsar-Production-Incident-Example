# Apache Pulsar Production Incident Postmortem  
**Incident:** BookKeeper Ledger Fencing & Topic Read Failures  
**Topic:** `persistent://cc/cdr/cdr-acct-records-partition-0`  
**Subscription:** `cdrs-aaa-subscription`  
**Date:** 2024-10-01  

---

## 1. Overview
On October 1, 2024, a partitioned Pulsar topic experienced repeated **read failures** due to BookKeeper ledger fencing. Consumers were unable to make progress, leading to increased backlog. The broker logs reported multiple errors including:

ManagedLedgerException: LastConfirmedEntry is null when reading ledger <ledger-id>


This postmortem documents the **incident timeline**, **root cause**, **mitigation**, and **preventive measures**.

---

## 2. Incident Timeline

### 2.1 Normal Operating State
- Topic owned by **broker-1**
- Producers and consumers operating normally
- BookKeeper ledgers actively written and read
- No alerts triggered

### 2.2 Trigger Event
- Likely caused by:
  - Broker restart or transient instability
  - BookKeeper quorum disruption
  - Network latency or partitioning
  - Topic hotspot with high throughput
- Ledgers ownership state became unstable

### 2.3 Ledger Fencing Begins
BookKeeper fenced multiple ledgers to protect data integrity:

Ledger: 19423 fenced by: /127.0.0.6:41627
Ledger: 19480 fenced by: /127.0.0.6:41911
Ledger: 43917 fenced by: /127.0.0.6:35403


> Fencing prevents concurrent writes to the same ledger and protects against data corruption.

### 2.4 Broker Read Failures
Broker logs reported:

ManagedLedgerException: LastConfirmedEntry is null when reading ledger 43917


- Broker could not determine the last confirmed entry
- Consumers unable to read messages
- Multiple ledgers affected

### 2.5 Consumer Impact
- Subscription `cdrs-aaa-subscription` stalled
- Backlog increased
- Read failures persisted across multiple ledgers

### 2.6 Mitigation Action
The topic was unloaded from the broker to reset state:

pulsar-admin topics unload persistent://cc/cdr/cdr-acct-records-partition-0

- Released broker ownership
- Cleared in-memory managed ledger state
- Allowed clean reassignment

### 2.7 Recovery
- Topic reassigned to a broker successfully
- Consumers resumed normal processing
- BookKeeper reads resumed without further fencing

### 3. Root Cause Analysis
- Multiple ledgers were fenced due to broker instability or BookKeeper quorum issues
- Broker was unable to determine LastConfirmedEntry, leading to read failures
- Topic was temporarily “hot,” causing high load on a single broker
- Zookeeper coordination remained stable, but ledger state was inconsistent

### 4. Preventive Measures
Monitoring
- Alert on repeated ledger fencing
- Track LastConfirmedEntry read failures
- Monitor broker heap usage and BookKeeper latency
- Track Zookeeper ephemeral node growth

Operational Safeguards
- Proactively unload hot topics
- Ensure sufficient BookKeeper quorum and disk capacity
- Review topic partitioning to distribute load evenly
- Document and monitor broker failover processes

### 5. Key Takeaways
- Ledger fencing is a protective mechanism, not a failure by itself.
- Broker instability or BookKeeper disruptions can block consumers.
- Unloading topics can safely reset broker-ledger state.
- Proactive monitoring and alerting can prevent incidents from impacting consumers.

### 6. References
- Apache Pulsar Documentation – Ledger Fencing
- BookKeeper ManagedLedger Exception Handling