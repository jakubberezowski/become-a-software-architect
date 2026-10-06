# Transaction strategy

Answer collection will be designed following **Async Chain of Persistence** pattern principles:

- Message cannot be removed from storage until it has been durably stored by the next step.
- Message delivery will be implemented using queues with at-least-once delivery principle - a message that was not acknowledged will be redelivered eventually or end up in a dead letter queue if it can't be processed.
- Message processing must be idempotent, to support possible redeliveries. Repeated delivery should not cause any side-effects compared to being delivered just once.
- The final state is eventually consistent - no unprocessed messages upstream means that everything has been processed.

## Async Chain of Persistence message delivery flow

1. **Message sender** - inserts message to queue.
2. **Queue** - acknowledges message has been stored durably (with redundancy).
3. **Message sender** - reports success upstream.
4. **Message receiver** - fetches message from queue.
5. **Message receiver** - processes message, ensures transaction completes, data is stored now.
6. **Message receiver** - acknowledges message was processed.
7. **Queue** - removes the message completely.

Failure handling:

- If acknowledgement is not received in the expected timeframe, the message reappears in the queue.
- After the maximum number of failed attempts, it is delivered to the dead letter queue for investigation.

## Relation to the Anthology Saga pattern

This approach matches the Anthology Saga pattern characteristics:

- Eventual consistency
- Async communication
- Choreographed coordination

However, due to the additive nature and idempotent operations of the grading pipeline, there is no need for distributed rollbacks / compensating updates in the system design. It doesn't have any parallel operations where failure of one would require rolling back the other - just a sequence of operations where each step makes the workflow more complete, and can be repeated with no side effects.

## System segments that can use queue-based async communication

- Test taking service → Grading service - non-graded answer collection
- Grading service → LLM Grader - non-graded short-answer question answers
- LLM Grader → Grading service - completed LLM graded short answers
- Grading service → Results Steward - graded answers

## Benefits of using this approach

- **Individual scalability of deployment units** - each can operate at its optimal speed, without affecting others.
- **Fault tolerance** - failures in one component can't affect others (as long as the queues are available).
- **Security** - no direct data sharing, only allowed messages are exchanged between services.

## Challenges

### Queue capacity provisioning

- The entry point queue (the one receiving answers from Test Taking service) must provide enough capacity to accommodate all answers even if nothing processes them downstream. The requirements are predictable as tests are pre-scheduled and the number of questions/users is known in advance.
- Other queues process data at a controlled pace, so they may have limited capacity as the processing speed can be adjusted if necessary.

### Figuring out when all answers have been delivered for a particular test/user

- Can be addressed by including a "user has ended test with answer count x" message when the test ends. This allows verifying that all answers were retrieved by comparing the total received answer count with the amount stated in the marker.
- **NB:** the end marker can be delivered out of order like any other message, so receiving the marker doesn't mean we are in a consistent state while messages still continue to arrive. This can be addressed by verifying completion after each received answer, not just the end marker.

### LLM grading consistency

- LLM is non-deterministic, but test grading should not be - identical answers must always yield the same grade. This ensures fair test grading, lowers LLM usage costs and also