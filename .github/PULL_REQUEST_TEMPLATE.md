## Summary / Why

<!-- One or two sentences: what changes and why. Keep PRs small and focused. -->

## Spec linked

<!-- Required for feat/fix with behavior change. Link one file + scenarios. -->
<!-- For docs/chore with no behavior change, write: N/A — docs/chore, no behavior change -->

- Spec: `plans/domains/.../NN-*.md`
- Scenarios covered:
  -

## Type

<!-- Delete as applicable -->

- [ ] feat
- [ ] fix
- [ ] docs
- [ ] chore

## Checklist (do not delete unchecked items — pr-guard blocks merge until all are checked)

- [ ] Spec linked (`plans/domains/.../NN-*.md`), scenarios listed
- [ ] Types: `mypy bottom_up` clean; OpenAPI updated if API changed
- [ ] TDD: failing-first tests added; `just pytest` green
- [ ] Verify: every scenario maps to a test; migrations committed (`makemigrations --check`)
- [ ] Secure: `pre-commit run --all-files` clean; no secrets committed
- [ ] Small PR, 1 review, squash-merge

## Gates evidence (paste commands + results)

<!-- Required. Paste minimal proof so reviewers and agents can trust the gates. -->

```text
uv run mypy bottom_up
# paste result

just pytest
# paste result

just manage makemigrations --check
# paste result

uv run pre-commit run --all-files
# paste result
```

## Test steps / Screenshots

<!-- How to verify manually. Add screenshots for UI changes. -->

1.
2.
