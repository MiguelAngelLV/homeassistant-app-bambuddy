## 0.2.4

**Bambuddy v0.2.4**

## ⚠ Upgrade Notes — Read Before Updating

**Most users are upgrading from 0.2.3.2.** The 0.2.4 stable release lands on `main`, so the in-app **Apply Update** button in Settings → System → Updates works for everyone — the 0.2.3.x updater is hardcoded to `origin/main`, and the 0.2.4-beta updater resolves to the latest release tag via the GitHub releases API (respecting `include_beta_updates`). Either way, you get 0.2.4. Below are the explicit Docker + native paths for users who'd rather drive the upgrade from the command line.

**Make a backup before upgrading** via Settings → Backup → Create Backup. Native install with `update.sh` snapshots the database automatically and rolls back on failure. Docker and fully-manual paths don't.

### Docker

Make sure your `docker-compose.yml` `image:` line points at `:0.2.4` (or `:latest` for the rolling stable tag).

```bash
docker compose pull
docker compose up -d
```

While you're there, refresh `docker-compose.yml` from the repo: the entrypoint changes in 0.2.4 mean `user: "${PUID:-1000}:${PGID:-1000}"` is now removed (the new gosu entrypoint owns privilege drop), `PUID` / `PGID` env vars are added with the same defaults, and the `./virtual_printer:/app/data/virtual_printer` bind mount is commented out by default. If you depended on that bind mount to share VP certs with a host install, uncomment it again — first start auto-chowns it.

### Native install — recommended path

```bash
sudo BRANCH=v0.2.4 /opt/bambuddy/install/update.sh
```

The `BRANCH=` env var tells `update.sh` to fetch the `v0.2.4` tag instead of tracking `origin/main`. Omit it (`sudo /opt/bambuddy/install/update.sh`) to follow `main` going forward. The script handles backup, service stop/start, `pip install`, and the frontend build with the correct working directory.

### Native install — manual path

```bash
sudo systemctl stop bambuddy
cd /opt/bambuddy
sudo -u bambuddy git fetch origin --tags
sudo -u bambuddy git checkout v0.2.4
sudo /opt/bambuddy/venv/bin/pip install -r requirements.txt
sudo systemctl start bambuddy
```

### Behaviour changes to know about

- **Docker entrypoint replaces the `chmod 777 /app/data` workaround.** If you've been carrying that, drop it — gosu chowns `/app/data` + `/app/logs` to `PUID:PGID` on first start (idempotent via sentinel file, so restarts skip the recursive traversal). Bind-mounted host directories are chowned through the mount the first time the entrypoint sees wrong ownership.
- **PostgreSQL restore from SQLite Local Backup now works end-to-end.** The `cannot drop table printers` abort that bit Postgres adopters in 0.2.3.x is gone — unblocks the SQLite → Postgres migration path.
- **Path-prefixed reverse proxies remain unsupported.** The `base: ''` Vite config that shipped briefly in 0.2.4b2 was reverted because it broke camera popups and deep-route initial loads. If you've been running Bambuddy at `/bambuddy/` on Traefik / nginx / Cloudflare Tunnel subpath, the documented workaround (NPM addon + Cloudflare Tunnel at a real domain + HA Webpage panel via `TRUSTED_FRAME_ORIGINS`) keeps working.
- **SpoolBuddy kiosks: in-app update + System buttons now work.** Settings → Update Daemon and the QuickMenu System buttons (Restart Daemon / Restart Browser / Reboot / Shutdown) no longer return "API keys cannot be used for administrative operations" — all five now route through `INVENTORY_UPDATE`. Stop reaching for SSH.

---

**Highlights**

0.2.4 is the cumulative result of three beta cycles (b1 → b3) and post-b3 work — three big-ticket features sit at the centre: **Server-side slicing** via OrcaSlicer / Bambu Studio sidecar (closes the gap where Bambuddy users had to keep a slicer install just to re-slice their archive), **Slicer Bundle (.bbscfg) import** so preset selection no longer has to round-trip Bambu Cloud, and a **unified Spoolman inventory UI** with AMS slot assignments, Storage Location, NFC write, and a filament-catalog picker that brings Spoolman feature parity with the local-mode inventory. Around them: **MakerWorld URL-paste import**, **MFA at-rest encryption auto-bootstrap** (default-on, no setup), **GitHub backup extended to Gitea + Forgejo**, **Tailscale integration for virtual printers**, **per-event ntfy priority**, **long-lived camera stream tokens for HA / Frigate / kiosks**, **Library Trash Bin + auto-purge**, and **spool label printing** including a new 40×30 mm template, hex colour code, and a bolder brand line.

Plus the largest contributor wave to date — see attribution per item below.

---

**New Features**

- **Virtual Printer non-proxy modes now mirror the live target printer to the slicer** ([#1193](https://github.com/maziggy/bambuddy/issues/1193) follow-up) — Cached-as-base mirror so AMS/k-profile/camera/status passes through the VP to BambuStudio when running queue or review mode (previously only proxy mode).
 
- **Server-side slicing via OrcaSlicer / Bambu Studio sidecar** ([PR #1144](https://github.com/maziggy/bambuddy/pull/1144) by @maziggy) — Bambuddy now slices STL/STEP/3MF server-side via an `orca-slicer-api` (or BambuStudio-API) sidecar container. Pick a printer + filament + process preset triplet in the SliceModal, dispatch, and the resulting 3MF is dropped into the library / queue. Embedded-settings fallback when the CLI can't resolve a preset.

- **Slicer Bundle (.bbscfg) import** — Upload a BambuStudio "Printer Preset Bundle" once per printer, then pick from it for every subsequent slice. Closes the long tail of preset-resolution corner cases (cloud presets behind login, "from User" sentinels, the `# `-prefix clone trick, dangling `inherits` on renamed parents). Works alongside the cloud/local/standard preset tiers — pick "Slicer bundle" in the SliceModal to skip PresetRef resolution entirely.

- **Multi-color slicing in the Slice modal with per-plate filament discovery** ([PR #1205](https://github.com/maziggy/bambuddy/pull/1205) by @maugsburger) — Slice modal auto-discovers per-plate filament requirements via a preview-slice call on unsliced project files, so the AMS-slot picker knows which slots the plate actually consumes before dispatch.

- **MakerWorld URL-paste import** ([PR #1099](https://github.com/maziggy/bambuddy/pull/1099) by @maziggy) — Paste a MakerWorld model URL into the import dialog and Bambuddy resolves the plate instances, lets you pick one, and drops it into the library. Authenticated via your existing Bambu Cloud session.

- **Unified Spoolman inventory UI + AMS slot assignments + Storage Location + NFC write + filament-catalog picker** ([PR #1063](https://github.com/maziggy/bambuddy/pull/1063), [PR #1114](https://github.com/maziggy/bambuddy/pull/1114), [PR #1241](https://github.com/maziggy/bambuddy/pull/1241) by @netscout2001 / @maziggy) — Spoolman-backed installs now get feature parity with local inventory: the inventory page renders the same way regardless of backend, "Assign to AMS" works both directions, the SpoolBuddy kiosk can write NFC tags directly to Spoolman spools, and the filament catalog picker browses Spoolman's filament library inline.

- **Tailscale integration for virtual printers** ([PR #1070](https://github.com/maziggy/bambuddy/pull/1070) by @legend813, follow-up by @maziggy) — Virtual printer card surfaces the host's Tailscale IP + MagicDNS hostname with a copy button, so you can paste a tailnet IP into the slicer for private WireGuard reach without port forwarding. Per-VP opt-out toggle.

- **MFA at-rest encryption — default-on via auto-bootstrap** ([PR #1231](https://github.com/maziggy/bambuddy/pull/1231) by @netscout2001) — OIDC `client_secret` and TOTP secret rows are now Fernet-encrypted by default. The key is auto-bootstrapped from `MFA_ENCRYPTION_KEY` env var → `DATA_DIR/.mfa_encryption_key` → auto-generated. New Settings → Authentication → Security tab surfaces the key source, encrypted/legacy row counts, and a `decryption_broken` recovery flag. Key file is included in Local Backup ZIPs so the restore is self-contained.

- **GitHub Backup extended to Gitea + Forgejo** ([PR #1160](https://github.com/maziggy/bambuddy/pull/1160), [PR #1255](https://github.com/maziggy/bambuddy/pull/1255) by @BurntOutHylian) — Settings → Backup → Git Provider now supports Gitea (1.18+) and Forgejo (all versions) alongside GitHub.com and GitHub Enterprise. Gitea uses the Contents API for atomic multi-file commits; Forgejo v15+ token-scope quirk on `/repos/` is handled via a `/user` pre-check. Tested against Gitea 1.24.7 / 1.25.4 / 1.26.1 and Forgejo v11 / v15 LTS.

- **Stock forecasting + Logistics view** ([PR #1184](https://github.com/maziggy/bambuddy/pull/1184) by @Keybored02) — New Inventory → Logistics tab forecasts when each material/colour combo will run out based on rolling burn rate and lets you flag spools for re-order.

- **Spool label printing — DK label printers + AMS layouts** ([#809](https://github.com/maziggy/bambuddy/issues/809)) — Print labels for any spool directly from inventory: roomy single-label (62×29, 40×30), tight AMS-strip (3-up), and Avery 5160 sheets. New 40×30 mm template, hex colour code on every label (`#RRGGBB`), and a bolder Helvetica-Bold brand line at arm's-length-readable size.

- **Embedded GCode viewer** ([PR #963](https://github.com/maziggy/bambuddy/pull/963) by @Soopahfly) — "3D Preview" button on every archive and library file opens a PrettyGCode viewer iframe — no external slicer needed for visual confirmation of which plate is which.

- **Copy spool — duplicate any spool in two clicks** ([PR #1246](https://github.com/maziggy/bambuddy/pull/1246) by @MiguelAngelLV) — Copy button next to Edit on every inventory row prefills SpoolFormModal with the source spool's settings (except `weight_used`, reset to 0). Saves serial-data-entry time for users with many near-identical spools.

- **Build-plate icon on archive cards + uniform printer/model line** ([#1253](https://github.com/maziggy/bambuddy/issues/1253), reported by @tonygauderman) — Archive cards now show an OrcaSlicer-style bed icon (Cool / Cool SuperTack / Engineering / High Temp / Textured PEI / Smooth PEI) so users can tell which build plate the print was sliced for at a glance. Backfill script available for older archives.

- **Library Trash Bin + Admin Bulk Purge + Auto-Purge** ([#1008](https://github.com/maziggy/bambuddy/issues/1008)) — Deleted library files now land in a trash bin instead of being permanently removed. Admin → Library → Trash for bulk purge, plus a configurable auto-purge schedule.

- **Archive Auto-Purge** ([#1008](https://github.com/maziggy/bambuddy/issues/1008) follow-up) — Same pattern for print archives — auto-delete archives older than N days, with bulk-purge admin UI.

- **Project URL + cover photo** ([#1155](https://github.com/maziggy/bambuddy/issues/1155)) — Projects can now carry a source URL and a cover photo (auto-derived from the source 3MF or user-uploaded), rendered on the project list and detail views.

- **"Not Printed" / "Printed" collections on the Archives page** ([#1153](https://github.com/maziggy/bambuddy/issues/1153)) — Filter archives by whether they've been printed at least once, useful for "what's still untried" hunting.

- **Virtual-printer archive name source toggle** ([#1152](https://github.com/maziggy/bambuddy/issues/1152)) — Choose whether VP-spawned archives are named from the source 3MF filename or the slicer-supplied print job name.

- **Enhanced filament colour handling: multi-colour gradients, transparency, visual effects** ([#1154](https://github.com/maziggy/bambuddy/issues/1154)) — Spool form supports multi-colour gradient stops, transparency, and effect overlays (Sparkle, Silk, Matte) — rendered correctly on the spool cards, AMS slot view, and inventory.

- **Per-spool category + low-stock threshold override** ([#729](https://github.com/maziggy/bambuddy/issues/729)) — Each spool can override the global low-stock threshold and carry a custom category, surfaced in the Logistics view.

- **Per-event ntfy priority** ([#990](https://github.com/maziggy/bambuddy/issues/990)) — Set ntfy push priority per notification event type (Print complete → default, Filament out → urgent, etc.).

- **Long-lived camera-stream tokens for HA / Frigate / kiosks** ([#1108](https://github.com/maziggy/bambuddy/issues/1108)) — Generate scoped, non-expiring tokens for embedding Bambuddy camera streams in Home Assistant Webpage cards, Frigate dashboards, or kiosk displays without baking in admin credentials.

- **Filament Track Switch (FTS) support** — Detect and surface FTS-equipped printers' track-switch events on the printer card.

- **AMS slot Load / Unload from the printer card** ([#891](https://github.com/maziggy/bambuddy/issues/891), reported by @NNeerr00, +1 from @cadtoolbox) — Click any AMS slot on the printer card to load or unload the filament — no more reaching for the BambuStudio app.

- **API keys can read Bambu Cloud presets on the owner's behalf** ([#1182](https://github.com/maziggy/bambuddy/issues/1182), reported by @turulix) — API keys can now route Bambu Cloud preset reads through the owner's stored cloud session, so Home Assistant / external automations can slice without separate cloud auth.

- **Home Assistant addon detection** — Bambuddy auto-detects when it's running as the HA OS addon and surfaces the right webhook URL / iframe origin on the Settings page.

- **OIDC: Azure Entra ID support — configurable email claim & verification + Remember Me persistent login** ([PR #1126](https://github.com/maziggy/bambuddy/pull/1126) by @netscout2001) — Custom email claim path (Azure's `preferred_username`/`emails[0]`), per-provider "Require email verified" toggle, and a "Remember me" checkbox on the login page for opt-in 30-day sessions.

- **OIDC auto-created users now get readable usernames and land in a configurable group** ([PR #1176](https://github.com/maziggy/bambuddy/pull/1176) by @netscout2001) — `preferred_username` / `name` claim is consumed first, falling back to email-prefix; default group for auto-provisioned OIDC users is now a Settings dropdown instead of hardcoded "Viewers".

- **Slicer presets now span Cloud, imported, and slicer-bundled tiers, end-to-end** — Unified preset resolution across the three sources so SliceModal, the dispatch path, and the preview-slice cache all agree on which preset a pick resolves to.

- **Plate-clear tracking and visibility on printer cards** ([PR #939](https://github.com/maziggy/bambuddy/pull/939) by @EdwardChamberlain) — Plate-clear gate persists across container restarts and Auto-Off power cycles, with a clearer pill on the printer card showing "Awaiting plate clear" vs "Ready to print".

- **Printer page header update** ([PR #1203](https://github.com/maziggy/bambuddy/pull/1203) by @EdwardChamberlain) — Cleaner header layout on the Printers page.


---

**Improved**

- **Docker data-volume ownership normalised at startup via gosu entrypoint** ([#1211](https://github.com/maziggy/bambuddy/issues/1211)) — Replaces the `chmod 777 /app/data` hack with a proper entrypoint that chowns `/app/data` + `/app/logs` to `PUID:PGID` and drops to that uid via gosu. Subsequent restarts skip the recursive traversal via a sentinel file — no startup penalty on multi-GB archive directories.

- **Spool edit form: persistent Extra Colours, distinguishable Dual Color vs Gradient, more visible Sparkle/checkerboard visuals** ([#1154](https://github.com/maziggy/bambuddy/issues/1154) follow-up, reported by @maugsburger) — UX pass on the colour-effects added in b1.

- **AMS slot "Assign to inventory spool" picker now lists every spool, including RFID-tagged Bambu Lab ones** ([#1133](https://github.com/maziggy/bambuddy/issues/1133)) — Previously hid RFID-bound spools, forcing users to clear the RFID tag first.

- **Inventory: "Delete Tag" button renamed to "Clear RFID Tag"** ([#729](https://github.com/maziggy/bambuddy/issues/729) follow-up) — Less alarming wording; clearer what the button actually does.

- **Nozzle icon on the dual-nozzle status card** ([#1115](https://github.com/maziggy/bambuddy/issues/1115)) — Visually distinguish left/right extruder activity at a glance.

- **Settings page: permission-gated instead of admin-only** — Every settings sub-page now respects per-group permissions (e.g. `SETTINGS_NOTIFICATIONS` for Notifications, `SETTINGS_PROFILES` for Profiles, etc.) so non-admin users with specific scopes can manage their own sections.

- **i18n: full key parity across all 8 locales** — CI gate now enforces parity across en/de/fr/it/ja/pt-BR/zh-CN/zh-TW.

- **Per-request trace ID column on every log line** — Plumbed through HTTP access log, application logs, and response headers so a support bundle can trace one request end-to-end.

- **Live slicer progress in the persistent slice toast** — Real progress percentage instead of a spinner, with URL-decoded filenames in the toast title.

- **Background-dispatch toast no longer reads as "frozen at 100%" for fast uploads** — Fast uploads (small files / fast networks) no longer leave the toast stuck at 100% for several seconds.

- **SpoolBuddy kiosk no longer shows main-app toasts** — Kiosk-side notification suppression so operator-side toasts don't leak onto the kiosk display.

- **SpoolBuddy kiosk: "Plate ready" pills under the printer status badges** — At-a-glance plate state alongside printer state.

- **MakerWorld URL-paste resolver shows which printer each plate was sliced for** — So the picker shows e.g. "Plate 1 (X1C)" / "Plate 2 (P1S)" instead of unlabelled.

---

**Fixed**

### Slicing / Dispatch

- 3MF profile-driven slicing silently produced wrong-printer output (every slice fell back to the source's embedded printer regardless of the picked profile).
- Sliced-archive card listed every project-wide AMS slot instead of just the filaments the print actually used.
- Sliced output of a "single-color" plate had filaments the user never picked.
- Slice modal had no warning when the picked printer profile didn't match the source 3MF's bound printer.
- Settings warning when OrcaSlicer is selected as the preferred slicer (vs Bambu Studio).
- MakerWorld P2S 3MFs failed to slice with "Param values in 3mf/config error: -1 not in range" ([#1201](https://github.com/maziggy/bambuddy/issues/1201), reported by @inorichi).
- Slicer "Send to printer" silently rejected the cached push_status with "storage needs to be inserted" on P1S/A1-class targets ([#1228](https://github.com/maziggy/bambuddy/issues/1228), reported by @rtadams89 and @smandon).
- Slicing a library file via API key fails with "no Bambu Cloud session is stored" even when the key has cloud access ([#1182](https://github.com/maziggy/bambuddy/issues/1182) follow-up, reported by @turulix).
- Slice button no longer enabled before the preview slice resolves.
- Reprint-from-archive failed with `0500_4003` SD R/W errors after a stuck dispatch ([#1136](https://github.com/maziggy/bambuddy/issues/1136)).
- P1P print dispatch failed with `0500_4003 "can't parse print file"` when the printer was slow to acknowledge ([#1150](https://github.com/maziggy/bambuddy/issues/1150), reported by @d3ni3).
- H2D Pro multi-plate dispatch double-/triple-fire ([#1157](https://github.com/maziggy/bambuddy/issues/1157)).
- Background-dispatch reported "Print started successfully" when the printer never actually transitioned ([#1134](https://github.com/maziggy/bambuddy/issues/1134), follow-up to [#1042](https://github.com/maziggy/bambuddy/issues/1042)).
- Queue auto-dispatched the next print onto a fouled bed after an aborted or cancelled print ([#1171](https://github.com/maziggy/bambuddy/issues/1171), reported by @tom5677).
- Queue item stuck at "printing" when print failed before reaching RUNNING ([#1111](https://github.com/maziggy/bambuddy/issues/1111)).
- Queue: batch (quantity>1) double-dispatched onto the same printer.
- Queue: active-item progress bar flashed 100% before dropping to 0%.
- Virtual Printer queue mode auto-dispatched onto the wrong colour when multiple compatible printers were available ([#1188](https://github.com/maziggy/bambuddy/issues/1188), reported by @EdwardChamberlain).

### AMS / Filament / Inventory

- X2D / H2D dual-nozzle without AMS: filament mapping reported "Required filament type not found in printer" even when the spools were physically loaded ([#1257](https://github.com/maziggy/bambuddy/issues/1257)).
- New AMS RFID rolls auto-named to the wrong colour when the hex is shared across material variants (e.g. PLA Matte rolls named "Jade White") ([#1227](https://github.com/maziggy/bambuddy/issues/1227)).
- Bambu RFID auto-match created duplicate inventory rows for Quick-Add and non-Bambu-branded spools ([#918](https://github.com/maziggy/bambuddy/issues/918)).
- Filament usage double-counted when AMS auto-falls-back to a same-material spool ([#957](https://github.com/maziggy/bambuddy/issues/957)).
- Spool form's "Slicer Preset" dropdown silently dropped Local Profiles when Bambu Cloud was connected, and collapsed per-printer/per-nozzle variants into a single entry ([#1248](https://github.com/maziggy/bambuddy/issues/1248), reported by @andretietz).
- Spool auto-assign hit `IntegrityError` on Postgres when AMS pushes arrived in quick succession.
- AMS slot configuration intermittently fails to reach the printer after several configs in a row ([#1164](https://github.com/maziggy/bambuddy/issues/1164), reported by @RosdasHH).
- Spool assignment to a reset AMS slot left the slot unconfigured both in Bambuddy and on the printer.
- AMS slot truncation hid the `@printer 0.4 nozzle` suffix; inline hover expansion added ([#1237](https://github.com/maziggy/bambuddy/issues/1237), reported by @basziee).

### Virtual Printer

- VP queue mode dispatched onto the wrong colour with multiple compatible printers ([#1188](https://github.com/maziggy/bambuddy/issues/1188)).
- VP cached-as-base mirror now overlays SD/storage indicators (`home_flag`, `sdcard`, `storage`) so P1S/A1-class targets don't fail BambuStudio's pre-flight ([#1228](https://github.com/maziggy/bambuddy/issues/1228)).
- VP Tailscale cert-renewal restart silently failed mid-way (follow-up to [#1070](https://github.com/maziggy/bambuddy/issues/1070)).
- Virtual printer card's Tailscale FQDN copy button failed on HTTP.

### Backup / Git Providers

- Gitea backups silently failed after the first run; Forgejo v15 token-scope quirk broke "Test Connection"; many failure paths surfaced cryptic one-word errors ([#1224](https://github.com/maziggy/bambuddy/issues/1224) reported by @rtadams89, [#1239](https://github.com/maziggy/bambuddy/issues/1239) + [PR #1255](https://github.com/maziggy/bambuddy/pull/1255) by @BurntOutHylian).
- Backups to Gitea / Forgejo failed with "Failed to create tree" on empty repos and `list indices must be integers or slices, not str` on populated repos ([#1224](https://github.com/maziggy/bambuddy/issues/1224), [#1225](https://github.com/maziggy/bambuddy/issues/1225)).
- Backup restore silently lost most data (Postgres restore from a SQLite backup).
- Postgres restore from a SQLite Local Backup aborted with `cannot drop table printers`.

### OIDC / Auth

- OIDC callback code/state max_length raised from 512 to 2048 ([#1024](https://github.com/maziggy/bambuddy/issues/1024), [PR #1024](https://github.com/maziggy/bambuddy/pull/1024) by @netscout2001).
- OIDC `auto_link_existing_accounts` now works with custom email claims (Azure Entra ID) ([#1088](https://github.com/maziggy/bambuddy/issues/1088), [PR #1142](https://github.com/maziggy/bambuddy/pull/1142) by @netscout2001).
- OIDC issuer trailing-slash mismatch ([#995](https://github.com/maziggy/bambuddy/issues/995), [#985](https://github.com/maziggy/bambuddy/issues/985)).
- OIDC settings form: "Require email verified" toggle no longer jumps layout when auto-link is enabled.
- Setup: re-enabling auth could 422 on a password the form no longer needs.

### Camera

- Camera preview popup opened to a blank page; deep-route refresh and direct URL load broken ([#1221](https://github.com/maziggy/bambuddy/issues/1221), reported by @enjoylifenow / @Haeckan / @elit3ge / @jc21).
- External-camera frames returned as black on go2rtc and other MJPEG sources ([#1177](https://github.com/maziggy/bambuddy/issues/1177), reported by @nkm8).
- Camera page ignored `?fps=N` URL parameter ([#1131](https://github.com/maziggy/bambuddy/issues/1131) diagnostic).
- Camera stream second viewer fails / kicks the first off ([#1089](https://github.com/maziggy/bambuddy/issues/1089)).
- Camera TLS proxy logged unhandled exceptions when ffmpeg dropped its half of the connection mid-stream under uvloop.

### File Management / Library

- 3D Preview returned `{"detail":"Not Found"}` in Docker installs ([#1218](https://github.com/maziggy/bambuddy/issues/1218)).
- Archive 3MFs (and library file bytes) silently deleted from disk on every print completion ([#1212](https://github.com/maziggy/bambuddy/issues/1212), reported by @abbasegbeyemi).
- Archive created with wrong plate metadata when consecutive plates of the same model are printed back-to-back ([#1204](https://github.com/maziggy/bambuddy/issues/1204), reported by @BurntOutHylian).
- Archive Reprint colliding with originals (carryover fix from 0.2.3 cycle).
- Moving a file to an external folder updated the DB row but never wrote the bytes to the mount ([#1112](https://github.com/maziggy/bambuddy/issues/1112) follow-up).
- Uploads to writable external folders silently landed in internal storage ([#1112](https://github.com/maziggy/bambuddy/issues/1112)).
- GCode Viewer had no in-app way to navigate back (only the browser's back button worked).
- Archives card's "Reprint" / "Schedule" / "Slice" button labels truncated to "Re..." / "Sc..." on narrow browser windows ([#1249](https://github.com/maziggy/bambuddy/issues/1249)).
- Printer file download 500'd on non-ASCII filenames; same crash latent in three sibling endpoints ([#1245](https://github.com/maziggy/bambuddy/issues/1245), reported by @1000Delta).
- Reprint-from-Archive left `created_by_id` as `NULL` ([#730](https://github.com/maziggy/bambuddy/issues/730) follow-up).

### Print Queue / Notifications

- Print-complete notification reported the slicer's pre-print estimate instead of the actual elapsed time ([#1198](https://github.com/maziggy/bambuddy/issues/1198), reported by @BurntOutHylian).
- User-cancelled prints surfaced as "1 problem" on the printer card AND were archived as "Layer shift" failures.
- Pending review card and the resulting archive name disagreed; `.gcode.3mf` filename suffix wasn't fully stripped ([#1152](https://github.com/maziggy/bambuddy/issues/1152) follow-up, reported by @smandon).
- Auto-Print G-code Injection: start snippet landed before printer startup, and `{placeholder}` substitution was silently broken ([#422](https://github.com/maziggy/bambuddy/issues/422) follow-up).
- Plate-clear button stayed visible after the API cleared `awaiting_plate_clear` outside the printer-card click path ([#1128](https://github.com/maziggy/bambuddy/issues/1128)).

### Printer Card / UI

- Printer card's "Show on Printer Card" smart-plug button toggled power without confirmation ([#1260](https://github.com/maziggy/bambuddy/issues/1260), reported by @thkl).
- Printer card always shows the first plate's thumbnail when printing a multi-plate 3MF ([#1166](https://github.com/maziggy/bambuddy/issues/1166), reported by @smandon).
- Printer Info modal: serial-number and IP-address copy buttons silently did nothing on plain-HTTP LAN deployments ([#1174](https://github.com/maziggy/bambuddy/issues/1174), reported by @BurntOutHylian).
- Label picker modal clipped the 4th template option and Cancel button on short viewports ([#1230](https://github.com/maziggy/bambuddy/issues/1230), reported by @elit3ge).
- Project cover photo thumbnail too small to recognise the print ([#1155](https://github.com/maziggy/bambuddy/issues/1155) follow-up, reported by @smandon).
- Project picker UX in archives ([#1151](https://github.com/maziggy/bambuddy/issues/1151)).

### iframes / Reverse Proxies

- iframe embedding from trusted origins (e.g. Home Assistant Webpage panel) no longer blocked ([#1191](https://github.com/maziggy/bambuddy/issues/1191), reported by @azurusnova).
- Frontend served behind a path-prefixed reverse proxy loaded a blank page in 0.2.4b2 — reverted (see Upgrade Notes) ([#1195](https://github.com/maziggy/bambuddy/issues/1195) → reverted in 0.2.4b3).
- Spoolman iframe silently blank on HTTPS Bambuddy with HTTP Spoolman ([#1096](https://github.com/maziggy/bambuddy/issues/1096)).

### MakerWorld

- MakerWorld sidebar entry visible to every user regardless of group permissions ([#1175](https://github.com/maziggy/bambuddy/issues/1175)).
- MakerWorld P2S 3MF parse error on slice ([#1201](https://github.com/maziggy/bambuddy/issues/1201)).

### SpoolBuddy (Kiosk)

- SpoolBuddy install.sh re-run failed with `Permission denied` on root-owned files in update mode.
- SpoolBuddy SSH update aborted with `TypeError: startswith first arg must be bytes or a tuple of bytes, not str` after the host-key store succeeded.
- SpoolBuddy SSH update crashed on Postgres with `value too long for type character varying(500)` when storing the device's RSA host key.
- SpoolBuddy SSH update fails with "permission denied for user spoolbuddy" after Bambuddy keypair rotation.
- SpoolBuddy kiosk Settings → Update button returned "API keys cannot be used for administrative operations".
- SpoolBuddy with Spoolman: NFC tag scan looked up local DB first; Assign-to-AMS no-op on freshly-linked spools; AMS slot picker hid the assigned spool; LinkSpoolModal showed "Unknown color"; tag-write didn't enforce uniqueness; kiosk display held stale state.
- SpoolBuddy AMS page: re-assigning a just-unassigned spool sometimes showed an empty picker ([#1133](https://github.com/maziggy/bambuddy/issues/1133) follow-up).
- SpoolBuddy kiosk screen-blank timeout setting was ignored after the first save.
- SpoolBuddy kiosk screen never blanked while a load cell was producing noisy readings.

### Docker / Install / Logging

- Docker permission errors on `/app/data/virtual_printer` and similar paths — fixed via gosu entrypoint (see Highlights).
- In-app upgrade was hardcoded to `origin/main` and silently no-op'd whenever the latest release wasn't on main.
- In-app upgrade clobbered SSH `origin` on developer checkouts.
- Native-install in-app upgrade silently skipped `pip install` and the new dependencies never landed.
- Settings table filled with duplicate rows on legacy SQLite installs.
- Install script failed for first-time users.
- "Open in Slicer" fails on Windows / Linux for any filename containing spaces or special characters ([#1059](https://github.com/maziggy/bambuddy/issues/1059)).
- `bambuddy.log` filling with `Exception terminating connection ... CancelledError` + `database is locked` cascades on long uploads ([#1112](https://github.com/maziggy/bambuddy/issues/1112) follow-up).
- Windows install: `bambuddy.log` filling with `WinError 10054`.
- `logs/bambuddy.log` was silently dropping records from named child loggers.
- Uvicorn HTTP access log was missing from `bambuddy.log`.
- Swagger UI link in Settings → API Keys rendered a blank page.

### Misc

- H2C dual-nozzle detection missed post-2026 serial batches ([#1105](https://github.com/maziggy/bambuddy/issues/1105)).
- Groups: edits to custom-group permissions appeared lost on reopen ([#1083](https://github.com/maziggy/bambuddy/issues/1083)).
- Settings: failed-save toast looped forever when the user lacked `settings:update`.
- Settings → API Keys: deleted key stayed on screen until manual reload.
- i18n placeholder mismatches in Japanese rendered literal `{{count}}` / `{{name}}` strings in the UI.
- `formatTimeOnly` tests failed under non-`:`-separator locales ([#1213](https://github.com/maziggy/bambuddy/issues/1213), reported by @maugsburger).

---

**Security**

- **python-multipart bumped to >=0.0.27** to clear CVE-2026-42561 (b3).
- **pip upgraded to >=26.1 inside the Docker image** to clear CVE-2026-6357 (medium; GitHub code-scanning alert #778). No `requirements.txt` change — the floor is enforced at the image-build layer where the vulnerable copy lived. (libexpat1 alert #795 also flagged by code-scanning is a DoS-only XML attribute-collision CVE with no patched Debian trixie package yet — left open as a tracking signal.)

---

**Contributors**

Big thanks to everyone who shipped code this cycle:

@netscout2001, @BurntOutHylian, @EdwardChamberlain, @Soopahfly, @legend813, @Keybored02, @maugsburger, @MiguelAngelLV

(See [CHANGELOG.md](https://github.com/maziggy/bambuddy/blob/main/CHANGELOG.md) for the full per-fix detail.)

