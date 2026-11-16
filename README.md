# Clash / v2rayN / Hiddify / sing-box Error Codes — Causes and Fixes

A structured reference for the exact error strings these clients print, what each one actually means, and the shortest verifiable path to fix it.

**Why this exists:** most of these errors are reported as GitHub Issues with no resolution, or surface in a Reddit thread where the answer is "reinstall it." This repo maps each error string to the stage where it fails, so you stop guessing.

> Maintained alongside [节点手册 / jiedianshouce.com](https://jiedianshouce.com/) — a Chinese-language site that documents these same faults with test evidence. Each entry below links to the full walkthrough with step-by-step commands, screenshots, and stated evidence boundaries.

---

## Quick index

| Error string | Client | Fails at | Jump |
|---|---|---|---|
| `set system dns failed: exit code 1` | Clash Verge Rev | DNS rewrite | [#1](#1-set-system-dns-failed-exit-code-1) |
| `the remote profile data is invalid yaml` | Clash Verge Rev | Profile fetch/parse | [#2](#2-the-remote-profile-data-is-invalid-yaml) |
| `An attempt was made to access a socket in a way forbidden by its access permissions` | Clash Verge Rev | Profile activation | [#3](#3-an-attempt-was-made-to-access-a-socket) |
| `program not found` / repeated "Install Service" | Clash Verge Rev | Service install | [#4](#4-program-not-found--install-service-loop) |
| `service and cores` not satisfied | Clash Verge Rev | Runtime mode | [#5](#5-service-and-cores-not-satisfied) |
| IPC timeout on first core start | Clash Verge Rev | Core IPC handshake | [#6](#6-ipc-timeout-on-first-core-start) |
| System proxy on, no internet | Clash Verge Rev | Traffic takeover | [#7](#7-system-proxy-on-no-internet) |
| TUN on, no internet | Clash Verge Rev | TUN / routing | [#8](#8-tun-on-no-internet) |
| `failed to open geosite.dat` | v2rayN | Rule file load | [#9](#9-failed-to-open-geosite-dat) |
| `ArgumentOutOfRange_Count` | v2rayN | Subscription parse | [#10](#10-argumentoutofrange_count) |
| `Failed to import subscription content` | v2rayN | Import / gzip | [#11](#11-failed-to-import-subscription-content) |
| `运行Core失败` / `bind: address already in use` | v2rayN | Core start | [#12](#12-core-start-failed--bind-address-already-in-use) |
| `allowInsecure has been removed` | v2rayNG | TLS config migration | [#13](#13-allowinsecure-has-been-removed) |
| `unknown field "tunnel-per-resolver"` | Hiddify | Config schema | [#14](#14-unknown-field-tunnel-per-resolver) |
| `netip.ParsePrefix("/24"): ParseAddr(""): unable to parse IP` | Hiddify | WARP outbound build | [#15](#15-netipparseprefix24) |
| `exit status 127` | Hiddify | Linux TUN helper | [#16](#16-exit-status-127) |
| `HiddifyCli.exe requires elevation` | Hiddify | Windows privilege | [#17](#17-hiddifycliexe-requires-elevation) |
| `auto_redirect` → `operation not supported` | sing-box | Linux TUN / nftables | [#18](#18-auto_redirect--operation-not-supported) |
| Remote rule-set download failed | sing-box | Rule-set fetch | [#19](#19-remote-rule-set-download-failed) |
| `configuration file config.yaml is empty` | FlClash | Profile switch | [#20](#20-configuration-file-configyaml-is-empty) |

---

## Read this before you change anything

Three rules that apply to every entry below:

1. **Save the first error line, verbatim.** The first error is the cause; the ones after it are usually cascade. Copy it including the code and any quoted path.
2. **Change one variable per round.** Reinstalling, deleting the profile, and switching nodes at once destroys the evidence you need.
3. **Redact before you post.** Subscription URLs, tokens, node passwords, and unmasked logs are credentials. Hide them before pasting into an Issue.

None of these errors prove "the node is dead." Most of them fail before traffic ever reaches a node.

---

## Clash Verge Rev

### 1. `set system dns failed: exit code 1`

**What it means:** the client failed at the step where it rewrites the system DNS configuration. It does **not** prove the subscription, node, or Mihomo config is invalid — and it does not mean "change your DNS to 1.1.1.1 and it's fixed."

**Usually caused by:** another VPN taking over the network stack (Tailscale, WireGuard, corporate clients), the service mode not actually installed, or a stale leftover proxy setting from a previous session.

**Fix:**
1. Turn off TUN and system proxy. Confirm plain internet works without the client — this is your baseline.
2. Check for a second network-takeover tool running. Two clients both rewriting DNS will fight each other.
3. Record which layer failed: system proxy, TUN, or the whole network stack. Verify them separately, not together.
4. If the service mode shows not installed, install the service first — the DNS rewrite goes through it.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/clash-verge-rev-set-system-dns-failed-exit-code-1/)

---

### 2. `the remote profile data is invalid yaml`

**What it means:** the downloaded profile is not parseable YAML. The failure is in the fetch/parse stage, before any core starts.

**Usually caused by:** the subscription URL returning an HTML error page (login page, 404, or a Cloudflare challenge) with a 200 status; a base64 subscription being fed into a YAML parser; or a truncated download.

**Fix:**
1. Open the subscription URL in a browser and look at the first line. If it starts with `<`, you are receiving HTML, not a profile.
2. Check the content type: base64-encoded node lists must be converted, not imported as YAML directly.
3. Compare the byte size against what the provider claims. A 200 with 200 bytes is an error page.
4. Do not edit the local YAML copy to force it to load — that hides a fetch problem as a config problem.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/clash-verge-rev-remote-profile-invalid-yaml/)

---

### 3. `An attempt was made to access a socket in a way forbidden by its access permissions`

**What it means:** Windows denied a socket bind. On Clash Verge Rev this is most often the **Host Network Service (HNS)** holding the port range, not a firewall rule.

**Fix:**
1. Save the full error line first — this specific string matches the official FAQ exactly, and near-matches are a different fault.
2. Check HNS state (`hnsdiag` / service status). A hung HNS reserves ports and blocks the client from binding.
3. If the error instead reads `client error(Connect)`, you have a subscription-fetch problem — go to #2.
4. If the profile activates but there is no internet, go to #7 or #8.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/clash-verge-rev-cannot-select-profile-socket-error/)

---

### 4. `program not found` / "Install Service" loop

**What it means:** the privileged service binary is missing or was never installed. The client keeps asking for elevation because the service it needs isn't there.

**Fix:**
1. Check whether System Info still shows `Is Admin false` after you granted admin — if so the elevation itself failed.
2. If the error contains `program not found`, the service executable is absent (often removed by AV quarantine).
3. Distinguish "service install failed" from "service installed but TUN broken" — they have different fixes.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/clash-verge-rev-service-install-failed/)

---

### 5. `service and cores` not satisfied

**What it means:** the runtime mode requires a service + core combination that isn't present. Switching modes won't help until both halves exist.

**Fix:**
1. Confirm the core binary is present and matches the client version.
2. Reinstall the service after confirming the core, not before.
3. If the error appeared only after an upgrade, check for a leftover service from the previous version.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/clash-verge-rev-service-install-failed/)

---

### 6. IPC timeout on first core start

**What it means:** the GUI and the core service failed to finish their IPC handshake within the timeout — usually on the very first start, and usually **succeeds on retry**.

**Fix:**
1. Retry once. If the second attempt is clean and stays clean, you have a cold-start race, not a broken install.
2. If it recurs every launch, check for a second core process still holding the IPC channel.
3. Do not toggle several settings at once — you lose the ability to tell which one mattered.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/clash-verge-rev-service-core-ipc-timeout-retry/)

---

### 7. System proxy on, no internet

**What it means:** the takeover is on but traffic isn't reaching a working outbound. This is a chain problem: system proxy → local port → core → node → rules/DNS.

**Fix:**
1. Turn the takeover off. Confirm normal internet returns — that separates "proxy broke it" from "network was already broken."
2. Turn on **only** the system proxy and verify each of the four links in the chain separately.
3. Check what port the system proxy actually points at, and that the core is listening on that exact port.
4. If the same proxy works in another client, the fault is in this client's config or rules — not the node.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/clash-verge-rev-system-proxy-no-internet/)

---

### 8. TUN on, no internet

**What it means:** the virtual adapter or routing table is misconfigured. Frequently confused with DNS failure, which looks identical from the browser.

**Fix:**
1. Run a three-state comparison: takeover off / system proxy only / TUN on — same node and same rules each time.
2. Check service mode and privileges first; TUN needs the service.
3. Verify the virtual NIC actually exists after enabling TUN.
4. Check for a second default route or a multi-NIC conflict.
5. Separate "no network" from "DNS fails" — they need different fixes.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/clash-verge-rev-tun-not-working/)

---

## v2rayN

### 9. `failed to open geosite.dat`

**What it means:** the rule database file is missing, truncated, or unreadable. Two different causes hide behind one message.

**Fix:**
1. Read the **first** core error line. Missing file and EOF-during-read produce different follow-on errors.
2. If the file is absent, re-download the rule set rather than copying one from another install.
3. If the file exists but the error says EOF, the download was interrupted — delete and re-fetch.
4. Keep v2rayN and core versions consistent; mismatched rule formats fail the same way.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/v2rayn-geosite-dat-missing/)

---

### 10. `ArgumentOutOfRange_Count`

**What it means:** the subscription **content** was fetched successfully, then failed during parse/import/replace. The network is fine; the data or the parser is not.

**Fix:**
1. **Stop re-updating immediately.** Each retry may overwrite the last working list.
2. Confirm the content type of the subscription response before parsing again.
3. Run a minimal control: one subscription, one import, observe where it breaks.
4. Restore the node list from backup rather than clearing the database and config directory.

> Note: the official issue covering this (v7.25.6) is one specific case — it does not explain every empty-list report.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/v2rayn-argumentoutofrange-count-subscription-empty/)

---

### 11. `Failed to import subscription content`

**What it means:** the import step rejected the payload. Most often a gzip/encoding mismatch, not an invalid subscription.

**Fix:**
1. Check the `Content-Encoding` response header — a gzip body handed to a parser expecting plain text fails here.
2. Verify the response is not an HTML error page (same trap as #2).
3. Test the same URL with a different client to isolate parser vs content.
4. Do not paste the subscription URL into public issues — redact the token first.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/v2rayn-subscription-gzip-import-failed/)

---

### 12. Core start failed / `bind: address already in use`

**What it means:** "运行Core失败" is a **stage label**, not a diagnosis. Read the first core error underneath it.

**Fix:**
1. Record: v2rayN version, core name and version, OS, first core error, and the action that triggered it.
2. `bind: address already in use` → a local port conflict. Recover a single-core baseline before anything else.
3. Config validation errors → check v2rayN/core version compatibility; don't hand-edit the generated config unless you understand the generator.
4. Only after the core runs and a node is selected should you look at system proxy.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/v2rayn-core-start-failed/)

---

## v2rayNG

### 13. `allowInsecure has been removed`

**What it means:** the `allowInsecure` TLS option was removed from the schema. Old share links carrying it now fail validation, so the node never starts.

**Fix:**
1. Match the error string exactly — this is a schema removal, not a certificate error.
2. Do **not** work around it by disabling certificate verification. That trades a config error for a real MITM exposure.
3. Regenerate the share link / node entry without the removed field, using a current client.
4. Verify the node starts cleanly before testing connectivity.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/v2rayng-allowinsecure-removed-tls-config/)

---

## Hiddify

### 14. `unknown field "tunnel-per-resolver"`

**What it means:** the config contains a field the current core version doesn't recognize — a version skew between the config author and your installed core.

**Fix:**
1. Identify which step stopped: fetch, parse, or apply.
2. Check the core version against the version that introduced `tunnel-per-resolver`.
3. Let the client regenerate the config from the profile rather than patching the field by hand.
4. If it still fails after regeneration, the profile itself targets a newer core.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/hiddify-tunnel-per-resolver-unknown-field/)

---

### 15. `netip.ParsePrefix("/24")`

Full string: `error patching outbound ... netip.ParsePrefix("/24"): ParseAddr(""): unable to parse IP`

**What it means:** WARP outbound generation produced a prefix with no address in front of `/24`. This happens at **outbound build time**, before any traffic flows.

**Fix:**
1. Turn WARP off and confirm the original config still connects — establishes a recoverable baseline.
2. Rebuild the WARP config through the official flow, not by editing the generated outbound.
3. Change one variable per attempt; record client version and platform each time.
4. If it reproduces with a fresh WARP config, the account/registration state is the suspect, not the node.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/hiddify-warp-parseprefix-empty-ip/)

---

### 16. `exit status 127`

**What it means:** on Linux, the TUN helper command was not found — a missing binary or a PATH problem in the privileged context, not a broken config.

**Fix:**
1. Confirm which step failed before touching system files.
2. Check that the helper binary exists and is executable.
3. Do **not** copy shell commands from Issues blindly — they often assume a different distro layout.
4. When reporting, include distro, kernel, install method, and the exact command that returned 127.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/hiddify-linux-tun-exit-status-127/)

---

### 17. `HiddifyCli.exe requires elevation`

**What it means:** the Windows VPN mode needs administrator rights that the current process doesn't have. The client started, the privileged child did not.

**Fix:**
1. Confirm this is the same fault and not a TUN-configuration fault — the messages are similar.
2. Run the client elevated, and verify the mode switches to VPN/TUN afterwards.
3. Keep a rollback path: exit the client fully and confirm the network returns before retrying.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/hiddify-windows-tun-hiddifycli-elevation-required/)

---

## sing-box

### 18. `auto_redirect` → `operation not supported`

**What it means:** the nftables-based `auto_redirect` path isn't supported by the running kernel/netfilter stack. The config is valid; the platform isn't.

**Fix:**
1. Read the log to identify the failing stage — kernel module, nftables ruleset, or TUN setup.
2. Verify nftables is available and the required modules are loadable on this kernel.
3. Fall back to TUN without `auto_redirect` if the kernel doesn't support it — don't force the ruleset.
4. Stop troubleshooting once you confirm a hard platform limitation; it's not a config bug.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/sing-box-auto-redirect-nftables-operation-not-supported/)

---

### 19. Remote rule-set download failed

**What it means:** sing-box couldn't fetch a remote rule-set at startup. **First start with no cache** and **periodic update with cache** are different failures with different fixes.

**Fix:**
1. Back up the config and any working cache before changing anything.
2. Record the exact sing-box version — the download client changed across versions.
3. Check reachability of the specific rule-set URL from the machine, not from your browser.
4. Never delete entire routing sections or disable certificate checks just to make the core boot — that silently changes where traffic goes.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/sing-box-remote-rule-set-startup-failed/)

---

## FlClash

### 20. `configuration file config.yaml is empty`

**What it means:** the profile switch wrote an empty config. This does **not** prove the node is dead or the subscription URL is empty.

**Fix:**
1. Keep a restore point and get plain internet back first.
2. Check the FlClash version — a fix for the "missing current profile writes empty config" path landed in 0.8.97.
3. Confirm the current profile still exists before regenerating.
4. Separate "fetch succeeded" from "config generated" — they fail differently.
5. Restore advanced settings one at a time.

> Note: the official issue (FlClash 0.8.92, Windows 25H2) is one specific case and does not explain every occurrence of this message.

→ [Full walkthrough (ZH)](https://jiedianshouce.com/articles/flclash-config-yaml-empty-profile-switch/)

---

---

If this saved you a reinstall, star the repo — it's the only signal that tells other people the entry is worth reading.

## Related

- [Proxy client comparison matrix](https://github.com/overnice2025/proxy-client-matrix) — platform, protocol, TUN, rule system, license, price
- [Subscription format notes](https://github.com/overnice2025/subscription-format-notes) — base64 vs YAML, `subscription-userinfo`, conversion pitfalls
- [Full site index (ZH)](https://jiedianshouce.com/troubleshooting/)

## Contributing

If you hit one of these errors with a different root cause, open an issue with:

- the exact error string (redacted)
- client and core version
- OS and version
- what you changed immediately before it appeared

Corrections with a reproducible source are merged. Screenshots without a timestamp are not.

## License

MIT — see [LICENSE](LICENSE).
