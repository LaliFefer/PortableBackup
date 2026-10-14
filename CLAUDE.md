\# PortableBackup



Portable, offline, encrypted backup tool for Windows 10/11. Runs from an external drive, no installation on the host PC. Built as a simple UI on top of a proven engine (restic). Do NOT write a custom backup engine.



\## Product summary



The user plugs in an external drive, double-clicks PortableBackup.exe, and backs up chosen folders incrementally. Settings persist on the drive until changed. Old versions and deleted files are kept per a retention policy. Everything is encrypted with a password. The tool never touches the network.



Typical data size: about 60 GB on C:. Planned drive: 1-2 TB external HDD, NTFS.



\## Hard rules (never violate)



1\. \*\*Zero network.\*\* No telemetry, no update checks, no HTTP libraries imported. Add a test that fails if `socket`, `requests`, `urllib.request`, or `http.client` is imported by project code.

2\. \*\*Never write the password to disk or logs.\*\* Pass it to restic only via the `RESTIC\_PASSWORD` environment variable of the child process. Mask it in any log or exception text.

3\. \*\*Never modify source data.\*\* Backup is read-only on the source. Restore never overwrites originals by default; restore target is always chosen explicitly.

4\. \*\*All paths relative to the executable's location\*\* (support any drive letter). Resolve via `sys.executable` when frozen, `\_\_file\_\_` otherwise.

5\. \*\*Never test on real user data.\*\* Tests use generated dummy folders under a temp directory only.

6\. \*\*Do not invent restic flags.\*\* Check `restic --help` / `restic <cmd> --help` of the bundled version before using any flag.

7\. \*\*Fail loudly and safely.\*\* On any unexpected error: stop, log, show a clear message, leave the repo untouched.



\## Stage A scope (build this first)



1\. \*\*Settings screen:\*\* folders to back up, exclusion list, retention policy. Stored in `config/settings.json` on the drive. Remembered until changed.

2\. \*\*"Back up now" button:\*\* incremental backup with progress bar (parse `restic backup --json`).

3\. \*\*Password encryption:\*\* chosen at repo creation. Warn clearly that a lost password cannot be recovered.

4\. \*\*Snapshots/versions:\*\* every backup is a snapshot. Browse by date, restore any file or folder from any snapshot, including deleted files.

5\. \*\*Restore UI:\*\* pick snapshot, pick files, pick destination folder (never default to the original location).

6\. \*\*Integrity check:\*\* run `restic check` after backup and report the result in plain language. Offer a periodic deeper check (`--read-data-subset`).

7\. \*\*Free-space check\*\* before starting; clear message if insufficient.

8\. \*\*Resume after interruption:\*\* a disconnected drive mid-backup must not corrupt the repo. The next run continues. Handle stale locks safely (`restic unlock` only after confirming no restic process is running).

9\. \*\*Open files:\*\* use `--use-fs-snapshot` (Volume Shadow Copy). Requires admin, so build the exe with a UAC admin manifest and explain the prompt to the user. If VSS fails, offer to continue without it and warn.

10\. \*\*Readable log\*\* in `logs/` (rotating) and an on-screen result (success / warning / failure).

11\. \*\*Safe eject\*\* prompt at the end.

12\. \*\*Drive identity:\*\* on first setup, record the drive's volume serial number in config; warn if the tool is run from a different drive (do not block, since portability is a feature).



\### Retention policy (defaults, user-editable)



keep-daily 7, keep-weekly 4, keep-monthly 12, keep-yearly 3. Apply with `restic forget --prune` only after a successful backup and a successful check. Show the user how much space will be freed before pruning (`--dry-run`).



\### Default exclusions



`$Recycle.Bin`, `System Volume Information`, `pagefile.sys`, `hiberfil.sys`, `swapfile.sys`, `C:\\Windows\\Temp`, `AppData\\Local\\Temp`, browser caches, `node\_modules`, and the backup drive itself (never back up the drive onto itself).



\## Stage B (only after A is stable and tested)



\- \*\*System image mode:\*\* launch Windows built-in `wbadmin` (needs admin) or Veeam Agent Free from the app; the app only starts, schedules, and shows status. Verify on a VM how `wbadmin` behaves with an external drive target before relying on it. Note: wbadmin images are same-machine restore only and are not encrypted, so store them on a BitLocker To Go protected partition.

\- Scheduling and a background drive-detection helper (optional).



\## Out of scope



Cloud backup, cross-machine software migration, Mac support, auto-update.



\## Architecture



```

PortableBackup/

├── PortableBackup.exe     UI (Python + CustomTkinter, packaged with PyInstaller --onefile --uac-admin)

├── engine/restic.exe      bundled binary, version pinned and SHA256 recorded in engine/VERSION.txt

├── config/settings.json

├── logs/

└── repo/                  restic encrypted repository

```



Source layout:



```

src/portablebackup/

&#x20; paths.py          path resolution

&#x20; config.py         load/save/validate settings (schema-validated, atomic writes)

&#x20; engine.py         restic wrapper: init\_repo, backup, list\_snapshots, restore, check, forget, unlock

&#x20; errors.py         typed exceptions (WrongPassword, DriveMissing, NoSpace, RepoLocked, Interrupted ...)

&#x20; disk.py           free space, volume serial, safe eject

&#x20; ui/               CustomTkinter screens (Hebrew labels, RTL-aware)

tests/

```



\- UI never reads or writes repo data directly. It only calls `engine.py`.

\- Run restic with `subprocess` (no shell=True), `--json` where supported, stream output line by line, run in a worker thread so the UI never freezes, support cancel.

\- Atomic config writes (write temp file, then replace).

\- UI language: Hebrew. Keep strings in one module for easy editing.



\## Testing requirements



pytest, using a temp dir with generated dummy files. Required tests before each milestone is considered done:



\- init repo, first backup, second backup stores only changes (check snapshot stats)

\- delete a file in the source, restore it from the older snapshot, compare SHA256 with the original

\- wrong password gives a clear error and no repo change

\- simulated interruption (kill restic mid-backup), then next backup succeeds and `check` passes

\- missing drive / path gives a clear error

\- low free space gives a clear error before starting

\- password never appears in logs (grep test)

\- no network imports (see hard rule 1)

\- retention: forget/prune keeps the expected snapshots



\## Workflow rules for Claude Code



\- Work on ONE milestone at a time. First propose a plan (files, tests), wait for approval, then write tests, then code.

\- Small commits after each passing milestone. Never commit the repo/ folder, logs, or settings containing paths.

\- After each milestone, summarize: what was built, what was tested, known limitations.

\- If a requirement is unclear or a restic behavior is uncertain, ask or verify with `--help`; do not guess.

\- Keep dependencies minimal and pinned (requirements.txt with exact versions).



\## Milestones



1\. restic wrapper (init, backup, list, restore, check) with tests, no UI

2\. Failure-mode tests (wrong password, disconnect, no space, lock)

3\. Config file and retention policy

4\. UI: main screen, settings, progress

5\. UI: restore

6\. VSS and free-space check

7\. Packaging to exe, test on a clean VM, note antivirus false-positive risk and document how to verify the exe hash

