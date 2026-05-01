# Explain

Create a document that explains a situation — an architecture, a system, a bug, a flow, a process — using rich, graphical evidence. The audience should finish reading and *understand*, not just be told.

## Mindset

Explanation is not proof. There is no before/after to capture. The goal is clarity: show how things are shaped, how they connect, and how they move. Diagrams do most of the heavy lifting; prose ties them together. A reader who skims only the images should still get the gist.

**Lean heavily on graphics.** A page of text with one diagram is a failure of this skill. Aim for multiple diagrams covering different angles: structure, flow, sequence, state, data shape.

## Output location

**ALL resources created by this skill — the showboat document, Mermaid sources, rendered diagrams, screenshots, rodney working state — live under `/tmp`.** Nothing is ever written to the working directory.

Pick a working directory under `/tmp` at the start and stick to it:

```bash
EXPLAIN_DIR="/tmp/explain/$(date +%Y%m%d)-short-topic"
mkdir -p "$EXPLAIN_DIR/assets"
cd "$EXPLAIN_DIR"
```

`cd` into `$EXPLAIN_DIR` *before* running `rodney start` so its `.rodney/` working directory lands there too. Run all subsequent commands from `$EXPLAIN_DIR` (or use absolute paths under it).

## Step 0: Ensure tooling is available

### Check for uvx

```bash
command -v uvx >/dev/null 2>&1 || { curl -Ls https://astral.sh/uv/install.sh | sh; }
```

### Check for SHOWBOAT_REMOTE_URL

```bash
echo "${SHOWBOAT_REMOTE_URL:-NOT_SET}"
```

If it is **not set**, ask the user for the value and prefix every `uvx showboat` invocation with it:

```bash
SHOWBOAT_REMOTE_URL=<value> uvx showboat <command>
```

If it **is set**, use `uvx showboat <command>` normally.

### Learn the tools

Run both — their help output is the complete API reference:

```bash
uvx showboat --help
uvx rodney --help
```

## Step 1: Discovery

Before writing, understand what you're explaining:

1. **What is the topic?** Architecture of a service? A specific subsystem? A bug and its mechanism? A request lifecycle? A data flow?
2. **Who is the audience?** A new engineer? A reviewer? A non-engineer stakeholder? Adjust depth and vocabulary accordingly.
3. **What sources are available?**
   - Source code (read it; don't guess)
   - Configuration files, docker compose, infra-as-code
   - A running stack to inspect (`docker compose ps`, processes, ports)
   - A web UI to screenshot
   - Database schema, API definitions, message contracts
4. **What angles will help?** Plan the diagrams *before* writing prose. Sketch on scratch paper or in your head:
   - Component map — what pieces exist and how they connect
   - Sequence — what happens in time during a key operation
   - State — what lifecycle an entity goes through
   - Data — what shape information has
   - Flow — how a request, event, or value travels

If a web UI is involved and login is required, ask the user for credentials.

## Step 2: Plan the document

A typical structure (adapt as needed):

1. **Overview** — one paragraph + one high-level diagram. The reader should know the shape of the answer immediately.
2. **Components** — what each piece is and what it does. One diagram showing the component map.
3. **Interactions** — how the pieces talk to each other. Sequence diagrams for key operations.
4. **Flows / lifecycles** — how data or state moves through the system over time.
5. **Edge cases / gotchas** — things that aren't obvious from the diagrams alone.
6. **Summary** — restate the key insight in one or two sentences.

This is a guideline, not a template. If the topic is a single bug, the structure may be: symptom → mechanism diagram → root cause → fix sketch. Match the structure to what you're explaining.

## Step 3: Build the document

**IMPORTANT: NEVER edit the document directly.** No file editors, redirections, `sed`, `cat >`, or any other modification method. ALL changes go through `uvx showboat` (`init`, `note`, `exec`, `image`, `pop`). Showboat manages the format and syncs with the remote server — direct edits corrupt both.

### Initialize

```bash
PROJECT_NAME=$(basename "$(pwd -P)")  # or set manually if explaining something not tied to a project
uvx showboat init "$EXPLAIN_DIR/explain.md" "$PROJECT_NAME — topic being explained"
```

The title should name what is being explained, e.g. `myapp — how authentication works` or `payments-service — refund flow`.

### Assemble content

- `uvx showboat note <file> "text"` — prose: framing, transitions, key takeaways
- `uvx showboat exec <file> bash "command"` — show real evidence (file tree, schema dump, config excerpt, sample output)
- `uvx showboat image <file> '![alt text](path.png)'` — embed a diagram or screenshot
- `uvx showboat pop <file>` — remove last entry if wrong

Tips:
- `exec` prints output to stdout AND appends to doc — useful for grounding explanations in real artifacts (e.g. `cat schema.sql | head -40`).
- `image` copies files into the doc directory with UUID filenames — the original path doesn't need to persist.
- Filter noise (warnings, log timestamps) with `grep -v` patterns.

### Mermaid diagrams (the main attraction)

Diagrams are the core of an explanation. Produce *multiple* — one diagram is rarely enough. Match chart type to purpose:

- `flowchart` — component maps, decision trees, high-level data flow
- `sequenceDiagram` — interactions over time between actors/services
- `stateDiagram-v2` — entity lifecycles, state machines
- `classDiagram` — object/data structure relationships
- `erDiagram` — database schema, entity relationships
- `gantt` — timelines, phased rollouts
- `gitGraph` — branching strategies, release flows

Write each Mermaid source to a `.mmd` file under `$EXPLAIN_DIR/assets/`, render to `.png` with `mmdc`, then embed:

```bash
cat > "$EXPLAIN_DIR/assets/component-map.mmd" <<'EOF'
flowchart LR
  Client -->|HTTP| API
  API -->|SQL| DB[(Postgres)]
  API -->|GET/SET| Cache[(Redis)]
  API -->|publish| Queue[(RabbitMQ)]
  Worker -->|consume| Queue
  Worker -->|SQL| DB
EOF
npx -p @mermaid-js/mermaid-cli mmdc \
  -i "$EXPLAIN_DIR/assets/component-map.mmd" \
  -o "$EXPLAIN_DIR/assets/component-map.png"
uvx showboat image "$EXPLAIN_DIR/explain.md" '![Component map: API serves clients, talks to Postgres and Redis, dispatches work to Worker via RabbitMQ](assets/component-map.png)'
```

Alt text should *describe what the diagram shows*, not just label it. A reader using a screen reader, or skimming, should get the point from the alt text alone.

**Diagram quality:**
- Keep each diagram focused on one idea. Two simple diagrams beat one cluttered one.
- Label edges with what flows across them (`HTTP`, `SQL`, `publishes event`).
- Use consistent node shapes: cylinders for data stores, rectangles for services, rounded for external actors.
- For sequence diagrams, include only the actors relevant to the interaction being shown.

### Browser automation (when a UI is part of the explanation)

```bash
cd "$EXPLAIN_DIR"     # ensure .rodney/ lands under /tmp
rodney start
rodney open "http://localhost:PORT"
rodney waitload
rodney screenshot -w 1440 -h 900 "$EXPLAIN_DIR/assets/ui-overview.png"
rodney stop
```

Annotate UI screenshots with surrounding prose — explain *what the reader is looking at* and *why it matters*.

For login or interaction, see `rodney --help`. If `rodney input`/`click` time out, fall back to JS:

```bash
rodney js "(function(){ let setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set; setter.call(document.querySelector('#field'), 'value'); document.querySelector('#field').dispatchEvent(new Event('change', {bubbles: true})); return 'done'; })()"
```

### Grounding in real artifacts

Where useful, anchor explanations in actual code, schema, or config — don't paraphrase what the reader could be shown directly:

```bash
uvx showboat exec "$EXPLAIN_DIR/explain.md" bash "sed -n '40,70p' app/services/auth.rb"
uvx showboat exec "$EXPLAIN_DIR/explain.md" bash "docker compose exec db psql -U postgres -d app -c '\\d users'"
uvx showboat exec "$EXPLAIN_DIR/explain.md" bash "curl -s http://localhost:3000/api/health | jq"
```

A Mermaid component map is more powerful when followed by an `exec` showing the actual `docker compose ps` output that the diagram represents.

## Step 4: Finalize

End with a `note` summarizing the key takeaway in one or two sentences — what the reader should walk away knowing. If the explanation enumerated several mechanisms, a small recap table can help.

The document is not expected to pass `showboat verify` — it captures explanatory artifacts, not a reproducible sequence.

## Step 5: Cleanup is optional

`/tmp` is ephemeral by nature. The showboat document has already been transmitted to the remote viewer, so local files under `$EXPLAIN_DIR` are disposable. No gitignore step is needed since nothing was written to the working directory.

## Quality checklist

- The document contains **multiple diagrams** covering different angles (structure, sequence, state, data, flow as relevant).
- Every diagram has descriptive alt text that conveys its meaning.
- Prose between diagrams explains *why* something matters, not just *what* the diagram shows.
- Real artifacts (code excerpts, schema, config, command output) ground the explanation where appropriate.
- Each diagram is focused on one idea — no kitchen-sink diagrams.
- A reader skimming only headings, diagrams, and alt text gets the core message.
- Nothing was written to the working directory; everything is under `/tmp`.
