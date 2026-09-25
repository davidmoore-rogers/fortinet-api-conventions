# FortiSwitch / FortiAP local web UI over HTTPS — the firmware-upgrade protocol

The device's OWN web UI (not FortiOS REST on the gate, not FortiManager). Username +
password only; there is no API token. Every path below is what a browser session does when
an operator uploads and deploys an image by hand.

**Provenance and status.** Captured from browser sessions against real devices by the
standalone `fortiupgrade` tool and transcribed into a management application's HTTPS
engines. Neither has completed a bench validation on hardware with a console cable
attached. Treat the multipart field ORDER on the switch upload and the 202-vs-dropped-socket
rule on the AP as transcribed, not proven, until that test has run. A flash that fails
partway can leave a device unbootable: bench first.

## Transport, for both

- HTTPS on 443 (the standalone port; FortiLink-managed devices may not expose it at all).
  Factory or self-signed certificates — do not verify; some older builds need the
  `SSL_OP_LEGACY_SERVER_CONNECT` retry on "unsafe legacy renegotiation disabled".
- **Never follow redirects.** A 302 IS the answer on the switch login; following it turns a
  success into a GET of `/`. Keep a bare `name=value` cookie jar; the devices set no
  attributes worth honouring.
- Precompute `Content-Length` on the multipart upload and stream the file. The switch UI
  refuses a chunked body.
- A **dropped socket after the full body has been sent** is a normal outcome on the deploy /
  upgrade call, not a failure: the device has started flashing and tore the session down.

## FortiSwitch

| Step | Call | What matters |
|---|---|---|
| login | `POST /login` form `username`, `password`, `next_link=` | **The redirect TARGET is not the verdict.** Older builds answer a good password `302 → /`; FortiSwitchOS 7.6.6 answers it `302 → /login`, setting a quoted `APSCOOKIE_<n>` (value carries `AuthHash`) and an `ssession` cookie — and a BAD password there still gets an anonymous `ssession`. So: require a session cookie, then prove it with `GET /`. "Logged out" (redirect to `/login`, 401/403, or a 200 whose body contains `name="password"`) is the right test for every request AFTER login, and the wrong one for the login response itself. Verified on a real switch 2026-09-25. |
| identity | `GET /system/config/firmware/image?_=<ms>` | JSON `os_version`, `build`, `serial_number`, `model`, `hostname`, `admin_timeout` (minutes). **Never append `?cur_size`** — that variant answers about the STAGED upload, not the running image. Check the serial against your own record before touching anything. |
| stage | `POST /api/v2/execute/upload/file` multipart, fields IN THIS ORDER: `upgrade_from=undefined`, `firmware_version=undefined`, `file_size`, `size`, then the file as `file` | 2xx + JSON `status:"success"`. Never send `allow_firmware_downgrade_status`. Re-read the identity call afterwards: the serial must still match. |
| compat | `POST /system/config/firmware/compatible` form `action=check&account=true` | JSON `downgrade` / `check_signature` / `inc_adminpw` / `inc_snmppw` as STRINGS. Abort on `downgrade:"true"` or `check_signature:"false"`; `inc_adminpw` / `inc_snmppw` `"false"` are warnings (the admin / SNMP passwords may not carry over). |
| deploy | `POST /system/config/firmware/deploy` form `file_size=<n>` | Dropped socket = flashing started. |
| progress | poll the identity call | While flashing it also carries `erase_progress`, `write_progress`, `verify_progress`, `restart_progress`, `cur_step`, `tot_step`. **The four `*_progress` values are 0..1 FRACTIONS, not percent** — erase an exact 1 once done, the others long decimals (verified on FortiSwitchOS 7.6.6, 2026-09-25: read as percent, a finished erase shows "1%" and the next stage never looks started). `cur_step`/`tot_step` sit at 6/40 for the whole flash — useless for display. Ignore `msg` and `status`. The switch stops answering the moment verify completes, so the last poll often lands mid-verify. Roughly 60 s to the first progress line, erase ~3m45s, write ~3m30s, verify ~40 s on a 108F. One re-login is legitimate mid-flash. |
| reboot | poll `GET /login` | ANY HTTP answer = up. Not seeing it go down is not a failure (the window is short). |
| verify | fresh login + identity call until `os_version`/`build` equal the image | Retry on a 15 s interval; stop after two authentication rejections (the admin password did not carry over — see `inc_adminpw`). |
| abort | `POST /system/config/firmware/compatible` `action=reset` | Only between stage and deploy; clears the staged image. |
| logout | `GET /logout` | |

The `admin_timeout` on the identity call matters: if it is shorter than the upload takes
(image MB ÷ ~60 s per MB on a slow link), the session expires mid-upload. Warn, or raise it
first.

## FortiAP

| Step | Call | What matters |
|---|---|---|
| probe | bodiless `POST /logincheck` | 200 / 401 / 403 all mean the local UI is present. Connection refused or a timeout means it is disabled — the usual state of a FortiGate-managed AP — and nothing below will work; there is no fallback through the gate on this path. |
| login | `POST /logincheck` form `username`, `secretkey` | Cookie `FORTIPASS` = session. Some builds also return an `X-CSRF-TOKEN` header: echo it as a request header on every later call ONLY when it was issued. 401 = bad credentials, 403 = locked out. |
| identity | `GET /api/v1/sys-status` | `firmware_version` (e.g. `FP231K-v7.4.3-build0542`), `serial_number`, `hostname`. Cross-check the serial. |
| upgrade | `POST /api/v1/upgrade-image` multipart, ONE field `image` | **202 = accepted**; a dropped connection after the full body is also accepted; 400 = rejected. |
| reboot | poll `GET /api/v1/sys-perf` with the OLD session | Up until it answers 401/403 — that is the reboot, not an auth problem. **Never re-login during the wait**; a fresh login on a rebooting AP can wedge the UI. Interval ~10 s. |
| verify | fresh login + `/api/v1/sys-status` | A handful of retries 15 s apart. |
| logout | bodiless `POST /logout` | |

## Reading an image file

The first 512 bytes of a `.out` carry a token like `S108FF-7.06-FW-build1164-260709-patch08`
(FortiSwitch) or `FP231K-7.06-AP-build1105-260519-patch05` (FortiAP): **platform token**,
major.minor, product line (`FW` / `AP`), build, date, optional patch. `7.06` + `patch08` is
7.6.8. **The platform token is the first six characters of the serial numbers the image was
built for** — match on that, never on the model string (an FMG-discovered switch carries the
literal model "FortiSwitch" until an SNMP identity query fills it in, and models are
operator-editable; serials are neither). The file NAME (`FSW_108F_FPOE-v7-build1164-FORTINET.out`)
carries a marketing name and the major + build only; an image whose header cannot be read
should be stored but never offered.

Version compare: major → minor → patch → build, skipping a component missing on either side;
an indeterminate comparison is "not newer". Downgrades are never attempted from the outside —
the switch's own `compatible` check is the authority and a `downgrade:"true"` aborts.
