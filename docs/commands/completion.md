# squirrel completion

Generate shell completion scripts for squirrel.

```
squirrel completion <shell>
```

---

## Supported Shells

| Shell | Argument |
|---|---|
| Bash | `bash` |
| Zsh | `zsh` |
| Fish | `fish` |
| PowerShell | `powershell` |

---

## Installation

=== "Bash"
    **System-wide:**
    ```bash
    squirrel completion bash | sudo tee /etc/bash_completion.d/squirrel
    ```

    **User-level:**
    ```bash
    mkdir -p ~/.local/share/bash-completion/completions
    squirrel completion bash > ~/.local/share/bash-completion/completions/squirrel
    ```

    Reload your shell or run:
    ```bash
    source /etc/bash_completion.d/squirrel
    ```

=== "Zsh"
    ```zsh
    squirrel completion zsh > "${fpath[1]}/_squirrel"
    ```

    Or add to `~/.zshrc`:
    ```zsh
    source <(squirrel completion zsh)
    ```

=== "Fish"
    ```fish
    squirrel completion fish > ~/.config/fish/completions/squirrel.fish
    ```

=== "PowerShell"
    Add to your `$PROFILE`:
    ```powershell
    squirrel completion powershell | Out-String | Invoke-Expression
    ```

    Or load on-demand:
    ```powershell
    squirrel completion powershell | Out-String | Invoke-Expression
    ```

---

## What Completions Provide

- Subcommand names (`backup`, `restore`, `snapshots`, `forget`, …)
- Flag names and their types
- Snapshot ID completion (where applicable)
- Shell-specific completion (tab, double-tab, etc.)
