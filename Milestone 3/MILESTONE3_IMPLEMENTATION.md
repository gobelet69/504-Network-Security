# Milestone 3 - IDS Implementation

This document describes the current Milestone 3 implementation for the IDS part of the 504 Network Security project.

The implementation is based on:

- the reverse proxy from Milestone 2 at `10.0.4.2`;
- NGINX access logs in `/var/log/nginx/access.log`;
- attack simulation scripts stored in `/opt/ids-attacks`;
- a real-time Python monitor stored at `/opt/monitor_realtime.py`.

Items marked `(new)` are additions required to make the provided files conform more completely to `Milestone 3/Milestone 3.md`.

## 1. Objective

Milestone 3 adds a log-based IDS prototype on top of the reverse proxy.

The goal is to:

- generate normal HTTP traffic;
- generate malicious HTTP traffic;
- detect suspicious behavior from NGINX logs;
- show alerts in real time;
- analyze logs after the attack;
- produce evidence for the final presentation.

## 2. Architecture

```text
External test client / attack host
        |
        | HTTP requests
        v
Reverse proxy NGINX - 10.0.4.2
        |
        | /var/log/nginx/access.log
        v
IDS scripts:
  - real-time monitoring
  - post-mortem analysis
  - final report generation
```

The IDS is log-based. It observes NGINX access logs and detects suspicious activity using indicators such as:

- repeated `404` responses;
- repeated `401` or `403` responses;
- dangerous HTTP methods;
- sensitive file paths;
- path traversal patterns;
- SQL injection patterns;
- XSS patterns;
- command injection patterns;
- high request rate;
- oversized request failures.

## 3. NGINX Logging Configuration

Milestone 2 already configured NGINX access logging. For Milestone 3, confirm that the reverse proxy contains a detailed log format.

Recommended NGINX block:

```nginx
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

access_log /var/log/nginx/access.log detailed;
error_log /var/log/nginx/error.log warn;
```

Validation commands:

```bash
nginx -t
tail -f /var/log/nginx/access.log
```

From another terminal:

```bash
curl -i http://10.0.4.2/
curl -i http://10.0.4.2/.env
```

Expected result:

- NGINX config is valid;
- `access.log` shows source IP, timestamp, request, status code, response size, referer, and user-agent.

## 4. Attack Scripts Provided

The following scripts are already covered by the files provided in the request.

Create the attack directory:

```bash
mkdir -p /opt/ids-attacks
```

Create all provided attack scripts:

```bash
cat > /opt/ids-attacks/01_sensitive_files.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== Sensitive files access test ====="

curl -i "$TARGET/.env"
curl -i "$TARGET/.git/config"
curl -i "$TARGET/nginx.conf"
curl -i "$TARGET/passwd"
curl -i "$TARGET/shadow"
curl -i "$TARGET/etc/passwd"

echo "===== End sensitive files test ====="
EOF

cat > /opt/ids-attacks/02_admin_probe.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== Admin area probing test ====="

curl -i "$TARGET/admin"
curl -i "$TARGET/admin/"
curl -i "$TARGET/admin/index.html"
curl -i "$TARGET/admin/login"
curl -i "$TARGET/admin/panel"

echo "===== End admin probing test ====="
EOF

cat > /opt/ids-attacks/03_dangerous_methods.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== Dangerous HTTP methods test ====="

curl -i -X PUT "$TARGET/"
curl -i -X DELETE "$TARGET/"
curl -i -X PATCH "$TARGET/"
curl -i -X TRACE "$TARGET/"
curl -i -X OPTIONS "$TARGET/"

echo "===== End dangerous HTTP methods test ====="
EOF

cat > /opt/ids-attacks/04_directory_traversal.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== Directory traversal test ====="

curl -i --path-as-is "$TARGET/../../../../etc/passwd"
curl -i --path-as-is "$TARGET/..%2F..%2F..%2F..%2Fetc%2Fpasswd"
curl -i --path-as-is "$TARGET/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"

echo "===== End directory traversal test ====="
EOF

cat > /opt/ids-attacks/05_injection_patterns.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== Injection pattern test ====="

curl -i "$TARGET/login?user=admin'OR'1'='1"
curl -i "$TARGET/search?q=union%20select%20username,password%20from%20users"
curl -i "$TARGET/comment?msg=%3Cscript%3Ealert(1)%3C/script%3E"
curl -i "$TARGET/test?cmd=cat%20/etc/passwd"
curl -i "$TARGET/test?exec=id"
curl -i "$TARGET/test?shell=/bin/bash"

echo "===== End injection pattern test ====="
EOF

cat > /opt/ids-attacks/06_404_scan.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== 404 scan test ====="

for i in $(seq 1 30); do
    curl -s -o /dev/null -w "scan /test$i -> %{http_code}\n" "$TARGET/test$i"
done

echo "===== End 404 scan test ====="
EOF

cat > /opt/ids-attacks/07_common_sensitive_paths.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== Common sensitive web paths test ====="

curl -i "$TARGET/phpmyadmin"
curl -i "$TARGET/wp-admin"
curl -i "$TARGET/wp-login.php"
curl -i "$TARGET/backup.zip"
curl -i "$TARGET/db.sql"
curl -i "$TARGET/dump.sql"
curl -i "$TARGET/config.php"
curl -i "$TARGET/server-status"
curl -i "$TARGET/actuator/env"
curl -i "$TARGET/cgi-bin/"

echo "===== End common sensitive paths test ====="
EOF

cat > /opt/ids-attacks/08_large_post.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== Large POST request test ====="

dd if=/dev/zero of=/tmp/bigfile-ids bs=2M count=1 >/dev/null 2>&1
curl -i -X POST --data-binary @/tmp/bigfile-ids "$TARGET/"

echo "===== End large POST request test ====="
EOF

cat > /opt/ids-attacks/09_rate_limit_burst.sh << 'EOF'
#!/bin/bash

TARGET="http://10.0.4.2"

echo "===== Burst / rate limiting test ====="

for i in $(seq 1 60); do
    curl -s -o /dev/null -w "%{http_code}\n" "$TARGET/" &
done

wait

echo "===== End burst / rate limiting test ====="
EOF

chmod +x /opt/ids-attacks/*.sh
```

### 4.1 Sensitive Files Access Test

File:

```text
/opt/ids-attacks/01_sensitive_files.sh
```

Purpose:

- tests access to `.env`;
- tests access to `.git/config`;
- tests access to `nginx.conf`;
- tests access to `passwd`, `shadow`, and `etc/passwd`.

Run:

```bash
/opt/ids-attacks/01_sensitive_files.sh
```

Expected IDS result:

- `SENSITIVE PATH` alerts;
- possible `403` or `404` responses depending on NGINX rules.

### 4.2 Admin Area Probing

File:

```text
/opt/ids-attacks/02_admin_probe.sh
```

Purpose:

- tests `/admin`;
- tests `/admin/`;
- tests `/admin/index.html`;
- tests `/admin/login`;
- tests `/admin/panel`.

Run:

```bash
/opt/ids-attacks/02_admin_probe.sh
```

Expected IDS result:

- `FORBIDDEN`, `REPEATED FORBIDDEN`, or `DIRECTORY SCAN` alerts.

### 4.3 Dangerous HTTP Methods

File:

```text
/opt/ids-attacks/03_dangerous_methods.sh
```

Purpose:

- tests `PUT`;
- tests `DELETE`;
- tests `PATCH`;
- tests `TRACE`;
- tests `OPTIONS`.

Run:

```bash
/opt/ids-attacks/03_dangerous_methods.sh
```

Expected IDS result:

- `DANGEROUS METHOD` alerts.

### 4.4 Directory Traversal

File:

```text
/opt/ids-attacks/04_directory_traversal.sh
```

Purpose:

- tests raw `../../../../etc/passwd`;
- tests encoded traversal using `%2F`;
- tests encoded traversal using `%2e%2e`.

Run:

```bash
/opt/ids-attacks/04_directory_traversal.sh
```

Expected IDS result:

- `Path Traversal`;
- `ATTACK DETECTED` after repeated exploit patterns.

### 4.5 Injection Patterns

File:

```text
/opt/ids-attacks/05_injection_patterns.sh
```

Purpose:

- SQL injection;
- XSS;
- command injection;
- suspicious shell parameters.

Run:

```bash
/opt/ids-attacks/05_injection_patterns.sh
```

Expected IDS result:

- `SQL Injection`;
- `XSS`;
- `Command Injection`;
- `ATTACK DETECTED`.

### 4.6 404 Scan

File:

```text
/opt/ids-attacks/06_404_scan.sh
```

Purpose:

- generates 30 requests to non-existing paths.

Run:

```bash
/opt/ids-attacks/06_404_scan.sh
```

Expected IDS result:

- `MULTIPLE 404`;
- `DIRECTORY SCAN`.

### 4.7 Common Sensitive Web Paths

File:

```text
/opt/ids-attacks/07_common_sensitive_paths.sh
```

Purpose:

- tests common exposed paths such as `phpmyadmin`, `wp-admin`, backup files, SQL dumps, `server-status`, `actuator/env`, and `cgi-bin`.

Run:

```bash
/opt/ids-attacks/07_common_sensitive_paths.sh
```

Expected IDS result:

- `SENSITIVE PATH`;
- `DIRECTORY SCAN` if many return `404`.

### 4.8 Large POST Request

File:

```text
/opt/ids-attacks/08_large_post.sh
```

Purpose:

- creates a 2 MB file;
- sends it as a POST body to the reverse proxy.

Run:

```bash
/opt/ids-attacks/08_large_post.sh
```

Expected IDS result:

- `LARGE REQUEST`;
- HTTP `413` if NGINX `client_max_body_size` is `1m`.

### 4.9 Burst / Rate Limiting

File:

```text
/opt/ids-attacks/09_rate_limit_burst.sh
```

Purpose:

- sends 60 parallel requests to the reverse proxy.

Run:

```bash
/opt/ids-attacks/09_rate_limit_burst.sh
```

Expected IDS result:

- `FLOOD / DOS`;
- possible `503` or `429` depending on NGINX rate limiting behavior.

## 5. Normal Activity Script (new)

Milestone 3 requires normal browsing traffic to prove that the IDS can distinguish normal behavior from attacks.

Create:

```bash
cat > /opt/ids-attacks/00_normal_activity.sh << 'EOF'
#!/bin/bash

TARGET="${1:-http://10.0.4.2}"

echo "===== Normal activity test ====="

curl -s -o /dev/null -w "GET / -> %{http_code}\n" "$TARGET/"
sleep 2

curl -s -o /dev/null -w "GET /about -> %{http_code}\n" "$TARGET/about"
sleep 2

curl -s -o /dev/null -w "GET /contact -> %{http_code}\n" "$TARGET/contact"
sleep 2

curl -s -o /dev/null -w "HEAD / -> %{http_code}\n" -I "$TARGET/"
sleep 2

curl -s -o /dev/null -w "GET /favicon.ico -> %{http_code}\n" "$TARGET/favicon.ico"

echo "===== End normal activity test ====="
EOF

chmod +x /opt/ids-attacks/00_normal_activity.sh
```

Run:

```bash
/opt/ids-attacks/00_normal_activity.sh
```

Expected IDS result:

- `OK`, `INFO`, or low-volume `404`;
- no `CRITICAL` alert unless the application does not contain the tested pages and thresholds are too low.

## 6. Dedicated Brute Force Script (new)

The provided admin probing script is useful, but Milestone 3 specifically asks for repeated login or admin attempts. Add a dedicated script.

Create:

```bash
cat > /opt/ids-attacks/10_bruteforce_login.sh << 'EOF'
#!/bin/bash

TARGET="${1:-http://10.0.4.2}"

echo "===== Brute force login/admin test ====="

for password in admin password 123456 qwerty letmein changeme test root; do
    curl -s -o /dev/null -w "login password=$password -> %{http_code}\n" \
        -X POST \
        -d "username=admin&password=$password" \
        "$TARGET/admin/login"
done

echo "===== End brute force login/admin test ====="
EOF

chmod +x /opt/ids-attacks/10_bruteforce_login.sh
```

Run:

```bash
/opt/ids-attacks/10_bruteforce_login.sh
```

Expected IDS result:

- repeated `403` or `404`;
- `REPEATED FORBIDDEN` if NGINX returns `403`;
- `DIRECTORY SCAN` if the route does not exist and returns `404`.

## 7. Real-Time Monitoring Script Provided

The provided monitor is:

```text
/opt/monitor_realtime.py
```

Create it on the reverse proxy, where `/var/log/nginx/access.log` exists:

```bash
cat > /opt/monitor_realtime.py << 'EOF'
#!/usr/bin/env python3
"""
IDS Monitoring en Temps Reel
Analyse les logs NGINX pour detecter des activites suspectes.
"""

import sys
import time
import re
from collections import defaultdict, deque
from datetime import datetime, timedelta

LOG_FILE = "/var/log/nginx/access.log"

SCAN_THRESHOLD = 8
BRUTE_FORCE_THRESHOLD = 5
FLOOD_THRESHOLD = 30
TIME_WINDOW = 30
EXPLOIT_ALERT_THRESHOLD = 2

class C:
    RED = '\033[0;31m'
    YELLOW = '\033[1;33m'
    GREEN = '\033[0;32m'
    BLUE = '\033[0;34m'
    BOLD = '\033[1m'
    NC = '\033[0m'

ip_stats = defaultdict(lambda: {
    '404': 0,
    'auth_fail': 0,
    'exploits': [],
    'sensitive': 0,
    'dangerous_methods': 0,
    'requests': deque(maxlen=500),
    'first_seen': None
})

def parse_nginx_log(line):
    pattern = r'(\S+) - \S+ \[(.*?)\] "([^"]+)" (\d+) (\d+) "([^"]*)" "([^"]*)"'
    match = re.match(pattern, line)

    if not match:
        return None

    return {
        'ip': match.group(1),
        'timestamp': match.group(2),
        'request': match.group(3),
        'status': match.group(4),
        'size': match.group(5),
        'referer': match.group(6),
        'user_agent': match.group(7)
    }

def extract_request_parts(request):
    try:
        method, url, protocol = request.split(maxsplit=2)
        return method, url, protocol
    except ValueError:
        return "UNKNOWN", request, "UNKNOWN"

def check_sensitive_paths(url):
    patterns = [
        r'\.env',
        r'\.git',
        r'passwd',
        r'shadow',
        r'nginx\.conf',
        r'backup',
        r'dump',
        r'config',
        r'wp-admin',
        r'phpmyadmin',
        r'server-status',
        r'actuator',
        r'cgi-bin'
    ]

    detected = []
    for pattern in patterns:
        if re.search(pattern, url, re.IGNORECASE):
            detected.append(pattern)
    return detected

def check_exploit_patterns(request):
    patterns = {
        'SQL Injection': r"(union\s+select|'OR'1'='1|' OR '1'='1|select.+from|drop\s+table)",
        'XSS': r'(<script|javascript:|onerror=|onload=|alert\()',
        'Path Traversal': r'(\.\./|\.\.\\|%2e%2e|%2f|etc/passwd|etc/shadow)',
        'Command Injection': r'(;cat|;ls|\|\s*cat|`|\$\(|cmd=|exec=|shell=|/bin/bash)',
        'LFI/RFI': r'(file://|php://|data://)'
    }

    detected = []
    for attack_type, pattern in patterns.items():
        if re.search(pattern, request, re.IGNORECASE):
            detected.append(attack_type)

    return detected

def check_suspicious_ua(user_agent):
    tools = [
        'nikto', 'sqlmap', 'nmap', 'masscan', 'dirbuster',
        'wpscan', 'burp', 'metasploit', 'acunetix', 'nessus',
        'python-requests', 'go-http-client'
    ]

    ua_lower = user_agent.lower()
    for tool in tools:
        if tool in ua_lower:
            return tool.upper()

    if 'curl' in ua_lower:
        return 'CURL'

    return None

def alert(level, category, message, details=None):
    icons = {
        'NORMAL': '[OK]',
        'INFO': '[INFO]',
        'WARNING': '[WARN]',
        'CRITICAL': '[CRIT]',
        'EXPLOIT': '[EXPLOIT]',
    }

    colors = {
        'NORMAL': C.GREEN,
        'INFO': C.BLUE,
        'WARNING': C.YELLOW,
        'CRITICAL': C.RED,
        'EXPLOIT': C.RED,
    }

    icon = icons.get(level, '[*]')
    color = colors.get(level, C.NC)
    ts = datetime.now().strftime("%H:%M:%S")

    print(f"{color}{icon} [{ts}] [{category}]{C.NC} {message}")

    if details:
        for key, val in details.items():
            print(f"   - {key}: {val}")

def analyze_request(data):
    ip = data['ip']
    status = data['status']
    request = data['request']
    ua = data['user_agent']

    method, url, protocol = extract_request_parts(request)

    now = datetime.now()

    if ip_stats[ip]['first_seen'] is None:
        ip_stats[ip]['first_seen'] = now

    ip_stats[ip]['requests'].append(now)

    recent_count = sum(
        1 for req_time in ip_stats[ip]['requests']
        if req_time > now - timedelta(seconds=TIME_WINDOW)
    )

    if recent_count > FLOOD_THRESHOLD:
        alert('CRITICAL', 'FLOOD / DOS',
              "High request rate detected",
              {
                  'IP': ip,
                  'Requests': f"{recent_count} in {TIME_WINDOW}s",
                  'Rate': f"{recent_count / TIME_WINDOW:.1f} req/s"
              })

    if method in ['PUT', 'DELETE', 'PATCH', 'TRACE', 'OPTIONS']:
        ip_stats[ip]['dangerous_methods'] += 1
        alert('CRITICAL', 'DANGEROUS METHOD',
              f"Blocked or suspicious HTTP method: {method}",
              {
                  'IP': ip,
                  'URL': url,
                  'Status': status
              })
        return

    sensitive = check_sensitive_paths(url)
    if sensitive:
        ip_stats[ip]['sensitive'] += 1
        alert('CRITICAL', 'SENSITIVE PATH',
              "Access attempt to sensitive resource",
              {
                  'IP': ip,
                  'URL': url,
                  'Status': status,
                  'Matched': ', '.join(sensitive)
              })
        return

    exploits = check_exploit_patterns(request)
    if exploits:
        ip_stats[ip]['exploits'].extend(exploits)
        exploit_count = len(ip_stats[ip]['exploits'])

        if exploit_count >= EXPLOIT_ALERT_THRESHOLD:
            alert('EXPLOIT', 'ATTACK DETECTED',
                  "Multiple exploit attempts detected",
                  {
                      'IP': ip,
                      'Types': ', '.join(set(ip_stats[ip]['exploits'])),
                      'Total': exploit_count,
                      'Request': request[:100]
                  })
        else:
            alert('EXPLOIT', 'EXPLOIT ATTEMPT',
                  f"Suspicious pattern: {', '.join(exploits)}",
                  {
                      'IP': ip,
                      'Request': request[:100]
                  })
        return

    tool = check_suspicious_ua(ua)
    if tool and status not in ['200', '301', '302', '304']:
        alert('WARNING', 'AUTOMATED CLIENT',
              f"Automated tool detected: {tool}",
              {
                  'IP': ip,
                  'User-Agent': ua[:80],
                  'URL': url,
                  'Status': status
              })

    if status == '404':
        ip_stats[ip]['404'] += 1
        count_404 = ip_stats[ip]['404']

        if count_404 >= SCAN_THRESHOLD:
            alert('CRITICAL', 'DIRECTORY SCAN',
                  "Massive directory scan detected",
                  {
                      'IP': ip,
                      'Total 404': count_404,
                      'Last URL': url
                  })
        elif count_404 >= 3:
            alert('WARNING', 'MULTIPLE 404',
                  "Multiple not found pages",
                  {
                      'IP': ip,
                      'Count': count_404,
                      'URL': url
                  })
        else:
            alert('INFO', '404', f"{ip} -> {url}")
        return

    if status in ['401', '403']:
        ip_stats[ip]['auth_fail'] += 1
        count_fail = ip_stats[ip]['auth_fail']

        if count_fail >= BRUTE_FORCE_THRESHOLD:
            alert('CRITICAL', 'REPEATED FORBIDDEN',
                  "Repeated access denial detected",
                  {
                      'IP': ip,
                      'Failed attempts': count_fail,
                      'URL': url
                  })
        else:
            alert('WARNING', 'FORBIDDEN',
                  "Forbidden access",
                  {
                      'IP': ip,
                      'Status': status,
                      'Count': count_fail,
                      'URL': url
                  })
        return

    if status in ['413', '429', '500', '503']:
        category = {
            '413': 'LARGE REQUEST',
            '429': 'RATE LIMIT',
            '500': 'SERVER ERROR',
            '503': 'RATE LIMIT / UNAVAILABLE'
        }.get(status, 'HTTP ERROR')

        alert('WARNING', category,
              f"Suspicious HTTP status {status}",
              {
                  'IP': ip,
                  'URL': url,
                  'Status': status
              })
        return

    if status in ['200', '304']:
        alert('NORMAL', 'OK', f"{ip} -> {method} {url}")
        return

    if status in ['301', '302']:
        alert('INFO', 'REDIRECT', f"{ip} -> {url}")
        return

    alert('INFO', f'HTTP {status}', f"{ip} -> {method} {url}")

def tail_log(filepath):
    try:
        with open(filepath, 'r') as f:
            f.seek(0, 2)

            print(f"\n{C.BOLD}{'='*70}{C.NC}")
            print(f"{C.BOLD}IDS REAL-TIME MONITORING{C.NC}")
            print(f"{C.BOLD}{'='*70}{C.NC}")
            print(f"Log file: {filepath}")
            print(f"Thresholds: Scan={SCAN_THRESHOLD}, Forbidden={BRUTE_FORCE_THRESHOLD}, Flood={FLOOD_THRESHOLD}/{TIME_WINDOW}s")
            print(f"Started: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
            print(f"{C.BOLD}{'='*70}{C.NC}")
            print("Press Ctrl+C to stop and see statistics.\n")

            while True:
                line = f.readline()
                if line:
                    parsed = parse_nginx_log(line.strip())
                    if parsed:
                        analyze_request(parsed)
                else:
                    time.sleep(0.1)

    except KeyboardInterrupt:
        print(f"\n\n{C.BOLD}{'='*70}{C.NC}")
        print(f"{C.BOLD}FINAL STATISTICS{C.NC}")
        print(f"{C.BOLD}{'='*70}{C.NC}\n")
        show_stats()
        sys.exit(0)

    except FileNotFoundError:
        print(f"{C.RED}Error: file {filepath} not found{C.NC}")
        sys.exit(1)

def show_stats():
    if not ip_stats:
        print("No activity detected.")
        return

    sorted_ips = sorted(
        ip_stats.items(),
        key=lambda x: len(x[1]['requests']),
        reverse=True
    )[:10]

    print(f"{C.BOLD}Top IPs:{C.NC}")

    for i, (ip, stats) in enumerate(sorted_ips, 1):
        print(f"\n{i}. {C.BLUE}{ip}{C.NC}")
        print(f"   Total requests: {len(stats['requests'])}")
        print(f"   404 errors: {stats['404']}")
        print(f"   Forbidden/auth failures: {stats['auth_fail']}")
        print(f"   Sensitive path attempts: {stats['sensitive']}")
        print(f"   Dangerous methods: {stats['dangerous_methods']}")

        if stats['exploits']:
            print(f"   {C.RED}Exploit attempts: {len(stats['exploits'])} ({', '.join(set(stats['exploits']))}){C.NC}")

        duration = (datetime.now() - stats['first_seen']).total_seconds()
        if duration > 0:
            rate = len(stats['requests']) / duration
            print(f"   Average rate: {rate:.2f} req/s")

    total_requests = sum(len(s['requests']) for s in ip_stats.values())
    total_404 = sum(s['404'] for s in ip_stats.values())
    total_exploits = sum(len(s['exploits']) for s in ip_stats.values())
    total_auth_fail = sum(s['auth_fail'] for s in ip_stats.values())
    total_sensitive = sum(s['sensitive'] for s in ip_stats.values())
    total_methods = sum(s['dangerous_methods'] for s in ip_stats.values())

    print(f"\n{C.BOLD}Global summary:{C.NC}")
    print(f"   Total requests: {total_requests}")
    print(f"   Total 404s: {total_404}")
    print(f"   Total forbidden/auth failures: {total_auth_fail}")
    print(f"   Total sensitive path attempts: {total_sensitive}")
    print(f"   Total dangerous methods: {total_methods}")
    print(f"   Total exploit attempts: {total_exploits}")
    print(f"   Unique IPs: {len(ip_stats)}")

if __name__ == "__main__":
    if len(sys.argv) > 1:
        LOG_FILE = sys.argv[1]

    tail_log(LOG_FILE)
EOF

chmod +x /opt/monitor_realtime.py
```

It detects:

- flood / DoS based on request count in a time window;
- dangerous HTTP methods;
- sensitive paths;
- SQL injection;
- XSS;
- path traversal;
- command injection;
- suspicious automated user-agents;
- repeated 404 directory scans;
- repeated forbidden/auth failures;
- large request and rate limit status codes.

Run:

```bash
/opt/monitor_realtime.py
```

Optional custom log file:

```bash
/opt/monitor_realtime.py /var/log/nginx/access.log
```

Stop:

```text
Ctrl+C
```

Expected result:

- live colored alerts;
- final statistics after stopping.

## 8. Post-Mortem Log Analyzer (new)

Milestone 3 requires an offline analyzer. This complements the real-time monitor by producing a report after the attacks.

Create:

```bash
cat > /opt/analyze_logs.py << 'EOF'
#!/usr/bin/env python3

import re
import sys
from collections import Counter, defaultdict
from datetime import datetime

LOG_FILE = sys.argv[1] if len(sys.argv) > 1 else "/var/log/nginx/access.log"

SCAN_THRESHOLD = 8
BRUTE_FORCE_THRESHOLD = 5
FLOOD_THRESHOLD = 30

log_pattern = re.compile(r'(\S+) - \S+ \[(.*?)\] "([^"]+)" (\d+) (\d+) "([^"]*)" "([^"]*)"')

exploit_patterns = {
    "SQL Injection": re.compile(r"(union\s+select|'OR'1'='1|' OR '1'='1|select.+from|drop\s+table)", re.I),
    "XSS": re.compile(r"(<script|javascript:|onerror=|onload=|alert\()", re.I),
    "Path Traversal": re.compile(r"(\.\./|\.\.\\|%2e%2e|%2f|etc/passwd|etc/shadow)", re.I),
    "Command Injection": re.compile(r"(;cat|;ls|\|\s*cat|`|\$\(|cmd=|exec=|shell=|/bin/bash)", re.I),
    "LFI/RFI": re.compile(r"(file://|php://|data://)", re.I),
}

sensitive_pattern = re.compile(r"(\.env|\.git|passwd|shadow|nginx\.conf|backup|dump|config|wp-admin|phpmyadmin|server-status|actuator|cgi-bin)", re.I)

total = 0
by_ip = Counter()
status_by_ip = defaultdict(Counter)
exploits_by_ip = defaultdict(Counter)
sensitive_by_ip = Counter()
dangerous_by_ip = Counter()
requests_by_second = defaultdict(Counter)

def parse_time(value):
    try:
        return datetime.strptime(value.split()[0], "%d/%b/%Y:%H:%M:%S")
    except ValueError:
        return None

with open(LOG_FILE, "r", encoding="utf-8", errors="replace") as f:
    for line in f:
        match = log_pattern.match(line.strip())
        if not match:
            continue

        ip, timestamp, request, status, size, referer, ua = match.groups()
        total += 1
        by_ip[ip] += 1
        status_by_ip[ip][status] += 1

        parts = request.split()
        method = parts[0] if parts else "UNKNOWN"

        if method in {"PUT", "DELETE", "PATCH", "TRACE", "OPTIONS"}:
            dangerous_by_ip[ip] += 1

        if sensitive_pattern.search(request):
            sensitive_by_ip[ip] += 1

        for name, pattern in exploit_patterns.items():
            if pattern.search(request):
                exploits_by_ip[ip][name] += 1

        parsed_time = parse_time(timestamp)
        if parsed_time:
            requests_by_second[ip][parsed_time.strftime("%Y-%m-%d %H:%M:%S")] += 1

print("=== IDS LOG ANALYSIS REPORT ===")
print(f"Log file: {LOG_FILE}")
print(f"Total requests: {total}")
print()

print("[1] Top IPs")
for ip, count in by_ip.most_common(10):
    print(f"- {ip}: {count} requests")
print()

print("[2] Directory scans - repeated 404")
for ip, statuses in status_by_ip.items():
    count = statuses["404"]
    if count >= SCAN_THRESHOLD:
        print(f"- {ip}: {count} HTTP 404 responses")
print()

print("[3] Brute force / forbidden attempts")
for ip, statuses in status_by_ip.items():
    count = statuses["401"] + statuses["403"]
    if count >= BRUTE_FORCE_THRESHOLD:
        print(f"- {ip}: {count} HTTP 401/403 responses")
print()

print("[4] Exploit patterns")
for ip, attacks in exploits_by_ip.items():
    if attacks:
        details = ", ".join(f"{name}={count}" for name, count in attacks.items())
        print(f"- {ip}: {details}")
print()

print("[5] Sensitive path attempts")
for ip, count in sensitive_by_ip.items():
    if count:
        print(f"- {ip}: {count} attempts")
print()

print("[6] Dangerous HTTP methods")
for ip, count in dangerous_by_ip.items():
    if count:
        print(f"- {ip}: {count} attempts")
print()

print("[7] Flood indicators")
for ip, per_second in requests_by_second.items():
    peak = max(per_second.values()) if per_second else 0
    if peak >= FLOOD_THRESHOLD:
        print(f"- {ip}: peak {peak} requests in one second")
EOF

chmod +x /opt/analyze_logs.py
```

Run:

```bash
/opt/analyze_logs.py /var/log/nginx/access.log
```

Expected result:

- structured IDS report;
- top IPs;
- detected scans;
- detected forbidden/admin failures;
- exploit attempts;
- flood indicators.

## 9. Final Report Generator (new)

This optional script saves the post-mortem analysis into a timestamped file.

Create:

```bash
cat > /opt/generate_report.sh << 'EOF'
#!/bin/bash

LOG_FILE="${1:-/var/log/nginx/access.log}"
OUT_DIR="${2:-/opt/ids-reports}"
TS="$(date +%Y%m%d-%H%M%S)"
OUT_FILE="$OUT_DIR/ids-report-$TS.txt"

mkdir -p "$OUT_DIR"

/opt/analyze_logs.py "$LOG_FILE" | tee "$OUT_FILE"

echo
echo "Report saved to: $OUT_FILE"
EOF

chmod +x /opt/generate_report.sh
```

Run:

```bash
/opt/generate_report.sh
```

Expected result:

- report printed to terminal;
- report saved in `/opt/ids-reports`.

## 10. Test Plan

Use three terminals on the reverse proxy/test environment.

### Terminal 1 - Real-Time Monitor

```bash
/opt/monitor_realtime.py
```

### Terminal 2 - Raw NGINX Logs

```bash
tail -f /var/log/nginx/access.log
```

### Terminal 3 - Traffic Generation

Run normal activity first:

```bash
/opt/ids-attacks/00_normal_activity.sh
```

Then run attacks one by one:

```bash
/opt/ids-attacks/01_sensitive_files.sh
/opt/ids-attacks/02_admin_probe.sh
/opt/ids-attacks/03_dangerous_methods.sh
/opt/ids-attacks/04_directory_traversal.sh
/opt/ids-attacks/05_injection_patterns.sh
/opt/ids-attacks/06_404_scan.sh
/opt/ids-attacks/07_common_sensitive_paths.sh
/opt/ids-attacks/08_large_post.sh
/opt/ids-attacks/09_rate_limit_burst.sh
/opt/ids-attacks/10_bruteforce_login.sh
```

After the test:

```bash
/opt/analyze_logs.py /var/log/nginx/access.log
/opt/generate_report.sh /var/log/nginx/access.log
```

## 11. Validation Matrix

| Test | Script | Expected NGINX result | Expected IDS result |
| --- | --- | --- | --- |
| Normal browsing | `00_normal_activity.sh` | Mostly `200`, maybe low `404` | No critical attack alert |
| Sensitive files | `01_sensitive_files.sh` | `403` or `404` | `SENSITIVE PATH` |
| Admin probing | `02_admin_probe.sh` | `403` or `404` | `FORBIDDEN` or `DIRECTORY SCAN` |
| Dangerous methods | `03_dangerous_methods.sh` | `403` or `405` | `DANGEROUS METHOD` |
| Directory traversal | `04_directory_traversal.sh` | `400`, `403`, or `404` | `Path Traversal` |
| Injection patterns | `05_injection_patterns.sh` | `200`, `403`, or `404` | `SQL Injection`, `XSS`, `Command Injection` |
| 404 scan | `06_404_scan.sh` | many `404` | `DIRECTORY SCAN` |
| Common sensitive paths | `07_common_sensitive_paths.sh` | `403` or `404` | `SENSITIVE PATH` |
| Large POST | `08_large_post.sh` | `413` | `LARGE REQUEST` |
| Burst / rate limit | `09_rate_limit_burst.sh` | `200` plus possible `503` | `FLOOD / DOS` |
| Brute force | `10_bruteforce_login.sh` | repeated `403` or `404` | `REPEATED FORBIDDEN` or `DIRECTORY SCAN` |

## 12. Detection Thresholds

Current thresholds in `/opt/monitor_realtime.py`:

| Indicator | Threshold |
| --- | --- |
| Directory scan | `8` HTTP 404 responses |
| Brute force / forbidden | `5` HTTP 401/403 responses |
| Flood | `30` requests in `30` seconds |
| Exploit escalation | `2` exploit attempts |

These values are intentionally low for a classroom demo. In production, they should be tuned using baseline traffic.

## 13. Limitations

This IDS is useful for demonstration, but it has clear limits:

- it only sees what NGINX logs;
- it is not a packet-based IDS like Snort or Suricata;
- long-running monitoring can produce false positives if thresholds are too low;
- encrypted traffic is only visible after TLS termination at NGINX;
- a skilled attacker can use slower attacks to avoid simple rate thresholds;
- log rotation must be handled if monitoring runs for a long time.

## 14. Improvements

Possible improvements:

- use sliding windows for 404 and forbidden counts;
- add alert cooldowns to reduce repeated flood alerts;
- export alerts to a file;
- add JSON output for reports;
- add IP allowlists for known test clients;
- compare this log-based IDS with Snort or Suricata;
- add Wireshark captures and correlate packets with NGINX logs.

## 15. Demo Flow

Recommended presentation flow:

1. Show reverse proxy IP and NGINX log file.
2. Start `/opt/monitor_realtime.py`.
3. Run `00_normal_activity.sh` and show normal alerts only.
4. Run `01_sensitive_files.sh` and show sensitive path alerts.
5. Run `05_injection_patterns.sh` and show exploit alerts.
6. Run `09_rate_limit_burst.sh` and show flood/rate alerts.
7. Stop the monitor with `Ctrl+C` and show final statistics.
8. Run `/opt/generate_report.sh` and show the post-mortem report.

## 16. Final Result Expected

At the end of Milestone 3:

- NGINX logs detailed HTTP requests;
- normal traffic can be simulated;
- malicious traffic can be simulated;
- the real-time IDS detects attacks as they happen;
- the post-mortem analyzer summarizes attacks after the test;
- a final report can be generated;
- the team has enough evidence for the presentation.
