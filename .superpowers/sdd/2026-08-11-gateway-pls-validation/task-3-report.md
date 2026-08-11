# Task 3 Report

## Summary
- Updated `README.md` to add the new option-08 PLS path.
- Extended the architecture diagram with the internal-LB → PLS → PE → consumer VNet/ACI flow.
- Added learning objective 8 for Service annotation passthrough and PLS/PE validation; renumbered cleanup to 9.
- Added module 08 to the module table, timing table, cost table, and references.
- Updated cleanup wording to module 09.

## Validation
- `grep -nE '08.*Private Link|09.*정리|07.*08.*옵션' README.md`
- `test -f docs/08-private-link-service.md && test -f docs/09-cleanup.md`
- `git diff --check`
- `curl -L` checks for both official references returned `200`

## Self-review
- Verified the commit message includes the required `Co-authored-by` trailer.
- Confirmed `git status` is clean after commit.
- Reviewed the final diff for numbering consistency and link updates.

## Commit
- `c1bf8977143f081c34834dafedf24c1a4ce06285`

## Concerns
- None blocking; Task 4 will replace the provisional 08 timing with measured numbers.

## Fix round 1

### Changed lines
- `README.md:90` — module 08 timing now says `20–30분 (임시, Task 4 라이브 리허설 후 실측값으로 교체 예정)`.
- `README.md:92` — total timing now marks the 08 option estimate as provisional until Task 4.
- `README.md:107-108` — added the short-lived qualifier to both option-08 PE and ACI cost rows.

### Validation commands
- `grep -nE '08.*Private Link|09.*정리|07.*08.*옵션' README.md`
- `test -f docs/08-private-link-service.md && test -f docs/09-cleanup.md`
- `git diff --check`

### Outputs
- `grep -nE ...` returned the expected numbered references, including the updated module rows at 90, 92, and 107-108.
- `test -f ... && test -f ...` exited 0 with no output.
- `git diff --check` exited 0 with no output.
