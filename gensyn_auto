#!/usr/bin/env bash
# A sturdier watchdog for run_rl_swarm.sh with graceful kills, backoff, and log rotation.

set -o pipefail

# ---------- Defaults (can be overridden by env or CLI flags) ----------
SCRIPT="${SCRIPT:-./run_rl_swarm.sh}"
TMP_LOG="${TMP_LOG:-/tmp/rlswarm_stdout.log}"
IDLE_SECS="${MAX_IDLE:-900}"          # same as your MAX_IDLE
CASE_INSENSITIVE="${CASE_INSENSITIVE:-1}" # 1 = case-insensitive grep for errors
BACKOFF_START="${BACKOFF_START:-3}"   # seconds
BACKOFF_MAX="${BACKOFF_MAX:-60}"      # seconds
KEYWORDS_FILE="${KEYWORDS_FILE:-}"    # optional: path to a file with one keyword per line

# Built-in default keywords (used if KEYWORDS_FILE not provided)
DEFAULT_KEYWORDS=(
  "BlockingIOError"
  "EOFError"
  "RuntimeError"
  "ConnectionResetError"
  "CUDA out of memory"
  "P2PDaemonError"
  "OSError"
  "error was detected while running rl-swarm"
  "Connection refused"
  "requests.exceptions.ConnectionError"
)

# Processes to nuke if they’re running from the same working dir or match patterns
NUKE_PATTERNS=(
  "hivemind_cli/p2pd"
  "node_modules/.bin/next"
  "modal-login"
  "rgym_exp.runner.swarm_launcher"
)

# ---------- CLI ----------
usage() {
  cat <<EOF
Usage: $0 [options]

Options:
  -s PATH   Path to runner script (default: ${SCRIPT})
  -l PATH   Temp log file (default: ${TMP_LOG})
  -i SEC    Idle seconds threshold (default: ${IDLE_SECS})
  -k FILE   Keywords file (one per line). If omitted, uses built-in list.
  -c 0|1    Case-insensitive error matching (default: ${CASE_INSENSITIVE})
  --backoff-start SEC  Initial backoff (default: ${BACKOFF_START})
  --backoff-max   SEC  Max backoff (default: ${BACKOFF_MAX})
  -h         Show help
EOF
}

while (( "$#" )); do
  case "$1" in
    -s) SCRIPT="$2"; shift 2;;
    -l) TMP_LOG="$2"; shift 2;;
    -i) IDLE_SECS="$2"; shift 2;;
    -k) KEYWORDS_FILE="$2"; shift 2;;
    -c) CASE_INSENSITIVE="$2"; shift 2;;
    --backoff-start) BACKOFF_START="$2"; shift 2;;
    --backoff-max) BACKOFF_MAX="$2"; shift 2;;
    -h|--help) usage; exit 0;;
    *) echo "Unknown option: $1"; usage; exit 1;;
  esac
done

# ---------- Helpers ----------
log() {
  # levels: INFO, WARN, ERR, DBG
  local lvl="$1"; shift
  printf '[%(%Y-%m-%d %H:%M:%S)T] %-4s %s\n' -1 "$lvl" "$*"
}

rotate_log() {
  local base="$1"
  local max=5
  for (( i=max; i>=1; i-- )); do
    if [[ -f "${base}.${i}.gz" ]]; then
      if (( i == max )); then rm -f "${base}.${i}.gz"; else mv "${base}.${i}.gz" "${base}.$((i+1)).gz"; fi
    fi
  done
  if [[ -f "$base" ]]; then
    gzip -c "$base" > "${base}.1.gz" 2>/dev/null || true
    : > "$base"
  fi
}

script_dir="$(cd "$(dirname "$SCRIPT")" && pwd)"
lockfile="/tmp/rlswarm_guard.lock"

# Load keywords
KEYWORDS=()
if [[ -n "$KEYWORDS_FILE" && -r "$KEYWORDS_FILE" ]]; then
  # shellcheck disable=SC2207
  KEYWORDS=($(grep -v '^\s*$' "$KEYWORDS_FILE"))
else
  KEYWORDS=("${DEFAULT_KEYWORDS[@]}")
fi

grep_flags="-q"
if [[ "$CASE_INSENSITIVE" == "1" ]]; then
  grep_flags="-qi"
fi

# Determine mtime in a GNU/BSD compatible way
file_mtime() {
  local f="$1"
  if command -v stat >/dev/null 2>&1; then
    if stat --version >/dev/null 2>&1; then
      stat -c %Y "$f" 2>/dev/null
    else
      stat -f %m "$f" 2>/dev/null
    fi
  else
    # last resort: use perl
    perl -e '(@s)=stat(shift); print $s[9] if @s' "$f" 2>/dev/null
  fi
}

kill_tree_graceful() {
  local pid="$1"
  local grace="${2:-10}"

  if [[ -z "$pid" || ! "$pid" =~ ^[0-9]+$ ]]; then return 0; fi
  if ! kill -0 "$pid" 2>/dev/null; then return 0; fi

  # Try SIGTERM first
  kill -TERM "-$pid" 2>/dev/null || kill -TERM "$pid" 2>/dev/null || true

  # Wait up to grace seconds
  local end=$((SECONDS + grace))
  while kill -0 "$pid" 2>/dev/null && (( SECONDS < end )); do
    sleep 0.5
  done

  # Then SIGKILL if still alive
  if kill -0 "$pid" 2>/dev/null; then
    kill -KILL "-$pid" 2>/dev/null || kill -KILL "$pid" 2>/dev/null || true
  fi
}

kill_node_procs() {
  # Kill processes whose cwd == script_dir and are likely spawn types
  ps -eo pid,comm,ppid --no-headers | while read -r pid comm ppid; do
    if [[ -d "/proc/$pid/cwd" ]]; then
      local cwd
      cwd="$(readlink -f "/proc/$pid/cwd" 2>/dev/null || true)"
      if [[ "$cwd" == "$script_dir" ]]; then
        case "$comm" in
          python|python3|bash|sh|node)
            local pcomm
            pcomm="$(ps -p "$ppid" -o comm= 2>/dev/null || echo unknown)"
            case "$pcomm" in
              bash|sh|run_rl_swarm.sh|node|npm)
                kill_tree_graceful "$pid" 5
              ;;
            esac
          ;;
        esac
      fi
    fi
  done

  # Kill by pattern list
  for pat in "${NUKE_PATTERNS[@]}"; do
    pkill -9 -f "$pat" 2>/dev/null || true
  done
}

check_errors_in_log() {
  local logf="$1"
  [[ -f "$logf" ]] || return 1
  for kw in "${KEYWORDS[@]}"; do
    grep $grep_flags -- "$kw" "$logf" && return 0
  done
  return 1
}

is_idle() {
  local logf="$1"
  [[ -f "$logf" ]] || return 1
  local mtime now idle
  mtime="$(file_mtime "$logf")"
  now="$(date +%s)"
  idle=$(( now - mtime ))
  (( idle > IDLE_SECS ))
}

# ---------- Traps ----------
RUN_PID=""
cleanup() {
  log INFO "Cleaning up..."
  if [[ -n "$RUN_PID" ]]; then
    kill_tree_graceful "$RUN_PID" 5
  fi
  kill_node_procs
}
trap cleanup EXIT INT TERM

# ---------- Ensure single instance ----------
if ! command -v flock >/dev/null 2>&1; then
  log WARN "flock not found—continuing without single-instance guard."
else
  exec 9>"$lockfile"
  if ! flock -n 9; then
    log ERR "Another guardian is already running (lock: $lockfile). Exiting."
    exit 1
  fi
fi

# ---------- Main loop ----------
backoff="$BACKOFF_START"

while true; do
  log INFO "Starting Gensyn node (${SCRIPT})"
  rotate_log "$TMP_LOG"

  # Auto-answer input same as your script
  set -m
  ( sleep 1 && printf "n\n\n\n" ) | bash "$SCRIPT" 2>&1 | tee -a "$TMP_LOG" &
  RUN_PID=$!
  set +m

  log INFO "Spawned child pid=$RUN_PID (logging to $TMP_LOG)"

  while kill -0 "$RUN_PID" 2>/dev/null; do
    sleep 5

    if is_idle "$TMP_LOG"; then
      log WARN "Idle for > ${IDLE_SECS}s. Restarting."
      kill_tree_graceful "$RUN_PID" 10
      RUN_PID=""
      kill_node_procs
      break
    fi

    if check_errors_in_log "$TMP_LOG"; then
      log WARN "Error keyword detected in log. Restarting."
      kill_tree_graceful "$RUN_PID" 10
      RUN_PID=""
      kill_node_procs
      break
    fi
  done

  # If we broke out because child died or we killed it, wait a moment
  sleep 1

  # Exponential backoff (caps at BACKOFF_MAX)
  log INFO "Backing off for ${backoff}s before restart..."
  sleep "$backoff"
  if (( backoff < BACKOFF_MAX )); then
    backoff=$(( backoff * 2 ))
    if (( backoff > BACKOFF_MAX )); then backoff="$BACKOFF_MAX"; fi
  fi
done
