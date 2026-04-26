---
name: clarify-specs
description: Interview the user relentlessly about a plan or design until reaching shared understanding, resolving each branch of the decision tree. Use when user wants to stress-test a plan, get grilled on their design, or mentions "clarify specs".
---

# clarify-specs

This skill guides the agent to aggressively interview the user to refine, clarify, and stress-test their plan or design. 

## When to use

- The user wants to stress-test a plan or architecture.
- The user asks you to "clarify specs" on a design or idea.
- The user provides a high-level, vague plan and asks for feedback or clarification.

## Instructions

1. **Analyze and Verify:** Read the initial plan or design provided by the user. **If a question about the design can be answered by exploring the existing codebase, explore the codebase instead of asking the user.**
2. **Pace the Interview:** Ask the questions **one at a time**. Do not overwhelm the user with a massive list of questions at once. Wait for their answer before moving to the next question.
3. **Provide Recommendations:** For each question you ask, provide your recommended answer as multiple choice options (e.g., 1, 2, 3, etc.) to make it easy for the user to respond and think through the tradeoffs.
4. **Walk the Tree:** Walk down each branch of the design tree systematically, resolving dependencies between decisions one-by-one.
5. **Iterate:** Interview the user relentlessly about every aspect of the plan until a shared understanding is reached and all ambiguities are resolved.
