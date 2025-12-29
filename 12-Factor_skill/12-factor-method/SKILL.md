---
name: 12-factor-method
description: Guide for implementing the 12-Factor Method for building production-ready AI agents. Use this skill when the user asks to "implement the 12 Factor Method", build micro-agents, create production-ready AI systems, or needs guidance on agent architecture following 12-Factor principles. The methodology advocates for small, focused agents embedded within deterministic systems rather than monolithic "loop until solved" agents. Covers natural language to tool calls, prompt ownership, context engineering, control flow, error handling, human-in-the-loop, and stateless design patterns.
---

# 12-Factor Method for AI Agents

## Overview

The 12-Factor Method is an architectural approach for building production-ready AI agents by embedding small, focused micro-agents within mostly deterministic software systems. This ensures reliability, maintainability, and scalability by giving developers full control over prompts, context structure, error handling, and control flow.

**Core Philosophy**: "Agents don't follow the 'here's your prompt, here's a bag of tools, loop until you hit the goal' pattern. Rather, they are comprised of mostly just software" with LLM steps strategically placed at key decision points.

## The 12 Factors (Quick Reference)

1. **Natural Language to Tool Calls** - Convert natural language to structured JSON
2. **Own Your Prompts** - Full control over prompt templates
3. **Own Your Context Window** - Custom context formats optimized for your use case
4. **Tools Are Structured Outputs** - Clean separation: LLM decides what, code controls how
5. **Unify Execution State** - Single source of truth for all state
6. **Launch/Pause/Resume** - Simple APIs for workflow management
7. **Contact Humans with Tools** - Human interaction as tool calls
8. **Own Your Control Flow** - Custom control structures for specific needs
9. **Compact Errors into Context** - Enable self-healing through error feedback
10. **Small, Focused Agents** - 3-10 steps max per agent
11. **Trigger from Anywhere** - Events, crons, webhooks support
12. **Stateless Reducer** - Agents as pure functions

## Reference Documentation

### For Quick Understanding
**Read**: `references/core_principles.md`
- High-level philosophy and principles
- Examples of each factor with concrete use cases
- Why micro-agents vs monolithic agents
- Critical problems solved (context window, framework lock-in, production reliability)

### For Complete Implementation
**Read**: `references/comprehensive_guide.md`
- Full implementation guide with code examples
- API reference and core classes
- Architecture patterns and system components
- Testing strategies and deployment guidance
- Production-ready code samples

### For Learning Path
**Read**: `references/learning_path.md`
- Week-by-week learning schedule (12 weeks)
- Specific goals and metrics for each phase
- Common pitfalls to avoid
- Success metrics and best practices
- Resources for continued learning

### For System Prompt Template
**Read**: `references/system_prompt_template.md`
- Ready-to-use Claude system prompt for 12-Factor agents
- Structured XML context format
- Required output format specifications
- Critical escalation tools and protocols

### For Deep Dive
**Read**: `references/architecting_details.md`
- Comprehensive theoretical background
- Detailed explanation of each factor
- Context engineering techniques
- Advanced patterns and strategies
- Real-world examples and case studies

## Implementation Workflow

1. **Start with Core Principles**: Read `core_principles.md` to understand the philosophy
2. **Choose Your Path**:
   - **Quick Implementation**: Use `system_prompt_template.md` and `comprehensive_guide.md`
   - **Structured Learning**: Follow `learning_path.md` week by week
   - **Deep Understanding**: Study `architecting_details.md` for theory
3. **Build Incrementally**: Start with one micro-agent, perfect it, then expand
4. **Test Everything**: Implement comprehensive testing from day one

## Key Success Criteria

- Agent Success Rate: > 95% for well-defined tasks
- Keep agents focused: 3-10 steps maximum
- Deterministic code for critical operations
- Human approval for high-stakes actions
- Context window optimization is critical
- Error handling with self-healing capability

## When to Use Each Pattern

**Use Micro-Agents For**:
- Customer review triage (1 focused task)
- Safety hazard detection (single analysis)
- Data classification (structured output)
- Decision points in workflows

**Use Deterministic Code For**:
- Database operations
- API calls and external integrations
- Business logic execution
- State persistence

**Use Human-in-the-Loop For**:
- High-value decisions (>$500 refunds)
- Safety-critical operations
- Policy exceptions
- Edge cases requiring judgment
