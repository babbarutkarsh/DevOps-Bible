---
title: Shell Scripting & Python
nav_order: 71
description: "Shell Scripting & Python — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Shell Scripting & Python for DevOps — Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Bash Essentials

### 🔥 Q: Explain `set -euo pipefail`. Why should every script have it?

**Answer:**

| Flag | Behavior |
|------|----------|
| `-e` | Exit immediately if any command returns non-zero |
| `-u` | Treat unset variables as errors (no silent empty strings) |
| `-o pipefail` | Pipe fails if ANY command in the pipeline fails (not just last) |

**Without pipefail:**
```bash
# BAD: curl fails silently, jq processes empty input
curl http://down.example.com/api | jq '.status'  # exit 0 if jq succeeds
```

**With pipefail:**
```bash
set -euo pipefail
curl http://down.example.com/api | jq '.status'  # exits with curl's error code
```

### 🔥 Q: Variables, quoting, and parameter expansion.

```bash
#!/bin/bash
set -euo pipefail

# Variable assignment (no spaces around =)
NAME="production"
readonly VERSION="1.0.0"      # Immutable
export DB_HOST="postgres"     # Available to child processes

# Parameter expansion
echo "${NAME}"                 # Safe: production
echo "${NAME^^}"               # Uppercase: PRODUCTION
echo "${NAME,,}"               # Lowercase: production
echo "${NAME:0:4}"             # Substring: prod
echo "${NAME/prod/stage}"      # Replace first: stageduction
echo "${NAME//o/0}"            # Replace all: pr0ducti0n

# Defaults & fallbacks
echo "${VAR:-default}"         # Use "default" if VAR unset/empty
echo "${VAR:=default}"         # Assign default if unset/empty
echo "${VAR:?error msg}"       # Exit with error if unset/empty
echo "${VAR:+alt_value}"       # Use alt_value if VAR is set

# Length
echo "${#NAME}"                # 10 (string length)

# Arrays
HOSTS=(web1 web2 web3)
echo "${HOSTS[0]}"             # web1
echo "${HOSTS[@]}"             # All elements
echo "${#HOSTS[@]}"            # Array length: 3

# Associative arrays (Bash 4+)
declare -A ENVS=([dev]="dev.example.com" [prod]="prod.example.com")
echo "${ENVS[prod]}"           # prod.example.com
for k in "${!ENVS[@]}"; do echo "$k -> ${ENVS[$k]}"; done
```

### 🔥 Q: Single vs double quotes. When does each matter?

**Single quotes:** Literal strings, no expansion.
```bash
echo '$HOME'           # $HOME (literal)
echo '$(date)'         # $(date) (literal)
```

**Double quotes:** Variable expansion, command substitution.
```bash
echo "$HOME"           # /home/user
echo "$(date)"         # Mon Jan 20 ...
echo "Total: $((2+3))" # Total: 5
```

**Unquoted (dangerous):** Word splitting & globbing.
```bash
FILES="*.txt"
echo $FILES            # Expands glob, splits on whitespace
echo "$FILES"          # *.txt (literal)
```

### ⭐ Q: Conditionals: `[[ ]]` vs `[ ]` vs `test`.

| Feature | `[ ]` (POSIX) | `[[ ]]` (Bash) |
|---------|---------------|----------------|
| String comparison | `=`, `!=` | `=`, `!=`, `<`, `>`, `=~` (regex) |
| Logic | `-a`, `-o` | `&&`, `\|\|` |
| Pattern matching | No | `[[ $str == *.log ]]` |
| Word splitting | Yes (quote vars!) | No (safe unquoted) |
| Portable | sh, dash, bash | Bash/zsh only |

**File tests:**
```bash
[[ -e file ]]   # Exists
[[ -f file ]]   # Regular file
[[ -d dir ]]    # Directory
[[ -L link ]]   # Symlink
[[ -r file ]]   # Readable
[[ -w file ]]   # Writable
[[ -x file ]]   # Executable
[[ -s file ]]   # Non-empty
[[ f1 -nt f2 ]] # f1 newer than f2
[[ f1 -ot f2 ]] # f1 older than f2
```

**String tests:**
```bash
[[ -z "$str" ]]       # Empty
[[ -n "$str" ]]       # Non-empty
[[ "$a" == "$b" ]]    # Equal
[[ "$a" != "$b" ]]    # Not equal
[[ "$a" < "$b" ]]     # Lexicographic less-than (Bash only)
[[ "$str" =~ ^[0-9]+$ ]]  # Regex match (Bash only)
```

**Numeric comparison:**
```bash
[[ $a -eq $b ]]   # Equal
[[ $a -ne $b ]]   # Not equal
[[ $a -lt $b ]]   # Less than
[[ $a -le $b ]]   # Less than or equal
[[ $a -gt $b ]]   # Greater than
[[ $a -ge $b ]]   # Greater than or equal
```

### 🔥 Q: Loops in Bash.

```bash
# For loop
for i in {1..5}; do echo "$i"; done

# C-style for
for ((i=0; i<5; i++)); do echo "$i"; done

# While loop
while read -r line; do
    echo "Processing: $line"
done < input.txt

# Until loop
COUNT=0
until [[ $COUNT -ge 5 ]]; do
    echo "$COUNT"
    ((COUNT++))
done

# Loop over array
HOSTS=(web1 web2 web3)
for host in "${HOSTS[@]}"; do
    ssh "$host" 'uptime'
done

# Parallel execution with wait
for host in "${HOSTS[@]}"; do
    ssh "$host" 'uptime' &
done
wait
```

### ⭐ Q: Functions & local variables.

```bash
#!/bin/bash
set -euo pipefail

# Function definition
deploy() {
    local env="$1"        # Local variable
    local ver="${2:-latest}"
    echo "Deploying $env with version $ver"
    return 0              # Exit code
}

# Call function
deploy "production" "v1.2.3"

# Return values via echo + command substitution
get_version() {
    echo "1.0.0"
}
VERSION=$(get_version)
echo "$VERSION"

# Exit codes
check_health() {
    curl -sf http://localhost:8080/health > /dev/null
    return $?
}
if check_health; then
    echo "Healthy"
else
    echo "Unhealthy"
fi
```

### 🔥 Q: Exit codes & `$?`.

```bash
# Last command exit code
ls /nonexistent
echo $?   # 2 (error)

# Explicitly set exit code
exit 1    # Non-zero = failure

# Check exit code
if kubectl apply -f deploy.yaml; then
    echo "Applied successfully"
else
    echo "Failed with code $?"
    exit 1
fi

# Ignore errors for specific command
rm -f /tmp/lock || true
```

### ⭐ Q: Trap for cleanup & signal handling.

```bash
#!/bin/bash
set -euo pipefail

LOCKFILE="/tmp/deploy.lock"

cleanup() {
    echo "Cleaning up..."
    rm -f "$LOCKFILE"
}

# Trap on EXIT (always), ERR (on error), INT (Ctrl+C), TERM (kill)
trap cleanup EXIT ERR INT TERM

# Acquire lock
echo $$ > "$LOCKFILE"

# Do work
echo "Deploying..."
sleep 5

# cleanup() runs automatically on exit/error/signal
```

### ⭐ Q: Here-docs & here-strings.

```bash
# Here-doc: multi-line input
cat <<EOF > config.yaml
server:
  host: ${HOST}
  port: ${PORT}
EOF

# Suppress expansion with quoted delimiter
cat <<'EOF' > script.sh
#!/bin/bash
echo "Variables like $HOME are literal"
EOF

# Here-string: single-line input
grep "error" <<< "This is an error message"

# Pipe to command
while read -r line; do
    echo "Line: $line"
done <<EOF
first
second
third
EOF
```

### 💡 Q: Process substitution.

```bash
# Compare two command outputs
diff <(ls dir1) <(ls dir2)

# Read from multiple sources
while read -r line; do
    echo "Combined: $line"
done < <(cat file1 file2)

# Provide input to command expecting a file
some_tool --config <(echo "key=value")
```

### 🔥 Q: Pipes & redirection.

```bash
# Stdout to file
echo "log" > output.txt         # Overwrite
echo "log" >> output.txt        # Append

# Stderr to file
command 2> error.log            # Stderr only
command > output.log 2>&1       # Both stdout & stderr to same file
command &> combined.log         # Bash shorthand for above

# Discard output
command > /dev/null             # Discard stdout
command 2> /dev/null            # Discard stderr
command &> /dev/null            # Discard both

# Redirect stderr to stdout in pipeline
grep error log.txt 2>&1 | less

# Tee: write to file AND stdout
command | tee output.log        # Overwrite
command | tee -a output.log     # Append
```

### ⭐ Q: Command substitution.

```bash
# Modern syntax (nestable)
NOW=$(date +%Y-%m-%d)
FILES=$(ls *.txt)
LINES=$(wc -l < file.txt)

# Backticks (legacy, avoid)
NOW=`date +%Y-%m-%d`

# Nested substitution
HOSTNAME=$(echo $(hostname) | tr '[:upper:]' '[:lower:]')
```

### ⭐ Q: Arithmetic.

```bash
# $(( )) for arithmetic
echo $((2 + 3))           # 5
echo $((10 / 3))          # 3 (integer division)
echo $((10 % 3))          # 1 (modulo)

# Increment/decrement
COUNT=0
((COUNT++))
((COUNT+=5))
echo $COUNT               # 6

# let (older style)
let "A = 5 + 3"
echo $A                   # 8

# expr (POSIX, very old)
expr 5 + 3                # 8 (needs spaces!)
```

### ⭐ Q: Argument parsing with `getopts`.

```bash
#!/bin/bash
set -euo pipefail

usage() {
    echo "Usage: $0 -e ENV -v VERSION [-d]"
    exit 1
}

DEBUG=false
while getopts "e:v:dh" opt; do
    case $opt in
        e) ENV="$OPTARG" ;;
        v) VER="$OPTARG" ;;
        d) DEBUG=true ;;
        h) usage ;;
        *) usage ;;
    esac
done

# Check required args
[[ -z "${ENV:-}" ]] && usage
[[ -z "${VER:-}" ]] && usage

echo "Deploying ENV=$ENV VER=$VER DEBUG=$DEBUG"
```

### ⭐ Q: String manipulation.

```bash
STR="hello-world-123"

# Length
echo "${#STR}"                    # 15

# Substring
echo "${STR:0:5}"                 # hello
echo "${STR:6}"                   # world-123

# Replace
echo "${STR/world/universe}"      # hello-universe-123 (first)
echo "${STR//o/0}"                # hell0-w0rld-123 (all)

# Remove prefix/suffix
FILE="archive.tar.gz"
echo "${FILE%.gz}"                # archive.tar (remove shortest suffix)
echo "${FILE%%.*}"                # archive (remove longest suffix)
echo "${FILE#*.}"                 # tar.gz (remove shortest prefix)
echo "${FILE##*.}"                # gz (remove longest prefix)

# Case conversion
echo "${STR^^}"                   # HELLO-WORLD-123
echo "${STR,,}"                   # hello-world-123
```

---

## The Workhorse Trio: grep, sed, awk

### 🔥 Q: grep essentials & common patterns.

```bash
# Basic search
grep "error" app.log                   # Simple match
grep -i "error" app.log                # Case-insensitive
grep -v "debug" app.log                # Invert match (exclude)
grep -c "ERROR" app.log                # Count matches

# Context lines
grep -A3 "FATAL" app.log               # 3 lines After
grep -B2 "FATAL" app.log               # 2 lines Before
grep -C2 "FATAL" app.log               # 2 lines Context (both)

# Regex
grep -E "error|warning|fatal" app.log  # Extended regex (alternation)
grep -E "^[0-9]{3}" app.log            # Lines starting with 3 digits
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" app.log  # Perl regex (IP)

# Recursive search
grep -r "TODO" src/                    # Recursive
grep -rn "TODO" src/                   # With line numbers
grep -rl "TODO" src/                   # Files only (no content)
grep -rE "error|warning" /var/log/ --include="*.log"

# Files & output
grep -l "error" *.log                  # List matching files
grep -L "error" *.log                  # List non-matching files
grep -o "http://[^[:space:]]*" file    # Only matching part
grep -q "error" app.log && echo "Found errors"  # Quiet (exit code only)
```

### 🔥 Q: sed for text transformation.

```bash
# Substitute
sed 's/old/new/' file                  # Replace first occurrence per line
sed 's/old/new/g' file                 # Replace all occurrences
sed 's/old/new/2' file                 # Replace 2nd occurrence only
sed 's/old/new/gi' file                # Case-insensitive replace

# In-place editing
sed -i 's/old/new/g' file              # GNU sed
sed -i '' 's/old/new/g' file           # BSD/macOS sed (requires empty string)

# Address ranges
sed -n '10,20p' file                   # Print lines 10-20 only
sed '5d' file                          # Delete line 5
sed '10,20d' file                      # Delete lines 10-20
sed '/pattern/d' file                  # Delete lines matching pattern

# Multiple edits
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file

# Regex groups
echo "2024-01-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# Output: 15/01/2024

# Common patterns
sed 's/^[ \t]*//' file                 # Remove leading whitespace
sed 's/[ \t]*$//' file                 # Remove trailing whitespace
sed '/^$/d' file                       # Remove empty lines
sed -n '/start/,/end/p' file           # Print between patterns
```

### 🔥 Q: awk for column processing & aggregation.

```bash
# Print columns
awk '{print $1}' file                  # First column
awk '{print $1, $3}' file              # Columns 1 and 3
awk '{print $NF}' file                 # Last column
awk '{print $(NF-1)}' file             # Second-to-last column

# Custom delimiter
awk -F: '{print $1}' /etc/passwd       # Field separator ":"
awk -F, '{print $2}' data.csv          # CSV

# Filtering
awk '$3 > 100' file                    # Lines where col3 > 100
awk '$2 == "ERROR"' log                # Lines where col2 is ERROR
awk '/pattern/ {print $0}' file        # Lines matching pattern (like grep)
awk '$9 >= 500' access.log             # HTTP 5xx errors

# Built-in variables
awk '{print NR, $0}' file              # NR = line number
awk 'NR==10' file                      # Print line 10 only
awk 'NR>=10 && NR<=20' file            # Lines 10-20
awk '{print NF}' file                  # NF = number of fields

# Calculations
awk '{sum += $1} END {print sum}' file        # Sum column 1
awk '{sum += $1; count++} END {print sum/count}' file  # Average
awk '{if ($1 > max) max = $1} END {print max}' file    # Max value

# BEGIN & END blocks
awk 'BEGIN {print "IP,Count"} {print $1}' access.log
awk 'END {print "Total lines:", NR}' file

# Associative arrays (count occurrences)
awk '{count[$1]++} END {for (ip in count) print ip, count[ip]}' access.log

# Format output
awk '{printf "%-20s %10s\n", $1, $2}' file
```

### 🔥 Q: Interview classic — Top 10 IPs from access log.

**Input (access.log):**
```
192.168.1.10 - - [01/Jan/2024:12:00:00] "GET /api HTTP/1.1" 200
10.0.0.5 - - [01/Jan/2024:12:00:01] "POST /login HTTP/1.1" 200
192.168.1.10 - - [01/Jan/2024:12:00:02] "GET /home HTTP/1.1" 200
```

**Solution (Bash one-liner):**
```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

**Explanation:**
1. `awk '{print $1}'` → Extract first column (IP)
2. `sort` → Sort IPs
3. `uniq -c` → Count occurrences
4. `sort -rn` → Sort numerically, reverse (highest first)
5. `head -10` → Top 10

**Pure awk solution:**
```bash
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log | sort -rn | head -10
```

### 🔥 Q: jq for JSON processing.

```bash
# Extract field
jq '.name' data.json
jq '.items[].metadata.name' pods.json

# Raw output (no quotes)
jq -r '.status' response.json

# Filter
jq '.[] | select(.age > 25)' users.json
jq '.items[] | select(.status == "Running")' pods.json

# Map
jq '.items[] | {name: .metadata.name, status: .status}' pods.json

# Array operations
jq 'length' array.json                 # Array length
jq '.[0]' array.json                   # First element
jq '.[-1]' array.json                  # Last element
jq '.[] | .name' array.json            # Iterate

# Multiple filters
jq '.[] | select(.age > 25) | .name' users.json

# From/to JSON
echo '{"status":"ok"}' | jq -r '.status'
jq -n --arg name "Alice" '{"user": $name}'  # Construct JSON

# Common pipeline
curl -s https://api.example.com/users | jq -r '.[] | select(.active) | .email'
```

### ⭐ Q: xargs for batch operations.

```bash
# Basic usage
find . -name "*.log" | xargs rm -f
find . -type f -name "*.txt" | xargs grep "error"

# Handle spaces in filenames
find . -name "*.log" -print0 | xargs -0 rm -f

# Parallel execution
find . -name "*.jpg" | xargs -P 4 -I {} convert {} {}.png

# Limit args per command
echo {1..10} | xargs -n 2 echo        # Process 2 args at a time

# Interactive confirmation
find /tmp -name "*.tmp" | xargs -p rm

# Build command from stdin
cat hosts.txt | xargs -I {} ssh {} 'uptime'
```

---

## Common Bash Pitfalls

### 🔥 Q: What are the most common Bash bugs in production scripts?

**1. Unquoted variables (word splitting & globbing)**
```bash
# BAD
FILES=$(ls *.txt)
rm $FILES                     # Splits on whitespace, expands globs

# GOOD
FILES=$(ls *.txt)
rm "$FILES"                   # Treats as single string
# BETTER: avoid parsing ls
for f in *.txt; do rm "$f"; done
```

**2. Parsing `ls` output**
```bash
# BAD
for file in $(ls); do
    echo "$file"              # Breaks on spaces, quotes, newlines
done

# GOOD
for file in *; do
    [[ -e "$file" ]] || continue
    echo "$file"
done

# BEST: use find
find . -maxdepth 1 -type f -print0 | while IFS= read -r -d '' file; do
    echo "$file"
done
```

**3. Not handling spaces in filenames**
```bash
# BAD
find . -name "*.txt" | xargs rm   # Breaks on spaces

# GOOD
find . -name "*.txt" -print0 | xargs -0 rm
find . -name "*.txt" -exec rm {} +
```

**4. `[` vs `[[` confusion**
```bash
# BAD
VAR="hello world"
[ $VAR == "hello world" ]     # Word splitting error

# GOOD
[[ $VAR == "hello world" ]]   # No word splitting in [[
```

**5. Ignoring exit codes**
```bash
# BAD
kubectl apply -f deploy.yaml
echo "Deployment successful"  # Prints even if apply failed

# GOOD
if kubectl apply -f deploy.yaml; then
    echo "Deployment successful"
else
    echo "Failed"
    exit 1
fi
```

**6. Not quoting command substitution**
```bash
# BAD
COUNT=$(wc -l file.txt)       # Returns "42 file.txt"
if [[ $COUNT -gt 10 ]]; then  # Comparison fails (not a number)

# GOOD
COUNT=$(wc -l < file.txt)     # Returns "42" only
```

**7. Using `cd` without checking success**
```bash
# BAD
cd /some/path
rm -rf *                      # Dangerous if cd failed!

# GOOD
cd /some/path || exit 1
rm -rf *

# BETTER
pushd /some/path || exit 1
rm -rf *
popd
```

**8. Incorrect array iteration**
```bash
# BAD
HOSTS=(web1 web2 web3)
for h in $HOSTS; do echo "$h"; done  # Only prints web1

# GOOD
for h in "${HOSTS[@]}"; do echo "$h"; done
```

**9. Backticks instead of `$()`**
```bash
# BAD (legacy, not nestable)
NOW=`date +%Y-%m-%d`

# GOOD
NOW=$(date +%Y-%m-%d)
YEAR=$(date -d "$(cat timestamp.txt)" +%Y)  # Nestable
```

**10. Not using `set -euo pipefail`**
```bash
# BAD
#!/bin/bash
# Script continues on errors, uses undefined vars

# GOOD
#!/bin/bash
set -euo pipefail
```

---

## Python for DevOps

### 🔥 Q: subprocess — running shell commands safely.

```python
import subprocess

# Modern API (Python 3.5+)
# Safe: no shell injection
result = subprocess.run(
    ["ls", "-la", "/tmp"],
    capture_output=True,
    text=True,
    check=True          # Raise CalledProcessError on non-zero exit
)
print(result.stdout)
print(result.returncode)

# Dangerous: shell=True (avoid unless necessary)
subprocess.run("ls -la /tmp", shell=True, check=True)  # Shell injection risk!

# Capture output
result = subprocess.run(
    ["kubectl", "get", "pods"],
    capture_output=True,
    text=True
)
if result.returncode == 0:
    print(result.stdout)
else:
    print(result.stderr)

# Pipe between commands (without shell=True)
p1 = subprocess.Popen(["ps", "aux"], stdout=subprocess.PIPE)
p2 = subprocess.Popen(["grep", "python"], stdin=p1.stdout, stdout=subprocess.PIPE)
p1.stdout.close()
output, _ = p2.communicate()
print(output.decode())

# Timeout
try:
    subprocess.run(["sleep", "10"], timeout=5, check=True)
except subprocess.TimeoutExpired:
    print("Command timed out")

# Working directory & environment
subprocess.run(
    ["make", "build"],
    cwd="/path/to/project",
    env={"PATH": "/usr/bin", "ENV": "production"},
    check=True
)
```

### 🔥 Q: When to use Python over Bash?

| Criterion | Bash | Python |
|-----------|------|--------|
| **Simple glue** | ✅ Pipes, one-liners | ❌ Verbose |
| **Complex logic** | ❌ Becomes unreadable | ✅ Clear control flow |
| **Error handling** | ❌ Trap/exit codes limited | ✅ try/except, retries |
| **Data structures** | ❌ Only arrays/strings | ✅ Lists, dicts, objects |
| **JSON/YAML** | ❌ jq/yq external tools | ✅ Native libraries |
| **API calls** | ❌ curl + parsing | ✅ requests library |
| **Portability** | ✅ Everywhere | ⚠️ Needs Python install |
| **Cloud SDKs** | ❌ CLI wrappers only | ✅ boto3, google-cloud-* |
| **Testing** | ❌ Hard to unit test | ✅ pytest, unittest |

**Rule of thumb:** Start with Bash for simple automation (<50 lines, mostly calling other tools). Switch to Python when you need real error handling, data structures, or cloud SDK integration.

### ⭐ Q: File & text processing.

```python
import os
import sys
from pathlib import Path

# pathlib (modern, recommended)
path = Path("/var/log/app.log")
print(path.exists())
print(path.is_file())
print(path.stat().st_size)
print(path.suffix)              # .log
print(path.stem)                # app

# Iterate files
for f in Path("/tmp").glob("*.log"):
    print(f)

for f in Path("/var/log").rglob("*.log"):  # Recursive
    print(f)

# Read file
content = path.read_text()
lines = path.read_text().splitlines()

# Write file
path.write_text("new content")

# os & os.path (legacy, but still common)
print(os.path.exists("/tmp"))
print(os.path.isfile("/etc/passwd"))
print(os.path.isdir("/var"))
print(os.path.basename("/var/log/app.log"))  # app.log
print(os.path.dirname("/var/log/app.log"))   # /var/log
print(os.path.splitext("file.tar.gz"))       # ('file.tar', '.gz')

# Line-by-line processing (memory-efficient)
with open("/var/log/app.log") as f:
    for line in f:
        if "ERROR" in line:
            print(line.strip())

# Context manager ensures file is closed
with open("/tmp/output.txt", "w") as f:
    f.write("log entry\n")
```

### ⭐ Q: argparse for CLI tools.

```python
import argparse

def main():
    parser = argparse.ArgumentParser(description="Deploy service")
    parser.add_argument("environment", choices=["dev", "staging", "prod"])
    parser.add_argument("version", help="Deployment version")
    parser.add_argument("-d", "--dry-run", action="store_true", help="Dry run")
    parser.add_argument("-v", "--verbose", action="count", default=0)
    parser.add_argument("--timeout", type=int, default=300)
    
    args = parser.parse_args()
    
    print(f"Deploying {args.version} to {args.environment}")
    if args.dry_run:
        print("Dry run mode")
    print(f"Verbosity: {args.verbose}")
    print(f"Timeout: {args.timeout}s")

if __name__ == "__main__":
    main()
```

**Usage:**
```bash
python deploy.py prod v1.2.3 --dry-run --verbose --timeout 600
```

### 🔥 Q: requests for API calls.

```python
import requests

# GET
resp = requests.get("https://api.example.com/status", timeout=10)
resp.raise_for_status()         # Raise HTTPError for 4xx/5xx
data = resp.json()
print(data["status"])

# POST with JSON
payload = {"name": "service1", "replicas": 3}
resp = requests.post(
    "https://api.example.com/deploy",
    json=payload,
    headers={"Authorization": "Bearer token"},
    timeout=30
)
print(resp.status_code)

# Error handling
try:
    resp = requests.get("https://api.example.com/health", timeout=5)
    resp.raise_for_status()
except requests.Timeout:
    print("Request timed out")
except requests.HTTPError as e:
    print(f"HTTP error: {e.response.status_code}")
except requests.RequestException as e:
    print(f"Request failed: {e}")

# Session (connection pooling)
session = requests.Session()
session.headers.update({"Authorization": "Bearer token"})
resp = session.get("https://api.example.com/users")
```

### ⭐ Q: JSON & YAML parsing.

```python
import json
import yaml

# JSON
data = json.loads('{"status": "ok", "count": 42}')
print(data["status"])

with open("config.json") as f:
    config = json.load(f)

with open("output.json", "w") as f:
    json.dump({"result": "success"}, f, indent=2)

# YAML (requires pyyaml: pip install pyyaml)
with open("config.yaml") as f:
    config = yaml.safe_load(f)

with open("output.yaml", "w") as f:
    yaml.dump({"service": "web", "replicas": 3}, f, default_flow_style=False)
```

### ⭐ Q: Logging best practices.

```python
import logging

# Configure once at module level
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s',
    handlers=[
        logging.FileHandler("app.log"),
        logging.StreamHandler()
    ]
)

log = logging.getLogger(__name__)

# Use appropriate levels
log.debug("Detailed diagnostic info")
log.info("Deployment started")
log.warning("Deprecated API used")
log.error("Failed to connect to database")
log.critical("System out of memory")

# Structured logging with context
log.info("Deployment complete", extra={
    "environment": "prod",
    "version": "v1.2.3",
    "duration_sec": 45
})

# Exception logging
try:
    1 / 0
except Exception:
    log.exception("Unexpected error")  # Includes traceback
```

### ⭐ Q: Retry logic with exponential backoff.

```python
import time
from functools import wraps

def retry(attempts=3, delay=1, backoff=2):
    """Retry decorator with exponential backoff"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            _delay = delay
            for attempt in range(1, attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == attempts:
                        raise
                    print(f"Attempt {attempt} failed: {e}. Retrying in {_delay}s...")
                    time.sleep(_delay)
                    _delay *= backoff
        return wrapper
    return decorator

@retry(attempts=5, delay=2, backoff=2)
def flaky_api_call():
    import requests
    resp = requests.get("https://api.example.com/health", timeout=5)
    resp.raise_for_status()
    return resp.json()

# Usage
try:
    result = flaky_api_call()
    print(result)
except Exception as e:
    print(f"Failed after retries: {e}")
```

### ⭐ Q: Working with cloud SDKs (boto3 example).

```python
import boto3

# EC2
ec2 = boto3.client('ec2', region_name='us-west-2')
response = ec2.describe_instances(
    Filters=[
        {'Name': 'tag:Environment', 'Values': ['production']},
        {'Name': 'instance-state-name', 'Values': ['running']}
    ]
)

for reservation in response['Reservations']:
    for instance in reservation['Instances']:
        print(f"{instance['InstanceId']}: {instance.get('PrivateIpAddress')}")

# S3
s3 = boto3.client('s3')
s3.upload_file('/tmp/backup.tar.gz', 'my-bucket', 'backups/backup.tar.gz')

objects = s3.list_objects_v2(Bucket='my-bucket', Prefix='logs/')
for obj in objects.get('Contents', []):
    print(obj['Key'])

# Systems Manager Parameter Store
ssm = boto3.client('ssm')
param = ssm.get_parameter(Name='/prod/db/password', WithDecryption=True)
print(param['Parameter']['Value'])

# Resource interface (higher-level)
s3_resource = boto3.resource('s3')
bucket = s3_resource.Bucket('my-bucket')
for obj in bucket.objects.filter(Prefix='logs/'):
    print(obj.key)
```

### ⭐ Q: Virtual environments & dependency management.

```bash
# Create venv
python3 -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows

# Install packages
pip install requests boto3 pyyaml
pip install -r requirements.txt

# Freeze dependencies
pip freeze > requirements.txt

# Deactivate
deactivate
```

**requirements.txt example:**
```
requests==2.31.0
boto3==1.34.0
pyyaml==6.0.1
```

---

## Coding-Round Questions (Bash vs Python)

### 🔥 Q: Parse access log & report top 10 IPs + status code distribution.

**Sample access.log:**
```
192.168.1.10 - - [20/Jan/2024:10:15:30] "GET /api/users HTTP/1.1" 200 1234
10.0.0.5 - - [20/Jan/2024:10:15:31] "POST /api/login HTTP/1.1" 401 89
192.168.1.10 - - [20/Jan/2024:10:15:32] "GET /api/orders HTTP/1.1" 500 456
```

**Bash solution:**
```bash
#!/bin/bash
set -euo pipefail

LOGFILE="${1:?Usage: $0 <logfile>}"

echo "Top 10 IPs:"
awk '{print $1}' "$LOGFILE" | sort | uniq -c | sort -rn | head -10

echo -e "\nStatus code distribution:"
awk '{print $9}' "$LOGFILE" | sort | uniq -c | sort -rn
```

**Python solution:**
```python
#!/usr/bin/env python3
import re
import sys
from collections import Counter

def parse_log(logfile):
    pattern = re.compile(
        r'(?P<ip>\S+) .* \[.*\] "(?P<method>\S+) (?P<path>\S+) .*" (?P<status>\d+)')
    
    ips = Counter()
    status_codes = Counter()
    
    with open(logfile) as f:
        for line in f:
            match = pattern.search(line)
            if match:
                ips[match['ip']] += 1
                status_codes[match['status']] += 1
    
    print("Top 10 IPs:")
    for ip, count in ips.most_common(10):
        print(f"  {ip}: {count}")
    
    print("\nStatus code distribution:")
    for code, count in sorted(status_codes.items()):
        print(f"  {code}: {count}")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} <logfile>")
        sys.exit(1)
    parse_log(sys.argv[1])
```

### ⭐ Q: Log rotation — delete files older than N days, keep latest M files.

**Bash solution:**
```bash
#!/bin/bash
set -euo pipefail

LOGDIR="/var/log/myapp"
DAYS=7
KEEP=10

# Delete files older than N days
find "$LOGDIR" -name "*.log" -type f -mtime +"$DAYS" -delete

# Keep only latest M files
ls -t "$LOGDIR"/*.log 2>/dev/null | tail -n +$((KEEP + 1)) | xargs -r rm -f

echo "Log rotation complete"
```

**Python solution:**
```python
#!/usr/bin/env python3
import os
import time
from pathlib import Path

LOGDIR = Path("/var/log/myapp")
DAYS = 7
KEEP = 10

# Delete files older than N days
cutoff = time.time() - (DAYS * 86400)
for logfile in LOGDIR.glob("*.log"):
    if logfile.stat().st_mtime < cutoff:
        logfile.unlink()
        print(f"Deleted: {logfile}")

# Keep only latest M files
logfiles = sorted(LOGDIR.glob("*.log"), key=lambda f: f.stat().st_mtime, reverse=True)
for old_file in logfiles[KEEP:]:
    old_file.unlink()
    print(f"Deleted (exceeded keep limit): {old_file}")

print("Log rotation complete")
```

### ⭐ Q: Retry a flaky command with exponential backoff.

**Bash solution:**
```bash
#!/bin/bash
set -euo pipefail

retry() {
    local max_attempts="$1"
    local delay="$2"
    local command="${*:3}"
    local attempt=1
    
    while (( attempt <= max_attempts )); do
        if eval "$command"; then
            return 0
        fi
        
        if (( attempt < max_attempts )); then
            echo "Attempt $attempt failed. Retrying in ${delay}s..." >&2
            sleep "$delay"
            delay=$((delay * 2))
        fi
        
        ((attempt++))
    done
    
    echo "Failed after $max_attempts attempts" >&2
    return 1
}

# Usage
retry 5 2 "curl -sf https://api.example.com/health"
```

**Python solution:**
```python
#!/usr/bin/env python3
import subprocess
import time
import sys

def retry_command(command, max_attempts=5, initial_delay=2, backoff=2):
    delay = initial_delay
    for attempt in range(1, max_attempts + 1):
        try:
            subprocess.run(command, shell=True, check=True, 
                          capture_output=True, text=True)
            return True
        except subprocess.CalledProcessError as e:
            if attempt < max_attempts:
                print(f"Attempt {attempt} failed. Retrying in {delay}s...", 
                      file=sys.stderr)
                time.sleep(delay)
                delay *= backoff
            else:
                print(f"Failed after {max_attempts} attempts", file=sys.stderr)
                return False

# Usage
if __name__ == "__main__":
    success = retry_command("curl -sf https://api.example.com/health")
    sys.exit(0 if success else 1)
```

### ⭐ Q: Monitor disk usage & alert if above threshold.

**Bash solution:**
```bash
#!/bin/bash
set -euo pipefail

THRESHOLD=85
WEBHOOK="${SLACK_WEBHOOK:-}"

df -h --output=pcent,target | tail -n+2 | while read -r usage mount; do
    pct="${usage%\%}"
    if [[ $pct -ge $THRESHOLD ]]; then
        msg="⚠️ DISK ALERT: $mount at ${usage} on $(hostname)"
        echo "$msg"
        
        if [[ -n "$WEBHOOK" ]]; then
            curl -s -X POST "$WEBHOOK" \
                -H "Content-Type: application/json" \
                -d "{\"text\":\"$msg\"}"
        fi
    fi
done
```

**Python solution:**
```python
#!/usr/bin/env python3
import os
import shutil
import requests
import socket

THRESHOLD = 85
WEBHOOK = os.getenv("SLACK_WEBHOOK")

def check_disk_usage():
    for part in ["/", "/var", "/home"]:
        if not os.path.exists(part):
            continue
        
        usage = shutil.disk_usage(part)
        percent = (usage.used / usage.total) * 100
        
        if percent >= THRESHOLD:
            msg = f"⚠️ DISK ALERT: {part} at {percent:.1f}% on {socket.gethostname()}"
            print(msg)
            
            if WEBHOOK:
                requests.post(WEBHOOK, json={"text": msg}, timeout=10)

if __name__ == "__main__":
    check_disk_usage()
```

### ⭐ Q: Find & kill processes matching a pattern.

**Bash solution:**
```bash
#!/bin/bash
set -euo pipefail

PATTERN="${1:?Usage: $0 <pattern>}"

# Find matching PIDs
PIDS=$(pgrep -f "$PATTERN")

if [[ -z "$PIDS" ]]; then
    echo "No processes found matching: $PATTERN"
    exit 0
fi

echo "Found processes:"
ps -fp $PIDS

read -r -p "Kill these processes? (y/N) " confirm
if [[ "$confirm" =~ ^[Yy]$ ]]; then
    echo "$PIDS" | xargs kill
    echo "Killed processes"
else
    echo "Aborted"
fi
```

**Python solution:**
```python
#!/usr/bin/env python3
import subprocess
import sys
import signal

def find_and_kill(pattern):
    # Find matching processes
    result = subprocess.run(
        ["pgrep", "-f", pattern],
        capture_output=True,
        text=True
    )
    
    if result.returncode != 0:
        print(f"No processes found matching: {pattern}")
        return
    
    pids = [int(pid) for pid in result.stdout.strip().split()]
    
    # Show processes
    subprocess.run(["ps", "-fp"] + [str(p) for p in pids])
    
    # Confirm
    confirm = input("Kill these processes? (y/N) ").strip().lower()
    if confirm == 'y':
        for pid in pids:
            try:
                os.kill(pid, signal.SIGTERM)
                print(f"Killed process {pid}")
            except ProcessLookupError:
                print(f"Process {pid} already terminated")
    else:
        print("Aborted")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} <pattern>")
        sys.exit(1)
    
    import os
    find_and_kill(sys.argv[1])
```

### ⭐ Q: Fetch JSON from API, filter, and take action.

**Scenario:** Query GitHub API for open PRs, filter by label, post to Slack.

**Python solution:**
```python
#!/usr/bin/env python3
import requests
import os

GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
SLACK_WEBHOOK = os.getenv("SLACK_WEBHOOK")
REPO = "myorg/myrepo"

headers = {"Authorization": f"token {GITHUB_TOKEN}"}

# Fetch open PRs
resp = requests.get(
    f"https://api.github.com/repos/{REPO}/pulls",
    headers=headers,
    params={"state": "open"},
    timeout=10
)
resp.raise_for_status()
prs = resp.json()

# Filter by label
urgent_prs = [pr for pr in prs if any(label["name"] == "urgent" for label in pr["labels"])]

if urgent_prs:
    message = f"🚨 {len(urgent_prs)} urgent PRs need review:\n"
    for pr in urgent_prs:
        message += f"• {pr['title']} - {pr['html_url']}\n"
    
    # Post to Slack
    requests.post(SLACK_WEBHOOK, json={"text": message}, timeout=10)
    print(f"Posted {len(urgent_prs)} urgent PRs to Slack")
else:
    print("No urgent PRs")
```

### 💡 Q: Watch a directory & react to new files.

**Bash solution (inotifywait):**
```bash
#!/bin/bash
set -euo pipefail

WATCHDIR="/var/uploads"

inotifywait -m -e create -e moved_to --format '%w%f' "$WATCHDIR" | while read -r file; do
    echo "New file: $file"
    # Process file
    process_upload "$file"
done
```

**Python solution (watchdog library):**
```python
#!/usr/bin/env python3
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import time

class UploadHandler(FileSystemEventHandler):
    def on_created(self, event):
        if not event.is_directory:
            print(f"New file: {event.src_path}")
            self.process_file(event.src_path)
    
    def process_file(self, filepath):
        # Process the uploaded file
        print(f"Processing {filepath}...")

if __name__ == "__main__":
    path = "/var/uploads"
    event_handler = UploadHandler()
    observer = Observer()
    observer.schedule(event_handler, path, recursive=False)
    observer.start()
    
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    
    observer.join()
```

---

## Common DevOps Scripts (Production-Ready)

### 🔥 Q: Deployment script with health check & rollback.

```bash
#!/bin/bash
set -euo pipefail

APP="myapp"
NS="production"
TIMEOUT=120

usage() {
    echo "Usage: $0 <version>"
    exit 1
}

health_check() {
    local ver="$1"
    echo "Waiting for rollout to complete..."
    if kubectl rollout status "deployment/$APP" -n "$NS" --timeout="${TIMEOUT}s"; then
        echo "✅ Deployment healthy"
        return 0
    else
        echo "❌ Health check failed"
        return 1
    fi
}

deploy() {
    local ver="$1"
    
    echo "📦 Deploying $APP:$ver to $NS"
    
    # Record current version for rollback
    CURRENT_IMAGE=$(kubectl get deployment "$APP" -n "$NS" -o jsonpath='{.spec.template.spec.containers[0].image}')
    echo "Current image: $CURRENT_IMAGE"
    
    # Deploy new version
    kubectl set image "deployment/$APP" "$APP=$APP:$ver" -n "$NS" --record
    
    if ! health_check "$ver"; then
        echo "🔄 Rolling back to $CURRENT_IMAGE"
        kubectl rollout undo "deployment/$APP" -n "$NS"
        kubectl rollout status "deployment/$APP" -n "$NS" --timeout="${TIMEOUT}s"
        exit 1
    fi
    
    echo "✅ Deployment successful"
}

[[ -z "${1:-}" ]] && usage
deploy "$1"
```

### ⭐ Q: Backup script with rotation & validation.

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/backups"
DB_NAME="myapp"
KEEP_DAYS=7
S3_BUCKET="s3://backups-prod"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.sql.gz"

echo "Starting backup: $BACKUP_FILE"

# Create backup
pg_dump "$DB_NAME" | gzip > "$BACKUP_FILE"

# Validate backup
if ! gunzip -t "$BACKUP_FILE"; then
    echo "❌ Backup validation failed"
    exit 1
fi

BACKUP_SIZE=$(stat -f%z "$BACKUP_FILE" 2>/dev/null || stat -c%s "$BACKUP_FILE")
echo "✅ Backup created: $(( BACKUP_SIZE / 1024 / 1024 ))MB"

# Upload to S3
aws s3 cp "$BACKUP_FILE" "$S3_BUCKET/"

# Cleanup old local backups
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -type f -mtime +"$KEEP_DAYS" -delete
echo "Cleaned up backups older than $KEEP_DAYS days"

# Cleanup old S3 backups
aws s3 ls "$S3_BUCKET/" | \
    awk '{print $4}' | \
    grep "${DB_NAME}_" | \
    head -n -"$KEEP_DAYS" | \
    xargs -I {} aws s3 rm "$S3_BUCKET/{}"
```

### ⭐ Q: Docker cleanup script (safe production version).

```bash
#!/bin/bash
set -euo pipefail

echo "🧹 Docker cleanup started"

# Remove stopped containers
echo "Removing stopped containers..."
docker container prune -f

# Remove dangling images
echo "Removing dangling images..."
docker image prune -f

# Remove images older than 7 days (careful in production!)
echo "Removing old images..."
docker image prune -a --filter "until=168h" -f

# Remove unused volumes
echo "Removing unused volumes..."
docker volume prune -f

# Remove build cache older than 7 days
echo "Removing old build cache..."
docker builder prune -f --filter "until=168h"

# Report disk usage
echo -e "\n📊 Docker disk usage:"
docker system df

echo "✅ Cleanup complete"
```

### ⭐ Q: Health check script for multiple services.

```bash
#!/bin/bash
set -euo pipefail

SERVICES=(
    "api:http://localhost:8080/health"
    "web:http://localhost:3000/health"
    "worker:http://localhost:9000/health"
)

FAIL_COUNT=0

for svc in "${SERVICES[@]}"; do
    NAME="${svc%%:*}"
    URL="${svc#*:}"
    
    echo -n "Checking $NAME... "
    
    if curl -sf "$URL" -o /dev/null --max-time 5; then
        echo "✅ OK"
    else
        echo "❌ FAILED"
        ((FAIL_COUNT++))
    fi
done

echo -e "\nResults: $((${#SERVICES[@]} - FAIL_COUNT))/${#SERVICES[@]} healthy"

[[ $FAIL_COUNT -eq 0 ]] && exit 0 || exit 1
```

### ⭐ Q: Certificate expiration checker.

```bash
#!/bin/bash
set -euo pipefail

DOMAINS=(
    "example.com"
    "api.example.com"
    "www.example.com"
)

WARN_DAYS=30

for domain in "${DOMAINS[@]}"; do
    echo -n "Checking $domain... "
    
    EXPIRY=$(echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null | \
             openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
    
    if [[ -z "$EXPIRY" ]]; then
        echo "❌ Unable to fetch certificate"
        continue
    fi
    
    EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s 2>/dev/null || date -j -f "%b %d %T %Y %Z" "$EXPIRY" +%s)
    NOW_EPOCH=$(date +%s)
    DAYS_LEFT=$(( (EXPIRY_EPOCH - NOW_EPOCH) / 86400 ))
    
    if [[ $DAYS_LEFT -lt 0 ]]; then
        echo "❌ EXPIRED $((DAYS_LEFT * -1)) days ago"
    elif [[ $DAYS_LEFT -lt $WARN_DAYS ]]; then
        echo "⚠️ Expires in $DAYS_LEFT days"
    else
        echo "✅ Valid ($DAYS_LEFT days remaining)"
    fi
done
```

### 💡 Q: Process watchdog with restart limit.

```bash
#!/bin/bash
set -euo pipefail

PROCESS="myapp"
MAX_RESTARTS=5
WINDOW_SEC=300  # 5 minutes
LOGFILE="/var/log/watchdog.log"

declare -a RESTART_TIMES=()

restart_app() {
    NOW=$(date +%s)
    RESTART_TIMES+=("$NOW")
    
    # Clean old restart timestamps outside window
    for i in "${!RESTART_TIMES[@]}"; do
        if (( NOW - RESTART_TIMES[i] > WINDOW_SEC )); then
            unset 'RESTART_TIMES[i]'
        fi
    done
    RESTART_TIMES=("${RESTART_TIMES[@]}")  # Re-index
    
    # Check restart limit
    if (( ${#RESTART_TIMES[@]} >= MAX_RESTARTS )); then
        echo "$(date) ERROR: Restart limit exceeded ($MAX_RESTARTS in ${WINDOW_SEC}s)" | tee -a "$LOGFILE"
        exit 1
    fi
    
    echo "$(date) Restarting $PROCESS (attempt ${#RESTART_TIMES[@]})" | tee -a "$LOGFILE"
    systemctl restart "$PROCESS"
}

while true; do
    if ! pgrep -x "$PROCESS" > /dev/null; then
        echo "$(date) $PROCESS not running" | tee -a "$LOGFILE"
        restart_app
    fi
    sleep 10
done
```

---

## Scenario-Based Questions

### ⭐ Q: Parse a CSV and find duplicate entries by key column.

```bash
# Using awk
awk -F, '{seen[$1]++} END {for (k in seen) if (seen[k]>1) print k, seen[k]}' data.csv

# More detailed with headers
awk -F, 'NR==1 {print; next} {seen[$1]++; lines[$1]=lines[$1] $0 "\n"} END {
    for (k in seen) if (seen[k]>1) print "Duplicate key:", k, "Count:", seen[k]
}' data.csv
```

### ⭐ Q: Find files modified in the last 24 hours, sorted by size.

```bash
# Safer than parsing ls
find /path -type f -mtime -1 -printf '%s %p\n' 2>/dev/null | sort -rn | head -10

# Alternative with human-readable sizes
find /path -type f -mtime -1 -exec ls -lh {} \; 2>/dev/null | awk '{print $5, $9}' | sort -rh | head -10
```

### ⭐ Q: Extract unique IP addresses from multiple log files.

```bash
# From all log files
awk '{print $1}' /var/log/apache2/access.log* | sort -u

# With count
awk '{print $1}' /var/log/apache2/access.log* | sort | uniq -c | sort -rn
```

### ⭐ Q: Check if a port is open on multiple hosts.

```bash
#!/bin/bash
HOSTS=("web1" "web2" "web3")
PORT=443

for host in "${HOSTS[@]}"; do
    if timeout 2 bash -c "cat < /dev/null > /dev/tcp/$host/$PORT" 2>/dev/null; then
        echo "✅ $host:$PORT open"
    else
        echo "❌ $host:$PORT closed"
    fi
done
```

### ⭐ Q: Generate a report of failed systemd services.

```bash
#!/bin/bash
set -euo pipefail

echo "Failed Services Report - $(date)"
echo "================================"

systemctl list-units --state=failed --no-pager --no-legend | while read -r unit rest; do
    echo -e "\n❌ $unit"
    systemctl status "$unit" --no-pager -n 5 | tail -n 5
done
```

---

## Advanced Topics

### 💡 Q: Parallel execution with GNU parallel.

```bash
# Install: apt install parallel / brew install parallel

# Process files in parallel
find . -name "*.log" | parallel gzip {}

# Run commands on multiple hosts
parallel -j 4 ssh {} 'uptime' ::: web1 web2 web3 web4

# Process with argument from file
parallel -a hosts.txt ssh {} 'df -h /'

# More control
parallel --jobs 8 --bar 'process {}' ::: $(ls *.txt)
```

### ⭐ Q: Debugging Bash scripts.

```bash
# Enable tracing
set -x              # Print commands before executing
set -v              # Print script lines as read

# Debug specific section
set -x
complex_function
set +x

# Run with debug flag
bash -x script.sh

# Check syntax without running
bash -n script.sh

# Use trap for debugging
trap 'echo "Line $LINENO: $BASH_COMMAND"' DEBUG
```

### ⭐ Q: Writing idempotent scripts.

```bash
#!/bin/bash
set -euo pipefail

# Idempotent: can run multiple times safely

# Check before create
if [[ ! -d /opt/myapp ]]; then
    mkdir -p /opt/myapp
fi

# Use -f for idempotent removal
rm -f /tmp/lock

# Conditional operations
if ! grep -q "myconfig" /etc/app.conf; then
    echo "myconfig" >> /etc/app.conf
fi

# Replace entire block idempotently
sed -i '/# START MANAGED/,/# END MANAGED/d' config
cat >> config <<'EOF'
# START MANAGED
setting=value
# END MANAGED
EOF
```

### 💡 Q: Python — async I/O for concurrent operations.

```python
import asyncio
import aiohttp

async def check_health(session, url):
    try:
        async with session.get(f"{url}/health", timeout=5) as resp:
            return url, resp.status
    except Exception as e:
        return url, str(e)

async def main():
    urls = [
        "http://api1.example.com",
        "http://api2.example.com",
        "http://api3.example.com",
    ]
    
    async with aiohttp.ClientSession() as session:
        tasks = [check_health(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        
        for url, status in results:
            print(f"{url}: {status}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 💡 Q: Python — context managers for resource cleanup.

```python
from contextlib import contextmanager
import tempfile
import shutil

@contextmanager
def temp_directory():
    """Context manager for temporary directory"""
    tmpdir = tempfile.mkdtemp()
    try:
        yield tmpdir
    finally:
        shutil.rmtree(tmpdir)

# Usage
with temp_directory() as tmpdir:
    # Work with tmpdir
    print(f"Working in {tmpdir}")
    # Automatically cleaned up on exit
```

---

## Quick Reference

### Bash One-Liners (Common Interview Questions)

```bash
# Top 10 memory-consuming processes
ps aux --sort=-%mem | head -10

# Top 10 CPU-consuming processes
ps aux --sort=-%cpu | head -10

# Count files by extension
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Find and replace recursively
find . -type f -name "*.txt" -exec sed -i 's/old/new/g' {} +

# Kill all processes matching pattern
pkill -f pattern

# Download file with retry
wget -t 5 --retry-connrefused https://example.com/file

# HTTP request with retry (curl)
curl --retry 5 --retry-delay 3 --retry-max-time 60 https://api.example.com

# Compare two directories
diff -r dir1 dir2

# Merge PDFs
gs -dBATCH -dNOPAUSE -q -sDEVICE=pdfwrite -sOutputFile=merged.pdf *.pdf

# Extract tar preserving permissions
tar -xzf archive.tar.gz --same-owner --same-permissions

# Network usage by process
nethogs

# Monitor log file changes in real-time
tail -f /var/log/syslog | grep --line-buffered "error"
```

### Python Snippets for DevOps

```python
# Read environment variables with defaults
import os
DB_HOST = os.getenv("DB_HOST", "localhost")
DEBUG = os.getenv("DEBUG", "false").lower() == "true"

# Parse ISO timestamp
from datetime import datetime
dt = datetime.fromisoformat("2024-01-20T10:15:30Z".replace("Z", "+00:00"))

# Calculate duration
import time
start = time.time()
# ... do work ...
duration = time.time() - start
print(f"Took {duration:.2f}s")

# Atomic file write
import tempfile
import os
import shutil

def atomic_write(filepath, content):
    fd, tmppath = tempfile.mkstemp(dir=os.path.dirname(filepath))
    try:
        with os.fdopen(fd, 'w') as f:
            f.write(content)
        shutil.move(tmppath, filepath)
    except:
        os.unlink(tmppath)
        raise

# HTTP basic auth
import requests
from requests.auth import HTTPBasicAuth

resp = requests.get(
    "https://api.example.com",
    auth=HTTPBasicAuth("user", "pass")
)
```

---

## Key Resources

### Official Documentation
- **Bash Reference Manual** — https://www.gnu.org/software/bash/manual/
- **Advanced Bash-Scripting Guide** — https://tldp.org/LDP/abs/html/
- **Python Documentation** — https://docs.python.org/3/

### Style Guides & Best Practices
- **Google Shell Style Guide** — https://google.github.io/styleguide/shellguide.html
- **ShellCheck** (linter) — https://www.shellcheck.net
- **Bash Pitfalls** — https://mywiki.wooledge.org/BashPitfalls

### Learning Resources
- **ExplainShell** (command explainer) — https://explainshell.com
- **Automate the Boring Stuff with Python** — https://automatetheboringstuff.com
- **Python for DevOps** (book) — Noah Gift & Kennedy Behrman
- **The Art of Command Line** — https://github.com/jlevy/the-art-of-command-line

### Tools & Libraries
- **subprocess** (Python) — https://docs.python.org/3/library/subprocess.html
- **requests** (Python HTTP) — https://requests.readthedocs.io
- **boto3** (AWS SDK) — https://boto3.amazonaws.com/v1/documentation/api/latest/index.html
- **GNU Parallel** — https://www.gnu.org/software/parallel/
- **jq** (JSON processor) — https://stedolan.github.io/jq/

### Regex & Text Processing
- **Regex101** (regex tester) — https://regex101.com
- **awk tutorial** — https://www.grymoire.com/Unix/Awk.html
- **sed tutorial** — https://www.grymoire.com/Unix/Sed.html
