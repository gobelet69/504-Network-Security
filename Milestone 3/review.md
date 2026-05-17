# Milestone 3 - Implementation Review

This review compares `Milestone 3/Milestone 3.md` with the two files provided for implementation:

- `NEWFILE 1`: attack scripts under `/opt/ids-attacks`
- `NEWFILE 2`: real-time monitoring script `/opt/monitor_realtime.py`

## Executive Summary

The provided files cover a strong part of Phase 2 for malicious activity and real-time IDS monitoring. They also reuse protections from Milestone 2, because the attack scripts target the reverse proxy at `http://10.0.4.2`.

However, Milestone 3 is not fully complete yet. The main missing pieces are:

- normal activity simulation;
- post-mortem log analysis;
- final report generation;
- explicit NGINX logging configuration for IDS;
- documented validation procedure;
- presentation/demo preparation;
- evidence collection such as sample logs, screenshots, or Wireshark captures.

## What Has Been Done

### Phase 2.2 - Malicious Activity Scripts

Covered by `NEWFILE 1`.

| Milestone requirement | Status | Evidence from provided files |
| --- | --- | --- |
| Directory/admin probing | Done | `02_admin_probe.sh`, `06_404_scan.sh`, `07_common_sensitive_paths.sh` |
| Sensitive path testing | Done | `01_sensitive_files.sh`, `07_common_sensitive_paths.sh` |
| Dangerous HTTP methods | Done | `03_dangerous_methods.sh` tests `PUT`, `DELETE`, `PATCH`, `TRACE`, `OPTIONS` |
| Path traversal | Done | `04_directory_traversal.sh` |
| Exploit patterns | Done | `05_injection_patterns.sh` tests SQLi, XSS, command injection |
| Flood / DoS simple | Done | `09_rate_limit_burst.sh` |
| Large request test | Done | `08_large_post.sh` |

### Phase 2.4 - Real-Time Monitoring

Covered by `NEWFILE 2`.

| Milestone requirement | Status | Evidence from provided files |
| --- | --- | --- |
| Create `monitor_realtime.py` | Done | `/opt/monitor_realtime.py` |
| Tail NGINX access log | Done | Uses `/var/log/nginx/access.log` and follows new lines |
| Detect patterns in real time | Done | Detects 404 scan, 401/403, exploit patterns, flood, sensitive paths, dangerous methods |
| Colored alerts | Done | ANSI color class `C` |
| Categorize severity | Done | `NORMAL`, `INFO`, `WARNING`, `CRITICAL`, `EXPLOIT` |
| Final statistics on stop | Done | `show_stats()` runs on `Ctrl+C` |

### Reuse From Milestone 2

Milestone 2 already included several reverse proxy controls that support Milestone 3:

- access logs in `/var/log/nginx/access.log`;
- custom NGINX log format;
- blocked sensitive paths;
- blocked dangerous methods;
- request size limit;
- rate limiting;
- `/admin` restriction.

These are good foundations for IDS detection, because the attacks generate clear HTTP status codes and suspicious request strings in the access logs.

## What Is Not Done Yet

### Phase 1 - NGINX Logging Configuration

Partially done from Milestone 2, but Milestone 3 still needs explicit confirmation.

Missing items:

- verify that the deployed NGINX config still uses the detailed log format;
- document the exact NGINX logging block used for IDS;
- run `nginx -t`;
- prove that `/var/log/nginx/access.log` receives IP, timestamp, request, status, size, referer, and user-agent;
- optionally check log rotation.

### Phase 2.1 - Normal Activity Script

Not done in the provided files.

Milestone 3 requires normal browsing simulation so the IDS can prove it does not only detect attacks. A new `normal_activity.sh` should generate slow, legitimate requests such as `/`, `/about`, `/contact`, and normal downloads.

### Phase 2.3 - Post-Mortem Analysis Script

Not done in the provided files.

The real-time monitor detects live events, but Milestone 3 also asks for an offline analyzer that reads existing logs and produces a structured report:

- total requests;
- top IPs;
- repeated 404 scans;
- repeated 401/403 access failures;
- exploit patterns;
- flood/rate indicators.

### Phase 2.5 - Final Report Generator

Not done, optional.

This can be a small wrapper that runs the post-mortem analyzer and saves the result into a timestamped report file.

### Phase 3 - Tests and Validation

Not done as evidence.

The scripts exist, but the project still needs a validation matrix showing:

- each attack script was run;
- expected status codes appeared in NGINX logs;
- the monitor detected the attack;
- the post-mortem analyzer reported it;
- thresholds were adjusted if needed.

### Phase 4 - Documentation and Analysis

Partially done by this review and the implementation document, but still missing final proof artifacts.

Missing:

- screenshots or terminal outputs from the monitor;
- example logs before and after attacks;
- explanation of false positives and false negatives based on real test results;
- final IDS limitations and improvements tied to observed behavior.

### Phase 5 - Presentation

Not done in the provided files.

Missing:

- slides;
- 5 to 10 minute demo flow;
- backup plan;
- rehearsed timing.

### Phase 6 - Final Deliverables

Not complete yet.

The project still needs:

- all scripts copied to the reverse proxy/client and tested;
- sample logs;
- generated analysis report;
- final presentation PDF;
- demo backup files.

## Risks Found In The Provided Scripts

### Risk 1 - Hardcoded Target

All attack scripts use:

```bash
TARGET="http://10.0.4.2"
```

This is correct for the current topology, but it makes reuse harder. A safer version would allow:

```bash
TARGET="${1:-http://10.0.4.2}"
```

### Risk 2 - Brute Force Requirement Not Directly Covered

The milestone asks for repeated attempts on `/login` or `/admin` generating `401/403`. The provided scripts probe `/admin`, but there is no dedicated brute-force script with repeated credentials.

This should be added as `attack_bruteforce.sh` or covered by a new section in the implementation guide.

### Risk 3 - Monitor Counts 404s Over Whole Runtime

`monitor_realtime.py` counts total 404s per IP for the whole session. That is useful for a demo, but it can create false positives during a long run.

Improvement: count 404s inside a sliding time window.

### Risk 4 - Flood Detection Can Print Repeated Alerts

Once an IP passes the flood threshold, every new request can print another flood alert. This is acceptable for a demo, but noisy in longer monitoring.

Improvement: add cooldown per IP and alert type.

### Risk 5 - Real-Time Monitor Is Not A Full IDS

The implementation is a log-based IDS prototype. It does not inspect packet payloads directly, cannot detect encrypted HTTPS payload content after TLS termination unless NGINX logs it, and depends on log quality.

This limitation should be clearly stated in the final documentation and presentation.

## Completion Status By Milestone Phase

| Phase | Status | Notes |
| --- | --- | --- |
| Phase 1 - Preparation and logging | Partial | Logging exists from Milestone 2, but needs explicit IDS verification |
| Phase 2 - Scripts | Partial | Malicious scripts and real-time monitor done; normal activity and post-mortem missing |
| Phase 3 - Tests and validation | Not complete | Need recorded test evidence |
| Phase 4 - Documentation and analysis | Partial | This review and implementation guide help, but final evidence is still needed |
| Phase 5 - Presentation | Not started | Slides/demo flow still needed |
| Phase 6 - Final deliverables | Partial | Core scripts provided, but report/logs/slides missing |

## Recommended Next Work

1. Add `normal_activity.sh`.
2. Add a dedicated `attack_bruteforce.sh`.
3. Add `analyze_logs.py` for post-mortem analysis.
4. Add `generate_report.sh` as an optional report wrapper.
5. Confirm NGINX logging format on the reverse proxy.
6. Run the full test matrix and save logs.
7. Build final slides and demo script.

