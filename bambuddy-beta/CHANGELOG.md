## 0.2.4b2

  **Bambuddy v0.2.4b2**

  The post-b1 polish cycle. The headline is Forecast & Reorder Intelligence — a new tab in the Inventory page that analyses your print history, projects stock runout dates per material/brand/subtype, and surfaces a shopping-list workflow. Two long-tail user pain points also closed: spool label printing (PDF in four sizes, including AMS-holder and Avery sheets) and Virtual Printer non-proxy modes now mirror the live target-printer state to the slicer (AMS configuration, FTS routing, camera, and k-profile lookup all work from BambuStudio / OrcaSlicer without proxy mode). On top of those, ~30 fixes — most notably a silent backup-restore data-loss bug (#1211 / #668), an archive-3MF file-deletion bug introduced by recent dispatch changes (#1212), and a string of VP / queue / scheduler edge cases.

  **Upgrade Notes — Read Before Updating**

  If you're already on **0.2.4b1**, the in-app *Apply Update* button
  in Settings → System → Updates can install b2 directly — b1's updater
  resolves the target tag from the GitHub releases API and respects
  `include_beta_updates`. The 0.2.3.x updater can't (it's hardcoded to
  `origin/main`), so 0.2.3.x → b2 still needs the explicit branch path
  documented in the b1 release notes.

  Make a backup before upgrading via Settings → Backup → Create Backup.
  Native install with `update.sh` snapshots the database automatically
  and rolls back on failure.

  **Docker**

  Make sure your `docker-compose.yml` `image:` line points at `:0.2.4b2`
  (or `:beta` for the rolling beta tag).

  docker compose pull
  docker compose up -d

  **Native install — recommended path**

  sudo BRANCH=0.2.4b2 /opt/bambuddy/install/update.sh

  The `BRANCH=` env var tells `update.sh` to pull `origin/0.2.4b2`
  instead of `origin/main`. The script handles backup, service
  stop/start, pip install, and frontend build with the correct
  working directory.

  **Native install — manual path**

  sudo systemctl stop bambuddy
  cd /opt/bambuddy
  sudo -u bambuddy git fetch origin
  sudo -u bambuddy git checkout 0.2.4b2
  sudo /opt/bambuddy/venv/bin/pip install -r requirements.txt
  sudo systemctl start bambuddy

  The new Forecast tab adds two columns to `filament_sku_settings` and two new tables (`filament_shopping_list`, plus stock-alert columns on `notification_providers`); migrations are idempotent and run automatically on startup. Existing groups with `inventory:read` / `inventory:update` are auto-granted the new `inventory:forecast_read` / `inventory:forecast_write` permissions
  on first start, so existing users aren't locked out of the tab.

  ---

  **Highlights**

  - **Virtual Printer non-proxy modes mirror the live target printer to the slicer** ([#1193](https://github.com/maziggy/bambuddy/issues/1193) follow-up) — Until now, Immediate / Review / Print Queue VPs looked like a stub Bambu Lab printer to the slicer: AMS dropdowns were empty, no live state, no camera, no per-filament k-profile lookup. The user could send a sliced file and that was it. Now the VP fans out the **target printer's live MQTT state** to the slicer (AMS units, FTS / dual-extruder routing, nozzle, temps, k-profiles, AMS load / dry / calibration commands) and proxies the **camera RTSPS stream** on port 322 — so the slicer treats the VP as a fully-functional Bambu printer while Bambuddy's queue / archive / dispatch features stay in the loop. Shares Bambuddy's existing per-printer MQTT subscription (no second session on the printer — firmware in-flight budget unaffected). Tested e2e with both BambuStudio and OrcaSlicer against H2D (dual-nozzle, AMS 2 Pro + AMS HT) and X1C (single-nozzle, AMS) across all three non-proxy modes. Setup nuance: for the slicer's RTSPS camera path to authenticate, the VP's access code must match the target printer's access code — one-time configuration step. Proxy mode is unaffected and continues to use its existing dedicated MQTT/FTP/RTSP/Bind/Aux proxies.

  - **Forecast & Reorder Intelligence** ([#1184](https://github.com/maziggy/bambuddy/pull/1184), contributed by @Keybored02) — A new *Forecast* tab on the Inventory page. The panel groups your spools by material / brand / subtype, computes a daily-consumption rate from the print-event history (exponentially weighted with a 30-day half-life so recent prints dominate; falls back to a delta calculation when only one print event exists), and projects per-SKU stock runout dates. Each row has a configurable lead time and safety margin (in days or grams), with the global lead time acting as a floor. Two alert tiers — *Reorder Now* (stock has dropped below the reorder point) and *Stock Break Risk* (stock will run out before reorder arrives) — surface in a collapsible banner above the table. A new shopping-list panel lets you add by quantity or by *days of stock desired* (auto-converted from the daily rate), with status badges (pending / ordered / received) and an inline urgency indicator. Per-SKU alert snooze for materials being intentionally wound down. All gated by two new permissions (`inventory:forecast_read` / `inventory:forecast_write`) with a backward-compat policy that auto-grants them to existing groups. i18n across all 8 locales.

  - **Spool label printing** ([#809](https://github.com/maziggy/bambuddy/issues/809)) — A per-spool *Print label* button on every Inventory card and a *Print labels (N)* header action that prints labels for the currently filtered view. Generates a PDF in one of four fixed sizes — AMS holder (30×15 mm) for the popular Makerworld AMS Filament Label Holder, single box label (62×29 mm) for Brother PT/QL or Dymo small labels, Avery L7160 for A4 sheet stock (38.1×63.5 mm × 21 per page), and Avery 5160 for US Letter sheet stock (25.4×66.7 mm × 30 per page). Each label shows a colour swatch (with multi-colour gradient stripes for spools that have `extra_colors` set), brand + material, the spool's own name, the spool ID — the field originally requested for "find spool 7 in my closet" identification — and a QR code that deep-links to `/inventory?spool=<id>` so a phone scan jumps straight back to that
  spool's row. The box-size template additionally surfaces the storage-location field. Pure-Python ReportLab renderer — no headless browser, no system libs. Both local and Spoolman-backed inventories supported.


  **New Features**

  - **AMS slot Load / Unload from the printer card** ([#891](https://github.com/maziggy/bambuddy/issues/891), reported by @NNeerr00, +1 from @cadtoolbox) — The MQTT primitives for "load filament from a tray" and "unload the currently loaded tray" already existed in the codebase but were unused — there was no HTTP route and no UI, so every Load / Unload had to happen on the printer touchscreen, and external-spool users on dual-nozzle H2D had no way to drive Ext-R from the desktop at all. Now the AMS slot popover (the one with *Re-read RFID*) gains *Load* and *Unload* entries, and the external-spool slot — which had no popover at all before — gets one with the same entries. On dual-nozzle H2D each external slot drives its own extruder. Hidden while the printer is `RUNNING`, gated on `Permission.PRINTERS_CONTROL`.

  - **API keys can read Bambu Cloud presets on the owner's behalf** ([#1182](https://github.com/maziggy/bambuddy/issues/1182), reported by @turulix) — Headless slicing pipelines hit a wall: the auth gate returned `None` for API-keyed requests, so `/cloud/*` routes always saw `user=None` and resolved an empty token. API keys now carry a `user_id` (FK to the creator) and a new opt-in `can_access_cloud` scope; cloud routes resolve the cloud token via the key's owner. Legacy ownerless keys keep working against every non-cloud route.
    
  - **Home Assistant addon detection — Settings → Updates and the in-app update banner now defer to the HA Supervisor** ([#1167](https://github.com/maziggy/bambuddy/issues/1167)) — Bambuddy on HA Supervisor used to show its own update controls alongside the addon's, leading users to "update Bambuddy" inside the app, then have HA roll it back to whatever version
   the addon shipped. The Updates page and update banner now detect the HA addon environment via `/data/options.json` + `SUPERVISOR_TOKEN` and surface "Update via Home Assistant" instead, with a deep link into the HA addon panel. Native and Docker-Compose deployments are unaffected.

  - **OIDC auto-created users now get readable usernames and land in a configurable group** ([#1173](https://github.com/maziggy/bambuddy/issues/1173), via [#1176](https://github.com/maziggy/bambuddy/pull/1176)) — Auto-created OIDC users were getting the raw `sub` claim as their username (a long opaque GUID on Azure / Google), and were defaulting to the most-permissive group available. Now the username comes from `preferred_username` / `name` / `email` (with a configurable claim) and the default group is settable per OIDC provider so SSO logins can drop straight into Operators or Viewers instead of Administrators.

  - **Filament Track Switch (FTS) support** ([#1162](https://github.com/maziggy/bambuddy/issues/1162)) — H2D / X2D printers with the FTS accessory installed used to render an empty filament dropdown in the print modal because the routing extension wasn't recognised. The print modal now correctly enumerates FTS sources alongside AMS slots, and a debug log records the slicer-launched `project_file` payload for routing diagnostics in case future FTS configurations surface new edge cases.

  - **External-camera snapshot URL override** ([#1177](https://github.com/maziggy/bambuddy/issues/1177)) — go2rtc, Frigate, and similar reverse-proxied camera setups can now expose a separate snapshot URL distinct from the live stream URL — useful for thumbnailing, archives, and the Cameras page where pulling an MJPEG keyframe is overkill.

  - **iframe embedding from trusted origins via `TRUSTED_FRAME_ORIGINS`** ([#1191](https://github.com/maziggy/bambuddy/issues/1191), reported by @azurusnova) — Anti-clickjacking defaults (`X-Frame-Options: SAMEORIGIN`, CSP `frame-ancestors 'none'`) blocked legitimate Home Assistant Webpage panel embeds even on same-LAN setups. New `TRUSTED_FRAME_ORIGINS` env var takes a comma-separated list of `scheme://host[:port]` origins; when set, the middleware drops the legacy `X-Frame-Options` header and the CSP `frame-ancestors` directive becomes `'self' <origin>...`. Default empty keeps the strict behaviour.

  - **Backup destinations beyond GitHub** ([#1160](https://github.com/maziggy/bambuddy/pull/1160)) — Settings → Backup → Git providers now supports GitLab, Gitea, and Bitbucket alongside GitHub. Same auth/token flow, same scheduled-backup pipeline.

  **Improved**

  - **Multi-colour filament rendering: Extra Colours hydrate on edit, Dual Color renders as hard-split bars, Sparkle/checkerboard visuals are more visible** ([#1154](https://github.com/maziggy/bambuddy/issues/1154) follow-up, reported by @maugsburger) — Four bugs against the original multi-colour work in b1: the Spool edit form lost the Extra Colours value when reopened (form initialised from a different state branch than the new field), Dual Color rendered identically to Gradient (the special-case stops weren't being detected), and Sparkle / checkerboard layers were too subtle on light backgrounds. All fixed; PR also adds a deterministic seeded PRNG (`utils/random.ts`) so visual variation is stable per spool key.

  - **Project cover-photo hover preview** ([#1155](https://github.com/maziggy/bambuddy/issues/1155) follow-up, reported by @smandon) — The 40×40 thumbnail wasn't readable for "is this the right model?" recognition; enlarging it would have shifted the dense grid layout the user chose. Hovering a card now mounts a 384×384 portal popover with the full cover image
  (`object-contain` so portrait shots aren't cropped), edge-flipping when the thumbnail is near the viewport's right side.

  - **Pending review card and the resulting archive name agree** ([#1152](https://github.com/maziggy/bambuddy/issues/1152)) — When the VP archive-naming source toggle was set to use the embedded 3MF metadata title, the resulting archive name kept the trailing `.gcode.3mf` suffix while the pending-review card stripped it, so the two views disagreed. Stripping is now consistent.

  - **Permission gating for MakerWorld nav entry** ([#1175](https://github.com/maziggy/bambuddy/issues/1175)) — The MakerWorld sidebar entry was visible to every user regardless of group permissions, even though the routes themselves were correctly gated. Sidebar now hides when the user lacks `makerworld:view`.

  - **Printer Info modal: copy buttons work on plain HTTP** ([#1174](https://github.com/maziggy/bambuddy/issues/1174)) — `navigator.clipboard.writeText` requires HTTPS or `localhost`; on plain-HTTP LAN deployments the serial-number and IP-address copy buttons silently failed. Now falls back to the legacy `document.execCommand('copy')` path so copying works regardless of context.

  - **MQTT inflight ceiling raised to prevent QoS=1 session wedge** ([#1164](https://github.com/maziggy/bambuddy/issues/1164), diagnosis credit @RosdasHH for the QoS=1/0/2 bisect) — paho's default 20-message inflight ceiling was getting filled by chatty AMS slot config sequences, leaving the session wedged until reconnect. Lifted to a higher value plus an `ams_filament_setting` response counter reset that was missing from the unanswered-counter logic.

  **Changed**

  - **Virtual Printer Tailscale toggle no longer provisions Let's Encrypt certs — it's now informational** — End-to-end testing confirmed the original premise was wrong: BambuStudio and OrcaSlicer both refuse hostname input in the Add Printer dialog (IP-only), and their printer-MQTT trust path validates only against the bundled BBL CA store (`printer.cer`), not the system trust store. LE-issued certs don't chain to BBL CA, so the slicer rejects with the well-known "-1" before any hostname/IP logic runs. The Tailscale toggle is kept (it's still useful for surfacing the FQDN + private WireGuard reach without port-forwarding) but the cert-provisioning machinery (renewal task, daily restart, on-disk LE files per VP) is gone. The `tailscale_disabled` DB column is preserved as the persisted toggle state. CA import into the slicer is unchanged. Wiki, README, and i18n copy across all 8 locales updated to drop the "no cert import needed" framing.

  **Fixed**

  - **Backup restore silently lost most data** ([#1211](https://github.com/maziggy/bambuddy/issues/1211), reported by @Carter3DP; same shape as previously-closed [#668](https://github.com/maziggy/bambuddy/issues/668)) — Restoring a backup ZIP appeared successful but the user found settings reverted to defaults and most printers / archive rows missing. Cause: SQLite WAL state from the fresh container start was being silently re-applied on top of the restored DB. The original code used `shutil.copy2` after
  `engine.dispose()` — neither of which checkpoints the WAL, and SQLAlchemy's dispose() doesn't close checked-out connections (the route handler's own `db: Depends(get_db)` is one). On the next open SQLite re-applied the stale frames over the restored content, partially clobbering it with fresh-install state. Fixed by replacing the file copy with SQLite's online backup API (`src.backup(dst)`), which understands WAL semantics and routes new pages through the destination's own WAL. Pinned with 6 regression tests including one that asserts the bug *manifests* under the un-checkpointed-WAL condition so a future "small simplification" can't silently re-introduce file-copy semantics. PostgreSQL path was already row-by-row and is unchanged.

  - **Archive 3MFs (and library file bytes) silently deleted from disk on every print completion** ([#1212](https://github.com/maziggy/bambuddy/issues/1212), reported by @abbasegbeyemi) — Reprint and View G-code on a freshly-completed archive returned 404 with no log line; the DB row was intact, the archive grid kept showing the entry, but `archive.file_path` pointed at a path that no longer existed on disk. Same shape independently reported by a daily-build user whose `.gcode.3mf` "disappeared by itself overnight" between Saturday's print and Monday morning's reprint attempt. Cause was a regression introduced by [#1166](https://github.com/maziggy/bambuddy/issues/1166)'s cover-cache pre-population: dispatch sites started caching the live archive copy in the shared 3MF download cache, but `clear_3mf_cache(printer_id, delete_files=True)` — called from `on_print_complete` to keep the temp dir from accumulating — happily `unlink()`'d every cached path. Pre-#1166 every cached path was a temp file and deletion was correct; post-#1166 the cleanup was destroying user data. Cache cleanup now refuses to delete any path outside `archive_dir/temp`. Affected daily builds since `889c8bd8` (Apr 29) — recovery is to re-import the source 3mf or re-archive from FTP.

  - **MakerWorld P2S 3MFs failed to slice with "Param values in 3mf/config error: -1 not in range"** ([#1201](https://github.com/maziggy/bambuddy/issues/1201), reported by @inorichi) — Slicing any MakerWorld model sliced for the P2S bombed with `Slicer process failed (exit code 238)`. Cause: BambuStudio writes `"-1"` into `Metadata/project_settings.config` for fields the user wants inherited from the parent process preset. The headless CLI runs `StaticPrintConfig`'s range validator against the embedded settings *before* `--load-settings` overrides apply, so the sentinel `"-1"` trips the field's lower-bound check. New   _sanitize_project_settings_sentinels` opens the embedded config and removes only allowlisted keys 
  (`raft_first_layer_expansion`, `tree_support_wall_count`, `prime_tower_brim_width`) when their value is exactly `"-1"`. Other fields and legitimate negative values are left untouched.

  - **Archive created with wrong plate metadata when consecutive plates of the same model are printed back-to-back** ([#1204](https://github.com/maziggy/bambuddy/issues/1204), reported by @BurntOutHylian) — Print Plate 2 of any multi-plate project, let it complete, then immediately print Plate 1: the resulting archive was named "MyModel - Plate 2" with Plate 2's filament slots and slicer estimate, even though Plate 1 was the print actually running. Cause was an MQTT lag in the `print_start` data: the trigger fires on `gcode_file` (plate-specific, always fresh) but `subtask_name` (model-level) can still echo the previous job. Fix cross-references the downloaded 3MF's plate index against the parsed gcode_file plate, and on mismatch retries the FTP fetch with a corrected name.

  - **Print-complete notification reported the slicer's pre-print estimate instead of actual elapsed time** ([#1198](https://github.com/maziggy/bambuddy/issues/1198), reported by @BurntOutHylian) — `{{duration}}` template variable was being filled from `print_time_seconds` (the slicer's estimate) — a 2-minute cancellation of a 3-hour estimate told the user "duration: 3h". Now computes `actual_time_seconds` from `(completed_at - started_at)` and prefers it. Also fixes the related case where `cancelled` prints didn't get a `completed_at` timestamp (only `completed/failed/aborted` did).

  - **Frontend served behind a path-prefixed reverse proxy loaded a blank page** ([#1195](https://github.com/maziggy/bambuddy/issues/1195), reported by @Spegeli) — Vite's default `base: '/'` emits absolute asset URLs in the built `index.html`, so deployments behind Traefik / nginx / Cloudflare Tunnel with a path prefix would 404 on every asset. Switched to `base: ''` so Vite emits relative paths; the SPA now loads correctly under any subpath. Note: API base remains absolute by design — full subpath-aware  bootstrapping is still out of scope for HA Ingress.

  - **Virtual Printer queue mode auto-dispatched onto the wrong colour when multiple compatible printers were available** ([#1188](https://github.com/maziggy/bambuddy/issues/1188), reported by @EdwardChamberlain) — A job sliced for matte white PLA would land on a printer with no white loaded. Cause: the VP queue-write path skipped extracting per-slot filament requirements from the 3MF, so the scheduler fell back to model-only matching. Now extracts requirements at queue-add time. New per-VP `queue_force_color_match` setting (default off for upgrade safety) — flip it on to require colour matching for queue dispatch.

  - **Slicing a library file via API key fails with "no Bambu Cloud session is stored"** ([#1182](https://github.com/maziggy/bambuddy/issues/1182) follow-up, reported by @turulix) — After the `/cloud/*` gate from the main #1182 fix, the slice route still failed because it lives on `/library/*` and didn't see the API key's owner. New permissive route-level dep `resolve_api_key_cloud_owner` wired into `POST /library/files/{id}/slice` and `GET /slicer/presets` so cloud-token resolution works end-to-end for API-keyed requests with the cloud scope.

  - **Project cover photo thumbnail too small to recognise the print** ([#1155](https://github.com/maziggy/bambuddy/issues/1155) follow-up, reported by @smandon) — See *Improved* section above.

  - **SpoolBuddy kiosk screen-blank timeout setting was ignored after the first save** (reported by maziggy) — Picking a new "Screen Blank Timeout" in SpoolBuddy Settings → Display didn't change actual behaviour. Cause: blanking is driven by `swayidle`, started once at autostart with the timeout as a command-line argument. The Python daemon's `set_blank_timeout()` updated an in-memory variable that never reached `swayidle`. Fix extends the wake FIFO protocol with a `reload-timeout N` line; the watchdog kills and restarts swayidle when it sees one. Changes apply live — no kiosk restart required.

  - **SpoolBuddy kiosk screen never blanked while a load cell was producing noisy readings** — A noisy HX711 / load-cell mount that bounced the reported weight by ≥50 g around its midpoint kept the kiosk display permanently lit because every bounce fired `display.wake()`. Wake gate now requires the scale's `stable=True` flag (consecutive readings agree within 2 g over 1 s).

  - **SpoolBuddy SSH update fails with "permission denied for user spoolbuddy" after Bambuddy keypair rotation** — When Bambuddy rotates its SSH key, kiosks installed before the rotation are stuck on the old public key. Fix: the daemon now syncs the current public key over heartbeat and re-deploys it on the kiosk if it differs from what's currently in `authorized_keys`.

  - **External-camera frames returned as black on go2rtc and other MJPEG sources** ([#1177](https://github.com/maziggy/bambuddy/issues/1177)) — Some MJPEG sources emit a "warm-up" first frame that's all-black; we now skip it and return the second frame.

  - **Queue auto-dispatched the next print onto a fouled bed after an aborted or cancelled print** ([#1171](https://github.com/maziggy/bambuddy/issues/1171)) — The plate-clear gate was only raising for `failed/completed`; `aborted` and `cancelled` are also terminal states that need a clear before the next dispatch.

  - **Printer card always shows the first plate's thumbnail when printing a multi-plate 3MF** ([#1166](https://github.com/maziggy/bambuddy/issues/1166)) — Now resolves the actual plate index from `gcode_file` and pulls the matching thumbnail.

  - **AMS slot configuration intermittently fails to reach the printer after several configs in a row** ([#1164](https://github.com/maziggy/bambuddy/issues/1164)) — See *Improved* section above (paho inflight ceiling).

  - **`formatTimeOnly` tests failed under non-`:`-separator locales** ([#1213](https://github.com/maziggy/bambuddy/issues/1213), reported by @maugsburger) — `toLocaleTimeString` returns `02.30 pm` on `en_DK.UTF-8` and similar locales, so the hard-coded `:` in the test assertions broke contributor test runs. Switched to `\D+` (any non-digit) so the regex accepts any locale separator.

  ---

  **Contributors**

  Thank you to the contributors who helped make this release possible:

  - @Keybored02 — Forecast & Reorder Intelligence ([#1184](https://github.com/maziggy/bambuddy/pull/1184))
  - @netscout2001 — OIDC auto-created username + group claim handling ([#1173](https://github.com/maziggy/bambuddy/issues/1173) via [#1176](https://github.com/maziggy/bambuddy/pull/1176))
  - @maugsburger — Multi-colour filament fixes ([#1154](https://github.com/maziggy/bambuddy/issues/1154) follow-up), locale-agnostic test fix ([#1213](https://github.com/maziggy/bambuddy/issues/1213))
  - @EdwardChamberlain — Virtual Printer queue colour-match fix ([#1188](https://github.com/maziggy/bambuddy/issues/1188))

