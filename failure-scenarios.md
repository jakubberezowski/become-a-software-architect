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