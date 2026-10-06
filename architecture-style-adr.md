# ADR-001: Service-Based Architecture with a Scalable Test-Taker Service

**Status**: Proposed <br>
**Date**: 2026-09-21

## Context

"Make the Grade" is a web-based testing system for 800,000+ students, with up to 200,000 taking timed tests at the same time. Student answers and the grade for each question are eventually consolidated into a single relational test answer database that allows a maximum of 300 connections. Two administrators handle test creation, scheduling, roster maintenance, and reporting. The system must be delivered within 6 months.


## Driving architecture characteristics

| Characteristic | Why it matters |
|---|---|
| **Data Integrity/Durability** | "It is absolutely imperative that no student answers are ever lost for any reason." |
| **Scalability** | Up to 200,000 concurrent test-takers during scheduled windows, with little load outside them. The database allows only 300 connections. |
| **Feasibility (time)** | The system is needed in 6 months. This is a constraint that drives the choice of style. |
| **Security**| It is vital that the test answers be protected from students trying to hack into the system. |

## Decision

We will use a service-based architecture with domain services, with these boundaries:

* **Admin domain**: Roster maintenance, test and answer-key maintenance, and scheduling. It is low-load and stays plain service-based.
* **Reporting domain**: Generates student, teacher, school, and question-validation reports after testing. It stays service-based and reads from the consolidated database.
* **Test-Taker domain**: Presents questions, captures answers, and durably stores them. It is designed as an independently deployable, scalable, isolated unit so its internal style can differ from the others. Given the scale (200k concurrent users vs. 300 DB connections), we will decide in next ADR: implement it in a message-based from the start or start with service-based plus a buffering layer and move to message-based at threshold X. In either case, an answer is acknowledged to the student only after it has been durably stored.


Additionally, mandatory rule is that an answer must never be acknowledged to the student (and the next question must never be presented) until the answer has been durably stored:

* **Durably stored** means the answer is persisted in a way that survives the failure of any single node, for example by synchronous replication across multiple nodes and/or a durable log, before the acknowledgment is sent.
* Acknowledging after an in-memory write on a single node is not sufficient.
* If durable storage cannot be confirmed, the answer is not acknowledged, and the client retries. Submissions must be idempotent so retries do not create duplicates.


## Rationale

* Service-based gives good feasibility, simplicity, and cost for a 6-month timeline, with far less complexity than microservices, event-driven or space-based everywhere.
* Isolating the Test-Taker means the one component with extreme scale demands can use a different style without affecting the rest.
* Separating Test-Taker from Admin isolates answer keys and admin functions from student-facing traffic, which supports security.
* Because tests are scheduled, the Answer Storage capacity is pre-scaled ahead of each scheduled test window and scaled down afterwards. Reactive elasticity is a safety net, not the primary mechanism.

## Consequences
Positive:

* Fast delivery and low operational complexity.
* Scaling concerns are contained in one service.
* Admin and reporting failures don't affect students mid-test.
* Limiting the risk of the database as bottleneck.

Negative / risks:

* The Test-Taker's mixed styles add operational complexity.
* If space-based is used, there is data loss risk unless replication and durability guarantees are explicitly designed (needs its own ADR).
* Service-based has limited fault tolerance and elasticity compared with the alternatives.


## Alternatives considered

### Modular Monolith

| Category               | Rating | Comments |
|------------------------|--------|----------|
| Data Integrity/Durability | `*`   | If the number of concurrent users grows we can reach the db open connection limit. This could cause timeouts and potentially data loss. Monolith means single point of loss. |
| Scalability            |     `*`   | Load is asymmetric (test taking vs admin activities). Horizontal scaling multiplies the db connection problem. Single blast radius in deployments. |
| Feasibility            | `*****`  | Optimized for simplicity and time-to-market. |

### Service-Based

| Category               | Rating | Comments |
|------------------------|--------|----------|
| Data Integrity/Durability | `***`   | Relocates the point of fragility. In case of usage a shared database The 300-connection ceiling remains. |
| Scalability            |     `***`   | The db is still a bottleneck: durable-write-before-acknowledge, idempotency and retry semantics. The parts of the system can be now scaled independently.|
| Feasibility            | `***` | More moving parts. More teams can work in parallel. Isolated app paths. Prepared for higher load trough components responsibility split. |

### Microservices

| Category               | Rating | Comments |
|------------------------|--------|----------|
| Data Integrity/Durability | `**`   | Monolithic database is not well suited here. Distributed transactions are the core durability risk. With durability logic spread across many independently deployed services. |
| Scalability            |     `***`   | The db is still a bottleneck: difficulties to isolate the DB-bound service correctly. Coordination overhead grows with service count The parts of the system can be now scaled independently.|
| Feasibility            | `**` | Even more moving parts. Team size and structure. Operational maturity requirement. Correctly identifying service boundaries.|


### Space-Based

| Category               | Rating | Comments |
|------------------------|--------|----------|
| Data Integrity/Durability | `***`   | Submitted answer is cashed before and asynchronously persisted to the database. Need for an acknowledgment-based implementation. The use of data pumps sends the data to a data writer. The approach follows the eventual consistency model. |
| Scalability            |     `*****`   | Scalability is guaranteed as the database is decoupled from the services. Domain can be partitioned. Multiple db supported: operational & reporting. Replacing direct database access with an in-memory data solutions  and processing units that scale out horizontally. |
| Feasibility            | `*` | Very complex; middle-ware, caching or data grid strategies. Extra engineering effort to harden durability: synchronous replication, write-ahead persistence |

### Conclusion
Service-based architecture is a hybrid variant of the microservices architectural style and is considered one of the most pragmatic styles available. This flexibility allows us to extract the hot path, making it the most appropriate style for this system.

## Open questions and follow-up ADRs

1. ADR for choosing the architectural style for the Test-Taker component.
1. Durability mechanism: What exactly counts as "durably stored" before acknowledgment (replication factor, durable log, or both)?
1. Consolidation: How and how often does Answer Storage write to the relational database, and how is replay handled after failures?
1. Latency: What is the acceptable end-to-end time for submitting an answer and receiving the next question?
1. Scale-out plan: How is capacity calculated and pre-provisioned per scheduled test window?
1. Security: How are answer keys and stored answers protected from students, including grading, which should never expose the answer key to the client?
1. Proctor-ended tests: How are in-flight answers handled when the proctor ends the test and students are signed out?



