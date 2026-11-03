# Monitor a scheduled job with Watchgoose

Runs a command on a schedule and reports each run to a [Watchgoose](https://watchgoose.com) check. Watchgoose alerts you when a run fails, and also when a run never starts, for example because the Dagu host is down.

Each run sends two pings, `/start` and then either a success ping or `/fail`:

- `/start` before the command runs, so Watchgoose can measure run time.
- A success ping when the run succeeds.
- `/fail` when the run fails. The body names the run ID, status and log file path.

## Prerequisites

- Dagu 2.13.0 or newer.
- A Watchgoose check with a schedule and grace time that match the workflow's `schedule`.

## Inputs and secrets

Set `WATCHGOOSE_CHECK` in the Dagu process environment to the part of the check's ping URL after `https://watchgoose.com/ping/`:

- `<uuid>` for a UUID ping URL.
- `<ping-key>/<slug>` for a slug ping URL.

The workflow reads it as a Dagu secret, so it is masked in logs. Set the default of the `command` param to the job to run (scheduled runs use the default), and change `schedule` to match the check.

## What it changes

Runs `command` and sends up to two HTTPS requests per run to watchgoose.com. Nothing else.

`retry_policy.limit: 0` turns off DAG-level automatic retries for this workflow, so a failed run is reported at once. If you remove it and your `base.yaml` enables DAG retries, Dagu skips the failure handler until the last retry, so the fail ping comes later, and runs started with `dagu start` outside the scheduler never send it.

If Watchgoose cannot be reached, the start ping is ignored and the command still runs. A failed success or fail ping marks the run as failed in Dagu.

## Verify

Run once with a command that succeeds, then with one that fails:

```sh
dagu start workflow.yaml -- command="echo ok"
dagu start workflow.yaml -- command="false"
```

The check's event log in Watchgoose shows a start and a success ping, then a start and a failure ping.

## Recovery

Fix the command and rerun the workflow. The next success ping marks the check as up again.
