
# LLM Tiers

## Overview

LLM tiers represent different model sizes and capabilities, each optimized for specific types of tasks. Proper tier selection is critical for balancing cost, latency, and quality in agentic workflows.

**Golden Rule**: Always use the **smallest tier that reliably solves the task**. Over-provisioning wastes resources; under-provisioning degrades quality.

---

## Tier Semantics

| Tier     | Typical Use                            | Cost/Speed             | Context Window |
| -------- | -------------------------------------- | ---------------------- | -------------- |
| `micro`  | Parsing, extraction, formatting        | Fastest / cheapest     | Small          |
| `small`  | Classification, light reasoning        | Fast / cheap           | Medium         |
| `medium` | Multi-step reasoning, query generation | Balanced               | Large          |
| `large`  | Deep analysis, synthesis, long-context | Slowest / most capable | Very Large     |
| `xlarge` | Complex synthesis, research tasks      | Most expensive/capable | Largest        |

---

## Tier Descriptions

### `micro` - Parse & Extract
**Best For**: Deterministic tasks with structured output
- **Examples**:
  - Extracting fields from JSON/XML
  - Converting formats (CSV → JSON)
  - Template filling
  - Simple regex-like pattern matching
  - Splitting or normalizing text
- **Characteristics**: 
  - Minimal reasoning required
  - Clear, well-defined inputs/outputs
  - High throughput scenarios
  - Fails with complex and nested JSON object
- **Anti-patterns**: 
  - Don't use for ambiguous inputs
  - Avoid for tasks requiring world knowledge

### `small` - Classify & Route
**Best For**: Tasks requiring light reasoning or categorization
- **Examples**:
  - Sentiment analysis
  - Intent classification
  - Topic categorization
  - Simple question answering (fact lookup)
  - Entity recognition
  - Language detection
- **Characteristics**:
  - Single-step reasoning
  - Pattern recognition
  - Moderate context understanding
  - Fails with complex and nested JSON object
- **Anti-patterns**:
  - Complex multi-step logic
  - Deep semantic analysis

### `medium` - Reason & Generate
**Best For**: Multi-step reasoning with moderate complexity
- **Examples**:
  - Query generation from natural language
  - Code generation for simple functions
  - Summarization with context awareness
  - Multi-step data transformations
  - Planning simple workflows
  - Generating structured responses
- **Characteristics**:
  - Handles 2-4 reasoning steps
  - Good balance of speed and capability
  - Can follow complex instructions
- **Anti-patterns**:
  - Very long documents (use `large`)
  - Deep research tasks requiring synthesis

### `large` - Synthesize & Analyze
**Best For**: Complex reasoning, synthesis, and long-context tasks
- **Examples**:
  - Analyzing multiple documents
  - Code review and refactoring suggestions
  - Creating detailed reports from data
  - Complex planning and decision-making
  - Nuanced text generation
  - Cross-referencing information
- **Characteristics**:
  - 5+ reasoning steps
  - Handles ambiguity well
  - Excellent context retention
  - Strong at synthesis
- **Use When**:
  - Quality is paramount
  - Task requires deep understanding
  - Multiple sources must be integrated

### `xlarge` - Expert-Level Reasoning
**Best For**: Most complex tasks requiring expert-level analysis
- **Examples**:
  - Deep research and literature review
  - Complex codebase analysis
  - Strategic planning and decision support
  - Creative problem-solving
  - Multi-document synthesis with nuance
- **Characteristics**:
  - Highest capability
  - Best at handling ambiguity
  - Strongest reasoning chains
- **Cost Consideration**: Reserve for tasks where `large` is insufficient

---

## Selection Framework

### Decision Tree

```
Is the task deterministic/rule-based?
├─ YES → micro
└─ NO → Does it need reasoning?
    ├─ NO → micro or small
    └─ YES → How many reasoning steps?
        ├─ 1 step → small
        ├─ 2-4 steps → medium
        ├─ 5+ steps → large
        └─ Expert-level → xlarge
```

### Cost vs. Quality Tradeoff

```
Quality
  ↑
  │                                    ⚫ xlarge
  │                          ⚫ large
  │                 ⚫ medium
  │        ⚫ small
  │  ⚫ micro
  └─────────────────────────────────────────→ Cost/Latency
```

---

## Best Practices

### 1. Start Small, Scale Up
Begin with the smallest tier and only upgrade if quality suffers. Test edge cases at each tier.

### 2. Pipeline Optimization
Use different tiers for different pipeline stages:
```
User Query → [small: classify intent] 
          → [medium: generate query] 
          → [large: synthesize results]
```

### 3. Batch Similar Tasks
Group similar tasks to the same tier for efficient resource utilization.

### 4. Monitor and Iterate
- Track tier performance metrics
- A/B test tier choices
- Adjust based on real-world results

### 5. Context Awareness
Smaller tiers have smaller context windows. If input+output exceeds limits, tier up.

---

## Anti-Patterns to Avoid

❌ **Using `large` for everything** → Wastes money and time  
❌ **Using `micro` for reasoning** → Poor quality results  
❌ **Ignoring context limits** → Truncation errors  
❌ **No tier experimentation** → Missed optimization opportunities  
❌ **Tier selection by intuition** → Use data and testing instead

---

## Machine Configuration

Below are the AWS Bedrock model mappings for each tier:

```
machine micro
        provider provider:bedrock/foundation/converse
        model eu.amazon.nova-micro-v1:0
        region eu-west-1

machine small
        provider provider:bedrock/foundation/converse
        model eu.amazon.nova-pro-v1:0
        region eu-west-1

machine medium
        provider provider:bedrock/foundation/converse
        model global.anthropic.claude-haiku-4-5-20251001-v1:0
        region eu-west-1

machine large
        provider provider:bedrock/foundation/converse
        model global.anthropic.claude-sonnet-4-6
        region eu-west-1

machine xlarge
        provider provider:bedrock/foundation/converse
        model global.anthropic.claude-opus-4-6-v1
        region eu-west-1

machine base
        provider provider:bedrock/foundation/converse
        model global.anthropic.claude-sonnet-4-6
        region eu-west-1
```

**Notes**:
- `base` tier defaults to `large` (Claude Sonnet)
- Configuration uses AWS Bedrock Foundation Model Converse API
- Regional deployment optimizes for latency and availability
- Models are versioned for reproducibility

---

## Quick Reference Card

| Task Type                  | Tier     | Example                         |
| -------------------------- | -------- | ------------------------------- |
| JSON parsing               | `micro`  | Extract email from user JSON    |
| Sentiment detection        | `small`  | Is this review positive?        |
| SQL query generation       | `medium` | Convert question to SQL         |
| JSON generation            | `medium` | Convert raw data to JSON        |
| Code review                | `large`  | Analyze PR for issues           |
| Multi-doc research         | `xlarge` | Compare 10 papers on topic      |
| Simple format conver       | `small`  | CSV to JSON                     |
| Complexi format conversion | `medium` | Large text to JSON              |
| Language detection         | `small`  | What language is this text?     |
| API response composition   | `medium` | Build structured API response   |
| Architecture design review | `large`  | Evaluate system design proposal |
| Strategic analysis         | `xlarge` | Comprehensive market research   |

---

## Further Reading

- [Agent Architecture](architecture.md) - How tiers fit into agent design
- [Patterns](patterns.md) - Common tier usage patterns
- [Prompt Templates](prompt-templates.md) - Tier-specific prompt strategies
