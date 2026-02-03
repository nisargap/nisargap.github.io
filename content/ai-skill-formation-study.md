+++
title = "The Hidden Cost of AI Assistance: What New Research Reveals About Skill Development"
date = 2026-02-03
description = "A recent study finds that AI coding assistants can impair learning while providing productivity gains, but reveals interaction patterns that preserve skill development."
[taxonomies]
tags = ["ai", "software-engineering", "learning", "research", "tools"]
categories = ["Engineering"]
+++

A new research paper from Anthropic has quantified something many developers have suspected: using AI coding assistants can make you worse at programming. But the findings are more nuanced than that headline suggests, and they contain practical guidance for anyone using AI tools to write code.

<!-- more -->

## The Study

Researchers Judy Hanwen Shen and Alex Tamkin conducted a randomized experiment studying how developers learned a new Python asynchronous programming library (Trio) with and without AI assistance. Fifty participants completed coding tasks, with half having access to an AI chat assistant and half working without it.

The methodology was straightforward: participants first completed a warm-up task, then had 35 minutes to complete two coding exercises using the Trio library. Afterward, all participants took a 14-question quiz covering conceptual understanding, code reading, and debugging—without any AI help.

## The Core Finding

The results were stark. Participants with AI assistance completed tasks faster (Cohen's d=1.11, p=0.03), demonstrating improved task efficiency. However, they performed significantly worse on the knowledge quiz (Cohen's d=1.7, p=0.003).

AI use impaired conceptual understanding, code reading, and debugging abilities. Debugging questions showed the largest performance gap between the two groups.

In post-study feedback, participants in the AI group reported feeling "lazy" and acknowledged "there are still a lot of gaps in my understanding." They wished they had paid more attention to the library details during the task.

This is the crux of the problem: AI assistance traded short-term productivity for long-term competence.

## Six Interaction Patterns

The most valuable part of the research isn't the headline finding—it's the detailed analysis of how different participants used AI assistance. The researchers identified six distinct interaction patterns, which split cleanly into low-scoring and high-scoring groups.

### Low-Scoring Patterns (24%-39% on quiz)

**AI Delegation**: Participants wholly relied on AI to write code and complete the task. This group finished fastest and encountered few errors during the task itself. But they learned almost nothing.

**Progressive AI Reliance**: Participants started with one or two questions, then gradually delegated all code writing to the AI. They began engaged but ended passive. Their quiz scores were poor because they never mastered concepts from the second task.

**Iterative AI Debugging**: Participants wrote code themselves but relied on AI to debug and verify it. They asked many questions but used AI to solve problems rather than to understand them. The pattern of getting stuck, asking AI to fix it, and moving on left no lasting knowledge.

### High-Scoring Patterns (65%-86% on quiz)

**Conceptual Inquiry**: Participants only asked conceptual questions and used their improved understanding to write code themselves. Though they encountered many errors, they resolved them independently. This mode was the fastest among high-scoring patterns—only the pure AI Delegation approach was quicker overall.

The high scorers shared a common trait: they stayed cognitively engaged. They either asked conceptual questions instead of requesting code generation, or they asked for explanations alongside any generated code. They used AI as a teacher, not a replacement.

## What This Means for Software Development

These findings have direct implications for how we use AI tools in professional development work.

### The Supervision Problem

AI assistance produces real productivity gains—the study confirms this. But supervising AI effectively requires understanding what it's doing. Developers who delegate without comprehension can't catch subtle bugs, security vulnerabilities, or architectural problems.

This creates a paradox: the more you rely on AI for tasks you don't understand, the less capable you become of evaluating AI output for those same tasks. The skill gap compounds over time.

### Experience Level Matters

The researchers note that novice workers are most at risk. Experienced developers have existing mental models; AI assistance fills in implementation details but doesn't undermine foundational knowledge.

For developers learning a new language, framework, or domain, the temptation to lean heavily on AI is highest—and the cost of doing so is also highest. This is precisely when you need to build understanding, not bypass it.

### Safety-Critical Domains

The paper emphasizes that these findings are particularly concerning for safety-critical work. If a developer learned to use a library entirely through AI delegation, they lack the debugging intuition and conceptual understanding needed when things go wrong in production.

Code that works is not the same as code you understand. The difference becomes apparent during incidents.

## Practical Guidance

Based on the high-scoring interaction patterns, here's how to use AI assistance without sacrificing skill development:

**Ask "why" not "what"**: Instead of "write code that does X," ask "explain how X works in this library." Use the conceptual understanding to write code yourself.

**Request explanations with code**: If you ask for generated code, always ask for an explanation of how it works. Read the explanation. Make sure you could reproduce the approach.

**Debug yourself first**: When you hit an error, spend time understanding it before asking AI for help. Form a hypothesis. The learning happens in the struggle, not the solution.

**Use AI for exploration, not completion**: Ask AI to outline multiple approaches to a problem. Understand the trade-offs. Then implement yourself.

**Verify understanding actively**: After using AI help, ask yourself: could I do this without assistance now? If not, you haven't learned—you've just completed a task.

## The Bigger Picture

This study quantifies a trade-off that exists throughout AI-assisted work: efficiency versus learning, speed versus depth, completion versus comprehension.

The right balance depends on context. For a one-off task in a technology you'll never touch again, pure delegation makes sense. For skills you need to maintain and deepen, cognitive engagement is essential.

The research suggests that productivity and learning don't have to be opposites. The Conceptual Inquiry pattern—asking AI to explain rather than to do—was nearly as fast as pure delegation while preserving learning outcomes. The key is staying engaged, not staying slow.

As AI coding assistants become more capable and more integrated into daily workflows, these findings matter more, not less. The developers who thrive will be those who use AI to amplify their understanding, not replace it.

---

*The research paper ["How AI Impacts Skill Formation"](https://arxiv.org/abs/2601.20245) by Judy Hanwen Shen and Alex Tamkin was published on arXiv in January 2026.*
