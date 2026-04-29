# Company Monitoring Stack Report

**Generated:** 2026-04-27  
**Device:** Windows 11 Pro (company-managed)  
**Identity:** Trump@DC66.NET

---

## Identity & Management Layer

| Layer | Component | Detail |
|---|---|---|
| MDM | **Intune** (EnrollmentType 26 + 6) | UPN: `Trump@DC66.NET` — full device management enrolled |
| Co-management | **SCCM/WMI Bridge** | `WMI_Bridge_SCCM_Server` — classic enterprise policy push also active |
| AAD | `Microsoft.AAD.BrokerPlugin` | Azure AD SSO broker for all M365 logins |
| Push policy | `PushLaunch` + `PushRenewal` | 2 enrollment channels — MDM + MS DM Server |

**Device is double-enrolled (Intune cloud + SCCM on-prem). This is co-management — IT pushes policy from both systems simultaneously.**

---

## Active Monitoring Processes (running)

| Process | RAM | Role | Controllable? |
|---|---|---|---|
| `MsSense` | **370 MB** | Defender for Endpoint — EDR sensor | ❌ IT-mandated |
| `SenseNdr` | 52 MB | Network Detection & Response | ❌ IT-mandated |
| `SenseIR` | 19 MB | Incident Response component | ❌ IT-mandated |
| `SenseTVM` | 9 MB | Threat & Vulnerability Mgmt | ❌ IT-mandated |
| `IntuneWindowsAgent` | 51 MB | MDM policy enforcement | ❌ IT-mandated |
| `DiagTrack` (service) | — | Windows telemetry → Microsoft | ❌ GPO-locked |
| `ms-teams` × 2 | 54 + 30 MB | Two Teams instances running | ⚠️ Partially |
| `msedgewebview2` × 15+ | ~970 MB | WebView2 runtime (Teams + widgets) | ⚠️ Partially |
| `agent_ovpnconnect` | 2.4 MB | OpenVPN agent | Depends on IT |
| `crashpad_handler` × 2 | 7.5 MB | Crash reporting (Chrome/Edge/Teams) | ⚠️ Partially |

**Total monitoring/management RAM burn: ~1.5 GB**
(Sense suite + Intune + WebView2 from Teams alone)

---

## Scheduled Tasks — Telemetry & Policy Refresh

| Task | Purpose |
|---|---|
| `Intune Management Extension Health Evaluation` | Checks MDM compliance on schedule |
| `Intune Management Extension Client Cert Checker` | Validates client certificate |
| `PushLaunch` / `PushRenewal` × 2 | Re-registers MDM enrollment |
| `Automatic-Device-Join` | AAD join maintenance |
| `ExploitGuard MDM policy Refresh` | Re-applies attack surface rules |
| `BitLocker MDM policy Refresh` | Enforces disk encryption policy |
| `DmClient` / `DmClientOnScenarioDownload` | Feedback/telemetry data upload |
| `UsageDataReporting` / `UsageDataFlushing` | Windows usage stats push |
| `Office Performance Monitor` | Office telemetry |
| `Office Serviceability Manager` | Office update telemetry |

---

## Login Flow (Teams + Microsoft Group)

1. **AAD Broker** (`Microsoft.AAD.BrokerPlugin`) wakes up → validates token
2. **Intune** checks compliance → `PushLaunch` fires MDM policy refresh
3. **Teams** launches two processes + spawns 10+ `msedgewebview2` workers
4. **Sense** EDR logs the login event → uploads to Defender cloud

---

## Refactor Plan

### Cannot touch (IT-owned)

- Entire Sense suite: `MsSense` / `SenseNdr` / `SenseIR` / `SenseTVM`
- Intune agent + all scheduled MDM tasks
- DiagTrack (company GPO keeps it on)
- AAD Broker

### Can reduce (no IT conflict)

| Action | RAM Impact | Risk |
|---|---|---|
| Kill duplicate Teams instance — only one should run | -30 MB | Low |
| Disable Teams auto-start for duplicate process | -~200 MB WebView2 workers | Low |
| Switch to Teams Web (`teams.microsoft.com`) instead of desktop app | -~1 GB (all WebView2) | Low |
| Disable `crashpad_handler` via Teams/Edge settings | -8 MB | Low |
| Ask IT to clean up stale SCCM enrollment (2nd MDM channel) | Reduces policy conflicts | Needs IT |

### Biggest win

**Teams is the #1 memory hog at ~1 GB via WebView2 (15+ processes).**  
Use the browser version during meetings and kill the desktop app when not in calls.

---

## Bottom Line

This machine runs a full enterprise **EDR + MDM co-management stack** — ~500 MB locked for monitoring agents that cannot be removed. The actionable waste is Teams running double instances with 15 WebView2 processes. All Sense/Intune components are company-mandated and untouchable without IT escalation.
