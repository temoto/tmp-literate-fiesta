# What

One time research repo.

Problem is well known: github actions with both `on: push:` and `on: pull_request:` conditions, trigger duplicate runs.
- https://github.community/t/duplicate-checks-on-push-and-pull-request-simultaneous-event/18012/5
- https://github.com/orgs/community/discussions/26940

Solution options:
1. job-level condition like `if: github.event_name == 'push' || github.event.pull_request.head.repo.full_name != 'org/repo'`
  Pros: works correctly, duplicates are skipped.
  Cons: repeated condition in each job, repeated org/repo reference.
  Likely can be replaced with more recent repo.fork variable:
  `if: github.event_name == 'push' || github.event.pull_request.head.repo.fork`
2. workflow-level `concurrency` configuration:
  ```
  concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
  ```
  TODO verify.
  Pros: once per workflow, clean, makes sense.
  Expected cons: duplicates are cancelled (red marker, looks alarming).
3. depend on external action skip-duplicate-actions:
  ```
  jobs:
    test:
      runs-on: ubuntu-latest
      steps:
        - name: Skip Duplicate Actions
          uses: fkirc/skip-duplicate-actions@master
          with:
            cancel_others: true
  ```
  TODO verify.
  Pros: clean, probably skips (not cancels).
  Cons: external dependance, even if cancels_others for whole workflow it is configured in some primary job or in all.

# Results

1. Condition works and is fastest way. New syntax `repo.fork` avoids repeating repo name.
2. Concurrency is cleanest, but for quick actions often does not trigger. When it does trigger, shows red cross marker. Sometimes triggers runner-level cancel, something about transferrable jobs.
3. Could not make skip-duplicate-actions to work at all. Same name from another source: step-security/skip-duplicate-actions returns "Error: Subscription is not valid." when called within forked repo, before PR. Sometimes triggers runner-level cancel, something about transferrable jobs.

## Conclusion

IMHO, this is the winner:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' || github.event.pull_request.head.repo.fork
    steps:
      ...
```
