# Rollback Windows

!!! important

    Available for blue-green and canary rollouts since v1.4

By default, when an older Rollout manifest is re-applied, the controller treats it the same as a spec change, and will execute the full list of steps, and perform analysis too. There are two exceptions to this rule:

1. the controller detects if it is moving back to a blue-green ReplicaSet which exists and is still scaled up (within its `scaleDownDelay`)
2. the controller detects it is moving back to the canary's "stable" ReplicaSet, and the upgrade had not yet completed.

It is often undesirable to re-run analysis and steps for a rollout, when the desired behavior is to rollback as soon as possible. To help with this, a rollback window feature allows users to indicate that the promotion to the ReplicaSet within the window will skip all steps.

Example:

```yaml
spec:
  rollbackWindow:
    revisions: 3

  revisionHistoryLimit: 5
```

Assume a linear revision history: `1`, `2`, `3`, `4`, `5 (current)`. A rollback from revision 5 back to 4 or 3 will fall within the window, so it will be fast tracked.

!!! important

  Bad ReplicaSet rollback protection turned on for blue-green and canary rollouts since v1.9

Bad ReplicaSet rollback protect ensures that a replicaset that has an historical abort event associated with it will not be allowed to undergo a fast rollback and skip all steps.

Example using the same `rollbackWindow` of `3` above:

1. Assume a linear revision history with these revisions:
  `1(good) - 2(good) - 3(good)(stable)`
2. A deployment starts then gets aborted so now we have:
  `2(good) - 3(good)(stable) - 4(aborted)(canary)`
3. A change to fix the aborted version is made:
  `3(good) - 4(aborted)(bad) - 5(good)(stable)`
4. This sets up a condition where a fast rollback from version `6` to version `4` would be permitted pre 1.9

!!! note

    The window is not dynamic based on aborted/good replicasets. This means that in the above example,
    we would _not_ extend the `rollbackWindow` to include revision `2`. ReplicaSet instances that are determined
    to be invalid are simply filtered from the already calculated window.
