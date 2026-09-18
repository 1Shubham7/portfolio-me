---
title: "A Spec-Driven Workflow for AI Coding Agents, and the Hook That Enforces It"
description: "A spec-driven workflow for Claude Code agents: a numbered spec that gets attacked before any code, checks set up before features, three roles with walls between them, a PreToolUse hook that enforces the rules, and injected bugs to grade the tests. Written after running it once and logging every confident mistake."
dateString: September 2026
draft: false
tags: ["Claude Code", "AI", "Agents", "Hooks", "Testing", "DevOps"]
weight: 1
cover:
    image: "/blog/spec-driven-multi-agent-workflow/cover.png"
---

I have been trying out spec-driven development with AI coding agents, and experimenting with different workflows around it. This post shows one workflow I designed and ran from start to finish with Claude Code. I think it came out very cool, mostly because of one idea: the agents are kept in line by what they are not allowed to see, and by a script they cannot argue with.

Spec-driven development, if you have not heard the term, means you write the requirements down first, as a numbered list where every item can be tested, and the agents build and test against that document. Not against a chat. A chat scrolls away; a file says the same thing every time anyone reads it.

The workflow puts three agents around that spec: one writes the code, one writes the tests without ever seeing the code, and one reviews the diff on a different model. The rules that keep them apart are enforced by a Claude Code hook, a small script that runs before every command or file access and can refuse it. Here is the whole workflow, how to set up each part on your own project, and what each part caught when I ran it.

## The whole workflow, on one screen

1. **Write a numbered spec.** Every requirement gets an ID and has to be testable.
2. **Have an agent attack the spec** before any code exists: check it against the real world and write down everything wrong with it.
3. **Set up the checks before the first feature.** A fast tier the agent runs after every edit, a slow tier for the end, and no agent can edit either.
4. **Split the work into three roles with walls between them**: dev writes the code, QA writes the tests from the spec and never sees the code, and a critic on a different model sees only the diff.
5. **Enforce the rules with a hook**, a script Claude Code runs before every tool call, which can block the call.
6. **Inject bugs to grade the tests.** Break the code on purpose. If no test fails, the tests were not testing that.
7. **Log every confident mistake**: what was wrong, what caught it, what changed.

The sections below follow that list in order, and the hook gets the most space.

## What I ran it on

I ran this once, on a small read-only command line tool, written in Rust, against a hosting platform's REST API. One of its commands compares a declared config file to live state and exits non-zero on drift, the way [`terraform plan -detailed-exitcode`](https://developer.hashicorp.com/terraform/cli/commands/plan) does, so CI can gate on it.

The tool is about 1,800 lines of Rust, not counting comments, built in one working session on 7 September 2026 with Claude Code. I did not type the Rust. I wrote the spec, defined the steps and what had to be true at the end of each, and handed the lead to the AI.

## Step 1: a numbered spec

`spec/SPEC.md` is a list of numbered requirements, R1 to R60. One of them, verbatim:

> **R17** Malformed or unexpected JSON in a response MUST produce a parse error naming the JSON path of the field that failed, not a panic.

Write each one like that: what must happen, what must not, and nothing a test could not check.

The numbers are what everything else hangs on. Dev puts a `// R17` comment at the code that satisfies it, QA names every test by requirement ID, and each of the critic's findings starts with one. When something fails, the argument is about a sentence in a file and not about what somebody meant.

## Step 2: attack the spec before any code

Give an agent one job: read the spec against whatever ground truth you have, write everything wrong with it into a file, and stop. Here the ground truth was the vendor's own Go client, Node CLI and Terraform provider, and the file was `docs/SPEC-REVIEW.md`. It came back with thirty findings, four of them blocking.

The best one was my mistake. The tool is meant to be read-only, and my safety requirement said it "MUST NOT issue any HTTP method other than GET". The vendor's API uses POST for every read, and one read shares its URL with every destructive action, differing by a word in the request body. On that API, read-only cannot be a property of the HTTP method. The requirement became a fixed list of the exact requests the tool may send (HTTP method, URL path, and the action word in the body), checked in the tool's HTTP code before anything goes out.

None of the four would have been caught by a test, because tests are written from the spec. The amended spec is the contract for everything after it, and it is one of the files the hook will not let an agent edit.

## Step 3: checks first, in two tiers, frozen

Write the checks before the first feature, and get them green on the empty project.

```makefile
verify-fast:
	cargo fmt --check
	cargo clippy --all-targets -- -D warnings
	cargo test

verify-full: verify-fast
	cargo audit
	cargo deny check
	cargo mutants --file src/diff.rs
```

`verify-fast` is the formatter, the linter (`-D warnings` turns lint warnings into errors) and the tests, and takes about ten seconds here. Make your fast tier that cheap, because it only works if an agent runs it after every edit. Dev's instructions say so in one line: "Done" means green, not "should be green". `verify-full` adds the slow checks: a vulnerability audit, the dependency policy in `deny.toml`, and a mutation run, which is step 6.

An agent with a failing check has two ways to make it pass, and one of them is editing the check. So the verify targets and `deny.toml` are frozen: the hook in step 5 refuses any agent edit to them.

## Step 4: three roles, and what each one is not given

| Role | Model | Is given | Is not given |
| :-- | :-- | :-- | :-- |
| dev | Claude Fable 5.1 | The spec, the spec review, the vendor's reference repos, all of `src/` | Permission to write the test suite |
| qa | Claude Fable 5.1 | The spec, `docs/API.md` (public signatures, no function bodies), `tests/`, `Cargo.toml` | `src/` except `src/lib.rs` (little more than the module list), the git history, the dev conversation |
| critic | Claude Opus 5 | The spec and the diff | The conversation, the source tree, the ability to run anything |

The last column is what I mean by a wall. Dev is the main Claude Code session, the one I talk to. It spawns QA and the critic as subagents: separate runs that start with none of my conversation and report back when they finish.

Each role is a Markdown file in `agents/`. The prompt that spawns an agent tells it first to read its role file and follow it exactly, then gives the paths of its inputs and outputs. Keep the spawn prompt a pointer and put everything durable in the file.

### QA never sees the code

QA's role file puts the reason for the wall before the rules:

> If the same context writes both the code and the tests, the tests encode the same misunderstanding as the code and pass anyway.

That happened in the build. The agent that wrote the network code decided that a garbled response, a `200 OK` whose body was not JSON, should be treated like a network failure and tried three times. It left a comment defending the choice: "a truncated body is the usual cause". Reasonable, and not what the spec says: a garbled response is an error, and it is not on the spec's list of things that may be retried. A test written by the same agent would have expected three requests and passed.

QA had never seen the code. It wrote the test from the spec, expected one request and an error, watched it fail, and reported the failure instead of adjusting the test. Two sentences in its role file made it do that: "Do not change the implementation to make a test pass; report it. Do not change a test to match the implementation unless the spec says the implementation is right." QA wrote 235 tests and left that one failing on purpose. The fix was about ten lines, and the test passed unchanged.

You have to build what crosses the wall. QA cannot write tests without knowing what functions exist, so it gets `docs/API.md`. The dev session wrote a throwaway Python script, about 80 lines, that emits every public item's signature and doc comment with the function bodies removed.

### The critic sees a diff and nothing else

The critic's role file gives the same reason, about review: "A model reviewing its own output tends to re-derive the same reasoning and approve it." It gets a copy of the spec and one file: the output of `git diff <first commit> <last commit> -- src Cargo.toml`, redirected to a path outside the project. It is a [Claude Code subagent](https://code.claude.com/docs/en/sub-agents), and its definition in `.claude/agents/critic.md` starts like this:

```yaml
---
name: critic
description: Reviews a diff against spec/SPEC.md on a different model from the one that wrote the code. Use for Phase 5 review. Give it the spec path and a diff file path; it reads nothing else.
model: opus
tools: Read, Write
---
```

The body under that frontmatter tells the critic to read `agents/critic.md` first and follow it exactly, so the rules stay in the role file.

That `model: opus` line did not exist at first. I asked where "different model" was defined, and the answer was a parameter on the call that spawned the critic, plus a sentence in the role file. Something that was true because an agent chose it became true because a file says so.

The critic found five real defects. One was the retry bug, flagged with no knowledge of QA's test. The most serious: reqwest, the HTTP library, follows up to ten redirects by default. A 307 or 308 redirect from an allowed URL would resend the same request, sign-in token included, to whatever host the redirect named, and the tool's check of what it may send only ever saw the original URL. QA's tests for the read-only requirement have the form "the fake server received zero requests", which cannot see a request that went to a different host. The fix was to disable redirects.

The critic was also wrong twice. It cannot run code or see past its diff, so treat its findings as leads to verify.

## Step 5: a hook, because a prompt is only a sign on the door

I had to have hooks explained to me, and this is the explanation that landed.

The model never executes anything. It produces text, and some of that text is a request: run this command, write this file, fetch this URL. Claude Code, the program around the model, reads the request and does the work. That includes the shell. Bash is a tool like any other, so `cargo test` is a tool call whose input happens to be a string.

A prompt constrains the requester. The model reads it, weighs it against everything else in context, and usually complies. A hook constrains the executor. A [`PreToolUse` hook](https://code.claude.com/docs/en/hooks) runs in the gap between the request and the execution, outside the model, and the model's opinion of the rule is not an input to it. A prompt is a sign on the door. A hook is a lock.

Since everything an agent does to the world is a tool call, that gap is the one place every action has to pass through.

### Registering it

A few lines in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Edit|Write|MultiEdit|NotebookEdit|WebFetch|Read|Grep|Glob",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/pre-tool-use.py\""
          }
        ]
      }
    ]
  }
}
```

The matcher is the list of tools the script gets to see, and the hook covers only that list. This one leaves out `WebSearch`, the tool that spawns subagents, and every MCP tool (the ones external servers add, named `mcp__<server>__<tool>`). So this hook's check of network hosts covers `WebFetch` and shell network tools and says nothing about web search. When you write yours, go through every tool your agents have and decide each one on purpose.

### The protocol: stdin, exit 0, exit 2

Claude Code pipes a JSON object describing the call to the script's stdin. It has about a dozen top-level fields, and this script reads three: `tool_name`, `tool_input` and, in a later section, `agent_type`. For a shell command the part that matters looks like this:

```json
{"tool_name": "Bash", "tool_input": {"command": "git push --force origin main"}}
```

For `Write` and `Edit` the input carries a `file_path`, and for `WebFetch` a `url`. The script's answer is its exit code. Exit 0 allows the call. Exit 2 blocks it, and whatever the script wrote to stderr goes back to the model as the reason:

```python
def block(reason: str) -> None:
    sys.stderr.write(f"BLOCKED by .claude/hooks/pre-tool-use.py: {reason}\n")
    sys.exit(2)
```

The rest is a dispatch on the tool name. This is `main()`, minus the `agent_type` read and an inert debug aid that were added on 18 September:

```python
def main() -> None:
    try:
        payload = json.load(sys.stdin)
    except json.JSONDecodeError:
        # Malformed input: fail closed. An unparseable call is not a known-safe call.
        block("could not parse hook input")
    tool = payload.get("tool_name", "")
    inp = payload.get("tool_input", {}) or {}

    if tool == "Bash":
        check_bash(inp.get("command", ""))
    elif tool in ("Edit", "Write", "MultiEdit", "NotebookEdit"):
        check_file_tool(tool, inp)
    elif tool in ("WebFetch",):
        check_host(inp.get("url", ""))
    elif tool in ("Read", "Grep", "Glob"):
        check_read(inp)
    sys.exit(0)
```

### What it blocks

Recursive force deletes, force pushes, network calls to hosts that are not on a list, and writes outside a short list of directories. The rules the workflow depends on are the frozen files from steps 2 and 3:

```python
FROZEN = {"deny.toml", "spec/SPEC.md"}

def check_frozen(path: Path, content_hint: str) -> None:
    rel = relative_to_project(path)
    if rel in FROZEN:
        block(f"'{rel}' is frozen; changing it means changing the contract, which is the user's call")
    if rel == "Makefile" and re.search(r"verify|cargo|clippy|fmt|test|audit|deny|mutants", content_hint):
        block("the verify targets in 'Makefile' are frozen; do not weaken a check to make it pass")
```

Write the stderr text for the model: the last message above tells it what not to try next.

### Fail closed, including when the hook itself breaks

Fail closed means a broken hook blocks the call; fail open means it lets the call through. In Claude Code, any exit code other than 0 or 2, from a script that prints nothing to stdout, is a non-blocking error: you see a hook error notice and the call proceeds. So a hook that crashes fails open. Wrap the entry point before you write the first rule. This script did not.

Look at `main()` again. The comment says "fail closed", and for input that is not JSON it did. But valid JSON that is not an object, `[1,2,3]` say, has no `.get`. Python raised `AttributeError` and exited 1, and the call went through. So would any bug in any checker. Since 17 September the entry point is wrapped:

```python
if __name__ == "__main__":
    try:
        main()
    except SystemExit:
        raise
    except BaseException as exc:  # noqa: BLE001 - deliberate catch-all
        sys.stderr.write(f"BLOCKED by .claude/hooks/pre-tool-use.py: hook crashed ({type(exc).__name__}); failing closed\n")
        sys.exit(2)
```

`SystemExit` has to pass through, because that is how `block()` and the normal path leave; everything else becomes a block. A syntax error in the script, a missing `python3` or a timeout still fails open, and closing the first two needs a wrapper outside the script.

### Who is asking

Walls need the hook to know which agent is making the call, and Claude Code tells it. A call from inside a subagent carries two fields a main-session call does not, `agent_id` and `agent_type`, and for the critic the type was `"critic"`. Both are in the [hooks reference](https://code.claude.com/docs/en/hooks).

An earlier version of this post said the opposite here. The dev session had told me the hook cannot tell agents apart, when I asked why QA had no hook of its own. The agent reviewing the post doubted it, and on 18 September the hook was made to record the field names it received, which settled it.

So define each role that has a wall as a named subagent and key its rules on `agent_type`. Condensed from the hook, with the block messages shortened (`.qa-mode` is a fallback, explained after the code):

```python
QA_MARKER = PROJECT / ".qa-mode"
GIT_HISTORY = re.compile(r"\bgit\s+(log|diff|show|blame|grep)\b")
AGENT_TYPE = ""  # main() sets this from payload.get("agent_type")

def qa_mode() -> bool:
    return AGENT_TYPE == "qa" or QA_MARKER.exists()

# inside check_bash(command):
    if qa_mode():
        for m in re.finditer(r"src/[A-Za-z0-9_/]+\.rs", command):
            if m.group(0) != "src/lib.rs":
                block(f"QA mode: reading '{m.group(0)}' is forbidden")
        if GIT_HISTORY.search(command):
            block("QA mode: git log/diff/show would reveal the implementation")
```

`Read`, `Grep` and `Glob` get the same treatment by path. The same field closed a wall nothing had enforced: the critic's instruction to read only the spec and the diff. When `agent_type` is `critic`, reads inside the project are refused except `spec/` and the critic's own two files. The diff lives outside the project and stays readable.

`.qa-mode` is a marker file, an empty file whose existence switches the QA rules on. It is the fallback for a QA that is not a named subagent. During the build QA ran as a general-purpose subagent, so its `agent_type` said nothing about its role. The dev session ran `touch .qa-mode` immediately before spawning QA and deleted the file when QA reported back. The hook looks for the file on every call, so the switch takes effect on the next one. Put the marker in `.gitignore`. Its cost is that it applies to everyone: while it was on, dev was locked out of its own source too.

The `GIT_HISTORY` line was not in the first version either. I asked to be shown QA isolation working, and the demonstration exposed a hole: `git log -p` prints source without naming a path. Until then that case was covered only by a sentence in QA's prompt.

### It is not a sandbox, so give it a self-test

The script's own docstring says "this is not a sandbox": `python3 -c "open(...).write(...)"` would get past the write check. What it buys is that the forbidden thing takes a deliberate act.

Regexes over shell commands go wrong in both directions, so the hook has a self-test, `.claude/hooks/selftest.py`: 58 cases, each starting with the exit code it expects. `P` is the project root, and `ALLOW` and `BLOCK` are 0 and 2.

```python
(BLOCK, "Bash", {"command": "git push --force origin main"}),
(ALLOW, "Bash", {"command": "git push origin main"}),
(BLOCK, "Bash", {"command": "sed -i 's/cargo test/true/' Makefile"}),
(ALLOW, "Bash", {"command": "echo https://evil.example.com"}),
(BLOCK, "Edit", {"file_path": P + "/spec/SPEC.md", "old_string": "a", "new_string": "b"}),
```

Keep the cases in a file: the first version was an inline shell heredoc, and the hook blocked it for containing the literals it was testing.

### Which walls are still signs

QA's wall is held by the hook, and since 18 September so is the critic's reading list. The critic cannot run anything because its tool list is `Read, Write`. Dev's ban on writing tests is still a line in `agents/dev.md` that nothing enforces. The agent reviewing an earlier version of this post found the unenforced walls and the crash that let calls through. I had found neither.

## Step 6: break the code on purpose

Coverage tells you a test ran a line, not whether the test would notice if the line were wrong. Mutation testing checks the second thing, and [`cargo-mutants`](https://mutants.rs/) does it for Rust: it makes one small change to the code (a mutant), runs the suite, and records whether any test failed. A mutant that no test notices has survived (cargo-mutants' own word is "missed"). An unviable one did not compile.

| Target | Mutants | Caught | Unviable | Timed out | Survived |
| :-- | :-- | :-- | :-- | :-- | :-- |
| `src/diff.rs` | 14 | 11 | 3 | 0 | 0 |
| `src/report.rs` and `src/drift_config.rs` | 18 | 15 | 3 | 0 | 0 |
| `src/client.rs`, before | 61 | 48 | 6 | 2 | 5 |
| `src/client.rs`, after | 60 | 51 | 6 | 3 | 0 |

Two of the five survivors were in the retry delay. The tool doubles its wait between retries, computed from `attempt - 1`. The mutants changed that to `attempt / 1` and `attempt + 1`, and no test noticed. The test timed a call that retries twice from a 100 ms base and accepted any total from 300 ms up to 2 s. The correct total is 300 ms. The mutants take 600 ms and 1.2 s, both inside the bound.

The arithmetic was extracted into a pure function, `backoff_delay(base, attempt)`, so a test can assert the exact value. QA, still isolated and told what the spec requires but not what the mutants were, added 8 tests, and the rerun had no survivors.

Run one mutation job at a time. Two concurrent runs starved each other for CPU and were stopped.

## Step 7: log every confident mistake

Every time a model was confidently wrong it went into `docs/FAILURE-LOG.md` under three labels: what was wrong, what caught it, what changed. There are twenty-three entries. By what caught them:

| Caught by | Entries |
| :-- | :-- |
| Dev checking its own state or running the binary | 7 |
| The critic | 5 |
| An agent reviewing an earlier version of this post | 3 |
| The hook | 2 |
| The first live API call | 2 |
| The spec attack (step 2) | 1 |
| QA, from the spec | 1 |
| Mutation testing | 1 |
| The compiler | 1 |
| Dev, knowing the roles from step 4 (the critic flagged "no tests in the diff"; QA was writing them) | 1 |

That sums to twenty-four because the retry bug was caught twice.

The largest row is the least glamorous: the dev session looking at what it had done, reading `git show --stat` after a commit or noticing that the binary under test was a stale placeholder. The walls bought the catches that looking could not reach: a deliberate, commented, wrong decision, and a request that leaves for a host no test server is watching.

The live API row came from outside the workflow. The tests that needed an API ran against a fake server modelled on the vendor's own client libraries, which handle a failed sign-in as a `200` response whose body says it failed. The first request to the real API, on 8 September, got HTTP 401. The spec, the fake server and the tests all assumed the `200`, and 243 tests were passing. After the fixes from that run, QA added 5 tests and the suite ended at 248.

## What this does not show

This was run once, with one configuration of models, and makes no claim about cost or about how it would go on another project.

Isolation is a hook plus a fresh context, and it leaks: a `cargo test` compile error can echo lines of library source into QA's terminal. Mutation testing covered four of the nine library modules. The one live call used placeholder credentials, so all I have seen of the real API is how it rejects a bad login.

The part I would copy first is the hook, and the habit that goes with it: for every rule in every role file, ask what enforces it. I asked three times. Twice the answer came down to a sentence in a prompt, and once it was a claim about what the hook receives that nobody had measured.
