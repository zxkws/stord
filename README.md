# stord

Single-file storage deploy helper (MariaDB / MongoDB / Redis) for a small server.

## Install (one-liner)

```bash
curl -fsSL https://raw.githubusercontent.com/zxkws/stord/main/stord | bash
```

This installs `stord` into:
- root: `/usr/local/bin/stord`
- non-root: `~/.local/bin/stord` (add it to `PATH` if needed)

## Run

```bash
stord
```

## Update

- Auto prompt: when you open the interactive menu, it will check remote version once and ask whether to update.
- Manual:

```bash
stord check-update
stord self-update
```

## Notes

- Default data dir: `/opt/stord` (if you run without root, it will提示权限并建议 `sudo -E stord`；你也可以用 `STORD_HOME=/path` 改到用户目录).
