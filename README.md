# TrajectoryCI

**Evaluate the path, not just the destination.**

> Every major eval tool — LangSmith, Braintrust, Arize — scores the *output*. TrajectoryCI scores the *path*.

---

## The Problem Nobody Is Solving

Real agents don't make one LLM call. They take 15+ steps, call 6 tools, make branching decisions — and can produce a final output that looks correct even when everything in the middle was broken:

- Agent got the right answer through the **wrong path**
- It **skipped fraud_check** before processing a $5,000 payment — output said "success"
- It **hallucinated a product ID** that never appeared in search results
- It **called send_email with a customer ID** instead of an email address — right tool, wrong argument
- After you changed your prompt, **something broke in step 7** — but the final output still looks fine
- An attacker **injected instructions via a support ticket** and the agent issued a refund to the wrong account

Your eval said "pass." TrajectoryCI would have caught all of these.

---

## Quickstart

```bash
pip install trajectoryci
```

```python
from trajectoryci import TrajectoryTracer, TrajectorySpec, EvaluationEngine
from trajectoryci.validators import RequiredToolsValidator, ToolArgumentValidator, StepCostValidator

# 1. Define what "correct" looks like
spec = TrajectorySpec("refund_flow")
spec.add_step_validator(RequiredToolsValidator({"fraud_check"}, step_indices=[1]))
spec.add_step_validator(ToolArgumentValidator(
    tool_name="send_email",
    arg_validators={"to": lambda v: "@" in str(v)}  # must be a real email
))
spec.add_step_validator(StepCostValidator(max_cost_usd=0.02))
spec.set_max_steps(10)
spec.set_max_total_cost(0.10)

# 2. Wrap your agent
tracer = TrajectoryTracer(agent_name="refund_agent")
with tracer.run("Process refund for order #789") as trajectory:
    # ... your agent runs here ...
    trajectory.finish(final_output={"status": "resolved"})

# 3. Evaluate
engine = EvaluationEngine(verbose=True)
result = engine.evaluate(trajectory, spec)
# TrajectoryCI Evaluation: PASSED ✅
#   Steps: 4 total, 0 failed | Cost: $0.0072 | Duration: 163ms
```

Works with any framework — raw Anthropic/OpenAI calls, LangChain, LlamaIndex, or your own agent. Zero mandatory dependencies.

---

## What It Catches

### Basic Failures (5/5 in test suite)

| Failure | How It's Caught |
|---|---|
| Infinite loop (same tool, same args, repeated 6x) | `NoRepeatingStepsValidator` + `StepCountValidator` |
| Silent step skip (fraud_check bypassed, payment went through) | `RequiredToolsValidator` |
| Wrong tool arguments (customer ID passed instead of email) | `ToolArgumentValidator` |
| Cost explosion (context accumulation, $0.89 for 5-step run) | `StepCostValidator` + `TotalCostValidator` |
| Ghost success (right answer, wrong path — identity never verified) | `RequiredToolsValidator` + `set_min_steps` |

### Enterprise Failures (7/7 in test suite)

| Failure | Validator | Real Cost |
|---|---|---|
| KYC skipped before $250K wire transfer | `ToolOrderValidator` | $1M–$50M compliance fines |
| Agent deleted 50K records without human approval | `PrivilegeEscalationGuard` | $47K avg per incident |
| Attacker hijacked support agent via malicious ticket | `PromptInjectionDetector` | Full account takeover |
| SSN sent to external analytics API | `DataLeakageDetector` | GDPR: up to 4% global revenue |
| Recommended product ID that never existed in search | `HallucinationReferenceGuard` | $62M (IBM Watson precedent) |
| Shipping status polled 8x instead of using a webhook | `RateLimitBurstDetector` | System outage at scale |
| Patient records accessed without consent or audit log | `ComplianceAuditValidator` | $100K–$1.9M HIPAA fine |

---

## Enterprise Validators

```python
from trajectoryci.validators.enterprise import (
    ToolOrderValidator,          # enforce strict tool sequencing (KYC before wire transfer)
    PrivilegeEscalationGuard,    # block destructive actions without prior authorization
    PromptInjectionDetector,     # catch hostile input hijacking agent behavior
    DataLeakageDetector,         # catch PII/secrets leaking into tool args
    HallucinationReferenceGuard, # catch agents using IDs they never retrieved
    RateLimitBurstDetector,      # catch agents hammering external APIs
    ComplianceAuditValidator,    # HIPAA/SOX/GDPR audit trail enforcement
)

# HIPAA-compliant patient data agent
spec.add_step_validator(ComplianceAuditValidator(
    required_steps=[
        ("patient_consent_check", {"status": "approved"}),
        ("identity_verify", {"verified": True}),
        ("access_patient_records", None),
        ("audit_log", {"logged": True}),
    ],
    regulation="HIPAA",
))

# Financial workflow — enforce KYC before wire transfer
spec.add_step_validator(ToolOrderValidator(
    required_sequence=["kyc_check", "aml_screening", "execute_wire"]
))

# Block destructive actions without human approval
spec.add_step_validator(PrivilegeEscalationGuard(
    privileged_tools={"delete_records", "drop_table"},
    requires_prior_tool="get_human_approval",
))

# Catch hallucinated product IDs
spec.add_step_validator(HallucinationReferenceGuard(
    tool_that_provides_ids="search_products",
    result_key="id",
    tool_that_uses_ids="recommend",
    argument_key="product_id",
))
```

---

## Trajectory Regression Testing

Changed your prompt? Swapped models? Run historical trajectories through the new version and see exactly what changed at every step.

```python
from trajectoryci.regression import TrajectoryStore, RegressionRunner

runner = RegressionRunner(store=TrajectoryStore("./baselines"))
runner.save_baseline(trajectory, label="refund_flow_v1")

# After changing your prompt...
comparison = runner.run_regression(
    label="refund_flow_v1",
    new_trajectory=new_trajectory,
    spec=spec,
)

if comparison.regression_detected:
    print(comparison.new_failures)
    # ["Required tools not called: {'verify_eligibility'}", ...]
```

---

## Why Not LangSmith / Braintrust / Arize?

| Feature | TrajectoryCI | LangSmith | Braintrust | Arize |
|---|---|---|---|---|
| Step-level validation | ✅ Native | ❌ Output only | ❌ Output only | ❌ Output only |
| Deterministic path checks | ✅ Core feature | ❌ | ❌ | ❌ |
| Prompt injection detection | ✅ Built-in | ❌ | ❌ | ❌ |
| PII / data leakage detection | ✅ Built-in | ❌ | ❌ | ❌ |
| HIPAA / SOX compliance audit | ✅ Built-in | ❌ | ❌ | ❌ |
| Hallucination reference guard | ✅ Built-in | ❌ | ❌ | ❌ |
| Loop detection | ✅ Automatic | ❌ | ❌ | ❌ |
| Trajectory regression testing | ✅ Built-in | ⚠️ Partial | ⚠️ Partial | ❌ |
| Zero dependencies | ✅ | ❌ Requires SDK | ❌ Requires SDK | ❌ |

---

## Run the Tests

```bash
# Basic failures (5/5)
python tests/test_real_failures.py

# Enterprise failures (7/7)
python tests/test_enterprise.py
```

---

## Roadmap

- [ ] Async support
- [ ] LangChain / LlamaIndex native integrations
- [ ] GitHub Actions CI integration
- [ ] Web UI dashboard for trajectory visualization
- [ ] LLM-as-judge hybrid scoring (for non-deterministic steps)
- [ ] Cost anomaly alerting webhooks
- [ ] More compliance frameworks (SOC2, PCI-DSS, EU AI Act)

---

## Contributing

This is early. Issues, PRs, and feedback are extremely welcome.

Found a specific agent failure mode that TrajectoryCI doesn't catch? Open an issue — that's exactly what we want to know about.

---

## License

MIT © 2026 Derrick Turnbow
