# Implementation Plan

- [x] 1. Create core data models
  - Implement PageElement and PageAnalysis dataclasses for vision model output
  - Implement Action and ExecutionPlan dataclasses for orchestrator output
  - Implement ExecutionError and ExecutionResult for error handling
  - Implement ToolCall, InfoSeekingQuery, ElementLocation, and ElementLocationFailure for query types
  - Implement MetricsSummary for performance tracking
  - Add proper type hints and documentation
  - _Requirements: 1.2, 1.3, 2.5, 3.2, 3.3, 5.1, 5.2, 5.3, 6.1, 8.5_

- [ ]* 1.1 Write property test for PageAnalysis structure completeness
  - **Property 2: PageAnalysis structure completeness**
  - **Validates: Requirements 1.2**

- [ ]* 1.2 Write property test for ExecutionPlan contains actionable steps
  - **Property 8: ExecutionPlan contains actionable steps**
  - **Validates: Requirements 2.5**

- [x] 2. Implement QwenVisionModel
  - Create QwenVisionModel class with async OpenAI client
  - Implement analyze_page method to analyze screenshots and produce PageAnalysis
  - Implement locate_element method to respond to Info Seeking Queries
  - Add prompt engineering for structured element identification with coordinates
  - Handle element location failures by returning ElementLocationFailure
  - Add API error handling with retry logic
  - _Requirements: 1.2, 3.1, 3.2, 3.3, 3.4, 3.5_

- [ ]* 2.1 Write property test for form field type distinction
  - **Property 11: Form field type distinction**
  - **Validates: Requirements 3.5**

- [ ]* 2.2 Write property test for Info Seeking Query returns coordinates
  - **Property 9: Info Seeking Query returns coordinates**
  - **Validates: Requirements 3.2**

- [ ]* 2.3 Write property test for element location failure reporting
  - **Property 10: Element location failure reporting**
  - **Validates: Requirements 3.3**

- [x] 3. Implement GPT4Orchestrator
  - Create GPT4Orchestrator class with async OpenAI client
  - Implement analyze_and_plan method to analyze goals and create/update ExecutionPlan
  - Implement parse_output method to classify outputs as ToolCall or InfoSeekingQuery
  - Add prompt engineering for plan generation and revision
  - Handle error information and location failures in plan revision
  - Add API error handling with retry logic
  - _Requirements: 1.3, 2.2, 2.3, 2.4, 5.1, 5.6_

- [ ]* 3.1 Write property test for ExecutionPlan generation from goals
  - **Property 3: ExecutionPlan generation from goals**
  - **Validates: Requirements 1.3**

- [ ]* 3.2 Write property test for empty plan triggers generation
  - **Property 5: Empty plan triggers generation**
  - **Validates: Requirements 2.2**

- [ ]* 3.3 Write property test for error triggers plan revision
  - **Property 6: Error triggers plan revision**
  - **Validates: Requirements 2.3**

- [ ]* 3.4 Write property test for plan revision incorporates feedback
  - **Property 7: Plan revision incorporates feedback**
  - **Validates: Requirements 2.4**

- [ ]* 3.5 Write property test for output classification
  - **Property 17: Output classification**
  - **Validates: Requirements 5.1**

- [ ]* 3.6 Write property test for query failure triggers revision
  - **Property 21: Query failure triggers revision**
  - **Validates: Requirements 5.6**

- [x] 4. Implement ActionBatchExecutor
  - Create ActionBatchExecutor class for executing action batches
  - Implement create_batch method to extract consecutive same-page actions from plan
  - Implement execute_batch method to execute actions sequentially
  - Add logic to stop batch execution on first error
  - Integrate with browser automation primitives
  - _Requirements: 4.1, 4.2, 4.3, 4.5_

- [ ]* 4.1 Write property test for same-page action batching
  - **Property 12: Same-page action batching**
  - **Validates: Requirements 4.1**

- [ ]* 4.2 Write property test for no intermediate screenshots in batch
  - **Property 13: No intermediate screenshots in batch**
  - **Validates: Requirements 4.2**

- [ ]* 4.3 Write property test for batch stops on error
  - **Property 14: Batch stops on error**
  - **Validates: Requirements 4.3**

- [ ]* 4.4 Write property test for sequential action execution
  - **Property 16: Sequential action execution**
  - **Validates: Requirements 4.5**

- [x] 5. Implement MetricsTracker
  - Create MetricsTracker class with metric recording methods
  - Implement record_screenshot, record_vision_call, record_orchestrator_call methods
  - Implement record_plan_revision method
  - Implement get_summary method to produce MetricsSummary
  - Add timing logic with start_time and end_time tracking
  - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

- [ ]* 5.1 Write property test for screenshot count tracking
  - **Property 28: Screenshot count tracking**
  - **Validates: Requirements 8.1**

- [ ]* 5.2 Write property test for vision call count tracking
  - **Property 29: Vision call count tracking**
  - **Validates: Requirements 8.2**

- [ ]* 5.3 Write property test for orchestrator call count tracking
  - **Property 30: Orchestrator call count tracking**
  - **Validates: Requirements 8.3**

- [ ]* 5.4 Write property test for plan revision count tracking
  - **Property 31: Plan revision count tracking**
  - **Validates: Requirements 8.4**

- [ ]* 5.5 Write property test for metrics summary completeness
  - **Property 32: Metrics summary completeness**
  - **Validates: Requirements 8.5**

- [x] 6. Implement AdaptiveAgent core workflow
  - Create AdaptiveAgent class extending BaseAgent
  - Implement __init__ to configure QwenVisionModel, GPT4Orchestrator, ActionBatchExecutor, MetricsTracker
  - Implement step method with parallel analysis workflow
  - Add screenshot capture logic
  - Implement parallel invocation of Qwen and GPT-4o
  - Add result combination logic after parallel analysis
  - _Requirements: 1.1, 1.4, 1.5_

- [ ]* 6.1 Write property test for parallel model invocation
  - **Property 1: Parallel model invocation**
  - **Validates: Requirements 1.1, 1.4**

- [ ]* 6.2 Write property test for parallel analysis result combination
  - **Property 4: Parallel analysis result combination**
  - **Validates: Requirements 1.5**

- [x] 7. Implement query routing and handling
  - Add query classification logic to AdaptiveAgent
  - Implement Tool Call handling via llm_call_step
  - Implement Info Seeking Query routing to QwenVisionModel
  - Add logic to pass Vision Model responses back to Orchestrator
  - Handle ElementLocationFailure by triggering plan revision
  - _Requirements: 5.1, 5.2, 5.3, 5.5, 5.6, 5.7_

- [ ]* 7.1 Write property test for tool call invokes llm_call_step
  - **Property 18: Tool call invokes llm_call_step**
  - **Validates: Requirements 5.2**

- [ ]* 7.2 Write property test for info query routes to Vision Model
  - **Property 19: Info query routes to Vision Model**
  - **Validates: Requirements 5.3**

- [ ]* 7.3 Write property test for vision response passed to Orchestrator
  - **Property 20: Vision response passed to Orchestrator**
  - **Validates: Requirements 5.5**

- [ ]* 7.4 Write property test for llm_call_step handles execution and logging
  - **Property 22: llm_call_step handles execution and logging**
  - **Validates: Requirements 5.7**

- [x] 8. Implement action batch execution in AdaptiveAgent
  - Integrate ActionBatchExecutor into step method
  - Add logic to execute action batches from ExecutionPlan
  - Implement conditional screenshot capture after batch completion
  - Add screenshot capture only if navigation occurred
  - Handle batch execution errors with screenshot capture
  - _Requirements: 4.4, 6.1_

- [ ]* 8.1 Write property test for conditional screenshot after batch
  - **Property 15: Conditional screenshot after batch**
  - **Validates: Requirements 4.4**

- [ ]* 8.2 Write property test for error triggers screenshot
  - **Property 23: Error triggers screenshot**
  - **Validates: Requirements 6.1**

- [x] 9. Implement error handling and plan revision
  - Add error capture logic to AdaptiveAgent
  - Implement error detail passing to GPT4Orchestrator
  - Add plan revision trigger on execution errors
  - Implement revision limit enforcement (max 3 consecutive revisions)
  - Add task failure logic after revision limit reached
  - _Requirements: 6.2, 6.5_

- [x] 9.1 Write property test for error details passed to Orchestrator
  - **Property 24: Error details passed to Orchestrator**
  - **Validates: Requirements 6.2**

- [ ]* 9.2 Write property test for revision limit enforcement
  - **Property 25: Revision limit enforcement**
  - **Validates: Requirements 6.5**

- [x] 10. Implement performance optimizations
  - Add logic to skip Orchestrator calls when ExecutionPlan has valid actions
  - Ensure Vision Model is called once per screenshot
  - Integrate MetricsTracker throughout AdaptiveAgent workflow
  - Add metrics recording at appropriate points (screenshots, model calls, revisions)
  - _Requirements: 7.1, 7.2_

- [ ]* 10.1 Write property test for vision call per screenshot
  - **Property 26: Vision call per screenshot**
  - **Validates: Requirements 7.1**

- [ ]* 10.2 Write property test for orchestrator call optimization
  - **Property 27: Orchestrator call optimization**
  - **Validates: Requirements 7.2**

- [x] 11. Implement action selection logic
  - Add element type to action type mapping
  - Implement logic to select type action for text inputs
  - Implement logic to select select action for dropdowns
  - Implement logic to select date input action for date pickers
  - Integrate action selection into ActionBatchExecutor
  - _Requirements: 10.1, 10.2, 10.3_

- [ ]* 11.1 Write property test for text input action selection
  - **Property 37: Text input action selection**
  - **Validates: Requirements 10.1**

- [ ]* 11.2 Write property test for dropdown action selection
  - **Property 38: Dropdown action selection**
  - **Validates: Requirements 10.2**

- [ ]* 11.3 Write property test for date picker action selection
  - **Property 39: Date picker action selection**
  - **Validates: Requirements 10.3**

- [x] 12. Implement backward compatibility layer
  - Ensure AdaptiveAgent works with existing Task format
  - Implement task definition parsing for goals and URLs
  - Integrate with existing AgentBrowser interface
  - Ensure ExperimentResult format matches existing system
  - Test integration with existing TaskExecution workflow
  - _Requirements: 9.1, 9.2, 9.3, 9.4_

- [ ]* 12.1 Write property test for task definition parsing
  - **Property 33: Task definition parsing**
  - **Validates: Requirements 9.1**

- [ ]* 12.2 Write property test for browser primitive compatibility
  - **Property 34: Browser primitive compatibility**
  - **Validates: Requirements 9.2**

- [ ]* 12.3 Write property test for result format compatibility
  - **Property 35: Result format compatibility**
  - **Validates: Requirements 9.3**

- [ ]* 12.4 Write property test for TaskExecution workflow integration
  - **Property 36: TaskExecution workflow integration**
  - **Validates: Requirements 9.4**

- [x] 13. Implement turn 1 planning constraints
  - Add logic to detect when agent is in turn 1 (no prior observations)
  - Modify orchestrator prompt to limit turn 1 plans to single exploratory actions
  - Add validation to ensure turn 1 plans don't assume page structure
  - Allow multi-step planning only after first page observation
  - Update prompt engineering to emphasize observation-first strategy
  - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5_

- [ ]* 13.1 Write property test for turn 1 plan simplicity
  - **Property 40: Turn 1 plan simplicity**
  - **Validates: Requirements 11.1, 11.2**

- [ ]* 13.2 Write property test for turn 1 avoids assumptions
  - **Property 41: Turn 1 avoids assumptions**
  - **Validates: Requirements 11.4**

- [ ]* 13.3 Write property test for subsequent turns enable detailed planning
  - **Property 42: Subsequent turns enable detailed planning**
  - **Validates: Requirements 11.3**

- [x] 14. Add comprehensive error handling
  - Implement retry logic for Vision Model API failures
  - Implement retry logic for Orchestrator Model API failures
  - Add fallback logic for empty PageAnalysis
  - Add fallback logic for failed action planning
  - Implement partial batch execution recovery on action errors
  - Add screenshot retry logic
  - _Requirements: 6.1, 6.2_

- [ ]* 14.1 Write unit tests for error handling
  - Test Vision Model retry logic
  - Test Orchestrator Model retry logic
  - Test action execution error recovery
  - Test screenshot capture failures
  - _Requirements: 6.1, 6.2_

- [x] 15. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.
