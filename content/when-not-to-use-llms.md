+++
title = "When Traditional Code Beats LLMs"
date = 2026-01-09
description = "LLMs are impressive, but for many problems, deterministic code is simpler, faster, and more reliable."
[taxonomies]
tags = ["llm", "software-engineering", "architecture"]
categories = ["Productionization"]
+++

There's a pattern I've seen repeatedly: a team reaches for an LLM to solve a problem that would be better served by traditional code. The LLM works in demos, struggles in production, and eventually gets replaced by the straightforward solution that should have been built first.

<!-- more -->

## The Allure of the Universal Hammer

LLMs feel like universal problem solvers. Natural language in, useful output out. No need to enumerate cases or write parsing logic. Just describe what you want.

This flexibility is genuinely powerful for genuinely fuzzy problems. But many problems that seem fuzzy are actually well-structured once you look closely.

## Case Study: Data Extraction

**The LLM approach**:
```
Prompt: Extract the date, amount, and vendor from this receipt:
        "COFFEE SHOP - $4.50 - 01/15/2024"
```

**The traditional approach**:
```python
import re

def parse_receipt(text):
    pattern = r"(.+) - \$(\d+\.\d{2}) - (\d{2}/\d{2}/\d{4})"
    match = re.match(pattern, text)
    if match:
        return {
            "vendor": match.group(1),
            "amount": float(match.group(2)),
            "date": match.group(3)
        }
    raise ParseError("Unexpected format")
```

The regex is faster (microseconds vs. seconds), cheaper (free vs. API calls), deterministic (same input, same output), and explicit about what it can and cannot parse.

"But what about varied formats?" If your receipts come in 5 formats, write 5 parsers. If they come in 500 formats, maybe then consider an LLM—but also consider whether your data pipeline is broken.

## Case Study: Classification

**The LLM approach**:
```
Prompt: Classify this support ticket as: billing, technical, or general.
        Ticket: "My payment didn't go through"
```

**The traditional approach**:
```python
BILLING_KEYWORDS = {"payment", "invoice", "charge", "billing", "refund", "subscription"}
TECHNICAL_KEYWORDS = {"error", "bug", "crash", "login", "password", "loading"}

def classify_ticket(text):
    words = set(text.lower().split())
    billing_score = len(words & BILLING_KEYWORDS)
    technical_score = len(words & TECHNICAL_KEYWORDS)

    if billing_score > technical_score:
        return "billing"
    elif technical_score > billing_score:
        return "technical"
    return "general"
```

This keyword approach handles 80% of cases correctly. For the remaining 20%, you could:

1. Expand the keyword lists based on misclassified examples
2. Use a small fine-tuned classifier (not an LLM)
3. Accept imperfect classification with human review

An LLM might get 85% accuracy—marginally better—but at 1000x the cost and latency.

## Case Study: Validation

**The LLM approach**:
```
Prompt: Is this a valid email address? user@example.com
```

**The traditional approach**:
```python
import re

def is_valid_email(email):
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))
```

Using an LLM for validation is absurd. It's slower, costs money, and might randomly decide "user@example.com" isn't valid because of some training data quirk.

Yet I've seen teams use LLMs for validation because "it's easier than figuring out the regex."

## Case Study: Templating

**The LLM approach**:
```
Prompt: Generate an email welcoming {user_name} to {company_name}.
```

**The traditional approach**:
```python
def welcome_email(user_name, company_name):
    return f"""
    Hi {user_name},

    Welcome to {company_name}! We're excited to have you.

    Best,
    The {company_name} Team
    """
```

Templates are predictable, fast, and free. LLM-generated emails vary randomly, might include hallucinated details, and require review before sending.

"But templates feel robotic." Then write better templates. Hire a copywriter for an afternoon. Don't introduce non-determinism into customer communications.

## The Decision Framework

Before using an LLM, ask:

### 1. Can I enumerate the inputs?

If inputs follow known patterns, traditional parsing is better. LLMs shine when inputs are genuinely unpredictable.

### 2. Is exact correctness required?

LLMs are probabilistic. If wrong answers have significant consequences, you need either 100% human review or a deterministic system.

### 3. What's the latency budget?

LLMs add seconds of latency. For real-time applications, this may be unacceptable.

### 4. What's the cost at scale?

Calculate LLM costs at your expected volume. Compare to engineering time for traditional solutions.

### 5. How will I test it?

Traditional code has clear test strategies. LLM testing is harder and less conclusive.

### 6. Who maintains it?

When the LLM behavior changes (and it will), who debugs it? Traditional code is easier to understand and modify.

## The Hybrid Approach

Sometimes the right answer is both:

1. **Traditional code first**: Handle the common, well-structured cases
2. **LLM fallback**: For cases that fail structured parsing
3. **Human escalation**: For low-confidence LLM outputs

```python
def process_query(text):
    # Try structured parsing first
    result = try_structured_parse(text)
    if result:
        return result

    # Fall back to LLM for unstructured input
    llm_result = llm_parse(text)
    if llm_result.confidence > 0.9:
        return llm_result.value

    # Escalate uncertain cases
    return escalate_to_human(text)
```

This gives you the efficiency of traditional code, the flexibility of LLMs, and the reliability of human review—each applied where they're most effective.

## The Meta-Lesson

LLMs are tools, not magic. Like any tool, they have appropriate and inappropriate uses.

The best engineers I know treat LLMs with the same skepticism they'd apply to any dependency: What are the failure modes? What's the cost? What's the alternative? They reach for LLMs when the problem genuinely requires fuzzy reasoning—and reach for traditional code when it doesn't.

The worst engineers treat LLMs as a way to avoid thinking carefully about their problem. The result is systems that are slow, expensive, unpredictable, and hard to debug.

Don't be the latter. Sometimes the boring solution is the right one.
