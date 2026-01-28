# Gitea Repository Backup

This tool backs up (mirrors) every repository your **Gitea account token can access** into a local folder. It will:

- **Clone** repos that aren’t present locally yet
- **Update** repos you already have (fetch + pull)
- **Archive** repos that disappear from your accessible repo list (deleted, moved, or permission removed)

It is safe to run repeatedly. :contentReference[oaicite:1]{index=1}

---

## MVP: Setup / Install (do this first)

### 1) Get the files

Put these files in a folder (or clone the tool repo):

- `gitea_repository_backup.py`
- `config_default.json`
- `requirements.txt` :contentReference[oaicite:2]{index=2}

### 2) Install dependencies

```bash
pip install -r requirements.txt
````

### 3) Create a Gitea token (required)

In Gitea:

* User Settings → Applications → Generate New Token

Recommended minimum:

* `repository: Read`
* `user: Read`
* `organization: Read` (optional; some installs need it for org visibility)

Copy the token immediately (you usually can’t view it again). 

### 4) Put the token in an environment variable (recommended)

Default env var: `GITEA_TOKEN` (controlled by `gitea.token_env`)

PowerShell:

```powershell
$env:GITEA_TOKEN="YOUR_TOKEN_HERE"
```

cmd.exe:

```bat
set GITEA_TOKEN=YOUR_TOKEN_HERE
```

> You *can* store the token in `config.json` (`gitea.token`), but env var is safer. 

---

## MVP: First run (creates config)

Run:

```bash
python gitea_repository_backup.py
```

On first run, the tool will:

* Create `config.json` automatically (copied from `config_default.json`)

Then you can edit settings and run again. 

---

## MVP: Minimal config you must set

Open `config.json` and set at minimum:

* `gitea.base_url` (your Gitea site URL)
* `sync.output_dir` (where backups go)

Example:

```json
{
  "gitea": { "base_url": "https://git.example.com" },
  "sync": { "output_dir": "D:/GiteaBackup" }
}
```

---

## MVP: Run safely (dry-run) then run live

### Dry-run (recommended first)

Dry-run prints what it *would* do, without changing your filesystem.

If you are using CLI flags:

```bash
python gitea_repository_backup.py --dry-run
```

### Live run

```bash
python gitea_repository_backup.py --no-dry-run
```

> If your build is GUI-controlled (customtkinter), use **Run DRY** then **Run LIVE** from the UI instead of flags. 

---

## What gets created in your backup folder

Your configured `sync.output_dir` becomes the backup root.

Inside it, the tool creates:

* Your repo folders (default layout: `owner/repo`)
* `_repo_sync_manifest.json` (tracks what the tool manages)
* `_archive/` (where removed/missing repos are moved)

Example:

````
D:/GiteaBackup/
  alice/
    cool-project/
  bob/
    infra/
  _repo_sync_manifest.json
  _archive/
    20260102_140233/
      bob/old-repo/
``` :contentReference[oaicite:7]{index=7}

---

## Keep your config private (highly recommended)

`config.json` is intended to be **local-only** (it may include private paths or secrets).

If you’re using this tool from inside a git repo, add this to `.gitignore`:

```gitignore
config.json
_repo_sync_manifest.json
``` :contentReference[oaicite:8]{index=8}

---

## Common “it works” choices (recommended defaults)

### Git protocol: SSH (recommended)

In `config.json`:

```json
{
  "git": { "protocol": "ssh" }
}
````

You must have:

* SSH keys set up locally
* Your public key added to Gitea 

### Sync layout: owner/repo (recommended)

```json
{
  "sync": { "layout": "owner/repo" }
}
```

This avoids name collisions and keeps a clean structure. 

---

## Advanced: SSH key + port control

### Force a specific SSH key (config-driven)

```json
{
  "git": {
    "protocol": "ssh",
    "ssh": { "key_path": "C:/Users/you/.ssh/id_ed25519_gitea" }
  }
}
```

### Custom SSH port (global default)

```json
{
  "git": {
    "protocol": "ssh",
    "ssh": {
      "key_path": "C:/Users/you/.ssh/id_ed25519_gitea",
      "port": 222
    }
  }
}
```

### Per-host SSH overrides (recommended with multiple servers)

```json
{
  "git": {
    "protocol": "ssh",
    "ssh": {
      "key_path": "C:/Users/you/.ssh/id_ed25519_default",
      "port": 22,
      "host_overrides": {
        "git.example.com": {
          "key_path": "C:/Users/you/.ssh/id_ed25519_example",
          "port": 222
        }
      }
    }
  }
}
```

**Port precedence:**

1. Port embedded in `ssh_url` (example `ssh://...:222/...`)
2. `host_overrides[host].port`
3. `git.ssh.port`

**Key precedence:**

1. `host_overrides[host].key_path`
2. `git.ssh.key_path` 

---

## Advanced: HTTPS mode (if you can’t/won’t use SSH)

Set:

```json
{
  "git": { "protocol": "https" }
}
```

### HTTPS auth modes

#### 1) prompt (default)

Git will prompt for credentials as needed.

#### 2) extraheader (unattended / automation-friendly)

```json
{
  "git": {
    "protocol": "https",
    "https_auth": {
      "mode": "extraheader",
      "extraheader_basic_user": "your-username"
    }
  }
}
```

Security note: treat logs carefully; don’t paste them publicly. 

---

## Advanced: Updating behavior

When a repo already exists locally, the tool will typically:

* `git fetch origin --prune` (if enabled)
* checkout the repo’s default branch (if available)
* `git pull --ff-only`

### Dirty repos

If the tool detects local changes (uncommitted work), it will **skip updating that repo** by default to avoid overwriting anything. 

---

## Advanced: Archiving behavior (when repos “disappear”)

A repo is archived when it:

* was previously managed (tracked in manifest), and later
* is not present in your API-accessible repo list

When that happens, the local folder is moved into:

* `<output_dir>/_archive/<timestamp>/...`

This prevents accidental deletion while keeping your backup folder clean. 

---

## Advanced: Config reference (all options)

Config is loaded in this order:

1. `config_default.json` (template, committed)
2. `config.json` (local overrides, auto-created, should be gitignored)
3. CLI flags (highest priority) 

### Top-level

* `dry_run` (bool): if true, prints actions but does not modify filesystem.

### `gitea`

* `base_url` (string): `https://git.example.com`
* `api_base_path` (string): default `"/api/v1"`
* `token_env` (string): default `"GITEA_TOKEN"`
* `token` (string): optional fallback; env var wins
* `verify_tls` (bool): default `true`
* `timeout_s` (int): default `30`
* `page_limit` (int): default `50`
* `user_agent` (string): optional

### `sync`

* `output_dir` (string): required
* `layout` (string): `"owner/repo"` (recommended), `"owner__repo"`, `"flat"` (collision risk)
* `archive_dir_name` (string): default `"_archive"`
* `manifest_file_name` (string): default `"_repo_sync_manifest.json"`
* `clone_missing` (bool): default `true`
* `update_existing` (bool): default `true`
* `archive_missing_remote` (bool): default `true`
* `archive_stamp_format` (string): default `"%Y%m%d_%H%M%S"`

### `git`

* `executable` (string): default `"git"` (use absolute path if needed)
* `protocol` (string): `"ssh"` or `"https"`
* `fetch_prune` (bool): default `true`
* `reset_hard_to_origin` (bool): default `false` (more destructive; usually keep false for backups)
* `on_dirty_repo` (string): default `"skip"`

### `git.ssh` (only if `git.protocol="ssh"`)

* `key_path` (string): if set, forces key via `core.sshCommand`
* `port` (int): global default port
* `identity_only` (bool): default `true`
* `host_overrides` (object): per-host `{ key_path, port }`

### `git.https_auth` (only if `git.protocol="https"`)

* `mode` (string): `"prompt"` or `"extraheader"`
* `extraheader_basic_user` (string): required for `"extraheader"` 

---

## Troubleshooting

### “No token provided”

* Set your token env var (default: `GITEA_TOKEN`)
* Or (not recommended) put token in `config.json` under `gitea.token` 

### Auth failures when cloning

* Try SSH first
* If using HTTPS, use `https_auth.mode = extraheader` and set `extraheader_basic_user`

### “Dirty repo; skipping update”

You have local changes inside that repo folder.
Commit/stash/clean it, or keep it as-is and accept it won’t update.

### Repo missing locally but not cloned

Check:

* `sync.clone_missing` is true
* `sync.output_dir` is valid and writable
* `git` is installed and on PATH 
