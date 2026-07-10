# clip-bmp2pngd

A small background daemon that watches the WSLg (Wayland) clipboard and
re-encodes clipboard images from **BMP** back to **PNG**.

By Salim Gasmi (salim@gasmi.net)

## Why this exists

When you use [Claude Code](https://claude.com/claude-code) inside **WSL** on
Windows and take a screenshot (e.g. `Win`+`Shift`+`S`), Windows puts the image
on the clipboard as **BMP**. WSLg mirrors that clipboard into the Linux
(Wayland) side, but Claude Code doesn't recognize the `image/bmp` type when you
paste — so the screenshot never comes through.

`clip-bmp2pngd` fixes this transparently: it detects when a BMP image lands on
the clipboard, converts it to PNG, and writes the PNG back to the clipboard.
Now pasting into Claude Code (or anything else that expects PNG) just works.

## How it works

The daemon polls the clipboard once per second:

1. Checks whether the clipboard advertises an `image/bmp` type (`wl-paste -l`).
2. If so, reads the BMP data and hashes it (`sha256sum`) to detect *new*
   content and avoid reprocessing the same image.
3. Converts BMP → PNG with ImageMagick's `convert` and puts it back with
   `wl-copy --type image/png`.
4. Re-applies the PNG a second time after a short delay, because WSLg sometimes
   resyncs the Windows clipboard and overwrites the freshly-set PNG.

It runs detached via `setsid` and identifies its own process using a unique
marker string, so `start` / `status` / `stop` work reliably without PID files.

## Dependencies

| Package | Provides | Used for |
| --- | --- | --- |
| `imagemagick` | `convert` | Re-encode BMP → PNG |
| `coreutils` | `sha256sum` | Detect clipboard changes |
| `wl-clipboard` | `wl-paste` / `wl-copy` | Read/write the WSLg (Wayland) clipboard |

### Dependencies installation

**Debian / Ubuntu / WSL**
```bash
sudo apt install imagemagick coreutils wl-clipboard
```

**Fedora**
```bash
sudo dnf install ImageMagick coreutils wl-clipboard
```

**Arch**
```bash
sudo pacman -S imagemagick coreutils wl-clipboard
```

## Script installation

Drop the script somewhere on your `PATH` and make it executable:

```bash
install -m 755 clip-bmp2pngd ~/bin/clip-bmp2pngd
```

## Usage

```bash
clip-bmp2pngd            # start in daemon mode (default)
clip-bmp2pngd start      # same as above
clip-bmp2pngd status     # report whether the daemon is running
clip-bmp2pngd stop       # stop the daemon
```

### Start it automatically

To have it running whenever your WSL shell starts, add this to your
`~/.bashrc` (or `~/.zshrc`):

```bash
clip-bmp2pngd start >/dev/null 2>&1
```

The `start` command is idempotent — if the daemon is already running it just
reports the existing PID and exits, so it's safe to call on every shell launch.

## Requirements / notes

- Designed for **WSLg** (WSL 2 with the built-in Wayland clipboard). It relies
  on `wl-paste` / `wl-copy` talking to the WSLg compositor.
- Polling interval is 1 second, so there is a brief lag between taking a
  screenshot and the PNG becoming available — usually unnoticeable in practice.

## License

Do whatever you like with it. Provided as-is, without warranty.
