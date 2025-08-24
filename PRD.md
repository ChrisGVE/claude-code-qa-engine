# Claude Debugger MCP - Product Requirements Document

## Overview

The Claude Debugger MCP is a specialized MCP server that orchestrates complex testing workflows using state-as-data principles. It manages the complete QA lifecycle from test campaign creation to bug resolution, integrating multiple QA tools through durable workflow orchestration.

## Core Functionality

### Scope Definition

**Primary Responsibility**:
- **Testing Workflow Orchestration**: Complete QA lifecycle management using Temporal for durable workflows
- **State-as-Data Architecture**: All QA state lives in specialized systems, not prompts
- **Multi-Tool Integration**: Thin interface layer over Gitea + Kiwi TCMS + ReportPortal + Temporal

**Architecture Decision Rationale**:
- **Task-master Context Drift**: Experimentation showed Task-master alone causes significant context drift during test campaigns
- **Campaign Completion Guarantee**: Temporal workflows ensure QA campaigns complete successfully despite context limitations
- **Specialized Tool Requirements**: QA workflows require purpose-built tools for proper state management

## Technical Architecture

### Tool Stack Integration

**Core QA Infrastructure** (self-hosted, GitHub-compatible):
- **Gitea/Forgejo**: Bug tracking with Issues (GitHub Actions-compatible syntax)
- **Kiwi TCMS**: Test campaign and case management
- **ReportPortal**: Test result analytics, automation results, flaky test detection
- **Temporal**: Durable workflow orchestration, retries, audit history
- **Langfuse**: Token budgets, role drift monitoring

**Port Allocation**:
- Gitea/Forgejo: localhost:3000
- Kiwi TCMS: localhost:8082
- ReportPortal: localhost:8081
- Temporal: localhost:7233

### QA MCP Tool Surface

**Campaign Management** (→ Kiwi TCMS):
```python
qa.create_campaign(task_id, scope, entry_criteria, exit_criteria)
qa.get_campaign(campaign_id)
qa.update_campaign_status(campaign_id, status)
```

**Test Execution** (→ ReportPortal + local runners):
```python
qa.start_test_run(campaign_id, test_suite)
qa.record_result(run_id, test_id, status, artifacts)
qa.get_test_results(run_id)
```

**Bug Management** (→ Gitea Issues):
```python
qa.create_bug(title, repro_steps, severity, failing_run_id)
qa.link_bug_to_task(bug_id, task_id)
qa.update_bug_status(bug_id, status)
qa.get_bug_details(bug_id)
```

**Analytics** (→ ReportPortal):
```python
qa.get_flaky_tests(campaign_id)
qa.get_coverage_trends(project_id)
qa.get_failure_patterns(project_id, time_range)
```

**Human Approval Gates**:
```python
qa.request_approval(campaign_id, approval_type, context)
qa.check_approval_status(approval_id)
```

## Workflow Orchestration

### Task Hierarchy Integration

**Simplified Task Structure** (integrates with Task-master):
```
Task 15: Implement feature X
Task 16: Test campaign for feature X (depends on 15)  
Task 17: Fix bugs from test campaign 16 (depends on 16, created if bugs found)
  ├── Subtask 17.1: Fix Bug #101 (links to Gitea Issue #101)
  ├── Subtask 17.2: Fix Bug #102 (links to Gitea Issue #102)
  └── Subtask 17.3: Fix Bug #103 (links to Gitea Issue #103)
Task 18: Regression test after fixes (optional)
```

**Workflow Handoff Pattern**:
1. **Claude Code** completes development Task 15
2. **Claude Code** encounters QA Task 16 → delegates to **Claude Debugger MCP**
3. **Claude Debugger MCP** (via Temporal workflow):
   - Creates test campaign in Kiwi TCMS
   - Spawns test execution subagents
   - **If bugs found**: creates bugs in Gitea Issues + Task 17 with subtasks in Task-master
   - Marks Task 16 as `done` (campaign complete)
4. **Claude Code** handles subtasks 17.1, 17.2, 17.3 (bug fixes)
5. **Claude Debugger MCP** verification: re-runs tests, closes bugs if successful

### State-as-Data Principle

**QA State Distribution**:
- **Test campaigns/cases**: Kiwi TCMS (operational state)
- **Test results/analytics**: ReportPortal (historical data, trends)
- **Bug details/lifecycle**: Gitea Issues (detailed tracking)
- **Workflow orchestration**: Temporal (process state, retries)
- **Project context/decisions**: Qdrant (summaries, architectural decisions)
- **Task workflow progress**: Task-master (high-level progress tracking)

**Cross-System Linking**:
- Task-master tasks reference QA entities via stable link strings:
  - `bug:gitea://repo/issues/101` ↔ `task 17.1`
  - `campaign:kiwi://project/<id>` ↔ `task 16`
  - `run:reportportal://project/<launch_id>` ↔ test execution

## Temporal Workflow Design

### Durable Workflow Patterns

**Test Campaign Workflow**:
1. **Initialize Campaign**: Create Kiwi TCMS campaign, validate entry criteria
2. **Execute Test Suite**: Parallel test execution with automatic retries
3. **Collect Results**: Aggregate results in ReportPortal
4. **Bug Triage**: Analyze failures, create Gitea issues for genuine bugs
5. **Task Generation**: Create Task-master fix tasks if bugs found
6. **Campaign Closure**: Human approval gate, mark campaign complete

**Error Recovery Strategies**:
- **Test Execution Failures**: Automatic retry with exponential backoff
- **Tool Connectivity Issues**: Circuit breaker pattern, graceful degradation
- **Human Approval Timeouts**: Escalation workflows, default decisions
- **Workflow Interruption**: Resume from last checkpoint, preserve state

### Subagent Integration

**QA-Specific Subagents** (managed by Claude Debugger MCP):
- **Test Executor**: Runs specific test suites, reports results
- **Bug Triage Analyst**: Analyzes failures, determines if genuine bugs
- **Coverage Analyst**: Analyzes test coverage, identifies gaps
- **Performance Analyst**: Monitors performance metrics, identifies regressions

**Resource Management**:
- Subagents spawned on-demand per workflow step
- Token/time budgets enforced by Temporal workflows
- Failed subagents automatically retried or replaced

## Integration Patterns

### Claude Code Integration

**Delegation Pattern**:
```python
# Claude Code detects QA task
if task.type == "qa_campaign":
    qa_result = qa_engine_mcp.execute_campaign(
        task_id=task.id,
        scope=task.details,
        entry_criteria=task.entry_criteria
    )
    
    # Claude Debugger MCP handles complete workflow
    # Returns when campaign complete or requires intervention
    
    if qa_result.bugs_found:
        # Task-master updated with fix tasks by Claude Debugger MCP
        continue_with_bug_fixes()
    else:
        mark_task_complete(task.id)
```

**Human Approval Integration**:
- Claude Debugger MCP requests approval via Claude Code interface
- Approval UI can be CLI prompts or web interface
- Timeout handling with sensible defaults

### GitHub Compatibility

**Migration Path**:
- Gitea Actions use GitHub Actions YAML syntax
- QA workflows portable to GitHub environment
- CI/CD integration points preserved

**Update Strategy**:
- Batch updates for QA tool stack (Gitea, Kiwi TCMS, ReportPortal, Temporal)
- Rolling updates with rollback capability
- Service health monitoring during updates

## Operational Safeguards

### System Reliability

**Health Monitoring**:
- All QA services monitored for availability and performance
- Automated alerts for service degradation
- Health check endpoints for each integrated tool

**Data Protection**:
- Regular backups of Gitea, Kiwi TCMS, ReportPortal data
- Backup verification and restore procedures
- Data retention policies for test results and bug history

**Service Dependencies**:
- Clear startup order and dependency management
- Failure isolation between QA components
- Graceful degradation when services unavailable

### Performance Optimization

**Resource Management**:
- Test execution parallelization limits
- Memory and CPU constraints for test runners
- Cleanup procedures for test artifacts

**Caching Strategy**:
- Test result caching for repeated test runs
- Artifact caching for performance tests
- Smart cache invalidation on code changes

## Development Requirements

### Dependencies

**Core Infrastructure**:
- Docker/Podman for service orchestration
- Temporal server and Python SDK
- Gitea/Forgejo with Actions enabled
- Kiwi TCMS with API access
- ReportPortal with project configuration

**Python Dependencies**:
- temporal-python-sdk (workflow orchestration)
- requests (HTTP API integration)
- claude-code-sdk (MCP framework)
- pydantic (data validation)

### Testing Strategy

**Unit Tests**:
- Individual tool integrations (Gitea, Kiwi TCMS, ReportPortal)
- Workflow logic and error handling
- Bug creation and linking algorithms

**Integration Tests**:
- End-to-end campaign workflows
- Multi-tool state synchronization
- Temporal workflow durability

**Performance Tests**:
- Concurrent campaign execution
- Large test suite handling
- Resource utilization under load

## Success Metrics

**Primary Metrics**:
- Campaign completion rate (target: >95%)
- Bug detection accuracy (genuine bugs vs false positives)
- Time to campaign completion (target: <2 hours for typical campaigns)

**Secondary Metrics**:
- Test execution parallelization efficiency
- Human approval response time
- System uptime and reliability

**Quality Metrics**:
- Flaky test reduction rate
- Test coverage improvement over time
- Bug fix verification success rate