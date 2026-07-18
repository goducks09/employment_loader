# Case Study: Redesigning a Legacy Enterprise Integration

## Project Overview
This application serves as a mission-critical integration bridge between a core Timekeeping application and Oracle PeopleSoft. It automates a data pipeline for employment contracts and directly manages employee lifecycle updates within PeopleSoft. Upon completion, the application generates a detailed Excel status report and emails it to human resources.

* **Role:** Solo Software Engineer
* **Tech Stack:** Python, `pytest`, PeopleSoft Component Interfaces (CI), REST APIs, Excel/Email generation libraries
* **Scale:** Processing hundreds of automated employment offers, end dates, and short work breaks every semester.
* ***Note:** The source code for this project is owned by my employer, so I am unable to share it publicly.*

---

## The Challenge: Unreliable Legacy Code
The university previously relied on an automated employment loader that, while functionally complex, had become brittle, difficult to maintain, and prone to edge-case bugs. 

The system wasn't flexible enough to easily absorb new business requirements. Additionally, critical operational updates, such as automatically setting job end dates, were entirely out of scope for the old system, requiring heavy manual overhead from staff. 

The objective shifted from a simple patch job to an **architectural redesign**: replacing the current system with an extensible, deterministic, and highly testable integration engine.

---

## Technical Architecture: A Layered, Domain-Driven CLI
To ensure the new application could seamlessly scale to handle different employee types and operation types without decaying into spaghetti code, I implemented a modular, layered architecture with a strict separation of concerns.

~~~
[Entry Point] ──> [Command Layer] ──> [Service Layer] ──> [Adapter Layer]
                                                     │                     │
                                             (Business Logic)      (PeopleSoft / APIs)
~~~

### 1. Decoupled Architectural Layers
*   **Entry Point:** Handles CLI argument parsing using `argparse` and bootstraps the application's configuration. It evaluates environment and runtime flags to determine configuration states.
*   **Command Layer:** Orchestrates the specific operations. It maps CLI subcommands to dedicated command classes, decoupling user input from core execution.
*   **Service Layer:** The isolated brain of the application. It contains the business rules and determines *who* gets processed, completely separate from infrastructure details.
*   **Data Models:** Encapsulates PeopleSoft Component Interface (CI) schemas, transforming raw incoming JSON payloads into the exact dictionary schemas required by the database.
*   **Adapter Layer:** Manages raw external network transport, enclosing the REST API communications and underlying PeopleSoft CI connections.

---

## Implementing Design Patterns for Extensibility
To fulfill the requirement for maximum flexibility, I leveraged several design patterns that make adding future business features trivial.

### Open/Closed Principle
The entire architecture is open for extension but closed for modification. Adding a completely new process (such as a hypothetical `Termination` action) requires zero changes to the core execution or orchestration files. A developer simply creates a new `Service` class, a new `Command` class, and registers the subcommand.

*   **Template Method Pattern:** Established an abstract base class for employment actions. This allowed the core synchronization workflow to be written once, while concrete subclasses simply provide the specialized data payload overrides required for different employee types.
*   **Strategy Pattern & Python Protocols:** Defined expected API and PeopleSoft client behaviors using Python Protocols. Based on CLI arguments and configuration variables, the application can dynamically swap operational behaviors, such as safely switching to simulated mock clients for testing.
*   **Command & Factory Patterns:** A factory function analyzes CLI input and dynamically instantiates the correct `Command` object, ensuring clean decoupling between the user interface and the backend business logic.

---

## Overcoming Key Engineering Hurdles

### 1. Translating Complex Business Processes
The hardest phase of development was translating ambiguous, complex business rules into predictable code. I had to account for countless edge cases determining exactly who should be updated and, more importantly, **who must be left entirely alone**. 

I solved this in two ways. First, by writing Oracle SQL queries to pull the initial pool of potential records from the database. Second, by implementing strict validation checks inside the **Service Layer**. For example, the system queries updatable records and explicitly verifies if an employee has conflicting open timesheets *before* allowing a PeopleSoft commit to occur.

### 2. Fault Tolerance and Graceful Degradation
Enterprise integrations cannot afford to crash halfway through a batch of hundreds of records because a single row contains bad data. 

I implemented robust exception handling to manage partial failures. If a record triggers a PeopleSoft validation error partway through a run, the application catches the exception, logs the failure details, flags it explicitly for human review in the final Excel report, and seamlessly proceeds to process the next record.

### 3. Bulletproof Testing with `pytest`
Because database connections and API clients are instantiated at the application root and passed down via **Dependency Injection**, the core business logic is completely decoupled from the network. This allowed me to build a comprehensive unit testing suite using `pytest`, ensuring that complex validation algorithms could be verified safely using mock clients rather than live production data.

---

## Results & Impact
*   **Reduced Operational Overhead:** Eliminated hours of manual data entry by expanding automation to include critical end-dating and short work break workflows.
*   **Enterprise Reliability:** Replaced a legacy, hard-to-maintain codebase with an extensible architecture backed by a predictable automated testing suite.
*   **Human-in-the-Loop Visibility:** Provided HR teams with immediate, actionable feedback on every run via an automated, clear Excel summary report.
