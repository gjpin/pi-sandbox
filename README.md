# Pi Sandbox

Forked from [claude-code-sandbox](https://github.com/neko-kai/claude-code-sandbox)

A macOS sandbox configuration for Pi Coding Agent that restricts filesystem READ access for enhanced security.

Just about all sandbox-exec attempts for coding agents you'll find on GitHub allow full filesystem read access and full network access – all they protect against is file overwrites. But that's not very secure - prompt injection could leak data from all over your filesystem if the agent can read it.

This project provides a macOS `sandbox-exec` profile that limits `pi`'s access to your filesystem. It prevents Pi from reading your home directory (except for the current working directory) and restricts writes to only the target directory and temporary locations.

## Features

- **Restricted Read Access**: Blocks reading from file system except for:
  - Current working directory (`TARGET_DIR`)
  - Pi configuration (`~/.pi`)
  - Git configuration files (`.gitconfig`, `.config/git`, `.config/jj`)
  - Nix configuration (`~/.config/nix`, `~/.local/share/nix`, `~/.nix-profile`, `~/.local/state/nix`)
  - Java/Scala tooling (`~/.sbt`, `~/.ivy2`, `~/.m2`, Coursier, Scala CLI, JGit, `~/Library/Java`, `/Library/Java`)
  - System directories (`/usr`, `/bin`, `/opt`, `/var`, `/nix`, `/etc`, `/System`, `/Library/Java`)
  - It allows _listing_ directories leading up to `TARGET_DIR`, because otherwise pi will glitch and set PATH to "" for the agent.
    However, even though files in `~` can be listed with `ls` by pi, they (and their metadata) cannot be read

- **Restricted Write Access**: Only allows writing to:
  - Current working directory (`TARGET_DIR`)
  - Temporary directories (`/tmp`, `/var/folders`)
  - Cache directory (`~/.cache`)
  - Pi configuration (`~/.pi`)

- **Network Access**: Full network access enabled (required for LLM API)
- **Keychain Access**: Allows reading from macOS Keychain for API key storage (used by `/login`)

## Installation

### Manual Installation (Without Nix)

Run the installation script:

```bash
./install
```

This will:
1. Concatenate the default sandbox profile `noread.sb` and `pi-sandbox` script together to make the script self-contained.
2. Install the `pi-sandbox` script to `~/.local/bin/pi-sandbox`
3. Make the script executable

Ensure that `~/.local/bin` is in your PATH.

### Using Nix (Recommended)

If you have Nix installed, you can run `pi-sandbox` directly without installing:

```bash
nix run github:gjpin/pi-sandbox -- pi
```

Or install it to your profile:

```bash
nix profile install github:gjpin/pi-sandbox
```

You can also add it to a flake-based NixOS or home-manager configuration:

```nix
{
  inputs.pi-sandbox.url = "github:gjpin/pi-sandbox";
  inputs.pi-sandbox.inputs.nixpkgs.follows = "nixpkgs";
  inputs.pi-sandbox.inputs.flake-utils.follows = "flake-utils";

  # Then use inputs.pi-sandbox.packages.${system}.default
}
```

## Usage

Instead of running `pi` directly, use the `pi-sandbox` wrapper:

```bash
# run pi (default)
pi-sandbox

# run pi with explicit program
pi-sandbox pi

# run bash to browse the sandbox-accessible filesystem
pi-sandbox bash

# write generated sandbox-exec profile to a file and exit
pi-sandbox --write-profile curprofile.sb

# dump the built-in noread.sb to a file for customization
pi-sandbox --write-base-profile custom.sb

# use a custom profile template instead of the built-in noread.sb
pi-sandbox --use-profile custom.sb -- pi
```

## How to add access to more directories

Modify `noread.sb` and run `./install` again, or use `--use-profile` with a custom profile template.

Find out which rules to add by running `Console.app` and filtering errors by 'sandbox'

## How It Works

The `pi-sandbox` wrapper uses macOS's `sandbox-exec` command to apply a security profile defined in `noread.sb`. This sandbox profile is based on the [Para sandboxing profile](https://github.com/2mawi2/para/blob/218259b6e260be43334f308a74108f31920f7ca4/src/core/sandbox/profiles/standard.sb) and [anthropic's sandbox-runtime's dynamic profile](https://github.com/anthropic-experimental/sandbox-runtime/blob/1bafa66a2c3ebc52569fc0c1a868e85e778f66a0/src/sandbox/macos-sandbox-utils.ts#L200), with additional configuration for Pi Coding Agent compatibility.
Specifically, for some reason pi needs list access to all parent directories of current working directory - it doesn't need read access to the content of directories, only access to list the content of directories. Without this access it will set PATH in the agent to "" and disable colored output. To avoid this the script adds these listing rules dynamically.
