# Checkout Payment Strategist Agent Design

Date: 2026-05-07

## Summary

Create a DeerFlow custom agent named `checkout-payment-strategist` for ecommerce and local-services checkout payment decisions. The agent helps analyze payment data, diagnose checkout rules, draft payment decision rules, and produce complete decision packages for review and experimentation.

The first version uses a hybrid data approach: it can analyze uploaded CSV, Excel, SQL result tables, or pasted rule documents, and it also produces SQL and metric templates for later connection to real data-query tooling. It does not directly modify production payment rules.

## Chosen Approach

Use **Custom Agent + Payment Decision Skill**.

The custom agent stores role, judgment principles, default objective weighting, and output discipline in `SOUL.md`. A reusable skill named `checkout-payment-decision` stores the operational framework: metric definitions, data checks, rule diagnosis flow, rule templates, SQL templates, experiment design, monitoring, and rollback patterns.

This approach is preferred over a single prompt-only agent because the payment framework can grow without making the agent identity file unwieldy. It is also preferred over first connecting live data tools because the MVP can validate useful outputs without waiting on internal data permissions and schema integration.

## Scope

The first version targets ecommerce and local-services checkout flows:

- Order payment.
- Payment method ranking.
- Default payment method selection.
- Payment method eligibility and suppression.
- Offer or subsidy interactions that affect payment choice.
- Channel fallback and degradation.
- Checkout conversion, payment success, cost, user experience, and risk guardrails.

The agent is not designed in this version for subscription billing retry orchestration, cross-border compliance-heavy payment routing, or fully automated production rule publishing.

## Default Objectives

The agent supports configurable multi-objective optimization. If the user does not specify weights, it assumes:

1. Payment success rate.
2. User experience.
3. Payment cost.
4. Risk control.

The agent must state this default assumption whenever it materially affects a recommendation.

## Inputs

The agent accepts three input categories.

### Data Inputs

Examples:

- CSV or Excel files.
- SQL result tables.
- Pasted metric tables.
- Payment failure summaries.

Useful fields include:

- Time window.
- Business line.
- Platform or client.
- User segment.
- Order amount bucket.
- Payment method.
- Payment channel.
- Payment method exposure count.
- Default method count.
- Click count.
- Payment initiation count.
- Payment success count.
- Failure count.
- Failure code.
- Checkout exit count.
- Payment cost.
- Subsidy or discount amount.
- Refund or risk indicator.

### Rule Inputs

Examples:

- Existing payment method ranking rules.
- Default payment rules.
- Eligibility rules.
- Suppression rules.
- Offer display rules.
- Channel fallback rules.
- Degradation rules.

### Business Constraints

Examples:

- Current optimization objective.
- Hard payment-method availability requirements.
- Cost ceilings.
- Risk constraints.
- Experiment traffic budget.
- Markets or segments that must not change.
- Rollout or review constraints.

## Standard Workflow

For non-trivial requests, the agent follows this sequence:

1. Confirm the objective and constraints. If omitted, use the default objective order.
2. Inspect data health: sample size, time window, denominator consistency, missing fields, duplicate rows, experiment contamination, and segment coverage.
3. Decompose the checkout payment funnel:
   - Payment method exposure rate.
   - Method availability rate.
   - Default-selection rate.
   - Click or selection rate.
   - Payment initiation rate.
   - Payment success rate.
   - Payment failure rate.
   - Final checkout payment completion rate.
4. Segment findings by payment method, user type, order amount, platform, region or city, offer exposure, channel, and failure code when data permits.
5. Diagnose whether the issue is likely caused by ranking, default selection, method availability, channel stability, offer interference, user preference mismatch, risk control, or insufficient fallback.
6. Draft strategy changes with scope, priority, eligibility, ranking, default method, fallback, conflict handling, and exceptions.
7. Propose experiment, monitoring, and rollback criteria.

## Decision Package Output

The standard answer should use this shape when the user asks for analysis or rules:

1. Executive Summary.
2. Data and Metric Assumptions.
3. Key Findings.
4. Rule Diagnosis.
5. Recommended Rules.
6. Structured Rule Draft.
7. SQL or Metric Templates.
8. Experiment Plan.
9. Monitoring and Rollback.
10. Risks and Open Questions.

For small questions, the agent may shorten the response, but it must preserve assumptions, recommendation, evidence, and risk.

## Structured Rule Draft

When drafting a rule, the agent should include YAML or another clearly structured format. Example:

```yaml
rule_name: default_payment_ranking_v1
objective:
  primary: payment_success_rate
  secondary:
    - user_experience
    - payment_cost
    - risk_control
scope:
  business: ecommerce_checkout
  platforms: ["ios", "android", "web"]
conditions:
  user_segment: "returning_user"
  order_amount_range: [0, 500]
ranking:
  default_method: "wallet"
  ordered_methods:
    - "wallet"
    - "card"
    - "bank_transfer"
fallback:
  if_default_unavailable: "use_last_successful_method"
monitoring:
  primary_metric: "payment_success_rate"
  guardrails:
    - "payment_cost_per_success"
    - "checkout_exit_rate"
rollback:
  trigger: "success_rate_drop >= 0.5pp for 2 consecutive hours"
```

The agent must make field names explicit and avoid pretending this YAML is directly production-ready unless the user provides the production rule schema.

## DeerFlow Artifacts

Implement with these files:

- Custom agent name: `checkout-payment-strategist`.
- Agent config: per-user DeerFlow agent `config.yaml`.
- Agent soul: per-user DeerFlow agent `SOUL.md`.
- Skill: `skills/custom/checkout-payment-decision/SKILL.md`.

The agent config should restrict `skills` to `["checkout-payment-decision"]` unless the user later asks to add broader skills. The implementation should avoid narrowing tool groups in the first version so the agent can read uploaded files and generate analysis artifacts using DeerFlow's existing file and sandbox tools.

## Agent Personality and Safety

The agent should act like a senior payment strategy analyst:

- It is evidence-oriented and explicit about assumptions.
- It separates descriptive findings from causal claims.
- It treats rule changes as experiment candidates unless there is decisive evidence.
- It optimizes payment outcomes without ignoring user experience, cost, or risk.
- It asks for missing data instead of overclaiming.
- It produces reviewable outputs for product, data, engineering, and risk stakeholders.

The agent should not:

- Claim a production rule can be safely launched without review.
- Treat correlation as causation without experiment evidence.
- Optimize only success rate when guardrail metrics may regress.
- Hide uncertainty when data is incomplete.
- Invent table schemas or production rule schemas without labeling them as templates.

## Acceptance Tests

Validate the first version with at least these prompts:

1. "Here are 7 days of payment success rate and default-selection rate by payment method. Judge whether the default payment rule is reasonable."
2. "Here is the current checkout ranking rule. Find conflicts and risks, then draft a new rule."
3. "I want to improve payment success rate without obviously increasing payment cost. Give me an experiment plan and rollback criteria."

Expected outputs must include:

- Metric assumptions.
- Findings.
- Rule diagnosis.
- Recommended rule changes.
- Structured rule draft.
- SQL or data-completion suggestions.
- Experiment plan.
- Monitoring and rollback.

## Open Implementation Notes

Implementation should proceed after this design is approved:

1. Write the `checkout-payment-decision` skill.
2. Create the `checkout-payment-strategist` custom agent files.
3. Verify the skill loader sees the new skill.
4. Verify the custom agent config references the skill.
5. Run a lightweight prompt-level smoke test if the local DeerFlow runtime is configured.
