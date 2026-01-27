# Standard Planning Workflow

This workflow is used by all planning prompts unless otherwise specified.

## Workflow Steps

### Step 1: Parse and Load

1. Validate required arguments (error if missing)
2. Read input files (research documents, scaffold code, etc.)
3. Ask clarifying questions if requirements are ambiguous

### Step 2: Analyze Requirements

4. Extract relevant information from input files
5. Identify key patterns and characteristics
6. Map to dlt built-in features and patterns
7. Determine if standard approaches apply or custom logic needed

### Step 3: Design Tests (TDD)

8. Create test cases covering:
   - Happy path scenarios
   - Edge cases
   - Error conditions
9. Design mock responses and fixtures
10. Define expected outcomes

### Step 4: Create Specification

11. Design implementation approach
12. Document configuration requirements
13. Write detailed specification using appropriate template
14. Create task breakdown (if generating main spec)

### Step 5: Validate and Output

15. Present summary to user:
    - Key design decisions
    - Complexity rationale (for appendix prompts)
    - Test coverage
16. Confirm approach with user
17. Generate specification document
18. Save to appropriate output file

## Notes

- Each specialized prompt may add specific analysis steps in Step 2
- Test design (Step 3) should always follow TDD principles
- Output format varies: main spec vs. appendix (see individual prompt templates)
