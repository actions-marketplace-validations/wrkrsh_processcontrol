# ProcessControl

ProcessControl is AI QA for pull requests. For each pull request it tests, an
agent builds and opens the app at that commit on a machine of its own, tests
the claims it finds in the PR description, and checks every screen it looks at
against the team's UI standards. The verdict lands on the commit as a GitHub check that
links to the run's evidence, such as screenshots and a screen recording. No
test code is written or maintained. Linux is the self-serve platform: a claim
only another platform can show is reported as blocked.

This repository holds the GitHub Action, the setup steps for coding agents and
the issue tracker. ProcessControl runs at
[processcontrol.dev](https://processcontrol.dev); its source is not public.

## GitHub App

The [GitHub App](https://github.com/apps/processcontrol-app) is how
ProcessControl tests pull requests. Sign in at
[processcontrol.dev](https://processcontrol.dev), open **Setup** and install
the app on the repositories to test. Under **Repositories**, each one has a
trigger:

- **Mentioned PRs**, the default: a collaborator with write access starts a new
  comment on a pull request with `@processcontrol-app`. Once such a comment
  starts a run, the pull request's later pushes and edits are tested too.
- **Every PR**: opening a ready pull request, reopening it, marking it ready,
  pushing to it or editing its title or description requests a test of its
  latest commit. A pull request that was already open is first tested at the
  next of these, or at a mention.

[Getting started](https://processcontrol.dev/docs/onboarding) covers the rest.

A coding agent can do the setup from the repository's checkout. Tell it:

> Set up ProcessControl using processcontrol.dev/agents.md

[agents.md](./agents.md) is that file: you sign in with GitHub, install the
app and approve the agent's access, and the agent does the rest.

## GitHub Action

The Action is for CI where the GitHub App cannot be installed. It submits a
[test plan](https://processcontrol.dev/docs/plans) over the HTTP API, waits for
the verdict and fails the job unless the run passes.

A plan is a YAML file in your repository with goals in plain language. Without
the GitHub App, the machine does not get your checkout or read the pull
request: a plan tests what the machine already has, the build an `installer`
URL points at, or what its `setup` commands fetch.

```yaml
# .processcontrol/testplan.yaml
version: 1
name: Release smoke test
target:
  os: linux
installer:
  source: url
  ref: https://example.com/releases/app_1.2.0_amd64.deb
  installHint: sudo apt install /tmp/pc/app_1.2.0_amd64.deb, then launch App from the menu.
tests:
  - id: opens
    goal: Install the app and launch it from the applications menu.
    expect:
      - The app's main window appears.
```

```yaml
# .github/workflows/processcontrol.yml
on: push

jobs:
  processcontrol:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: wrkrsh/processcontrol@v1
        with:
          api-key: ${{ secrets.PROCESSCONTROL_API_KEY }}
```

On `pull_request`, GitHub checks out the pull request merged into its base
branch, while the run is for the pull request's head commit. Give
`actions/checkout` that commit, `ref: ${{ github.event.pull_request.head.sha }}`,
so the plan submitted is the one in it; otherwise the Action warns.

Create the key under **Settings → API keys** at processcontrol.dev and store it
as a secret. On the Free plan a run's repository must be connected through the
GitHub App, so the Action without the app needs a
[paid plan](https://processcontrol.dev/pricing).

| Input             | Default                         | Meaning                                                                                            |
| ----------------- | ------------------------------- | -------------------------------------------------------------------------------------------------- |
| `api-key`         | required                        | The API key.                                                                                       |
| `plan-path`       | `.processcontrol/testplan.yaml` | The plan to submit, relative to the workspace.                                                     |
| `timeout-minutes` | `135`                           | How long to wait for the verdict and, once a `humanReview` plan's tests pass, for people's review. |
| `api-url`         | `https://processcontrol.dev`    | The API's base URL.                                                                                |

The outputs are `run-id` and `run-url`, the run's page in the dashboard with
each test's verdict and evidence. The job also fails when the run is blocked,
errors or is cancelled, and when the wait runs out. When the wait runs out or
the job is cancelled, the Action asks ProcessControl to cancel the run; a run
that has finished testing and only waits for review stays in the review queue.
The runner needs `bash`, `curl` and `jq`, which GitHub's hosted Linux and macOS
runners have.

## Pricing

[processcontrol.dev/pricing](https://processcontrol.dev/pricing) lists each
plan's runs at the same time, machine minutes and model budget.

## Issues

Report bugs and ask for features in this repository's
[issues](https://github.com/wrkrsh/processcontrol/issues). For anything about
your account, write to [support@processcontrol.dev](mailto:support@processcontrol.dev).
