# Test Review Command

Review the test code in the current context or specified file(s) for adherence to testing best practices.

## What to Check

1. **Behavior vs Implementation** - Tests should verify what the code does, not how it does it
2. **Mocking Boundaries** - Only system boundaries (HTTP, filesystem, external APIs) should be mocked, not internal modules
3. **Test Data** - Using Effect Arbitrary/Schema for generation, not static fixtures
4. **Component Mocking** - React components should NOT be mocked; render real component trees
5. **Test Type Appropriateness** - Is this the right level of testing (unit/integration/E2E)?

## Review Process

1. Read the test file(s) to review
2. Check each test against the criteria above
3. Identify specific violations with line numbers
4. Suggest concrete improvements with code examples

## Output Format

Provide a structured review:

### Summary
Brief overview of test quality and main issues found.

### Issues Found
For each issue:
- **Location**: `file:line`
- **Problem**: What violates best practices
- **Suggestion**: How to fix it with example code

### Strengths
What the tests do well.

### Recommended Actions
Prioritized list of improvements.

## Reference

See [SKILL.md](../SKILL.md) and [TESTING_BEST_PRACTICES.md](../TESTING_BEST_PRACTICES.md) for full testing guidelines.
