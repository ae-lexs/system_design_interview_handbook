# 07 — Distributed Coordination

> How distributed systems achieve agreement and mutual exclusion.

**Prerequisites:** [01 — Foundational Concepts](./01_FOUNDATIONAL_CONCEPTS.md), [02 — Communication Patterns](./02_COMMUNICATION_PATTERNS.md), [03 — Consistency & Transactions](./03_CONSISTENCY_AND_TRANSACTIONS.md)
**Builds toward:** [08 — Resilience Patterns](./08_RESILIENCE_PATTERNS.md), [09 — Scaling & Infrastructure](./09_SCALING_AND_INFRASTRUCTURE.md)
**Estimated study time:** 2-3 hours

---

## Chapter Overview

This module presents production-grade patterns for coordination in distributed systems. These patterns address the fundamental challenges of mutual exclusion and distributed transactions that emerge when systems span multiple nodes.

---

## 1. Distributed Locking

> **References:**
> - Lamport, L. (1978). "Time, Clocks, and the Ordering of Events in a Distributed System." Communications of the ACM.
> - Martin Kleppmann's analysis (2016). "How to do distributed locking." (critique of Redlock algorithm)

### The Problem

Multiple processes across different nodes need mutually exclusive access to a shared resource.

```mermaid
sequenceDiagram
    participant P1 as Process 1
    participant P2 as Process 2
    participant R as Shared Resource

    P1->>R: Acquire lock
    P2->>R: Acquire lock
    Note over R: Who wins? Race condition!
    P1->>R: Modify resource
    P2->>R: Modify resource
    Note over R: Data corruption!
```

### Solution: Distributed Lock Manager

```mermaid
sequenceDiagram
    participant P1 as Process 1
    participant P2 as Process 2
    participant LM as Lock Manager (Redis/ZK)
    participant R as Shared Resource

    P1->>LM: SETNX lock:resource owner=P1 TTL=30s
    LM->>P1: OK (lock acquired)
    P2->>LM: SETNX lock:resource owner=P2 TTL=30s
    LM->>P2: FAIL (already locked)

    P1->>R: Safely modify resource
    P1->>LM: DEL lock:resource

    P2->>LM: SETNX lock:resource owner=P2 TTL=30s
    LM->>P2: OK (lock acquired)
```

### Implementation Patterns

#### Redis-Based Lock (Redlock)

```python
import redis
import uuid
import time

class DistributedLock:
    def __init__(self, redis_client, lock_name, ttl_ms=30000):
        self.redis = redis_client
        self.lock_name = f"lock:{lock_name}"
        self.ttl_ms = ttl_ms
        self.token = str(uuid.uuid4())

    def acquire(self, retry_count=3, retry_delay_ms=100):
        """Attempt to acquire the lock with retries."""
        for attempt in range(retry_count):
            # SET NX with TTL (atomic operation)
            if self.redis.set(
                self.lock_name,
                self.token,
                nx=True,  # Only set if not exists
                px=self.ttl_ms  # TTL in milliseconds
            ):
                return True
            time.sleep(retry_delay_ms / 1000)
        return False

    def release(self):
        """Release lock only if we own it (compare-and-delete)."""
        # Lua script ensures atomic check-and-delete
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        return self.redis.eval(lua_script, 1, self.lock_name, self.token)

    def extend(self, additional_ms):
        """Extend TTL if we still own the lock."""
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("pexpire", KEYS[1], ARGV[2])
        else
            return 0
        end
        """
        return self.redis.eval(
            lua_script, 1, self.lock_name,
            self.token, self.ttl_ms + additional_ms
        )
```

#### ZooKeeper-Based Lock (Sequential Ephemeral Nodes)

```mermaid
sequenceDiagram
    participant P1 as Process 1
    participant P2 as Process 2
    participant ZK as ZooKeeper

    P1->>ZK: Create /locks/resource/lock-0001 (ephemeral, sequential)
    P2->>ZK: Create /locks/resource/lock-0002 (ephemeral, sequential)

    P1->>ZK: List children of /locks/resource
    ZK->>P1: [lock-0001, lock-0002]
    Note over P1: I'm lowest → I have the lock

    P2->>ZK: List children of /locks/resource
    ZK->>P2: [lock-0001, lock-0002]
    Note over P2: I'm not lowest → watch lock-0001

    P2->>ZK: Watch /locks/resource/lock-0001

    Note over P1: Process 1 finishes
    P1->>ZK: Delete /locks/resource/lock-0001
    ZK->>P2: Watch triggered (node deleted)

    P2->>ZK: List children of /locks/resource
    ZK->>P2: [lock-0002]
    Note over P2: I'm lowest now → I have the lock
```

### Lock Safety Considerations

```mermaid
flowchart TD
    subgraph Problems["Lock Safety Problems"]
        P1[Lock Expiry<br/>Before Work Done]
        P2[Split Brain<br/>Two Lock Holders]
        P3[Client Crash<br/>Lock Never Released]
        P4[Clock Skew<br/>TTL Mismatch]
    end

    P1 -->|Solution| S1[Fencing Tokens]
    P2 -->|Solution| S2[Quorum-Based Locking]
    P3 -->|Solution| S3[TTL + Heartbeat Extension]
    P4 -->|Solution| S4[Logical Clocks]
```

#### Fencing Tokens Pattern

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant C2 as Client 2
    participant LM as Lock Manager
    participant DB as Database

    C1->>LM: Acquire lock
    LM->>C1: Token=33 (monotonically increasing)

    Note over C1: GC pause / network delay
    Note over LM: TTL expires, lock released

    C2->>LM: Acquire lock
    LM->>C2: Token=34

    C2->>DB: Write (token=34)
    DB->>DB: Record token=34

    C1->>DB: Write (token=33)
    DB->>DB: Reject: 33 < 34
    DB->>C1: Error: stale token
```

### Complexity Analysis: Distributed Locks

| Operation | Redis Single | Redlock (N nodes) | ZooKeeper |
|-----------|--------------|-------------------|-----------|
| **Acquire** | O(1), 1 RTT | O(N), N RTTs | O(1), 2 RTTs |
| **Release** | O(1), 1 RTT | O(N), N RTTs | O(1), 1 RTT |
| **Failure Detection** | TTL expiry | TTL expiry | Session timeout |
| **Network Calls** | 1 | N (quorum) | 2-3 (leader) |

### Trade-Off Analysis

| Approach | Consistency | Availability | Complexity | Use Case |
|----------|-------------|--------------|------------|----------|
| **Redis Single** | Weak | High | Low | Non-critical mutual exclusion |
| **Redlock (Multi)** | Moderate | Moderate | Medium | Important but not life-critical |
| **ZooKeeper** | Strong | Moderate | High | Critical coordination |
| **etcd** | Strong | Moderate | Medium | Kubernetes-native apps |
| **Database** | Strong | Lower | Low | Already have RDBMS |

---

## 2. Distributed Transactions

> **References:**
> - Gray, J. (1978). "Notes on Database Operating Systems." Operating Systems: An Advanced Course.
> - Garcia-Molina, H. & Salem, K. (1987). "Sagas." ACM SIGMOD.
> - Mohan, C. et al. (1986). "Transaction Management in the R* Distributed Database System." ACM TODS.

### The Two Generals Problem

```mermaid
sequenceDiagram
    participant G1 as General 1
    participant M as Messenger
    participant G2 as General 2

    G1->>M: "Attack at dawn"
    Note over M: Might be captured!
    M->>G2: "Attack at dawn"
    G2->>M: "Acknowledged"
    Note over M: Might be captured!
    M->>G1: "Acknowledged"

    Note over G1,G2: Neither can be certain<br/>the other received the message
```

**Implication:** Perfect consensus in unreliable networks is impossible. We accept trade-offs.

### Two-Phase Commit (2PC)

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P1 as Participant 1
    participant P2 as Participant 2

    rect rgb(230, 245, 255)
        Note over C,P2: Phase 1: Prepare (Vote)
        C->>P1: PREPARE
        C->>P2: PREPARE
        P1->>P1: Write to WAL, acquire locks
        P2->>P2: Write to WAL, acquire locks
        P1->>C: VOTE_COMMIT
        P2->>C: VOTE_COMMIT
    end

    rect rgb(230, 255, 230)
        Note over C,P2: Phase 2: Commit
        C->>C: Decision: COMMIT (all voted yes)
        C->>P1: COMMIT
        C->>P2: COMMIT
        P1->>P1: Apply changes, release locks
        P2->>P2: Apply changes, release locks
        P1->>C: ACK
        P2->>C: ACK
    end
```

#### 2PC Failure Scenarios

```mermaid
stateDiagram-v2
    [*] --> Init
    Init --> Prepared: PREPARE received
    Prepared --> Committed: COMMIT received
    Prepared --> Aborted: ABORT received

    Prepared --> Blocked: Coordinator crashes<br/>after PREPARE

    note right of Blocked
        Participant holds locks
        Cannot proceed until
        coordinator recovers
    end note

    Committed --> [*]
    Aborted --> [*]
```

| Failure | State | Resolution |
|---------|-------|------------|
| Participant fails before PREPARE | Init | Coordinator aborts |
| Participant fails after VOTE | Prepared | Recovery from WAL |
| Coordinator fails before decision | Prepared | **BLOCKED** until recovery |
| Coordinator fails after decision | Committed/Aborted | Participants query coordinator |

### Transaction Protocol Complexity

| Protocol | Message Complexity | Time Complexity | Blocking | Partition Tolerance |
|----------|-------------------|-----------------|----------|---------------------|
| **2PC** | O(3n) | O(2 RTT) | Yes (coordinator crash) | No |
| **3PC** | O(4n) | O(3 RTT) | Reduced | No |
| **Saga** | O(2n) | O(n steps) | No | Yes |
| **Paxos Commit** | O(n²) | O(2 RTT) | No | Yes |

Where n = number of participants.

### Three-Phase Commit (3PC)

Adds a `PRE_COMMIT` phase to reduce blocking, but still not partition-tolerant.

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P1 as Participant 1
    participant P2 as Participant 2

    C->>P1: CAN_COMMIT?
    C->>P2: CAN_COMMIT?
    P1->>C: YES
    P2->>C: YES

    Note over C,P2: Added Phase: Pre-Commit
    C->>P1: PRE_COMMIT
    C->>P2: PRE_COMMIT
    P1->>C: ACK
    P2->>C: ACK

    C->>P1: DO_COMMIT
    C->>P2: DO_COMMIT
```

**Key Insight:** If participant times out after PRE_COMMIT, it can safely commit (knows others were ready).

### Saga Pattern (Preferred for Microservices)

Long-lived transactions as a sequence of local transactions with compensating actions.

```mermaid
flowchart LR
    subgraph "Saga: Order Processing"
        T1[Create Order]
        T2[Reserve Inventory]
        T3[Process Payment]
        T4[Ship Order]
    end

    subgraph "Compensations"
        C1[Cancel Order]
        C2[Release Inventory]
        C3[Refund Payment]
    end

    T1 -->|Success| T2
    T2 -->|Success| T3
    T3 -->|Success| T4

    T3 -->|Failure| C2
    C2 --> C1

    T2 -->|Failure| C1
```

#### Choreography vs Orchestration

```mermaid
flowchart TB
    subgraph Choreography["Choreography (Event-Driven)"]
        O1[Order Service]
        I1[Inventory Service]
        P1[Payment Service]

        O1 -->|OrderCreated| I1
        I1 -->|InventoryReserved| P1
        P1 -->|PaymentProcessed| O1

        P1 -->|PaymentFailed| I1
        I1 -->|InventoryReleased| O1
    end

    subgraph Orchestration["Orchestration (Central Coordinator)"]
        ORC[Saga Orchestrator]
        O2[Order Service]
        I2[Inventory Service]
        P2[Payment Service]

        ORC --> O2
        ORC --> I2
        ORC --> P2

        O2 --> ORC
        I2 --> ORC
        P2 --> ORC
    end
```

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| **Coupling** | Loose (event-based) | Central coordinator |
| **Visibility** | Hard to track flow | Clear execution path |
| **Complexity** | Distributed logic | Centralized logic |
| **Failure handling** | Each service handles | Orchestrator handles |
| **Best for** | Simple sagas | Complex business logic |

#### Saga Implementation Example

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Callable, Optional

class SagaStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    FAILED = "failed"

@dataclass
class SagaStep:
    name: str
    action: Callable
    compensation: Callable
    status: str = "pending"

class SagaOrchestrator:
    def __init__(self, saga_id: str, steps: List[SagaStep]):
        self.saga_id = saga_id
        self.steps = steps
        self.current_step = 0
        self.status = SagaStatus.PENDING
        self.completed_steps: List[SagaStep] = []

    async def execute(self, context: dict) -> bool:
        """Execute saga steps forward."""
        self.status = SagaStatus.RUNNING

        for i, step in enumerate(self.steps):
            self.current_step = i
            try:
                # Execute the step
                await step.action(context)
                step.status = "completed"
                self.completed_steps.append(step)

                # Persist saga state (for recovery)
                await self._persist_state()

            except Exception as e:
                step.status = "failed"
                await self._compensate(context, str(e))
                return False

        self.status = SagaStatus.COMPLETED
        return True

    async def _compensate(self, context: dict, error: str):
        """Execute compensation in reverse order."""
        self.status = SagaStatus.COMPENSATING

        # Compensate in reverse order
        for step in reversed(self.completed_steps):
            try:
                await step.compensation(context)
                step.status = "compensated"
            except Exception as comp_error:
                # Log and continue - compensation must be best-effort
                step.status = "compensation_failed"

        self.status = SagaStatus.FAILED

    async def _persist_state(self):
        """Persist saga state for crash recovery."""
        # Store to database: saga_id, current_step, completed_steps, status
        pass
```

---

## Chapter Summary

### Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│           DISTRIBUTED COORDINATION CHEAT SHEET                  │
├─────────────────────────────────────────────────────────────────┤
│ DISTRIBUTED LOCKING:                                            │
│   • Redis SETNX + TTL: Simple, fast, weak consistency          │
│   • ZK ephemeral nodes: Strong consistency, higher latency      │
│   • Fencing Token: Monotonic counter prevents stale operations │
│   • Redlock: Multi-node Redis for improved safety              │
├─────────────────────────────────────────────────────────────────┤
│ DISTRIBUTED TRANSACTIONS:                                       │
│   • 2PC: Prepare→Commit (blocking, strong consistency)         │
│   • 3PC: Adds PRE_COMMIT phase to reduce blocking              │
│   • Saga: Local TXNs + compensations (non-blocking, eventual)  │
│   • Choreography: Event-driven, loosely coupled                │
│   • Orchestration: Central coordinator, clear visibility       │
├─────────────────────────────────────────────────────────────────┤
│ DECISION GUIDE:                                                 │
│   • Non-critical mutex        → Redis Single                   │
│   • Important coordination    → Redlock or ZooKeeper           │
│   • Microservice transactions → Saga Pattern                   │
│   • Strong consistency needed → 2PC (if acceptable blocking)   │
├─────────────────────────────────────────────────────────────────┤
│ INTERVIEW TIPS:                                                 │
│   1. Always mention trade-offs (consistency vs availability)   │
│   2. Discuss failure scenarios and recovery                    │
│   3. Explain why Saga is preferred for microservices           │
│   4. Know the Two Generals Problem implication                 │
└─────────────────────────────────────────────────────────────────┘
```

### Interview Articulation Patterns

> "When would you use 2PC vs Saga?"

"2PC provides strong consistency but blocks during failures—it's suitable when you need atomic commits across a small number of participants, like in a single datacenter. Saga trades consistency for availability by using compensating transactions; it's better for microservices where blocking is unacceptable and you can tolerate brief inconsistency."

> "How do you handle distributed locking safely?"

"I'd use Redis with SETNX + TTL for simplicity, but always include a unique token for safe release. For critical operations, I'd add fencing tokens so the resource can reject stale lock holders. If strong consistency is required, ZooKeeper's sequential ephemeral nodes provide better guarantees at the cost of higher latency."

---

## Practice Questions

1. Design a distributed lock service that handles network partitions gracefully. What guarantees can you provide?

2. A saga orchestrator crashes mid-transaction. How do you ensure the system recovers to a consistent state?

3. Compare the trade-offs between choreography and orchestration for a 5-step saga involving payment, inventory, and shipping services.

4. Why can't the Two Generals Problem be solved? How does this affect distributed transaction design?

---

## Revision History

| Date | Change |
|------|--------|
| 2025-01 | Extracted from 06_REPLICATION_AND_PARTITIONING.md (Part I: Coordination Patterns) |

---

## Navigation

**Previous:** [06 — Replication & Partitioning](./06_REPLICATION_AND_PARTITIONING.md)
**Next:** [08 — Resilience Patterns](./08_RESILIENCE_PATTERNS.md)
**Index:** [README](./README.md)
