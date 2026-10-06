# Stressor analysis

> In Residuality Theory, a stressor is defined as some unpredictable event that forces you to change the architecture. Case in point: the state legislature just mandated adaptive testing — the next question a student sees depends on their previous answers, so students in the same grade no longer take an identical test in an identical order. Do not design the solution. Instead, describe what in your architecture might have to change to accommodate this new requirement. What other stressors can your team think of that you didn't predict? 


## Adaptive testing - if based on multiple choice answers only 
Introduction of adaptive testing would change the system design a lot, requiring many complicated changes. In the simplest version we assume only questions with predefined answers are eligible to decide adaptive testing flow - so can be verified instantly.


### Assumptions broken:
- Test is an ordered list of questions each student takes in predefined order
  - No longer true, each student can get a different subset of the configured questions in different order
- Test Taking has no access to answer keys
  - Either Test Taking needs access to verify answer correctness, or some other service needs to be able to grade answers quickly and calculate the next question to present to the user.
- Test length and question sequence is known from the test definition
  - Test sequence can vary based on student answers and can be different for each student
- Student position in the test is determined by the number of answered questions
  - No longer true - as test sequence changes, we need to preserve full answer history.
- Final grade can be calculated as the total score from correct answers
  - As each student can get a different set of questions with varying difficulty, results are not comparable if calculated just as a sum of answer scores - as a smarter student might not even get some easier questions if the system is already confident about the level of knowledge.
  - So a more complex approach is needed that weighs the difficulty of the questions that were presented to the student and whether they were answered correctly.

### Changes required:
- Test Admin - more complex test configuration is needed - the system has to be aware of question difficulty levels and different topics they correspond to, to be able to select questions that thoroughly test student knowledge about every topic. All this has to be configurable on the Admin side
- Test Taker - needs to keep the whole path of answered questions to determine the position in the test and be able to calculate the next question + avoid showing the same question twice. Increased session state storage requirements in the component.
- Test Taker - needs to grade questions in real time - 
- Answer Grader - needs to calculate grades taking in account individual difficulty of questions student received, not just based on sum of answered questions
- System also needs to log all next question decisions for auditability purposes

### What still works:
- Async pipeline that stores answers and grades short answers is still usable - although question grading is partially already done in Test Taker 
- Authentication and proctor actions don't need to change
- Test taking UI works with no big changes required
- Sending test end marker from Test Taker to when student has completed the test

### Important notes:
- Idempotency still needs to be supported - if client retries answer submission, system still should present the same next question in response (even if random/non-deterministic choices are involved)
- Grading a question in Test Taker introduces two places that verify correctness, potentially leading to two sources of truth / inconsistencies
- Allowing Test Taker to do grading undoes one of the main security aspects the current architecture relies on - isolating the answer key from the most exposed component. 

## Adaptive testing - including short answer questions
A more complex scenario would also require short answer questions to be used in adaptive flow decisions. Compared to the multiple choice-only version, this would be much less realistic to implement than multiple-choice based adaptive testing.


### Assumptions broken:
- Short-answer grading can be async and take as long as it likes
  - If used for adaptive question selection, short answer grading has to be near-instant
  - For the massive amount of concurrent users LLM based grading might not be an achievable solution due to limited scalability + slow response times

# Other potential stressors
- Requirement to allow Student to move to previous questions and change answers
  - Breaks the immutable answer principle or delays answer submission to the end of the test (adding risk that answers may be lost)
  - Not compatible with Adaptive testing
- Requirement that student must be presented with results immediately after test / shown summary of their answers
  - Not compatible with async eventually consistent grading pipeline
  - Could be doable for the predefined answer questions
- Regulations restrict LLM usage in test grading, only human grading allowed
- Different test scheduling approach (e.g. teacher can start it on any date, not based on prescheduled times)
- Sudden LLM cost increase
- System will be used internationally and must be scalable to support arbitrary amount of users and global deployments
- Requirement to allow retroactively fixing mistake in answer key and regrading related answers
- Requirement for teachers to be able to see live progress of each student.
- Changes in privacy / data storage regulations
- Test ended by accident, need to allow resuming
- Cloud provider outage during test
- Network connectivity issues in classroom
