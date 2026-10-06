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


# Failure scenarios
## Test Taker → Answer Capturer
- The student moves to the next question only after Answer Capturer confirms that the answer is durably stored. Holding an answer in its space-based buffer does not count unless that buffer itself provides the required durability.
- Connection failure or lost acknowledgement: Test Taker keeps the answer on screen and retries with the same answer ID. If Answer Capturer already stored it, the retry returns the existing acknowledgement.
- Answer Capturer fails before durable storage: The answer has not been accepted. Test Taker retains it and retries; the student stays on the current question.
- Durable queue is unavailable: Answer Capturer can continue accepting answers while its durable buffer has capacity. Once that capacity is exhausted, it stops acknowledging new answers, applying backpressure to Test Taker.
- Answer Capturer fails after durable storage: The answer is recovered and forwarded when service resumes. A retry from Test Taker does not create another answer.

## Asynchronous choreographed flow
- Publishing fails or its acknowledgement is lost: The sender retries from its durable outbox. The next queue may receive a duplicate.
- A receiver fails before acknowledging: The queue redelivers the message. If the receiver had already committed its work, it recognizes the message ID and acknowledges it without repeating the effects.
- A receiver cannot publish the next message: It saves its result and outgoing message in one local transaction. Its outbox publishes the message after recovery.
- Processing repeatedly fails: The message moves to a dead-letter queue for investigation and replay. It remains unresolved until successfully processed.
- The end marker arrives out of order: The attempt remains incomplete until all expected distinct answers have reached their required final state.



## Critical requirements mapped to architecture characteristics that support them:
1. **No acknowledged answer is ever lost or duplicated** (data integrity/durability)
1. **The system holds up at peak concurrency** (scalability, responsiveness)
1. **Grading is consistent and every result is traceable** (auditability)
1. **System follows the principle of the least privilege when sharing data between components and users** (security)
1. **Failures recover without student-visible data loss** (reliability)
1. **System is ready in 6 months** (feasibility)



## Fitness functions to verify capabilities
### No acknowledged answer is ever lost or duplicated _(data integrity/durability)_
1. Every code path acknowledges success only after the data is stored in a durable queue/storage, following the Async Chain of Persistence pattern. Storage configuration guarantees redundancy + durability
Results database schema has a constraint that prevents duplicate records for the same (test_id + student_id + question_id) key.
1. Consistency checks - during test runs every unique ACK-ed answer added to the queue must end up processed and graded in the result database, unless it ended up in DLQ.
1. Out-of-order handling - test queue events arriving out of order, including duplicate messages, the final result still must be correctly assembled.
1. Chaos engineering - randomly kill components during load test, ensure consistency checks still pass, even if crash happens right after recorded answer.

### The system holds up at peak concurrency _(scalability, responsiveness)_
1. Capacity provisioning - from the number of scheduled tests+questions+users calculate memory and storage requirements to hold all the answers. Validate that Test Takers allocated memory, queue storage limits and database capacity will be able to accommodate all the data.
1. Concurrency - perform a load test that simulates system load that exceeds the expected simultaneous users. Ensure that system response time stays in allowed limits during peak load. Measure how increased load affects performance to ensure provisioned capacity requirements are estimated correctly.
1. 300 DB connection limit guard - verify in the app configuration that configured connection pool size * max deployed processing units of all components that access DB does not exceed the allowed limit (leaving some extra headroom of spare capacity).
1. DB free test taking component - app has no DB credentials, no access via firewall, no code using DB. All data has to be pre-loaded from outside. Also applies to other external calls - so ideally it must run in a secured environment preventing any network access besides the expected connections - prevents unwanted external calls by design that may become a performance bottleneck.

### Grading is consistent and every result is traceable _(auditability)_
1. Verify LLM grading results when receiving identical answers are cached and reused, even if both answers are processed in parallel. Measure that only one of such calls was actually processed by LLM.
1. Verify that every graded answer has traceable answer recording and grading log entries. 

### System follows the principle of the least privilege when sharing data between components and users _(security)_
1. Verify that configuration prevents network connectivity between components not supposed to have one
1. Verify that access is limited to read/write only as necessary
1. Verify that user access scope match their role and access to restricted data is not allowed
1. Ensure data structures passed between system services expose only data needed by the other side, as defined by the contract. 

### Failures recover without student-visible data loss _(reliability)_
1. Turn off components to verify that parts that are supposed to work independently still function.
1. Stop queue processing, allow data to accumulate. Verify that the system is still stable during downtime and after resuming everything is fully processed.

### System is ready in 6 months _(feasibility)_ 
- Mostly a project management concern, that affects architecture choices, but is not measurable by architecture fitness functions
