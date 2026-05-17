# Milestone 3 - Quick Setup

This file gives one copy-paste setup block per machine.

- Run the **Debian attacker** block on the attack/test machine.
- Run the **Toolbox reverse proxy** block on the machine that has NGINX logs in `/var/log/nginx/access.log`.

All scripts are Bash scripts and use only standard shell tools.

## 1. Debian Attacker - Attack Scripts

Copy and run this whole block on the Debian attacker machine:

```bash
mkdir -p /opt/ids-attacks

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

cat > /opt/ids-attacks/01_sensitive_files.sh << 'EOF'
#!/bin/bash
TARGET="${1:-http://10.0.4.2}"

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
TARGET="${1:-http://10.0.4.2}"

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
TARGET="${1:-http://10.0.4.2}"

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
TARGET="${1:-http://10.0.4.2}"

echo "===== Directory traversal test ====="
curl -i --path-as-is "$TARGET/../../../../etc/passwd"
curl -i --path-as-is "$TARGET/..%2F..%2F..%2F..%2Fetc%2Fpasswd"
curl -i --path-as-is "$TARGET/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"
echo "===== End directory traversal test ====="
EOF

cat > /opt/ids-attacks/05_injection_patterns.sh << 'EOF'
#!/bin/bash
TARGET="${1:-http://10.0.4.2}"

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
TARGET="${1:-http://10.0.4.2}"

echo "===== 404 scan test ====="
for i in $(seq 1 30); do
    curl -s -o /dev/null -w "scan /test$i -> %{http_code}\n" "$TARGET/test$i"
done
echo "===== End 404 scan test ====="
EOF

cat > /opt/ids-attacks/07_common_sensitive_paths.sh << 'EOF'
#!/bin/bash
TARGET="${1:-http://10.0.4.2}"

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
TARGET="${1:-http://10.0.4.2}"

echo "===== Large POST request test ====="
dd if=/dev/zero of=/tmp/bigfile-ids bs=2M count=1 >/dev/null 2>&1
curl -i -X POST --data-binary @/tmp/bigfile-ids "$TARGET/"
echo "===== End large POST request test ====="
EOF

cat > /opt/ids-attacks/09_rate_limit_burst.sh << 'EOF'
#!/bin/bash
TARGET="${1:-http://10.0.4.2}"

echo "===== Burst / rate limiting test ====="
for i in $(seq 1 60); do
    curl -s -o /dev/null -w "%{http_code}\n" "$TARGET/" &
done
wait
echo "===== End burst / rate limiting test ====="
EOF

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

chmod +x /opt/ids-attacks/*.sh
ls -l /opt/ids-attacks
```

Useful run order on Debian attacker:

```bash
/opt/ids-attacks/00_normal_activity.sh
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

## 2. Toolbox Reverse Proxy - IDS Scripts

Copy and run this whole block on the Toolbox reverse proxy:

```bash
mkdir -p /opt/ids-reports /var/log/nginx
touch /var/log/nginx/access.log

cat > /opt/monitor_realtime.sh << 'EOF'
#!/bin/bash

LOG_FILE="${1:-/var/log/nginx/access.log}"

SCAN_THRESHOLD=8
BRUTE_FORCE_THRESHOLD=5
FLOOD_THRESHOLD=30
TIME_WINDOW=30
EXPLOIT_ALERT_THRESHOLD=2

RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
BOLD='\033[1m'
NC='\033[0m'

declare -A COUNT_404
declare -A COUNT_AUTH
declare -A COUNT_EXPLOIT
declare -A COUNT_SENSITIVE
declare -A COUNT_METHODS
declare -A COUNT_TOTAL
declare -A REQUEST_TIMES

alert() {
    local level="$1"
    local category="$2"
    local message="$3"
    local color="$NC"
    local icon="[*]"
    local ts

    ts="$(date '+%H:%M:%S')"

    case "$level" in
        NORMAL) color="$GREEN"; icon="[OK]" ;;
        INFO) color="$BLUE"; icon="[INFO]" ;;
        WARNING) color="$YELLOW"; icon="[WARN]" ;;
        CRITICAL) color="$RED"; icon="[CRIT]" ;;
        EXPLOIT) color="$RED"; icon="[EXPLOIT]" ;;
    esac

    echo -e "${color}${icon} [$ts] [$category]${NC} $message"
}

is_sensitive_path() {
    echo "$1" | grep -Eiq '(\.env|\.git|passwd|shadow|nginx\.conf|backup|dump|config|wp-admin|phpmyadmin|server-status|actuator|cgi-bin)'
}

detect_exploit() {
    local req="$1"

    if echo "$req" | grep -Eiq "(union[[:space:]]+select|'OR'1'='1|' OR '1'='1|select.+from|drop[[:space:]]+table)"; then
        echo "SQL Injection"
        return
    fi

    if echo "$req" | grep -Eiq '(<script|javascript:|onerror=|onload=|alert\()'; then
        echo "XSS"
        return
    fi

    if echo "$req" | grep -Eiq '(\.\./|\.\.\\|%2e%2e|%2f|etc/passwd|etc/shadow)'; then
        echo "Path Traversal"
        return
    fi

    if echo "$req" | grep -Eiq '(;cat|;ls|\|[[:space:]]*cat|`|\$\(|cmd=|exec=|shell=|/bin/bash)'; then
        echo "Command Injection"
        return
    fi

    if echo "$req" | grep -Eiq '(file://|php://|data://)'; then
        echo "LFI/RFI"
        return
    fi
}

show_stats() {
    echo
    echo -e "${BOLD}================ FINAL STATISTICS ================${NC}"

    for ip in "${!COUNT_TOTAL[@]}"; do
        echo
        echo -e "${BLUE}$ip${NC}"
        echo "  Total requests: ${COUNT_TOTAL[$ip]}"
        echo "  404 errors: ${COUNT_404[$ip]:-0}"
        echo "  Forbidden/auth failures: ${COUNT_AUTH[$ip]:-0}"
        echo "  Sensitive path attempts: ${COUNT_SENSITIVE[$ip]:-0}"
        echo "  Dangerous methods: ${COUNT_METHODS[$ip]:-0}"
        echo "  Exploit attempts: ${COUNT_EXPLOIT[$ip]:-0}"
    done
}

trap 'show_stats; exit 0' INT

if [ ! -f "$LOG_FILE" ]; then
    echo -e "${RED}Error: log file not found: $LOG_FILE${NC}"
    exit 1
fi

echo
echo -e "${BOLD}================ IDS REAL-TIME MONITORING ================${NC}"
echo "Log file: $LOG_FILE"
echo "Thresholds: Scan=$SCAN_THRESHOLD, Forbidden=$BRUTE_FORCE_THRESHOLD, Flood=$FLOOD_THRESHOLD/$TIME_WINDOW sec"
echo "Started: $(date '+%Y-%m-%d %H:%M:%S')"
echo -e "${BOLD}===========================================================${NC}"
echo "Press Ctrl+C to stop and see statistics."
echo

while read -r line; do
    if [[ "$line" =~ ^([^[:space:]]+)[[:space:]]-[[:space:]]([^[:space:]]+)[[:space:]]\[([^]]+)\][[:space:]]\"([^\"]+)\"[[:space:]]([0-9]+)[[:space:]]([0-9]+)[[:space:]]\"([^\"]*)\"[[:space:]]\"([^\"]*)\" ]]; then
        ip="${BASH_REMATCH[1]}"
        request="${BASH_REMATCH[4]}"
        status="${BASH_REMATCH[5]}"
        user_agent="${BASH_REMATCH[8]}"

        read -r method url protocol <<< "$request"
        now="$(date +%s)"

        COUNT_TOTAL[$ip]=$(( ${COUNT_TOTAL[$ip]:-0} + 1 ))

        old_times="${REQUEST_TIMES[$ip]}"
        new_times=""
        recent_count=0

        for t in $old_times; do
            if [ "$t" -gt $((now - TIME_WINDOW)) ]; then
                new_times="$new_times $t"
                recent_count=$((recent_count + 1))
            fi
        done

        new_times="$new_times $now"
        recent_count=$((recent_count + 1))
        REQUEST_TIMES[$ip]="$new_times"

        if [ "$recent_count" -gt "$FLOOD_THRESHOLD" ]; then
            alert "CRITICAL" "FLOOD / DOS" "$ip sent $recent_count requests in ${TIME_WINDOW}s"
        fi

        if [[ "$method" =~ ^(PUT|DELETE|PATCH|TRACE|OPTIONS)$ ]]; then
            COUNT_METHODS[$ip]=$(( ${COUNT_METHODS[$ip]:-0} + 1 ))
            alert "CRITICAL" "DANGEROUS METHOD" "$ip used $method on $url - status $status"
            continue
        fi

        if is_sensitive_path "$url"; then
            COUNT_SENSITIVE[$ip]=$(( ${COUNT_SENSITIVE[$ip]:-0} + 1 ))
            alert "CRITICAL" "SENSITIVE PATH" "$ip requested $url - status $status"
            continue
        fi

        exploit_type="$(detect_exploit "$request")"
        if [ -n "$exploit_type" ]; then
            COUNT_EXPLOIT[$ip]=$(( ${COUNT_EXPLOIT[$ip]:-0} + 1 ))

            if [ "${COUNT_EXPLOIT[$ip]}" -ge "$EXPLOIT_ALERT_THRESHOLD" ]; then
                alert "EXPLOIT" "ATTACK DETECTED" "$ip multiple exploit attempts detected: $exploit_type"
            else
                alert "EXPLOIT" "EXPLOIT ATTEMPT" "$ip suspicious pattern: $exploit_type"
            fi
            continue
        fi

        if [[ "$user_agent" =~ curl|nikto|sqlmap|nmap|masscan|dirbuster|wpscan|burp|metasploit|nessus ]]; then
            if [[ ! "$status" =~ ^(200|301|302|304)$ ]]; then
                alert "WARNING" "AUTOMATED CLIENT" "$ip user-agent: $user_agent"
            fi
        fi

        if [ "$status" = "404" ]; then
            COUNT_404[$ip]=$(( ${COUNT_404[$ip]:-0} + 1 ))

            if [ "${COUNT_404[$ip]}" -ge "$SCAN_THRESHOLD" ]; then
                alert "CRITICAL" "DIRECTORY SCAN" "$ip has ${COUNT_404[$ip]} HTTP 404 errors. Last URL: $url"
            elif [ "${COUNT_404[$ip]}" -ge 3 ]; then
                alert "WARNING" "MULTIPLE 404" "$ip has ${COUNT_404[$ip]} HTTP 404 errors"
            else
                alert "INFO" "404" "$ip -> $url"
            fi
            continue
        fi

        if [[ "$status" =~ ^(401|403)$ ]]; then
            COUNT_AUTH[$ip]=$(( ${COUNT_AUTH[$ip]:-0} + 1 ))

            if [ "${COUNT_AUTH[$ip]}" -ge "$BRUTE_FORCE_THRESHOLD" ]; then
                alert "CRITICAL" "REPEATED FORBIDDEN" "$ip has ${COUNT_AUTH[$ip]} forbidden/auth failures"
            else
                alert "WARNING" "FORBIDDEN" "$ip -> $url - status $status"
            fi
            continue
        fi

        if [[ "$status" =~ ^(413|429|500|503)$ ]]; then
            alert "WARNING" "HTTP ERROR" "$ip -> $url - status $status"
            continue
        fi

        if [[ "$status" =~ ^(200|304)$ ]]; then
            alert "NORMAL" "OK" "$ip -> $method $url"
            continue
        fi

        if [[ "$status" =~ ^(301|302)$ ]]; then
            alert "INFO" "REDIRECT" "$ip -> $url"
            continue
        fi

        alert "INFO" "HTTP $status" "$ip -> $method $url"
    fi
done < <(tail -n 0 -F "$LOG_FILE")
EOF

cat > /opt/analyze_logs.sh << 'EOF'
#!/bin/bash

LOG_FILE="${1:-/var/log/nginx/access.log}"
SCAN_THRESHOLD=8
BRUTE_FORCE_THRESHOLD=5
FLOOD_THRESHOLD=30

if [ ! -f "$LOG_FILE" ]; then
    echo "Error: log file not found: $LOG_FILE"
    exit 1
fi

declare -A TOTAL_BY_IP
declare -A STATUS_404
declare -A STATUS_AUTH
declare -A EXPLOITS
declare -A SENSITIVE
declare -A METHODS
declare -A FLOOD_PEAK
declare -A PER_SECOND

TOTAL=0

is_sensitive_path() {
    echo "$1" | grep -Eiq '(\.env|\.git|passwd|shadow|nginx\.conf|backup|dump|config|wp-admin|phpmyadmin|server-status|actuator|cgi-bin)'
}

detect_exploits() {
    local req="$1"
    local found=""

    if echo "$req" | grep -Eiq "(union[[:space:]]+select|'OR'1'='1|' OR '1'='1|select.+from|drop[[:space:]]+table)"; then
        found="$found SQL_Injection"
    fi

    if echo "$req" | grep -Eiq '(<script|javascript:|onerror=|onload=|alert\()'; then
        found="$found XSS"
    fi

    if echo "$req" | grep -Eiq '(\.\./|\.\.\\|%2e%2e|%2f|etc/passwd|etc/shadow)'; then
        found="$found Path_Traversal"
    fi

    if echo "$req" | grep -Eiq '(;cat|;ls|\|[[:space:]]*cat|`|\$\(|cmd=|exec=|shell=|/bin/bash)'; then
        found="$found Command_Injection"
    fi

    if echo "$req" | grep -Eiq '(file://|php://|data://)'; then
        found="$found LFI_RFI"
    fi

    echo "$found"
}

while read -r line; do
    if [[ "$line" =~ ^([^[:space:]]+)[[:space:]]-[[:space:]]([^[:space:]]+)[[:space:]]\[([^]]+)\][[:space:]]\"([^\"]+)\"[[:space:]]([0-9]+)[[:space:]]([0-9]+)[[:space:]]\"([^\"]*)\"[[:space:]]\"([^\"]*)\" ]]; then
        ip="${BASH_REMATCH[1]}"
        timestamp="${BASH_REMATCH[3]}"
        request="${BASH_REMATCH[4]}"
        status="${BASH_REMATCH[5]}"
        read -r method url protocol <<< "$request"
        second="${timestamp%% *}"

        TOTAL=$((TOTAL + 1))
        TOTAL_BY_IP[$ip]=$(( ${TOTAL_BY_IP[$ip]:-0} + 1 ))

        if [ "$status" = "404" ]; then
            STATUS_404[$ip]=$(( ${STATUS_404[$ip]:-0} + 1 ))
        fi

        if [[ "$status" =~ ^(401|403)$ ]]; then
            STATUS_AUTH[$ip]=$(( ${STATUS_AUTH[$ip]:-0} + 1 ))
        fi

        if [[ "$method" =~ ^(PUT|DELETE|PATCH|TRACE|OPTIONS)$ ]]; then
            METHODS[$ip]=$(( ${METHODS[$ip]:-0} + 1 ))
        fi

        if is_sensitive_path "$url"; then
            SENSITIVE[$ip]=$(( ${SENSITIVE[$ip]:-0} + 1 ))
        fi

        for attack in $(detect_exploits "$request"); do
            key="$ip:$attack"
            EXPLOITS[$key]=$(( ${EXPLOITS[$key]:-0} + 1 ))
        done

        ps_key="$ip:$second"
        PER_SECOND[$ps_key]=$(( ${PER_SECOND[$ps_key]:-0} + 1 ))
        if [ "${PER_SECOND[$ps_key]}" -gt "${FLOOD_PEAK[$ip]:-0}" ]; then
            FLOOD_PEAK[$ip]="${PER_SECOND[$ps_key]}"
        fi
    fi
done < "$LOG_FILE"

echo "=== IDS LOG ANALYSIS REPORT ==="
echo "Log file: $LOG_FILE"
echo "Total requests: $TOTAL"
echo

echo "[1] Top IPs"
for ip in "${!TOTAL_BY_IP[@]}"; do
    echo "- $ip: ${TOTAL_BY_IP[$ip]} requests"
done
echo

echo "[2] Directory scans - repeated 404"
for ip in "${!STATUS_404[@]}"; do
    if [ "${STATUS_404[$ip]}" -ge "$SCAN_THRESHOLD" ]; then
        echo "- $ip: ${STATUS_404[$ip]} HTTP 404 responses"
    fi
done
echo

echo "[3] Brute force / forbidden attempts"
for ip in "${!STATUS_AUTH[@]}"; do
    if [ "${STATUS_AUTH[$ip]}" -ge "$BRUTE_FORCE_THRESHOLD" ]; then
        echo "- $ip: ${STATUS_AUTH[$ip]} HTTP 401/403 responses"
    fi
done
echo

echo "[4] Exploit patterns"
for key in "${!EXPLOITS[@]}"; do
    ip="${key%%:*}"
    attack="${key#*:}"
    echo "- $ip: $attack=${EXPLOITS[$key]}"
done
echo

echo "[5] Sensitive path attempts"
for ip in "${!SENSITIVE[@]}"; do
    echo "- $ip: ${SENSITIVE[$ip]} attempts"
done
echo

echo "[6] Dangerous HTTP methods"
for ip in "${!METHODS[@]}"; do
    echo "- $ip: ${METHODS[$ip]} attempts"
done
echo

echo "[7] Flood indicators"
for ip in "${!FLOOD_PEAK[@]}"; do
    if [ "${FLOOD_PEAK[$ip]}" -ge "$FLOOD_THRESHOLD" ]; then
        echo "- $ip: peak ${FLOOD_PEAK[$ip]} requests in one logged second"
    fi
done
EOF

cat > /opt/generate_report.sh << 'EOF'
#!/bin/bash

LOG_FILE="${1:-/var/log/nginx/access.log}"
OUT_DIR="${2:-/opt/ids-reports}"
TS="$(date +%Y%m%d-%H%M%S)"
OUT_FILE="$OUT_DIR/ids-report-$TS.txt"

mkdir -p "$OUT_DIR"
/opt/analyze_logs.sh "$LOG_FILE" | tee "$OUT_FILE"
echo
echo "Report saved to: $OUT_FILE"
EOF

chmod +x /opt/monitor_realtime.sh /opt/analyze_logs.sh /opt/generate_report.sh
ls -l /opt/monitor_realtime.sh /opt/analyze_logs.sh /opt/generate_report.sh
```

Useful run order on Toolbox reverse proxy:

```bash
/opt/monitor_realtime.sh
/opt/analyze_logs.sh /var/log/nginx/access.log
/opt/generate_report.sh /var/log/nginx/access.log
```
