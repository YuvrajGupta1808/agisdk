# Requirements Document

## Introduction

This document specifies requirements for an adaptive web automation agent system that uses plan-based execution with dynamic error recovery. The system uses a two-model architecture where both models work in parallel: Qwen VLM analyzes screenshots to identify page elements with coordinates, while GPT-4o analyzes the task goal and maintains an execution plan. GPT-4o acts as the central decision-maker, sending straightforward element location requests to Qwen only when there is a concrete plan to use them. The system achieves performance improvements through action batching while maintaining robustness through dynamic plan revision on errors.

## Glossary

- **Agent System**: The web automation system that executes tasks on web pages
- **Vision Model**: Qwen VLM used for analyzing screenshots and identifying page elements with coordinates
- **Orchestrator Model**: GPT-4o that analyzes task goals, maintains execution plans, and makes all decisions
- **Execution Plan**: Dynamic list of future actions maintained by the Orchestrator Model
- **Page Analysis**: Structured description of all page elements with types, labels, and coordinates from Vision Model
- **Action Batch**: Sequence of actions from the plan executed together without intermediate screenshots
- **Tool Call**: Direct action execution request from Orchestrator via llm_call_step tool (click, type, select, etc.)
- **Info Seeking Query**: Straightforward element location request from Orchestrator to Vision Model (e.g., "find the create button")
- **llm_call_step Tool**: The tool that GPT-4o calls to execute actions and handle execution context
- **Plan Revision**: Process of updating the execution plan when errors occur or plan is empty
- **Element Location Failure**: When Vision Model cannot locate a requested element and reports failure to Orchestrator

## Requirements

### Requirement 1

**User Story:** As a system architect, I want parallel analysis of screenshots and goals, so that both vision understanding and strategic planning happen simultaneously without blocking each other.

#### Acceptance Criteria

1. WHEN a screenshot is captured THEN the Agent System SHALL invoke the Vision Model and Orchestrator Model in parallel
2. WHEN the Vision Model receives a screenshot THEN the system SHALL analyze all page elements and produce a Page Analysis containing element types, labels, and coordinates
3. WHEN the Orchestrator Model receives a task goal THEN the system SHALL analyze the goal and create or update an Execution Plan
4. THE Agent System SHALL not block Orchestrator Model execution while waiting for Vision Model results
5. WHEN both models complete their parallel analysis THEN the Agent System SHALL combine their outputs for decision making

### Requirement 2

**User Story:** As a task orchestrator, I want GPT-4o to maintain a dynamic execution plan, so that the system can adapt to changing conditions and errors.

#### Acceptance Criteria

1. WHEN the Orchestrator Model receives a task goal THEN the system SHALL create an initial Execution Plan
2. WHEN the Execution Plan is empty THEN the Orchestrator Model SHALL generate a new plan
3. WHEN an action execution fails THEN the Orchestrator Model SHALL revise the Execution Plan
4. WHEN the Orchestrator Model revises a plan THEN the system SHALL incorporate error information and Vision Model feedback
5. THE Execution Plan SHALL contain specific, actionable steps with element identifiers

### Requirement 3

**User Story:** As a vision assistant, I want Qwen VLM to provide detailed page analysis and respond to straightforward element location requests, so that the orchestrator can locate elements when needed.

#### Acceptance Criteria

1. WHEN the Vision Model analyzes a screenshot THEN the system SHALL identify all interactive elements with their coordinates
2. WHEN the Orchestrator Model sends a straightforward element location request (e.g., "find the create button") THEN the Vision Model SHALL provide specific coordinates for that element
3. WHEN the Vision Model cannot locate a requested element THEN the system SHALL report the Element Location Failure to the Orchestrator Model
4. THE Vision Model SHALL identify element types (button, input, select, link, etc.) for all page elements
5. WHEN form fields are present THEN the Vision Model SHALL distinguish between text inputs, dropdowns, date pickers, and other field types

### Requirement 4

**User Story:** As an action executor, I want to batch multiple actions from the plan, so that form filling and multi-step operations complete faster.

#### Acceptance Criteria

1. WHEN the Execution Plan contains multiple same-page actions THEN the Agent System SHALL execute them as an Action Batch
2. WHEN executing an Action Batch THEN the system SHALL not capture intermediate screenshots
3. WHEN an action in a batch fails THEN the Agent System SHALL stop batch execution and capture a screenshot
4. WHEN a batch completes successfully THEN the Agent System SHALL capture a screenshot only if navigation occurred
5. THE Agent System SHALL execute actions sequentially within a batch

### Requirement 5

**User Story:** As an orchestrator, I want to send two types of queries, so that I can both execute actions directly and request element locations when needed.

#### Acceptance Criteria

1. WHEN the Orchestrator Model generates output THEN the system SHALL classify it as either Tool Call or Info Seeking Query
2. WHEN a Tool Call is issued THEN the Orchestrator Model SHALL call the llm_call_step tool to execute the action (click, type, select, etc.)
3. WHEN an Info Seeking Query is issued THEN the Agent System SHALL send the straightforward element location request to the Vision Model
4. THE Orchestrator Model SHALL only send Info Seeking Queries when there is a concrete plan to use the element location
5. WHEN the Vision Model responds to an Info Seeking Query THEN the system SHALL provide the coordinates or failure message to the Orchestrator Model
6. WHEN an Info Seeking Query fails THEN the Orchestrator Model SHALL receive the failure notification and revise the plan accordingly
7. THE llm_call_step tool SHALL handle action execution and logging within the same execution context

### Requirement 6

**User Story:** As an error handler, I want the system to gracefully handle execution failures, so that tasks can recover and continue.

#### Acceptance Criteria

1. WHEN an action execution fails THEN the Agent System SHALL capture a screenshot
2. WHEN an execution error occurs THEN the system SHALL provide error details to the Orchestrator Model
3. WHEN the Orchestrator Model receives error information THEN the system SHALL revise the Execution Plan
4. WHEN a field type mismatch occurs (e.g., typing into dropdown) THEN the Orchestrator Model SHALL update the plan with the correct action type
5. THE Agent System SHALL allow up to 3 consecutive plan revisions before failing the task

### Requirement 7

**User Story:** As a performance optimizer, I want the system to minimize API calls, so that execution is fast and cost-effective.

#### Acceptance Criteria

1. WHEN a screenshot is captured THEN the Vision Model SHALL be invoked once per screenshot
2. WHEN the Execution Plan contains valid actions THEN the Orchestrator Model SHALL not be invoked until the plan is empty or errors occur
3. WHEN multiple same-page actions exist THEN the Agent System SHALL batch them to reduce screenshot captures
4. THE Agent System SHALL not invoke the Vision Model between actions in a batch
5. WHEN the Orchestrator Model has sufficient information THEN the system SHALL not request additional Vision Model analysis

### Requirement 8

**User Story:** As a task executor, I want the system to track execution metrics, so that performance improvements can be measured.

#### Acceptance Criteria

1. WHEN a task executes THEN the Agent System SHALL record the number of screenshots captured
2. WHEN a task executes THEN the Agent System SHALL record the number of Vision Model invocations
3. WHEN a task executes THEN the Agent System SHALL record the number of Orchestrator Model invocations
4. WHEN a task executes THEN the Agent System SHALL record the number of plan revisions
5. WHEN a task completes THEN the Agent System SHALL provide a metrics summary including all recorded values

### Requirement 9

**User Story:** As a system integrator, I want backward compatibility with existing infrastructure, so that the new agent works with current benchmarks.

#### Acceptance Criteria

1. WHEN the Agent System receives a task definition THEN the system SHALL parse goals and URLs in the existing format
2. WHEN executing tasks THEN the Agent System SHALL use existing browser automation primitives
3. WHEN task execution completes THEN the Agent System SHALL produce results in the existing ExperimentResult format
4. THE Agent System SHALL integrate with existing TaskExecution workflow
5. WHEN running benchmark tasks THEN the system SHALL execute without requiring benchmark modifications

### Requirement 10

**User Story:** As a form automation specialist, I want intelligent handling of different field types, so that forms are filled correctly regardless of complexity.

#### Acceptance Criteria

1. WHEN the Vision Model identifies a text input THEN the system SHALL use type action
2. WHEN the Vision Model identifies a dropdown THEN the system SHALL use select action
3. WHEN the Vision Model identifies a date picker THEN the system SHALL use appropriate date input action
4. WHEN an action fails due to field type mismatch THEN the Orchestrator Model SHALL revise the plan with the correct action type
5. THE Execution Plan SHALL specify both the action type and target element for each step

### Requirement 11

**User Story:** As a planning strategist, I want the first turn to be observation-only, so that the agent can see what's actually on the page before creating complex multi-step plans.

#### Acceptance Criteria

1. WHEN the Agent System starts a task with no prior page observations THEN the Orchestrator Model SHALL create a plan with at most one exploratory action
2. WHEN creating the initial Execution Plan THEN the system SHALL limit turn 1 to navigation or simple observation actions
3. WHEN the Page Analysis is available from turn 1 THEN the Orchestrator Model SHALL create detailed multi-step plans in subsequent turns
4. THE Orchestrator Model SHALL not assume knowledge of page structure before observing it
5. WHEN generating plans for turn 1 THEN the system SHALL prioritize gathering information over executing complex workflows
