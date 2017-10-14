Bash Initalization Behaviour
============================

A Bash process may be:

| State               | Invocation
|---------------------|---------------------------------------------------
| Login (**L**)       | `argv[0][0] == '-'`, `--login`
| Interactive (**I**) | `-i`, stdin is tty
| Remote (**R**)      | stdin is "network connection" (see below)

* '•' = set; blank = not set; '-' = not checked or don't care.
* **p**: sources `/etc/profile`
  then first of `~/.bash_profile`, `~/.bash_login`, `~/.profile`
* **rc**: sources `/etc/bash.bashrc` then `~/.bashrc`

| L | I | R | | p | rc | | Example
|:-:|:-:|:-:|-|:-:|:--:|-|-----------------
| • | • | - | | • |    | | `ssh host.com` (sshd sets argv[0]="-bash")
| • |   | - | | • |    | | `ssh host.com </dev/null` (sshd sets argv[0]="-bash")
|   | • |   | |   | •  | | `bash -i`, `bash` on tty
|   |   |   | |   |    | | `bash hello.sh`, `bash -c echo foo`,
|   |   | • | |   | •  | | `ssh host.com 'echo $-'` (ssh runs `bash -c 'echo $-`)

Note that whether or not the shell is a "login shell" (by the [Bash
manpage definition][Invocation] is not the same as whether the shell
is a "fresh login" in the sense that the profile has ever been run.
The shell process from `ssh host command` is not considered a "login
shell" but has never had any of the profile files run.

Calling `ssh` with no command starts a login shell on the remote,
regardless of whether stdin is a tty or not. Calling ssh with a
command runs `bash -c ...` which is non-login and non-interactive, but
Bash detects it's "connected to a network connection" (the exact
details of how it decides this are not clear) and sources `.bashrc`
anyway.


`.bashrc` Notes
---------------

`.bashrc` is not sourced by an interactive login shell. However, the
default `/etc/profile` and `~/.profile` installed by most distros
will, if run under Bash, explicitly source their respective
`/etc/bash.bashrc` and `.bashrc` files.

`.bashrc` should never print anything to stdout as this will confuse
commands like scp etc. (And it should probably never print to stderr,
either.)


Testing State
-------------

Test for interactive by checking if `$-` includes `i` (this does not
work in old Bourne shell; unknown if it works in POSIX shell):

    case $- in
        *i*)    echo interactive;;
        *)      echo or not;;
    esac

You can also check with `if [ "$PS1" ] ...`, but this may not be as
reliable depending on what the user has previously run in his setup.


XXX
---

https://unix.stackexchange.com/a/46856/10489
https://www.gnu.org/software/gettext/manual/html_node/The-LANGUAGE-variable.html



Categories:
* Env has already been set up by me. If not ("fresh login"):
  * check $TERM with tset
  * umask
  * ssh-agent if not provided by login mechanism


* interactive or not: determines what aliases etc. are available?
  * set -o vi
* tty or not: determines TERM handling, command line type


cjs Initialization
------------------

When a shell starts, it may inherit state that has been previously set
up by the user or may be a "fresh login," having the default state set
up by the sysadmin. (This is not the same as whether Bash considers
itself to be a "login shell" according to [Initalization]; see above.)

I divide my initialization into several parts:

1. Defaults which should override the system configuration
   (`/etc/profile`) on a fresh login but not override subsequent user
   configuration (even just typed at the command line).  
   • Done in: `~/.bash_profile`  
   • Examples: `umask 0720`  

2. Interactive configuration which should always be set in interactive
   shells but should not be set when running shell scripts that might
   have their own commands.  
   • Done in: `.bashrc` only when the shell is interactive  
   • Examples: My `dum` shell function to run `du -ms "$@" | sort -n`  

3. Configuration that should always be forced in new shells or when
   re-initializing an existing shell.
   • Done in: `.bashrc`
   • Examples: Ensuring that `~/.local/bin/` is in the path

In all cases it appears that only `.bash_profile` or `.bashrc` is
sourced, never both. (XXX verify this.) Thus, each needs to source
the other. We make both idempotent and follow these rules:

* `.bash_profile`:
  * immediately `return`s (not `exit`!) if `CJS_PROFILE` env var is set
  * exports `CJS_PROFILE=true`
  * does setup to override system defaults (only)
  * does handy initialization (TERM confirmation, fortune) if interactive
  * sources `.bashrc`
* `.bashrc`:
  * sources `.bash_profile` (sets up or returns because set up)
  * constant config (3 above) that always overrides everything
  * interactive-only config if interactive

XXX how to deal with
* bashrc calls profile
  * profile calls bashrc
    * bashrc calls profile, even with immediate exit...

Ah, so we do continuation passing and if `.bashrc` has to call profile
it execs that, letting profile call back to bashrc to finish init?
But then `.bashrc` also has to know about `CJS_PROFILE` var.

XXX Notes I'm not sure what to do with
--------------------------------------

It's not clear why, but the clear usage intent is to have users source
their `.bashrc` from their `.profile`. (Otherwise your login
interactive shell has a different enviroment from subsequent
interactive shells.)

XXX POSIX `$ENV` behaviour

### `.profile`

If `.profile` is run, you're logging into a host and have no previous
environment set up. (However, you may or may not be interactive.) So
set up things that you'd do if you have no previous processes running
such as ssh-agent (if not already provided by sshd), `tset` (because
you don't yet know if this system has your terminfo entry), etc.

probably should check for interactive and not do interactive stuff in
non-interactive shells (created by 'ssh host <file')

`.profile` is run when logging into a host; it may be interactive or
not depending on what the stdin is for ssh or whatever.

### `.bashrc`

ulimit is something you want to "set on login," but must be done in
`.bashrc` because 'ssh somehost acommand' will, despite being a fresh
login to the host, not run .profile



[Invocation]: http://man7.org/linux/man-pages/man1/bash.1.html#INVOCATION
[debhelper]: https://manpages.debian.org/stretch/debhelper/dh.1.en.html
