# rustdar — Full Codebase Audit

**Date:** 2026-07-27 · **Scope:** every tracked file in the workspace (all 10 crates, Android/iOS shells, web PWA, CI, docs, build config) · **Method:** 11 parallel line-by-line review passes (one or two per package, plus a dedicated cross-package duplication pass and a repo/CI/docs pass), followed by lead verification of every high-severity claim against source.

**Verification pass (same day):** every finding in this document was subsequently re-verified by an independent adversarial pass (11 verifiers, one per section, instructed to refute each claim by tracing the actual code, callers, and dependency sources). Result: **all high and medium findings survived**; 9 findings were refuted or resolved and have been removed (a claimed missing length-check that grib 0.16.0 actually enforces, a viewport-wrapping hazard refuted against walkers 0.56 source, a convexity-documentation gap that already exists in the trait, a doc abbreviation confirmed correct per SPC SCN 26-11, and five cosmetic nits that don't exist in current source); ~110 line references were corrected against the working tree (the original pass drifted up to ~140 lines in a few files); several previously "unverified" flags were resolved with hard evidence and are now stated as confirmed (Alaska/Guam antimeridian exposure, keyboard-slider snap-back, GLM fetch-per-frame slider, the SiRF USB VID being wrong, serialport exclusive-open, `retarget_renders` phase gap).

---

## Remediation status (2026-07-27, same day)

Fixes were applied by parallel agents in isolated git worktrees, each diff reviewed before merge. **Merged to `main` (11 commits, workspace tests green):**

| Item | Status | Commit |
|---|---|---|
| P0-1 nexrad-level3 XDR decode panics (+ 3 offset-overflow sites) | ✅ fixed, 7 regression tests | `87ce5e5` |
| P0-2 GPS reader could never be stopped (+ 30 s UI freeze on GPS enable) | ✅ fixed, 3 tests pinning the stop states | `97fb97e` |
| P0-6 ZDR renders black below −2 dB (+ non-finite palette inputs) | ✅ fixed, 2 tests | `b0c0bee` |
| P0-7 Deselected GLM satellite kept rendering | ✅ fixed, 1 test | `535be6f` (merge) |
| SPC `parse_hex_color` panic on non-ASCII feed data | ✅ fixed, 1 test | `535be6f` |
| One malformed SPC feature blanked the whole outlook | ✅ fixed, 2 tests | `535be6f` |
| CIG hatch even-odd parity bug | ✅ fixed by subtraction mask, 3 tests | `535be6f` |
| BKN station-model glyph overpainting the map | ✅ fixed (inscribed circular segment), 1 test | `535be6f` |
| SW deploy probe blind to non-wasm deploys | ✅ fixed (probes wasm + index), 2 tests | `7482b2c` |
| SW rollback install could delete a pinned generation | ✅ fixed, 1 test | `73b431e` |
| `[profile.dev] strip = true` vs debuggability comment | ✅ fixed | `cd857c9` |
| features.md / copilot-instructions.md / README badge drift | ✅ fixed, claims grep-verified | `ded0582` |
| CI: fork PRs red before running; PR-event autocommit on detached ref | ✅ fixed | `2f9d4fa` |
| **P0-5** GOES buckets, NWS alerts URL and Level II bucket bypassing `DataSources` | ✅ fixed, 3 tests; hardcodes deleted | `6b3876c` (merge) |
| **P0-3** Android FINE-only permission (+ `JAVA` per-Activity staleness, no active location updates, `serial` on Android, keystore `storeFile`, CompassHelper leak + landscape heading) | ✅ all 7 fixed | `24e3612` (merge) |
| egui: polygon-hole hit-testing, unseeded new panes, Refresh site, stale hover readout, full-vector pane scans | ✅ all 5 fixed, 5 mutation-verified tests | `4f3e775` (merge) |
| **P0-4** Auto-poll compared every live site against the *active* pane's scan time | ✅ fixed (per-site lookup keyed on `scan_info.site`), 3 tests | `a8809eb` |
| Global render generation discarding other sites' in-flight renders | ✅ fixed (per-pane cancellation flags; global counter now only for full resets), 5 tests | `806551f` |
| Loop wedged in `Rendering` on empty listing / all-failed frames | ✅ fixed (loop switched off in both cases), 4 tests | `648fdd1` |
| Overlay render marks stranded on early-return paths | ✅ fixed | `99a69fc` |
| DST spring-forward navigation jumping to *now* | ✅ fixed, 4 tests vs a fixed-zone double | `ce93b46` |
| `JumpToLive` fetching an empty site code; duplicated `SwitchRadarSite` loops | ✅ fixed | `8526b39`, `c324780` |
| Deferred exit skipping `needs_process_exit` (Android's primary quit route) | ✅ fixed | `a74e1d7` |
| Desktop dark theme never reaching overlay rasterization | ✅ fixed (one `adopt_theme` funnel), tests | `564e9eb` |
| Unbounded `scan_data` (one decoded Level II volume per site ever visited) | ✅ fixed (per-frame sweep retaining shown sites), 2 tests | `66b8d57` |
| Scan + Level III channels draining one message per frame | ✅ fixed (`while let`), 2 tests | `7e2d5e8`, `e2fe08d` |
| Stale-discard leaving the spinner stuck | ✅ fixed (discard ends the wait it belonged to) | `03c1608` |
| Results applied after `gui.ui()` built the frame | ✅ fixed (`gui.ui()` moved last), source-probe test | `0b80a01` |
| `loop_speed_fps` panicking `Duration::from_secs_f32` | ✅ fixed (clamp + named constants), 2 tests | `cab502f` |
| Escape hitting the back-out funnel while egui holds keyboard focus | ✅ fixed | `e30339a` |
| Dead `scale_factor` state | ✅ removed | `3e683d6` |
| **P0-9** Autosave wake-up saved nothing and hot-spun the event loop | ✅ fixed (save moved into `about_to_wait`; the idle path returns the loop to `ControlFlow::Wait`), 5 tests | `PLACEHOLDER` |

**All P0s and mediums in this document are now fixed**, with two deliberate exceptions:

- **P0-8** Burned keystore password in public history — closed as accepted risk by the repo owner: the key is known-burned and future real releases will not use it. No code change.
- **`handle_navigate_one_scan` not updating `radar.config.timestamp`** (the time picker drifting after stepping scan-by-scan) — the only finding from the frontend cluster left unfixed. It spans two agents' file ownership: the adjacent scan's timestamp is not known until the response lands in `app.rs::poll_data_channels`, so the patch belongs there, gated on `manual_nav_pending`. Small and well-understood; not yet applied.

Two adjacent items surfaced during remediation and were deliberately left alone:

- `latest_cached_scans` (`app.rs`) has the same unbounded shape `scan_data` had — one `Arc<Scan>` per site, removed only by `handle_jump_to_live`. Smaller in practice (only sites auto-polled while every pane on them is historic) and outside the original finding's wording; `evict_unshown_scans` is one line from covering it.
- `rustdar-radar/examples/render_product.rs` is an untracked debugging utility written during this work (renders one product tilt to a PPM for eyeballing against reference viewers). It carries a `needless_range_loop` warning, so it would need a fix before being committed.

**Verification caveats on the Android work** (`24e3612`): the host had no Gradle/AGP, so no APK was built and R8/lint never ran. What *was* verified: `cargo ndk check` + `clippy -D warnings` for `aarch64-linux-android` (covers `android_main` and all target-gated JNI), and JDK-17 `javac -Xlint:all` of all three `com.rustdar` sources against android-36's `android.jar`. On-device behaviour — the permission dialog, live location updates, landscape compass heading — is unexercised.

Formatting drift is resolved: the repo was reformatted at `462b96d` and `cargo fmt --check` now reports **0 diffs**. Adding a `cargo fmt --check` CI gate to keep it that way is still outstanding.

Post-remediation gate on `main`: `cargo test --workspace` green (frontend 120 → 148 tests, overlays 292 → 304, egui 206 → 211, nexrad-level3 11 → 18, radar 98 → 100, service worker 45 → 48), `cargo clippy --workspace` zero warnings, `cargo fmt --check` zero diffs.

Note on history: `main` was rebased onto `b53bdc4` partway through this work, which replayed each fix commit individually and so discarded the earlier merge resolutions. Seven conflicts were re-resolved in favour of the fix side and re-verified (the load-bearing check being `glm/fetch.rs`, which two agents had edited — both the satellite filter and the `DataSources` URL survive). All 26 commits above the base are GPG-signed.

## Mechanical baseline

| Check | Result |
|---|---|
| `cargo clippy --workspace --all-targets` | Clean (enforced by CI) |
| `cargo test --workspace` | Green — ~754 passed, 0 failed, 25 ignored (live-network tests) |
| `cargo fmt --check` | **1,056 diffs across 91 files.** No `rustfmt.toml`, no fmt check in any CI workflow — formatting is unenforced and has drifted everywhere. Fix: run `cargo fmt` once, add `cargo fmt --check` to CI. |
| Repo hygiene | Clean: no secrets/binaries/artifacts tracked; `rustdar.jks`, `jniLibs/*.so`, `coverage/` all correctly ignored. One historical wound (see P0-8). Also: `.claude/worktrees/` holds ~2,900 stale `.rs` files on disk (ignored) — worth a periodic cleanup. |

Totals after the verification pass: **8 high**, **~35 medium**, **~68 low**, **~84 nit** findings (329 severity-tagged items). Overall code quality is unusually strong — heavy load-bearing documentation, mutation-resistant tests, disciplined pinning — which is exactly why the dominant defect classes are *drifted docs*, *duplicated sources of truth*, and *the untested halves* of otherwise well-tested crates.

---

## P0 — Fix now (crashes, wrong data on screen, broken features)

1. **[nexrad-level3] Two remotely-reachable panics in the XDR radial decoder** — `decode/radial.rs:393` reads the radial count as `i32` and casts to `usize`: a negative value panics `Vec::with_capacity` ("capacity overflow"). `radial.rs:424-437`: a negative array length makes `arr_len * 4` wrap in release so `data_end = o - 4` passes the bounds check and the slice `data[o..data_end]` panics — in **both** build profiles, from malformed downloaded bytes, in a crate that `#![deny(clippy::unwrap_used)]`s precisely to prevent this. *Fix:* validate via `u32::try_from` + `checked_mul`/`checked_add` + a sanity cap; a shared `read_len(data, o, max)` helper closes the whole class (a third instance: `decode/mod.rs:44` offset math can overflow on wasm32).

2. **[rustdar-gps] The serial GPS reader can never be stopped** — `serial.rs:109` holds `_stop_signal: mpsc::Sender<()>` that nothing ever sends on, and every check is `stop_rx.try_recv().is_ok()` (lines 153/175/189/222), which is false both while the sender is alive (`Err(Empty)`) *and after it is dropped* (`Err(Disconnected)`). Dropping `SerialGpsReader` — the documented stop, relied on by `rustdar-platform/src/platform.rs:134` — is a no-op. A quiet receiver leaks the thread holding the port open with `TIOCEXCL`, so stop→start on the same port fails to reopen forever while the UI logs success. *Fix (one line):* `if !matches!(stop_rx.try_recv(), Err(mpsc::TryRecvError::Empty))` — or add an explicit `Drop` that sends `()`.

3. **[rustdar-android] Location permission can never be granted on Android 12+** — `src/lib.rs:153-162` requests `ACCESS_FINE_LOCATION` alone in a 1-element array; API 31+ silently discards FINE-without-COARSE requests (no dialog), and the bounded retry (`MAX_PERMISSION_REQUESTS = 2`) burns both attempts on discarded requests. GPS is dead for the life of the install unless granted from Settings. *Fix:* request `[FINE, COARSE]` (both already declared in the manifest). Related medium: `lib.rs:330-345` only polls `getLastKnownLocation` — if no other app requests location, it returns null forever; register a real `LocationListener`.

4. **[rustdar-frontend] Auto-poll compares every site against the active pane's timestamp** — `app_fetch.rs:266-308` uses `self.gui.get_scan_info()` (the *active pane's* scan) as "current" for one `CheckForNewScans` per *unique live site*. With panes on two sites: updates for the non-active site are skipped round after round, or (active pane older) the other site's full Level II scan is re-downloaded and fully re-applied **every poll interval** despite being unchanged. *Fix:* track latest-applied timestamp per site and compare per-site.

5. **[cross-package] The GOES bucket names, NWS alerts URL, and Level II bucket bypass `DataSources`** — `sources.rs` promises "every network origin, declared in one place", and the Android network-security-config test and web service-worker never-cache list are *derived from it* — but GLM fetches via `GlmSatellite::bucket()` hardcodes (`glm/mod.rs:31-37`), NWS via a literal (`nws/fetch.rs:6`), and Level II via `ARCHIVE_BUCKET` (`archive.rs:30`, whose doc cites a pinning test that **does not exist**). The declared fields have zero production consumers, so the next satellite rotation would leave the security/caching validations checking stale hosts while real traffic goes undeclared. *Fix:* route all three through `DataSources`; generalize SPC's "no URL bypasses the declared origin" test workspace-wide.

6. **[rustdar-radar] ZDR renders black below −2.0 dB** — `palette.rs:20-26`: gradient scales whose first stop is `f32::NEG_INFINITY` compute `t = inf/inf = NaN`, and NaN casts to `0u8` — semi-transparent black instead of the intended gray, reachable in production (L2 ZDR spans ≈ −8..+8 dB). *Fix:* skip the gradient branch when `!last_threshold.is_finite()`, or use a finite floor (−8.0). While in there: add one `is_finite` guard at the top of `get_color_for_value` (NaN currently paints velocity dark-green; it's `pub` and called from rustdar-egui).

7. **[rustdar-overlays] Deselecting a GLM satellite doesn't stop its lightning rendering** — `glm/fetch.rs:208-214` (`flashes_in_window`) filters by time only, never by the `satellites` argument, and the handler clears the cache on level toggles but not satellite changes — "Both"→"East" keeps drawing GOES-West flashes for up to 30 minutes. *Fix:* add `&& satellites.contains(&f.satellite)` to the filter, or clear the cache on satellite change.

8. **[rustdar-frontend] The config autosave's wake-up never saves, and burns a core doing it** (**P0-9**, found 2026-08-09 — after this sweep, in code written after it; the IDs skip 8 because P0-8 is the keystore item below) — `app.rs:1585-1600` (`schedule_autosave_wakeup`, called from `about_to_wait` :1921) sets `ControlFlow::WaitUntil(last_check + AUTOSAVE_INTERVAL)` so an unwritten change gets a chance to be persisted. A `WaitUntil` expiry dispatches `new_events` and `about_to_wait` and **nothing else** (winit 0.30.13 `x11/mod.rs:477-505`; `ResumeTimeReached` never implies a redraw), but `autosave_config` (:1486) is reachable only from `handle_redraw` (:655), i.e. only from the `RedrawRequested` arm (:1970). So the wake-up cannot reach the save it exists for, and `autosave.touched` — cleared only inside `autosave_config` (:1497) — stays set. Once the deadline is behind the clock the re-arm computes a saturated-to-zero delay, `wait_duration(0)` is `WaitUntil(now)`, and it expires on the spot. Two separate defects, and both survive fixing only the other: `set_control_flow` is sticky, so a `!touched` early return that leaves an expired `WaitUntil` in place spins just as hard. Measured against a real winit loop on X11: **~164,000 iterations/s at 99% of one core, 0 `RedrawRequested` delivered, 0 configs written** — and with the save wired up but the early return untouched, ~162,000/s. The user-visible half is the one the autosave was added for: pan the map, walk away, lose the pan (web has no other save point at all — no `beforeunload` handler exists, by design). *Fix:* run `autosave_config(false)` in `about_to_wait`, where the timer's own dispatch actually lands, and return `ControlFlow::Wait` — the loop's resting state — whenever nothing is owed. Not by requesting a redraw: that spends a whole frame to write a few hundred bytes of JSON on a timer whose premise is that the app is asleep, and it still needs the `Wait` reset.

## P1 — Should fix before feature work (worst of the mediums)

- **[egui] Stale hover readout in the status bar** — `pane.hover_value` is only ever cleared inside the radar-metadata path (`ui_map_pane.rs:110-171`), and `render_hover_info` (`ui_chrome.rs:539-553`) scans the *full* panes vector including hidden panes — two independent ways a dead lat/lon/value readout persists indefinitely. Fix both: clear at top of `render_pane_map_content`; iterate the visible `panes()` slice (same sweep fixes `any_city_labels` at `ui_map.rs:33-36` and the unclamped indexing in `render_map`/`sync_viewports`).
- **[egui] Refresh fetches the wrong site with per-pane sites** — both the status-bar refresh (`ui_chrome.rs:224-230`) and menu "Refresh Radar" (`ui_menu.rs:249-251`) send the *global* `radar.config.site`, not the active pane's; `check_auto_polls` already demonstrates the correct substitution.
- **[egui] Panes added via the pane-count picker render no overlays with Sync Layers off** (`ui.rs:869-877`) — new panes get empty overlay maps and nothing re-initializes them; masked only by the default `sync_layers = true`.
- **[egui] Polygon holes ignored in overlay hit-testing** (`overlay_cache.rs:362-383`) **and in rasterization** (`rasterize.rs:749-768`) — clicks inside a cut-out open the wrong popup, and the hole area *paints* as alerted. The rasterize fix is one line (all rings + `FillRule::EvenOdd`, as hatch.rs already does).
- **[overlays] One malformed SPC feature blanks the whole outlook** (`spc/outlook.rs:187-191`) — contrast NWS (skips bad features) and METAR (counts rejections). Also `spc/colors.rs:2-11` can *panic* on a non-ASCII hex color from the feed (byte slicing), wedging the overlay in "fetching".
- **[overlays] Station-model BKN glyph overpaints the map** (`station_model.rs:202-219`) — the background-colored rect's corners extend beyond the circle, leaving gray blotches over radar echoes; and barb quantization under-plots winds ending in 8/9 kt by 5 kt (`:307-313`).
- **[overlays] CIG hatch parity bug** (`hatch.rs:165-192`) — the even-odd exclusion mask re-fills doubly-nested regions (CIG1∧CIG2∧CIG3 → odd → hatched again), the exact case commit b7f2ebd meant to fix. Needs a check whether live CIG polygons nest; fix by excluding only the next level, or Winding + mask intersection.
- **[frontend] Wakeup/drain gaps** — `poll_data_channels` and `poll_level3_results` drain **one** message per frame (every sibling uses `while let`), results are applied *after* `gui.ui()` builds the frame with nothing re-arming a redraw, and a failed/empty loop listing wedges the loop permanently in `Rendering` with a comment claiming an error state that doesn't exist (`app_fetch.rs:587-595`). One policy — drain everything, before the UI pass, and request a redraw when anything was applied — closes the class.
- **[frontend] Unbounded `scan_data`** (`app.rs:166`) — one full decoded Level II scan (tens of MB) retained per site ever visited, no eviction anywhere; the crate otherwise budgets memory carefully. Evict sites no pane shows.
- **[frontend] Global render generation** (`render_dispatch.rs:371`) — every scan arrival for site A discards site B's in-flight 2048² renders, recurring every poll in multi-site layouts. Make it per-site like `fetch_generations`.
- **[frontend] Desktop dark theme never reaches overlay rasterization** (`app.rs:176` + `app_render.rs:100-110`) — `cached_dark_theme` is only written in the `None` arm, so on Windows/macOS overlays rasterize light-themed under a dark UI; the deferred menu-Exit replay also drops the `needs_process_exit` branch Android depends on (`app.rs:966-971`).
- **[android] Second-Activity staleness** — `JAVA` OnceLock (`lib.rs:71`) and `CompassHelper.register` (`:64-124`) are both write-once but their caller runs once per Activity; the same file documents this model for `EVENT_LOOP_PROXY` and handles it there. Also `CompassHelper` never remaps for display rotation (heading wrong by ±90° in landscape) and reports magnetic, not true, north.
- **[android] `features = ["serial"]` on rustdar-gps** (`Cargo.toml:64`) re-enables the serialport stack for the whole Android graph, defeating rustdar-platform's deliberate mobile exclusion. Drop it.
- **[android] Keystore path resolution contradiction** (`app/build.gradle.kts:252-254` vs `keystore.properties.example:54-55`) — `file()` always returns absolute so the `rootProject.file` fallback is dead; following the example's instructions verbatim fails at signing.
- **[web] Service-worker deploy probe watches only the wasm file** (`sw.js:100-101`) — an HTML/manifest/icon-only deploy is never detected and the old shell is served indefinitely; the test suite's deploy model (all ETags change together) structurally can't see this. Also `installShell`'s failure path can delete a pre-existing in-use cache generation on a rollback deploy (`sw.js:391-421`).
- **[gps] `detect_baud` blocks the UI thread up to ~30 s** (`serial.rs:115-127`) — runs synchronously in the GUI action handler; move detection into the spawned thread.
- **[workspace] `[profile.dev] strip = true`** contradicts the adjacent "preserve debuggability" comment — dev builds generate debuginfo and throw it away.
- **[CI] Fork PRs fail before running anything** (test.yaml/clippy.yaml unconditionally mint an App token from secrets), clippy's PR-event run asks git-auto-commit to commit from a detached merge ref, and build.yaml/clippy.yaml double-run every same-repo PR (~11-row matrix ×2). Decide the fork posture, gate the token step, restrict `push:` to main.
- **[docs] features.md contradicts shipped reality** (HRRR ❌, GLM ❌, Web app ❌ — all shipped), **copilot-instructions.md is badly stale** ("seven crates" of 10, "No web target", rc.5 pins, phantom filenames) and self-mandates being current while steering automated edits; README's Release badge points at a workflow that doesn't exist.

## P2 highlights (fuller lists in the per-package sections)

- **Dead code cluster** from the handler-registry refactor: `LayerManager`/`LayerState` (~120 lines, zero callers), `spc::fetch::available_products`, `SpcOutlook.valid/expire`, `Visibility::parse`, five never-constructed error variants in nexrad-level3, `RasterPacket` stub, `scale_factor` dead state, `RUSTDAR_TLS_PROBE` env no one reads. One deletion sweep.
- **Duplication worth extracting**: S3 ListObjectsV2 client ×2 (radar's capped/tested one vs GLM's hand-rolled uncapped one — publish radar's and delete roxmltree); `lat_rad_to_mercator_y` ×3 + 85.05 clamp ×2 (publish radar's); unit factors (0.514444, 1.94384, 2.23694, 3.28084, °F formulas, inHg) re-hardcoded across 5 crates while `UserPreferences::temperature` is a **dead setting** (offered in the UI, zero call sites — every temp display hardcodes °F); the ~130-line color-scale renderer pair in ui_map_pane; the "Updated Xs ago" block ×6 across overlay handlers; HRRR's retry scaffold ×2; `strip_closing_dup` ×2; the SPC day→product table ×3 (two dead).
- **Error-policy inconsistencies**: overlays uses `Result<_, String>` everywhere vs radar's thiserror enums; storm-reports total outage returns `Ok(vec![])` while METAR errors; nexrad-level3 swallows bz2 failure at `debug!` level and parses the compressed tail as symbology.
- **Doc drift as a defect class**: stale egui 0.34.1 line citations, a "foreground layer" comment after a deliberate switch to `Order::Background`, the phantom archive test, N0S 342-vs-294, `%Z` doesn't render "CDT" (chrono FixedOffset prints "+05:00"), "thirty seconds" vs 45.
- **Missing tests where the bugs are**: rustdar-units (0 tests — constants verified correct by hand this audit), rustdar-gps (0 tests), nexrad-level3's decode layer (0 in-crate tests — both P0 panics live there), SW_VERSION migration, SW stale-while-revalidate end-to-end.

---

*Detailed findings follow, one section per audit scope. Line numbers were verified at audit time against the working tree at commit c20d531.*

---

# Audit findings: rustdar-radar

## archive.rs
- **medium** — :27-30 — Doc claims test `archive_bucket_matches_the_declared_origin` pins ARCHIVE_BUCKET against DataSources; no such test exists anywhere (grep verified). Two unpinned copies of the bucket name. Fix: add the test or delete the claim.
- **low** — :403-436 vs get_bytes :363-374 — `download_file` reimplements get_bytes inline and diverges (NotFound carries key vs URL). Fix: call get_bytes, map error.
- **low** — :126-146 vs sources.rs:126-128 — `list_url` hand-builds `https://{bucket}.s3.amazonaws.com/` while object URLs use DataSources::s3_object_url — origin format declared twice despite "one place" contract. Fix: shared s3_origin helper.
- **nit** — :105-114 — Identifier::date_time slices 4..12/13..19 without checking byte 12 is '_'; malformed name still parses. Fix: check separator.
- **nit** — :815 — test doc-link `[super::client]` dead (constructor is crate::tls::client).

## level3.rs
- **low** — :80-82 — `site_code` `&id[1..]` guarded only by len==4; 4-byte id starting with multibyte UTF-8 panics on non-char-boundary. Latent (callers pass ASCII). Fix: `id.get(1..).unwrap_or(id)`.
- **nit** — :386-390 — comment "nearly a day and a half old" for a 13h05m stamp (assert says 13h). Fix: "over half a day".
- **nit** — :30-36 — module doc self-contradicts on N0S daily volume (342 vs 294; srm.rs:12 says 294). Reconcile or date-stamp.

## palette.rs
- **medium** — :20-26 with :206 — Gradient scales starting at f32::NEG_INFINITY (ZDR): value below second threshold → t = inf/inf = NaN → NaN interpolants cast to 0u8 → ZDR below −2.0 dB renders semi-transparent BLACK (0,0,0,180) instead of dark gray (66,66,66). Reachable: L2 ZDR spans ~−7.9..+7.9; render.rs filters only NaN/≥999. Fix: skip gradient branch when !last_threshold.is_finite(), or replace NEG_INFINITY stop with finite floor (−8.0).
- **low** — :104-112 — NaN handling inconsistent per product: velocity_lookup doesn't filter → NaN paints dark green (0,100,0); SW/RHO/PHI/KDP/ZDR return first stop color for NaN; REF/NROT filter. Shielded by render.rs today, but get_color_for_value is pub and called from rustdar-egui. Fix: one `if !value.is_finite() { return (0,0,0,0); }` at top of dispatcher.
- **low** — :116 vs :367-386 — NROT transparency cutoff |nrot|<0.5 but scales' first stop is 0.25 ("weak") — [0.25,0.5) can never render, yet get_legend_scale presents 0.25 as visible. Fix: align cutoff to 0.25 or start scales at 0.5.
- **nit** — :22-25 — interpolation `as u8` truncates; bias up to 1 LSB low. Fix: round.
- **nit** — :374 — profanity in comment `(2.75, (255,255,255)), // oh fuck`. Rename e.g. "catastrophic".
- **nit** — :3 — const named TRANSPARENCY holds alpha/opacity 180 — name says opposite. Rename ALPHA/OPACITY.
- **nit** — :106 — velocity exactly 0.0 classified inbound (paints 0,100,0). Boundary asymmetry; comment or >= 0.0.
- **nit** — :237-238 — PHI comment claims 0°/360° "visually continuous"; scale_color never interpolates past last stop → 345..360 flat then step. Soften comment or add synthetic 360 stop.

## render.rs
- **low** — :150-154 — pixel coords `as i32` truncate toward zero: values in (−1,0) truncate to 0 and pass >=0 bounds check → samples just outside frame paint edge row/col 0; pixel 0 catchment 2× wide. Fix: `.floor() as i32`.
- **low** — :140,152 vs types.rs:50 — inconsistent Earth constants: render_gate km→lat via EARTH_RADIUS_KM=6371 (111.195 km/°); ImageBounds uses 1/111.32 (implies R≈6378). 0.11% mismatch ≈ 0.26 km / 1.2 px at 230 km edge; outermost N/S gates pushed marginally outside declared bounds. Fix: derive one from the other (LAT_KM_PER_DEG = R·π/180 in types.rs).
- **low** — :114-166 — no slant-to-ground (cos(elevation)) or 4/3-Earth beam model: echoes displaced outward with elevation (~1.7%/~4 km at 230 km @10° tilt). Standard simplification, but unrecorded. Fix: cos(elevation) per sweep or document.
- **low** — :900 vs :1061-1085 — `if gate_value <= 1 { continue; }` hardcodes digital encoding for ALL L3 products incl. legacy 16-level where level 1 is real data (the LUT would decode it). Latent (no legacy product rendered since N0S dropped). Fix: skip only ==0 when legacy LUT present, or document restriction.
- **low** — :536,588,853 with :377-382 — returned max_range is data extent (super-res reaches ~300-460 km) while raster clips at MAX_RANGE_KM=230 and bounds span ±230 — a consumer scaling by max_range_km would mis-georeference. Verified: nothing scales by max_range_km (_max_range_km unused at ui_map_pane.rs:155). Fix: clamp to MAX_RANGE_KM or document.
- **low** — :562, 680, 906 — 999.0 no-data sentinel bare at 3 sites. Fix: named const with provenance comment.
- **nit** — :596 — avg_spacing = 360/num_radials assumes full-circle sweep; sector scan would inflate NROT wedges. Latent. compute_azimuth_spacing exists and could be reused.

## scan.rs
- **low** — :139-152 — "requested time too new → use latest" branch unreachable (best_time drawn from set whose max is latest_time; condition unsatisfiable). Behavior still correct; dead logic + impossible comment + log that can't fire. Fix: delete branch, or compare full NaiveDateTime for cross-midnight intent.
- **nit** — :227 — list_files_with_fallback inside per-day loop re-LISTs previous day on empty mid-range day (deduped later; wasted round trip). Fix: plain list_files in loop.
- **nit** — :68, 99, 182, 226, 267, 285 — "parse HHMMSS from name().split('_').nth(1)" reimplemented six times across five functions with variations. Fix: shared name_time() helper.

## sites.rs
- **nit** — :6 — `pub elev: Option<i32>` no unit doc; values are feet (KGJX 9992). Fix: doc "feet MSL".
- Spot-checks passed (KTLX, KMPX, Guam/Korea/Okinawa/Azores signs, 45 TDWRs).

## sources.rs
- **nit** — :146-151 — hrrr_key accepts any u8 run/forecast hour; run_hour 25 builds nonexistent key (404 far from mistake). Fix: debug_assert or doc.
- CORS/preflight tables consistent.

## srm.rs
- **nit** — :577-580 — quantizer doc formula indexes edges[-1] for level 1 (code correct; comment wrong). Fix: restate ranges.
- **nit** — :594-601 — quantize_to_rpg_levels(NaN) falls through to 14 (strongest outbound). Latent (validation only). Fix: debug_assert finite / sentinel.
- Production code (SRM sign convention, m/s→kt, center-azimuth correction, DERIVED_SCALE/OFFSET, NaN-safe saturating casts) verified correct; pinned by offline tests.

## tls.rs
- **nit** — :49 — USER_AGENT hardcodes "rustdar/1.0" regardless of version. Fix: env!(CARGO_PKG_VERSION) or document frozen.
- **nit** — :380 — probe subprocess sets RUSTDAR_TLS_PROBE=1; nothing reads it. Remove or assert on it.

## types.rs
- **low** — :105-113 — unknown site ID falls back to lat/lon 0,0 "Null Island" RadarSite: renders geographically wrong but plausible image instead of failing; warn only. Fix: propagate error/Option from ScanInfo::from_scan, or is_fallback flag surfaced in UI.
- **nit** — :50-51 — lon_deg_per_km degenerate as |lat|→90; northernmost site PAPD 65° safe. Latent.

## Cargo.toml
No defects; pins/gating match code, comments accurate.

## Systemic observations
1. Duplicated single sources of truth: S3 origin format ×2, archive bucket ×2 (with phantom pinning test), 999.0 sentinel ×3.
2. Load-bearing comments mostly excellent; the few that contradict code are a real defect class here (phantom test, N0S counts, day-and-a-half, unreachable branch, PHI continuity).
3. Non-finite handling inconsistent at module boundary; one is_finite guard in get_color_for_value closes the NaN-input inconsistency — but not the ZDR black-pixel bug, which fires on finite inputs (the NaN is manufactured inside interpolation) and needs the finite-floor/skip-gradient fix separately.
4. Projection constants 111.32 and 6371 are independent, not derived from each other (PIXELS_PER_KM is already derived: types.rs:23 = IMAGE_SIZE/(2·MAX_RANGE_KM)).
5. Error handling and network hygiene strong (strict 200-only, truncation-refusing pagination, page caps).
6. Test discipline unusually high.

---

# Audit: rustdar-overlays GLM + HRRR + Cargo.toml
(Verified against grib 0.16.0 and hdf5-pure 0.25.0 sources and the GLM render handler.)

## glm/mod.rs
- **nit** — :23-26 — Doc mixes sign and hemisphere ("-25°W") and GOES-West far bound is wrong hemisphere (~156-170°E per cf.rs, not "170°W"). Fix: signed degrees consistently.

## glm/h5.rs
No defects. Verified `convert_attr` `_ => None` arm can't drop GLM scale_factor (hdf5-pure widens to F64/I64).
- **nit** — :235-236 — comment names attribute types that can't occur (no object-reference variant in hdf5-pure AttrValue). Fix: name real variants.

## glm/cf.rs
No defects. Verified _Unsigned reinterpretation, fill/valid_range in raw domain, time-unit table, epoch parsing.
- **nit** — :217-219 — `attr_as_f64` silently takes first element of >1-length vector attr (pinned by test; deliberate). A warn_once for len>1 would match module posture.

## glm/fetch.rs
- **medium** — :208-214 — Deselecting a satellite doesn't stop its cached flashes rendering (up to 30 min): `flashes_in_window` filters by time only, never by `satellites`; handler (handlers/glm.rs:741-751) does NOT clear_cache on satellite change (does for level toggles). "Both"→"East" keeps drawing GOES-West lightning. Fix: `&& satellites.contains(&f.satellite)` in filter, or clear cache on satellite change.
- **low** — :461 — Byte-index slicing of S3 key can panic on non-UTF-8 boundary (`&s_field[..14]`) on a network-data path. Fix: `is_char_boundary(14)` check or verify ASCII digits first.
- **low** — :30-43 — `CachedGranule::new` comment claims records land at/after granule_start; module's own tests show time axis spans [-5s, +20s]. Granule with only-negative-offset records gets `newest < granule_start` → evicted while still listed → re-downloaded each poll. Fix: `max(granule_start)`.
- **low** — :389-391 — Listing lower bound drops the granule straddling window start → cold start missing up to ~20s of oldest lightning. Warm caches self-heal. Fix: widen filter to `file_start >= start - 20s`, let flashes_in_window trim.
- **low** — :145 — Listing failure on second satellite `?`-aborts whole poll after satellite 1 absorbed but before cache_granules → satellite 1's batch re-downloaded next poll. Fix: record error, continue; or cache inside loop.
- **nit** — :606-618 — `download_bytes` no size cap (~250KB typical; misbehaving server can stream unbounded ×20 concurrent). Fix: cap a few MB.
- **nit** — :144-173 — satellites fetched sequentially; "Both" doubles latency. Fix: futures::join!.
- **nit** — :526 — `buffer_unordered(20)` magic number referenced by comments at :170/:510-511. Fix: named const.
- **nit** — :212 — every in-window flash cloned every poll (up to ~500k records); handler re-wraps in Arc (handlers/glm.rs:476/:505). Fix: Arc-based cache or indices. Perf only.

## glm/tests.rs
- **nit** — :295 — comment arithmetic wrong ("15.0079 - 5 = 10.0079"; actually 15.0088−5=10.0088; assert right). Fix digits.
- All other numeric fixtures independently verified correct.

## hrrr/mod.rs
- **low** — :1003-1025 — `GridCoords::nearest` doc says None when grid doesn't cover point; Explicit branch is unconditional argmin (Lambert branch refuses, pinned by test). Hover over London on non-3.30 grid reports grid-edge reading. Fix: cap accepted distance (~one cell) or amend doc.
- **nit** — :1014-1016 — Explicit nearest uses unweighted degree distance (no cos(lat), no antimeridian). Fallback path only.
- **nit** — :389-502 vs 543-897 — `legend_thresholds` hand-duplicates every color-ramp anchor; currently all agree (verified); nothing stops drift. Fix: one anchor table or a cross-check test.
- **nit** — :1078-1082 — blank notice counts NaN points in "all N points" claim. Fix: use finite count.
- **nit / unverified** — :41-44, 124-129 — VUCSH/VVCSH treated as m/s bulk shear components (→kt); generic GRIB2 table says s⁻¹; ranges match kt-scale. Needs product-doc check.

## hrrr/fetch.rs
- **low** — :121 — `next.offset - 1` underflows u64 on malformed index (offset 0 / non-monotonic) → debug panic; release sends bytes=start-1844...615 → S3 serves to EOF (~130MB). Fix: `checked_sub(1).filter(|e| *e >= start)`.
- **low** — :406-429 vs 454-484 — latest-run/previous-run retry scaffold duplicated verbatim between `fetch_hrrr_data` and `fetch_composite_hrrr_data`. Fix: shared `retry_over_runs` helper.
- **nit** — :511-517 — composite merge takes grids[0]/grids[1]; `< 2` guard effectively dead; a 3-part composite would silently drop part 3. Fix: `!= 2` guard.
- **nit** — :283-311 — bounds computed by inverse-projecting all 1,905,141 points every fetch; boundary walk (~5.7k) suffices. Also empty coords → inverted MAX/MIN bounds; guard the empty case when computing bounds.
- **nit** — :425-426 — surfaced error names only the fallback run's failure; first error only in warn log. Fix: include both.

## hrrr/lambert.rs
No math errors (verified against Snyder: cone constant, n<0 signs, t/m/rho/theta, index_bounds extremum set, -0.0 edge, seam double-cut).
- **nit** — :401-414, 433-434 — `detect_longitude_wrap` inverse-projects every boundary point twice. Fix: carry previous raw_b.
- **nit** — :472-474 — `len()` unchecked `ni * nj`; wasm32 (32-bit usize) hostile section 3 wraps in release before check_point_count rejects. Fix: checked_mul in from_template.

## Cargo.toml (overlays)
- **nit** — :19-22 — grib `png-unpack-with-png-crate` (DRT 5.41) enabled; no HRRR field uses it; pulls png crate; only feature without a justifying comment. Fix: drop or justify.

## Systemic observations
- Error-handling asymmetry: GLM has edge-triggered dedup failure taxonomy; HRRR plain warn!/error! per poll, last error string only.
- Selection changes vs cache invalidation inconsistent (level toggles clear; satellite changes don't; fetch filters by neither).
- Real duplication: HRRR retry scaffold; color anchors.
- Test culture unusually strong (golden values from netCDF-C/PROJ/EPSG); gaps are exactly the unpinned spots.
- Sequential awaits where concurrency is free (GLM per-satellite, HRRR composite parts).

---

# Audit: rustdar-overlays (metar/, nws/, spc/, render/ top-level, lib.rs, types.rs)
(render/handlers/ read only for usage verification; cargo test spc::discussion run to confirm CDATA merge claim — passes.)

## lib.rs
No findings.

## types.rs
- **nit** — :44-45 vs 46-54 — parse_polygon_coords: single non-array RING aborts whole polygon (`?`), malformed POINT silently filtered. One bad ring in MultiPolygon drops good rings. Fix: filter_map rings too, or document.

## metar/types.rs
- **low** — :46-60 — Visibility::parse dead code, zero callers (fetch uses visibility_from on IEM data; doc still refers to abandoned AWC feed). Delete or mark planned-fallback.
- **nit** — :3, 109-116 — FlightCategory/CloudLayer derive Deserialize (+rename) but nothing deserializes them. Drop derives.
- **nit** — :64-71 — Visibility::label {:.1} prints 1/4-mile as "0.2". Fix: two decimals below 1 or render fractions.

## metar/fetch.rs
- **low** — :373-377 — raw_visibility_is_a_bound scans ENTIRE raw report (unlike raw_wind_group :503-513 which stops at RMK/TEMPO/BECMG): "... 4000 BR BECMG 9999" → 4000 m measured but trend-group 9999 marks or_greater. Fix: same section-stop logic.
- **nit** — :490-498 — classify_wind accepts bearings > 360 (corrupt "99905KT" → Degrees(999), barb draws at trig-equivalent). Fix: reject d > 360.
- **nit** — :289-297 — negative numeric cells saturate ((-1.0).round() as u16 → 0 → misclassified Calm/Variable) instead of counted by Rejections. Fix: filter v >= 0.0 (and upper bound), note via rejects.

## metar/networks.rs
- **nit** — :450-454 — sort comparator recomputes centre_distance per comparison. Fix: sort_by_cached_key on squared distance.

## nws/alert.rs
- **nit** — :143 vs :111 (reused :199) — inner `let features` shadows outer JSON-array binding; correct but misreadable. Rename overlay_features.

## nws/colors.rs
No findings (pinned by tests).

## nws/fetch.rs
- **medium** — :6 — NWS alerts URL hardcoded literal bypassing DataSources (nws_api_base exists; SPC has origin tests). Fix: thread &DataSources, build from sources.nws_api_base. [= cross-package MEDIUM-1]

## nws/zones.rs
- **low** — :49-58 — O(n²) URL dedup via Vec::contains on 1000+ URLs (~10⁶ compares per refresh). Fix: HashSet/IndexSet.
- **nit** — :126-137 — each alert referencing a shared zone clones polygons and re-runs ear-clip triangulation; N alerts over same county = N triangulations. Fix: cache per-URL geometry, recolor per alert.
- **nit** — :225-231 — zone_cache_key doesn't sanitize URL last segment as filename. Unverified: can affectedZones URLs carry non-path chars (not today). Fix: filter [A-Za-z0-9_-] or hash.

## spc/colors.rs
- **medium** — :2-11 — parse_hex_color can PANIC on network data, contradicting own doc ("Falls back to grey"): `&hex[0..2]` on multi-byte UTF-8 fill/stroke property panics on non-char-boundary. Runs inside outlook fetch task; panic swallowed by runtime → overlay stuck "fetching" (failure mode spc/discussion.rs documents for its own parser). Fix: hex.get(0..2) or is_ascii() check, fall back to grey.

## spc/discussion.rs
- **low** — :255-262 — decode_html_entities replaces &amp; FIRST → double-encoded text decodes twice (&amp;lt; → <). Standard rule: &amp; last. Fix: move to end.
- **low** — :48-54 — classify_md_type bare substring "ICE" matches OFFICE/SERVICE/NOTICE → misclassified WinterWeather (blue tint) for MDs mentioning "forecast office" with no convective keyword. Fix: word boundaries or specific phrases ("ICE STORM", "FREEZING RAIN").
- **nit** — :219, 226, 234 — title with no extractable number → number = 0, labelled "MD 0" on map. Fix: Option<u32> or title label.
- **nit** — :281-283 — while contains("\n\n\n") replace loop O(n²). Fix: single pass.

## spc/fetch.rs
- **low** — :97-112 — available_products dead code, zero callers (verified; other hits are radar scan_info field), duplicates OutlookDay::products() verbatim. Delete.

## spc/outlook.rs
- **medium** — :187-191 (with 217-222, 278-280, 313-318) — One malformed feature aborts the ENTIRE outlook: parse_outlook_feature? propagates; parse_multi_polygon errors if any single polygon invalid (all-rings-under-3-points becomes fatal). Contrast nws::alert::parse_alerts which skips bad features. Single degenerate polygon in SPC feed blanks the whole product. Fix: log-and-skip malformed features (Rejections-style count); Err only for unusable envelope.
- **low** — :111-112, 263-270 — SpcOutlook.valid/expire parsed, stored, never read (dead fields); NaiveDateTime from UTC %Y%m%d%H%M — future consumer comparing to local now would be wrong by UTC offset. Fix: remove, or doc /// UTC (better DateTime<Utc>) + consumer.

## spc/reports.rs
- **low** — :67-82 — all three CSV fetches failing still returns Ok(vec![]) (warn only); total outage renders as normal quiet-day empty layer. METAR treats same as Err. Fix: Err when all three Err, matching METAR policy.

## render/geo.rs
- **low** — :94-100 — strip_closing_dup byte-identical duplicate of rasterize.rs:825-831 (same crate); can drift (len>3 threshold). Fix: one pub(crate) copy in geo.rs.
- **nit** — :40-67 — rdp_simplify recursion depth O(n) worst case. Unverified: any NWS ring big enough to threaten wasm stack. Fix: iterative.

## render/hatch.rs
- **medium** — :165-192 — Exclusion mask built with single EvenOdd fill of hatched ring + ALL exclusion rings; parity breaks in exactly the nested case the pass exists for (commit b7f2ebd): point inside CIG1∧CIG2∧CIG3 crosses 3 rings → odd → FILLED → CIG1 hatch drawn inside CIG3 again. Also: exclusion ring disjoint from hatched polygon but inside bbox gets lower level's hatching. Unverified: whether SPC CIG polygons actually nest in live feed; if so, live bug. Fix: for Cig1 exclude only cig2 (cig3⊂cig2 keeps parity), or Winding fill + per-ring mask intersection.
- **nit** — :168 — full-texture Mask::new per hatched polygon; several multi-MB transient allocs per rasterize. Fix: allocate once, clear between.

## render/layers.rs
- **low** — :107-230 — LayerManager, LayerState, and every method + most LayerKind methods have ZERO callers outside the file (verified). Leftover central layer table from before handler-owned-state design (overlay_state.rs:147: "there is no central layer table"); spc_layers_for_day duplicates the day→product table a third time. Only LayerKind + all() used (by ui_config.rs). Fix: delete all but LayerKind + all().

## render/overlay_state.rs
- **low** — :495-500 vs 521-526 — build_enabled_map and save_enabled_map byte-identical under two names (both called from ui.rs). Fix: one delegates.

## render/rasterize.rs
- **low** — :408-414 — dead computation in draw_hail_symbol: adx/ady computed then `let _ =` discarded — remains of intended diagonal-tick layout. Delete or finish.
- **low** — :749-768 (with geo.rs:109) — draw_feature fills only polygon.first() (exterior); holes parsed/stored (types.rs:24) but never subtracted → NWS zone/alert polygon with holes paints hole area as alerted. Fix: build path with all rings + FillRule::EvenOdd (hatch.rs already does; one-line change). [pairs with egui overlay_cache hit-test holes finding]
- **nit** — :674-680 vs :688 — GLM first cull rejects flashes strictly outside geo bounds, making pixel-cull's ±base_size slack unreachable → bolts pop at texture edge (mitigated by 3-viewport overdraw). Fix: pad geo cull or drop it.
- **nit** — :631-636 — energy_size_scale(Some(negative)): log10(neg)=NaN propagates into bolt_size/hit-map (path builder rejects NaN; nothing guards). Unverified: GLM energy can't be negative post-CF. Fix: Some(e) if e > 0.0.
- **nit** — :833 — `use crate::hrrr::HrrrGridData;` mid-file.

## render/station_model.rs
- **medium** — :202-219 — BKN "open slice" paints opaque background-colored RECT whose corners extend beyond the circle → overpaints map/radar pixels beneath station (METAR draws above Radar). Visible grey/dark blotch over colored echo. Fix: clip to circle (inscribed chord polygon) or filled circle + unfilled wedge outline.
- **low** — :307-313 — barb quantization deviates from WMO standard the doc claims: remainder 3-9 kt always half barb → 8-9 kt draws as 5 (standard rounds to nearest 5: 8→10 = full barb); 58 kt → pennant+half (55) not pennant+full (60). Fix: round to nearest 5 before decomposing.
- **low** — :239-253 — cloud_cover_fraction omits "OVX" (obscured) which CloudLayer docs list and ceiling_ft treats as ceiling → OVX plots as CLEAR circle while deriving LIFR. Fix: add "OVX" to 1.0 arm.
- **low** — :53-60, 133-141 — temp/dewpoint hardcoded °F in plot and hover while same hover converts wind via prefs.speed; UserPreferences.temperature exists. Fix: convert via prefs (hover at least); document US-station-model °F convention if plot stays. [= cross-package HIGH-3]
- **nit** — :89 — in_hg_tenths holds hundredths (comment above says so). Rename.

## render/{mod,controls,draw}.rs
No findings.

## Systemic observations
1. SPC day→product table exists in THREE places: OutlookDay::products() (live), spc::fetch::available_products (dead), LayerManager::spc_layers_for_day (dead). Delete two.
2. Malformed-feed policy inconsistent: NWS skips bad features; METAR counts rejections; SPC aborts entire product. METAR's Rejections pattern is best; share it.
3. Origin-table discipline uneven (SPC tested, IEM threaded, NWS hardcoded).
4. Small-helper duplication in-crate: strip_closing_dup ×2; AABB overlap ×2 (GeoBounds::intersects vs StateNetwork::intersects); build/save_enabled_map ×2; Polygon/MultiPolygon dispatch ×2 with opposite error policies.
5. Error-swallowing gradient: METAR all-fail → Err; storm-reports all-fail → Ok(empty); zone-geometry failures → debug log only.
6. Dead-code cluster from handler-registry refactor: LayerManager+LayerState, available_products, SpcOutlook.valid/expire, Visibility::parse + serde derives, adx/ady. One sweep removes all.
7. Positives: Fahrenheit/InchesOfMercury newtypes, Rejections counter, NWS color rename-detection tests, projection_window equivalence sweeps — several findings are just other modules not yet held to that standard.

---

# Audit report — rustdar-egui core (lib.rs, actions.rs, config_store.rs, input_harness.rs, overlay_cache.rs, pane.rs, point_painter.rs, tile_source.rs, tile_source/tests.rs, tiles.rs, Cargo.toml)

All files read in full. Crate compiles clean. No TODO/FIXME/HACK markers in scope. No critical or high findings.

## actions.rs
- **nit** — actions.rs:22 (with pane.rs:602) — Default site `"KTLX"` magic string duplicated in two places; drift would make `RadarConfig::default()` and `PaneState::new()` disagree silently. Fix: hoist `pub const DEFAULT_SITE: &str = "KTLX";` and use in both.

## config_store.rs
No findings. Poisoned-mutex `load` returning `None` (line 54) documented as intended.

## input_harness.rs
- **low** — input_harness.rs:112, 173, 557–559 — `pane_rect` hardcoded to (220,80)–(1004,690) for 1024×768, never rescaled by `with_screen`/`set_screen`, so `map_center()` lies off-screen on narrower screens. E.g. `the_hover_readout_follows_the_modality_not_the_width` (3524–3527) clicks at x≈612 on a 500pt-wide screen; passes only because modality latches on the event, not position. Fix: derive `pane_rect` from `screen_rect`, or assert `screen_rect.contains(pos)` in `mouse_click`/`touch_tap`.
- **nit** — input_harness.rs:1824 — Comment says "thirty seconds" but loop runs `HOLD_MUST_SURVIVE_S` = 45.0 (line 980). Fix: say 45 or name the constant.
- **nit** — input_harness.rs:30/42 vs 651–653/670 — Version-pinned line references drift: header cites egui-winit/eframe 0.35.0 and `lib.rs:784`; `cursor_left` doc cites 0.34.1 `lib.rs:796`; web section says "mirrors eframe 0.34.1". Fix: standardize on one version's references.
- **nit** — input_harness.rs:1835, 2373 (dup "7."), 1909, 2415 (dup "7b."), 3116/3181/3236 (27/29/28 out of order), 2895 vs 3708 ("16b" before "16") — test numbering duplicated/non-monotonic. Fix: renumber or drop numbers.
- **nit** — input_harness.rs:3724 — Redundant mid-module import (`OverlayKind`) already available via `use super::*;` (the adjacent `GuiAction` import at :3723 is required). Fix: move to top of `mod tests` or delete.
- **nit** — input_harness.rs:601–607 — `frames_for(0, _)` returns `FrameOutcome::default()`, vacuously satisfies negative assertions. No current caller passes 0. Fix: `assert!(count > 0)` in helpers.
- Harness otherwise strong: every "must not happen" test carries a positive control.

## overlay_cache.rs
- **medium** — overlay_cache.rs:362–383 — `geo_point_in_feature` tests only the exterior ring (`polygon.first()`), ignoring holes. Click inside a hole of an SPC/NWS polygon reports "inside" → wrong feature popup. Live path: ui_map_overlays.rs:146. Fix: after exterior hit, iterate `polygon[1..]`, return false if any hole contains point.
- **low** — overlay_cache.rs:29–30 — Doc names wrong crate: decode lives in `rustdar-frontend/src/app_fetch.rs:474,508`, not `rustdar-platform`. Fix: s/rustdar-platform/rustdar-frontend/.
- **low** — overlay_cache.rs:356–373 — No antimeridian wrap handling in point-in-polygon. Alaska/Guam sites exist (sites.rs:887-929 — PABC/PAEC/PAHG/PAPD/PGUA), so overlays crossing ±180° are reachable.
- **nit** — overlay_cache.rs:179 — Field doc hardcodes "(`zoom * 32`)"; goes stale if `ZOOM_QUANTIZATION_FACTOR` changes. Fix: name the constant.
- `plan_overlay_texture`/`pan_exceeds_coverage` math and containment proof verified OK.

## pane.rs
- **low** — pane.rs:792–814 — `PaneLayout::for_count`: count==0 or >6 falls back to `grid = vec![1]` while `pane_count` keeps the passed value; `pane_rect` for idx≥1 returns `total_rect` (all panes full-size, stacked). Currently unreachable (ui_config.rs:202 clamps). Fix: `count.clamp(1, MAX_PANES_DESKTOP)` before both uses, or debug_assert.
- **nit** — pane.rs:549–571 — `Vec::contains` in loop, O(budget²); fine at current sizes.
- **nit** — pane.rs:616 — `time_step_secs: 600` magic number; deserves named constant.
- **nit** — pane.rs:917–922 — `drag_divider` rejects overshooting drag wholesale instead of clamping at `MIN_RATIO`; fast drag makes divider stick. Fix: clamp `ratio_delta`.
- **nit** — pane.rs:1433–1434 (test) — pointless rebinding `let same_site = ...; let mut same_site = same_site;`.
- **low** — pane.rs:497–521 (+ app_render.rs:1057) — `retarget_renders` blanks frame textures but not `phase`, and the live caller app_render.rs:1057 does NOT demote phase — a mid-playback retarget keeps phase Playing over blanked textures. Fix: demote phase when retarget_renders returns true.
- Loop-state test suite (1016–1803) exemplary.

## point_painter.rs
No findings. (Convexity requirement is already documented on the trait — rustdar-overlays/src/render/draw.rs:48 "Convex only." — and all four call sites pass convex shapes.)

## tile_source.rs
- **low** — tile_source.rs:309–312, 392–414 — De-dup guarantee weaker than documented: pending-`None` marker shares the 256-entry LRU with real tiles; >255 touches while in flight evicts marker → duplicate request. Doc claims unconditional "never requested twice". Fix: soften doc or pin pending entries.
- **low** — tile_source.rs:57–60, 103–107 — Timeout claim doesn't hold on wasm: `rustdar_radar::tls::client` (tls.rs:99–102) ignores `timeout` and `https_only` on wasm32. A never-answering tile does hold a download slot forever in the browser. Fix: doc the wasm exception or wrap `fetch_one` in a futures timeout on wasm.
- **nit** — tile_source.rs:291 — `tile_client()` `.expect(...)` startup panic; consistent with app posture; noted only.
- Overflow-hardening (checked_pow 222–224) and shift-safety argument (246–258) verified OK.

## tile_source/tests.rs
- **nit** — tests.rs:272–274 — Redundant `..` rest pattern on exact-field variant.
- **nit** — tests.rs:1150–1165 — `the_tile_client_accepts_https` asserts `is_connect()`; on silently-dropped port 1 traffic becomes timeout → fails. Fix: accept `is_connect() || is_timeout()`.
- **nit** — tests.rs:240–256 — `serve_one` assumes whole request line in first read; documented; a read-until-CRLF loop would remove the assumption.
- Suite quality high; no vacuous tests.

## tiles.rs
- **low** — tiles.rs:85–97 — `lon_to_tile_x`/`lat_to_tile_y` clamp only the lower bound: lon==180 returns `n` (one past grid); lat < −85.05° returns ≥ n. Sole caller (ui_map_overlays.rs:184–187) clamps max end so failure is empty range, not panic; contract undocumented. Fix: symmetric `.min(n-1.0)` clamp or document.
- **low** — tiles.rs:202–209 — `MapTileState::clear` theme flip is dead logic with misleading comment; `ensure_base_tiles` overwrites the flag every frame. Fix: delete flip + comment.
- **nit** — tiles.rs:59 — Subdomain sharding `x % 4` only; column pins to one host; conventional is `(x+y) % 4`. Cosmetic.
- **nit** — tiles.rs:157–163 — `ensure_label_tiles` doesn't update `current_theme_is_dark` (order-dependent API, correct only at current call site ui_map.rs:32/38). Fix: set flag in both or doc.

## Cargo.toml
No findings; all exact pins verified against Cargo.lock.

## Systemic observations
1. Doc drift is the dominant defect class (at least six places).
2. Guarantees documented as absolute but only probabilistic (tile de-dup, wasm timeout).
3. Duplicated defaults: "KTLX" in actions.rs:22 and pane.rs:602.
4. Antimeridian handling consistently unaddressed — fine for CONUS, but Alaska/Guam sites do exist (sites.rs:887-929); one deliberate decision needed.
5. No unwrap/expect/indexing on fallible runtime paths in production code; no races found in tile cache.
6. Test harness quality well above typical.

---

# Audit: rustdar-egui UI files

## ui.rs
- **medium** — ui.rs:869-877 — Panes created by the pane-count picker get empty enabled_overlays/overlay_configs; `initialize_pane_enabled()` never called after (verified only callers ui.rs:564, ui_config.rs:280). With Sync Layers OFF a new pane renders NO overlays at all incl. Radar (pane.rs:645-647 falls back false). Masked by default sync_layers=true. Fix: seed new pane from `build_enabled_map()`/`save_pane_configs()` or call initialize after growing.
- **low** — ui.rs:1027-1031 — time-step dropdown shows "10 min" for any value not in TIME_STEP_OPTIONS (hand-edited/legacy config loads unvalidated via ui_config.rs:240/250) — label lies. Fix: format raw value as fallback or snap on load.
- **low** — ui.rs:1141-1160 — lookback slider commits only on `drag_stopped()`; keyboard edits never fire it → snaps back. Verified: egui 0.35 keyboard/typed edits set no drag flags — snap-back is real. Fix: commit on changed()&&!dragged() or debounce.
- **low** — ui.rs:444-472, 1334-1341 — generic ControlItem::Slider pushes ControlUpdate every frame of a drag; the GLM time_window slider maps to ControlEffect::Fetch (glm.rs:595-603, 752-757) with no in-flight guard — a drag fires a fetch per frame. Fix: commit on release like lookback slider.
- **low** — ui.rs:1367-1379 vs 1593-1604 — `set_active_pane_overlay` and `set_overlay_on_pane_for_test` near-identical. Fix: one private `set_pane_overlay(idx, kind, on)`.
- **low** — ui.rs:1856-1899 (sync_viewports) + ui_map.rs:93,97 — indexes panes for 0..pane_count without visible_pane_count() clamp that ui.rs:1463-1492 documents as mandatory. Invariant currently holds; posture inconsistent. Fix: use visible_pane_count().
- **nit** — ui.rs:980-981 — dead fallback `.unwrap_or(0.0)` (guarded non-empty at :972).
- **nit** — ui.rs:101, 515 — 60s base poll interval bare literal twice; 300s cap third (:106). Fix: BASE_POLL_SECS/MAX_POLL_SECS consts.
- **nit** — ui.rs:1311-1315 — separator drawn before every kind after first even if kind emitted no controls → doubled separators. Current handlers always return ≥1 control, so latent-only. Fix: skip when empty.
- **nit** — ui.rs:1869, 1877 — magic sync thresholds (0.0001 zoom, 0.00001 deg). Fix: named consts.

## ui_chrome.rs
- **medium** — ui_chrome.rs:224-230 (and ui_menu.rs:249-251) — Refresh button/menu fetch `self.radar.config` verbatim; `config.site` is a global last-switched site, not the active pane's (check_auto_polls overrides it, ui.rs:648-649; app_fetch.rs:313-315 writes global even with sync off). With per-pane sites, Refresh can fetch a site the active pane isn't viewing. Fix: substitute active pane's site at both refresh sites.
- **medium** — ui_chrome.rs:539-553 (called :284 with &self.panes) — `render_hover_info` scans FULL panes vector, not visible panes() slice; hidden pane keeps last hover_value forever (nothing clears) → stale readout surfaces in status bar indefinitely. Fix: pass visible slice. (Related clear bug in ui_map_pane.)
- **low** — ui_chrome.rs:442-446 + ui.rs:118-121 — `time_until_next()` returns Some(0) after elapse; polling gated on is_any_pane_live() — all-historic panes show "Auto-poll (next in 0s)" forever. Fix: show "paused" when 0/no live pane.

## ui_config.rs
- **low** — ui_config.rs:134-152 vs :203-210, 236 — save writes PaneConfig for every pane incl. hidden; load only restores `.take(count)` → "hidden panes remembered on re-split" doesn't survive restart though data is in file. Fix: restore all entries, keep layout at count.
- **low** — ui_config.rs:226-227 — loaded numerics unvalidated (fps 0.0, lookback 0 accepted; save-side guard only fixes what this app wrote). Fix: clamp on load.
- **low** — ui_config.rs:83-103 — `storm_motion_override` not persisted; typed storm vector lost each restart. Unverified: deliberate (stale override arguably dangerous). Fix: persist with enabled=off on load, or document.

## ui_input.rs
- **low** — ui_input.rs:460 vs 576-578 — doc says "up = zoom in"; code adds positive dy (down) to zoom → down = zoom in (Google Maps convention; doc wrong). Fix: correct doc.
- **low** — ui_input.rs:670-680 — LongPressDetector cancels on movement but re-arms next frame; pausing ≥0.8s mid-pan fires long-press, suppresses pan mid-gesture. Unverified: intended UX? Fix: latch cancelled flag until release.
- **low** — ui_input.rs:734-740 — `any_click()` includes right/middle → right-click opens overlay pager popup. Fix: primary_clicked().
- **low** — ui_input.rs:540-557, 584-611 — second tap's press+release batched in one frame (file documents web batching :220-226): handle_press enters ZoomDragging but unconditional release handling runs with stale press_time from first tap; can overwrite ZoomDragging with WaitingForSecondTap. Fix: skip release when state became ZoomDragging this frame.
- **nit** — ui_input.rs:144, 180, 196, 412, 944, 1139 — comments cite egui-0.34.1 internals with line numbers; workspace pins 0.35.0. Fix: re-verify + update.
- **nit** — ui_input.rs:578 — zoom clamp (1.0, 19.0) magic; everything else named. Fix: MIN_ZOOM/MAX_ZOOM (agree with tile pyramid depth).

## ui_layout.rs
- **nit** — :20-21 with 240-244 — COMPACT_DIALOG_MARGIN=32 doc says "either side" but subtracted once (16/side; ui_popups.rs:46-47 says 16 each side). Fix: reword "total".
Otherwise clean; tests thorough.

## ui_map.rs
- **low** — ui_map.rs:33 — `any_city_labels` scans full panes vector vs visible slice; hidden pane keeps label-tile fetching alive. Fix: take(pane_count)/panes().
- **low** — ui_map.rs:93,97 — pane loop indexes panes[0..pane_layout.pane_count] without visible_pane_count() clamp — config with count ahead of list would panic in renderer (exact scenario a test pins elsewhere). Fix: visible_pane_count().
- **nit** — ui_map.rs:234-246, 262-288 — divider drags not visible to layer_id_at; press on divider can flip active pane while resizing. Verified: detect runs first and layer_id_at can't see the child-Ui divider — claiming the press would not help. Fix: skip detect when press starts on divider.

## ui_map_overlays.rs
- **nit** — :184-187 — min_tx/min_ty not clamped to n-1 like max side; rests on helpers clamping internally (they clamp only lower bound — see tiles.rs finding). Fix: clamp both ends symmetrically.

## ui_map_pane.rs
- **medium** — :110-171 with 440-504 — `pane.hover_value` only cleared inside radar-metadata path; disabling Radar overlay / losing cache / removing from draw_order leaves last hover string frozen; status bar displays stale lat/lon/value indefinitely. Fix: clear at top of render_pane_map_content (as overlay_hover_value at :229) and let radar arm re-set.
- **low** — :203-210 — comment + `fg_layer` name say Foreground; code uses Order::Background (deliberately changed in c5251dd; comment/name stale). Fix: rename + rewrite comment.
- **low** — :71-86 vs 564-600 — site-icon geometry (size formula + expand(100.0) cull margin) duplicated between site_icon_rects and handle_radar_site_interactions; drift breaks click-through exclusion. Fix: one site_icon_rect() helper.
- **low** — :602-639 — overlapping site icons: one SwitchRadarSite pushed per overlapping icon (no break); loading_site overwritten; hover draws tooltip per site. Fix: act on nearest, stop.
- **low** — :704-708 — GPS-dot tooltip checks contains(hover_pos) but not is_pos_blocked (site-icon hover does) → tooltip through dialogs. Fix: add guard.
- **low** — :837-1035 vs 1039-1192 — render_color_scale / render_overlay_color_scales duplicate ~130 lines (bar rects, gradient, label thinning, titles). Fix: extract draw_scale_bar().
- **typo** — :374 — "…hover tooltip (loop playback path) (loop playback path)." Fix: delete dup.
- **nit** — :419, 723 — magic 111.32 (km/deg lat) and 1.94384 (m/s→kt) inline; latter belongs in rustdar_units. Fix: named consts / unit helpers.

## ui_menu.rs
- **low** — :249-251 — "Refresh Radar" same stale-site issue as status-bar refresh (verified vs app_fetch.rs:313-315). Fix: substitute active pane's site.
- **nit** — :123 — top-level bar leaf rendered with in_menu=true → close_kind(Menu) outside any menu. Verified: close_kind with no menu parent is a no-op that logs an egui warning, and the path is currently dead (menu_model emits only submenus). Fix: pass false.

## ui_popups.rs
- **low** — :89 — every KeyValueGrid section uses fixed id "popup_kv_grid"; two sections → egui id clash. Fix: salt with section index.
- **low** — :186-201 — two same-frame triggered actions both remove(page) → second operates on shifted vec (panics with one item). Pointer clicks are one-per-frame in egui; the reachable path is focus+Enter/Space or accesskit concurrent with a click — rare but real. Fix: handle first only / break after removal.

## ui_settings.rs
- **nit** — :229-237 — direction DragValue 0.0..=360.0 admits both 0 and 360. Fix: 0..=359.9 or % 360 on commit.
- **nit** — :158-167 — Connect/Disconnect GPS both always enabled, no state feedback (comment acknowledges). Fix: single toggle driven by fix presence.

## Systemic observations
1. "All panes must use panes()" convention enforced only where recently fixed — render_hover_info, any_city_labels, raw pane_count indexing in render_map/sync_viewports bypass it. One sweep closes the class.
2. `radar.config.site` is a second source of truth for "which site"; three fetch paths read the global; only check_auto_polls compensates.
3. load-configs/save-configs handler-swap dance repeated 5× (ui.rs:634, 1299, 1370, 1596; ui_map_pane.rs:64). Fix: with_pane_handlers() helper.
4. Comment/version drift: egui 0.34.1 citations; stale "foreground layer" comment after deliberate order change.
5. Long functions: render_pane_map_content ~320 lines; render_map ~245; color-scale pair ~350 total duplicated (highest-value extraction); render_loop_controls ~175.
6. Test instrumentation hygiene excellent; no TODO/FIXME/HACK; UI strings typo-free.

---

# Audit: rustdar-frontend (app orchestration layer)

## src/app.rs
- **high** — app.rs:1585-1600 with :1921, :1486, :655, :1970 (line numbers at `1df1330`, not the audit base — the autosave postdates this sweep) — schedule_autosave_wakeup arms ControlFlow::WaitUntil so an unsaved change gets a look, but a WaitUntil expiry dispatches only new_events + about_to_wait; autosave_config is reachable only from handle_redraw, i.e. only from the RedrawRequested arm. The wake-up therefore cannot reach its own save, autosave.touched (cleared only at :1497) stays set, and the re-arm computes a zero delay against a deadline already behind the clock. wait_duration(0) is WaitUntil(now), which expires immediately: measured ~164,000 iterations/s at 99% of one core with nothing written, indefinitely. The sticky-control-flow half survives fixing the save alone (~162,000/s) because the !touched early return leaves the expired WaitUntil in place. Fix: call autosave_config(false) from about_to_wait, and return ControlFlow::Wait when nothing is owed. [= P0-9]
- **high** — app_fetch.rs:266-308 (get_scan_info call at :274; emit at ui.rs:662-671; apply path app.rs:598-637) — Auto-poll compares every site's latest scan against the ACTIVE pane's timestamp: get_scan_info() (ui.rs:1802) returns active pane's info but GUI emits one CheckForNewScans per unique live site; check_and_fetch_latest (scan.rs:200) fetches only when latest > current. Two live sites: (a) active site newer → other site's updates skipped round after round; (b) active pane older (historic view) → every poll re-downloads other site's full L2 scan unchanged + full apply path (reset_panes_for_site, five L3 refetches, re-renders) each interval. Fix: per-site current timestamp (look up a pane on radar_config.site, or HashMap<String, NaiveDateTime>).
- **medium** — app.rs:966-971 vs 655-675 — Deferred exit replay calls only event_loop.exit(), skipping needs_process_exit branch (:668-670). On Android (needs_process_exit=true; menu Exit is primary quit) process-exit half dropped. Fix: replay through request_exit(Some(event_loop)) guarding double save.
- **medium** — app.rs:176/979, app_render.rs:100-110, app_fetch.rs:473/507 — cached_dark_theme never populated when window.theme() is Some: desktop poll_theme hardwired None (platform.rs:65-68), ThemeChanged arm (:977-981) sets cache to None → on Windows/macOS dark mode, spawn_overlay_render uses unwrap_or(false) → overlays rasterize light-theme under dark UI. Also ThemeChanged never bumps radar-sites gen (Android poll path does, app_fetch.rs:427) → site labels baked old-theme colors on desktop theme flip. Fix: write cache in Some arm + bump sites gen on change.
- **medium** — app.rs:166 (inserts app.rs:617 + app_fetch.rs:704; get app_render.rs:544) — scan_data HashMap grows without bound: one full L2 Scan (tens of MB) per site ever visited; no remove/retain/clear anywhere (verified). Contrast bounded RenderCache/LOOP_TEXTURE_BUDGET. Mobile OOM concern. Fix: evict sites no pane shows, or bound like RenderCache.
- **medium** — app.rs:573 — poll_data_channels drains at most ONE ScanResponse per frame (every other poller uses while let); multiple responses coalesce into one RedrawRequested; end-of-frame re-arm (app.rs:412-418) only for render-in-flight/auto-poll/loop → queued response can sit until unrelated OS event. Fix: while let.
- **medium** — app.rs:574-582 — stale-fetch discard path clears no UI state (Err path clears loading_site + fetching). Poll landing while SwitchRadarSite fetch in flight (that path never sets radar.fetching → poll gate doesn't protect) silently discards switch's result → loading_site spinner persists until next auto-poll volume. Fix: clear loading_site/fetching on stale discard, or don't bump generation for a check.
- **low** — app.rs:952-954 + input.rs:49-66 — Escape processed by app back-out funnel regardless of egui keyboard focus (event fed to InputHandler independent of egui consume). Escape in text field both unfocuses AND dismisses top layer/exits. Fix: skip funnel when wants_keyboard_input().
- **low** — app.rs:537 + app_fetch.rs:376 — last_viewport is a single global overwritten by every pane's RenderOverlay; with sync off, region fetches scoped to whichever pane processed last. Fix: per-pane viewport.
- **low** — app.rs:889-897 (install :812-818; ensure_rendering_state :440-455) — resumed() creates and installs new window unconditionally; double resumed → self.window replaced while state holds old surface (ensure_rendering_state won't rebuild). Unverified: unpaired resumed in winit 0.30 on supported platforms. Fix: guard if window.is_some(), or drop state on replace.
- **nit** — app.rs:930-931 — two distinct comments fused onto code lines in suspended() (bad merge). Restore.

## src/app_fetch.rs
- **medium** — app_fetch.rs:587-595 + app_render.rs:1267-1312, 933-948 — failed/empty scan listing leaves loop permanently in Rendering: comment says "Send empty list so UI can show error state" but accept_scan_listing sets phase=Rendering unconditionally (:1293), installs zero frames; update_loop_readiness (:941, 946) skips empty; any_loop_active() (ui.rs:1823-1831) false; nothing retries or surfaces error. Same terminal state when every frame retired render_failed. Loop hangs silently. Fix: deactivate loop on empty listing (or Error phase UI shows); promote all-failed loops out.
- **low** — app_fetch.rs:436-440 vs 451-453, 502-504 — early returns after marking render_in_flight=true for ALL pane_indices leave flags stuck on other panes (prepare_rasterize None path carefully clears, :476-484). Narrow reachability; asymmetry is the defect. Fix: clear marks in both early-return paths.
- **low** — app_fetch.rs:634/654/688 vs app.rs:631-632/716 — manual_nav_pending never cleared on failed navigation → next successful scan of any kind triggers spurious reinit_active_loops (all loops torn down). Fix: clear in Err + stale-discard arms.
- **low** — app_fetch.rs:156-162 — DST spring-forward gap: from_local_datetime().latest().unwrap_or_else(Local::now) → navigation jumps to NOW instead of ±1h of requested. Fix: fall back to timestamp ± standard offset.
- **low** — app_fetch.rs:647-683 — handle_navigate_one_scan never updates radar.config.timestamp (navigate_time :637-640 and jump_to_live :708-710/723-725 both sync) → time-picker drifts after stepping. Fix: update on adjacent-scan apply.
- **low** — app_fetch.rs:691-695, 729 — invalid pane index → pane_site unwrap_or_default → spawn_fetch(site: ""). Fix: let-else return.
- **nit** — app_fetch.rs:317-336 — SwitchRadarSite sync/non-sync branches duplicate identical 5-line body; only index range differs. Collapse.

## src/app_render.rs
- **medium** — app_render.rs:111 vs 123-132 (+ app.rs:398-405) — results applied AFTER gui.ui() built the frame; presented frame doesn't show new texture; another frame only if render-in-flight/auto-poll/loops → last static/overlay render with auto-poll off not presented until unrelated event (mouse move). Fix: poll channels before gui.ui(), or request redraw when a poller applied something.
- **medium** — app_render.rs:317-320 — poll_level3_results drains one response per frame while scan load spawns a dozen+ L3 fetches (same coalescing issue; every sibling uses while let). Fix: while let.
- **low** — app_render.rs:1008 — Duration::from_secs_f32(1.0 / loop_speed_fps) panics on non-finite/negative; load_ui_config assigns stored value directly at ui_config.rs:227 (ui_config.rs:129-133 guards only NaN/∞ at SAVE time; 0.0/negative from edited config passes). Fix: clamp here or at load.
- **nit** — app_render.rs:84-92 — zoom_factor uses width ratio only; height-only surface clamp would mis-scale vertically. Edge case.
- **nit** — app_render.rs:432-439 vs 450-453 — overlay result uploads texture before staleness check; stale-for-all-panes still costs full upload. Reorder.

## src/app_state.rs
- **nit** — :92, 109 — expect on request_adapter/request_device: GPU-less desktop dies with panic; app's only no-GPU path.
- **nit** — :128 — alpha_modes[0] would panic on empty list; select_surface_format above defends formats — inconsistent trust.
- **nit** — :137 + app_render.rs:90 — scale_factor dead state (init 1.0, never written; verified) → zoom_factor reduces to CSS ratio. Remove or wire.

## src/channels.rs
- **nit** — :49-52 — garbled doc sentence (phrase dropped mid-edit). No concurrency issues in hub.

## src/constants.rs
- **nit** — :27 — doc says "per pane"; LoopDownloadManager enforces globally (loop_downloads.rs:24-25). Fix comment.

## src/egui_renderer.rs / src/input.rs / src/lib.rs
No defects (input.rs: see app.rs Escape finding).

## src/loop_downloads.rs
- **low** — :154-159 + 109-111 (used at app_render.rs:710-716) — clear_all zeroes in_flight_count while old downloads still running; their responses decrement fresh counter → transient overshoot past MAX_CONCURRENT_LOOP_DOWNLOADS (e.g. 13 concurrent) right at site switch — exactly when mobile bandwidth protection matters. Fix: complete only when (site,ts) was in in_flight_set (complete_download returns whether removed).

## src/mobile_cfg.rs / src/offload.rs
No issues.

## src/platform.rs
- **nit** — :340-363 — poller_stops_sampling_after_exit timing-based/racy on loaded CI. Fix: settle-within-deadline loop.

## src/platform_double.rs
Double genuinely diverges where real bridges do (verified).
- **nit** — :168-179 — theme_channel() available on desktop variant whose real poll_theme is hardwired None; test could drive a path real desktop can't take. Fix: debug_assert(!reads_theme_itself).

## src/render_dispatch.rs
- **medium** — :371 (+ app_render.rs:147) — reset_panes_for_site bumps GLOBAL render_generation → every scan arrival for site A invalidates in-flight renders for site B panes; discarded + respawned = wasted 2048² render + value grid per cross-site poll, recurring every interval in multi-site layouts. Fix: per-site render generation (like fetch_generations) or site-stamped responses.
- **low** — :592 (doc), :651/:667 (returns) vs :706-708 (decline) — try_spawn_level3_render returns true even when spawn_render declined for budget (silent no-op at MAX_CONCURRENT_RENDERS); doc contract false; harmless only because sole caller (app_render.rs:537-543) ignores value. Fix: spawn_render returns bool, propagate.
- **nit** — :728 — pane_render[pane_idx] bare index; get_mut or debug_assert to document invariant.
- Note: check-then-increment on renders_in_flight safe only because increments main-thread-only; invariant real but unstated — worth a comment.

## build.rs / tests/wgpu_guard.rs / Cargo.toml
No issues.

## Systemic observations
1. Inconsistent channel-drain policy: render/overlay/loop pollers use while let; scan + level3 take one message per frame → wakeup-coalescing stalls. One policy (drain everything every frame) eliminates the class.
2. Wakeup model fall-through gaps (three verified): single-drain channels; results applied after gui.ui() with nothing re-arming; loops wedged in Rendering.
3. Generation/staleness architecture strong with two global choke points where per-site was meant: render_generation, get_scan_info() in auto-poll.
4. Bounded-memory discipline uneven: RenderCache/loop budgets careful; scan_data (largest per-entry object) unbounded.
5. Stale-discard paths drop responses without unwinding UI state the request set (loading_site, radar.fetching, manual_nav_pending); error paths do unwind. Audit every silent-discard site.
6. Source-probe tests pin formatting; deliberate, documented; fine.
7. Main-thread-only invariants load-bearing but implicit (renders_in_flight, texture free protocol). State them.

---

# Audit findings: nexrad-level3, rustdar-units, rustdar-gps, rustdar-platform

## nexrad-level3/Cargo.toml
- **nit** — :8 — repository points at danielway/nexrad upstream; misleading if published. Fix: point at rustdar repo or drop.
- **nit** — :11 — bzip2 pinned directly while siblings come from workspace deps. Cosmetic.

## nexrad-level3/src/lib.rs
- **nit** — :12 — `#![allow(clippy::too_many_arguments)]` but no fn exceeds 3 params. Delete.

## nexrad-level3/src/result.rs
- **low** — :24-40 — five of six error variants never constructed (only UnexpectedEof used); the failure modes they name are silently swallowed. Fix: construct at validation points or remove.

## nexrad-level3/src/decode/mod.rs
- **medium** — :44 — `symbology_offset as usize * 2` on untrusted u32 can overflow on wasm32 (CI compiles workspace for wasm32): debug panic / release silent wrap → misparse. Same class at radial.rs:226 (o + block_length) and symbology.rs:40 (o + layer_length). Fix: checked_mul/checked_add → Error.
- **low** — :67-80 — strip_wmo_header scans first ~100 bytes of ANY input for \r\r\n; binary header/PDB can legitimately contain 0x0D0D0A → valid data silently truncated. Fix: only strip when buffer starts with printable ASCII (SDUS/digit), or validate post-strip parse.
- **low** — :99-101, 141-143 — zlib/bz2 read_to_end with no output cap: decompression bomb → unbounded alloc (fed with downloaded bytes via rustdar-radar level3.rs:224). Fix: .take(MAX_PRODUCT_BYTES).
- **low** — :154-161 — BZ2 failure on PDB-declares-compressed product downgraded to log::debug and raw compressed tail parsed as symbology; DecompressionFailed variant exists unused. Fix: return the error (or warn).

## nexrad-level3/src/decode/header.rs
- **low** — :107 + symbology.rs:24,26,35 — block divider (must be -1), block ID (must be 1), layer dividers read but never validated; wrong offset decodes garbage instead of failing fast. Fix: validate, return existing variants.
- **nit** — :79-103 — halfword numbering convention mixed (local 1-51 vs ICD 10-60). Document convention.

## nexrad-level3/src/decode/radial.rs
- **high** — :393-396 — XDR radial count read as i32, cast to usize: negative (-1) sign-extends to ~2^64 → Vec::with_capacity "capacity overflow" panic/abort. Reachable from malformed/hostile downloaded bytes despite crate's no-panic posture. Fix: range-check 0..=MAX_RADIALS before cast.
- **high** — :424-437 — XDR array length i32→usize: arr_len=-1 → `arr_len * 4` debug-panics; release wraps so data_end = o-4 passes the `> len` check then `data[o..data_end]` panics "slice index starts at X but ends at Y". Panic in BOTH build modes from malformed data. Fix: u32::try_from + checked_mul/add + data_end >= o.
- **low** — :408,446 — num_bins unvalidated i32→usize→u16 truncation; never reconciled with gate_values.len(). Fix: clamp to gate_values.len(), error on negative.
- **low** — :60-68 vs 165-176 — legacy RLE truncates gate_values to num_range_bins; digital path never reconciles. Fix: truncate/pad in digital path too.
- **low** — :333-335 — invalid UTF-8 in XDR attr string → unwrap_or("") silently drops Scale/Offset for whole product (renders wrong physical values). Fix: from_utf8_lossy or warn.
- **nit** — :253 — `if let Some(ci) = (0..num_components).next()` obfuscated `if num_components > 0`. Simplify.

## nexrad-level3/src/decode/symbology.rs
- **low** — :50-62 vs 64-78 — inconsistent failure policy: malformed packet-28 → warn+skip layer; malformed packet-16/0xAF1F → abort entire product via `?`, discarding decoded layers. Pick one.
- **low** — :28, 40, 98-103 — block_length stored but never bounds the layer loop; garbage layer_length → confusing UnexpectedEof from next layer instead of InvalidSymbologyBlock. Fix: validate layer_end <= data.len().
- **nit** — :50 — legacy 0xAF1F packet pushed as DataPacket::DigitalRadial (name contradicts is_legacy: true). Rename variant.

## nexrad-level3/src/model/header.rs
- **low / unverified: ICD threshold encoding for product 94** — :137-152 — doc+test claim digital "codes 94+" store scale/offset as IEEE-754 float pairs; per ICD/MetPy product 94 encodes min×10/inc×10/count → would decode to 1.0/0.0 fallback (raw gate values as physical). Dormant (workspace fetches only 56/99/154). Fix: verify real N0Q/94 capture; if min/inc, add 94 to min_increment list (counterweight test at :331 pins the opposite).
- **nit** — :8 — "days since 1/1/1970" off by one (MJD 1 = Jan 1 1970 → epoch Dec 31 1969). Comment only.

## nexrad-level3/src/model/raster.rs
- **low** — :5-8 — RasterPacket { _private: () } stub never constructed; DataPacket::Raster unreachable dead code all consumers must match. Delete until Phase 2.

## rustdar-units
- All conversion constants verified correct to 6 sig figs. No conversion defects.
- **low** — timezone.rs:32-34 (and :51) — comment claims %Z renders "CDT"/"EST"; chrono formats FixedOffset Display = "+HH:MM" (verified chrono 0.4.45 source). Users see numeric offset. Fix: accept numeric + fix comment, or chrono-tz + iana-time-zone.
- **nit** — label() capitalization inconsistent across sibling enums ("mph","knots","in/hr" vs "Kilometers","Feet","Inches") feeding same settings UI.
- **nit** — zero tests in whole crate whose job is conversion constants. Add round-trip/known-value tests.

## rustdar-gps/Cargo.toml
- **low** — :13 — nmea unconditional but only consumer is #[cfg(feature="serial")]; no-default-features builds (wasm/iOS/Android) compile whole nmea crate for nothing. Fix: optional = true + dep:nmea in serial feature.
- **low** — :15 — serde_json declared, never used (grep verified). Remove.

## rustdar-gps/src/nmea_parser.rs
- **low** — :52-58 — GGA-before-RMC date falls back to host "today" (Utc::now): around UTC midnight fix stamped ~24h future; trusts host clock over receiver. Fix: shift ±1 day when delta > 12h, or timestamp None until RMC.
- **nit** — :49 — num_of_fix_satellites Option<u32> truncated `as u8` (≥256 wraps). Fix: min(255).
- Checked, NOT a defect: nmea 0.7 merge overwrites lat/lon with None on fix loss → no stale-position leak.

## rustdar-gps/src/serial.rs
- **high** — :109 with 153,175,189,222 — Stop mechanism never fires: `_stop_signal` is a Sender<()> nothing sends on; every check is `stop_rx.try_recv().is_ok()` which is false while alive (Err(Empty)) AND after drop (Err(Disconnected)). Dropping SerialGpsReader (the documented stop, relied on by platform.rs:134 stop_gps) never stops the thread. Quiet/unfixed receiver leaks thread holding the port open — serialport 4.9 opens Unix ports exclusive (TIOCEXCL, verified in serialport source posix/tty.rs:131-136) → subsequent start_gps on same port fails to reopen every 5s forever. Fix: treat Err(Disconnected) as stop (`!matches!(try_recv(), Err(Empty))`), or explicit Drop sending ().
- **medium** — :115-127 (with 86-104) — start runs detect_gps_ports + detect_baud synchronously on caller thread; detect_baud blocks up to ~30s (4 bauds × 5 read_lines × 1.5s + opens). Caller is GUI action handler on main thread (app_fetch.rs:246) → enabling GPS can freeze UI tens of seconds. Fix: move detection into the spawned thread.
- **low** — :138 — .expect on thread spawn in a library fn otherwise returning Option. Fix: log + None.
- **low** — :195-216 — read_line into String returns InvalidData on non-UTF-8 → treated as disconnect (break + 5s reconnect); u-blox/SiRF binary-interleaved receivers cycle open/close. Fix: read_until raw + lossy, or treat InvalidData like timeout.
- **low** — :115-121 with 31-82 — auto-detect falls back to FIRST serial port of any kind (Arduino/debug UART polled forever); BluetoothPort skipped entirely so BT GPS pucks never in picker. Fix: auto-start only VID-matched; reconsider BT exclusion.
- **low** — :19 — SiRF VID entry 0x4292:0x0603 very likely wrong: 0x4292 is absent from usb.ids and SiRF's VID is 0x0541 — dead weight. Fix: correct or remove.

## rustdar-gps/src/types.rs
- **low** — :59-66 — from_lat_lon no range validation, unconditionally stamps FixQuality::Gps; JNI/web bridges can inject out-of-range coords as genuine fix. Fix: validate (Option) or debug_assert.

## rustdar-platform/src/platform.rs
- **high (cross-ref)** — :21, 134-139 — stop_gps relies on "dropped to stop" which is false given the gps bug; stop→start cycles fail to reopen port while UI logs success. Fix: fix rustdar-gps; platform code then correct.
- **nit** — :46-60 — macOS lands in ~/.config//~/.cache instead of ~/Library/...; works, violates convention.
- cfg coverage checked OK (wasm32 → DesktopPlatform degrades gracefully; IosPlatform no-ops documented).

## rustdar-platform/src/run.rs
- **low** — :4 — EventLoop::new().unwrap() panics inside the fn whose caller exists to report startup errors cleanly (no display server → panic not eprintln path). Fix: make fallible.
- **low** — :24-36 — panic hook suppresses ANY panic whose payload contains "EventLoopClosed" (genuine bugs embedding that string become undiagnosable; still unwinds, report silenced). Fix: also require non-main thread, or log suppressed payload at debug.

## rustdar-platform/src/config_store.rs
- **low** — :17-19 — path_for interpolates key into filename unsanitized; `/` or `..` escapes config dir. Keys internal today; trait public. Fix: reject separators.
- **low** — :47 — fs::write not atomic; torn ui.json on crash loses all settings. Fix: tmp + rename.

## rustdar-platform/src/network_security_config.rs
No defects. (Nit: DomainRule::covers allocates per call; test-only.)

## lib.rs, main.rs
No defects.

## Systemic observations
1. Unvalidated integer-width casts at binary-decode boundary (nexrad-level3): helper layer bounds-safe but derived quantities (i32 as usize ×3, u32 as usize ×3) unchecked — all three panics are one pattern; a read_len(data, o, max) helper eliminates the class.
2. Log-and-continue error swallowing is house style in decode/IO; five never-constructed error variants; malformed data nearly always renders-as-garbage rather than reported.
3. mpsc drop-as-signal misuse: try_recv().is_ok() cannot observe disconnection; platform comment repeats the false claim. Standardize on matching Err(Disconnected).
4. kt→m/s 0.514444 duplicated in units + gps + radar (render.rs:1123) (cross-ref MEDIUM-4).
5. Test asymmetry: nexrad-level3 model layer exceptionally tested, decode layer zero in-crate tests; units and gps zero tests. All panics + stop bug live in untested halves.
6. cfg "desktop" = "not android/ios" silently includes wasm32; degrades gracefully today; worth explicit not(wasm32) or comment.

---

# rustdar-web audit findings

## src/lib.rs
No defects. cfg-gating matches Cargo.toml rationale.

## src/entry.rs
- **low** — entry.rs:39-40 (with geolocation.rs:45) — Geolocation watch starts unconditionally at boot: permission prompt fires on first load with no user gesture; on denial the app never re-asks (error callback only logs; channel stays empty forever). Fix: defer `start_watch` until a "locate me" action, or gate on `navigator.permissions.query` == granted and prompt only on gesture. (Partly deliberate per docs, but no-gesture prompt is a permissions-hygiene defect.)

## src/bridge.rs
- **nit** — bridge.rs:32-50 — `poll_theme` re-parses `matchMedia(DARK_SCHEME_QUERY)` every frame. Fix: store the `MediaQueryList` at construction, or listen for `change` and cache.
- Cross-checks verified: `drain_latest` at platform.rs:15; PlatformBridge method set matches; "honest Nones" comments accurate.

## src/config_store.rs
No defects. Err/Ok(None) folding matches ConfigStore trait contract (verified vs rustdar-egui config_store.rs:27-32); storage_key tests meaningful.

## src/geolocation.rs
- **nit** — geolocation.rs:20-35 — Browser supplies `position.timestamp` but `fix_from_coords` discards it; `GpsFix.timestamp` stays None. No current consumer (verified) — latent. Fix: map it in, or doc the intentional drop.
- **nit** — geolocation.rs:55,70 + Cargo.toml:73-76 — Uses legacy `web_sys::Position`/`Coordinates`/`PositionError`; spec-current names are `GeolocationPosition`/`GeolocationCoordinates`. Not deprecated in 0.3.103; future-proofing only.
- `Closure::forget()` (96-97) correct, bounded, documented — not a finding.

## index.html
- **low** — index.html:310-327 (status div 144) — If `./pkg/rustdar_web.js` 404s/fails to parse, the module script never runs a statement → try/catch never fires → page shows "Loading rustdar…" forever with no error. Fix: `window.addEventListener("error", ...)` updating `#rustdar-status` when filename is the module, or a watchdog timeout.
- **low** — index.html:3-27 — No CSP meta at all; two inline scripts. Fix: meta CSP with script-src 'self' + inline hashes, connect-src limited to data origins; or document omission.
- **nit** — index.html:5 — `user-scalable=no` a11y anti-pattern, ignored by iOS ≥10; `touch-action: none` (55) already owns gestures. Fix: drop it.
- **nit** — index.html:138-157 — No `<noscript>` fallback; JS-disabled shows "Loading rustdar…" forever.
- **nit** — index.html:6 — Title casing: `Rustdar` vs `rustdar` everywhere else. Pick one.
- Bootstrap logic (offline banner, update prompt, controllerchange guard, throttle, force-update hatch) verified correct against SW message protocol.

## sw.js
- **medium** — sw.js:100-101, 252-261 — Deploy-version probe HEADs only the wasm module; a deploy changing index.html/manifest/icons/CSS without changing wasm bytes yields an identical token and is never detected — cache-first navigations serve the old shell indefinitely (only `rustdarForceUpdate()` or a later wasm-changing deploy recovers). Fix: probe 2-3 assets (wasm + index), concatenate validator tokens.
- **medium** — sw.js:391-421 — `installShell` failure path deletes cache by name; on a rollback deploy (token seen before) `caches.open(name)` opens a pre-existing generation a live client may be pinned to; if `addAll` fails, `caches.delete(name)` destroys the in-use generation → mixed-shell hazard caused by the installer. Fix: check `caches.has(name)` before open, only delete on failure if it didn't pre-exist; or always install into unique name (as forceReinstall does).
- **low** — sw.js:555-573 (with 482-542) — `forceReinstall` awaits in-flight `updateCheck` but doesn't occupy the slot; a navigation-triggered `checkForUpdate` can interleave — each purge's keep-set omits the other's fresh cache; one can delete the other's new shell (self-heals via null path; wastes ~10MB install). Fix: forceReinstall assigns itself to `updateCheck` (cleared in finally).
- **low** — sw.js:66-68, 429-434 — Client pins don't survive an `SW_VERSION` bump: v3 `cachesToKeep` reads pins only from `rustdar-meta-v3`; first purge deletes every `rustdar-shell-v2-*` including generations still-open tabs are pinned to (reachable via skipWaiting + clients.claim seizing tabs) → later shell refetches mix generations. Fix: migrate pins from prior meta cache on activate before first purge.
- **low (unverified: WebKit resultingClientId support)** — sw.js:601 — On browsers not populating `resultingClientId` for navigations (historically Safari), no pin is ever taken; per-client atomicity silently degrades on exactly those browsers. Fix: verify WebKit; fallback worker-global pin.
- **nit** — sw.js:584-588 — Dead param: `serveShell(request, clientId, key)` — no caller passes `key`; `key ?? request` always `request`. Fix: drop it.
- **nit** — sw.js:666-667 — Comment typo: "so walkers sees an ordinary network error" — presumably "so callers see".
- **nit** — sw.js:668-671 — Tile cache miss fully buffers body into cache before returning (stale-revalidate path already does put inside waitUntil). Fix: `event.waitUntil(cacheTile(..., response.clone()))`, return immediately.
- **nit** — sw.js:220-222, 599-610 — Any navigation under ROOT (incl. direct visits to /pkg/rustdar_web.js, manifest) answered with cached index.html; surprising when debugging. Optional fix: check `isShellAsset(u)` before navigate rule.
- **nit** — sw.js:778-794 — Test hook `self.__rustdarSwInternals` ships in production worker; zero security exposure; acceptable.
- Verified correct: NEVER_CACHE_HOSTS exactly matches DataSources::production() (sources.rs:93-105); BASEMAP_HOST regex matches tiles.rs:59 URL; empty install handler deliberate; pin-before-probe ordering sound; token-null paths never delete a working shell.

## manifest.webmanifest
- **nit** — no `id` member; identity defaults to resolved start_url; moving deploy path forks installed-app identity. Fix: add `"id": "./"`. Everything else verified against files on disk and pinned by pwa_assets.rs.

## tests/sw_routing.test.mjs
Genuinely behavioral, not tautological. Gaps:
- **low** — 792-905 — Tile stale-while-revalidate flow never exercised end-to-end (stale served immediately, background refetch replaces, failed revalidation keeps stale). Fix: add the test.
- **low** — (absent) with sw.js:100-101 — No test models an HTML-only deploy (wasm ETag unchanged); `publishDeploy` always changes all ETags together, so the probe gap is structurally invisible to the suite. Fix: add publisher changing index only; pin intended behavior.
- **nit** — 313-322 — Cache-busting query test asserts only routeFor classification; `ignoreSearch` match never exercised end-to-end.

## tests/index_bootstrap.test.mjs
Behavioral. Gaps:
- **low** — Module script (init/start success/failure paths) has no test; only classic bootstrap executed. Fix: extract module script, stub init/start, pin removal-on-success and error text.
- **nit** — 57-61 — `extractBootstrap` takes first attribute-free `<script>`; adding another plain script above silently swaps what's tested. Fix: assert exactly one, or anchor on marker.

## tests/sw_harness.mjs
- **nit** — 507-517 vs sw.js:85-95 — `SHELL_ASSETS` hand-duplicates `SHELL_PATHS`; drift caught loudly but could derive from `worker.internals.SHELL_URLS`.
- **nit** — 412-414 — `clients.matchAll()` ignores options; currently harmless.

## tests/pwa_assets.rs
- **nit** — 204-230 — Only manifest-declared icons get dimension checks; favicon-32 (declared 32x32 in index.html:20) and apple-touch-icon existence-only. Fix: add size checks.
- **nit** — 37-45 — `without_line_comments` truncates at first `//` incl. inside string literals; currently safe. Fix: respect quotes or doc constraint.

## tests/sw_behaviour.rs
No defects. "every *.test.mjs is invoked" meta-test closes the forget hole.

## Cargo.toml
- **nit** — 52-53 — `js-sys` and `wasm-bindgen-futures` declared but never referenced in src/ (verified); function only as transitive pins; comment doesn't say so. Fix: state it or remove.

## Systemic observations
1. Overall quality high; sw.js + tests strongest part. All cross-crate contracts checked are exact matches.
2. Single-asset version probe is the main architectural soft spot; test deploy model can't see the gap.
3. Same-token cache-name reuse is the recurring sharp edge (two writers, one token-derived name).
4. SW_VERSION bumps are an untested migration path (pins/meta don't carry across; no test boots v(N+1) over v(N) storage).
5. Error paths degrade to console-only; eternal-"Loading" is the one users hit.
6. No TODO/FIXME markers; comments accurate.

---

# Audit: Mobile Shells (rustdar-android + ios)

## rustdar-android/src/lib.rs
- **high** — lib.rs:153-162 — Runtime permission request sends ACCESS_FINE_LOCATION alone; Android 12+ (API 31+, targetSdk 34) silently ignores FINE-without-COARSE requests — no dialog, no grant. Retry loop burns both bounded attempts (MAX_PERMISSION_REQUESTS=2, :327) on discarded requests → GPS off for life of install unless granted from Settings. Fix: 2-element array [FINE, COARSE] (both already in manifest :8-9).
- **medium** — lib.rs:71, 1004-1007 — `JAVA` is write-once OnceLock but `android_main` runs once per Activity (same file's EVENT_LOOP_PROXY doc :695-702 pins this and made the proxy replaceable for exactly that reason). Second Activity: JAVA.set silently no-ops → insets/moveTaskToBack/requestPermissions/density/theme all call the destroyed Activity; global ref pins it forever. Fix: Mutex<Option<JavaContext>> replaced at top of every android_main. Mitigation: common exit is process::exit.
- **medium** — lib.rs:330-345 — GPS fed only by polling `getLastKnownLocation`; nothing requests location updates. If no other app recently requested location, returns null every 10s forever. Fix: register LocationListener via requestLocationUpdates (Looper/getMainExecutor) or getCurrentLocation. Related: has_location_permission (:108-117) checks only FINE → "approximate only" grant on Android 12+ reads as unpermissioned.
- **low** — lib.rs:229-238 — provider loop `.ok()?` aborts whole function on JNI error mid-"gps" iteration, killing "network" fallback for that pass. Fix: continue to next provider.
- **low** — lib.rs:306, 866 — magic startup sleeps (sleep(3), sleep(4)) as synchronization; safe via retries but fragile. Fix: gate on JAVA.get()/COMPASS_CLASS.get() with short poll.
- **nit** — lib.rs:813-823 — nativeBackPressed has no panic guard (panic aborts process; path panic-free by construction; defensible). Option: catch_unwind → JNI_FALSE.
- Verified clean: JNI symbol/signature matches; jni 0.22.4 attach auto-detaches (verified in vendored crate); call macros catch exceptions; local frames bound refs; predictive-back funnel genuinely unified (d131f24 claim confirmed end-to-end vs frontend app.rs:838-841, 703, 727).

## rustdar-android/Cargo.toml
- **medium** — :64 — `rustdar-gps = { features = ["serial"] }` re-enables serialport stack for the whole Android graph, defeating rustdar-platform's explicit mobile exclusion (platform/Cargo.toml:39-48). lib.rs uses only GpsFix/FixQuality (no feature needed). Fix: drop features = ["serial"] (host-side copy at :74 already documents "without serial").
- **low** — :36 — `pollster` declared, never referenced. Remove.

## android/settings.gradle.kts / build.gradle.kts / gradle.properties
- No findings settings; AGP 9.3.1/Gradle 9.6.1 consistent (compat matrix unverifiable offline).
- **nit** — gradle.properties:4 — configureondemand incubating, no effect in 2-project build.

## android/app/build.gradle.kts
- **medium** — :252-254 — relative `storeFile` resolves against app/, not android/ as keystore.properties.example:54-55 documents: `file(storePath)` always returns absolute → `isAbsolute` always true → rootProject.file fallback is dead code. Following the template verbatim = "keystore not found" at signing. Fix: `val f = File(storePath); if (f.isAbsolute) f else rootProject.file(storePath)`.
- **low** — :110-114 — bundled-repo version via lexicographic maxOrNull(): "0.9.0" > "0.10.0". Latent (one version exists). Fix: numeric/semver compare.
- **low** — :86-94 — `cargo metadata` at configuration time on every Gradle invocation (incl. clean/sync); acknowledged tradeoff; friction point.
- **nit** — :226-227 — versionCode=1 hardcoded; versionName "0.1.0" is third unlinked copy of version (workspace Cargo.toml, project.yml).
- Verified clean: proguard rules match JNI surface exactly; jniLibs staging closes debug-.so-in-release hole; NDK pin coherent; minSdk 28/targetSdk 34/compileSdk 36 documented.

## AndroidManifest.xml
- **low** — :6 — ACCESS_NETWORK_STATE declared, nothing uses it (repo-wide grep); rustls-platform-verifier AAR verified to need no permissions (binary manifest + classes.jar checked) — removal is safe. Fix: remove.
- **low** — predictive-back path inert in shipped config: no `android:enableOnBackInvokedCallback`, targetSdk 34 → registered callback never fires; only legacy KEYCODE_BACK live. d131f24 funnel correct but zero on-device exercise until targetSdk 35. Option: add the manifest opt-in now.
- **nit** — :23 — icon is system placeholder sym_def_app_icon.
- Verified clean: exported=true required; cleartext policy coherent with NSC; uiMode load-bearing.

## network_security_config.xml
No findings; enforcement test exists (rustdar-platform/src/network_security_config.rs).

## CompassHelper.java
- **medium** — :64-124 — `register` not idempotent but caller runs once per Activity: each call registers a fresh never-unregistered ActivityLifecycleCallbacks and overwrites sListener. If new Activity resumed before register(), sListening true at overwrite → old listener stays registered forever (the exact everlasting-sensor drain the class header says was fixed). Fix: unregister existing sListener at top of register(); register lifecycle callbacks once (static guard).
- **medium** — :79-86 — azimuth never remapped for display rotation: getOrientation in natural frame; app supports landscape → heading wrong by ±90°/180° in landscape. Fix: SensorManager.remapCoordinateSystem keyed off Display.getRotation() before getOrientation.
- **low** — :83 — heading is magnetic, not true north; no declination correction anywhere (grep). Radar map vs true north → bias up to ~±15° CONUS. Fix: GeomagneticField.getDeclination() with last GPS fix. Unverified: whether frontend interprets heading as true or magnetic.
- Note: no low-pass filter exists (ROTATION_VECTOR already fused); SENSOR_DELAY_UI + 200ms poll coherent; -1f sentinel matches Rust side.

## BackHandler.java
No defects; funnel verified; "inert until targetSdk 35" point above.

## keystore.properties.example
- **medium** — :54-55 — documents storeFile resolution the build does not perform (counterpart of build.gradle.kts finding; whichever side is fixed, must agree).

## Wrapper / binaries / keystore
- **nit** — no distributionSha256Sum in gradle-wrapper.properties (validateDistributionUrl checks URL not content).
- Prebuilt .so files on disk (22.9/27.5 MB, debug per staging-bug comment) NOT tracked (ignored); inert at build; action: one-time `./gradlew clean`.
- rustdar.jks NOT tracked, never in git history (verified all refs).
- **medium** — Old keystore password IS in public git history cleartext: commit a9e64fd added `keystore_password = "rustdar"` to rustdar-android/Cargo.toml; repo is public. Build comments already declare it burned. Residual risk: keystore file leak (password public) + any published APK signed with it. Recommendation: generate new key per example, delete rustdar.jks from working tree, confirm no release artifact signed with burned key.

## ios/Makefile
- **low** — :28 — IOS_MIN ?= 14.0 is one of three unenforced copies (Info.plist:37, project.yml:19/31); comment says "Must match" but nothing checks. Fix: derive via yaml_val or grep assertion.
- **nit** — :101 — yaml_val is first-match sed over whole YAML; safe today; anchor to `^ *KEY:`.
- Verified clean: framework list matches project.yml; lib name matches; docker chown/digest pin/platform strings sound.

## ios/Info.plist
- **nit** — no app icon keys. Verified: NSLocationWhenInUseUsageDescription present; UIDeviceFamily 1+2.

## ios/project.yml
- **nit** — :39-40 — $(RUST_LIB_DIR) defined nowhere in file; xcodegen+Xcode build fails unless dev knows to supply. Fix: comment or settings.base default.

## ios/Sources/main.m + .gitignore
No findings; extern decl matches rustdar_ios_main ABI (verified platform/src/lib.rs:26).

## Systemic observations
1. Comment accuracy exceptional; two breaks: keystore.properties.example storeFile claim, JAVA OnceLock vs multi-android_main model (both Java helpers share this: register() written call-once, caller per-Activity).
2. Predictive-back funnel correct but unreachable in shipped config.
3. JNI hygiene strong (no leaks/UB found).
4. Error swallowing pervasive but deliberate; misleads only in location-permission path (the high finding).
5. Version/constant triplication: app version ×3, iOS minimum ×3.
6. Secrets posture good now; one historical wound (burned keystore password in public history, file still in working tree).

---

# Audit: repo root, CI, docs, build hygiene

## Cargo.toml (workspace)
- **medium** — Cargo.toml:66-68 — `[profile.dev] strip = true` strips debuginfo from dev builds, contradicting adjacent comment "preserve debuggability" (dev's `debug = true` generates debuginfo then throws it away). Fix: remove `strip = true` from dev (keep in release), or move/reword comment acknowledging un-debuggable dev builds.
- **nit** — Cargo.toml:16-18 — `[workspace.package]` lacks `license`/`rust-version`; only nexrad-level3 declares them. Fix: add `license = "MIT"`, `rust-version = "1.85.0"`, inherit in members.

## Member Cargo.toml hygiene
- **low** — rustdar-android/Cargo.toml:36 — `pollster` declared, zero references in src (verified). Fix: delete.
- **low** — rustdar-overlays/Cargo.toml:10 — `flate2` direct dep, zero references in src; decompression via hdf5-pure's `deflate` feature. Fix: remove, or comment it as a version anchor.
- **nit** — rustdar-web/Cargo.toml:53 — `js-sys` never referenced; anchor vs leftover indistinguishable. Fix: comment or remove.
- **low** — futures "=0.3.33" (rustdar-egui:36, rustdar-overlays:24), wasm-bindgen-futures "=0.4.76" (frontend:128, egui:63, web:52) duplicated member-local pins not hoisted to workspace. Fix: hoist to [workspace.dependencies].
- **nit** — nexrad-level3/Cargo.toml:8 — `repository` points at upstream danielway/nexrad. Unverified: upstreaming intent. Fix: point at rustdar repo or comment.

## rust-toolchain.toml
No defects. Observation: floating `stable` + clippy `-D warnings` = each new stable can redden CI on untouched PRs; autofix bot mitigates.

## README.md
- **medium** — README.md:3 — Release badge + link target `.github/workflows/release.yaml` which does not exist (only build/clippy/test). Badge renders "no status", link 404s. Fix: point at build.yaml or delete.
- **low** — README is 3 lines; all real docs live in .github/copilot-instructions.md where visitors won't find them. Fix: add description + build commands (corrected ones).
- **nit** — README.md:3 — two badges share alt text "Release". Fix: rename second "Latest release".

## data.md
Sampled claims verified against code; all held (unidata-nexrad-level3, noaa-hrrr-bdp-pds, SPC RSS MDs, IEM currents.json, GLM/HRRR ✅).
- **nit** — data.md:83 — stray `?` in table cell "ENTLN / Vaisala / Allison House?". Fix: drop or footnote.

## features.md
- **medium** — features.md:28 — HRRR marked ❌ for Rustdar; contradicts data.md:17 and shipped hrrr/ module. Fix: ✅.
- **medium** — features.md:75, 87 — GLM lightning marked ❌ in Satellite and Lightning tables; contradicts shipped glm/ module. Fix: ✅ both.
- **medium** — features.md:117 — "Web app" ❌ for Rustdar while rustdar-web is a full PWA deployed to Pages on every main push. Fix: ✅ (or Beta).
- **low** — features.md:118 — "Mobile: Android only" — CI builds an iOS .ipa; unverified whether iOS considered shipped. Fix: decide + update.

## .github/copilot-instructions.md (self-mandates being current; steers automated edits)
- **medium** — :11-21 — "seven crates" — workspace has 10; table omits rustdar-frontend, rustdar-web, rustdar-gps; attributes app.rs to rustdar-platform (it's in rustdar-frontend). Fix: regenerate from Cargo.toml.
- **medium** — :37 — "**No web target.** Native adapter limits only." — contradicted by rustdar-web + wasm32 CI + Pages deploy. Fix: replace with wasm32/WebGL2 constraints summary.
- **medium** — :35 — stale pin claim: says nexrad-data =1.0.0-rc.5, workspace pins rc.7. Fix: drop concrete numbers, point at [workspace.dependencies].
- **low** — :16, :62 — names ui_mobile.rs, ui_desktop.rs, rustdar-egui/src/geo.rs — none exist. Fix: refresh from git ls-files.
- **low** — :18 — "TGFTP Level III fetching" stale — sources.rs:15 documents tgftp as CORS-blocked; now unidata-nexrad-level3 S3. Fix: update.
- **low** — :31 — lint/unsafe claims drifted: platform uses deny(unsafe_code) not forbid (iOS entry needs it); "rustdar-android only unsafe crate" untrue; nexrad-level3 uses deny(unwrap_used/expect_used) not warn(clippy::all). Fix: restate per-crate.
- **low** — :43 — claims desktop needs `libasound2-dev`; nothing links audio; CI installs `libudev-dev` (serialport). Fix: change.

## .github/workflows/test.yaml
- **medium** — test.yaml:5, 26-31 — triggers on pull_request but unconditionally mints App token from secrets; fork PRs have no secrets → red run for every external contributor before any test (same in clippy.yaml:23-28). Fix: gate token step on same-repo, fall back to GITHUB_TOKEN; or document forks unsupported.
- Positive: no silent passes; coverage paths line up; actions SHA-pinned.

## .github/workflows/clippy.yaml
- **medium** — clippy.yaml:2-4, 29-31, 119 — On pull_request runs, checkout is detached PR merge ref; git-auto-commit-action then asked to commit autofixes — fails or targets wrong ref. Unverified: v7 exact detached-HEAD behavior. Fix: drop pull_request trigger (push covers all branches) or skip auto-commit on PR events.
- **low** — clippy.yaml:2-4 (also build.yaml:10-11) — `on: push` (all branches) + `pull_request` double-runs every same-repo PR (different concurrency groups; build.yaml = 11-row matrix twice). Fix: restrict push to main (as test.yaml) or drop pull_request.
- **nit** — clippy.yaml:6-8 vs 20-21 — workflow-level `pull-requests: read` dead (job-level permissions replace). Fix: delete.
- Verified non-findings: `clippy --fix || true` fenced by later `-D warnings` run; wasm32 gate coherent with b709c13.

## .github/workflows/build.yaml
- **low** — build.yaml:88-92 vs clippy.yaml:105 — Cross-workflow contradiction on the wasm32 gate: build.yaml comment deletes old wasm32 row because member-rooted resolution compiled rustdar-gps+serialport for wasm32 ("Do not restore the row"), yet clippy's widened `cargo check --workspace --target wasm32` is member-rooted and re-enables exactly that. Green but contradictory. Fix: reconcile comments, or `--exclude rustdar-gps` + separate `-p rustdar-gps --no-default-features` check.
- Verified sound: container pinned tag+digest; RUSTUP_TOOLCHAIN override matches toolchain doc; build-success aggregation handles skipped/cancelled; Pages deploy restricted to push@main; SHELL_PATHS parser matches sw.js formatting.

## .github/scripts/check-relative-paths.py
- **low** — :32-33 — "html attribute" rule includes bare `content`/`data` keywords and runs against .js/.mjs; `const data = "/x"` flagged → deploy fails (false positives). Fix: split rules by file type or require tag context.
- **low** — :54-55 — `ADDALL_ABS` only matches '/" quotes; template-literal `caches.addAll([`/x`])` passes unflagged (false negative in the critical check). Fix: add backtick to quote class.
- **nit** — :90-94 — RULES applied line-by-line; multi-line constructs missed; ADDALL pass demonstrates whole-text technique. Fix: whole-text with computed line numbers.
- **nit** — :85 — unreadable-file branch appends un-relativized (root-prefixed) path; others use root-relative. Fix: use rel.

## .github/renovate.json
- **low** — :15-21 — egui group omits `winit` (pinned =0.30.13), whose major bumps must travel with egui-winit; lone winit PR dead on arrival. Fix: add winit (and web-time?).
- Verified: gitIgnoredAuthors matches clippy bot email.

## coverage-baseline.tsv + badges/coverage.svg
Consistent: 19500/28895 = 67.49% vs badge 67.5%. No finding.

## LICENSE
MIT, 2025 Jacob McSwain. No finding.

## Repo hygiene (git) — all clean, verified
194 tracked files; rustdar.jks ignored via .gitignore:6 (*.jks), untracked; jniLibs .so ignored deliberately; coverage/, lcov.info, target-ios, .claude/worktrees/, generated .cargo/config.toml all ignored; no untracked strays; Cargo.lock tracked (correct).

## Systemic observations
1. Docs rot concentrates in copilot-instructions.md (most stale; consumed by codegen tools → consequential) and features.md; data.md accurate.
2. CI comment quality unusually high; single incoherence is wasm32-gate philosophy between build.yaml and clippy.yaml.
3. Duplicated ~5-step setup block across test.yaml and clippy.yaml (already subtly drifting: `rustup default stable` vs `rustup show`). Fix: composite action.
4. Fork-PR posture undefined (the two token-minting PR workflows — test, clippy — hard-require org secrets; build.yaml runs on fork PRs fine).
5. Version pinning disciplined; gaps: member-local duplicate pins, winit renovate omission.
6. No workflow_dispatch on any workflow — no manual re-run/deploy without a commit (nit).

---

# Cross-package duplication & consistency audit

## HIGH-1 — Full S3 ListObjectsV2 client implemented twice, divergent, two XML parsers
- radar/archive.rs:121-294 (list_url, parse_list_page, collect_keys w/ MAX_LIST_PAGES cap; xml=1.3.0) vs overlays/glm/fetch.rs:347-427, 1177-1191 (hand-rolled URL building incl. hand-rolled percent-encoding, roxmltree=0.21.1, uncapped page loop).
- Divergent behavior: archive captures <Key> only inside <Contents> and hard-errors on truncated-page-no-token; GLM scans ALL descendants named Key and warn-and-breaks (silently accepts incomplete listing).
- Fix: make radar's list_url/collect_keys pub (overlays already depends on radar), delete GLM copy, drop roxmltree.

## HIGH-2 — GOES bucket names have two authorities; fetch path bypasses the one validations derive from
- sources.rs:99-100 (noaa-goes19/noaa-goes18; module doc: "declared in one place") vs glm/mod.rs:31-37 GlmSatellite::bucket() hardcodes the same strings; fetch uses the enum. DataSources goes_*_bucket has ZERO production consumers — only its own test, Android NSC test (network_security_config.rs:112-121), PWA never-cache test (pwa_assets.rs:359-370).
- Risk: next satellite rotation edited in enum alone leaves NSC/SW validations validating stale hosts while real traffic goes undeclared.
- Fix: GlmSatellite resolves bucket through &DataSources, or delete enum bucket() and pass Source down.

## HIGH-3 — UserPreferences::temperature is a dead setting; every temperature display hardcodes °F
- Setting offered ui_settings.rs:63-68; TemperatureUnit convert_from_c/f have ZERO call sites. Hardcoded conversions: handlers/metar.rs:51,56; station_model.rs:58,69,134,136; hrrr/mod.rs:280-282 (own Kelvin variant, ignores prefs). Same popup correctly uses prefs.speed/prefs.height two lines away.
- Fix: route six C→F sites through prefs.temperature (add convert_from_k for HRRR).

## MEDIUM-1 — NWS alerts URL hardcoded (nws/fetch.rs:6), bypassing DataSources::nws_alerts_url() (sources.rs:187-189, production-dead; only its own test calls it). One of two fetchers in overlays taking no &DataSources (GLM also takes none); SPC has a test asserting no hardcoded origins; mock-server override design doesn't work for NWS. Fix: thread sources like spc/fetch.rs.

## MEDIUM-2 — Level II bucket two-authorities inside rustdar-radar: archive.rs:30 ARCHIVE_BUCKET (used by fetch) vs sources.rs:95-96 level2_bucket (used only by NSC/SW derivations + tests). level2_chunks_bucket has no production consumer at all. Fix: archive takes bucket from DataSources (level3.rs:171 already does).

## MEDIUM-3 — lat_rad_to_mercator_y byte-identical in three crates; 85.05 clamp in two
- radar/types.rs:29-32 (pub(crate)), egui/overlay_cache.rs:352-355, overlays/render/rasterize.rs:94-97; clamp: overlay_cache.rs:55 MERCATOR_LAT_LIMIT vs rasterize.rs:100 MAX_MERCATOR_LAT.
- web/lib.rs:61 documents the radar one by path — but it's pub(crate), which is WHY the others re-implemented it. Hit-testing in egui vs rasterization in overlays vs radar projection must all agree.
- Fix: make radar's fn + clamp pub; use everywhere (or rustdar-geo crate).

## MEDIUM-4 — Unit factors re-hardcoded across five crates
- kt→m/s 0.514444: gps/nmea_parser.rs:43, radar/render.rs:1123, radar/srm.rs:288 (as reciprocal const)
- m/s→kt 1.94384: egui/ui_map_pane.rs:723, hrrr/mod.rs:276
- m/s→mph 2.23694: radar/types.rs:27 (pub const MS_TO_MPH)
- F→C: metar/fetch.rs:53-57 (inside Fahrenheit newtype — newtype good, factor restated)
- m→ft 3.28084: handlers/metar.rs:116 (HeightUnit has no from-meters direction)
- hPa↔inHg 33.8639 (metar/fetch.rs:64-67) and 0.02953 ×2 (handlers/metar.rs:84, station_model.rs:89); no pressure module in rustdar-units
- Fix: export named pub consts / methods in rustdar-units; add from-meters to HeightUnit; small pressure helper; add rustdar-units dep to rustdar-gps.

## MEDIUM-5 — Fetch-layer error typing split: thiserror enums in radar (ArchiveError with deliberate NotFound/Status split, ScanError, Level3Error) + nexrad-level3; Result<_, String> throughout overlays (nws/spc/metar/hrrr; GLM's internal FileError flattened to String at boundary). thiserror not even in overlays' Cargo.toml. Fix: one OverlayFetchError (Transport/Status/Parse) or document string-errors as crate policy.

## LOW-1 — "Updated Xs/Xm ago" control-item block copy-pasted verbatim ×6: handlers/alert.rs:318-326, reports.rs:281-289, model.rs:250-258, metar.rs:371-379, discussion.rs:275-283, glm.rs:635-643 (refresh-button + "Fetching…" block also identical). egui's format_product_age is legitimately different. Fix: one updated_ago_item() in render/controls.rs.

## LOW-2 — 111.32 km/deg duplicated across crates + second formula for same thing inside radar (types.rs:50-51 vs egui ui_map_pane.rs:419 for the SAME 230 km range ring; render.rs:140 uses dy/EARTH_RADIUS_KM form). If types.rs switches form, ring and image drift. Fix: export km_to_lat_deg() from radar.

## LOW-3 — Dep pins duplicated per-crate: futures =0.3.33 (egui, overlays), wasm-bindgen-futures =0.4.76 (egui, frontend, web). Fix: hoist to workspace.dependencies. (hdf5-pure/grib local pins correctly local.)

## NIT-1 — Android builds GpsFix by struct literal (android/lib.rs:284-294) while web's twin documents exactly that as the drift hazard (geolocation.rs:27-34 uses ..GpsFix::from_lat_lon). Android needs non-default FixQuality::Estimated so can't use from_lat_lon as-is. Fix: GpsFix::with_quality(lat, lon, quality) used by both.

## NIT-2 — handle_back body repeated verbatim across four PlatformBridge impls, config_store across three (platform.rs Desktop :82-89,111-115; Android :208-215,263-267; iOS :375-382,406-410; platform_double.rs TestBridge handle_back :193-200 — its config_store :245-249 differs (SharedStore)). Fix: default trait methods over two accessors.

## Checked, OK
- config_store: NOT duplication — egui defines the ConfigStore trait; web + platform implement it. Shared trait already exists.
- HTTP client construction unified through radar tls::{client, simple_client, client_for} everywhere.
- SPC/METAR/HRRR/L3 URL building all thread DataSources (SPC has no-hardcoded-origin test).
- Retry/backoff: none exists anywhere to duplicate; single-shot by design.
- Haversine: single impl (egui ui_map.rs:340-352); metar/networks.rs:38 flat approx is deliberate.
- User-Agent single (tls.rs:49). Basemap tile URL single (tiles.rs:59); NSC/SW derive from it.
- GeoBounds defined once; radar ImageBounds is a different concept.
- Timezone formatting centralized in rustdar-units/timezone.rs, actually used by egui + overlays.
- Test harnesses target different layers; no reimplemented pattern.
- Platform abstraction clean; no cfg(target_os) leaks into frontend/egui logic.
- Per-service timeouts (archive 300s, HRRR 120s, METAR 60s, SPC 30s, tiles 20s, shared frontend 30s) different on purpose, each declared once.

## Systemic observations
1. DataSources is the right design, partially adopted: real for SPC/METAR/HRRR/L3; broken by NWS, GLM, L2. Stakes: Android NSC + web SW never-cache lists derive from DataSources, so every bypass is a fetch the validations cannot see. Generalize SPC's "no URL bypasses declared origin" test across crates.
2. rustdar-radar is the de-facto shared-infra crate (TLS, origins, Mercator, Earth radius, S3 listing) but keeps reusable pieces pub(crate) — direct cause of mercator triplication and GLM listing rewrite. Publish symbols or split rustdar-geo/rustdar-net.
3. rustdar-units adoption display-path-inconsistent; a grep-guard test in rustdar-units (no `9.0 / 5.0`, `1.94384` outside it) would stop re-multiplication.
4. Overlay handlers carry heavy per-handler boilerplate (refresh + fetching + updated-ago + timeout const + client builder); shared controls-builder ends the 5× blocks.
5. GLM downloads NetCDF batches through frontend's shared 30s client (handlers/glm.rs:515) while HRRR judged 30s too short for GRIB and builds its own 120s client — worth a deliberate decision.
