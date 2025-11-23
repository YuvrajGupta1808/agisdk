# Design Document

## Overview

This design implements an adaptive web automation agent using a plan-based execution model with dynamic error recovery. The system uses two models working in parallel:

1. **Qwen VLM**: Analyzes screenshots to identify all page elements with their types, labels, and coordinates
2. **GPT-4o**: Analyzes the task goal, maintains a dynamic execution plan, and makes all decisions

GPT-4o acts as the central decision-maker and sends two types of queries:
- **Tool Calls**: Direct action execution via llm_call_step tool (click, type, select, etc.)
- **Info Seeking Queries**: Straightforward element location requests to Qwen (e.g., "find the create button") - sent ONLY when there's a concrete plan to use the element

When Qwen cannot locate a requested element, it reports the failure back to GPT-4o, which then revises the execution plan. This approach combines the benefits of action batching (performance) with adaptive planning (robustness).

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Agent Step Workflow                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. Capture Screenshot                                  │ │
│  │  2. Parallel Analysis:                                  │ │
│  │     - Qwen VLM: Analyze screenshot → Page Analysis     │ │
│  │     - GPT-4o: Analyze goal → Update/Create Plan        │ │
│  │  3. GPT-4o Decision Making:                            │ │
│  │     - Tool Call → Execute via llm_call_step            │ │
│  │     - Info Query → Ask Qwen for element location       │ │
│  │  4. Execute Action Batch from Plan                      │ │
│  │  5. If Error or Qwen Fails → GPT-4o Revises Plan      │ │
│  │  6. If Plan Empty → GPT-4o Creates New Plan            │ │
│  │  7. Repeat until task complete                          │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    Parallel Analysis Phase                    │
│                                                               │
│  ┌──────────────┐                    ┌──────────────┐       │
│  │  Qwen VLM    │                    │   GPT-4o     │       │
│  │  (Vision)    │                    │(Orchestrator)│       │
│  └──────┬───────┘                    └──────┬───────┘       │
│         │                                   │                │
│         │ Analyzes Screenshot               │ Analyzes Goal  │
│         ▼                                   ▼                │
│  Page Analysis                       Execution Plan          │
│  - All elements                      - Next actions          │
│  - Types & labels                    - Batching strategy     │
│  - Coordinates                       - Decision logic        │
│         │                                   │                │
│         └───────────┬───────────────────────┘                │
│                     │                                        │
└─────────────────────┼────────────────────────────────────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │  GPT-4o Query Router   │
         │  - Tool Call?          │
         │  - Info Seeking Query? │
         └────────┬───────────────┘
                  │
         ┌────────┴────────┐
         │                 │
         ▼                 ▼
  ┌─────────────┐   ┌─────────────┐
  │llm_call_step│   │ Ask Qwen    │
  │   (execute) │   │ "find X"    │
  └──────┬──────┘   └──────┬──────┘
         │                 │
         │                 ▼
         │          ┌─────────────┐
         │          │Coords or    │
         │          │Failure      │
         │          └──────┬──────┘
         │                 │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Action Executor │
         │  (with batching)│
         └─────────────────┘
                  │
                  ▼
         ┌─────────────────┐
         │  Browser/Page   │
         └─────────────────┘
```

### Execution Flow

1. **Screenshot Capture**: Take screenshot of current page state

2. **Parallel Analysis**:
   - Qwen VLM analyzes screenshot → produces PageAnalysis with all elements and coordinates
   - GPT-4o analyzes task goal → updates or creates Execution Plan

3. **GPT-4o Decision Making**:
   - GPT-4o examines the plan and decides on next action
   - Two query types:
     - **Tool Call**: Calls llm_call_step to execute action directly
     - **Info Seeking Query**: Sends straightforward request to Qwen (e.g., "find the submit button") ONLY when there's a concrete plan to use it

4. **Info Seeking Query Handling** (if applicable):
   - Qwen receives straightforward element location request
   - Qwen returns coordinates OR reports failure
   - If failure → GPT-4o receives failure notification and revises plan

5. **Action Batch Execution**:
   - Extract next batch of same-page actions from plan
   - Execute actions sequentially via llm_call_step without intermediate screenshots
   - If error occurs → stop batch, capture screenshot, report to GPT-4o

6. **Plan Revision**:
   - On error or Qwen failure → GPT-4o revises plan based on error/failure details
   - On success → continue with remaining plan actions

7. **Repeat** until task is complete

## Components and Interfaces

### 1. AdaptiveAgent

The main agent class implementing the adaptive, plan-based workflow.

```python
class AdaptiveAgent(BaseAgent):
    """
    Adaptive agent using plan-based execution with dynamic error recovery.
    
    Architecture:
    - Qwen VLM and GPT-4o work in parallel
    - GPT-4o is the central decision-maker
    - Two query types: Tool Calls (via llm_call_step) and Info Seeking Queries
    - Qwen reports failures back to GPT-4o for plan revision
    """
    
    def __init__(
        self,
        vision_model: QwenVisionModel,
        orchestrator_model: GPT4Orchestrator,
        metrics_tracker: MetricsTracker
    ):
        self.vision_model = vision_model
        self.orchestrator_model = orchestrator_model
        self.metrics_tracker = metrics_tracker
        self.current_plan: ExecutionPlan = ExecutionPlan(actions=[])
    
    async def step(self, browser: AgentBrowser, state: AgentState) -> AgentState:
        """
        Execute one agent step using adaptive planning.
        
        Workflow:
        1. Capture screenshot
        2. Parallel Analysis:
           - Qwen analyzes screenshot → PageAnalysis
           - GPT-4o analyzes goal → ExecutionPlan
        3. GPT-4o Decision:
           - Tool Call → Execute via llm_call_step
           - Info Seeking Query → Ask Qwen for element location
        4. Handle Info Seeking Query (if applicable):
           - Qwen returns coordinates OR failure
           - If failure → GPT-4o revises plan
        5. Execute action batch from plan
        6. Handle errors by revising plan
        7. Return updated state
        """
        pass
```

### 2. QwenVisionModel

Vision model for page analysis and responding to element location requests.

```python
class QwenVisionModel:
    """
    Qwen VLM for visual page analysis and element location.
    Analyzes screenshots in parallel with GPT-4o's goal analysis.
    """
    
    def __init__(self, model: str, client: AsyncOpenAI):
        self.model = model
        self.client = client
    
    async def analyze_page(
        self, 
        screenshot: bytes,
        history: List[Dict[str, Any]]
    ) -> PageAnalysis:
        """
        Analyze screenshot and identify ALL interactive elements.
        This runs in parallel with GPT-4o's goal analysis.
        
        Args:
            screenshot: Screenshot bytes
            history: Conversation history for context
            
        Returns:
            PageAnalysis with ALL elements, types, labels, and coordinates
        """
        pass
    
    async def locate_element(
        self,
        screenshot: bytes,
        element_description: str,
        page_analysis: PageAnalysis
    ) -> Union[ElementLocation, ElementLocationFailure]:
        """
        Respond to straightforward element location request from GPT-4o.
        Called ONLY when GPT-4o has a concrete plan to use the element.
        
        Args:
            screenshot: Current screenshot
            element_description: Straightforward request (e.g., "find the create button")
            page_analysis: Current page analysis
            
        Returns:
            ElementLocation with coordinates, or ElementLocationFailure if not found
        """
        pass
```

### 3. GPT4Orchestrator

Orchestrator model that analyzes goals, maintains execution plans, and makes all decisions.

```python
class GPT4Orchestrator:
    """
    GPT-4o orchestrator that analyzes task goals and maintains dynamic execution plans.
    Acts as the central decision-maker, sending two types of queries:
    - Tool Calls: Execute actions via llm_call_step
    - Info Seeking Queries: Request element locations from Qwen
    """
    
    def __init__(self, model: str, client: AsyncOpenAI):
        self.model = model
        self.client = client
    
    async def analyze_and_plan(
        self,
        goal: str,
        current_plan: ExecutionPlan,
        page_analysis: PageAnalysis,
        error_info: Optional[ExecutionError] = None,
        location_failure: Optional[ElementLocationFailure] = None
    ) -> ExecutionPlan:
        """
        Analyze task goal and create/update execution plan.
        This runs in parallel with Qwen's screenshot analysis.
        
        Args:
            goal: Task goal to analyze
            current_plan: Current execution plan (may be empty)
            page_analysis: Latest page analysis from Qwen
            error_info: Error information if last action failed
            location_failure: Failure info if Qwen couldn't locate element
            
        Returns:
            Updated ExecutionPlan with actions to execute
        """
        pass
    
    def parse_output(self, response: str) -> Union[ToolCall, InfoSeekingQuery]:
        """
        Parse orchestrator output into tool calls or info seeking queries.
        
        Tool Call: Direct action execution via llm_call_step
        Info Seeking Query: Straightforward element location request to Qwen
                           (sent ONLY when there's a concrete plan to use it)
        
        Args:
            response: Raw response from GPT-4o
            
        Returns:
            Either ToolCall (execute via llm_call_step) or InfoSeekingQuery (ask Qwen)
        """
        pass
```

### 4. ActionBatchExecutor

Executes batches of actions from the plan.

```python
class ActionBatchExecutor:
    """
    Executes action batches with error handling.
    """
    
    def __init__(self, browser: AgentBrowser):
        self.browser = browser
    
    async def execute_batch(
        self,
        actions: List[Action],
        page_analysis: PageAnalysis
    ) -> ExecutionResult:
        """
        Execute a batch of actions sequentially.
        
        Args:
            actions: List of actions to execute
            page_analysis: Current page analysis for element lookup
            
        Returns:
            ExecutionResult with success status and error details if failed
        """
        pass
    
    def create_batch(self, plan: ExecutionPlan) -> List[Action]:
        """
        Extract next batch of same-page actions from plan.
        
        Args:
            plan: Current execution plan
            
        Returns:
            List of actions that can be batched together
        """
        pass
```

### 5. MetricsTracker

Tracks execution metrics for performance analysis.

```python
class MetricsTracker:
    """
    Tracks execution metrics.
    """
    
    def __init__(self):
        self.screenshot_count = 0
        self.vision_calls = 0
        self.orchestrator_calls = 0
        self.plan_revisions = 0
        self.start_time = None
        self.end_time = None
    
    def record_screenshot(self) -> None:
        """Record screenshot capture."""
        self.screenshot_count += 1
    
    def record_vision_call(self) -> None:
        """Record vision model invocation."""
        self.vision_calls += 1
    
    def record_orchestrator_call(self) -> None:
        """Record orchestrator model invocation."""
        self.orchestrator_calls += 1
    
    def record_plan_revision(self) -> None:
        """Record plan revision."""
        self.plan_revisions += 1
    
    def get_summary(self) -> MetricsSummary:
        """Get metrics summary."""
        pass
```

## Data Models

### PageAnalysis

Structured page analysis from Qwen VLM.

```python
@dataclass
class PageElement:
    """Interactive element on the page."""
    element_id: str  # Unique identifier
    element_type: str  # "button", "input", "select", "link", etc.
    label: str  # Visible text or aria-label
    coordinates: Tuple[int, int]  # (x, y) center coordinates
    field_type: Optional[str]  # For inputs: "text", "email", "password", etc.
    attributes: Dict[str, str]  # Additional attributes


@dataclass
class PageAnalysis:
    """Complete page analysis from vision model."""
    elements: List[PageElement]
    page_type: str  # "form", "search", "content", "navigation"
    content_summary: str  # Brief summary of page content
    timestamp: float
```

### ExecutionPlan

Dynamic plan maintained by orchestrator.

```python
@dataclass
class Action:
    """Single action in execution plan."""
    action_type: str  # "click", "type", "select", "scroll", "wait"
    target_element_id: str  # ID of target element from PageAnalysis
    parameters: Dict[str, Any]  # Action-specific parameters
    is_navigation: bool  # Whether action causes navigation
    description: str  # Human-readable description


@dataclass
class ExecutionPlan:
    """Dynamic execution plan."""
    actions: List[Action]
    reasoning: str  # Orchestrator's reasoning for this plan
    created_at: float
    revision_count: int = 0
```

### ExecutionResult

Result of action batch execution.

```python
@dataclass
class ExecutionError:
    """Error information from failed execution."""
    action: Action
    error_type: str  # "element_not_found", "type_mismatch", "timeout", etc.
    error_message: str
    screenshot_path: Optional[str]


@dataclass
class ExecutionResult:
    """Result of executing action batch."""
    success: bool
    actions_completed: int
    error: Optional[ExecutionError]
    navigation_occurred: bool
```

### Query Types

```python
@dataclass
class ToolCall:
    """
    Direct action execution request via llm_call_step tool.
    GPT-4o calls this to execute actions directly.
    """
    tool_name: str  # "llm_call_step"
    action_type: str  # "click", "type", "select", etc.
    parameters: Dict[str, Any]  # Action-specific parameters


@dataclass
class InfoSeekingQuery:
    """
    Straightforward element location request to Qwen.
    Sent ONLY when GPT-4o has a concrete plan to use the element.
    """
    element_description: str  # e.g., "find the create button", "locate search input"
    context: str  # Why this information is needed (for logging)


@dataclass
class ElementLocation:
    """Successful element location response from Qwen."""
    element_id: str
    coordinates: Tuple[int, int]
    element_type: str
    confidence: float


@dataclass
class ElementLocationFailure:
    """
    Failure response when Qwen cannot locate requested element.
    This is reported back to GPT-4o for plan revision.
    """
    element_description: str  # What was requested
    reason: str  # Why it failed (not found, ambiguous, etc.)
    available_elements: List[str]  # What elements ARE available
```

### MetricsSummary

```python
@dataclass
class MetricsSummary:
    """Execution metrics summary."""
    screenshot_count: int
    vision_calls: int
    orchestrator_calls: int
    plan_revisions: int
    total_actions: int
    successful_actions: int
    execution_time_seconds: float
    actions_per_screenshot: float
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Parallel model invocation
*For any* screenshot capture event, both the Vision Model and Orchestrator Model should be invoked without one blocking the other
**Validates: Requirements 1.1, 1.4**

### Property 2: PageAnalysis structure completeness
*For any* screenshot analyzed by the Vision Model, the resulting PageAnalysis should contain element types, labels, and coordinates for all identified elements
**Validates: Requirements 1.2**

### Property 3: ExecutionPlan generation from goals
*For any* task goal received by the Orchestrator Model, the system should produce an ExecutionPlan with actionable steps
**Validates: Requirements 1.3**

### Property 4: Parallel analysis result combination
*For any* agent step, after both parallel analyses complete, both PageAnalysis and ExecutionPlan should be available for decision making
**Validates: Requirements 1.5**

### Property 5: Empty plan triggers generation
*For any* agent state where the ExecutionPlan is empty, the Orchestrator Model should generate a new plan
**Validates: Requirements 2.2**

### Property 6: Error triggers plan revision
*For any* action execution failure, the Orchestrator Model should revise the ExecutionPlan
**Validates: Requirements 2.3**

### Property 7: Plan revision incorporates feedback
*For any* plan revision, the system should pass both error information and Vision Model feedback to the Orchestrator
**Validates: Requirements 2.4**

### Property 8: ExecutionPlan contains actionable steps
*For any* ExecutionPlan generated, all actions should have specific element identifiers and action types
**Validates: Requirements 2.5**

### Property 9: Info Seeking Query returns coordinates
*For any* straightforward element location request sent to the Vision Model, the response should contain either coordinates or a failure message
**Validates: Requirements 3.2**

### Property 10: Element location failure reporting
*For any* element that the Vision Model cannot locate, the system should produce an ElementLocationFailure and report it to the Orchestrator
**Validates: Requirements 3.3**

### Property 11: Form field type distinction
*For any* page containing form fields, the Vision Model should correctly distinguish between text inputs, dropdowns, date pickers, and other field types
**Validates: Requirements 3.5**

### Property 12: Same-page action batching
*For any* ExecutionPlan containing multiple consecutive same-page actions, the Agent System should execute them as a single Action Batch
**Validates: Requirements 4.1**

### Property 13: No intermediate screenshots in batch
*For any* Action Batch execution, the screenshot count should not increase until the batch completes or an error occurs
**Validates: Requirements 4.2**

### Property 14: Batch stops on error
*For any* Action Batch where an action fails, the system should stop batch execution immediately and capture a screenshot
**Validates: Requirements 4.3**

### Property 15: Conditional screenshot after batch
*For any* successfully completed Action Batch, a screenshot should be captured if and only if navigation occurred
**Validates: Requirements 4.4**

### Property 16: Sequential action execution
*For any* Action Batch, actions should execute in the order specified in the ExecutionPlan
**Validates: Requirements 4.5**

### Property 17: Output classification
*For any* Orchestrator Model output, the system should classify it as either ToolCall or InfoSeekingQuery
**Validates: Requirements 5.1**

### Property 18: Tool call invokes llm_call_step
*For any* ToolCall issued by the Orchestrator, the system should invoke the llm_call_step tool
**Validates: Requirements 5.2**

### Property 19: Info query routes to Vision Model
*For any* InfoSeekingQuery issued by the Orchestrator, the system should send the element location request to the Vision Model
**Validates: Requirements 5.3**

### Property 20: Vision response passed to Orchestrator
*For any* Vision Model response to an InfoSeekingQuery, the system should provide the coordinates or failure message to the Orchestrator Model
**Validates: Requirements 5.5**

### Property 21: Query failure triggers revision
*For any* InfoSeekingQuery that fails, the Orchestrator Model should receive the failure notification and revise the ExecutionPlan
**Validates: Requirements 5.6**

### Property 22: llm_call_step handles execution and logging
*For any* action executed via llm_call_step, both the action execution and logging should occur within the same execution context
**Validates: Requirements 5.7**

### Property 23: Error triggers screenshot
*For any* action execution failure, the Agent System should capture a screenshot
**Validates: Requirements 6.1**

### Property 24: Error details passed to Orchestrator
*For any* execution error, the system should provide error details to the Orchestrator Model
**Validates: Requirements 6.2**

### Property 25: Revision limit enforcement
*For any* task execution, the Agent System should fail the task after 3 consecutive plan revisions
**Validates: Requirements 6.5**

### Property 26: Vision call per screenshot
*For any* task execution, the number of Vision Model invocations should equal the number of screenshots captured
**Validates: Requirements 7.1**

### Property 27: Orchestrator call optimization
*For any* agent step where the ExecutionPlan contains valid actions, the Orchestrator Model should not be invoked
**Validates: Requirements 7.2**

### Property 28: Screenshot count tracking
*For any* task execution, the MetricsTracker should record the number of screenshots captured
**Validates: Requirements 8.1**

### Property 29: Vision call count tracking
*For any* task execution, the MetricsTracker should record the number of Vision Model invocations
**Validates: Requirements 8.2**

### Property 30: Orchestrator call count tracking
*For any* task execution, the MetricsTracker should record the number of Orchestrator Model invocations
**Validates: Requirements 8.3**

### Property 31: Plan revision count tracking
*For any* task execution, the MetricsTracker should record the number of plan revisions
**Validates: Requirements 8.4**

### Property 32: Metrics summary completeness
*For any* completed task, the MetricsSummary should include screenshot count, vision calls, orchestrator calls, and plan revisions
**Validates: Requirements 8.5**

### Property 33: Task definition parsing
*For any* task definition in the existing format, the Agent System should correctly parse goals and URLs
**Validates: Requirements 9.1**

### Property 34: Browser primitive compatibility
*For any* action execution, the Agent System should use existing browser automation primitives
**Validates: Requirements 9.2**

### Property 35: Result format compatibility
*For any* completed task, the Agent System should produce results in the existing ExperimentResult format
**Validates: Requirements 9.3**

### Property 36: TaskExecution workflow integration
*For any* task execution, the Agent System should integrate with the existing TaskExecution workflow
**Validates: Requirements 9.4**

### Property 37: Text input action selection
*For any* text input element identified by the Vision Model, the system should use the type action
**Validates: Requirements 10.1**

### Property 38: Dropdown action selection
*For any* dropdown element identified by the Vision Model, the system should use the select action
**Validates: Requirements 10.2**

### Property 39: Date picker action selection
*For any* date picker element identified by the Vision Model, the system should use the appropriate date input action
**Validates: Requirements 10.3**

### Property 40: Turn 1 plan simplicity
*For any* task execution starting with no prior page observations, the initial Execution Plan should contain at most one exploratory action
**Validates: Requirements 11.1, 11.2**

### Property 41: Turn 1 avoids assumptions
*For any* initial Execution Plan created before observing the page, the plan should not reference specific page elements that haven't been observed yet
**Validates: Requirements 11.4**

### Property 42: Subsequent turns enable detailed planning
*For any* Execution Plan created after the first page observation, the plan may contain multiple actions based on observed page structure
**Validates: Requirements 11.3**

## Error Handling

The system implements comprehensive error handling at multiple levels:

1. **Vision Model Failures**: When Qwen cannot locate an element, it returns ElementLocationFailure with available alternatives, allowing GPT-4o to revise the plan
2. **Action Execution Failures**: Failed actions trigger screenshot capture and plan revision with error context
3. **API Failures**: Both models implement retry logic for transient API errors
4. **Batch Execution Failures**: Batch execution stops immediately on first error, preserving partial progress
5. **Revision Limits**: System fails gracefully after 3 consecutive plan revisions to prevent infinite loops

## Testing Strategy

The system uses a dual testing approach combining unit tests and property-based tests:

### Unit Testing

Unit tests verify specific examples and integration points:
- Component initialization and configuration
- Error handling paths (API failures, timeouts, etc.)
- Edge cases (empty plans, missing elements, etc.)
- Integration with existing infrastructure (browser, task execution, etc.)

### Property-Based Testing

Property-based tests verify universal properties across all inputs using **Hypothesis** (Python PBT library):
- Each property test runs a minimum of 100 iterations
- Tests generate random inputs (screenshots, goals, plans, etc.) to verify properties hold universally
- Each property test is tagged with: `**Feature: hierarchical-action-batching, Property {number}: {property_text}**`
- Each correctness property is implemented by a SINGLE property-based test

Property tests focus on:
- Parallel execution behavior (Properties 1, 4)
- Plan generation and revision logic (Properties 3, 5, 6, 7, 8)
- Vision model analysis completeness (Properties 2, 11)
- Action batching optimization (Properties 12, 13, 14, 15, 16)
- Query routing and handling (Properties 17, 18, 19, 20, 21, 22)
- Error handling and recovery (Properties 23, 24, 25)
- Performance optimization (Properties 26, 27)
- Metrics tracking (Properties 28-32)
- Backward compatibility (Properties 33-36)
- Action selection logic (Properties 37-39)
