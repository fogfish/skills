---
name: go-agent
description: Implement Go native agents using github.com/kshard/thinker/agent/nanobot. Use when user is asked about "Agent in Go", "Go agentic workflow", "Go-native agents", "Multi-agent system" etc.
argument-hint: 'Describe the workflow you want to implement (e.g., "research agent that searches and summarizes")'
---

# Go-native agents

Implement production-quality multi-agent system in Go using `github.com/kshard/thinker/agent/nanobot`. This skill covers the full lifecycle using the itterative design approach: **architecture → flow → blackboard state design → skeleton and bot composition → prompt authoring → side effects → checkpointing**.

`thinker` is a minimal, strictly typed Go framework that represents all agents/sub-agents as composable side-effect functions (`ƒ: A ⟼ B`), referenced as bots and orchestrated by language models. The inputs and outputs are controlled by deterministic guradrails written in Go. The Markdown controls the language model behaviour, and which MCP servers to connect. The `agent/nanobot` toolkit provides execution environment, bindings and composable bot patterns that cover the most common use-cases.

## Step-by-Step Procedure

Use the universal checklist below to design and implement the agentic workflow. Each step has detailed instructions and examples.

- [ ] **Purpose**: The purpose of the agent (agentic system) is clear and well defined by user.
- [ ] **Formal specification**: Decomposes problems into executable agent specifications using a formal algebra.
- [ ] **Architecture blueprint**: The architecture blueprint is created and reviewed with the user.
- [ ] **Runtime**: The runtime environment is setup with necessary dependencies.
- [ ] **Blackboard design**: The blackboard state struct is designed with exactly one field per bot output type.
- [ ] **Agents skeleton**: The skeleton of the agents (bots) is declared in Go.
- [ ] **Agentic patterns**: The agentic patterns and compositions are identified and implemented.
- [ ] **Prompt drafts**: Every bot has a corresponding `prompts/*.md` file with valid YAML front-matter.
- [ ] **JSON output**: Prompts with `format: json` have a defined `schema.reply` for structured output decoding.
- [ ] **Prompts embedding**: All prompt files are embedded with `//go:embed prompts/*.md`
- [ ] **Unit tests**: Unit tests are implemented for each bot using mock LLM responses.
- [ ] **Tests passing**: All unit tests are passing before proceeding to the next steps.
- [ ] **Control flow**: Go idiomatic control flow is implemented and conditionals to orchestrate the bot calls and data flow between them.
- [ ] **Side-effects**: The side-effects are implemented in the workflow where deterministic logic or tool calls are needed.
- [ ] **Review prompts**: Review all prompts and make final edition focusing on the task-role-action structure.
- [ ] **LLM tier assignment**: Each prompt has the LLM tier assigned based on the expected complexity of the task.
- [ ] **Connect MCP servers**: The necessary MCP servers are connected based on the requirements of the workflow.
- [ ] **Observability**: The logging is implemented using `github.com/fogfish/chalk`.
- [ ] **Checkpointing**: The checkpointing is implemented to allow resuming from the last checkpoint in case of errors or interruptions.
- [ ] **Conditional running**: The bots are wrapped with conditionals to skip execution if the output already exists in the state (for idempotency).
- [ ] **Building and running**: The agentic system is built and run successfully, achieving the defined purpose.


### 1. Architecture: Understand the Purpose and Use Cases

Well defined purpose and use-cases of the agentic system is required before diving into implementation. Ask user the follow-up questions to clarify the specific purpose of agent the user wants to build **until you have a complete understanding** about use-cases, expected input and output.

Refect about defined purpose and use-cases mapping them into existing well known patterns, processes and workflows.

Decomposes problems into executable agent specifications using a formal algebra (see [agent algebra reference](./references/agent-algebra.md)). Every plan you produce is a composition of typed primitives from the Kleisli agent algebra. Create a formal specification (file at the workspace) of the agentic system using the algebra primitives — a blueprint that an agent runtime can execute directly. 

Conduct the review of the architecture blueprint with the user before proceeding to implementation. Communicate the overall agentic process and approach to the user using the formal specification. Explicitly ask confirmation from the user that the architecture is clear and acceptable before proceeding to the next step.


### 2. Setup the runtime

The runtime environment for production grade agents requires atleast following packages:
1.  `embed`: `embed.FS` for loading prompt templates from markdown files.
2.  `github.com/kshard/chatter/provider/autoconfig`: `autoconfig.MustFromConfig("~/.iqrc")` for loading instances of LLM clients.
3. `github.com/fogfish/chalk`: `chalk.Stdout` for I/O orchestration, human readbale logging and debugging. 
4. `github.com/kshard/thinker/agent/nanobot`: `nanobot.NewRuntime` actual runtime environment for bot execution.

```go
package main

import (
    "embed"

    "github.com/fogfish/chalk"
    "github.com/kshard/chatter/provider/autoconfig"
    "github.com/kshard/thinker/agent/nanobot"
)

//go:embed prompts/*.md
var fs embed.FS

var (
    llm = autoconfig.MustFromConfig("~/.iqrc")
    env = nanobot.NewRuntime(fs, llm).WithStdout(chalk.Stdout)
)

func main() { chalk.Start(eval) }

func eval(ctx context.Context, path string, r io.Reader, w io.Writer) error {
  // chalk declares the eval as an entry point for reading an input file and writing output.
  // chalk is able to orchestarte stdin and stdout across multiple file systems so that 
  // the agent can read/write from/to different sources (e.g. local fs, s3, gcs, etc.) without changing the code. 
}
```


### 3. Blackboard: Design the Agent State

The key design principle and outcome of this step is to create a central state struct that flows through all bots (aka Blackboard). The formal specification from the previous step should guide the design of the blackboard state. The struct must have **exactly one field per bot output type**. This way, the state serves as a blackboard for all agents/sub-agents to read/write information, and the Go type system ensures that the data flow is correct and explicit.

```go
type Input int

type Query string

type State struct {
  Input   Input   `yaml:"input,omitempty"`
  Queries []Query `yaml:"queries,omitempty"`
  // ...
}
```

If you are able to identify sub-agents in the workflow that acts independently from the main agent, you can create separate blackboard for them (own state structs) so that subagents are decoupled from each other and from the main agent.

Hints:
- One field per bot output; field type must match the bot's output type `B`
- Use `yaml` tags for checkpointing (`chalk.Commit`/`chalk.Recover`)
- `omitempty` prevents stale partial checkpoints from bloating disk writes


### 4. Agent skeleton: declare nanobots

The key design principle and outcome of this step is to declare atomic Reason-and-Act (ReAct) agents for a cyclical process of **thinking**, **acting**, and **observing** so that this loop continues iteratively until the defined goal is achieved. The formal specification from the previous step should guide the design of the blackboard state.

Declare bots as package-level `var` using `ReAct[S, A]` constructors. They are stateless objects; the state flows through `Prompt()` calls at runtime.

```go
var (
  botPlan    = nanobot.ReAct[*State, []Query](env, "prompts/plan.md")
  botSearch  = nanobot.ReAct[*State, []Link](env, "prompts/search.md")
  botSummary = nanobot.ReAct[*State, Summary](env, "prompts/summarize.md")
)
```

See [architecture reference](./references/architecture.md) for full API details.

#### 4.1 Agentic patterns and compositions

Identify and componse atomic agents into higher-order patterns. The library provides few composable patterns (see [architecture reference](./references/architecture.md) and [patterns reference](./references/patterns.md) for details):
* `Seq` sequence of agents where the output of one is the input of the next.
* `ThinkReAct`: Plan-and-Execute pattern implements two phase flow where the agent first plans a sequence of actions based on the current state and then executes those actions, potentially updating the state after each action.
* `Reflect`: Reflect implements judge-then-correct loop. A judge bot evaluates the current state. On rejection, a react bot corrects the state, and the review loop retries.

Keep the composition modular and explicit. Each high-order sub-agent should have a clear purpose and be focused on a specific step of the workflow. It is a cirtical point in the design. Sub-agents are componsed using the high-order patterns, but **Go idiomatic** control flow (e.g. loops, conditionals) should be used for the overall orchestration and data flow between the sub-agents. This way, the control flow is explicit and debuggable, and the agentic system is more maintainable and extensible in the long run.


### 5. Prompts: draft prompts

For each agent, draft the prompt template in a markdown file focusing on type coherence between input/output and blackboard definiton. Store each prompt as `prompts/<name>.md` (embed them with `//go:embed prompts/*.md`), use sub-folders to group prompts for different sub-agents. Each prompt file should have a frontmatter with `runs-on` and `format` fields, and the body is the template for the prompt content.

See [prompt template reference](./references/prompt-templates.md) for the full spec.

**Minimum viable prompt file:**

```markdown
---
name: Human-readable step name   # shown in chalk progress output
runs-on: base                    # selects the LLM tier
format: text|json                # text → string output; json → structured output
---
You are an expert in {{ .SomeField }}.

Perform the task: {{ .Input }}
```

**JSON-output prompt (array reply):**

```markdown
---
name: Generate queries
runs-on: medium
format: json
schema:
  reply:
    type: array
    items:
      type: string
      description: search queries
---
Generate {{ .Depth }} queries for: {{ .Input }}
```

Template data is the input struct `S` (blackboard) rendered via `text/template`. Every exported field is accessible as `{{ .FieldName }}`, ranges work: `{{ range .Items }}{{ . }}{{ end }}`.

Keep the output types simple (string, array of strings, simple struct) and use the `schema` field in front-matter to control how the output is decoded. Complex and deeply nested types increase the probability of hallucinations.

#### 5.1 Unit testing

Absence of type-safe notation in prompt templates makes them prone to errors. Implement unit tests for each bot using mock LLM responses to ensure that the prompts are correctly defined, the input and output types are coherent, and decoding works as expected. 

Writing unit tests requires:
* Mock LLM to return expected values, use `autoconfig.MustMock(...)`.
* Assets for mock responses, use `github.com/fogfish/it/v2`. 

Below is an unit test example for a bot that expects `{{ .Subject }}` as input and returns `[]Query` as output.

```go
import (
	"context"
	"testing"

	"github.com/fogfish/it/v2"
	"github.com/kshard/chatter/provider/autoconfig"
	"github.com/kshard/thinker/agent/nanobot"
)

func TestBotPrepare(t *testing.T) {
	expect := []Query{"q"}
	llm := autoconfig.MustMock(expect)
	env := nanobot.NewRuntime(fs, llm)
	bot := nanobot.ReAct[*State, []Query](env, "prompts/prepare.md")

	value, err := bot.Prompt(context.Background(), &State{Subject: "Subject"})
	it.Then(t).Should(
		it.Nil(err),
		it.Seq(value).Equal(expect...),
	)
}
```

All unit tests should be passing before proceeding to the next steps. This way, you can be confident that the prompts are correctly defined and the bots will work as expected at runtime.


### 6. Control flow

Implement the control flow in the `eval` function using Go idiomatic constructs (e.g. loops, conditionals). Call bots with `Prompt(ctx, state)` passing the blackboard state. The bot reads from the state, performs the LLM call, and writes its output back to the state.

```go
func eval(ctx context.Context, path string, r io.Reader, w io.Writer) (err error) {
  state := &State{Input: ...}

  state.Links, err = botSearch.Prompt(ctx, state)
  if err != nil {
    return err
  }

  state.Summary, err = botSummarize.Prompt(ctx, state)
  if err != nil {
    return err
  }
}
```

Preferably use the `github.com/fogfish/chalk` as the execution container for the bot. It provides effective I/O loop for reading input and writing output, and also allows you to orchestrate the execution across different file systems (e.g. local fs, s3, gcs) without changing the code.

```go
func main() { chalk.Start(eval) }

func eval(ctx context.Context, path string, r io.Reader, w io.Writer) error {
  // ...
}
```

`chalk.Start` handles:
- Walking input files/directories (`path`, `r`) passed as CLI args
- Rendering a rich progress TUI via `chalk.Stdout`
- Managing sub-context nesting for nested tasks


### 7. Side-effects

The library has defined a Kleisli category over the error monad allowing side effect exectuion. This means that you can perform side effects (e.g. API calls, database updates) in between of agent calls in a way that is explicit and testable. Architecture blueprint and formal specification should guide the design of the side effects in the workflow. Evaluate them and identify where the determinstic logic needs to be implpemented within Go control flow and where the side-effects are needed. 

For example, side-effectes allows to inject deterministic logic in between of LLM calls:
* the logic is part of sub-agent that is re-usable across different workflows.
* the logic substitutes the tool invocation.
* the logic is strict guardrail for the LLM output (e.g. validation, filtering, formatting).

See [architecture reference](./references/architecture.md) for definition of effects.

```go
nanobot.Seq(
  nanobot.Lift(effect1),
  nanobot.Bind(
    nanobot.ReAct[*State, ...](...),
    effect2,
  ),
)
```


### 8. Prompts: final edition

Perform prompt authoring iteratively, starting with a skeleton version of the prompt and gradually adding more details and instructions based. See [prompt template reference](./references/prompt-templates.md) for the full spec.

Each prompt should have a clear purpose and be focused on a specific step of the workflow. The prompt should instruct the LLM on
1. Role: what role to play (e.g. "You are an expert in X")
2. Task: what task to perform (e.g. "Generate search queries for the following research topic")
3. Requirements: any specific requirements or constraints on what and how information to use (e.g. "Each query must be ≤10 words. Return only the queries — no explanation.")
4. Input: what information is provided (e.g. "{{ .Input }}")
5. Output: what output to produce (e.g. "Return only the queries — no explanation.")

Use the `format` field in front-matter to control how the output is decoded (e.g. `text` for string output, `json` for structured output).

Assign the LLM tier for each prompt using the `runs-on` field in front-matter. The runtime will use the corresponding LLM provider for all calls from that prompt. See [LLM tiers reference](./references/llm-tiers.md) for the full spec.

**LLM tiers** are configured in `~/.iqrc` and selected by the `runs-on` field in prompt front-matter. For example, if `runs-on: medium`, the runtime will look for a provider named `medium` in the config and use that LLM for all calls from that prompt. See [LLM tiers reference](./references/llm-tiers.md) for the full spec.


### 8.1 Connect to MCP servers

Declare tool servers in the prompt file front-matter; nanobot wires them automatically:

```markdown
---
name: Web researcher
runs-on: large
format: text
servers:
  - name: websearch
    url: http://localhost:3000/mcp
  - name: filetool
    command: npx -y @modelcontextprotocol/server-filesystem /tmp
---
Use the websearch tool to find information about {{ .Topic }}.
```

Or attach a shared registry at runtime:

```go
reg := command.NewRegistry()
reg.ConnectUrl("websearch", "http://localhost:3000/mcp")
env = nanobot.NewRuntime(fs, llm).WithRegistry(reg).WithStdout(chalk.Stdout)
```


### 9. Observability

Use `github.com/fogfish/chalk` library for human readable logging and debugging. The library is designed to work seamlessly with `nanobot` runtime, allowing you to log the progress of the agents, the inputs and outputs of each bot, and any errors that occur during execution. Use `chalk.Stdout` as the output for the runtime to enable this integration. `thinker` library does not support logging inside all bots directly:
* `ReAct` performs logging only if prompt defined `name` field in front-matter.
* `ThinkReAct` perform logging if type `T` has String() method.
* `Map`, `Seq` does not perform logging at all, relying on the logging of the underlying bots.


### 10. Checkpointing

Use `github.com/fogfish/chalk` library to perform checkpointing of the agent state. This allows you to save the state of the agent at any point in time and resume execution from that point later if needed. This is especially useful for long-running agents or when you want to be able to recover from errors without losing all progress. Use `chalk.Checkpoint(state)` to save the state and `chalk.Resume()` to resume execution from the last checkpoint.

Declare commit function.

```go
func effCommit(ctx context.Context, s *State) (*State, error) {
	chalk.Commit(string(/** unique key **/), s)
	return s, nil
}
```

Declare automatic checkpointing after each bot execution and skipping the bot execution if the checkpoint already exists for the given state. This way, if the agent execution is interrupted for any reason (e.g. error, manual stop), you can resume it later without losing progress.

```go
nanobot.Bind(
  bot,
  effCommit,
).When(func(s *State) bool { return /* condition */ })
```

Recover from errors by resuming from the last checkpoint.

```go
func eval(ctx context.Context, path string, r io.Reader, w io.Writer) error {
  	state := chalk.Recover(/** unique key **/, &State{})
}
```

Usage of unique keys for checkpointing is arbitrary. It can be based on the input file name, a specific field in the state, input data, etc. The key point is that it should uniquely identify the state of the agent at a given point in time so that you can resume from that exact point later. Sub-agents can have their own checkpoints with their own unique keys, allowing you to have multiple checkpoints for different parts of the workflow or different sub-agents, giving you more flexibility in how you manage the state and recover from errors.
