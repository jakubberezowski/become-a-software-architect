# Make the Grade

## Architectural Characteristics

Driving characteristics:
- Data Integrity/Durability
- Security
- Feasibility
  
Supporting characteristics:
- Scalability
- Reliability
- Responsiveness
- Auditability

More details - see [Architectural Characteristics](architectural-characteristics.md).

## Architecture Style

We will use a service-based architecture with domain services, with these boundaries:

- **Admin domain**: Roster maintenance, test and answer-key maintenance, and scheduling. It is low-load and stays plain service-based.
- **Reporting domain**: Generates student, teacher, school, and question-validation reports after testing. It stays service-based and reads from the consolidated database.
- **Test-Taker domain**: Presents questions, captures answers, and durably stores them. It is designed as an independently deployable, scalable, isolated unit so its internal style can differ from the others. Given the scale, likely will be implemented in a manner inspired from service-based architecture to avoid DB reads and writes during massively parallel operations.

More details - [Architecture Style ADR](architecture-style-adr.md).

## Architecture

### System Context Diagram C1
By applying the C4 approach to visualize our software architecture, we were able to clearly illustrate the dependencies between different components of our application and highlight their relationships.
We will primarily focus on the C2 views to provide an overarching overview.

[//]: # (See [System Context Diagram]&#40;c4-context.excalidraw.png&#41;)
![System Context Diagram](c4-context.excalidraw.png)

### Container Diagram C2
![Container Diagram](c4-container.excalidraw.png)

### Answer Capturer Physical

![Answer Capturer Physical](c4-answer-capturer.excalidraw.png)

## Transaction Strategy
See [Transaction Strategy](transaction-strategy.md).

## Failure scenarios
See [Failure Scenarios](failure-scenarios.md).

## Fitness Functions
See [Fitness Functions](fitness-functions.md).

## Stressors
See [Stressors](stressors.md).
