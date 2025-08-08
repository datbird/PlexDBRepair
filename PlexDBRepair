#!/bin/bash
# PlexDBRepair.sh - Interactive DB check/repair for Plex (bare metal & Docker)
# Works with: Standard Linux (systemd), plexinc/pms-docker, binhex-plex, and custom paths.
# Run as root. Designed for Unraid & general Linux.

set -u

# ---------- Helpers ----------
die() { echo "ERROR: $*" >&2; exit 1; }
info() { echo -e "\n==> $*\n"; }
warn() { echo -e "\n[WARN] $*\n"; }
confirm() { local p="${1:-Proceed?} [y/N]: "; read -r -p "$p" ans; [[ "${ans,,}" =~ ^y(es)?$ ]]; }
prompt_default() { local var="$1" def="$2" q="$3" r; read -r -p "$q [$def]: " r; eval "$var=\"${r:-$def}\""; }
exists() { command -v "$1" >/dev/null 2>&1; }

get_env_from_container() {
  # $1=container $2=VAR name ; prints value or empty
  docker inspect "$1" --format '{{range .Config.Env}}{{println .}}{{end}}' \
    | awk -F= -v k="$2" '$1==k { $1=""; sub(/^=/,""); print; }'
}

get_service_user() {
  # Best effort to find the systemd User= for plexmediaserver
  local u
  u=$(systemctl cat plexmediaserver 2>/dev/null | awk -F= '/^User=/{print $2; exit}')
  [[ -n "$u" ]] && { echo "$u"; return; }
  # Fallback: process owner (if running)
  u=$(ps -eo user,comm | awk '$2 ~ /Plex Media Server/ {print $1; exit}')
  echo "${u:-plex}"
}

ensure_rw_config_mount() {
  # Docker-only
  local c="$1"
  local rw src
  rw=$(docker inspect "$c" --format '{{range .Mounts}}{{if eq .Destination "/config"}}{{.RW}}{{end}}{{end}}')
  src=$(docker inspect "$c" --format '{{range .Mounts}}{{if eq .Destination "/config"}}{{.Source}}{{end}}{{end}}')
  if [[ -z "$rw" ]]; then
    warn "Could not determine /config mount for container $c."
  else
    echo "INFO: /config -> $src (rw=$rw)"
    if [[ "$rw" != "true" ]]; then
      warn "/config is mounted read-only. Edit the Docker template to remove :ro or set RW."
    fi
  fi
}

find_or_stage_plex_sqlite_from_container() {
  # Copy Plex SQLite out of the container if not present on host
  # $1=container, $2=host_target_path -> sets global PQL
  local c="$1" target="$2" plugdir
  if [[ -x "$target" ]]; then PQL="$target"; return 0; fi
  info "Plex SQLite not found at $target. Attempting to copy from container $c..."
  docker start "$c" >/dev/null || true
  plugdir=$(docker exec "$c" sh -lc 'ls -d "/usr/lib/plexmediaserver/Resources/Plug-ins-"* 2>/dev/null | head -n1') || true
  [[ -z "$plugdir" ]] && die "Could not find Resources/Plug-ins-* inside container."
  mkdir -p "$(dirname "$target")"
  docker cp "$c":"$plugdir/Plex SQLite" "$target" || die "docker cp failed."
  chmod +x "$target"
  docker stop "$c" >/dev/null || true
  PQL="$target"
}

find_plex_sqlite_on_host() {
  # Bare metal default path
  local p
  p=$(ls -d /usr/lib/plexmediaserver/Resources/Plug-ins-* 2>/dev/null | head -n1)
  if [[ -n "$p" && -x "$p/Plex SQLite" ]]; then
    PQL="$p/Plex SQLite"
    return 0
  fi
  return 1
}

run_sql() { "$PQL" "$DB" "$1"; }

dump_and_rebuild() {
  local ts="$1"
  cd "$DBDIR" || die "cd DBDIR failed"
  mv -v com.plexapp.plugins.library.db "com.plexapp.plugins.library.broken.$ts.db" || die "rename failed"
  info "Dumping old DB to SQL (this can take a while)..."
  "$PQL" "com.plexapp.plugins.library.broken.$ts.db" '.mode insert' '.output dump.sql' '.dump' '.exit' \
    || die "Dump failed (severe corruption?)."
  info "Rebuilding fresh DB from dump..."
  "$PQL" com.plexapp.plugins.library.db < dump.sql || die "Rebuild failed."
  rm -f dump.sql com.plexapp.plugins.library.db-shm com.plexapp.plugins.library.db-wal
}

fix_perms_bare() {
  # Use the filesystem owner of "Library" or the service user, whichever is safer
  local libdir="$CONFIG/Library"
  local uid gid owner
  if [[ -d "$libdir" ]]; then
    uid=$(stat -c %u "$libdir") || true
    gid=$(stat -c %g "$libdir") || true
  fi
  if [[ -z "${uid:-}" || -z "${gid:-}" || "$uid" -eq 0 ]]; then
    owner=$(get_service_user)
    info "Chown to service user: $owner"
    chown -R "$owner":"$owner" "$CONFIG/Library" || warn "chown to $owner may have partially failed."
  else
    info "Chown to UID:GID $uid:$gid"
    chown -R "$uid":"$gid" "$CONFIG/Library" || warn "chown to $uid:$gid may have partially failed."
  fi
  rm -f "$DBDIR"/com.plexapp.plugins.library.db-{shm,wal}
  find "$CONFIG/Library" -type d -exec chmod 775 {} \; 2>/dev/null || true
  find "$DBDIR" -maxdepth 1 -type f -name 'com.plexapp.plugins.library.db*' -exec chmod 664 {} \; 2>/dev/null || true
}

fix_perms_binhex() {
  local c="$1" puid pgid
  puid=$(get_env_from_container "$c" "PUID"); pgid=$(get_env_from_container "$c" "PGID")
  [[ -z "$puid" ]] && puid=99
  [[ -z "$pgid" ]] && pgid=100
  echo "INFO: Using PUID=$puid PGID=$pgid (binhex)"
  chown -R "$puid:$pgid" "$CONFIG/Plex Media Server"
  rm -f "$DBDIR"/com.plexapp.plugins.library.db-{shm,wal}
  find "$CONFIG/Plex Media Server" -type d -exec chmod 775 {} \;
  find "$DBDIR" -maxdepth 1 -type f -name 'com.plexapp.plugins.library.db*' -exec chmod 664 {} \;
}

fix_perms_official() {
  # Try PLEX_UID/PLEX_GID, else adopt owner of CONFIG path
  local c="$1" ouid ogid
  local puid=$(get_env_from_container "$c" "PLEX_UID")
  local pgid=$(get_env_from_container "$c" "PLEX_GID")
  if [[ -n "$puid" && -n "$pgid" ]]; then
    echo "INFO: Using PLEX_UID=$puid PLEX_GID=$pgid (official image)"
    chown -R "$puid:$pgid" "$CONFIG/Plex Media Server"
  else
    ouid=$(stat -c %u "$CONFIG") || true
    ogid=$(stat -c %g "$CONFIG") || true
    if [[ -n "${ouid:-}" && -n "${ogid:-}" ]]; then
      echo "INFO: Using owner of CONFIG path UID:GID $ouid:$ogid"
      chown -R "$ouid:$ogid" "$CONFIG/Plex Media Server"
    else
      warn "Could not determine UID/GID. Skipping ownership change."
    fi
  fi
  rm -f "$DBDIR"/com.plexapp.plugins.library.db-{shm,wal}
  find "$CONFIG/Plex Media Server" -type d -exec chmod 775 {} \;
  find "$DBDIR" -maxdepth 1 -type f -name 'com.plexapp.plugins.library.db*' -exec chmod 664 {} \;
}

tail_logs() {
  local mode="$1" c="${2:-}"
  echo
  case "$mode" in
    docker)
      echo "---- docker logs (last 120s) ----"
      docker logs "$c" --since=120s 2>&1 | tail -200 || true
      echo "---------------------------------"
      ;;
    bare)
      echo "Tip: check /var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Logs/Plex Media Server.log"
      ;;
  esac
  local LOGS="$DBDIR/../..//Logs/Plex Media Server.log"
  LOGS="${CONFIG}/Plex Media Server/Logs/Plex Media Server.log"
  if [[ -f "$LOGS" ]]; then
    echo
    echo "---- Plex Media Server.log (last 100 lines) ----"
    tail -n 100 "$LOGS"
    echo "-----------------------------------------------"
  fi
}

# ---------- Menu ----------
echo "Plex DB Repair / Maintenance"
echo "Choose your setup:"
echo "  1) Standard Linux Plex Install (systemd/bare metal)"
echo "  2) Plex Official Docker (plexinc/pms-docker)"
echo "  3) binhex-plex (Docker, Unraid)"
echo "  4) Custom Plex Path"
read -r -p "Enter choice [1-4]: " CHOICE

INSTALL_MODE=""
CONTAINER=""
CONFIG=""
PQL=""
PQL_PATH=""
DBDIR=""
DB=""

case "$CHOICE" in
  1)
    INSTALL_MODE="bare"
    # Common defaults
    DEF_CONFIG="/var/lib/plexmediaserver"
    if [[ ! -d "$DEF_CONFIG/Library" && -d "/var/lib/plexmediaserver/Library/Application Support" ]]; then
      DEF_CONFIG="/var/lib/plexmediaserver"
    fi
    prompt_default CONFIG "$DEF_CONFIG" "Plex base path (contains 'Library')"
    DBDIR="$CONFIG/Library/Application Support/Plex Media Server/Plug-in Support/Databases"
    [[ -d "$DBDIR" ]] || die "DB directory not found: $DBDIR"
    if find_plex_sqlite_on_host; then
      PQL_PATH="$PQL"
    else
      # Let the user point us to it
      DEF_PQL="/usr/lib/plexmediaserver/Resources/Plug-ins-*/Plex SQLite"
      prompt_default PQL_PATH "$DEF_PQL" "Path to 'Plex SQLite' on host"
      [[ -e "$PQL_PATH" ]] || die "Plex SQLite not found: $PQL_PATH"
      PQL="$PQL_PATH"
    fi
    ;;
  2)
    INSTALL_MODE="docker_official"
    # Try to auto-pick container
    CANDIDATES=$(docker ps -a --format '{{.Names}} {{.Image}}' | awk '$2 ~ /plexinc\/pms-docker/ {print $1}')
    if [[ -z "$CANDIDATES" ]]; then
      DEF_CONTAINER="plex"
    else
      DEF_CONTAINER=$(echo "$CANDIDATES" | head -n1)
      echo "Detected official containers:"
      echo "$CANDIDATES" | sed 's/^/  - /'
    fi
    prompt_default CONTAINER "${DEF_CONTAINER:-plex}" "Container name"
    # Resolve /config path on host
    CONFIG=$(docker inspect "$CONTAINER" --format '{{range .Mounts}}{{if eq .Destination "/config"}}{{.Source}}{{end}}{{end}}')
    [[ -n "$CONFIG" ]] || prompt_default CONFIG "/mnt/user/appdata/plex" "Host path for /config"
    DBDIR="$CONFIG/Plex Media Server/Plug-in Support/Databases"
    [[ -d "$DBDIR" ]] || die "DB directory not found: $DBDIR"
    # Stage Plex SQLite to a stable host path
    PQL_PATH="$CONFIG/databasetools/plex_sqlite"
    find_or_stage_plex_sqlite_from_container "$CONTAINER" "$PQL_PATH"
    ;;
  3)
    INSTALL_MODE="docker_binhex"
    DEF_CONTAINER="binhex-plex"
    DEF_CONFIG="/mnt/user/appdata/binhex-plex"
    DEF_PQL="$DEF_CONFIG/databasetools/plexmediaserver/Plex SQLite"
    prompt_default CONTAINER "$DEF_CONTAINER" "Container name"
    prompt_default CONFIG "$DEF_CONFIG" "Host path for /config"
    prompt_default PQL_PATH "$DEF_PQL" "Path to 'Plex SQLite' on host"
    if [[ ! -x "$PQL_PATH" ]]; then
      # Attempt to copy from container
      find_or_stage_plex_sqlite_from_container "$CONTAINER" "$PQL_PATH"
    else
      PQL="$PQL_PATH"
    fi
    DBDIR="$CONFIG/Plex Media Server/Plug-in Support/Databases"
    [[ -d "$DBDIR" ]] || die "DB directory not found: $DBDIR"
    ;;
  4)
    INSTALL_MODE="custom"
    read -r -p "Is this a Docker container? [y/N]: " IS_DOCKER
    if [[ "${IS_DOCKER,,}" =~ ^y(es)?$ ]]; then
      prompt_default CONTAINER "plex" "Container name"
      CONFIG=$(docker inspect "$CONTAINER" --format '{{range .Mounts}}{{if eq .Destination "/config"}}{{.Source}}{{end}}{{end}}')
      if [[ -z "$CONFIG" ]]; then
        prompt_default CONFIG "/mnt/user/appdata/plex" "Host path for /config"
      fi
      DBDIR="$CONFIG/Plex Media Server/Plug-in Support/Databases"
      [[ -d "$DBDIR" ]] || die "DB directory not found: $DBDIR"
      PQL_PATH="$CONFIG/databasetools/plex_sqlite"
      if [[ ! -x "$PQL_PATH" ]]; then
        find_or_stage_plex_sqlite_from_container "$CONTAINER" "$PQL_PATH"
      else
        PQL="$PQL_PATH"
      fi
      INSTALL_MODE="docker_custom"
    else
      prompt_default CONFIG "/var/lib/plexmediaserver" "Plex base path (contains 'Library')"
      DBDIR="$CONFIG/Library/Application Support/Plex Media Server/Plug-in Support/Databases"
      [[ -d "$DBDIR" ]] || die "DB directory not found: $DBDIR"
      if ! find_plex_sqlite_on_host; then
        prompt_default PQL_PATH "/usr/lib/plexmediaserver/Resources/Plug-ins-*/Plex SQLite" "Path to 'Plex SQLite' on host"
        [[ -e "$PQL_PATH" ]] || die "Plex SQLite not found: $PQL_PATH"
        PQL="$PQL_PATH"
      fi
      INSTALL_MODE="bare_custom"
    fi
    ;;
  *)
    die "Invalid choice."
    ;;
esac

DB="$DBDIR/com.plexapp.plugins.library.db"
[[ -f "$DB" ]] || die "DB file not found: $DB"

# ---------- Stop Plex ----------
case "$INSTALL_MODE" in
  bare|bare_custom)
    info "Stopping service: plexmediaserver"
    if exists systemctl; then systemctl stop plexmediaserver || die "Failed to stop plexmediaserver"; else service plexmediaserver stop || die "Failed to stop plexmediaserver"; fi
    ;;
  docker_* )
    info "Stopping container: $CONTAINER"
    docker stop "$CONTAINER" || die "Failed to stop container."
    ;;
esac

# ---------- Backup ----------
ts=$(date +%F_%H%M%S)
BACKUP_DIR=~/plex-db-backups/"$ts"
info "Backing up DB files to $BACKUP_DIR"
mkdir -p "$BACKUP_DIR"
cp -av "$DBDIR"/com.plexapp.plugins.library.db* "$BACKUP_DIR"/ || die "Backup copy failed."

# ---------- Integrity checks ----------
info "Running PRAGMA integrity_check..."
IC_OUT=$(run_sql 'PRAGMA integrity_check;' 2>&1) || true
echo "$IC_OUT"
info "Running PRAGMA quick_check..."
QC_OUT=$(run_sql 'PRAGMA quick_check;' 2>&1) || true
echo "$QC_OUT"

IC_OK=false
if echo "$IC_OUT" | grep -qx "ok"; then IC_OK=true; fi

# ---------- Light repair or rebuild ----------
if $IC_OK; then
  info "Integrity looks OK."
  if confirm "Run light maintenance (REINDEX; VACUUM;)?"; then
    info "Running REINDEX; VACUUM;"
    run_sql 'REINDEX; VACUUM;' || warn "REINDEX/VACUUM failed."
  fi
else
  info "Integrity is NOT OK (see output above)."
  if confirm "Attempt dump & rebuild? (recommended)"; then
    dump_and_rebuild "$ts"
    info "Re-checking integrity after rebuild..."
    echo "$(run_sql 'PRAGMA integrity_check;')"
  else
    warn "Skipping rebuild at your request."
  fi
fi

# ---------- Permissions & mount checks ----------
case "$INSTALL_MODE" in
  bare|bare_custom)
    info "Fixing ownership/permissions (bare metal)..."
    fix_perms_bare
    ;;
  docker_binhex)
    info "Fixing ownership/permissions (binhex)..."
    fix_perms_binhex "$CONTAINER"
    info "Checking /config mount is read-write..."
    ensure_rw_config_mount "$CONTAINER"
    ;;
  docker_official|docker_custom)
    info "Fixing ownership/permissions (official/custom docker)..."
    fix_perms_official "$CONTAINER"
    info "Checking /config mount is read-write..."
    ensure_rw_config_mount "$CONTAINER"
    ;;
esac

# ---------- Start Plex ----------
case "$INSTALL_MODE" in
  bare|bare_custom)
    info "Starting service: plexmediaserver"
    if exists systemctl; then systemctl start plexmediaserver || die "Failed to start plexmediaserver"; else service plexmediaserver start || die "Failed to start plexmediaserver"; fi
    ;;
  docker_* )
    info "Starting container: $CONTAINER"
    docker start "$CONTAINER" || die "Failed to start container."
    ;;
esac

# ---------- Optional post-start integrity check ----------
if confirm "Run a post-start integrity_check (non-invasive)?"; then
  case "$INSTALL_MODE" in
    bare|bare_custom)
      # Stop briefly to ensure truly offline check
      if exists systemctl; then systemctl stop plexmediaserver; else service plexmediaserver stop; fi
      echo "$(run_sql 'PRAGMA integrity_check;')"
      fix_perms_bare
      if exists systemctl; then systemctl start plexmediaserver; else service plexmediaserver start; fi
      ;;
    docker_* )
      docker stop "$CONTAINER" >/dev/null 2>&1 || true
      echo "$(run_sql 'PRAGMA integrity_check;')"
      case "$INSTALL_MODE" in
        docker_binhex) fix_perms_binhex "$CONTAINER" ;;
        *) fix_perms_official "$CONTAINER" ;;
      esac
      docker start "$CONTAINER" >/dev/null 2>&1 || true
      ;;
  esac
fi

# ---------- Logs ----------
case "$INSTALL_MODE" in
  bare|bare_custom) tail_logs bare ;;
  docker_* ) tail_logs docker "$CONTAINER" ;;
esac

info "Done. Backups are in: $BACKUP_DIR"
echo "Tip: If library updates still stall, consider bumping inotify watches on the host:"
echo "  echo 'sysctl -w fs.inotify.max_user_watches=524288' >> /boot/config/go && reboot"
