---
name: checkout-payment-decision
description: Use when analyzing ecommerce or local-services checkout payment data, diagnosing payment-method ranking or default rules, drafting payment decision rules, or designing payment experiments.
allowed-tools: [bash, read_file, write_file, str_replace, ls, glob, grep]
---

# Checkout Payment Decision

## Use This Skill For

- Checkout payment success-rate analysis.
- Payment method default-selection and ranking diagnosis.
- Payment rule conflict review.
- Structured payment rule drafting.
- A/B experiment, monitoring, and rollback planning.

## Required Discipline

Separate facts, assumptions, and recommendations. Treat rule changes as experiment candidates unless the user provides decisive launch evidence. Do not claim production safety without review and monitoring.

## Standard Workflow

1. Confirm objective and constraints. Default objective order is payment success rate, user experience, payment cost, then risk control.
2. Check data health: time window, sample size, denominator consistency, missing fields, duplicates, segment coverage, and experiment contamination.
3. Decompose the funnel: exposure, availability, default selection, click or selection, payment initiation, payment success, payment failure, and final checkout completion.
4. Segment by method, user type, order amount, platform, region or city, offer exposure, channel, and failure code when available.
5. Diagnose whether the issue is caused by ranking, default selection, availability, channel stability, offer interference, user preference mismatch, risk control, or fallback gaps.
6. Draft rules with scope, conditions, ranking, default method, fallback, conflicts, exceptions, monitoring, and rollback.
7. Produce a decision package.

## Metrics

| Metric | Definition |
| --- | --- |
| exposure_rate | method_exposures / checkout_views |
| availability_rate | eligible_method_exposures / checkout_views |
| default_selection_rate | default_method_exposures / method_exposures |
| selection_rate | method_clicks_or_selections / method_exposures |
| initiation_rate | payment_initiations / checkout_views |
| method_initiation_rate | method_payment_initiations / method_exposures |
| success_rate | payment_successes / payment_initiations |
| method_success_rate | method_payment_successes / method_payment_initiations |
| completion_rate | payment_successes / checkout_views |
| failure_rate | payment_failures / payment_initiations |
| cost_per_success | payment_cost / payment_successes |

Always name denominators. If input data uses different denominators, state the mismatch.

## Data Health Checks

- Minimum viable time window: at least one complete business cycle unless the user is investigating an incident.
- Minimum sample warning: flag segments with fewer than 1,000 initiations or fewer than 100 successes.
- Check whether default-selection rate is based on exposure, click, initiation, or success.
- Check whether success rate excludes user cancellation, timeout, duplicate attempts, or risk rejection.
- Check whether retry attempts are counted as separate attempts or deduplicated by order.
- Check whether rule changes or experiments overlapped the observation window.

## Rule Diagnosis Patterns

- High default-selection rate and low method success rate: default may be over-promoting a weak method.
- Low exposure and high success: method may deserve higher ranking or broader eligibility.
- High click but low initiation: UX, offer, or redirection friction may exist.
- High initiation but low success: channel, issuer, risk, or failure-code concentration is likely.
- Success gains with cost spike: require cost guardrail or segment-specific rollout.
- Ranking rule with many exceptions: check precedence, shadowing, and stale fallback logic.

## Decision Package Format

Use this structure for substantial answers:

1. Executive Summary
2. Data and Metric Assumptions
3. Key Findings
4. Rule Diagnosis
5. Recommended Rules
6. Structured Rule Draft
7. SQL or Metric Templates
8. Experiment Plan
9. Monitoring and Rollback
10. Risks and Open Questions

## Structured Rule Template

```yaml
rule_name: checkout_payment_rule_v1
objective:
  primary: payment_success_rate
  secondary:
    - user_experience
    - payment_cost
    - risk_control
scope:
  business: ecommerce_checkout
  platforms: []
  segments: []
conditions:
  user_segment: null
  order_amount_range: null
  region: null
ranking:
  default_method: null
  ordered_methods: []
eligibility:
  include_methods: []
  suppress_methods: []
fallback:
  if_default_unavailable: use_last_successful_method
  if_channel_degraded: promote_next_best_available_method
conflict_resolution:
  precedence:
    - risk_blocks
    - method_availability
    - incident_degradation
    - user_last_successful_method
    - business_ranking
experiment:
  unit: user_id
  primary_metric: payment_success_rate
  guardrails:
    - checkout_exit_rate
    - payment_cost_per_success
    - risk_reject_rate
monitoring:
  primary_metric: payment_success_rate
  guardrails:
    - payment_cost_per_success
    - checkout_exit_rate
rollback:
  trigger: success_rate_drop >= 0.5pp for 2 consecutive hours
```

Label this as a draft unless the user provides a production schema.

## SQL Templates

```sql
-- Payment method funnel by day and method.
SELECT
  dt,
  payment_method,
  COUNT(DISTINCT checkout_id) AS checkout_views,
  SUM(method_exposed) AS method_exposures,
  SUM(is_default_method) AS default_method_exposures,
  SUM(method_selected) AS method_selections,
  SUM(payment_initiated) AS payment_initiations,
  SUM(payment_success) AS payment_successes,
  SUM(payment_failed) AS payment_failures,
  1.0 * SUM(is_default_method) / NULLIF(SUM(method_exposed), 0) AS default_selection_rate,
  1.0 * SUM(payment_success) / NULLIF(SUM(payment_initiated), 0) AS success_rate,
  1.0 * SUM(payment_success) / NULLIF(COUNT(DISTINCT checkout_id), 0) AS completion_rate
FROM checkout_payment_events
WHERE dt BETWEEN '${start_date}' AND '${end_date}'
GROUP BY dt, payment_method
ORDER BY dt, payment_method;
```

```sql
-- Failure-code concentration by method and channel.
SELECT
  payment_method,
  payment_channel,
  failure_code,
  COUNT(*) AS failures,
  1.0 * COUNT(*) / NULLIF(SUM(COUNT(*)) OVER (PARTITION BY payment_method, payment_channel), 0) AS failure_share
FROM payment_attempts
WHERE dt BETWEEN '${start_date}' AND '${end_date}'
  AND payment_status = 'failed'
GROUP BY payment_method, payment_channel, failure_code
ORDER BY failures DESC;
```

## Experiment Template

- Hypothesis: state why the rule should improve the primary metric.
- Unit: prefer user-level assignment for checkout UX changes.
- Primary metric: payment success rate or checkout completion rate.
- Guardrails: checkout exit rate, payment cost per success, risk reject rate, refund rate, customer support contacts.
- Segments: platform, payment method, user type, order amount, region.
- Rollback: define threshold, duration, and owner.
