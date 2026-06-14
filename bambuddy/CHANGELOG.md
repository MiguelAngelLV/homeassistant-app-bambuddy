## 0.2.4.7

**Bambuddy 0.2.4.7**

**⚠ Upgrade Notes — Read Before Updating**

0.2.4.7 is a fix-led patch release on the same 0.2.4 code base — no schema breaks beyond auto-migrated column additions (dialect-branched for SQLite and Postgres), no Docker entrypoint changes. The in-app Apply Update button in Settings → System → Updates works for Docker and for any native install already on 0.2.4.x.

Four behavior-change callouts to know about before you upgrade:

- Capture-Finish-Photo on the printer side is no longer force-toggled at dispatch (#1721, reported by @agrisci). The earlier workaround force-enabled the printer-side setting on every print to drive the finish-photo capture; the side effect was that intermediary progress notifications fell silent on A1, and the slicer's own Capture Finish Photo checkbox was silently overridden. Dispatch no longer flips the printer-side setting. Finish-photo capture is now driven by Bambuddy's own stage-22 pre-capture path with a FINISH-state fallback. If you previously unchecked Capture Finish Photo in the slicer to dodge the side effect, you can re-enable it. If you relied on the force-on to get finish photos without slicer changes, the FINISH-state fallback covers it — no UI change required.

- Slicer Bundle (.bbscfg) import removed (#1712, reported by @IndividualGhost1905). The legacy .bbscfg import path on the Slicer page is gone — it silently underdelivered on multi-tier preset matching and conflicted with the Orca Cloud / Bambu Cloud precedence work. Use the cloud sync (Orca Cloud or Bambu Cloud) or local imported presets instead. The SliceModal preset picker now reaches across Imported → Orca Cloud → Bambu Cloud → Standard with proper precedence and cross-tier dedup.

- Bambu Lab A2L support added (#1684). New "A2 Series" optgroup in the Add-Printer / Edit-Printer dropdowns and across the SpoolBuddy / inventory surface. The connection diagnostic, AMS slot routing, and camera path all auto-shape themselves to A2L's hardware (Wi-Fi-only, chamber-image protocol on port 6000 instead of RTSPS:322, single-extruder + cutter/plotter head with no deputy-slot routing). Existing X1 / H2 / P1 / P2 / A1 surfaces are unchanged. If you're on an A2L: re-test Configure AMS Slot — the picker now filters profiles to A2L-compatible filaments (#1623 fix below).

- Windows installer pipeline overhauled. The installer is self-versioning (filename matches the release tag, plus an unversioned alias on stable / beta), ships its own NSSM binary instead of fetching at build time, stops the Bambuddy service before file copy on upgrade, and bootstraps setuptools + wheel into its embedded Python so the post-install dependency install doesn't fail on a clean machine. Existing Windows installs upgrade in place via the Service / Update entries on the new installer; first-time installs no longer depend on a network trip to a flaky NSSM mirror.

Make a backup before upgrading via Settings → Backup → Create Backup. Native install with update.sh snapshots the database automatically and rolls back on failure.
Docker and fully-manual paths don't.

**Docker**

docker compose pull
docker compose up -d

docker-compose.yml doesn't need refreshing for 0.2.4.7. (If you map the VP passive FTP port range or run the slicer-API sidecar, the 0.2.4.6 upgrade notes still apply —
nothing new in 0.2.4.7.)

**Native install — recommended path**

sudo BRANCH=main /opt/bambuddy/install/update.sh

Snapshots the database first and rolls back on failure.

**Native install — manual path**

sudo systemctl stop bambuddy
cd /opt/bambuddy
sudo -u bambuddy git fetch --prune --tags --force origin
sudo -u bambuddy git checkout main
sudo -u bambuddy git reset --hard origin/main
sudo /opt/bambuddy/venv/bin/pip install -r requirements.txt
sudo systemctl start bambuddy

requirements.txt bumps the aiohttp floor to >=3.14.0 this release to clear two upstream advisories. No code change inside Bambuddy — only the floor moves so fresh installs and CI pick up the fix.

**Windows install**

0.2.4.7 ships a rebuilt Windows installer with the pipeline improvements from the Upgrade Notes. Download bambuddy-0.2.4.7-windows-x64-setup.exe from this release page (or the unversioned bambuddy-windows-x64-setup.exe alias for an always-latest link). Existing 0.2.4.5 / 0.2.4.6 installs upgrade in place — the installer stops the Bambuddy service, swaps files, restarts the service, and preserves your data directory.

---
**Highlights**

0.2.4.7 is a heavy fix-cycle release with one big add — Bambu Lab A2L support (#1684) — and a long tail of contributor-credited fixes across the Virtual Printer, Slicer, Print Queue, and connection-diagnostic surfaces.

The Virtual Printer surface got another full sweep driven by #1622 telemetry. The bridge cache now accumulates push_status per-field instead of replacing fields on each push (round 4), overlays incoming dict-shaped fields onto the cache instead of wholesale replacement (round 5, @shaddowlink), and applies the tray_exist_bits empty-slot cleanup to the slicer-facing cache (#1726, @needo37 with full code-level analysis). Net effect: BambuStudio's Device tab no longer greys out between pushalls, and Sync no longer sees phantom-loaded filaments in empty AMS slots. Three env-flagged debug paths landed in the same cycle (wire-payload dump, command-flow trace, bridge-synthesised-reply trace) and stay silent in normal operation.

Slicer (SliceModal) is the second-largest theme: full preset-lookup precedence rework + cross-tier dedup + signed-out banner + AMS slot badges (#1712, @IndividualGhost1905). Follow-ups in the same train fix the Orca Cloud / Bambu Cloud preset resolver to pin type and from to CLI-accepted values for the headless slicer-api sidecar, and the Library G-code preview now correctly renders sidecar-sliced .gcode.3mf rows as G-code instead of returning raw ZIP bytes as text/plain
(#1709, root cause + fix from @yanglei1980).

Print Queue + Archive polish: multi-plate Send All now enqueues one queue item per plate instead of one item per click, archive delete cascades to remove related queue items instead of leaving "cancelled" rows behind, the force-color-match checkbox is no longer missing when scheduling against a specific printer (#1717, @SamNuttall), and the filament-override panel surfaces Bambu Studio's sub-brand colour name instead of the raw 3MF base material (#1718, @SamNuttall). Multi-color filament rows in the Print Log render one swatch per colour instead of a single barely-visible gray dot (#1731 part 1, @IndividualGhost1905). Telegram (and other image-bearing) finish notifications on a reprint-from-archive now ship the new run's finish photo instead of the original print's (#1707, @kycrna).

Windows + restore reliability. Beyond the installer pipeline overhaul: /api/local-backup/status no longer 500s on ZoneInfoNotFoundError: 'No time zone found with key UTC' from a missing zoneinfo DB on the Windows installer (a stdlib UTC fallback covers it). Network-interface enumeration on Windows now uses psutil instead of the Linux-only path. Restore pauses timer-based DB writers before the swap, fixing a Postgres deadlock cascade observed during multi-printer restores.

---
**New Features**

- Bambu Lab A2L support (#1684). New "A2 Series" optgroup in Add-Printer / Edit-Printer dropdowns, with full capability resolution from BambuStudio's machine profile cross-checked against Bambu's official A2L specs page: linear rail, single FDM extruder + integrated cutter/plotter head, Wi-Fi-only (no Ethernet), Low-Rate-Kamera on the chamber-image protocol (port 6000, not RTSPS:322), no heated chamber. The dual-tool-head capability is correctly distinguished from dual-filament extrusion — A2L is not in DUAL_NOZZLE_MODELS, so AMS does not route to a deputy slot (firmware would reject with 07FF_8012). Registry updates touch utils/printer_models.py, firmware_check.py (wiki path follows the established /en/a2l/manual/a2l-firmware-release-history pattern; existing 404 handling makes this safe to ship before Bambu publishes the page), virtual_printer/manager.py (serial prefix 26A19), virtual_printer/mqtt_server.py, PrintersPage.tsx, and  SpoolBuddyAmsPage.tsx. Camera and dual-nozzle code paths need no edits — supports_rtsp() correctly falls through to chamber-image and is_dual_nozzle_model() correctly returns False.

- Re-print / Schedule modal allows cross-extruder AMS slot picks on dual-nozzle (#1722, reported by @privatsturm). The picker previously gated AMS slot choices to the same extruder as the source plate's filament group. On dual-nozzle hardware that's wrong — operators routinely re-route a left-side filament to a right-side AMS slot when the left bank is empty or busy. Fix removes the gate; the MQTT layer's existing dual-nozzle routing handles the rest.

- Support bundle now includes redacted cached push_status per connected printer. The raw push_status payload is the single highest-signal artefact for diagnosing slicer-facing VP issues. The new bundle entry walks the cached payload and scrubs net.info[*].ip plus the documented privacy-sensitive fields before serialising — the live state.raw_data is never mutated. Drops directly into the existing bundle archive alongside the existing log + config dumps.

- One-shot device identification probe for unknown printer models. When Bambuddy sees a serial prefix it doesn't recognise on the MQTT bus, it now fires a single push_all-only probe to surface the model code, instead of either silently dropping the connection or polluting the log with repeated unknown-model warnings. Aids future model rollouts (next H2 / A2 / X2 family variant).

---
**Changes**

- VP MQTT bridge cache shape rework (#1622 rounds 4 + 5). The cache now accumulates push_status per-field instead of replacing fields on each incoming push (round 4, fixes Device-tab greying), and overlays incoming dict-shaped fields onto the cache instead of replacing the whole sub-object (round 5, @shaddowlink, fixes vt_tray going "invalid" right after a slicer filament pick). Field accumulation is bounded by the field allowlist documented in the VP regression matrix.

- SliceModal preset-lookup precedence + cross-tier dedup + signed-out banner + AMS slot badges (#1712, reported by @IndividualGhost1905). The 4-tier picker (Imported → Orca Cloud → Bambu Cloud → Standard) now enforces the documented precedence end-to-end with explicit dedup on key collisions, surfaces a signed-out banner per cloud
tier instead of silently empty lists, and renders AMS slot badges next to the preset row so operators can see at a glance which slot the preset would land in. The legacy .bbscfg import path was dropped in the same train — see Upgrade Notes.

- Print-modal "off" toggles for flow_cali and nozzle_offset_cali now actually suppress the calibration stage. Live-tested on H2D 01.x. The previous behaviour wrote the toggle value into the project_file payload but the firmware still ran the stage; fix routes through the existing dual-nozzle gate and the calibration-suppress field at the MQTT layer.

- Support-bundle log noise demoted (#1721 adjacent, observed on the reporter's A1). Benign "not connected" and "may linger" warnings are now INFO-level so the bundle log dumps surface real problems instead of background heartbeat noise.

- aiohttp pinned to >=3.14.0 in requirements.txt for two upstream advisories. Bambuddy's aiohttp usage was unaffected by the underlying issues, but the floor moves so
fresh installs and CI pick up the fixed runtime.

---
**Fixed**

**UI / rendering**

- Print Log multi-color filament rows now render one swatch per colour instead of a single barely-visible gray dot (#1731 part 1, reported by @IndividualGhost1905). The renderer was averaging the multi-colour RGBA into a single greyscale fallback; the fix splits on the slicer's pipe-separated colour list and renders a small swatch row.

- Stats page Failure Analysis widget now renders translated failure reasons instead of raw camelCase keys (#1687 follow-up, reported by @IndividualGhost1905). The GET serialiser was dropping the canonical failure_reason mapping; the fix re-applies the same vocabulary the Archive Edit modal uses.

- System page boot time no longer renders with a doubled timezone offset (#1690 follow-up, reported by @IndividualGhost1905). The recorder was emitting a tz-naive datetime that the frontend then localised on top; both boot_time and generated_at now ship as tz-aware UTC.

- AMS slot card stays in sync with the new spool's preset name after RFID auto-assigns a new spool. Reporter observed H2D-1 / AMS-B3 / PLA-CF rendering as "Bambu PLA Silk+" until manual refresh. The fix invalidates the AMS slot's cached preset-name lookup on the same event that updates the spool binding.

**Virtual printer**

- Empty AMS slots no longer forwarded as phantom loaded filaments to BambuStudio Sync (#1726, reported with full code-level analysis by @needo37). The tray_exist_bits empty-slot cleanup now applies to the slicer-facing cache in addition to the live state — Sync sees the same empty-slot picture the real printer reports.

- The vt_tray external-spool object no longer goes "invalid" immediately after a slicer filament pick (#1622 round 5, reported by @shaddowlink). The cache was replacing the whole vt_tray sub-object on each push, dropping the slicer's just-picked state; the overlay rework preserves field-level updates.

- VP cache no longer drains capability / lifecycle fields between pushalls, which had greyed out Device-tab UIs (#1622 round 4, reported by @shaddowlink). The per-field accumulation fix from the Changes section.

**Print queue + dispatch + archive**

- Multi-plate Send All now enqueues one queue item per plate instead of one item per click. The route was accepting the plate count from the UI but the queue-insert was firing once per request; fix iterates the plate set on the server side.

- Archive delete now removes related queue items instead of leaving "cancelled" rows behind. The cascade was already declared at the schema level but the DELETE route was using a soft-delete path that bypassed it; fix routes the archive delete through the same cascading path the bulk-delete already used.

- Finish-photo capture: dispatch no longer force-toggles the printer-side Capture Finish Photo setting (#1721, reported by @agrisci). See Upgrade Notes for the full behaviour change.

- Telegram (and other image-bearing) finish notification on a reprint-from-archive no longer ships the original print's finish photo instead of the new run's (#1707, reported by @kycrna). The notification was reading the finish-photo path from the source archive instead of the new archive row.

- Print Queue filament-override panel now shows Bambu Studio's sub-brand colour name instead of the raw 3MF base material (#1718, reported by @SamNuttall). The panel was reading directly from the 3MF metadata instead of the resolved Bambu profile.

- Force-color-match checkbox no longer missing when scheduling against a specific printer (#1717, reported by @SamNuttall). The Charcoal-style label fix from earlier in the queue cycle was missing on the Specific-Printer panel; this also extends to a related label-fix follow-up (#1718 round 3).

**Inventory / AMS / connection diagnostic**

- Configure AMS Slot picker now filters filament profiles to the printer model (#1623, reported by @shaddowlink). The list was showing every imported profile regardless of printer compatibility; the fix routes the picker through the same compatibility filter the SliceModal uses.

- Connection diagnostic no longer flags external_storage: fail on A1 / A1 Mini (#1703, reported by @MartinNYHC). A1 and A1 Mini have no MicroSD slot; the diagnostic now skips the check for those models with a clear "n/a — printer has no SD slot" surface.

- A1 / A1 Mini internal-code map was swapped in PRINTER_MODEL_ID_MAP (surfaced while scoping A2L support, #1684). The swap had no user-visible symptoms but corrupted the printer-model resolver for any code path that round-tripped through the ID map.

**Slicer / library**

- Library G-code preview returned raw ZIP bytes as text/plain for sidecar-sliced .gcode.3mf rows (#1709, root cause + fix from @yanglei1980). The preview route was branching on filename suffix but the sidecar produces .gcode.3mf — a ZIP wrapper around a .gcode payload. Fix extracts the inner .gcode before serving and pins the Content-Type to text/x-gcode.

- Cloud + Orca Cloud preset resolver now pins type and from to CLI-accepted values (#1712 follow-up, reported by maziggy on a Mecha Mewtwo slice). The resolver was passing the user-tier-as-displayed string ("orca_cloud") to the headless slicer-api CLI, which only accepts a fixed enum; fix maps display tiers to CLI tiers explicitly.

**Windows / install / restore**

- /api/local-backup/status no longer 500s on ZoneInfoNotFoundError: 'No time zone found with key UTC' (from a user's log on the Windows installer). Some Windows hosts ship without the IANA zoneinfo database; the fix falls back to the stdlib UTC implementation when the IANA lookup raises.

- Network-interface enumeration on Windows now uses psutil instead of the Linux-only socket path. Affected the Add-Printer custom-subnet picker and any path that listed local interfaces.

- In-app updater now routes every git step through app_dir for separate-mount installs (#1715, reported by @francescocozzi). On installs where /opt/bambuddy/data is a separate mount from /opt/bambuddy, the updater's git commands ran from the data dir and failed silently on the .git lookup; fix explicitly passes -C app_dir.

- Restore now pauses timer-based DB writers before swap, fixing a Postgres deadlock cascade. Multi-printer restores were triggering concurrent timer writers against the same connection pool as the restore transaction; fix gates the timer loop on a "restore in progress" flag.

