**Document Metadata**

* **Author:** Vic Genin

* **Date:** 2026-05-01

**Versioning History**

| Version | Date | Author | Description |
| :---- | :---- | :---- | :---- |
| 1.0 | 2026-02-01 | Vic Genin | Initial Draft |
|  |  |  |  |

# Master Test Plan for Sundial Staging Environment and Sundial Testnet

## 1\. Introduction

This Master Test Plan defines the validation approach for the Sundial staging environment and Sundial testnet. The plan covers functional, integration, regression, baseline security, and controlled load testing for supported testnet workflows and exposed system interfaces.

Testing will verify that Sundial services can be deployed, operated, monitored, and exercised consistently through defined validation scenarios. The validation scope includes service startup, health checks, API behavior, transaction submission, mempool processing, block construction, L1 event ingestion, background worker execution, local state updates, state root generation, observability, negative input handling, automated regression checks, baseline security validation, and controlled load execution.

The objective is to produce repeatable evidence that the Sundial staging and testnet environments operate reliably under defined validation conditions and provide a stable foundation for further protocol hardening, audit preparation, and production-readiness work.

This MTP is structured as a testnet validation plan. It is intended to verify operational behavior, integration behavior, baseline security posture, and controlled-load stability of the Sundial staging and testnet environments.

The plan does not attempt to replace production-readiness reviews, formal verification, external audit processes, or mainnet launch assessments. Those activities are separate hardening steps and may be conducted after testnet validation.

The purpose of this document is to establish a controlled, repeatable, and evidence-based approach to validating the Sundial staging environment and Sundial testnet before the environment is used as a foundation for broader production-readiness follow-up. The plan defines what will be tested, how testing will be executed, what evidence will be collected, how defects will be managed, and what criteria will be used to determine completion of the validation cycle.

This document is intended for engineering, quality assurance, DevOps, security, protocol, and project stakeholders involved in planning, executing, reviewing, or approving Sundial testnet validation activities. It provides a common reference for expected test activities and ensures that validation is conducted in a structured manner aligned with accepted software testing and environment validation practices.

## 2\. Test Objectives

The objectives of this MTP are to:

1. Confirm that Sundial services can be deployed and started in the staging and testnet environments.

2. Validate health, readiness, and observability signals.

3. Verify supported API behavior and response consistency.

4. Validate transaction submission and mempool processing.

5. Verify block construction and state root generation.

6. Validate L1 event ingestion and downstream processing.

7. Confirm local execution and state update behavior.

8. Establish repeatable automated regression coverage.

9. Perform baseline security validation for exposed services and interfaces.

10. Measure stability under controlled load.

11. Document defects, resolutions, test evidence, and residual risks.

The test objectives are designed to provide confidence that the Sundial staging environment and Sundial testnet can support defined validation scenarios in a repeatable manner. Each objective is tied to observable outputs such as service status, API responses, transaction processing results, logs, metrics, generated state roots, worker execution records, regression test results, defect records, and load execution reports.

The validation effort will focus on proving operational consistency rather than asserting final production readiness. Successful completion of this MTP will demonstrate that the tested workflows are stable enough for continued protocol hardening, audit preparation, and production-readiness follow-up.

## 3\. Scope

This MTP covers validation of the Sundial staging environment and Sundial testnet.

Testing includes functional, integration, regression, baseline security, and controlled load validation of supported testnet workflows and exposed system interfaces.

The validation scope includes:

* service deployment and startup

* health and readiness checks

* API request/response behavior

* transaction submission

* mempool processing

* block construction

* L1 event ingestion

* local execution and state updates

* state root generation and tracking

* background worker execution

* log and metric observability

* negative input handling

* automated regression testing

* baseline service-level security checks

* controlled load and stability testing

The purpose of this scope is to verify that the Sundial staging and testnet environments can be deployed, operated, monitored, and exercised through defined validation scenarios.

The scope includes both individual component validation and end-to-end workflow validation where supported workflows require multiple services to interact. Functional testing will confirm that individual interfaces and workflows behave as expected under normal input conditions. Integration testing will confirm that connected services exchange data and produce expected observable outputs. Regression testing will confirm that repeatable automated checks can be executed consistently across validation cycles. Baseline security validation will focus on exposed service and interface risks visible within the staging and testnet environments. Controlled load testing will assess stability under defined test conditions without attempting to establish final production throughput limits.

Testing will be performed against defined configurations and supported testnet workflows only. Where environmental constraints, configuration assumptions, or test data limitations affect execution, those constraints will be recorded in the test evidence and final validation report.

## 4\. Out of Scope

The following areas are outside the scope of this MTP:

* mainnet deployment validation

* production funds handling

* external third-party audit certification

* production capacity benchmarking

* long-duration economic-security analysis

* production incident response procedures

* production operator onboarding

* post-testnet protocol hardening activities

* non-testnet operational processes

These items may be addressed separately as part of production-readiness, hardening, audit, or mainnet-preparation workstreams.

The exclusion of these areas ensures that this MTP remains focused on controlled testnet validation activities. This plan does not certify production readiness, does not replace independent security assessment, and does not define the operational processes required for mainnet deployment. Any conclusions produced through this validation cycle must therefore be interpreted within the boundaries of the Sundial staging environment and Sundial testnet.

## 5\. Test Strategy

The test strategy is based on staged validation of operational behavior, integration behavior, baseline security posture, and controlled-load stability. Testing will be executed using a combination of manual validation, automated regression checks, scripted API calls, service log review, metrics review, and controlled load execution.

The strategy is organized around the following validation layers:

### 5.1 Environment Validation

Environment validation will confirm that required Sundial services can be deployed, configured, started, and monitored in the staging and testnet environments. This includes verifying service startup behavior, readiness indicators, health endpoints, configuration presence, dependency availability, logging behavior, metrics exposure, and recovery behavior after controlled restarts.

The expected outcome is evidence that the environment can reach an operational state suitable for executing supported testnet workflows.

### 5.2 Functional Validation

Functional validation will focus on supported service interfaces and workflow behavior. Test cases will verify expected API request and response behavior, transaction submission outcomes, negative input handling, local execution behavior, state updates, and state root generation.

Functional testing will include both positive and negative scenarios. Positive scenarios will verify that valid requests produce expected observable outputs. Negative scenarios will verify that unsupported, malformed, or invalid requests are handled safely and consistently.

### 5.3 Integration Validation

Integration validation will verify that services interact correctly across workflow boundaries. This includes transaction submission into mempool processing, mempool data availability for block construction, L1 event ingestion into downstream processing, worker execution, local state update handling, and state root tracking.

The expected outcome is evidence that supported testnet workflows can progress across service boundaries and produce consistent logs, metrics, API outputs, and state records.

### 5.4 Regression Validation

Regression validation will establish repeatable automated checks for core supported workflows and exposed system interfaces. Automated regression tests will be executed during planned validation cycles and after relevant changes to confirm that previously validated behavior remains stable.

Regression evidence will include test execution results, pass/fail summaries, logs, and references to build or release identifiers where applicable.

### 5.5 Baseline Security Validation

Baseline security validation will focus on service-level and interface-level risks observable in the Sundial staging environment and Sundial testnet. The purpose is to identify critical security findings related to exposed services, request handling, configuration exposure, sensitive data leakage, and dependency or static analysis findings.

This validation does not replace external audit, formal verification, or production security review. It provides controlled evidence that critical service-level issues identified during testnet validation have been resolved or documented before the final validation package is completed.

### 5.6 Controlled Load Validation

Controlled load validation will assess stability of supported staging and testnet workflows under defined load conditions. The objective is to observe service behavior, response latency, error rate, resource usage, queued processing, worker execution, transaction handling, and block construction behavior while load is applied.

Controlled load testing is not intended to establish final production throughput limits. Results will be used to identify stability issues, operational constraints, and recommended production-readiness follow-up items.

## 6\. Test Items and Features to Be Tested

The following test items and features are included in the validation cycle.

### 6.1 Service Deployment and Startup

Testing will verify that Sundial services can be deployed and started in the target staging and testnet environments using the defined deployment procedures. Validation will confirm successful service startup, dependency connectivity, configuration loading, port binding, container or process availability, and generation of expected startup logs.

Test evidence may include deployment logs, service status outputs, orchestration status, startup timestamps, and health endpoint responses.

### 6.2 Health and Readiness Checks

Testing will verify that health and readiness checks provide accurate signals for service availability. Health endpoints, readiness endpoints, process status, and monitoring indicators will be reviewed to confirm that operational state is observable.

Validation will include expected behavior during normal operation and controlled restart scenarios where applicable.

### 6.3 API Request and Response Behavior

Testing will verify exposed API behavior for supported request types. Validation will include request structure, response format, status codes, error responses, timeout handling, and consistency of returned data.

Negative testing will include malformed input, missing parameters, unsupported methods, invalid payloads, oversized payloads, and unauthorized or unsupported request behavior where applicable.

### 6.4 Transaction Submission

### 6.5 Mempool Processing

Expected evidence may include API responses, transaction identifiers, mempool entries, logs, metrics, worker activity, and downstream processing records.

### 6.5 Mempool Processing

Testing will verify that submitted transactions are accepted into the appropriate processing flow and that mempool-related behavior can be observed. Validation will include transaction acceptance, ordering observations where applicable, error handling for invalid submissions, and availability of mempool-related logs or metrics.

### 6.6 Block Construction

Testing will verify that supported block construction workflows operate under defined validation conditions. The test will confirm that eligible transaction data can contribute to block construction and that resulting observable outputs are generated.

Evidence may include block metadata, logs, metrics, transaction inclusion observations, and downstream state tracking outputs.

### 6.7 L1 Event Ingestion

Testing will verify that L1 event ingestion can be exercised and observed in the staging and testnet environments. Validation will confirm that supported events are detected, processed, and reflected in downstream service behavior.

Expected evidence may include ingestion logs, event identifiers, processing status, worker activity, and related state changes.

### 6.8 Background Worker Execution

Testing will verify that background workers execute supported tasks as expected. This includes worker startup, queued task handling, retry behavior where applicable, error logging, task completion records, and stability during controlled load conditions.

### 6.9 Local Execution and State Updates

Testing will verify that local execution workflows produce expected state updates. This includes observing state transition behavior, confirming relevant logs and metrics, and validating that expected state-related outputs are available for review.

### 6.10 State Root Generation and Tracking

Testing will verify that state root generation and tracking can be observed as part of supported validation scenarios. Evidence will include generated state roots, associated timestamps or block references, logs, and any available interface outputs used to confirm state root visibility.

### 6.11 Observability

Testing will verify that logs, metrics, and operational outputs provide sufficient visibility into validation scenarios. Observability validation will include service logs, error logs, metrics exposure, transaction-related metrics, worker metrics, resource usage observations, and controlled load indicators.

The objective is to confirm that test execution can be monitored and that evidence can be collected to support defect analysis and validation reporting.

## 7\. Test Approach

The test approach will combine structured test case execution with exploratory review of logs, metrics, and observable outputs. Each planned test case will include a test case identifier, objective, preconditions, input data, execution steps, expected results, actual results, status, and evidence references.

Testing will follow a controlled sequence:

1. Confirm environment readiness.

2. Execute service deployment and startup checks.

3. Validate health, readiness, and monitoring indicators.

4. Execute functional API tests.

5. Execute transaction submission scenarios.

6. Validate mempool processing behavior.

7. Execute block construction scenarios.

8. Validate L1 event ingestion and downstream processing.

9. Confirm local execution and state update behavior.

10. Validate state root generation and tracking.

11. Execute negative input and error-handling tests.

12. Execute automated regression tests.

13. Perform baseline security validation.

14. Execute controlled load validation.

15. Consolidate evidence, defects, and residual risks.

Test execution may be performed iteratively where defects require correction and retesting. Retesting will be limited to affected areas and associated regression checks unless a broader impact assessment indicates that additional validation is needed.

## 8\. Test Environment

Testing will be performed in the Sundial staging environment and Sundial testnet. The environments must be configured to support execution of the validation scenarios defined in this MTP.

The environment baseline should include:

* required Sundial services

* service configuration files or environment variables

* required network connectivity between services

* exposed system interfaces for validation

* logging and metrics collection

* test accounts, keys, or credentials where applicable

* test data required for supported workflows

* deployment automation or documented deployment procedure

* regression test execution capability

* controlled load test execution capability

Before formal test execution begins, an environment readiness check will be performed. The readiness check will confirm that required services are available, configuration is present, health checks are responding, logs are accessible, metrics are available, and required test data or credentials are prepared.

Any environment limitation that affects execution will be recorded in the test evidence. If a limitation prevents execution of a planned test case, the test case will be marked blocked and reviewed for resolution or documented as a residual limitation within the validation report.

## 9\. Test Data Management

Test data will be prepared to support positive, negative, integration, regression, baseline security, and controlled load validation scenarios. Test data may include valid API payloads, malformed payloads, oversized payloads, transaction inputs, event inputs, configuration references, and expected output records.

Test data must be controlled and traceable to the test cases in which it is used. Where test data includes keys, credentials, or sensitive configuration, such data must be handled according to internal access-control and secret-management practices. Sensitive values must not be included in public logs, public issue records, or externally shared validation reports.

The test data approach will include:

* valid data for supported workflow execution

* invalid data for negative input handling

* boundary data for size and field validation

* repeatable transaction data where applicable

* controlled input sets for regression tests

* controlled payload sets for baseline security validation

* controlled request sets for load testing

Evidence collected during test execution should identify the type of data used without exposing sensitive values unnecessarily.

## 10\. Test Execution Management

Test execution will be managed through a defined execution schedule and a test tracking mechanism. Each test case will be assigned a status such as Not Started, In Progress, Passed, Failed, Blocked, Retest Required, or Not Executed.

Execution records will include actual results, evidence references, defect references, and tester notes. Where a test case fails, a defect will be created in the private defect register unless the issue is already known and tracked.

The test lead or designated validation owner will review execution progress regularly during the validation cycle. Blocked tests, repeated failures, critical findings, and environment instability will be escalated to the appropriate engineering or operational owners.

Retesting will occur after fixes, configuration updates, or environment corrections are applied. Regression testing will be executed after relevant changes to ensure that corrected behavior has not introduced unexpected impact to previously validated workflows.

## 11\. Exit and Acceptance Criteria

Testing will be considered complete when:

* planned functional tests for supported staging and testnet workflows have been executed;

* integration scenarios produce expected observable outputs;

* automated regression tests can be executed repeatably;

* no unresolved Critical defects remain within the tested scope;

* baseline security checks identify no unresolved Critical findings;

* controlled load testing completes without unrecoverable service failure;

* test execution logs, reports, metrics, and defect records are collected;

* any remaining non-critical findings are documented with severity, impact, and recommended follow-up.

Completion of the validation cycle does not imply completion of production-readiness reviews, external audits, formal verification activities, or mainnet launch assessments. It indicates that the defined validation scenarios for the Sundial staging environment and Sundial testnet have been executed and that the resulting evidence supports continued production-readiness follow-up.

## 12\. Entry Criteria

Testing may begin when the following entry criteria are satisfied:

* Sundial staging environment is available for validation.

* Sundial testnet is available for supported workflow execution.

* Required services can be deployed or accessed.

* Required configuration is available.

* Health and readiness endpoints are reachable where applicable.

* Logs and metrics are accessible to the test team.

* Test data and validation credentials are available where required.

* Test cases or validation checklists are prepared.

* Defect register is available for recording findings.

* Regression test execution method is available.

* Baseline security validation tools or scripts are available where applicable.

* Controlled load testing approach and limits are defined.

If any entry criterion is not met, the validation owner will determine whether testing can proceed with limitations or whether execution should be delayed until the environment is ready.

## 13\. Suspension and Resumption Criteria

Testing may be suspended if any of the following conditions occur:

* critical environment instability prevents reliable execution;

* required services cannot be started or accessed;

* health or readiness checks fail consistently;

* test data or credentials are unavailable;

* logging or metrics are unavailable for required evidence collection;

* a Critical defect prevents continuation of dependent scenarios;

* controlled load testing causes unrecoverable service failure;

* security validation identifies a Critical issue requiring immediate correction.

Testing may resume when the blocking condition is resolved, the affected services are available, and the validation owner confirms that reliable execution can continue. Resumption may include repeating affected setup steps, rerunning impacted test cases, and executing targeted regression checks.

## 14\. Baseline Security Validation Scope

Security validation will focus on service-level and interface-level risks observable in the staging and testnet environments.

The security test scope includes:

* malformed input handling

* oversized payload handling

* API error handling

* unauthorized or unsupported request behavior

* configuration exposure checks

* dependency and static analysis checks

* sensitive data leakage in logs

* basic abuse-case testing for exposed endpoints

The objective is to identify and resolve Critical service-level findings before the testnet validation package is finalized.

Security validation will include review of exposed system interfaces, request handling behavior, error responses, configuration visibility, dependency scan outputs, static analysis outputs where available, and logs generated during negative and abuse-case testing. Findings will be classified by severity and recorded in the private defect register.

The baseline security validation activity is limited to observable service-level and interface-level behavior within the staging and testnet environments. It does not replace external security audit, formal verification, mainnet security review, or long-duration economic-security analysis.

## 15\. Controlled Load Validation

Controlled load testing will be performed against supported staging and testnet workflows.

The load test scope includes:

* concurrent API requests

* concurrent transaction submissions

* sustained transaction submission over a defined test window

* block construction behavior during load

* worker stability during queued processing

* response latency

* error rate

* resource usage observations

The objective is to demonstrate environment stability under defined load conditions, not to establish final production throughput limits.

Controlled load testing will use defined request rates, concurrency levels, payload types, duration, and observation points. Load execution will be monitored through logs, metrics, resource usage, response status codes, latency measurements, error rates, worker queues, transaction processing observations, and block construction behavior.

Load testing will be executed in a controlled manner to avoid uncontrolled disruption to the validation environment. Test limits will be documented before execution. If unrecoverable service failure occurs, the test will be stopped, evidence will be collected, and a defect will be recorded.

The final load test summary will document test conditions, execution duration, request volumes, observed latency, error rates, resource usage, service stability, worker behavior, and recommended follow-up actions.

## 16\. Defect Management

Defects identified during testing will be tracked in a private defect register and summarized in the final validation report.

Each defect record will include:

* defect ID

* component

* description

* severity

* test case reference

* reproduction steps

* status

* resolution notes

* evidence link, log reference, or commit reference

Public issue creation is not required for this validation cycle. Where fixes are applied, the defect register may reference commits, pull requests, or release notes.

Defect severity will be assigned based on impact to the tested scope. Suggested severity definitions are:

* Critical: Prevents execution of major supported validation workflows, causes unrecoverable service failure under defined validation conditions, or exposes a Critical baseline security concern.

* High: Causes failure of an important supported workflow, significant inconsistency in observable outputs, or serious operational instability requiring correction.

* Medium: Causes partial workflow failure, inconsistent behavior with a defined workaround, or degraded observability that does not prevent completion of the validation cycle.

* Low: Minor functional, documentation, logging, or usability issue that does not materially affect validation outcomes.

Defects will be reviewed during execution. Critical defects must be resolved or formally documented with an approved disposition before completion of the validation cycle. Non-critical findings may remain open if their severity, impact, and recommended follow-up are documented in the final validation report.

## 17\. Test Deliverables

The following deliverables will be produced as part of the validation cycle:

* approved Master Test Plan

* test case inventory or validation checklist

* test execution records

* automated regression results

* baseline security validation summary

* controlled load test summary

* defect register

* evidence package containing logs, metrics, reports, and references

* final validation report

* residual risk summary

* production-readiness follow-up recommendations

Deliverables must be stored in an agreed internal location with appropriate access control. Evidence should be organized so that reviewers can trace validation conclusions back to executed test cases, logs, metrics, reports, and defect records.

The final validation report will summarize execution status, pass/fail results, blocked or not executed tests, critical findings, resolved defects, remaining non-critical findings, load test observations, baseline security validation results, and recommended next steps.

## 18\. Roles and Responsibilities

The following roles are expected to participate in the validation cycle:

### 18.1 Test Lead or Validation Owner

Responsible for planning, coordinating, tracking, and reporting the validation effort. The validation owner maintains execution status, coordinates defect review, confirms evidence collection, and prepares the final validation report.

### 18.2 QA or Test Engineers

Responsible for executing test cases, recording results, collecting evidence, creating defect records, performing retesting, and supporting regression execution.

### 18.3 Engineering Team

Responsible for supporting environment setup, investigating defects, implementing fixes, reviewing logs and metrics, and supporting retest activities.

### 18.4 DevOps or Infrastructure Team

Responsible for deployment support, environment availability, configuration management, service monitoring, log access, metrics availability, and controlled load execution support.

### 18.5 Security Reviewer

Responsible for supporting baseline security validation, reviewing security findings, classifying baseline security risks, and confirming resolution or documentation of Critical findings.

### 18.6 Project or Product Stakeholders

Responsible for reviewing validation outcomes, accepting residual non-critical findings where appropriate, and approving production-readiness follow-up recommendations.

## 19\. Test Evidence and Reporting

Evidence collection is a core requirement of this validation cycle. Each executed test case should include sufficient evidence to support the recorded result. Evidence may include screenshots, command outputs, API responses, logs, metrics, test reports, transaction identifiers, block references, state root records, defect references, commit references, pull request references, or release notes.

Evidence must be organized in a way that supports traceability. Each evidence item should map to a test case, defect, validation scenario, or report section. Where evidence contains sensitive data, access must be restricted and sensitive values must be redacted before broader distribution.

The validation report will include:

* validation summary

* environment summary

* scope confirmation

* test execution summary

* defect summary

* baseline security validation summary

* controlled load validation summary

* evidence references

* residual risks

* recommended production-readiness follow-up

* final conclusion

The report should clearly distinguish between completed validation activities and future production-readiness activities.

## 20\. Risks and Mitigation

The validation effort may be affected by environment instability, incomplete test data, limited observability, configuration differences, dependency issues, service restarts, delayed defect resolution, or insufficient load test controls.

Mitigation actions include:

* performing environment readiness checks before execution;

* confirming logs and metrics are available;

* preparing controlled test data before execution;

* tracking blocked tests separately;

* prioritizing Critical and High defects;

* rerunning impacted tests after fixes;

* maintaining clear evidence references;

* documenting residual non-critical findings;

* separating testnet validation conclusions from production-readiness conclusions.

Where risks cannot be fully mitigated within the validation cycle, they will be documented in the final validation report with recommended follow-up actions.

## 21\. Communication and Status Reporting

Validation status will be communicated to relevant stakeholders during the test cycle. Reporting may include execution progress, passed tests, failed tests, blocked tests, defect trends, Critical findings, environment issues, security findings, load test observations, and evidence collection status.

Status reporting should be concise and evidence-based. Key decisions, such as acceptance of residual non-critical findings or resumption after suspension, should be recorded for traceability.

The final validation report will serve as the formal summary of the testnet validation cycle.

## 22\. Change Control

Changes to this MTP should be reviewed and approved by the validation owner and relevant stakeholders. Changes may include scope updates, test approach changes, entry or exit criteria changes, environment changes, load test condition changes, or reporting requirements.

If changes occur during execution, the final validation report should identify material deviations from the approved plan and explain their impact on validation conclusions.

## 23\. Assumptions and Constraints

This MTP is based on the following assumptions:

* Sundial staging environment and Sundial testnet are available for validation.

* Supported testnet workflows are defined for execution.

* Exposed system interfaces are available for testing.

* Required logs and metrics are accessible.

* Test data and validation credentials are available where needed.

* Defects can be recorded in a private defect register.

* Regression tests can be executed repeatably.

* Controlled load testing can be performed within agreed limits.

Constraints may include environment availability, test data limitations, service access restrictions, monitoring limitations, dependency availability, execution windows, or load test limits. Any constraint that affects validation coverage or conclusions will be documented in the final validation report.

## 24\. Traceability

Traceability will be maintained between requirements, validation scenarios, test cases, execution results, defects, and evidence. Each major validation area should map to one or more test cases or checklist items.

Traceability will support review of whether planned functional, integration, regression, baseline security, and controlled load validation activities were completed. It will also support impact analysis when defects are identified or fixes are applied.

A traceability matrix may include the following fields:

* validation area

* test objective

* test case ID

* test type

* execution status

* defect ID, if applicable

* evidence reference

* final disposition

## 25\. Final Validation Conclusion

At the end of the validation cycle, the validation owner will prepare a final conclusion based on execution results, evidence, defects, baseline security findings, controlled load observations, and residual risks.

The conclusion will state whether the Sundial staging environment and Sundial testnet met the defined exit and acceptance criteria within the tested scope. The conclusion will also identify any non-critical findings that remain open, their impact, and recommended follow-up.

A successful validation conclusion means that Sundial services were deployed, operated, monitored, and exercised consistently through defined validation scenarios, and that no unresolved Critical defects or Critical baseline security findings remain within the tested scope.

This conclusion should be interpreted as completion of the defined testnet validation cycle only. Further production-readiness reviews, hardening, audit preparation, external audit activities, production operator preparation, and mainnet launch assessments remain separate workstreams.

## 26\. Approval

Approval of this MTP indicates agreement with the defined validation scope, objectives, approach, acceptance criteria, deliverables, and reporting expectations for the Sundial staging environment and Sundial testnet.

Approvers may include representatives from quality assurance, engineering, DevOps, security, and project leadership.

## Appendix A: Example Test Case Format

Each test case should be documented using a consistent format:

* Test Case ID

* Test Case Name

* Validation Area

* Objective

* Preconditions

* Test Data

* Execution Steps

* Expected Result

* Actual Result

* Status

* Defect Reference

* Evidence Reference

* Tester

* Execution Date

## Appendix B: Example Validation Areas

The following validation areas may be used to organize test cases:

* ENV: Environment readiness

* DEP: Deployment and startup

* HLT: Health and readiness checks

* API: API behavior

* TXN: Transaction submission

* MEM: Mempool processing

* BLK: Block construction

* L1E: L1 event ingestion

* WRK: Background worker execution

* STA: Local execution and state updates

* SRT: State root generation and tracking

* OBS: Observability

* NEG: Negative input handling

* REG: Automated regression testing

* SEC: Baseline security validation

* LOD: Controlled load validation

## Appendix C: Evidence Package Index

The validation evidence package should include references to:

* environment readiness records

* deployment logs

* health check outputs

* API request and response samples

* transaction submission records

* mempool processing logs

* block construction evidence

* L1 event ingestion evidence

* background worker logs

* local state update evidence

* state root records

* observability dashboards or metric exports

* negative test results

* automated regression test reports

* baseline security validation reports

* controlled load test reports

* defect register export

* final validation report

## Appendix D: Residual Risk Summary Format

Residual risks should be documented using a consistent format:

* Risk ID

* Description

* Affected Component

* Severity

* Impact

* Current Status

* Recommended Follow-Up

* Owner

* Target Resolution Window

Residual risks should be limited to non-critical findings unless an approved exception is documented. Any remaining risk must be reviewed in the context of production-readiness follow-up and should not be treated as a substitute for later hardening, audit, or mainnet-preparation activities.