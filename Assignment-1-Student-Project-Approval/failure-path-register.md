# Failure Path Register: Assignment 1 - Student Project Approval & Allocation System

This document provides the formal Failure Path Register required by the specification, covering every failure mode across all process steps and participants in the final-year project approval and guide allocation lifecycle.

---

## 📊 Summary Matrix (F1 – F11)

| ID | Failure Point | Cause / Trigger | Solution & BPMN Construct | Recovery Strategy | Lane / Owner |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **F1** | Proposal Validation | Missing mandatory fields, no abstract, invalid roll numbers | Exclusive Gateway (`Gateway_ValidationAttempts`) after automated validation task | Return proposal with itemized error list; student resubmits. A loop counter limits retries to 2 attempts, after which the case is escalated to the Coordinator. | Project Management System / Coordinator |
| **F2** | Team Rule Check | Team size > 4, or a student already belongs to another registered team | Exclusive Gateway (`Gateway_ValidationCheck`) routing to Error End Event (`EndEvent_InvalidTeam`) | Immediate hard rejection at validation; system notifies students to dissolve/re-form team. Terminates with BPMN Error End Event `"Invalid team"`. | Project Management System |
| **F3** | Duplicate / Plagiarised Topic | Similarity score $\ge 25\%$ against archived historical departmental projects | Exclusive Gateway (`Gateway_SimilarityCheck`) routing to Coordinator User Task (`Task_ReviewDuplicateTopic`) | Coordinator reviews side-by-side comparison report. Coordinator either issues formal topic modification request (loops back to proposal task) or outright rejects. | Project Coordinator |
| **F4** | Proposal Submission Deadline | Student team fails to submit proposal before departmental deadline | Interrupting Timer Boundary Event (`BoundaryTimer_SubmissionDeadline`) on submission User Task | Timer interrupts submission, flags team as delinquent, and sends late-submission request to Coordinator for exceptional approval or closes as lapsed. | Student / Project Coordinator |
| **F5** | Committee Asks for Revision | Proposal scope too wide, methodology vague, or insufficient technical novelty | Exclusive Gateway (`Gateway_RevisionCycles`) with revision loop counter | Revision loop with a strict 7-day timer. Maximum 2 revision cycles allowed; on 2nd cycle committee is forced to approve or reject. | Review Committee / Student |
| **F6** | Committee Rejection | Project fundamentally unfeasible, out of academic syllabus, or lacks lab resources | Exclusive Gateway (`Gateway_TopicRejectionAttempts`) routing to retry loop or End Event | System transmits committee rejection remarks to team. Exactly 1 new topic submission is permitted (loops to start); if rejected again, process ends permanently as `"Rejected"`. | Review Committee / Student |
| **F7** | Review Committee Delays | Committee fails to complete review within departmental Service Level Agreement (SLA) | Non-interrupting Timer Boundary Event (`BoundaryTimer_SLA1_Reminder`) & Interrupting Escalation Timer (`BoundaryTimer_SLA2_HoDEscalation`) | 5-day non-interrupting timer fires automated reminder notice; 10-day interrupting timer automatically escalates evaluation authority directly to Head of Department (HoD). | Review Committee (incl. HoD) |
| **F8** | No Faculty Guide Available | Domain mismatch or all qualified faculty guides have reached maximum supervision load (quota capped) | Business Rule Task returns `'none'`, routed via Exclusive Gateway (`Gateway_GuideMatch`) to Coordinator User Task | Coordinator manually inspects roster, assigns co-guide / adjunct / external faculty guide, or places team on prioritized waiting list. | Project Coordinator / PMS |
| **F9** | Guide Declines or Remains Silent | Assigned faculty guide declines due to sabbatical/workload, or fails to respond within 3 days | Interrupting Timer Boundary Event (3 days) + Exclusive Gateway (`Gateway_GuideDeclineAttempts`) | System excludes non-responsive/declined guide, updates blacklist for this team, and re-triggers allocation rule. If 3 consecutive guides decline, escalates to Coordinator for manual intervention. | Faculty Guide / Coordinator |
| **F10** | Notification / System Outage | Department mail server down, SMTP failure, or database transaction deadlock | Error Boundary Event (`BoundaryError_NotificationFailure`) on Service/Send Task with retry gateway | System catches technical exception, automatically retries up to 3 times with exponential backoff; if still failing, falls back to manual Coordinator notification task. | Project Management System / Coordinator |
| **F11** | Post-Approval Team Dissolution or Guide Leave | Team member drops out, academic dispute occurs, or allocated guide takes medical/research leave | Interrupting Message Boundary Event (`BoundaryMessage_Withdrawal`) on Allocation Sub-Process with Compensation | Message boundary event interrupts allocation sub-process, throws compensation event releasing reserved guide quota slot, and resets team to allocation phase. | Student / Faculty Guide / PMS |

---

## 🔍 Detailed Failure Analysis & Architectural Rationale

### F1: Proposal Validation Failures
- **Detection**: Automated XML/JSON schema validation service checking mandatory fields (`projectTitle`, `abstract`, `studentList`, `domainId`, `rollNumbers`).
- **BPMN Construct**: Exclusive Gateway (`Gateway_ValidationAttempts`) connected to `Task_ValidateProposalForm`.
- **Handling**:
  - `validationAttempts < 2`: System generates error digest and routes token back to `Task_SubmitProposal`.
  - `validationAttempts >= 2`: Token routes to `Task_EscalateValidationToCoordinator` allowing human coordinator to arbitrate unusual student situations (e.g., transfer students).
- **Justification**: Prevents garbage data from burdening faculty reviewers while preventing automated deadlocks for legitimate edge cases.

### F2: Team Composition Rule Breach
- **Detection**: SQL/NoSQL uniqueness and constraint queries (`SELECT COUNT(*) FROM student_enrollment WHERE student_id IN (...)`).
- **BPMN Construct**: Exclusive Gateway routing to Error End Event (`EndEvent_InvalidTeam`).
- **Handling**: Throws BPMN error `ERR_INVALID_TEAM`. Process instance terminates immediately; students receive explicit reason codes (e.g., `"Roll 21CS042 already registered in Team 08"`).
- **Justification**: Departmental policy strictly forbids multi-team enrollment. Fail-fast error end event eliminates illegal duplicate entries immediately.

### F3: Duplicate / Plagiarism Match
- **Detection**: PMS integration with similarity detection engine (e.g., Turnitin or vector semantic search against departmental thesis repository).
- **BPMN Construct**: Exclusive Gateway checking `similarityScore >= 25%`, directing flow to Coordinator User Task `Task_ReviewDuplicateTopic`.
- **Handling**:
  - `duplicateDecision == 'MODIFY'`: Coordinator flags matching citations and loops back to student with modification notes.
  - `duplicateDecision == 'REJECT'`: Severe verbatim copying leads to immediate rejection end event (`EndEvent_PlagiarismRejected`).
- **Justification**: Plagiarism cannot be evaluated by algorithms alone; faculty coordinator review protects student academic integrity while ensuring originality.

### F4: Submission Deadline Breach
- **Detection**: Scheduled cron/timer matching departmental submission window close.
- **BPMN Construct**: Interrupting Timer Boundary Event (`BoundaryTimer_SubmissionDeadline`, `P14D`) attached to `Task_SubmitProposal`.
- **Handling**: Timer cancels active user task, flags timestamp, and routes case to Coordinator's desk for formal penalty assessment or closure as `"Proposal Lapsed"`.
- **Justification**: Standard user tasks wait indefinitely without a timer boundary event. Attaching a deadline timer guarantees process progression and enforces academic schedule integrity.

### F5: Review Committee Revision Loops
- **Detection**: Review committee marks proposal evaluation status as `REVISE`.
- **BPMN Construct**: Exclusive Gateway (`Gateway_RevisionCycles`) evaluating `revisionCycles < 2`.
- **Handling**:
  - Cycle 1: Student is given 7 days to address committee remarks via `Task_ReviseProposalContent`. Flow re-enters `Task_CommitteeEvaluation`.
  - Cycle 2+: If team fails to meet standards after 2 revision attempts, gateway forces token into final Rejection path (`Gateway_TopicRejectionAttempts`).
- **Justification**: Prevents infinite review ping-pong cycles between students and faculty committee, preserving academic calendar deadlines.

### F6: Committee Rejection with Second Chance
- **Detection**: Review committee rejects proposal as unfeasible or outside syllabus bounds.
- **BPMN Construct**: Exclusive Gateway (`Gateway_TopicRejectionAttempts`) tracking `topicAttemptCount`.
- **Handling**:
  - First rejection: Student team receives detailed feedback report and is granted 1 fresh topic proposal submission. Flow loops back to `Task_SubmitProposal`.
  - Second rejection: Hard rejection via `EndEvent_CommitteeRejected`.
- **Justification**: Gives teams pedagogical opportunity to pivot while bounding total attempts to prevent administrative overload.

### F7: Review Committee SLA Delays & HoD Escalation
- **Detection**: Elapsed time after token arrives in committee review queue.
- **BPMN Construct**:
  - Non-interrupting Timer Boundary Event (`BoundaryTimer_SLA1_Reminder`, `P5D`, `cancelActivity="false"`).
  - Interrupting Timer Boundary Event (`BoundaryTimer_SLA2_HoDEscalation`, `P10D`, `cancelActivity="true"`).
- **Handling**:
  - At day 5: Non-interrupting timer fires email reminder without stopping committee review.
  - At day 10: Committee task is interrupted and yanked to HoD desk for executive determination.
- **Justification**: Non-interrupting timers allow seamless nudges, while interrupting timers prevent student projects from stalling due to delinquent faculty reviewers.

### F8: Guide Capacity Saturation / Domain Mismatch
- **Detection**: Business Rule Task evaluates DMN decision table matching `domainId` against `faculty_profiles` where `active_projects < max_quota`.
- **BPMN Construct**: Business Rule Task (`Task_CheckGuideWorkload`) returning `'none'` followed by Exclusive Gateway (`Gateway_GuideMatch`).
- **Handling**: Bypasses automated allocation and hands control to Coordinator to allocate co-guides or external industrial advisors.
- **Justification**: Automated algorithms cannot negotiate cross-departmental co-guidance or external industry partnerships; human intervention is essential here.

### F9: Guide Non-Response or Refusal
- **Detection**: Guide explicitly clicks `DECLINE` or fails to accept within 3 days.
- **BPMN Construct**: Interrupting Timer Boundary Event (`BoundaryTimer_GuideSilent`, `P3D`) on `Task_GuideResponse` + Exclusive Gateway (`Gateway_GuideDeclineAttempts`).
- **Handling**: Blacklists declined guide from team's preference list, increments retry counter, and re-triggers business rule task. After 3 rejections, escalates to Coordinator.
- **Justification**: Faculty members frequently have sudden workload spikes or research commitments. Automated retries avoid stalling while loop caps prevent endless loops.

### F10: System Infrastructure / Mail Server Failure
- **Detection**: Network socket timeout, SMTP 5xx error, or database connection pool exhaustion.
- **BPMN Construct**: Error Boundary Event (`BoundaryError_NotificationFailure`) on external service task + retry gateway (`Gateway_NotificationRetries`).
- **Handling**: System retries up to 3 times with exponential backoff. If mail server remains offline, task drops into Coordinator queue for manual telephone/offline dispatch.
- **Justification**: Resilient workflow architectures must isolate external infrastructure failures from core business logic.

### F11: Post-Approval Withdrawal / Faculty Leave
- **Detection**: Team dissolution petition or faculty medical leave submission.
- **BPMN Construct**: Interrupting Message Boundary Event (`BoundaryMessage_Withdrawal`) on `SubProcess_GuideAllocation` triggering Compensation Intermediate Throw Event (`Event_TriggerCompensation`).
- **Handling**: Fires compensation handler `Task_CompensationReleaseGuideSlot`, decrementing guide's project count in the database, and returns team to guide allocation pool.
- **Justification**: Using BPMN 2.0 transaction compensation ensures transactional consistency and frees up scarce faculty guide slots for other teams.
