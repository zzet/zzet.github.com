---
layout: post
title: "Unix shell initialization"
date: 2014-01-27

uglyURLs: true
aliases:
- /shell/learning/os/unix/2014/01/27/unix-shell-initialization
- /shell/learning/os/unix/2014/01/27/unix-shell-initialization.html
---

Shell initialization files are the way to maintain a unified configuration for parameters such as:

- `$PATH` and other environment variables
- shell prompt
- shell tab-completion
- aliases, functions
- key bindings

## Shell Modes

Which files are loaded and executed upon shell startup depends on which mode is used, or simply put, how the shell was launched. There are two main variants:

- **login** - i.e., when a user has logged into the system without initializing a graphical shell, or by using the SSH protocol;
- **interactive** - a shell that has a prompt and whose standard input and output are connected to a terminal.

These modes can be manually activated using the following flags:

- `l`, `-login`
- `i`

As a result, the following common operations will be executed as follows:

- log in to a remote system via SSH:
**login + interactive**
- execute a script remotely, e.g. `ssh user@host 'echo $PWD'` or with
[Capistrano](https://www.google.com/search?q=%5Bhttps://github.com/capistrano/capistrano/wiki%5D(https://github.com/capistrano/capistrano/wiki)): **non‑login, non‑interactive**
- execute a script remotely and request a terminal, e.g. `ssh user@host -t 'echo $PWD'`: **non-login, interactive**
- start a new shell process, e.g. `bash`:
**non‑login, interactive**
- run a script, `bash myscript.sh`:
**non‑login, non‑interactive**
- run an executable with `#!/usr/bin/env bash` shebang:
**non‑login, non‑interactive**
- open a new graphical terminal window/tab:
    - on Mac OS X: **login, interactive**
    - on Linux: **non‑login, interactive**

## Shell init files

In order of activation:

### bash

1. **login** mode:
    1. `/etc/profile`
    2. `~/.bash_profile`, `~/.bash_login`, `~/.profile` (only first one
    that exists)
2. interactive **non-login**:
    1. `/etc/bash.bashrc` (some Linux; not on Mac OS X)
    2. `~/.bashrc`
3. **non-interactive**:
    1. source file in `$BASH_ENV`

### Zsh

1. `/etc/zshenv`
2. `~/.zshenv`
3. **login** mode:
    1. `/etc/zprofile`
    2. `~/.zprofile`
4. **interactive**:
    1. `/etc/zshrc`
    2. `~/.zshrc`
5. **login** mode:
    1. `/etc/zlogin`
    2. `~/.zlogin`

### [dash](https://www.google.com/search?q=%5Bhttp://gondor.apana.org.au/~herbert/dash/%5D(http://gondor.apana.org.au/~herbert/dash/))

1. **login** mode:
    1. `/etc/profile`
    2. `~/.profile`
2. **interactive**:
    1. source file in `$ENV`

### [fish](https://www.google.com/search?q=%5Bhttp://ridiculousfish.com/shell/user_doc/html/index.html%23initialization%5D(http://ridiculousfish.com/shell/user_doc/html/index.html%23initialization))

1. `<install-prefix>/config.fish`
2. `/etc/fish/config.fish`
3. `~/.config/fish/config.fish`

### Practical guide to which files get sourced when

- Opening a new Terminal window/tab:
    - **bash**
        - OS X: `.bash_profile` or `.profile` (1st found)
        - Linux: `.profile` (Ubuntu, once per desktop login session) +
        `.bashrc`
    - **Zsh**
        - OS X: `.zshenv` + `.zprofile` + `.zshrc`
        - Linux: `.profile` (Ubuntu, once per desktop login session) +
        `.zshenv` + `.zshrc`
- Logging into a system via SSH:
    - **bash**: `.bash_profile` or `.profile` (1st found)
    - **Zsh**: `.zshenv` + `.zprofile` + `.zshrc`
- Executing a command remotely with `ssh` or Capistrano:
    - **bash**: `.bashrc`
    - **Zsh**: `.zshenv`
- Remote git hook triggered by push over SSH:
    - *no init files* get sourced, since hooks are running [within a
    restricted shell](http://git-scm.com/docs/git-shell)
    - PATH will be roughly:
    `/usr/libexec/git-core:/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin`

## Misc. things that affect `$PATH`

- OS X:
    - `/etc/paths`, `/etc/paths.d/*`
    - [`~/.MacOSX/environment.plist`](https://www.google.com/search?q=%5Bhttp://developer.apple.com/library/mac/%23documentation/MacOSX/Conceptual/BPRuntimeConfig/Articles/EnvironmentVars.html%23//apple_ref/doc/uid/20002093-113982%5D(http://developer.apple.com/library/mac/%23documentation/MacOSX/Conceptual/BPRuntimeConfig/Articles/EnvironmentVars.html%23//apple_ref/doc/uid/20002093-113982)) - affects **all** graphical
    programs
    - `/etc/launchd.conf`
    - TextMate: Preferences -> Advanced -> Shell Variables
- Linux:
    - `/etc/environment`

## Final notes

This guide was tested with:

- bash 4.2.37, 4.2.39
- Zsh 4.3.11, 5.0

On these operating systems/apps:

- Mac OS X 10.8 (Mountain Lion): Terminal.app, iTerm2
- Ubuntu 12.10: Terminal

See also:

- [Environment
Variables](https://help.ubuntu.com/community/EnvironmentVariables)
- path_helper(8)
- launchd.conf(5)
- pam_env(8)


  [Capistrano]: https://github.com/capistrano/capistrano/wiki
  [dash]: http://gondor.apana.org.au/~herbert/dash/
  [fish]:
http://ridiculousfish.com/shell/user_doc/html/index.html#initialization
  [plist]:
http://developer.apple.com/library/mac/#documentation/MacOSX/Conceptual/BPRuntimeConfig/Articles/EnvironmentVars.html#//apple_ref/doc/uid/20002093-113982
