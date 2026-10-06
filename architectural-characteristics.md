# Architecture Characteristics

## Data Integrity/Durability

**Explicit:** stated in requirements that data loss is unacceptable.

### Considerations

- 200k students and 300 available db connections at the time is a core mismatch.
- ~667 students for every 1 available connection at time.
- The act of capturing an answer has to be decoupled from the act of committing it to that database.
- The component sitting in between has to guarantee the durability on its own.
- We can scale the student-facing tier as wide as needed, but not the db connection pool.

### How to measure?

- Every submitted answer is durably captured.
- Modification are prevented
- Opened DB connections
- INSERT query duration
- Approximate age of an answer before persisting the answers.

### Fitness Function

- The DB never sees more than 300 concurrent connections.
- All answers are captured.

---

## Security

**Explicit** - requirements state that it is critical to protect data from unauthorized access.

### Considerations

- Implies other characteristics: confidentiality, integrity, non-repudiation, accountability, authenticity
- General security hygiene:
  - Data encryption at rest and in transit, in- out-put validation, etc.
- Keep evidence that a user actively approved and performed an action.
- Separate test, reporting, teacher application/data.
- Network segmentation.
- In-Depth Authentication and Authorization.
- Legal and policies.

### How to measure

- Static code analysis
- Pen tests.

---

## Feasibility

**Implicit** - requirements state a strict deadline when the system has to be ready, and feasibility prioritizes meeting this goal

### Considerations

- Strictly following the development roadmap, prioritizing non-negotiable requirements over minor improvements.
- Architecture must favor simplicity, potentially sacrificing extensibility and agility if development continues in the future - we need to focus on predetermined reachable goals, not prepare for an unknown future.

---

## Scalability

**Explicit:** Required user amount is clearly stated in requirements. As tests are scheduled, required scale is predictable.

### Scenario: Test answers persistence

#### Considerations

- Support a large student base: 800k+ registered students
- Up to 200k logged-in users at time.
- Risk is significant degradation in performance or reliability.
- Limited by 300 db connections.
- High concurrency on the student-facing tier.
- Low concurrency challenges on the persistence layer.

#### How to measure

- Throughput of answers
- Resource utilization
- Concurrent users
- Horizontal scaling efficiency

### Scenario: Admin activities

#### Considerations

- Two people are responsible for touching 800,000+ student records.
- Across roster maintenance.
- Reports creation.
- High-volume batch/aggregate operations.
- Bulk mode operations.

#### How to measure

- Data volume.
- Data throughput.

### Fitness Function

- Latency and system load stays under a defined bound.

### Additional links

- [DORA metrics](https://dora.dev/guides/dora-metrics/)

---

## Reliability

**Implicit:** with 200k simultaneously active users it is critical that the system is stable and reachable.

### Considerations

- The system must be fail-safe and perform its intended function correctly while handling a very high amount of concurrent users during a test run.
- Maintenance or reporting tasks performed by the admins should not slow down live application.
- Physical architecture components with redundancy.
- Additional infrastructure costs and more complex implementation

### How to measure

- Data loss rate.
- Response/latency time for submitting answers.
- Logged-in students.

### Fitness Function

- Data loss rate is 0.
- Expected response/latency time.

---

## Responsiveness

**Explicit** characteristic.

### Considerations

- The test time is limited and measured by the proctor.
- Rendering a page with the question and submitting answers must be fast.
- A fairness and legal problem.

### How to measure

- Page load time.
- Response/latency time for submitting answers.

### Fitness Functions

- Asserting max page load time.
- Asserting max response time of submitting answers.

---

## Auditability

**Implicit:** closely related to data integrity/security.

### Considerations

- To support data integrity and security, all changes in data must be trackable - who, when, how made the changes.
- Metrics are measured and persisted for internal and external audits.
- Increased development complexity, limits performance.

### How to measure

- Defined metrics are recorded.
- Data points are delivered with expected resolution.