# Code generation workflow

A code generation agent reads a specification (issue, ticket, conversation), writes code across one or more files, tests it, and submits the result. Unlike simple "write me a function" prompts, a production code generation workflow must understand the existing codebase, maintain consistency with project conventions, and verify that changes don't break anything.

## Pipeline structure

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Understand  │     │  Plan the    │     │  Generate    │
│  the task    │────▶│  changes     │────▶│  code        │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
┌──────────────┐     ┌──────────────┐     ┌──────▼───────┐
│  Submit or   │     │  Fix errors  │◀────│  Test &      │
│  report      │◀────│  (if any)    │     │  validate    │
└──────────────┘     └──────────────┘     └──────────────┘
                           ▲                      │
                           └──────────────────────┘
                              Tests still failing?
```

The edit-test-fix loop is the core of this workflow. A good code generation agent doesn't just write code — it runs it, reads the error, and fixes it.

## Tools involved

| Step | Tools | Why |
|------|-------|-----|
| Codebase understanding | Tree-sitter, ctags, grep | Parse and navigate existing code |
| Planning | LLM reasoning | Decide which files to modify and how |
| Code generation | LLM (direct) | Write the actual code |
| Sandboxed execution | E2B, Modal | Run tests safely |
| Testing | pytest, jest, cargo test | Verify correctness |
| Submission | GitHub API, git | Create PR or commit |

## Real projects implementing this workflow

**[Aider](https://github.com/paul-gauthier/aider)** — Edits code across multiple files in a git repo via natural language. Tags: `execution` `python`

**[SWE-agent](https://github.com/princeton-nlp/SWE-agent)** — Resolves GitHub issues autonomously by reading, planning, and patching code. Tags: `execution` `planning` `python`

**[Continue](https://github.com/continuedev/continue)** — Adds AI code assistance directly inside VS Code and JetBrains IDEs. Tags: `execution` `typescript`

**[Cline](https://github.com/cline/cline)** — An autonomous coding agent that works directly in VS Code to write, execute, and test code. Tags: `execution` `typescript`

**[Devika](https://github.com/stitionai/devika)** — An open-source AI software engineer that can understand high-level instructions and write code to build features. Tags: `execution` `planning` `python`

**[OpenHands](https://github.com/All-Hands-AI/OpenHands)** — Plans and executes multi-step software engineering tasks in a sandboxed environment (formerly OpenDevin). Tags: `planning` `execution` `python`

**[GPT Engineer](https://github.com/gpt-engineer-org/gpt-engineer)** — Generates entire codebases from a single specification prompt. Tags: `execution` `planning` `python`

## Pseudocode

```python
def code_generation_agent(task: str, repo_path: str, max_retries: int = 3):
    # Step 1: Understand the codebase
    file_tree = list_files(repo_path)
    relevant_files = llm.identify_relevant_files(task, file_tree)
    context = read_files(relevant_files)

    # Step 2: Plan the changes
    plan = llm.plan_changes(task, context)
    # plan = [{"file": "src/api.py", "action": "modify", "description": "..."}]

    # Step 3: Generate code
    changes = {}
    for step in plan:
        file_content = read_file(step["file"])
        new_content = llm.edit_code(step, file_content, context)
        changes[step["file"]] = new_content

    # Step 4: Apply and test
    for attempt in range(max_retries):
        apply_changes(changes)
        test_result = run_tests(repo_path)

        if test_result.passed:
            break

        # Step 5: Fix errors
        error_context = test_result.stderr
        fixes = llm.fix_errors(changes, error_context)
        changes.update(fixes)

    # Step 6: Submit
    if test_result.passed:
        create_pull_request(changes, task)
    else:
        report_failure(task, test_result)
```

## Failure modes

| Failure | Cause | Fix |
|---------|-------|-----|
| Wrong files edited | Agent doesn't understand project structure | Provide file tree and dependency graph as context |
| Style inconsistency | Agent uses different conventions | Include style examples from existing code in context |
| Test passes but logic wrong | Tests don't cover the change | Require the agent to write tests for new code |
| Infinite fix loop | Error is beyond the agent's ability | Set max_retries and escalate to human |
| Merge conflicts | Other changes landed while agent was working | Rebase before submitting, re-test after rebase |

## Production considerations

- **Isolation**: Always run agent-generated code in a sandbox (E2B, Docker). Never execute untrusted code directly on a production machine.
- **Cost**: A complex code change might need 5-15 LLM calls for planning + generation + 3 fix attempts. Budget $1-$10 per task.
- **Review**: Even if tests pass, require human review before merging. Agents can write correct-but-wrong code (technically works, logically incorrect).
- **Codebase size**: Most LLMs can't process an entire repo. Use retrieval (grep, embeddings, tree-sitter) to select relevant context.
