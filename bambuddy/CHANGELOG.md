## 0.2.4.4

# Bambuddy 0.2.4.4 — Critical security hotfix

**Severity: Critical (CVSS 9.8) — upgrade immediately if you have any Bambuddy instance reachable from an untrusted network.**

This release contains a single fix for [GHSA-6mf4-q26m-47pv][adv] — a fail-open authentication bypass that let an unauthenticated attacker trigger an internal database error and reach every protected endpoint during the resulting fail-open window.

[adv]: https://github.com/maziggy/bambuddy/security/advisories/GHSA-6mf4-q26m-47pv

## Who is affected

- Both authentication-enabled and authentication-disabled installs serve the vulnerable code path; an auth-disabled install simply lacked the protection the attacker was bypassing.
- The attack works from any network the Bambuddy instance is reachable  from — LAN, Tailscale, port-forwarded internet, reverse-proxied, containerised, bare-metal. Auth-enabled deployments behind a VPN are not protected during the fail-open window because the bypass happens *inside* Bambuddy, after the network reaches it.

## What the bug did

Two functions in the auth path caught every exception and treated the failure as "auth is disabled, allow the request":

- `is_auth_enabled` returned `False` on any database error.
- The global `auth_middleware` returned `await call_next(request)` on any exception during the auth-state probe.

An attacker who could trigger any DB exception — the reporter's documented proof-of-concept exhausts the process's file-descriptor budget by flooding `/api/v1/auth/login`, until the next SQLite `connect()` raises — could then send a request to any protected endpoint during that window with **no token at all** and have it served.

## What the fix does

- `is_auth_enabled` no longer catches the broad `Exception`. The legitimate "auth was never configured" path (settings row absent) still returns `False`; any actual exception now propagates so the caller can deny the request.
- `auth_middleware` now returns **503 Service Unavailable** on any probe failure instead of letting the request through.

The principle is "fail closed": if Bambuddy cannot verify the authentication state, the request is denied — not granted.

## What you should do

1. **Upgrade immediately.** Pull the new image or update your install:

### Docker

docker compose pull
docker compose up -d

docker-compose.yml doesn't need refreshing — none of the entrypoint, volume, or env-var conventions changed since 0.2.4.

### Native install — recommended path

sudo BRANCH=main /opt/bambuddy/install/update.sh

Snapshots the database first and rolls back on failure.

### Native install — manual path

sudo systemctl stop bambuddy
cd /opt/bambuddy
sudo -u bambuddy git fetch --prune --tags --force origin
sudo -u bambuddy git checkout main
sudo -u bambuddy git reset --hard origin/main
sudo /opt/bambuddy/venv/bin/pip install -r requirements.txt
sudo systemctl start bambuddy

## Credit

Reported responsibly by [@wondercrash][reporter] via private security advisory on 2026-05-30. The failure was ours.

[reporter]: https://github.com/wondercrash

## Tests

Four new regression tests pin the fail-closed contract; an existing security test was updated to accept the stronger middleware-level denial. Full backend suite green (5428 tests).

