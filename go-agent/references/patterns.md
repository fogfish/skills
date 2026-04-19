# Nanobot Composition Patterns

Annotated, copy-paste-ready patterns for common agentic workflow shapes.

---

## Pattern 1: Linear Pipeline (Seq)

**Shape:** step A cleans/transforms → step B analyses the cleaned output, making a sub-agent for repeatitive tasks.

**Real example:** `botKnowledge` — filter raw web page HTML → extract knowledge from cleaned text.

```go
type State struct {
  Url       Url       `yaml:"url"`
  Document  Document  `yaml:"document,omitempty"`   // ← filter.md writes here
  Knowledge Knowledge `yaml:"knowledge,omitempty"`  // ← extractor.md writes here
}

var botKnowledge = nanobot.Seq(env,
  nanobot.Lift(effFetch)
  nanobot.Arrow(nanobot.ReAct[*State, Document](env, "prompts/filter.md")),
  nanobot.Arrow(nanobot.ReAct[*State, Knowledge](env, "prompts/extractor.md")),
)

knowledge, err := botKnowledge.Prompt(ctx, &State{Url: url})
```

**filter.md** — `runs-on: micro, format: text` — strips noise, returns cleaned `Document`.
**extractor.md** — `runs-on: large, format: text` — extracts facts from `{{ .Document }}`.

The default lens wires `Document` from step 1 back into `State.Document` before step 2 reads it. Both steps receive the same `*State` pointer.

---

## Pattern 2: Plan-and-Execute / Scatter-Gather (ThinkReAct)

**Shape:** planner produces `[]Task`, executor runs each task, outer loop gathers results.

**Real example:** `botSearch` — generate N queries → for each query, fetch + parse search results.

```go
type State struct {
  Subject Subject
  Documents []Document
}

type Search struct {
  Query    Query    `yaml:"query,omitempty"`    // ← planner sets this per task
  Document Document `yaml:"document,omitempty"` // ← effect fills from HTTP
}

var botSearch = nanobot.ThinkReAct(env,
  nanobot.Think(
    nanobot.ReAct[*State, []Query](env, "prompts/prepare.md"),  // think: S → []Query
    func(s *State, q Query) *Search{ return &Search{Query: q} },
  ),
  nanobot.Seq(
    nanobot.Lift(effSearch),
    nanobot.Arrow(
      nanobot.ReAct[*Search, Links](env,  "prompts/websearch.md"), // react: T → Document
    ),
  ),
  func (s *State, r *Search) *State {
    s.Documents = append(s.Documents, r.Document)
    return r
  },
)

state, err := botSearch.Prompt(ctx, &Search{Subject: "..."})
```

**prepare.md** — `format: json, schema.reply: array of strings` — returns N query strings.
**websearch.md** — `format: text` — parses `{{ .Document }}` (raw HTML), returns Document.

After `ThinkReAct` finishes, it accumulate `Document` into the global state.

---

## Pattern 3: Single-Step Analysis (ReAct)

**Shape:** one prompt-file agent that does all reasoning internally.

**Real example:** thinking, acting, and observing using tools and MCPs.

```go
var botTriage   = nanobot.MustReAct[*State, []string](env, "prompts/triage.md")

urls,  err = botTriage.Prompt(ctx, state)
```

`triage.md` uses `format: json` with an array schema — decoded directly to `[]string`.

Both bots receive the full `*State` and can access any field via template `{{ .FieldName }}`.

---

## Pattern 4: Quality Guard (Reflect)

**Shape:** judge evaluates output; corrector fixes it; retry up to N times.

```go
type State struct {
  Document Document
  Feedback Feedback
}

var botJudge = nanobot.Judge(
  nanobot.MustReAct[*DocState, Feedback](env, "prompts/judge.md"),
  nanobot.EffEval[Feedback](
    func(ctx context.Context, v Vote[*DocState]) (Vote[*DocState], error) {
      if strings.Contains(v.State.Feedback, "APPROVED") {
        v.Approve()
        return v
      }
      v.Revise()
      return v
    },
  ),
)

var botWithGuard = nanobot.Reflect(env,
  judge,
  nanobor.Arrow(
    nanobot.MustReAct[*State, Document](env, "prompts/revise.md"),   // corrector: revises draft
  ),
).WithAttempts(3)

state, err := botWithGuard.Prompt(ctx, &State{Document: draft})
```

**judge.md** — evaluates `{{ .Document }}` against criteria, returns `"APPROVED"` or feedback.
**revise.md** — reads `{{ .Document }}` and `{{ .Feedback }}`, returns improved draft.

---

## Pattern 5: Nested Pipelines

Bots are values satisfying `Bot[S, A]`, liftable to `Arr[S]`; compose them however you need.

---

## Pattern 7: Side-Effect Functions

Side effects (`WithEffect`) transform the state between the "set task" and "run react" steps. They are pure Go functions — no LLM involved — used to inject real-world data.

```go
// Signature required by WithEffect:
func effSomething(ctx context.Context, s *State) (*State, error) {
    result, err := someExternalCall(ctx, s.CurrentTask)
    if err != nil {
        return s, err  // return s unchanged; ThinkReAct will propagate error
    }
    s.FetchedData = result
    return s, nil
}
```

Sse `nanobot.Bind` to lift the effect into the arrow `Arr[S]`.

