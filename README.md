# `op run` for Windows Subsystem for Linux

The 1Password CLI's `op run` utility re-implemented as a bash script using `op inject` to add Windows Subsystem for Linux support.

## Why does this exist?

I use Windows Subsystem for Linux (WSL) for development and rely on 1Password to manage secrets for production, staging, and local workflows. My organization requires SSO authentication for 1Password, which means I must use the Windows 1Password CLI from within WSL to access my work credentials.

**While most 1Password CLI features work in WSL through Windows interoperability, `op run` fails because it always executes commands in a Windows context, even when called from within WSL.**

With that in mind, I created `oprun` to emulate the ergonomics and utility of `op run` by using `op inject` under the hood.

## Prerequisites

Before using `oprun`, you'll need:

> [!IMPORTANT]
> Loading variables from a 1Password Environment (the `--environment` flag) requires the [latest beta build of the 1Password CLI](https://app-updates.agilebits.com/product_history/CLI2#beta), version `2.33.0-beta.02` or later. All other features work with the stable release.

1. **1Password CLI** - Install the 1Password CLI on your system:
   - **Windows/WSL**: Download from [1Password CLI releases](https://developer.1password.com/docs/cli/get-started/)
   - **Linux**: if you can use password based sign-in or service accounts `curl -sSfL https://downloads.1password.com/linux/tar/stable/x86_64/1password-cli-amd64-latest.tar.gz | tar -xzf - -C /usr/local/bin`
2. **Authentication** - Sign in to your 1Password account
3. **WSL Compatibility** - `oprun` automatically detects your environment:
   - Uses `op` if available in your Linux environment
   - Falls back to `op.exe` (Windows CLI) when running in WSL

## Installation

### Option 1: Direct Download (Recommended)

```bash
# Download and install oprun to /usr/local/bin
curl -sSL https://raw.githubusercontent.com/JanUnld/oprun.sh/main/oprun -o /usr/local/bin/oprun
chmod +x /usr/local/bin/oprun
```

### Option 2: Manual Installation

1. Download the `oprun` script from this repository
2. Place it in a directory that's in your `$PATH` (e.g., `/usr/local/bin` or `~/bin`)
3. Make it executable:

   ```bash
   chmod +x /path/to/oprun
   ```

### Verify Installation

```bash
oprun --help
```

## Getting Started

Once installed, you can use `oprun` with any WSL setup:

```bash
oprun -- npm run start

oprun -e ./env.template -- npm run start
oprun -e ./env.template -- docker compose up

oprun --help
```

## Options

`oprun` supports the following command-line options:

### `-e, --env-file FILE`

Specify a custom env file containing your 1Password secret references.

**Default:** `.env.template`

```bash
# Use a custom template file
oprun -e secrets.env -- npm run start
oprun --env-file production.env -- docker compose up
```

#### Example tempalte file

The env files should [follow the template syntax defined in 1Password's documentation for the `op inject` command.](https://developer.1password.com/docs/cli/reference/commands/inject).

Heres a quick reference:

```bash
DB_PASSWORD="op://mytodoapp-prod/database/password"
GENERIC_CRED="op://1password-vault/1password-credential/field-in-credential"
```

### `-E, --environment ID`

Load variables from a [1Password Environment](https://developer.1password.com/docs/environments/) (beta) using its environment ID.

This mirrors the native `op run --environment` flag and requires a [beta build of the 1Password CLI](https://app-updates.agilebits.com/product_history/CLI2#beta) (`2.33.0-beta.02` or later). To find an Environment's ID, open the 1Password app and navigate to **Developer** > **View Environments** > **View environment** > **Manage environment** > **Copy environment ID**.

```bash
# Load only from a 1Password Environment
oprun --environment blgexucrwfr2dtsxe2q4uu7dp4 -- printenv

# Combine an Environment with an env file (Environment variables take precedence)
oprun --environment blgexucrwfr2dtsxe2q4uu7dp4 -e extra.env -- ./my-script.sh
```

When the same variable is defined in both a 1Password Environment and an env file, the Environment value takes precedence, matching the native 1Password CLI behavior.

### `--mask` and `--no-masking`

By default, `oprun` replaces secret values resolved from 1Password with `******` in the command's output **when that output is redirected** (piped or written to a file) — which is when secrets are most likely to end up in a log.

When the command is attached to an interactive terminal, `oprun` runs it without masking so full-screen/TUI programs (for example, `copilot`, `htop`, or `vim`) work normally. Masking requires piping the command's output through a filter, which makes stdout and stderr a non-terminal and stops TUIs from rendering.

```bash
# Redirected output is masked
oprun -- npm run start > app.log

# Interactive commands run normally on a terminal
oprun copilot

# Force masking even on an interactive terminal (breaks TUI programs)
oprun --mask -- printenv

# Disable masking entirely
oprun --no-masking -- env
```

Use `--no-masking` (or the `OP_RUN_NO_MASKING=true` environment variable) to disable masking altogether, and `--mask` to force masking even when attached to a terminal.

Secrets are identified from env file entries whose value is a 1Password secret reference (`op://...`) and from every variable loaded from a 1Password Environment. Values that are not secret references (for example, `NODE_ENV=development`) are left untouched.

### `-v, --verbose`

Enable verbose mode to display loaded environment variables for debugging purposes.

**Security Note:** Variables are masked (shown as `***`) to avoid exposing secrets in logs.

```bash
# Debug what variables are being loaded
oprun -v npm run test
oprun --verbose -t staging.env -- python manage.py runserver
```

### `-h, --help`

Display help information and usage examples.

```bash
oprun --help
oprun -h
```

### `--`

Use the double dash separator to explicitly separate `oprun` options from your command. This is especially useful when your command has options that might conflict with `oprun` options.

```bash
# Recommended when your command has flags
oprun -- docker-compose up --build
oprun -e secrets.env -- npm run start --port 3000

# Not needed for simple commands
oprun npm run start
```

### Option Combinations

You can combine multiple options:

```bash
# Verbose mode with custom template
oprun -v -e production.env -- npm run start

# All options with explicit separator
oprun --verbose --env-file staging.env -- docker compose up --build
```

## Considerations

While this emulates the spirit of the 1Password CLI's `op run` command it doesn't match one-to-one.

### Masking is best-effort

`oprun` masks secrets by replacing literal occurrences of each resolved secret value in the command's output. This differs from the native CLI in a few ways:

- Only literal values are masked. If a command transforms a secret before printing it (for example, base64- or URL-encoding), the transformed value will not be caught.
- Secrets are matched per line, so a value split across multiple writes or lines may not be fully masked.
- Masking is skipped when the command is attached to an interactive terminal (unless `--mask` is passed), because piping the output would break full-screen/TUI programs.
- Because the CLI does not expose per-variable masking metadata, every value loaded from a 1Password Environment is treated as a secret, while env file values are only masked when they are `op://` secret references.

Use `--no-masking` if masking interferes with a command's output.
