# Agent setup

These steps are for a coding agent setting up ProcessControl in the repository
it works in. The person you work with signs in once and approves; you do the
rest. ProcessControl then tests pull requests, each on a machine of its own:
an agent builds and launches the app, checks the claims it finds in the PR
description, and reports a GitHub check that links to the run's evidence.

## Start a setup request

From the repository checkout:

```sh
repo=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
curl -fsS https://processcontrol.dev/api/v1/setup \
  -H 'Content-Type: application/json' -d "{\"repo\":\"$repo\"}"
```

The response has `url`, `userCode`, `deviceCode`, `interval` and `expiresIn`.
Show the person the `url` and the `userCode`, and ask them to open the link. On
that page they sign in with GitHub, install the GitHub App on the repository,
optionally connect Codex, and give you access. Keep `deviceCode` to yourself.

## Wait for access

Every `interval` seconds, until `expiresIn` runs out, poll and save the answer
to a file only you can read. The last answer holds the key, so it never goes
to your output:

```sh
umask 077
curl -sS https://processcontrol.dev/api/v1/setup/key \
  -H 'Content-Type: application/json' -d '{"deviceCode":"<deviceCode>"}' \
  -o ~/.processcontrol-<userCode>.json -D ~/.processcontrol-<userCode>.headers \
  -w '%{http_code}\n'
grep -i '^retry-after' ~/.processcontrol-<userCode>.headers || true
grep -o '"status":"[a-z]*"\|"error":"[^"]*"' ~/.processcontrol-<userCode>.json
```

The first line printed is the HTTP status:

- 200 with `"status":"pending"`: keep waiting. The person may take a few
  minutes, so if your commands have a time limit, poll in the background or
  across several commands.
- 200 with `"status":"complete"`: the person approved. The file holds the
  `apiKey`, returned only once, and the `mention`. Each of your commands may
  run in a new shell, so read the key from the file when you need it. Never
  commit, print or paste it.
- 403: the person denied the request. 410: it expired, or its key was already
  returned. If that answer never reached you, the key is lost: tell the person
  which key the `error` says to revoke. Start again only if they ask.
- 429 or 503: wait the seconds its `retry-after` line gives, then poll again.

## Describe how to run the app

The tester reads the repository’s AGENTS.md (or the CLAUDE.md it points to) and
READMEs at the tested commit. Check that they say which app to test, how to
install, build and start it, and which services and fixtures it needs. Add what
is missing in the pull request below. Never put credentials in the repository.
**Setup** lists what ProcessControl found the app needs: environment
variables, services and a test account. Ask the person to fill in what it
marks missing there; runs wait for the items it says they wait for.

## Test a pull request

Open a pull request whose description states what should work, for example
“The app builds and its home screen loads.” Then request a test with a new
comment that starts with the `mention` from the key response:

```sh
gh pr comment <number> --body "$(jq -r .mention ~/.processcontrol-<userCode>.json)"
```

## Read the result

When **Post results in** includes **PR thread**, the default, the app replies
on the pull request with the results. To wait for the result, look at
the **ProcessControl · Pull request** check every minute or so;
`gh pr checks --watch` would also wait for every other check on the pull
request:

```sh
gh pr checks <number> --json name,bucket,link \
  --jq '.[] | select(.name == "ProcessControl · Pull request")'
```

A check titled **Waiting for setup** names what the run waits for: send the
person its **Open Setup** link. Once `bucket` leaves `pending`, read the run;
its `runId` is the last segment of `link`:

```sh
curl -fsS https://processcontrol.dev/api/v1/runs/<runId> \
  -H "Authorization: Bearer $(jq -r .apiKey ~/.processcontrol-<userCode>.json)"
```

`status` is `passed`, `failed`, `blocked`, `errored` or `cancelled` once the
run ends. `awaiting_approval` means the pull request comes from a fork and the
run waits for an admin to approve it in ProcessControl; ask the person to.
`result.tests[].claims` holds each claim with its `verdict`
(`verified`, `contradicted` or `blocked`) and `evidence`; act on each one
that did not verify:

- `contradicted`: the app does not do what the pull request says. Fix the
  code, or the description if it overstates the change.
- `blocked` with `blocker: "setup"`: add the build steps, services or
  fixtures the tester was missing to AGENTS.md or the README. A missing key or
  other secret goes in **Setup**, never in the repository: ask the person to
  add it there.
- `blocked` with `blocker: "account"`: tell the person which app and login
  page the claim's evidence names. When the run's `status` is `blocked`, send
  them the check's **Add a test account** link, which opens the form in the
  run's workspace for this repository; otherwise the check has no such link,
  and the form is **App accounts** in ProcessControl's Settings.
- `blocked` with `blocker: "platform"`: the claim needs another operating
  system or device; say so in the pull request.

Then push: once a mention has started a run, a push to the pull request or an
edit to its title or description starts a test of its latest commit.
