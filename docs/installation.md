# Installation

## Binary Releases

Pre-built binaries for Linux, macOS, and Windows (amd64 / arm64) are available on the [GitHub Releases](https://github.com/elpol4k0/squirrel/releases) page.

Download the archive for your platform, extract the binary, and place it somewhere on your `PATH`.

---

## Homebrew (macOS / Linux)

```bash
brew install elpol4k0/tap/squirrel
```

---

## Go

Requires Go 1.21 or later.

```bash
go install github.com/elpol4k0/squirrel/cmd/squirrel@latest
```

---

## Docker

```bash
docker pull ghcr.io/elpol4k0/squirrel:latest
```

Run a backup inside Docker:

```bash
docker run --rm \
  -e SQUIRREL_PASSWORD=secret \
  -v /data:/data:ro \
  -v /backup:/backup \
  ghcr.io/elpol4k0/squirrel:latest \
  backup --path /data --repo /backup/repo
```

---

## Windows — Build from Source

### Without WinFsp (all commands except `mount`)

```powershell
$env:CGO_ENABLED = "0"
go build -o squirrel.exe .\cmd\squirrel\
```

!!! note
    `CGO_ENABLED=0` disables the `mount` command, which requires the WinFsp SDK.
    All other commands work fully without it.

### With WinFsp (enables `mount`)

1. Install [WinFsp](https://winfsp.dev/) from the official website.
2. Build without disabling CGO:

```powershell
go build -o squirrel.exe .\cmd\squirrel\
```

---

## Shell Completions

After installation, enable shell completions for a better CLI experience:

=== "Bash"
    ```bash
    squirrel completion bash > /etc/bash_completion.d/squirrel
    ```

=== "Zsh"
    ```zsh
    squirrel completion zsh > "${fpath[1]}/_squirrel"
    ```

=== "Fish"
    ```fish
    squirrel completion fish > ~/.config/fish/completions/squirrel.fish
    ```

=== "PowerShell"
    ```powershell
    squirrel completion powershell | Out-String | Invoke-Expression
    ```

See [`squirrel completion`](commands/completion.md) for details.

---

## Verifying the Installation

```bash
squirrel --version
```

Expected output: `squirrel v1.x.x`
