# Write Better Jev Questions

*An independent guide by Josh Pyle · Updated November 6, 2026*

**The question you give Jev shapes every answer it returns.**

Most bad Jev calls fail for the same reason: they ask for a decision when Jev is best at a bounded judgment about the text.

Ask:

- "What does the input text actually say?"
- "Which category is best supported by the evidence?"
- "Does this claim hold given the text?"

Not:

- "Should we approve this?"
- "Is this a good fit?"
- "Should we escalate this?"

Those are policy questions. Jev is the evidence layer. Your application code decides what to do next.

This guide covers the core single-pass primitives: `Choice`, `Score`, and `Noul`.

---

## The mental model: Jev reads evidence; your code applies policy

Jev is a fast, bounded semantic judgment engine for closed questions.

It is not a generic decision maker. It does not know your business rules, thresholds, or side effects. It evaluates the text under explicit criteria and returns a calibrated probability distribution.

That distinction matters:

1. Evidence judgment: what does the input text actually support?
2. Policy: based on those facts, what should our app do?

Keep them separate.

A clean pattern is:

- Jev answers the fact question
- Python or TypeScript applies the policy decision
- human review or business logic handles exceptions

---

## From playground to production: is a hot dog a sandwich?

TypeSafe's API Playground opens with "Is a hot dog a sandwich?" That makes a good demo. It is a bad production question unless your system defines the rules.

If your food labeling app defines a sandwich as "filling between two separate pieces of bread," ask Jev to evaluate the evidence instead of settling the debate for you.

```python
# An entry inside your questions dictionary.
"has_two_separate_bread_pieces": Noul(
    instructions=(
        "Does the food description explicitly indicate two separate pieces "
        "of bread surrounding a filling? "
        "Do not infer this from the food's name alone."
    ),
)
```

That question is narrow, testable, and useful. It does not commit your app to a universal policy on whether a hot dog is a sandwich. It just measures whether the input description supports the claim that the food has two separate bread pieces.

Your code can then decide:

- label it as a sandwich only if your rules say so
- route to manual review if evidence is thin
- reject ambiguous packaging claims when certainty is low

---

## The coding agent litmus test: kill the one-question monolith

Left to their own devices, coding agents often default to a one-question monolith:

- `should_approve`
- `recommend_action`
- `is_good_fit`
- `should_escalate`

These bundle multiple responsibilities into a single ambiguous prompt. They mix:

- fact extraction
- rule interpretation
- threshold logic
- action selection

That is exactly what to avoid.

Use this test:

- If a reviewer can make a bounded judgment from the text alone, ask Jev.
- If the next step requires thresholds, permissions, business priorities, or side effects, keep that in code.

| Instead of a One-Question Monolith… | Ask Jev for the evidence… | Put the policy in code… |
| :--- | :--- | :--- |
| "Should we escalate this ticket?" | Are the steps to reproduce in place? How broken is the feature? | Escalate based on SLA rules and `.confidence` |
| "Should we approve this request?" | What type of request is it? Is condition X stated? | Run your eligibility rules in Python or TypeScript |
| "Is this candidate qualified?" | Does the résumé show experience at the role's level? | Score and route in code; keep hiring calls with people |
| "Should we remove this post?" | Does it contain a personal attack? Is explicit content described? | Your content policy decides; Jev only reads the post |

---

## Four common question shapes (and where to start)

Jev gives you three primitives: `Choice`, `Score`, and `Noul`.

Each covers a common integration pattern:

| Pattern | Question shape | Primitive | What code owns |
| :--- | :--- | :--- | :--- |
| Classification | "Which bucket does this belong in?" | `Choice` | Downstream handling per bucket |
| Routing signal | "Which queue fits this issue?" | `Choice` + `Noul` flags | Confidence checks and actual dispatch |
| Rubric scoring | "Where does this sit on an ordered scale?" | `Score` | Cutoffs, weights, sorting |
| Detection | "Does this single yes/no claim hold?" | `Noul` | Thresholds for action vs. human review |

The best Jev questions are atomic. They ask for one bounded judgment you could verify by reading the text.

Jev evaluates all questions in a single forward pass, so adding more questions often has little impact on latency.

### What comes back

The Python SDK splits results by primitive type:

- `response.choices["key"]`: `.choice`, `.probabilities`, `.confidence`
- `response.scores["key"]`: `.score`, `.probabilities`, `.legend`, `.confidence`
- `response.nouls["key"]`: `.noul` (probability that the proposition is true, from `0` to `1`)

Important: `.confidence` measures how concentrated the probabilities are across your options. It is not a measure of truth. A confidence of `0.95` means the model strongly prefers one option; it does not mean the answer is definitely correct.

---

## 1. Write better Choice questions

Use `Choice` when you need one category from a defined set.

Use `Noul` for binary yes/no checks.

`Choice` usually fails when the criteria are circular labels that do not explain what separates one option from another.

### Bad example

```python
# Bad: circular labels
"category": Choice(
    instructions="What type of support issue is this?",
    criteria={
        "billing": "A billing issue",
        "technical": "A technical issue",
        "other": "Something else",
    },
)
```

This is almost impossible to evaluate consistently. It tells Jev the label, not the boundary.

### Better example

```python
# Good: contrastive boundaries -- what it is (and isn't)
"category": Choice(
    instructions="What is the primary topic of this support request?",
    criteria={
        "billing":   "Charges, invoices, refunds, payment methods; not API or product bugs.",
        "technical": "Product failures, API/SDK errors, broken features; not billing disputes.",
        "access":    "Login, SSO, API key permissions, account seats; not general API errors.",
        "unclear":   "Insufficient evidence or no clear primary category.",
    },
)
```

If you cannot explain in one sentence what separates two options, Jev probably cannot either.

Always include an `unclear` or `other` escape hatch for inputs that do not fit the rubric.

Fewer, well-defined options are easier to distinguish and validate.

---

## 2. Write better Score questions

Use `Score` for ordered judgments such as severity, quality, or degree of impact.

The common mistake is using bare adjectives like `["Low", "Medium", "High"]`. Those labels are too abstract and have no shared meaning across inputs.

### Bad example

```python
# Bad: unanchored adjectives
"severity": Score(
    instructions="How severe is the issue?",
    criteria=["Low", "Medium", "High"],
)
```

### Better example

```python
# Good: ordered levels anchored in observable impact
"severity": Score(
    instructions="How severely is product functionality affected by the reported behavior?",
    criteria=[
        "Cosmetic or minor inconvenience; core functionality still works.",
        "A feature fails, but a usable workaround is described.",
        "A core feature is blocked with no workaround described.",
    ],
)
```

The goal is for a given score to mean the same thing on the 50th case as on the first. If adjacent levels keep splitting cleanly labeled examples, make each level more concrete.

This is the upgrade pattern:

```python
"severity": Score(
    instructions="How severely is product functionality affected?",
    criteria=[
        {
            "what": "Cosmetic; no impact to functionality.",
            "examples": ["typo in a UI label", "misaligned icon in dark mode"],
        },
        {
            "what": "Feature degraded, but a usable workaround is described.",
            "examples": ["PDF export hangs, but CSV export succeeds"],
        },
        {
            "what": "Core feature blocked; no workaround described.",
            "examples": ["cannot log in", "webhook retries dropping payloads"],
        },
    ],
)
```

Use the fewest levels that matter. Do not use `Score` to count items, compare dates, or do math. Compute those in Python and pass the result in `state` if Jev needs it.

---

## 3. Write better Noul questions

`Noul` evaluates a single binary proposition.

Vague prompts create vague probabilities near `0.5`.

### Bad example

```python
# Bad: relevant to what?
"relevant": Noul(instructions="Is this message relevant?")
```

### Better example

```python
# Good: one observable yes/no check
"has_repro_steps": Noul(
    instructions="Does the message explicitly describe a sequence of steps to reproduce the problem?"
)
```

If two reviewers reading the same input would disagree, tighten the instruction or break the compound claim into multiple `Noul` checks.

For subtle boundaries, add explicit `Look for` and `Exclude` clauses:

```python
"contacted_before": Noul(
    instructions=(
        "Has the customer contacted support about this issue before? "
        "Look for: a ticket number, mention of a prior attempt, or that they have asked before. "
        "Exclude: general frustration or urgency with no evidence of a prior contact."
    )
)
```

### Evidence verification

Using the earlier `hot_dog_style` example, add a `Noul` question to check whether the evidence was sufficient:

```python
"style_has_sufficient_evidence": Noul(
    instructions=(
        "Does `item_description` contain enough topping detail to identify a regional style "
        "without relying on the item name?"
    )
)
```

Then route the output accordingly:

```python
style_has_sufficient_evidence_prob = response.nouls["style_has_sufficient_evidence"].noul
if style_has_sufficient_evidence_prob < 0.75:
    return "NEEDS_HUMAN_REVIEW"
```

This is especially useful when inputs are user-generated and evidence quality varies; a highly confident `Choice` result on thin evidence is still a thin result.

---

## 4. Putting it together: speculative fan-out and pure policy

Jev can evaluate multiple questions in one forward pass. That is a feature, not a bug. It means you can ask for several atomic judgments at once without paying a large penalty for each additional question.

This lets you do this pattern:

- Ask Jev for evidence on individual facts
- Normalize and combine the results in pure Python
- Route to a queue or decision based on policy logic

Do not ask Jev a giant policy blob like: "Which team should own this and how urgent is it?"

Ask a set of narrow questions instead, then route the output through a pure function.

`state` provides context (raw text, structured fields, computed values). Include only what is necessary for the judgment.

The example below also asks whether the sender explicitly requested a human. That routes to human support before category routing even runs.

```python
"""Companion example for 'Write Better Jev Questions'.

Install: pip install typesafe-sdk
Set TYPESAFE_API_KEY in your environment before running.
Thresholds below are illustrative; calibrate on your own labeled data.
"""

# 1. Pure, unit-testable application policy: no network calls, zero model dependency.

def route_ticket(
    category: str,
    category_confidence: float,
    severity_score: float,
    severity_confidence: float,
    repro_probability: float,
    wants_human_probability: float,
) -> str:
    """Return a recommended review queue based on bounded evidence."""
    if wants_human_probability >= 0.80:
        return "HUMAN_SUPPORT_REQUESTED"
    if category == "unclear" or category_confidence < 0.65:
        return "NEEDS_HUMAN_TRIAGE"
    if category == "technical" and severity_confidence < 0.60:
        return "NEEDS_HUMAN_TRIAGE"
    if category == "technical" and severity_score >= 1.5:
        if repro_probability >= 0.70:
            return "PRIORITY_TECHNICAL_REVIEW"
        return "NEEDS_HUMAN_TRIAGE"
    return {
        "technical": "TECHNICAL_SUPPORT",
        "billing":   "BILLING_SUPPORT",
        "access":    "ACCOUNT_ACCESS_SUPPORT",
    }.get(category, "NEEDS_HUMAN_TRIAGE")


def main() -> None:
    import os
    from typesafe_sdk import (
        TypeSafeClient,
        Choice,
        Score,
        Noul,
        TypeSafeRateLimitError,
        TypeSafeAPIConnectionError,
        TypeSafeAPITimeoutError,
        TypeSafeAPIError,
    )

    state = {
        "subject": "Report export fails",
        "message": (
            "Exporting a report returns HTTP 500 every time. "
            "Steps: sign in, open Reports, click Export. Started this morning."
        ),
    }
    questions = {
        "category": Choice(
            instructions="What is the primary topic of this support request?",
            criteria={
                "billing":   "Charges, invoices, refunds, payment methods; not product errors.",
                "technical": "Product failures, API/SDK errors, broken features; not billing disputes.",
                "access":    "Signing in, authentication, API keys, permissions; not general API errors.",
                "unclear":   "Insufficient evidence or no clear primary category.",
            },
        ),
        "severity": Score(
            instructions="How severely is product functionality affected by the reported behavior?",
            criteria=[
                "Cosmetic or minor inconvenience; functionality still works.",
                "A feature fails, but a reasonable workaround is described.",
                "A needed feature is blocked, and no workaround is described.",
            ],
        ),
        "has_repro_steps": Noul(
            instructions="Does the message explicitly describe a sequence of steps to reproduce the problem?"
        ),
        "wants_human": Noul(
            instructions=(
                "Does the sender explicitly request to speak or be contacted by a human? "
                "Look for: requests to talk, call, speak with, or be contacted by a person or human support agent. "
                "Exclude: frustration or urgency with no direct request for human contact."
            )
        ),
    }

    # Pin an exact version in production; moving aliases like jev-preview and jev-latest can shift distributions.
    try:
        with TypeSafeClient(api_key=os.environ["TYPESAFE_API_KEY"], model="jev-1.14.0") as client:
            response = client.system_one(state=state, questions=questions)
    except (TypeSafeAPIConnectionError, TypeSafeAPITimeoutError) as error:
        print(f"Network error: {error}")
        print("Queue: NEEDS_HUMAN_TRIAGE (Network unavailable)")
        return
    except TypeSafeRateLimitError as error:
        print(f"Rate limited: {error}")
        print("Queue: NEEDS_HUMAN_TRIAGE (Rate limit exceeded)")
        return
    except TypeSafeAPIError as error:
        print(f"Server error: {error}")
        print("Queue: NEEDS_HUMAN_TRIAGE (Server error)")
        return

    queue = route_ticket(
        category=response.choices["category"].choice,
        category_confidence=response.choices["category"].confidence,
        severity_score=response.scores["severity"].score,
        severity_confidence=response.scores["severity"].confidence,
        repro_probability=response.nouls["has_repro_steps"].noul,
        wants_human_probability=response.nouls["wants_human"].noul,
    )
    print(f"Model: {response.model} | Queue: {queue}")
    print(f"Category: {response.choices['category'].choice} (conf={response.choices['category'].confidence:.2f}) | Severity: {response.scores['severity'].score:.2f}")


if __name__ == "__main__":
    main()
```

Because `route_ticket()` is pure Python, the logic, fallback routing, and threshold adjustments can be unit tested offline without API calls. If a routing decision looks off, you know immediately whether the bug is in the evidence questions or in the policy rules.

---

## 5. Upgrade models cleanly

When TypeSafe updates Jev models, output distributions can shift even if prompts remain unchanged. Decoupling evidence evaluation from policy logic keeps upgrades routine.

1. Pin versions in production: use exact versions like `jev-1.14.0` instead of `jev-latest`, and log `response.model`.
2. Run side-by-side checks on frozen fixtures: compare distributions across regression inputs before changing router thresholds.
3. Separate regression checks from benchmark sweeps: validate both edge cases and broader distribution metrics before deployment.

---

## 6. Failure modes (and fast fixes)

| Symptom | Likely culprit | Fix |
| :--- | :--- | :--- |
| Wrong `Choice` on borderline inputs | Overlapping criteria | Add `not` exclusions and test mixed-intent messages |
| High confidence on a wrong answer | Extraneous context or missing catch-all bucket | Trim irrelevant `state` fields; add an `unclear` category |
| `Noul` hovers near `0.5` | Vague proposition | Tighten the instruction; route uncertain probabilities to human review |
| `Score` splits between adjacent levels | Abstract rubric anchors | Convert criteria strings into `{"what": ..., "examples": [...]}` objects |
| Answers flip when options are reordered | Option-order sensitivity | Test `Choice` questions with permuted option orders on frozen fixtures |
| Behavior shifts overnight | Using `jev-latest` alias | Pin the exact model version and re-validate thresholds before bumping |
| Bad date math or item counts | Asking Jev to calculate | Parse dates and count in Python; pass computed results in `state` |

Check the TypeSafe docs for version-specific edge cases before upgrading.

---

## 7. A pre-flight checklist

Before shipping any Jev question, check the basics:

- One judgment per question: ask for one clear, observable fact, not a bundled decision
- Contrastive `Choice` criteria: define inclusions, exclusions, and fallback categories for every option
- Anchored `Score` levels: use ordered behavioral descriptions rather than adjectives alone
- Single-claim `Noul`: evaluate one yes/no proposition per question
- Pure-function policy: put thresholds, weights, and fallback logic in application code
- Focused `state`: include only the fields needed for judgment
- Version hygiene: pin the model version, log results, and handle errors explicitly

---

## References

- [Python SDK](https://docs.typesafe.ai/sdk/python.md) · [JavaScript SDK](https://docs.typesafe.ai/sdk/javascript.md)
- Allie Laabs, [Don't Call It a Decision Model](https://typesafe.ai/blog/dont-call-it-a-decision-model), TypeSafe AI (Nov 3, 2026).
- Primitives: [Choice](https://docs.typesafe.ai/primitives/choice.md) · [Score](https://docs.typesafe.ai/primitives/score.md) · [Noul](https://docs.typesafe.ai/primitives/noul.md)
- Patterns: [Speculative fan-out](https://docs.typesafe.ai/patterns/fan-out.md)

---

Write a question Jev can answer. Put everything else in code.

*Questions or corrections? Open an issue on GitHub.*
