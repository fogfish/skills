# Nanobot Architecture Reference

Full API for `github.com/kshard/thinker/agent/nanobot`.

## Core Interface

Every bot — whether a leaf `BotReAct` or a composite `BotThinkReAct` / `BotReflect` — satisfies the same interface:

```go
type Bot[S, A any] interface {
    Prompt(ctx context.Context, input S, opt ...chatter.Opt) (A, error)
}
```

This uniform interface makes bots freely composable as arguments to other bots.

The Kleisli arrow type `Arr[S]` is central to composing stateful pipelines. The arrow is morphisms of the state with effects (it lifts the result of the bot into the state and evaluates side-effect over this state):

```go
type Arr[S any] func(context.Context, S, ...chatter.Opt) (S, error)
```

`Arr[S]` satisfies `Bot[S, S]` through its `Prompt` method, making arrows directly composable with bots.

---

## Runtime

The `Runtime` bundles everything the bot constructors need. Build it once and share across all bot declarations.

```go
type Runtime struct {
    FileSystem fs.FS              // prompt template files
    LLMs       LLMs               // model registry (autoconfig implements this)
    Registry   *command.Registry  // MCP tool registry (optional)
    Chalk      Chalk              // progress reporter (optional)
}

// Construction
rt := nanobot.NewRuntime(fs, llm)
rt = rt.WithStdout(chalk.Stdout)      // enable TUI progress
rt = rt.WithRegistry(reg)             // attach shared MCP tools available to each bot
```

The `LLMs` interface has one method: `Model(name string) (chatter.Chatter, bool)`. `autoconfig.MustFromConfig("~/.iqrc")` returns a value that satisfies it. The config file maps LLM tier names (`micro`, `small`, `medium`, `large`, `xlarge`) to provider endpoints.

---

## Kleisli State Algebra

The nanobot package is built on a Kleisli category over the error monad:

```
M A = Context → Either(Error, A)
```

Objects are types; morphisms `S ⇝ S` are Kleisli endomorphisms on the shared blackboard `S`.

### Core Types

```go
// Eval is the Kleisli endomorphism S ⇝ S
type Eval[S any] = func(context.Context, S) (S, error)

// Lens is the "put" half of a lens: folds bot output A into state S
type Lens[S, A any] = func(S, A) S

// Eff bundles Lens and Eval; both fields are optional
type Eff[S, A any] struct {
    Lens Lens[S, A]
    Eval Eval[S]
}

// EffLens constructs an Eff with only a custom Lens setter; Eval defaults to
// the identity. Both type parameters are inferred from lens.
func EffLens[S, A any](lens Lens[S, A]) Eff[S, A]

// EffEval constructs an Eff with only a custom Eval function; Lens defaults to
// the auto-derived lens. S is inferred from eval; A must be specified:
func EffEval[A, S any](eval Eval[S]) Eff[S, A] 

// Arr is the Kleisli arrow S ⇝ S, satisfies Bot[S, S]
type Arr[S any] func(context.Context, S, ...chatter.Opt) (S, error)
```

### Lifting Functions

```go
// Arrow lifts Bot[S, A] into Arr[S] using Eff
// If no Eff given, Lens is auto-derived from the first field of type A in S
Arrow[S, A any](bot Bot[S, A], eff ...Eff[S, A]) Arr[S]

// Bind is shorthand for Arrow with only Eval; Lens auto-derived
Bind[S, A any](bot Bot[S, A], eval Eval[S]) Arr[S]

// Lift injects a plain Eval[S] into Arr[S] (no bot call)
Lift[S any](eval Eval[S]) Arr[S]

// Pure lifts a deterministic function into Bot[S, A] (no LLM call)
Pure[S, A any](f func(context.Context, S) (A, error)) Bot[S, A]
```

### Conditional Execution

```go
// When wraps an arrow with a predicate; runs only if pred(s) returns true
func (f Arr[S]) When(pred func(S) bool) Arr[S]
```

**Use for:** Conditional steps in pipelines: "if field is empty, fill it".

---

## BotReAct[A, B]

A single-file prompt agent. Loads the template from `file`, renders it with input `A`, calls the LLM (optionally invoking MCP tools in a loop), and decodes the response into `B`.

```go
// Constructors
nanobot.NewReAct[A, B](rt *Runtime, file string) (*BotReAct[A,B], error)
nanobot.ReAct[A, B](rt *Runtime, file string) *BotReAct[A,B]  // panics on error

// Configuration methods (builder-style, each returns *BotReAct)
.WithMemory(memory thinker.Memory)
.WithRegistry(r *command.Registry)

// Prompt drives one full ReAct cycle
func (bot *BotReAct[A,B]) Prompt(ctx, input A, opt ...chatter.Opt) (B, error)
```

**When to use:** Any single-step LLM call. The atom of all more complex patterns. Autonomous patterns for thinking, acting, and observing

**Output types `B`:**
- `string` — `format: text` in the prompt file
- Named `type Foo string` — same, decoded via `encoding.TextUnmarshaler` or reflection
- Any type: `MyType` — `format: json` with a full schema block
- `[]string` — `format: json` with `schema.reply.type: array, items.type: string`
- Any slice: `[]MyType` — `format: json` with a full schema block

---

## Seq[S]

Composes Kleisli arrows left-to-right over a shared state. Each step receives the state produced by the previous one; the first error short-circuits the pipeline.

```
Seq : (S ⇝ S) ∘ (S ⇝ S) ∘ … ∘ (S ⇝ S)
  = f >=> g >=> h
  : s ↦ let s₁ = f(s), s₂ = g(s₁) in h(s₂)
```

```go
nanobot.Seq[S any](steps ...Arr[S]) Arr[S]
nanobot.NewSeq[S any](steps ...Arr[S]) (Arr[S], error)  // always returns nil error

// Prompt runs steps sequentially; returns final S or error
func (bot Arr[S]) Prompt(ctx, input S, opt ...chatter.Opt) (S, error)
```

**Use for:** Multi-step sub-agent where each step transforms the shared blackboard. Compose arrows created with `Arrow`, `Bind`, `Lift`, or nested `Seq`/`Reflect`/`ThinkReAct`.

**Example:**
```go
pipeline := Seq(
    Arrow(botExtract, EffLens(lensA)),
    Arrow(botTransform, EffEval[B](evalSideEffect)),
    Lift(func(ctx context.Context, s State) (State, error) {
        // pure transformation
        s.Processed = true
        return s, nil
    }),
)
```

---

## Think and BotThinkReAct[S, T]

Plan-then-execute (scatter/gather pattern). The `think` bot produces `[]T` (a task list), then the `react` arrow executes each task independently, and results are gathered back into the outer state.

### Think Helper

```go
// Think lifts Bot[S, []A] into Bot[S, []T] by applying scatter to each element
Think[S, A, T any](bot Bot[S, []A], scatter func(S, A) T) Bot[S, []T]
```

### BotThinkReAct

```
ThinkReAct : Bot[S, []T] × Arr[T] × (S × []T → S)? → Arr[S]
  : s ↦ gather(s, [react(tᵢ) | tᵢ ∈ think(s)])
```

```go
nanobot.NewThinkReAct[S, T](rt, think Bot[S,[]T], react Arr[T], gather ...func(S,[]T)S) (*BotThinkReAct[S,T], error)
nanobot.ThinkReAct[S, T](rt, think, react, gather...) *BotThinkReAct[S,T]  // panics on error

// Prompt runs think once, then react for each task; returns S
func (bot *BotThinkReAct[S,T]) Prompt(ctx, input S, opt ...chatter.Opt) (S, error)
```

**Parameters:**
- `think Bot[S, []T]` — produces the task list from outer state `S`. Each task is already in the inner state type `T`.
- `react Arr[T]` — a full Kleisli arrow (can be `Seq`, `Reflect`, nested `ThinkReAct`) that processes each task.
- `gather func(S, []T) S` — optional fold that merges results back into outer state. Default is auto-derived lens for []T.

**Unlike Seq:** Each task starts from an independent projection, not from the output of the previous task.

**Use for:** Multi-query search, processing a list of URLs, iterating over dynamic sub-tasks.

**Example with Pure:**
```go
// When task list already exists in state
ThinkReAct(rt,
    Think(
        Pure(func(_ context.Context, s S) ([]string, error) { 
            return s.Urls, nil 
        }),
        func(s S, url string) TaskState { 
            return TaskState{Url: url, Context: s.SharedData} 
        },
    ),
    reactPipeline,  // Arr[TaskState]
    func(s S, results []TaskState) S {
        s.Results = results
        return s
    },
)
```

---

## Vote, Judge, and BotReflect[S]

Guard/judge loop that forces a "Review" state before a "Done" transition. The judge evaluates the current state and returns a `Vote` with accept/reject decision. On rejection, the corrector runs and the loop retries.

### Vote Type

```go
// Vote carries the state updated with judge feedback and the decision signal
type Vote[S any] struct {
    State    S      // blackboard with judge's critique
    accepted int    // +1 accept, 0 retry, -1 hard reject (private)
}

// Methods to set the decision
func (v *Vote[S]) Accept()  // +1: accept, return Vote.State as final
func (v *Vote[S]) Revise()  // 0: retry, forward Vote.State to corrector
func (v *Vote[S]) Reject()  // -1: hard reject, return error immediately
```

### Judge Helper

Lifts `Bot[S, A]` into `Bot[S, Vote[S]]` using `Eff[Vote[S], A]`:

```go
Judge[S, A any](bot Bot[S, A], eff ...Eff[Vote[S], A]) Bot[S, Vote[S]]
```

**Default behavior:**
- `eff.Lens` injects `A` into `Vote[S].State` via auto-derived lens. Default is `Vote[S].State ∘ forProduct1[S, A]`.
- `eff.Eval` sets `Vote.Accepted`. Default is always-accept (+1).

**Example:**
```go
judge := Judge(
    botEvaluate,  // Bot[State, Critique]
    EffEval[Critique](func(ctx context.Context, v Vote[State]) (Vote[State], error) {
        if v.State.Score > 0.8 {
            v.Accept()
        } else if v.State.Score > 0.5 {
            v.Revise()
        } else {
            v.Reject()
        }
        return v, nil
    }),
)
```

### BotReflect

```
Reflect : (S → Vote[S]) ∘ [if rejected: (S ⇝ S) ∘ retry]
```

```go
nanobot.NewReflect[S](rt, judge Bot[S,Vote[S]], correct Arr[S]) (*BotReflect[S], error)
nanobot.Reflect[S](rt, judge, correct) *BotReflect[S]  // panics on error

.WithAttempts(n int)  // max iterations (default 1)

// Prompt runs the reflection loop; returns final S or error
func (bot *BotReflect[S]) Prompt(ctx, input S, opt ...chatter.Opt) (S, error)
```

**Flow:**
1. Judge evaluates current state `s` → `Vote[S]`
2. If `Vote.Accept()`: return `Vote.State` as final
3. If `Vote.Revise()`: pass `Vote.State` (with critique) to `correct` arrow, retry
4. If `Vote.Reject()`: return error immediately
5. If attempts exhausted: return error

**Use for:** Quality gates, format validation, factual consistency checks.

---

## Map

Transforms the output type of a `Bot[S, A]` into `Bot[S, B]`. The function `f` receives the current blackboard `S` and the bot's output `A`, and returns `B`.

```go
Map[S, A, B any](bot Bot[S, A], f func(S, A) B) Bot[S, B]
```

**Denotation:**
```
Map(bot, f) : s ↦ f(s, bot.Prompt(s))
```

**Use for:** Type transformations, extracting specific fields, wrapping outputs in new types. Use only in the context of sub-agent to make re-usable block.

**Example:**
```go
// Transform Bot[State, RawResult] into Bot[State, ProcessedResult]
processed := Map(botRaw, func(s State, raw RawResult) ProcessedResult {
    return ProcessedResult{
        Value: raw.Data,
        Timestamp: time.Now(),
        Context: s.Metadata,
    }
})
```

---

## Chalk Integration

The `Chalk` interface provides structured progress output:

```go
type Chalk interface {
    Sub(context.Context) context.Context          // create child context
    Task(context.Context, string, ...any)          // start a named task
    Done(...string)                                // complete the current task
    Fail(error)                                    // mark current task failed
    Printf(format string, args ...any)            // print a line
}
```

Use `chalk.Stdout` from `github.com/fogfish/chalk` as the implementation. A `BotReAct` agent only shows progress when the prompt file has a non-empty `name:` field.

---

## Prompt File Discovery

`NewReAct` reads the file from `rt.FileSystem` at the given path. The path is relative to the embedded FS root. Use `//go:embed prompts/*.md` and store prompts as `prompts/<name>.md`. Pass `"prompts/<name>.md"` to the constructor.
