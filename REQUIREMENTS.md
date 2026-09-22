# Requirements, quality attributes and tactics

This document lists the requirements of the pasta-production case study and the architectural tactics chosen to satisfy them. The paper in this repository verifies only the availability subset, which is marked below. Everything else is part of the case study but is deliberately outside the scope of the formal model, which is kept small on purpose.

## Quality-attribute scenarios

| ID | Attribute | Scenario | In the model |
|---|---|---|---|
| QA-1 | Availability | When a machine or the scheduler fails, the system continues to execute a valid production plan and recovers within an acceptable time bound. | Yes |
| QA-2 | Modifiability | The production plan and machine configuration can be adjusted without code changes and without interrupting production. | No |
| QA-3 | Scalability | Machines can be integrated at runtime and the workload redistributed, while responsiveness stays within the specified limits. | Partly (AV-6) |

## Functional requirements

| ID | Name | Description | In the model |
|---|---|---|---|
| FR-1 | Fresh pasta A | The system shall produce fresh pasta of type A. | No |
| FR-2 | Dried pasta A | The system shall produce dried pasta of type A. | No |
| FR-3 | Fresh pasta B | The system shall produce fresh pasta of type B. | No |
| FR-4 | Dried pasta B | The system shall produce dried pasta of type B. | No |
| FR-6 | Machine heartbeats | Each machine shall report that it is working correctly using heartbeats. | Yes (AV-3, AV-4) |
| FR-7 | Heartbeat monitoring | The scheduler shall check heartbeats and detect missing or new ones. | Yes (AV-3, AV-6) |
| FR-11 | Packaging time | The system shall package a product in less than 10 s. | No |
| FR-12 | Product sorting | The sorting system shall route a product either to the drying machine or directly to packaging. | No |
| FR-13 | Plan modification | When the production plan is modified, a machine shall react within 10 s. | Yes (AV-6) |
| FR-14 | Storage notification | When storage exceeds the limit for product A or B, the database shall inform the scheduler within 5 s. | No |
| FR-15 | Threshold cooldown | After a blade switch on a cutting machine, repeated threshold triggers shall be ignored for 5 s. | No |

## Non-functional requirements

| ID | Name | Description | In the model |
|---|---|---|---|
| NFR-5 | Production modifiability | The system shall adjust production according to the current system state. | Partly (AV-6, AV-7) |
| NFR-8 | Software deployability | The system shall allow software updates with minimal downtime. | No |
| NFR-9 | Scalability | The system shall support adding new machines without redesign. | Partly (AV-6) |
| NFR-10 | Availability | The system shall operate continuously with an uptime of at least 99.9%. | No (see note) |
| NFR-16 | Maintainability | The system shall support modular replacement of devices without downtime. | No |
| NFR-17 | Security | All external access shall be secured. | No |
| NFR-18 | Audit trail | All operator actions shall be logged for auditing. | No |
| NFR-19 | Message reliability | All messages between devices and the bus shall be delivered at least once. | Assumed |

Note on NFR-10: failures in the model are nondeterministic rather than probabilistic, so an uptime target cannot be established by symbolic model checking. What the model does establish is a bound on the recovery time after a failure, which is the part of availability that the architecture controls. NFR-19 is assumed rather than verified: the model treats message delivery as reliable and instantaneous.

## Architectural constraints

| ID | Constraint | Mitigation |
|---|---|---|
| C-1 | Centralised scheduling: the scheduler is the single coordination point that computes and distributes the production plan. | Standby scheduler with Redis-based leader election; the scheduler reacts to events rather than polling, so it is not a throughput bottleneck. |
| C-2 | Fixed heartbeat periods and thresholds: detection and storage limits depend on configured values that are not yet managed at runtime. | An Update Manager is foreseen to adjust them during production. It is neither implemented nor modelled. |

## Availability requirements verified in UPPAAL

These refine FR-6, FR-7 and QA-1 into properties that can be checked. B is the baseline model, R the refined one. Times are in seconds.

| ID | Requirement | B | R |
|---|---|---|---|
| AV-1 | At most one scheduler is active at any time. | pass | pass |
| AV-2 | If the active scheduler fails, a standby becomes active within the failover timeout (6 in B, 4 in R). | pass | pass |
| AV-3 | A silent machine is declared failed within 11 of its last heartbeat. | pass | pass |
| AV-4 | A machine that keeps emitting heartbeats is never declared failed. | pass | pass |
| AV-5 | A corrective decision follows detection within 5. Design parameter, no query. | -- | -- |
| AV-6 | A machine that reports itself ready is started within 10. | pass | pass |
| AV-7 | Every machine failure is handled within 25, regardless of scheduler failures. | fail (38) | pass (25) |
| AV-8 | The model is free of deadlocks. | pass | pass |

AV-3 and AV-6 are scoped to intervals without a scheduler failure, since they state the guarantee of a single mechanism. AV-7 is unscoped and is the requirement that the baseline violates.

## Architectural tactics

| Quality attribute | Tactic | Realisation | Verified here |
|---|---|---|---|
| Availability | Fault detection: heartbeat | Machines emit a heartbeat every 5 s; the scheduler declares a loss after 11 s of silence. | Yes (AV-3, AV-4) |
| Availability | Recovery: warm spare | A standby scheduler takes over when the active one stops renewing its lease. | Yes (AV-1, AV-2) |
| Availability | Recovery: state resynchronisation | Detection state is kept in Redis, so an incoming scheduler continues detection rather than restarting it. | Yes (AV-7) |
| Availability | Leader election with a lease | The active scheduler renews a Redis key every 3 s with a 4 s time-to-live; expiry triggers the takeover. | Yes (AV-1, AV-2, AV-7) |
| Availability | Fault isolation | Publish-subscribe communication prevents a failed consumer from blocking producers. | No, assumed by the model |
| Availability, reliability | At-least-once delivery | Kafka between core services, MQTT towards the machines. | No, assumed by the model |
| Modifiability | Intermediary | An event bus between the scheduler and the machines removes direct dependencies. | No |
| Modifiability | Restrict dependencies | Machine-specific protocols are confined to an adapter, so the scheduler is independent of machine types. | No |
| Modifiability | Defer binding | The production plan is data rather than code, and thresholds are configuration values; an Update Manager for runtime changes is foreseen. | No |
| Scalability | Publish-subscribe | New machines join by publishing heartbeats, with no registry to update. | Partly (AV-6) |
| Scalability | Stateless integration | A machine is admitted to the plan on the basis of its heartbeats alone. | Partly (AV-6) |
| Security | Limit access | External access is restricted to the dashboard, which reads from the Redis cache. | No |
| Auditability | Audit trail | Operator actions and system events are retained in Kafka topics. | No |
| Performance | Event-driven processing | The scheduler recomputes the plan on events instead of polling. | No |
