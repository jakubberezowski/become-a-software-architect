# Fitness functions

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
