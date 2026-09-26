# JSM Customer Support Request Lifecycle Model – Proposal for Review

**Status:** In Review  
**Scope:** Jira Service Management Cloud – Customer Support  
**Owner:** Customer Support / Atlassian Platform Governance  
**Purpose:** Proposed target model for customer request lifecycle, customer communication, internal handling visibility and SLA behaviour.

> **Proposal for review**  
> This page describes a proposed target lifecycle for JSM Customer Support requests.  
> Please use inline comments to challenge the model, suggest changes or identify operational cases that are not covered.
> 
> Workflow, SLA and automation configuration should be finalised only after the lifecycle model is agreed.

## 1. Proposal summary

The initial status proposal provides a good basis for the Customer Support workflow. Looking at the full lifecycle together, I think it would be useful to separate a few different concepts that are currently represented through statuses.

The current proposal includes, for example:

| Dimension | Examples |
| --- | --- |
| **Processing stage** | Triage, Development in Progress, Pending Release |
| **Responsibility / next action** | Waiting for Customer, Waiting for IPS |
| **Milestone / event** | Escalated to Development, Fix Identified |
| **Final outcome** | Resolved, Closed, Cancelled / Duplicate |

The intention of this proposal is not to remove useful information, but to represent it in the place where it is most meaningful.

For example:

- the **JSM Status** can show what is happening with the request now or who needs to act;
- the **Handling Stage** can show where the request is in the overall solution process;
- linked **Product / Development / Delivery items** can provide detailed internal execution status;
- **Capability Outcome / Resolution** can record how the request or assessment was concluded.

This should help keep the workflow easier to understand and maintain while preserving the information needed by both IPS and the customer.

### Proposed model

Separate the information into distinct layers:

| Layer | Answers | Visibility |
| --- | --- | --- |
| **JSM Status** | What is happening with the request now / who needs to act? | IPS + customer |
| **Handling Stage** | Where is the request in the overall solution process? | IPS + customer |
| **Linked Product / Development / Delivery item** | What detailed internal work is taking place? | IPS; selected information communicated to customer |
| **Capability Outcome** | What was decided about a requested capability or feature? | IPS + communicated to customer |
| **Resolution** | How was the Jira request finally completed? | IPS / reporting |

This keeps the JSM workflow understandable without losing visibility of the actual processing stage.

---

# 2. Proposed JSM Status Model

| Internal status | Customer-facing status | Meaning |
| --- | --- | --- |
| **New** | **Received** | Request has been submitted but not yet reviewed |
| **Triage** | **Under Review** | IPS is assessing the request, impact, ownership and required information |
| **In Progress** | **In Progress** | IPS is actively progressing or coordinating the request |
| **Waiting for Customer** | **Action Required from You** | Further meaningful progress is blocked until required customer information or action is received |
| **Pending Delivery / Deployment** | **Preparing Delivery** | The solution/change is known or ready but still requires a controlled delivery step |
| **Ready for Customer Validation** | **Ready for Your Validation** | The delivered result is available and customer validation is required |
| **Needs Further Adjustment** / **Validation Not Successful** | Same | Customer validation was not successful and IPS needs to reassess the required next action |
| **Closed** | **Resolved / Completed** | Processing of the request has been completed |
| **Cancelled** | **Cancelled** | The request was withdrawn or is no longer required |

Atlassian Cloud allows the status shown to the customer to have a different name from the underlying workflow status and also allows several internal statuses to be presented under the same customer-facing status where required. [Atlassian Support](https://support.atlassian.com/jira-service-management-cloud/docs/customize-the-workflow-statuses-for-a-request-type/?utm_source=chatgpt.com)

### Naming decision still required

Two alternatives are proposed for unsuccessful customer validation:

**Option A – Needs Further Adjustment**  
More customer-friendly and suitable for bugs, new functionality, configuration and other types of delivery.

**Option B – Validation Not Successful**  
More process-oriented and explicit that customer validation did not complete successfully.

Both IPS and the customer could see the same name.

**Preferred option for review:** `Needs Further Adjustment`

---

# 3. Customer-visible Handling Stage

## Purpose

`Handling Stage` complements the workflow status.

The two should answer different questions:

**Status**

> What is happening with my request right now / who needs to act?

**Handling Stage**

> Where is my request in the overall handling and solution process?

The target is for `Handling Stage` to be visible to both IPS and the customer but maintained by IPS or controlled automation rather than selected by the customer.

## Proposed values

| Handling Stage | Meaning |
| --- | --- |
| **Investigation** | IPS is analysing, reproducing, troubleshooting or clarifying the request |
| **Product Assessment** | Product capability, feasibility or roadmap evaluation is required |
| **Development** | Development work is required or underway |
| **Delivery / Deployment** | The agreed solution/change is progressing through controlled delivery |
| **Customer Validation** | The delivered result is awaiting customer validation |

### Example

Development is working on a request:

**Status:** In Progress  
**Handling Stage:** Development

Development then requires logs from the customer and cannot continue:

**Status:** Waiting for Customer  
**Handling Stage:** Development

The customer provides the requested information:

**Status:** In Progress  
**Handling Stage:** Development

The request therefore does not lose its Development context simply because the next action temporarily belongs to the customer.

### Why not `Previous Status`

A `Previous Status` field is not proposed.

Knowing that the previous Jira status happened to be `In Progress` does not tell us whether the actual request was in Investigation, Product Assessment, Development or Delivery.

`Handling Stage` records the business meaning we need rather than Jira workflow history.

### Implementation note

The required portal behaviour must be tested in the IPS tenant before implementation.

Atlassian documents that custom fields added to a request form can subsequently be visible on the customer request when they contain a value. Atlassian also notes that using **Use preset value and hide from portal** hides that field from the portal, so that configuration would not meet this requirement. [Atlassian Support](https://support.atlassian.com/jira-service-management-cloud/docs/customize-the-fields-of-a-request-type/?utm_source=chatgpt.com)

The implementation therefore needs to achieve:

> **Customer can see Handling Stage after submission, but IPS/automation controls its value.**

This should be validated with one request type before the model is rolled out.

---

# 4. Statuses proposed for removal

## Escalated to Development

**Remove as workflow status.**

Escalation is a milestone/event rather than a stable request state.

Instead:

**Status:** In Progress  
**Handling Stage:** Development  
**Linked item:** DEV work item

The linked Development item becomes the source of truth for detailed technical execution.

---

## Development in Progress

**Remove from JSM workflow.**

Development work should be tracked in the Development Jira space.

The Development item can contain its own lifecycle such as:

`To Do → In Progress → Testing → Done`

There is no benefit in duplicating that workflow in JSM.

---

## Fix Identified

**Remove as workflow status.**

Finding the cause or solution is a milestone.

If more work is required:

**Status:** In Progress

If the solution is ready but must still be delivered:

**Status:** Pending Delivery / Deployment

---

## Waiting for IPS

**Remove.**

Almost every active customer request is, in some sense, waiting for IPS.

The status therefore does not provide enough information to justify its existence.

---

## Pending Internal Dependency

**Do not introduce initially.**

If Support is waiting for Product, Development, Delivery or another IPS activity, the customer request can remain:

**Status:** In Progress

while `Handling Stage` and the linked work item provide the context.

For example:

**Status:** In Progress  
**Handling Stage:** Development

A separate internal-dependency status should only be introduced later if there is a demonstrated requirement for:

- a dedicated queue;
- specific escalation control;
- reporting of internally blocked time;
- another operational process that cannot be supported by the proposed model.

---

## Monitoring

**Remove.**

No distinct post-delivery monitoring process has currently been identified.

Once the result is delivered, the request moves to customer validation.

If the result works, the request is closed.

If further work is required, the request moves back into active processing.

---

## Reopened

**Do not use in the proposed model.**

The original reason for considering `Reopened` was useful: we need to distinguish a normal work-in-progress request from one where the customer has already tested the delivered result and says that it is not acceptable.

A more explicit status does this better:

**Needs Further Adjustment**

or

**Validation Not Successful**

This communicates the actual business meaning to both IPS and the customer rather than exposing an internal Jira concept such as `Reopened`.

---

## Resolved

**Do not introduce as a separate status initially.**

With a dedicated customer validation step, the lifecycle can be:

**Ready for Customer Validation → Closed**

when the customer accepts the result.

The customer-facing name for `Closed` can be `Resolved` or `Completed`.

A separate internal `Resolved` state should only be introduced if there is a specific process or reporting requirement for it.

---

# 5. Pending Delivery / Deployment

`Pending Delivery / Deployment` should not be limited to software bug fixes or hotfixes.

### Definition

> The solution or requested change is ready or agreed, but a controlled implementation step is still required before it becomes available in the target environment.

This may include:

- product release;
- hotfix or patch;
- MDP/package;
- upgrade;
- configuration change;
- migration;
- deployment;
- another coordinated delivery activity.

This status therefore applies to defects, changes, functionality and other solution types.

---

# 6. Customer Validation

After the agreed solution, functionality or change has been delivered, the request moves to:

**Status:** Ready for Customer Validation  
**Customer-facing:** Ready for Your Validation  
**Handling Stage:** Customer Validation

The customer then provides one of two outcomes.

Atlassian JSM Cloud allows selected workflow transitions to be exposed directly in the customer portal, meaning the customer can perform the validation transition themselves. [Atlassian Support](https://support.atlassian.com/jira-service-management-cloud/docs/show-a-workflow-transition-in-the-portal/?utm_source=chatgpt.com)

## Validation successful

Possible customer action:

**Accept / This meets my needs**

Result:

**Ready for Customer Validation → Closed**

No intermediate `Accepted by Customer` status is required.

Acceptance is the transition decision.

`Closed` is the resulting stable state.

---

## Validation unsuccessful

Possible customer action:

**Further adjustment required**

Result:

**Ready for Customer Validation → Needs Further Adjustment**

Alternative naming under review:

**Ready for Customer Validation → Validation Not Successful**

This state means:

> The delivered result has been reviewed by the customer but does not yet fully satisfy the expected outcome. IPS must reassess what further action is required.

It applies broadly, for example where:

- a reported problem remains;
- a delivered feature does not fully meet the agreed requirement;
- a configuration does not produce the expected result;
- part of the agreed functionality is missing;
- another change is required;
- the customer identifies an issue during validation.

This is deliberately broader than a bug-specific status such as `Issue Still Present`.

## Reassessment after unsuccessful validation

`Needs Further Adjustment` should remain visible long enough for Support to understand that customer validation failed and determine the next step.

After reassessment:

**Needs Further Adjustment → In Progress**

and `Handling Stage` is updated accordingly.

Examples:

**In Progress + Investigation**

or

**In Progress + Development**

or

**In Progress + Delivery / Deployment**

This also allows failed customer validations to be reported separately from normal active requests.

---

# 7. No response to customer validation

Requests should not remain indefinitely in `Ready for Customer Validation`.

The final operational model should define:

1. validation period;
2. reminder(s);
3. final notification;
4. automatic or manual closure after no response.

Example principle:

> If the customer does not respond within the defined validation period after appropriate reminder(s), IPS may close the request on the basis of the delivered result.

The exact timeframe should be agreed together with the SLA model rather than embedded into this proposal.

---

# 8. Product and functionality requests

Customer Support may receive requests for functionality that does not currently exist.

These requests require an explicit distinction between the **support request lifecycle** and the **future product lifecycle**.

A JSM request should not remain open for months or years only because the requested capability may potentially be delivered later.

Once the current assessment has been completed and communicated, the support request should be closed.

## Proposed capability outcomes

| Situation | JSM status | Capability Outcome | Longer-term tracking |
| --- | --- | --- | --- |
| Functionality is not available and **not planned** | Closed | **Not Planned** | Assessment recorded and communicated |
| Functionality may be useful but there is **no commitment or delivery date** | Closed | **Future Consideration** | Link/create JPD idea where appropriate |
| Functionality is **accepted for future roadmap / development** | Closed | **Planned / Future Roadmap** | Linked JPD / DEV / roadmap item becomes source of truth |

Example:

A customer requests Feature X.

Product assessment confirms that the capability is planned for future development, but no committed delivery date exists.

The customer request becomes:

**Status:** Closed  
**Capability Outcome:** Planned / Future Roadmap  
**Linked item:** JPD / DEV / roadmap item

Possible final customer communication:

> The requested capability has been reviewed and is planned for future product development. There is currently no committed delivery date. We are therefore completing this support request; future product work will be tracked separately.

Closing the JSM request does **not** mean abandoning the requested functionality.

It means that the current customer support / assessment process has been completed.

This principle is already consistent with the IPS STS capability model, where JPD or DEV work continues independently after the capability assessment itself has been completed. 260911\_atlassianknowledgebase-e…

---

# 9. Cancelled is not the same as Not Planned

`Cancelled` should represent cases such as:

- requester withdrew the request;
- request is no longer required;
- request was created by mistake;
- processing was deliberately stopped before reaching its normal outcome.

A request that was assessed and concluded as:

- Not Planned;
- Future Consideration;
- Planned / Future Roadmap;

is **not cancelled**.

The assessment was successfully completed. The business outcome simply differs.

---

# 10. Resolution and Capability Outcome

The detailed capability result should not automatically become a new global Jira Resolution value.

The existing IPS Resolution model already contains values including:

- Done;
- Won’t Do;
- Duplicate;
- Cannot Reproduce;
- Declined.

`Won’t Do` is already defined for a deliberate decision not to implement work. 260911\_atlassianknowledgebase-e…

IPS also already uses a more detailed Capability Assessment Result model in STS, including outcomes such as `On the roadmap`, `Linked to an existing JPD idea`, `Not planned` and `Not feasible`. 260911\_atlassianknowledgebase-e…

The proposed principle is therefore:

**Resolution**  
= high-level Jira completion classification

**Capability Outcome**  
= specific product/capability decision

This avoids unnecessary proliferation of global Jira Resolution values.

---

# 11. SLA principles

The workflow must represent the real state of the request.

Statuses should not be selected solely to stop or restart an SLA clock.

## Waiting for Customer

Use `Waiting for Customer` only where required customer action genuinely prevents further meaningful IPS progress.

If IPS can continue working while waiting for the answer:

**Status remains In Progress.**

If customer input genuinely blocks progress:

**Status becomes Waiting for Customer.**

Atlassian JSM Cloud supports separate SLA start, pause and finish conditions and explicitly identifies waiting for a customer response as a possible pause condition. [Atlassian Support](https://support.atlassian.com/jira-service-management-cloud/docs/set-up-sla-conditions/?utm_source=chatgpt.com)

## Ready for Customer Validation

The SLA behaviour at customer validation should be agreed based on what the specific SLA actually promises.

| SLA interpretation | Possible treatment |
| --- | --- |
| IPS commitment ends when the solution is made available | Stop SLA at Ready for Customer Validation |
| Customer confirmation forms part of resolution | Pause SLA while awaiting validation |
| Contract defines another point | Configure according to the agreed definition |

The workflow should not be distorted simply to achieve the desired SLA calculation.

---

# 12. Linked Product, Development and Delivery work

Detailed internal execution remains in the relevant Jira space.

The JSM request remains the customer-facing lifecycle and communication layer.

## JSM is responsible for

- customer communication;
- customer-facing status;
- Handling Stage;
- SLA;
- required customer action;
- customer validation;
- final request outcome.

## Linked internal work item is responsible for

- detailed technical execution;
- Development lifecycle;
- testing;
- Product discovery / prioritisation;
- delivery implementation;
- internal dependencies;
- detailed internal comments.

This follows the existing IPS principle of separating requester communication from deeper internal implementation work rather than duplicating execution workflows in JSM. 260911\_atlassianknowledgebase-e…

---

# 13. Customer communication automation

Automation should communicate **meaningful customer milestones**, not mirror every internal workflow transition.

Possible model:

| Event | Customer communication |
| --- | --- |
| Development work linked | Inform customer that additional Development work is required |
| Meaningful Development milestone | Optional progress update |
| Delivery work linked / created | Inform customer that the solution/change is being prepared for delivery |
| Delivery completed | Inform customer that the result is available for validation |
| Product assessment completed | Communicate the confirmed capability/roadmap outcome |
| Validation unsuccessful | Confirm receipt of feedback and that further assessment will take place |

Example when Development work is created:

> Additional technical work is required and has been passed to our Development team. We will continue to keep you updated through this request.

Example when delivery is completed:

> The solution/change is now available for validation. Please review the delivered result and confirm whether it meets the expected outcome.

Selected Development or Delivery status information may also be included where useful and appropriate for external communication.

Internal-only technical information should not be exposed automatically.

---

# 14. Proposed high-level lifecycle

### Standard request

**New → Triage → In Progress**

### Customer input required

**In Progress → Waiting for Customer → In Progress**

The `Handling Stage` remains unchanged unless the actual processing stage changes.

### Delivery required

**In Progress → Pending Delivery / Deployment → Ready for Customer Validation**

### Customer accepts

**Ready for Customer Validation → Closed**

### Customer requests further adjustment

**Ready for Customer Validation → Needs Further Adjustment → In Progress**

with the appropriate Handling Stage selected after reassessment.

### Product / capability assessment without immediate implementation

**In Progress → Closed**

with an appropriate `Capability Outcome`:

- Not Planned
- Future Consideration
- Planned / Future Roadmap

### Request withdrawn

**Active status → Cancelled**

---

# 15. Proposed target model at a glance

| JSM Status | Typical Handling Stage |
| --- | --- |
| New | — |
| Triage | Investigation |
| In Progress | Investigation / Product Assessment / Development / Delivery |
| Waiting for Customer | Keep existing Handling Stage |
| Pending Delivery / Deployment | Delivery / Deployment |
| Ready for Customer Validation | Customer Validation |
| Needs Further Adjustment | Customer Validation until reassessment |
| Closed | Final |
| Cancelled | Final |

The key principle is:

> **Status describes the current request state. Handling Stage preserves the wider solution context.**

This prevents the workflow from becoming a combination of every possible status and every possible technical stage.

---
