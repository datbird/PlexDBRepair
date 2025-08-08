# PlexDBRepair.sh

Interactive Plex database check/repair script for Linux & Docker (Unraid-friendly).  
Runs **safe, offline** integrity checks using **Plex’s own SQLite binary**, backs up your DB, offers light repair (REINDEX/VACUUM), can dump/rebuild if corruption is detected, and fixes permissions for common install types.

> 💾 **Safety first:** Every run creates a timestamped backup in `~/plex-db-backups/`.

---

## ✨ Features

- Works with:
  - **Standard Linux Plex Install** (systemd / bare-metal)
  - **Plex Official Docker** (`plexinc/pms-docker`)
  - **binhex-plex** (popular on Unraid)
  - **Custom Plex Path** (Docker or bare metal)
- Performs **offline** `PRAGMA integrity_check` / `quick_check`
- Optional **REINDEX; VACUUM;** light maintenance
- **Dump & rebuild** path for corrupted DBs
- Fixes ownership/perms:
  - binhex: respects **PUID/PGID**
  - official image: respects **PLEX_UID/PLEX_GID** or host owner
  - bare-metal: uses service user
- Verifies `/config` is **read-write**
- Tails Docker and Plex logs at the end

---

## 📦 Requirements

- Linux (Unraid or any standard distro)
- `bash`, `docker` (for Docker installs), `systemd` (for bare-metal service control)
- Run as **root** (needed to stop services/containers and fix permissions)

> Why not plain `sqlite3`? Plex uses custom FTS tokenizers/collations. The script uses the **Plex SQLite** binary bundled with Plex to avoid errors like `unknown tokenizer: collating`.

---

## 🔧 Installation

Save the script as `PlexDBRepair.sh`:

```bash
# In your repo or on the host:
chmod +x PlexDBRepair.sh
```

(If you’re on Windows, ensure LF line endings.)

---

## ▶️ Usage

```bash
sudo ./PlexDBRepair.sh
```

Pick your setup at the menu:

```
Plex DB Repair / Maintenance
Choose your setup:
  1) Standard Linux Plex Install (systemd/bare metal)
  2) Plex Official Docker (plexinc/pms-docker)
  3) binhex-plex (Docker, Unraid)
  4) Custom Plex Path
Enter choice [1-4]:
```

The script will:

1. Stop Plex (service or container)
2. Back up DB files (`com.plexapp.plugins.library.db*`)
3. Run `integrity_check` and `quick_check`
4. Offer light repair (**REINDEX; VACUUM;**)
5. If corruption is detected, offer **dump & rebuild**
6. Fix permissions and verify `/config` mount (Docker)
7. Restart Plex and show recent logs

Backups are stored at:

```
~/plex-db-backups/YYYY-MM-DD_HHMMSS/
```

---

## 🧠 What It Touches

- **Database path** (varies by install):
  - Docker: `<CONFIG>/Plex Media Server/Plug-in Support/Databases/com.plexapp.plugins.library.db`
  - Bare-metal: `<CONFIG>/Library/Application Support/Plex Media Server/Plug-in Support/Databases/...`
- **Plex SQLite binary**:
  - Bare-metal: `/usr/lib/plexmediaserver/Resources/Plug-ins-*/Plex SQLite`
  - Docker: auto-copied from the container to `<CONFIG>/databasetools/plex_sqlite` if missing
- **Permissions**:
  - binhex: `chown -R $PUID:$PGID "<CONFIG>/Plex Media Server"`
  - official: `chown -R $PLEX_UID:$PLEX_GID` (or host path owner)
  - bare-metal: chown to service user (e.g., `plex`)
  - Files `*.db{-wal,-shm}` cleaned before restart

---

## 🧩 Notes for Unraid

- binhex containers typically set `PUID=99`, `PGID=100` (nobody:users). The script detects these.
- Ensure your **appdata** share is writable and not mounted read-only.
- If library updates still stall after a clean DB:
  - Consider bumping inotify watches:

    ```bash
    echo 'sysctl -w fs.inotify.max_user_watches=524288' >> /boot/config/go
    # reboot to persist
    ```

---

## 🛠️ Troubleshooting

- **`attempt to write a readonly database`**  
  Ownership/perms wrong after offline work. The script’s **Fix permissions** step addresses this. Also confirm `/config` is `rw=true`.

- **`unknown tokenizer: collating`**  
  That’s plain `sqlite3`. Use **Plex SQLite** (the script handles this automatically).

- **Container won’t restart / config mount is ro**  
  Check the Docker template for a `:ro` on `/config` and remove it. The script prints the RW status.

- **Severe corruption**  
  Use the **dump & rebuild** flow. If the dump fails, you may need table-by-table salvage (open an issue with your logs).

---

## 🙋 FAQ

- **Is it safe?**  
  Yes. The script **always** makes a timestamped backup before touching your DB.

- **Will it delete my metadata or libraries?**  
  No. It only operates on the Plex library database files and standard temp pages (`-wal`/`-shm`). A dump/rebuild recreates the DB content from your existing DB.

- **Can I run this while Plex is running?**  
  No. It stops Plex to ensure a clean, offline DB.

---

## 📝 Contributing

PRs welcome! Ideas:
- Optional cache/bundles cleanup step
- Inotify tuning toggle
- Non-systemd service support improvements

---

## ⚖️ License

Add your preferred license (MIT/Apache-2.0/etc.) and include a `LICENSE` file in this repo.

---

## 🚨 Disclaimer

This script is provided “as is” without warranty. You’re responsible for your system. Always keep backups (the script creates one per run, but you should also back up your Plex config regularly).
