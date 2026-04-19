# Prompt Template Reference

Prompt files are Markdown documents stored under `prompts/` and embedded with `//go:embed`. Each file has a optional YAML front-matter block followed by a `text/template` body.


## Front-Matter Spec

```yaml
---
# Shown in chalk progress output. 
# (default: undefined - progress is silenced for this bot).
name: Human-readable task name      

# Required. LLM tier alias from autoconfig (See ./references/llm-tiers.md).
# micro | small | medium | large | xlarge | base
# (default: base)
runs-on: micro

# Output format: text | json
# (default: text)
format: text

# LLM retry budget, important for json
# (default: 3).
retry: 3

# Enable LLM I/O to stderr
# (default: false)
debug: false

# JSON schema to specify input and output
schema:

  # Informative schema for the input object, acting as specification.
  # In the future releases, the input schema became declarative to harden input
  # for type safety at compile and runtimes.
  # 
  # Declare only the fields used in the prompts. Do not over engineer.
  input:
    type: object
    required: [name]
    properties:
      name:
        type: string
        decision: name of the person


  # Declarative schema to enforse output of known and expected type (type-safety).
  # Do not over-engineer, keep types simple and use it to express semantic of
  # textual data rather strict specification. Very complex and hierarchically nested
  # types increase probability of hallucintaions.
  #
  # Example below is simple array of strings but objects are supported as well
  reply:
    type: array
    description: what the array contains
    items:
      type: string
      description: one item description

# List of MCP servers and tools available to agent 
servers:
  # Example of HTTP-based MCP server 
  - name: toolname
    url: http://host:port/mcp

  # Example of STDIO MCP server
  # Always used array to declare the command
  - name: localtool
    command:
      - npx
      - -y
      - @mcp/server
      - /path
---
```

## Template Body

The body is rendered by `text/template` with the bot's input struct `S` as `.`.

### Field Access

```
{{ .FieldName }}          direct field value or getter
{{ .Nested.Field }}       nested structs
{{ printf "%d" .Count }}  Go fmt formatting
```

### Iteration

```
{{ range .Items }}
- {{ . }}
{{ end }}

{{ range $i, $v := .Items }}
{{ $i }}: {{ $v }}
{{ end }}
```

### Conditionals

```
{{ if .OptionalField }}
Optional section: {{ .OptionalField }}
{{ end }}
```

### Multi-line Values

For types that may contain markdown or multi-line text, wrap in a fence block. For example, type like `Document` into `<input-document>...</input-document>`   

## Debugging Prompts

Set `debug: true` in front-matter to log raw LLM request/response to stderr. Always remove `debug: true` before committing.


## Embedding Prompt Files

Use Go `embed` package for deploying prompts within the binary unless other method is specified by user.

```go
//go:embed prompts/*.md
var fs embed.FS

// Pass to runtime — bots load from this FS
env = nanobot.NewRuntime(fs, llm)

// Reference in bot constructor
bot = nanobot.MustReAct[*State, string](env, "prompts/summarize.md")
```

The path string passed to `NewReAct` must match the path as seen by the embedded FS (relative to module root, using `/` separators).

