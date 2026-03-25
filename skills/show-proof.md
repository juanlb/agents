# Show Proof

Create a behavioral verification document that proves completed work through live evidence. The document follows a phase-based structure: **before state, action, after state**.

## Mindset

Git diff shows what changed. Proof shows what works. A human reviewer should read the document and be convinced the feature/fix works without running anything themselves.

## Step 0: Ensure tooling is available

### Check for uvx

Before using any `uvx` commands, verify it is installed:

```bash
command -v uvx >/dev/null 2>&1 || { curl -Ls https://astral.sh/uv/install.sh | sh; }
```

### Check for SHOWBOAT_REMOTE_URL

Before running any `showboat` command, check if the `SHOWBOAT_REMOTE_URL` environment variable is set:

```bash
echo "${SHOWBOAT_REMOTE_URL:-NOT_SET}"
```

If it is **not set**, ask the user for the value. Once obtained, prefix every `uvx showboat` invocation with the variable so documents are sent to the external server:

```bash
SHOWBOAT_REMOTE_URL=<value> uvx showboat <command>
```

If it **is set**, use `uvx showboat <command>` normally — the variable is already in the environment.

### Learn the tools

Run both of these first — their help output is the complete API reference, written for agents:

```bash
uvx showboat --help
uvx rodney --help
```

## Step 1: Discovery

Before building the proof, determine what's available:

1. **What was built/fixed?** Read recent changes and understand the work done. If in a git repo, use git log/diff; otherwise, review the relevant files directly.
2. **What proves it works?** Think about observable effects:
   - Database values changing?
   - Cache keys appearing or being invalidated?
   - Files created, modified, or deleted?
   - API responses changing?
   - UI showing new behavior?
   - CLI commands producing different output?
   - Log entries appearing?
   - Background jobs completing?
3. **What tools are available?**
   - `docker compose ps` — is there a running stack? What services?
   - Is there a web app? On what port?
   - Is there a database to query? Redis? Other services?
   - Are there CLI tools or rake tasks to check state?

If there's a web app, the user may need to provide login credentials — ask if needed.

## Step 2: Plan the phases

Structure every proof around these phases:

### Phase 1: Before State
Capture the current observable state BEFORE exercising the feature. This is the baseline.

Examples: DB query showing current values, Redis key existence check, `ls` showing a file doesn't exist yet, API response with old behavior, browser screenshot of current UI, CLI output.

### Phase 2: Action
Show the change happening — the feature being exercised.

Examples: Browser form submission, CLI command triggering the feature, API request, background job trigger.

### Phase 3: After State
Verify observable effects match expectations. This IS the proof.

Examples: Same DB query now shows new values, Redis key created/invalidated, file now exists with expected content, browser screenshot shows updated UI, different process reads correct state.

### Optional: Cross-cutting Verification
When relevant, verify from MULTIPLE vantage points (e.g., DB + cache + UI all agree).

## Step 3: Build the document

### Initialize

```bash
PROOF_DIR="show-proof/YYYYMMDD-short-description"
mkdir -p "$PROOF_DIR" show-proof/assets
PROJECT_NAME=$(basename "$PWD")
uvx showboat init "$PROOF_DIR/proof.md" "$PROJECT_NAME — task description"
```

Use today's date and a kebab-case summary for the directory name. The title must be `project-name — task description` (no "Proof of Work" — it's implicit on the viewer). Derive the project name from the current directory name.

### Record context

Immediately after init, record the working context as the first entry. If inside a git repo, record the current branch:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  uvx showboat exec "$PROOF_DIR/proof.md" bash "git branch --show-current"
else
  uvx showboat note "$PROOF_DIR/proof.md" "Working directory: $PWD"
fi
```

### Assemble content

Build the document sequentially using showboat:

- `uvx showboat note <file> "text"` — commentary explaining what's happening and why
- `uvx showboat exec <file> bash "command"` — run command, capture output
- `uvx showboat image <file> '![alt text](path.png)'` — embed screenshot
- `uvx showboat pop <file>` — remove last entry if wrong

Tips:
- `exec` prints output to stdout AND appends to doc — use it to verify commands worked
- `image` copies files into doc directory with UUID filenames
- Filter noisy output (warnings, SQL logs) with `grep -v` patterns in the command

### Browser automation (when a web UI exists)

```bash
rodney start                          # launch headless Chrome
rodney open "http://localhost:PORT"   # navigate
rodney waitload                       # wait for page load
rodney screenshot -w 1440 -h 400 show-proof/assets/name.png
rodney stop                           # clean up when done
```

The `show-proof/assets/` directory is created during init. Store all raw screenshots there.

**Login**: inspect form with `rodney html 'form'`, then use `rodney input` and `rodney click`. If those time out (common with number inputs or offscreen elements), fall back to JS:

```bash
rodney js "(function(){ let setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set; setter.call(document.querySelector('#field'), 'value'); document.querySelector('#field').dispatchEvent(new Event('change', {bubbles: true})); return 'done'; })()"
```

To submit: `rodney js "(function(){ document.querySelector('form').submit(); return 'ok'; })()"`
To scroll: `rodney js "(function(){ document.querySelector('#el').scrollIntoView({block:'center'}); return 'ok'; })()"`

### Docker compose patterns (when available)

```bash
docker compose exec app bundle exec rails runner "puts Model.first.value"
docker compose exec db psql -U postgres -d dbname -c "SELECT col FROM table;"
docker compose exec redis redis-cli GET 'key'
```

Filter verbose output: `... 2>&1 | grep -v 'warning\|Initializing'`

## Step 4: Do NOT restore state

This is not production. Leave the system in its post-action state — that IS the proof. Do not revert.

## Step 5: Finalize

End with a summary note — a table or bullet list mapping each phase to what was verified and the result.

The document is NOT expected to pass `showboat verify` — it captures a stateful before/after sequence. The value is the captured evidence, not reproducibility.

## Gitignore

If the project is a git repo, ensure `show-proof/` and `.rodney/` are in `.gitignore`. Proof documents are transmitted to the external viewer — local files don't need to be tracked. Rodney may create a `.rodney/` directory for its working state.

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  grep -qxF 'show-proof/' .gitignore 2>/dev/null || echo 'show-proof/' >> .gitignore
  grep -qxF '.rodney/' .gitignore 2>/dev/null || echo '.rodney/' >> .gitignore
fi
```

## Quality checklist

- Each phase has at least one piece of concrete evidence (command output, screenshot, or both)
- Commentary explains WHY each check matters, not just WHAT it shows
- Screenshots are wide enough to read (use `-w 1440`)
- Noisy output is filtered — only signal, no noise
- Summary ties all evidence together
