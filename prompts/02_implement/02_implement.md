# Implementation Phase Prompt

---
description: Implement a specification using TDD
argument-hint: <spec_file_path>
---

## Purpose

Implement the specification following strict TDD: tests first, then implementation. Follow the task list step by step.

## Arguments

**Required:**
- `spec_file_path`: Path to spec file (e.g., `specs/github_pagination_spec.md`)

## Critical Rules

1. **Load spec** → Extract task list and test cases
2. **TDD only**: Write test → verify it fails → implement → verify it passes
3. **Sequential**: Follow task list order, one task at a time
4. **ASK USER**: When spec is unclear, multiple approaches exist, or information is missing - STOP and ASK
5. **No assumptions**: Never guess credentials, config values, or requirements

## Workflow

For each task in spec:

1. Write test (red)
2. Run test - should fail
3. Implement minimal code (green)
4. Run test - should pass
5. Report progress
6. Move to next task

## When to Ask User

MUST ask when:
- Spec is ambiguous
- Missing information or credentials
- Tests fail unexpectedly
- Before deviating from task list

## Output After Each Task

```
[X] {task}
Changes: {files modified}
Tests: {PASS/FAIL}
Next: {next task}
```

## Validation

Before completing:
- [ ] All spec tests implemented and passing
- [ ] All tasks checked off
- [ ] Used dlt built-ins (no custom API clients)
- [ ] Asked user when needed

## Next Steps

After implementation is complete and all tests pass:

1. **Run full test suite**: `pytest tests/ -v`
2. **Test pipeline end-to-end**: Run a complete pipeline execution to verify data loads correctly
3. **Verify documentation**: Ensure README and usage examples are complete
4. **Check credentials setup**: Confirm `.dlt/secrets.toml.example` exists and `.dlt/secrets.toml` is in `.gitignore`

**Implementation complete!** Your dlt pipeline is ready to use.
