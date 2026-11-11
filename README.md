# dns-diag

Throwaway diagnostic sandbox for the GitHub-runner DNS breakage that
kills `apt-get update` inside docker builds (2026-11, hits the
opencode-permissions-kit e2e suites). NOT shipped anywhere — exists so
we can burn cheap one-job runs here instead of debug-commits on the
real repo's PRs.

## The incident in one paragraph

Runner host resolves fine all along (api.github.com, npm, gcr.io,
Azure blob), but apt inside `docker build` fails on EVERY name —
`archive.ubuntu.com`, `security.ubuntu.com`, later
`azure.archive.ubuntu.com` too — on bridge AND `--network=host`, with
`Could not resolve` / `Temporary failure resolving`. The kit's
e2e suite was green on the same build path at 05:19 UTC and red at
15:06+ the same day: something rolled platform-side mid-day. The
runner's `/etc/resolv.conf` is the systemd-resolved stub
(`127.0.0.53`); BuildKit substitutes loopback nameservers for build
containers, and whatever it substitutes is what breaks.

## What one run answers

| Step | Question |
|---|---|
| Q1+Q2 | Does the HOST resolve the archive domains? Which upstreams does systemd-resolved actually use? |
| Q3 | Which public resolvers answer from the runner host? (also: does host-side apt work?) |
| Q4 | What resolv.conf does a build container get on the bridge? |
| Q5 | Same with `--network=host` — does in-container resolution work at all? |
| Q6 | Does rewriting resolv.conf + restarting docker fix the build? (dockerd/buildkitd may only read resolv.conf at startup — our rewrite came after the daemon was up) |

## How to run

One job, one runner, ~2–3 minutes. Push anything (or dispatch) and
read the job log top to bottom — each step prints its own verdicts
(`OK` / `FAIL` / `NO ANSWER`). The interesting markers:
`BRIDGE-RESOLVE-*`, `HOSTNET-RESOLVE-*`, `AFTER-RESTART-RESOLVE-*`,
`HOST APT FAILED`.

Decision table for the fix on the real repo:

- Q1 FAIL → archive side/platform-wide, nothing container-related: wait.
- Q4/Q5 show a DIFFERENT nameserver than we wrote → BuildKit ignores
  the file; drop that approach, go for /etc/hosts IP pinning (resolve
  on host, pass as build-arg) or a prebuilt base image via GHCR.
- Q5 FAIL but Q6 OK → buildkitd caches resolv.conf at startup: the
  real repo needs `sudo systemctl restart docker` after the rewrite.
- Q6 FAIL with public resolvers answering on the host (Q3 OK) →
  container egress UDP/53 is filtered; DNS inside builds is dead —
  go /etc/hosts IP pinning or prebuilt image.
