# Write Better Jev Questions

*An independent guide by Josh Pyle · Updated November 5, 2026*

**The question you give Jev shapes every answer it returns.**

Asking *"Should we approve this?"* conflates two distinct tasks:

1. **Semantic judgment:** What does the messy input text actually say or imply?
2. **Policy:** Based on the facts, what do *our rules* say to do next?

Jev provides fast, bounded semantic judgments -- a "smart if-statement" for text. Specifically, Jev returns calibrated probability distributions for closed questions rather than generating freeform text. Deterministic code owns policy, math, and side effects, keeping application logic modular.

**Mental model:** **Jev reads the evidence; your code applies the rules.**

*This guide covers core single-pass primitives (Choice, Score, Noul). For multi-step patterns -- Search, Retrieval, Ranking, Extraction -- see the [TypeSafe Cookbooks](https://docs.typesafe.ai/cookbooks.md).*

---

## From Playground to Prod: Is a hot dog a sandwich?

TypeSafe's API Playground opens with **"Is a hot dog a sandwich?"** That makes a fine demo, but a **poor** production question without explicit rules.

### 1. Evidence demands constraints

Suppose your food labeling app defines a sandwich as *filling between two separate pieces of bread*. Ask Jev to evaluate what the text supports rather than settle the debate:

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

### 2. Code Rules OK

Application code owns policy; Jev strictly evaluates the input text.
Defining explicit criteria gives you something to test, not a guarantee the model gets it right. By keeping the rules in code, you can change your decision thresholds without rewriting your questions, test those rules without an API call, and trace an unexpected result back to a specific assessment. And if a database record already contains a clean `bread_pieces: 2` column, skip the model entirely and use an `if` statement.

### Hot dogs and Choice overlaps

Suppose you're classifying hot dogs from menu text. New York's version is deliberately simple (dirty water is implied):

```python
# An entry inside your questions dictionary.
"hot_dog_style": Choice(
    instructions=(
        "Which listed regional style is best supported by the preparation "
        "and toppings described? Favor ingredient evidence over menu labels. "
        "If details are missing or styles conflict, choose other_or_unclear."
    ),
    criteria={
        "chicago": (
            "Beef frank on a poppy-seed bun with yellow mustard, "
            "neon-green relish, white onion, tomato, pickle, "
            "sport peppers, and celery salt. "
            "For this application's rubric, ketchup is disqualifying."
        ),
        "new_york": (
            "Natural-casing frank with yellow mustard and "
            "sauerkraut or griddled onions. Simple preparation."
        ),
        "other_or_unclear": (
            "Conflicting signals, insufficient preparation detail, "
            "or a style not listed here."
        ),
    },
)
```

| Menu description | Expected label | Why |
| :--- | :--- | :--- |
| `"Chicago dog with ketchup"` | `other_or_unclear` | Ketchup directly conflicts with being a legitimate Chicagoan |
| `"Natural-casing frank, yellow mustard, sauerkraut"` | `new_york` | Clean signal, no conflict |
| `"Our famous house dog"` | `other_or_unclear` | The description lacks preparation details |

Ketchup sends the order to `other_or_unclear`. Your code decides what to do next.
The same logic applies to pineapple on pizza: Jev detects it; whether your app routes that order to a review queue is between you and your conscience.

---

## The coding agent litmus test: Kill the One-Question Monolith

Left to their own devices, AI coding assistants default to **The One-Question Monolith** — prompts like `should_approve`, `recommend_action`, or `is_good_fit` that bundle evidence gathering, domain assumptions, and product policy into a single opaque verdict. When an assessment looks wrong, you can't tell whether the model misread the evidence or your unstated rules made a bad call.

**The test:**

- Given the input and an *explicit* rubric, can a reviewer make a bounded judgment from the text alone? Ask Jev.
- Does the next step require thresholds, business priorities, permissions, or side effects? Keep that in code.

| Instead of a One-Question Monolith… | Ask Jev for the evidence… | Put the policy in code… |
| :--- | :--- | :--- |
| "Should we escalate this ticket?" | Are the steps to reproduce in place? How broken is the feature? | Escalate based on your SLA rules and `.confidence` |
| "Should we approve this request?" | What type of request is it? Is condition X stated? | Run your eligibility rules in Python or TypeScript |
| "Is this candidate qualified?" | Does the résumé show experience at the role's level? How much direct ownership of the function? | Score and route in code; keep hiring calls with people |
| "Should we remove this post?" | Does it contain a personal attack? Is explicit content described? | Your content policy decides; Jev just reads the post |

---

## Four common question shapes (and where to start)

Jev provides three primitives (`Choice`, `Score`, `Noul`) that handle four standard integration patterns:

| Pattern | Question shape | Primitive | What code owns |
| :--- | :--- | :--- | :--- |
| **Classification** | "Which bucket does this belong in?" | `Choice` | Downstream handling per bucket |
| **Routing signal** | "Which queue fits this issue?" | `Choice` + `Noul` flags | Confidence checks and actual dispatch |
| **Rubric scoring** | "Where does this sit on an ordered scale?" | `Score` | Cutoffs, weights, and sorting |
| **Detection** | "Does this single yes/no claim hold?" | `Noul` | Thresholds for action vs. human review |

Keep each question atomic (a single judgment verifiable on sight). Jev evaluates questions in parallel via speculative fan-out, so adding more questions usually has little effect on response time ([see the fan-out pattern guide](https://docs.typesafe.ai/patterns/fan-out.md)).

### What comes back

The Python SDK splits results by primitive type:

- **`response.choices["key"]`**: `.choice` (winning key), `.probabilities` (per-option), `.confidence`
- **`response.scores["key"]`**: `.score` (probability-weighted mean from `0` to `N-1`, fractional, e.g. `1.43` on a 3-level rubric), `.probabilities`, `.legend`, `.confidence`
- **`response.nouls["key"]`**: `.noul` (probability of the proposition being true, `0`–`1`), without a separate `.confidence` field.

**Note:** `.confidence` measures how concentrated the probabilities are across your defined options — not how often the answer is correct. A confidence of 0.95 indicates strong model preference for one option, not a 95% chance the answer is correct. Validate accuracy on labeled data.

---

## 1. Write better Choice questions

Use `Choice` to select one category from a defined set. For binary yes/no, use `Noul`.
`Choice` fails most often when criteria are circular labels that don't explain how to break ties on boundary cases.

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

If you can't explain in one sentence what separates two options, neither can Jev. Always include an `unclear` or `other` way out for inputs that fall outside your criteria.

Fewer, well-defined options are typically easier to distinguish and validate.

---

## 2. Write better Score questions

Use `Score` to measure the degree of a quality on an ordered scale.

`Score` takes ordered level descriptions from lowest to highest. Bare adjectives (`["Low", "Medium", "High"]`) cause Jev to split probabilities because "Medium" has no shared meaning in your domain.

```python
# Bad: unanchored adjectives
"severity": Score(
    instructions="How severe is the issue?",
    criteria=["Low", "Medium", "High"],
)
```

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

The goal is for a given score to reflect the exact same standard on the 50th case as on the first. If adjacent levels keep splitting on clear inputs, upgrade each to a structured object:

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

Use the fewest levels that actually matter, and make sure you can tell adjacent levels apart on real inputs before shipping.

Don't use `Score` to count items, compare dates, or do math. Compute those in Python and pass the result in `state` if Jev needs it.

---

## 3. Write better Noul questions

`Noul` evaluates a single binary proposition. Vague prompts cause probabilities to cluster around 0.5.

```python
# Bad: relevant to what?
"relevant": Noul(instructions="Is this message relevant?")
```

```python
# Good: one observable yes/no check
"has_repro_steps": Noul(
    instructions="Does the message explicitly describe a sequence of steps to reproduce the problem?"
)
```

If two reviewers reading the same state would disagree on the answer, tighten the instruction or split compound checks into separate `Noul` questions.

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

Using our `hot_dog_style` question from above, add a companion Noul to check whether the evidence was sufficient:

```python
"style_has_sufficient_evidence": Noul(
    instructions=(
        "Does `item_description` contain enough topping detail to identify a regional style "
        "without relying on the item name?"
    )
)
```

Then, in the routing logic:

```python
style_has_sufficient_evidence_prob = response.nouls["style_has_sufficient_evidence"].noul
if style_has_sufficient_evidence_prob < 0.75:
    return "NEEDS_HUMAN_REVIEW"  # Choice answered, but evidence was thin
```

This pattern is especially useful when inputs are user-generated and evidence quality varies (a 100% confident Choice result on thin evidence is still a thin result).

---

## 4. Putting it together: Speculative fan-out & pure function policy

Jev evaluates all questions in a single forward pass, so adding more questions usually has little effect on response time. Even when routing logic uses an answer conditionally (e.g. severity matters only if category is "technical"), include both in the same `system_one()` invocation. LLM coding assistants frequently default to serialized API calls; override this pattern explicitly.

To override it, add this instruction at the start of any Jev-related session with your coding agent: *"Jev's `system_one()` evaluates all questions in a single forward pass. Bundle questions that can be evaluated from the same input into one call, then apply conditional routing in code."*

Instead of asking Jev *"Which team should own this and how urgent is it?"*, evaluate atomic evidence questions in one `system_one()` call and route the output through a pure function.

`state` provides context (raw text, structured fields, computed values). Omit unused fields.

The example below also asks whether the sender explicitly requested a human — that routes to human support before category routing even runs.

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
    # Explicit user requests for human support take priority over confidence triage.
    if wants_human_probability >= 0.80:
        return "HUMAN_SUPPORT_REQUESTED"
    # Low category confidence supersedes standard routing; better to triage than misroute.
    if category == "unclear" or category_confidence < 0.65:
        return "NEEDS_HUMAN_TRIAGE"
    # Severity confidence only matters for technical tickets; don't penalize billing on a severity score.
    if category == "technical" and severity_confidence < 0.60:
        return "NEEDS_HUMAN_TRIAGE"
    # Repro steps needed for priority routing: high severity without them goes to human triage.
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

Because `route_ticket()` is pure Python, logic, fallback routing, and threshold adjustments can be unit tested offline without API calls. If a routing decision looks off, you know immediately whether the model assessed the evidence badly or your rules made the wrong call.

---

## 5. Upgrade models cleanly

When TypeSafe updates Jev models, output distributions can shift even if prompts remain unchanged. Decoupling evidence evaluation from application policy keeps model upgrades routine.

1. **Pin versions in production:** Pin exact versions like `jev-1.14.0` rather than `jev-latest`, and log `response.model`.
2. **Run side-by-side on frozen fixtures:** Compare distributions across benchmark inputs before adjusting policy thresholds.
3. **Separate regression checks from benchmarks:** Verify both specific edge cases and broad distribution metrics before deployment.

---

## 6. Failure modes (and fast fixes)

| Symptom | Likely culprit | Fix |
| :--- | :--- | :--- |
| Wrong `Choice` on borderline inputs | Overlapping criteria | Add `"not for..."` exclusions and test mixed-intent messages |
| High confidence on a wrong answer | Extraneous context, or categories that don't cover all inputs | Trim irrelevant `state` fields; add a catch-all way out; validate with labeled examples, not just confidence |
| `Noul` hovers near `0.5` | Vague proposition | Tighten the instruction; route middle probabilities to human review |
| `Score` splits between adjacent levels | Abstract rubric anchors | Convert criteria strings to `{"what": ..., "examples": [...]}` objects |
| Answers flip when options are reordered | Option-order sensitivity | Test `Choice` questions with permuted option orders on frozen fixtures |
| Behavior shifts overnight | Using `jev-latest` alias | Pin the exact model version and re-validate thresholds before bumping |
| Bad date math or item counts | Asking Jev to calculate | Parse dates and count in Python; pass computed results in `state` |

Check the TypeSafe docs for version-specific edge cases before upgrading.

---

## 7. A pre-flight checklist

- **One judgment per question**: Ask for clear, observable evidence, not a One-Question Monolith like 'should we approve'
- **Contrastive Choice criteria**: Define inclusions, exclusions, and fallback categories for every option
- **Anchored Score levels**: Use ordered behavioral descriptions instead of unanchored labels
- **Single-claim Noul**: Evaluate one yes/no proposition per question
- **Pure-function policy**: Execute thresholds, weights, and fallback logic in application code
- **Focused state**: Include only fields necessary for judgment
- **Version hygiene**: Pin model versions, log responses, and explicitly handle errors

---

## References

- [Python SDK](https://docs.typesafe.ai/sdk/python.md) · [JavaScript SDK](https://docs.typesafe.ai/sdk/javascript.md)
- Allie Laabs, [Don't Call It a Decision Model](https://typesafe.ai/blog/dont-call-it-a-decision-model), TypeSafe AI (Nov 3, 2026).
- Primitives: [Choice](https://docs.typesafe.ai/primitives/choice.md) · [Score](https://docs.typesafe.ai/primitives/score.md) · [Noul](https://docs.typesafe.ai/primitives/noul.md)
- Patterns: [Speculative fan-out](https://docs.typesafe.ai/patterns/fan-out.md)

---

Write a question Jev can answer. Put everything else in code.

*Questions or corrections? Open an issue on GitHub.*
