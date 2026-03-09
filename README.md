# Agently Skills

Reusable skills for AI agents and operator workflows.

Agently Skills is a collection of portable, composable skills designed to help AI agents do useful work more reliably. Some of these skills are published for public use. Others are used internally by our team to support product, operations, research, GTM, and workflow automation.

The goal is simple: turn repeatable tasks into reusable capabilities.

## Why this repo exists

We built this repository for two reasons.

First, to share useful skills publicly so developers, builders, and teams can use them in their own agent workflows.

Second, to maintain a common internal library of operational skills that help us run parts of Agently more consistently, from note-taking and meeting follow-ups to research, QA, documentation, and execution support.

In practice, many of the best internal skills are also useful externally.

## Who this is for

This repo may be useful if you are:
- building AI agents or agentic workflows
- creating structured prompt assets for repeated tasks
- standardising internal team processes with AI
- experimenting with tool-using assistants
- looking for production-oriented skill patterns rather than one-off prompts

## Design principles

We try to keep skills:
- **practical**: focused on real work, not demos
- **modular**: easy to compose with other skills
- **opinionated**: clear operating rules beat vague instructions
- **auditable**: outputs should be inspectable and improvable
- **tool-aware**: skills should assume real workflows, not just chat
- **portable**: useful across different agent runtimes where possible

## Repository structure

```text
skills/
  <skill-name>/
    SKILL.md
    examples/
    assets/