---
title: Shell Scripting & Python
nav_order: 71
description: "Shell Scripting & Python — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Shell Scripting & Python for DevOps — Interview Preparation

## Bash Essentials

### Q: Script best practices.

```bash
#!/bin/bash
set -euo pipefail    # Exit on error, undefined vars, pipe failures

# Variables & strings
NAME="production"
readonly VERSION="1.0.0"
export DB_HOST="postgres"
echo "${NAME^^}"               # PRODUCTION
echo "${NAME:0:4}"             # prod

# Conditionals & file tests
[[ -z "$VAR" ]]   # Empty string     [[ -f file ]]   # File exists
[[ -d dir ]]      # Directory         [[ -x file ]]   # Executable

# Loops
for s in web1 web2 web3; do deploy "$s" & done; wait

# Functions
deploy() { local env="$1"; echo "Deploying to $env"; }

# Error handling + cleanup
trap 'rm -f /tmp/lock' EXIT ERR

# Argument parsing
while getopts "e:v:h" opt; do
    case $opt in
        e) ENV="$OPTARG" ;; v) VER="$OPTARG" ;; *) exit 1 ;;
    esac
done
```

### Q: Text processing (grep, awk, sed, jq).

```bash
grep -rE "error|warning" /var/log/     # Recursive regex search
grep -c "ERROR" app.log                # Count matches
grep -A3 -B1 "FATAL" app.log          # Context lines

awk '{print $1, $9}' access.log        # Print columns 1 & 9
awk -F: '{print $1}' /etc/passwd       # Custom delimiter
awk '$9>=500' access.log               # Filter by status code
awk '{s+=$1} END{print s}' file        # Sum column

sed 's/old/new/g' file                 # Replace all
sed -i 's/old/new/g' file              # In-place
sed -n '10,20p' file                   # Print line range

jq '.items[].metadata.name' pods.json  # Extract field
jq '.[] | select(.age > 25)' data.json # Filter
jq -r '.status' <<< '{"status":"ok"}' # Raw output

sort file | uniq -c | sort -rn | head  # Top occurrences
find /tmp -name "*.log" -mtime +7 | xargs rm -f
```

---

## Python for DevOps

### Q: Core patterns.

```python
#!/usr/bin/env python3
import subprocess, json, logging, time, requests
from functools import wraps

logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(levelname)s] %(message)s')
log = logging.getLogger(__name__)

def run(cmd):
    r = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    if r.returncode != 0: raise RuntimeError(r.stderr)
    return r.stdout.strip()

def retry(attempts=3, delay=5):
    def dec(fn):
        @wraps(fn)
        def wrap(*a, **kw):
            for i in range(1, attempts+1):
                try: return fn(*a, **kw)
                except Exception as e:
                    if i == attempts: raise
                    time.sleep(delay * 2**(i-1))
        return wrap
    return dec

@retry(attempts=3)
def health_check(url):
    return requests.get(f"{url}/health", timeout=5).status_code == 200
```

### Q: Log parser (common interview question).

```python
import re
from collections import Counter

PATTERN = re.compile(
    r'(?P<ip>\S+) .* "(?P<method>\S+) (?P<path>\S+) .*" (?P<status>\d+)')

def parse(logfile):
    ips, codes = Counter(), Counter()
    with open(logfile) as f:
        for line in f:
            m = PATTERN.search(line)
            if m:
                ips[m['ip']] += 1
                codes[m['status']] += 1
    for ip, c in ips.most_common(10): print(f"  {ip}: {c}")
    for s, c in sorted(codes.items()): print(f"  {s}: {c}")
```

### Q: AWS boto3 example.

```python
import boto3
def get_instances(env):
    ec2 = boto3.client('ec2')
    r = ec2.describe_instances(Filters=[
        {'Name': 'tag:Environment', 'Values': [env]},
        {'Name': 'instance-state-name', 'Values': ['running']}])
    return [{'id': i['InstanceId'], 'ip': i.get('PrivateIpAddress')}
            for res in r['Reservations'] for i in res['Instances']]
```

---

## Common DevOps Scripts

### Q: Deployment with rollback.

```bash
#!/bin/bash
set -euo pipefail
APP="myapp"; NS="production"

health_check() {
    kubectl rollout status "deployment/$APP" -n "$NS" --timeout=120s
}

deploy() {
    local ver="$1"
    echo "Deploying $APP:$ver"
    kubectl set image "deployment/$APP" "$APP=$APP:$ver" -n "$NS"
    if ! health_check; then
        echo "FAILED — rolling back"
        kubectl rollout undo "deployment/$APP" -n "$NS"
        exit 1
    fi
    echo "SUCCESS"
}

deploy "${1:?Usage: $0 <version>}"
```

### Q: Docker cleanup script.

```bash
#!/bin/bash
set -euo pipefail
docker container prune -f
docker image prune -a --filter "until=168h" -f
docker volume prune -f
docker builder prune -f --filter "until=168h"
docker system df
```

### Q: Disk monitoring with Slack alert.

```bash
#!/bin/bash
set -euo pipefail
THRESHOLD=85
df -h --output=pcent,target | tail -n+2 | while read -r usage mount; do
    pct=${usage%\%}
    if [[ $pct -ge $THRESHOLD ]]; then
        msg="⚠️ DISK: $mount at ${usage} on $(hostname)"
        echo "$msg"
        [[ -n "${SLACK_WEBHOOK:-}" ]] && \
            curl -s -X POST "$SLACK_WEBHOOK" -d "{\"text\":\"$msg\"}"
    fi
done
```

---

## Scenario-Based Questions

### Q: Parse a CSV and find duplicate entries.

```bash
awk -F, '{seen[$1]++} END {for (k in seen) if (seen[k]>1) print k, seen[k]}' data.csv
```

### Q: Find the top 10 largest files modified in the last 24 hours.

```bash
find / -type f -mtime -1 -exec ls -lS {} + 2>/dev/null | head -10
```

### Q: Monitor a process and restart if it crashes.

```bash
#!/bin/bash
while true; do
    if ! pgrep -x "myapp" > /dev/null; then
        echo "$(date) myapp crashed, restarting..." >> /var/log/watchdog.log
        systemctl restart myapp
    fi
    sleep 10
done
```

---

## Key Resources

- **Advanced Bash-Scripting Guide** — https://tldp.org/LDP/abs/html/
- **Shell Style Guide (Google)** — https://google.github.io/styleguide/shellguide.html
- **Automate the Boring Stuff with Python** — https://automatetheboringstuff.com
- **Python for DevOps (book)** — Noah Gift & Kennedy Behrman
- **ExplainShell** — https://explainshell.com
