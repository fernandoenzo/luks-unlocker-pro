# AGENTS.md

## Project Overview

LUKS Unlocker Pro is a POSIX shell library for unlocking and mounting LUKS-encrypted
devices during early boot (initramfs) on Debian-based Linux systems. It consists of
two shell scripts at the repository root -- no subdirectories, no build system, no
package manager.

### File Layout

```
luks-unlocker-pro      # Main library of shell functions (sourced by boot scripts)
custom_luks_decrypt    # Example initramfs boot script (uses the library)
README.md              # Project documentation
LICENSE                # GPLv3+
```

Installation target paths:
- `luks-unlocker-pro` -> `/etc/initramfs-tools/scripts/luks-unlocker-pro`
- `custom_luks_decrypt` -> `/etc/initramfs-tools/scripts/local-top/custom_luks_decrypt`

After changes, the initramfs must be rebuilt: `update-initramfs -u`

## Build / Lint / Test Commands

There is no build system, test suite, or CI pipeline in this project.

### Linting (manual)

Use [ShellCheck](https://www.shellcheck.net/) to lint the scripts:

```sh
# Lint all scripts
shellcheck -s sh luks-unlocker-pro custom_luks_decrypt

# Lint a single file
shellcheck -s sh luks-unlocker-pro
```

Always pass `-s sh` to enforce POSIX sh mode (matching the `#!/bin/sh` shebang).

### Syntax check

```sh
sh -n luks-unlocker-pro
sh -n custom_luks_decrypt
```

### Testing

There is no automated test suite. Testing is done manually on a Debian system with
LUKS-encrypted devices in an initramfs environment. The scripts depend on system
utilities (`cryptsetup`, `mount`, `shred`, `plymouth`, `modprobe`, `/lib/cryptsetup/askpass`)
and initramfs-tools built-in functions (`/scripts/functions`).

## Code Style Guidelines

### Language and Compatibility

- **POSIX sh only** (`#!/bin/sh`). Do NOT use bash-specific features such as arrays,
  `[[ ]]`, `(( ))`, `declare`, `typeset`, process substitution (`<()`), or `here-strings`.
- Use `$()` for command substitution. Never use backticks.
- Use `[ ]` (single bracket) for test expressions, not `[[ ]]`.
- Use `printf` for all output operations.

### Indentation and Formatting

- **2-space indentation**. No tabs.
- No trailing whitespace.
- One blank line between function definitions.
- Keep lines to a reasonable length; break long commands with `\` if needed, but
  prefer concise one-liners when the logic is clear.

### Naming Conventions

- **Functions**: `snake_case` -- e.g., `unlock_device`, `mount_device`,
  `success_or_shell`.
- **Variables**: `snake_case` -- e.g., `mapper_name`, `mount_point`, `passwords_file`.
- **Constants/globals**: `snake_case` at file scope -- e.g., `passwords_file`.
  There are no `UPPER_CASE` constants in this codebase aside from the initramfs
  convention `PREREQ` in boot scripts.
- **Function arguments**: Assign positional parameters to named local variables
  immediately at the top of the function:
  ```sh
  my_function() {
    local device=$1
    local mapper_name=$2
    local max_attempts=${3:-3}
    # ...
  }
  ```

### Variable Declarations

- **All function-scoped variables MUST be declared `local`**. This is critical to
  avoid polluting the global namespace when the library is sourced.
- Use parameter expansion for default values: `${var:-default}`.
- Use parameter expansion for conditional arguments: `${var:+--flag $var}`.

### Function Documentation

Every function must have a comment block directly above it with this format:

```sh
# Function: function_name(arg1, arg2, optional_arg)
# Description: One-line or multi-line description of what the function does.
# Arguments:
#   arg1: Description of the first argument
#   arg2: Description of the second argument
#   optional_arg (optional, default: value): Description
# Return: Description of return codes (if non-obvious)
```

Include `# Usage Examples:` with concrete invocations when the function's interface
is not immediately obvious (see `mount_device` and `umount_and_close_device`).

### Comments

- Use `#` followed by a single space: `# This is a comment`.
- Inline comments are acceptable for short clarifications.
- Write comments that explain **why**, not just **what**, especially for non-obvious
  shell logic and chained conditionals.

### Error Handling

- Use `&&` / `||` chaining for simple control flow:
  ```sh
  command && print_text "success" || { print_text "failure" && return 1; }
  ```
- For multi-step error handling, prefer `if`/`then`/`else` to avoid precedence
  ambiguity:
  ```sh
  if command; then
    print_text "success"
  else
    print_text "failure"
    return 1
  fi
  ```
- Return `0` for success, `1` (or non-zero) for failure.
- Use `return` in functions, `exit` only in top-level scripts or for fatal errors.
- Wrap critical operations with `success_or_shell` in boot scripts to provide
  interactive recovery on failure.
- Always clean up sensitive data (passwords) on error paths using `shred_passwords`.

### Quoting

- Always quote variable expansions: `"$variable"`, `"$1"`.
- Quote command substitutions: `"$(command)"`.
- Exception: unquoted expansions are acceptable inside arithmetic contexts or when
  word splitting is intentionally desired (document these cases).

### Control Structures

- `case` statements follow this format:
  ```sh
  case "$variable" in
    value1) commands ;;
    value2) commands ;;
    *) fallback ;;
  esac
  ```
- `if`/`then`/`else` statements are preferred for multi-step logic or when
  `&&`/`||` chaining would create ambiguous precedence. Simple one-line
  conditionals may still use `&&`/`||` chaining.
- Use `for i in $(seq 1 "$n")` for numeric loops (POSIX-compatible).

### Imports / Sourcing

- Source external scripts with `. /path/to/script` (dot notation, not `source`).
- The main library sources `/scripts/functions` (initramfs-tools built-ins).
- Boot scripts source the library: `. /scripts/luks-unlocker-pro`.

### Output and User Interaction

- Use `print_text` for all user-facing messages. This function handles Plymouth
  (graphical boot) and falls back to `printf`.
- Use `exec_silently` to suppress stdout/stderr from commands that should run quietly.
- Use `/lib/cryptsetup/askpass` for password prompts (Debian initramfs convention).

## Runtime Dependencies

These are not managed by a package manager but are required at runtime in the
initramfs environment:

| Utility | Purpose |
|---|---|
| `cryptsetup` | LUKS encryption operations |
| `shred` | Secure password file deletion |
| `plymouth` | Optional graphical boot messages |
| `modprobe` | Loading kernel modules (e.g., `loop`) |
| `mount` / `umount` | Filesystem mount operations |
| `mktemp` | Temporary file creation |
| `grep` | Text pattern matching |
| `seq` | Numeric sequence generation |
| `/lib/cryptsetup/askpass` | Debian-specific password prompt |
| `/scripts/functions` | initramfs-tools built-in functions |

## Git Conventions

- Remote: `git@github.com:fernandoenzo/luks-unlocker-pro.git`
- Primary branch: `master`
- License: GPLv3+
