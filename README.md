# aur-check-updates

`aurcheck` tells you which of the AUR packages installed on your Arch system have
a newer version available, and prints a step-by-step guide for updating one by
hand.

It is **read-only by design**. It never builds, installs, upgrades, or touches
the pacman database, and it never asks for root. If you deliberately keep an AUR
helper off your machine so that nothing can install or upgrade packages behind
your back, this fills the one gap that leaves: knowing *when* something is out of
date, while you stay the one who runs `makepkg`.

```
$ aurcheck
AUR (2 foreign packages)
  ↑ brave-bin                    1:1.93.136-1 → 1:1.94.121-1
      https://aur.archlinux.org/packages/brave-bin
  ↑ r8125-dkms                   9.017.01-1 → 9.018.00-1
      https://aur.archlinux.org/packages/r8125-dkms

2 update(s) available. Run aurcheck update for instructions.
```

## How it works

Three steps, no magic:

1. **`pacman -Qm`** lists your *foreign* packages — everything installed that no
   configured sync repo provides. In practice that is your AUR packages, plus
   anything you built locally.
2. **One HTTPS request** per 100 packages to the official AUR RPC
   (`https://aur.archlinux.org/rpc/v5/info`), which returns the current version
   of each one as JSON. No repo is cloned and no PKGBUILD is downloaded.
3. **`vercmp`** — the same version comparator pacman itself uses — decides
   whether the AUR version is newer. That is why epochs like `1:1.93.136-1` are
   handled correctly, which naive string comparison gets wrong.

Nothing is cached and nothing is written to disk.

### What it also flags

- `[flagged out-of-date]` — someone has flagged the package as out of date in
  the AUR. The maintainer may not have pushed the new version yet.
- `[orphaned]` — the package has no maintainer. Worth extra care when reading
  the PKGBUILD.
- `[VCS package: the AUR version is not a reliable signal]` — for `-git`,
  `-svn`, `-hg` and `-bzr` packages, whose recorded version comes from the last
  build rather than from upstream. A comparison there means little.
- `not in the AUR (built locally?)` — a foreign package with no AUR entry, so
  there is nothing to compare it against.

## Requirements

Everything here is either in `base` or almost certainly already installed:

| Tool     | Provided by      | Used for                          |
| -------- | ---------------- | --------------------------------- |
| `bash`   | `bash`           | the script itself                 |
| `curl`   | `curl`           | querying the AUR RPC              |
| `jq`     | `jq`             | parsing the JSON response         |
| `vercmp` | `pacman`         | comparing versions                |
| `git`    | `git`            | only for commands the guide suggests you run |

No AUR helper, no Python, no runtime with a dependency tree.

## Install

### Recommended: for your user only, no root

`~/.local/bin` is on the default `PATH` on Arch and needs no privileges:

```bash
mkdir -p ~/.local/bin
curl -fsSL -o ~/.local/bin/aurcheck \
  https://raw.githubusercontent.com/joaquin021/aur-check-updates/main/aurcheck
chmod +x ~/.local/bin/aurcheck
```

Check that it is reachable:

```bash
command -v aurcheck && aurcheck --version
```

If the command is not found, `~/.local/bin` is missing from your `PATH`; add
`export PATH="$HOME/.local/bin:$PATH"` to your `~/.bashrc` or `~/.zshrc`.

### System-wide

Only if you want it available to every user on the machine:

```bash
sudo curl -fsSL -o /usr/local/bin/aurcheck \
  https://raw.githubusercontent.com/joaquin021/aur-check-updates/main/aurcheck
sudo chmod +x /usr/local/bin/aurcheck
```

### Read it before you run it

It is a single self-contained shell script, so you can and should look at it
first:

```bash
curl -fsSL https://raw.githubusercontent.com/joaquin021/aur-check-updates/main/aurcheck | less
```

### From a clone

```bash
git clone https://github.com/joaquin021/aur-check-updates.git
cd aur-check-updates
install -Dm755 aurcheck ~/.local/bin/aurcheck
```

### Update or uninstall

Updating is the same `curl` command again. Uninstalling is deleting one file:

```bash
rm ~/.local/bin/aurcheck        # or: sudo rm /usr/local/bin/aurcheck
```

## Usage

```
aurcheck [check]              list installed AUR packages that have a newer
                              version in the AUR (default command)
aurcheck update [package...]  print a step-by-step guide for updating by hand;
                              with no package, covers everything out of date
aurcheck list                 list installed foreign packages, one per line

  -q, --quiet     only print lines for packages that have an update
  -v, --verbose   also list packages that are already up to date
      --no-color  disable colors (also disabled automatically when piped)
  -h, --help      show help
  -V, --version   show version
```

### Exit codes

| Code | Meaning                          |
| ---- | -------------------------------- |
| `0`  | nothing to update / success      |
| `2`  | updates are available            |
| `1`  | error (no network, bad argument) |

A separate code for "there are updates" keeps it easy to script, without having
to parse the output:

```bash
out=$(aurcheck -q --no-color)
case $? in
  0) ;;                                          # nothing to do
  2) notify-send "AUR updates" "$out" ;;
  *) echo "aurcheck failed" >&2 ;;
esac
```

### The update guide

`aurcheck update <package>` prints what you would do to update it, and stops
there. It runs none of it:

```
$ aurcheck update brave-bin

══ brave-bin ══
  installed 1:1.93.136-1 → AUR 1:1.94.121-1
  page  https://aur.archlinux.org/packages/brave-bin

  1. Get the build files
    mkdir -p /home/you/.cache/aurcheck
    git clone https://aur.archlinux.org/brave-bin.git /home/you/.cache/aurcheck/brave-bin
    cd /home/you/.cache/aurcheck/brave-bin

  2. Review before you build  ← this is the whole point of doing it by hand
    less PKGBUILD
    ls -la; cat *.install 2>/dev/null
    # a PKGBUILD is a shell script and *.install runs as root at install time.
    ...
```

The guide adapts to what it finds:

- **Already cloned?** It gives you `git fetch` plus `git log`/`git diff` against
  `origin/master` so you review only what changed since your last build, then a
  `git merge --ff-only` so a force-push cannot rewrite your tree unnoticed.
- **Split package?** The AUR git repo is named after the *pkgbase*, not the
  package, so the clone URL it prints uses the right one.
- **DKMS package?** It adds a note about kernel headers, `dkms status`, and the
  fact that reloading a network driver drops the link.
- **VCS package?** It says plainly that the version number is not a useful
  signal there.
- **Flagged out of date or orphaned?** It says so before you start.

Run `aurcheck update` with no arguments to get a guide for every package that is
currently behind.

Set `AURCHECK_BUILD_DIR` to change the directory the guide suggests building in:

```bash
AURCHECK_BUILD_DIR=~/src/aur aurcheck update brave-bin
```

## Checking automatically

### A shell prompt or status bar

`-q` prints one line per available update and nothing else:

```bash
aurcheck -q
```

### A systemd user timer

Check once a day and get a desktop notification, without any of it running as
root. `~/.config/systemd/user/aurcheck.service`:

```ini
[Unit]
Description=Check for AUR updates
After=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c '%h/.local/bin/aurcheck -q --no-color > %t/aurcheck.txt; \
  test -s %t/aurcheck.txt && notify-send "AUR updates" "$(cat %t/aurcheck.txt)"'
```

`~/.config/systemd/user/aurcheck.timer`:

```ini
[Unit]
Description=Daily AUR update check

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
```

```bash
systemctl --user enable --now aurcheck.timer
```

`RandomizedDelaySec` spreads the load on the AUR servers — please keep it, and
do not check more often than a few times a day.

## Notes and limitations

- It compares **installed vs. AUR** versions. It says nothing about the official
  repos; `checkupdates` from `pacman-contrib` covers those.
- A foreign package with no AUR entry is reported as such rather than skipped
  silently, since that is usually something you built yourself.
- VCS packages (`-git` and friends) cannot be meaningfully compared this way.
  Rebuild them when you want them newer.
- The AUR version being newer does not mean the build works. Read the recent
  comments on the package page before building.
- Reading the PKGBUILD before every build is the actual security boundary here.
  A PKGBUILD is an arbitrary shell script, and `.install` files run as root. The
  guide exists to make that step routine, not to be skipped.

## License

MIT. See [LICENSE](LICENSE).
