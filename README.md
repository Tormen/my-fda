# my-fda

Full Disk Access for root jobs on macOS, kept stable across Homebrew upgrades.

## Why

macOS ties a Full Disk Access grant to a binary's path and contents. A
Homebrew binary changes both on every `brew upgrade`, so its grant silently
stops applying and the root job using it fails with "Operation not
permitted". And the list in System Settings grows with every version ever
granted.

The same holds, worse, for what such a job tends to be granted instead: a
shell or a general tool (curl, python3, perl, env, ...) gives Full Disk Access
to whatever runs it; a grant on a script does nothing, as macOS checks its
interpreter; and anything in Homebrew changes on upgrade and runs what its
packages bring. `list` flags each of these (`SHELLS`, `GENERAL_TOOLS`,
`BREW_TREES`), `denied` and `add` say the same, and the fix is always a
launcher for the job.

Such a row switched off (`denied[0]`) grants nothing: macOS writes it when it
refuses the program, and writes it back at the next refusal -- taking it away
by hand only hides it until then. So `list` marks it for its job, not for
`remove`, and `denied --catch` finds that job.

## How

A *launcher* holds the grant instead: a tiny compiled program, built once
and never rebuilt, that runs one fixed target -- a script, or a Homebrew
binary by its stable path such as `/opt/homebrew/bin/rsync`. The
LaunchDaemon starts the launcher; macOS counts the program launchd started
as responsible for everything it runs, so the target gets the launcher's
grant.

- A launcher is `/Library/PrivilegedHelperTools/my-fda-<NAME>`, root:wheel
  755, and reads as `my-fda-<NAME>` in the Full Disk Access list. No dot in
  the name: macOS takes what follows a dot for an extension and files the
  launcher as a Document, which the pane refuses -- dragged in or chosen with
  +. A launcher from before, `my-fda.<NAME>`, still runs its job; `status`
  marks it, and `add` refuses it with the fix (destroy, create). Executable
  by everyone, because System Settings, which runs as the screen's user,
  loses a dragged-in program that user cannot execute, without a word; yet
  run by root only, because the grant is that path's: the launcher refuses
  any other uid (ERROR, exit 77) before it starts the target, and is signed
  with the hardened runtime, so no `DYLD_` variable puts code into it before
  that check. A launcher from before is 744: `+` adds it, a drag does not.
  `add` makes an older 700 launcher readable first, and refuses any other
  file the screen's user cannot read, saying why.
- `create` builds it on the machine with its own `cc` (Command Line Tools);
  the C source lives inside `my-fda`, never a binary in git. It then signs it
  ad hoc as `my-fda-<NAME>` (`codesign -s -`): `cc` alone only linker-signs,
  and the pane loses such a file when it is dragged in.
- It spawns the target and waits, so the process holding the grant lives for
  the whole run; it passes on all arguments and SIGTERM/SIGINT/SIGHUP, and
  exits with the target's status.
- The target is compiled in (`MY-FDA-TARGET=<path>`), never taken from the
  command line; `status` reads it back out of the binary.
- `create` never rebuilds an existing launcher: a rebuild loses its grant.
- `create --user <USER>`: for a job that runs as a user. launchd starts the
  launcher as root (its LaunchDaemon names no `UserName`), and the launcher
  becomes USER -- groups, group, user, HOME/USER/LOGNAME/SHELL -- before it
  starts the target; when it cannot, the target does not run at all. The
  user is compiled in too (`MY-FDA-USER=<user>`), and `test` refuses a
  LaunchDaemon that would start such a launcher as anyone but root.

- `add` puts the program into the Full Disk Access list switched OFF -- a
  row in TCC.db with auth 0 and the program's code requirement, TCC.db
  backed up first -- and opens the pane: one click there grants it, so the
  grant stays a person's act. Where TCC.db cannot be written, it puts the
  path on the clipboard for +, Cmd-Shift-G, Cmd-V, or offers a drag into the
  list -- but only for a program the screen's user can execute: the pane
  loses any other one dragged in, without an error (macOS 27).

## Commands

    my-fda list [--service <SERVICE>|all]
                                      every grant: an ID, the client, allowed/denied, what to do;
                                      --service: another TCC service's grants (AppBundles: App Management), from the system
                                      database and each user's own, with columns for the service and whose database
    my-fda denied [<PROGRAM>] [<LAST> | --since <TIME> [--until <TIME>]] [--service <SERVICE>|all]
                                      every file check macOS refused, from the system log (default: the last 1d)
    my-fda denied --catch [<PROGRAM>] [--service <SERVICE>|all]
                                      waits for the next refusal: the refused process, its parents, its open files
    my-fda add <PATH>                 "+": lists PATH switched OFF, opens the pane -- you switch it on -- checks it
    my-fda remove <ID>...             "-": shows what it would take away
    my-fda remove go <ID>...          ...and takes it away
    my-fda remove --recommended [go]  every grant that list marks '-> remove'
    my-fda cleanup [go]               the GONE grants: shows them; with go, takes them away
    my-fda reset [go]                 all grants at once, and what to give back
    my-fda create [<NAME>] <TARGET>   build and install a launcher; NAME defaults to the target's file name (--user <USER>: the target runs as USER; --group <GROUP>: with that primary group)
    my-fda destroy [go] <NAME>        delete a launcher and its grant
    my-fda status [<NAME>]            each launcher, its target, its grant, the daemon using it
    my-fda test <NAME>                start that daemon through launchd

A destructive command shows first and acts only with `go`. GONE grants --
the one class safe to take away without looking -- go with `cleanup go`;
everything `list` marks `-> remove` (GONE, or a shell, a general tool, a
script or Homebrew that is switched on) with `remove --recommended go` -- a
job that ran through one of those loses its access, so look at the list
first. Any other
selection goes through the pipe `list` is made for:

    my-fda list | grep 'denied\[0\]' | cut -f 1 | my-fda remove        # shows
    my-fda list | grep 'denied\[0\]' | cut -f 1 | my-fda remove go     # acts

- **denied** finds the jobs that need a grant or a launcher: a job refused
  a file protection just fails, often with a misleading error ("Operation
  not permitted"), and only tccd's lines in the system log say so. One line
  per program, service, answer and tccd: the program as a grant names it,
  the service (`SystemPolicyAllFiles` -- Full Disk Access --,
  `SystemPolicyRemovableVolumes`, ...), the answer with its raw values
  (`denied[0] reason[5]`: authValue, authReason), the tccd that answered
  (`system[0]`, or a user's, `alice[501]`: a volume or folder grant is that
  user's), how often, when last, the binary that touched the file (`via`),
  the process macOS held responsible for it (`from`), and what to do --
  `/bin/dash  SystemPolicyAllFiles ... via diskutil  from dash  a shell` is
  a root shell script running diskutil, and needs a launcher. `from
  sshd-keygen-wrapper` is an ssh login: Remote Login passes no Full Disk
  Access on unless System Settings > General > Sharing > Remote Login (i)
  allows it for remote users. `from -` is a program started with no parent
  macOS names (an app or agent spawning git, say) -- tccd's log does not say
  who, so `denied --catch` waits for the next refusal and shows the process,
  its parents and its open files while it still runs -- tccd names no file
  either, and what the process has open is the nearest there is.
- **A probe is not a refusal.** For a file, macOS asks Full Disk Access
  first and, when that is not allowed, the narrower protection in the
  caller's very next request (its msgID is `<caller pid>.<sequence>`,
  whichever tccd answers). When that one is allowed for the same process,
  within a second, nothing was refused: `denied` counts such probes in its
  summary and shows them with `-V`. Over one observed day, a LaunchDaemon's
  python3.15 got Full Disk Access 0 120 times and its external volume
  allowed 196 times, by the console user's tccd -- no refusal at all.
- **A refusal tccd never answered.** When tccd cannot reach the tccd it
  forwards to (`XPC forwardMessage error: Connection invalid`), or
  sandboxd's request fails (`missing 'auth_value' in reply message`), there
  is no answer in tccd's lines -- the kernel refuses: `System Policy:
  touch(4717) deny(1) file-write-create /private/etc/x`. `denied` lists each
  such kernel line: the process (the kernel names no path), the service and
  the reason from a failure in the second before it, `no answer[<why>]`,
  `from` the program tccd was asked about, the operation and the path:

      touch  SystemPolicySysAdminFiles  no answer[Connection invalid]  tccd system[0]  1x  last ...  via touch  from python3[18476]  file-write-create /private/etc/x -> ...

  With two failures of different services in that second, the service is
  `?` -- never a guess. When `from` is `-` or only a shell (`com.apple.sh`: every
  script runs in one), nothing names the job, and the hint is `denied --catch`,
  never a launcher -- a launcher for the shell would be a grant to every script.
  The kernel's side of a refusal tccd answered is not
  listed twice. Its `Sandbox: ...` lines are other sandboxes' and are left
  out.
- **A kernel refusal with no privacy check.** A `System Policy:` line with
  no tccd or sandboxd failure in the second before it is another macOS
  protection -- a daemon writing under `/private/var/db`, a
  `system-privilege` -- not a privacy refusal: no grant and no launcher
  changes it. `denied` counts such lines in its summary and lists them only
  with `-V` (`other[kernel]`); `denied --catch` waits past them, and with
  `-V` stops at one too -- to look at what does it.
- **An answer with no request.** tccd's answer whose request lines (the
  service, often the program too) are not in the log refuses nothing you
  can name: `denied` counts it in its summary and lists it only with `-V`,
  by its tccd and msgID (`tccd[1645:41246.7230]`), the handle to find it
  in the log.
- **An app or a job.** A program refused a narrower protection is pointed at
  a launcher (`my-fda create`) when it is a path -- a job of yours; an app,
  named by its bundle id, at System Settings > Privacy & Security > Files and
  Folders, where its grant is given -- or left refused.
- **`DENIED_IGNORE`** (config): programs whose refusals are left out, by
  file name or glob, `|` between them, ignoring case, matched against the
  program, `via` and `from` -- counted, shown with `-V`, left out with `-Q`
  too: noise you decided to live with.
- **Which checks:** PROGRAM is a name (`git`), matched ignoring case against
  the program, `via` and `from`, or a path, matched with its links followed;
  a LAST starts with a digit (`30m`, `2h`, `3d`) or is `boot`; instead of
  a LAST, `--since '2026-11-12 08:15:00'` reads from that local time to now,
  however long ago -- so a caller that reads window after window asks each
  from where the last one ended, with no gap and no overlap; `--until
  '<TIME>'` ends it at a time the caller took before it asked, where its
  next window starts (only with `--since`, and not before it). By default
  the services in `DENIED_SERVICES` (`kTCCServiceSystemPolicy*`: the file
  and folder protections); `--service <SERVICE>` every service whose name
  holds SERVICE, ignoring case (`volumes`, `Camera`), `--service all` every
  one.
- **Without Full Disk Access** -- a LaunchDaemon's run -- `denied` still
  reads the log; the grants it cannot read are named in one line, and what
  to do is then from the log alone. `-Q` prints the lines alone.
- **IDs** are 6 hex characters of the client, the same on every run, followed
  by a TAB -- `cut -f 1` returns them.
- **A path is granted as its real path:** macOS records the file with its
  symlinks resolved, so `add` resolves PATH first and says so when it
  differs (`/opt/tools/sbin/x is /Volumes/Data/tools/sbin/x`); `list` shows
  that real path, which is what to `grep` for.
- **Changes** start with a backup (`sqlite3 .backup` into `BACKUP_DIR`,
  never `cp` -- tccd holds the file open). Each grant then goes the one way
  the machine allows: an app id through `tccutil`, macOS's own command,
  which knows installed apps only; the rest -- paths, and apps tccutil does
  not know, such as a GONE one -- through sqlite in one transaction where
  TCC.db can be written, else my-fda opens the pane and names each one as
  the pane does (a path's file name, an app's `.app` name) to take away
  with "-". A GONE app neither tccutil nor sqlite can take stays, said
  once: the pane does not show it either, and it applies to nothing --
  `list` says so too (`GONE: stays`, not `-> remove`), from
  `TCC_DB_WRITABLE`, which `my-fda --install` records -- `list` never writes
  TCC.db to find out (without it: `GONE -> remove (whether it can: ...)`). Then
  the grants are read again, and anything else still there is an error.
  Even root with Full Disk Access may be refused a write (macOS
  26); the probe for it must really write, since sqlite opens such a
  file read-only without a word. As root, the pane and Finder open in the
  session of the user at the screen (`launchctl asuser`): plain `open`
  from root is refused.
- **Listed, never removed:** an Endpoint Security extension's grant
  (TCC.db, `kTCCServiceEndpointSecurityClient` -- the pane shows it with the
  others) and a configuration profile's (`MDMOverrides.plist` -- the pane
  does not show it; it goes with the profile).
- **`reset`** is `tccutil reset SystemPolicyAllFiles`: every grant at once,
  by macOS itself; it saves the list beside the backup first, so the grants
  to give back are known.
- **`add` only guides**: macOS grants on a click (or a device-management
  profile), and writing grants is what malware does.
- **App ids** are `GONE` only when neither Spotlight nor LaunchServices
  knows them: Spotlight misses helper apps inside other apps, and
  LaunchServices, asked as root, may miss apps in a user's
  `~/Applications`. Apple's own are never `GONE`, and none at all when
  Spotlight finds no apps.
- **`test`** is the only run that proves a grant: a run from a Terminal uses
  the Terminal's.

All commands need root, from a Terminal that has Full Disk Access itself:
the permission database is protected. `my-fda --help` is the reference.

## Config

Plain shell, sourced at startup over the defaults: a key the file does not
set keeps its default, so an older config needs no edit when a new key
arrives. `my-fda --create-config` prints the defaults. Search order: `$MY_FDA_CONFIG`, `--config <FILE>`,
`$CONFIG_SEARCH_DIR/my-fda.conf`, `$CONFIG_SEARCH_DIR/my-fda`, `~/.my-fda.conf`,
`/etc/my-fda.conf`, `/usr/local/etc/my-fda.conf`.

**Global, then this machine's.** A config found in `CONFIG_SEARCH_DIR` (a
farm of links into the trees) is read from the trees instead: the site's
`$CONFIG_OFFER_DIR/my-fda.conf`, then `$CONFIG_LOCAL_DIR/my-fda.conf`, this
machine's own, which wins key by key -- whichever of the two the farm links.
`--config` and `$MY_FDA_CONFIG` name one file, read alone.

**`my-fda --install`** (root) records what this machine lets my-fda do,
found out once instead of on every run: today `TCC_DB_WRITABLE`, by a real
write test (macOS refuses it once, there, not on every `list`). The line
goes into `$CONFIG_LOCAL_DIR/my-fda.conf` (created when missing; without
`CONFIG_LOCAL_DIR`, the config it read -- refused where the site has a
`CONFIG_SEARCH_DIR`, whose configs every machine shares) and is committed
where that is a git tree. Run where TCC.db cannot even be opened (no Full
Disk Access), it records nothing and fails: the write was never tried. A
second run changes nothing and says so; run it again after a macOS update or
a change of grant.

**A site file, read first.** `/LINKS/default/site.conf`, when it exists, is
read before the config -- the one site path in my-fda. Plain `KEY="value"`
lines, for every tool of a site: `DATA_BASE_DIR` (`BACKUP_DIR`'s default is
`$DATA_BASE_DIR/<user>/my-fda`), `LOG_BASE_DIR`, `CONFIG_SEARCH_DIR` (searched
as above), `CONFIG_OFFER_DIR` and `CONFIG_OFFER_NOTE` (where "no config found"
offers to write one, and the note beside it), `CONFIG_LOCAL_DIR` (this
machine's config directory: read after the global one, written by
`--install`). Without the file: `/var/db`
and `/var/log`, no search directory, no site offer -- my-fda works with no
setup. There but unreadable: an error, never the fallback, which would keep
this user's state apart from everyone else's. Environment variables are never
read for these: a LaunchDaemon, `sudo` and a login shell would each see
different ones.

## Shell completion

Each run writes the zsh completion to `~/.zsh/completions/_my-fda` when it
is missing or older (mode 644, and the zcompdump goes, so a new shell loads
it), and gives `~/.zshrc` the `fpath` line once -- before oh-my-zsh when it
is there. It says so in one line when it writes; a run with nothing to write
says nothing. The launcher names it offers are those in `HELPER_DIR`.

## Tests

    my-fda --run-tests [<FILTER>]

They run against a TCC.db of their own, with stubs for launchctl, open and
Spotlight -- never against the real one, and without root.

Each test builds its own world, so they run in parallel: dealt round-robin
into one block per core, each numbered by its place in the list, a block
that dies leaving its tests `not ok`. The stub commands are written once per
run -- macOS checks every new executable on its first run, through a service
all blocks share, and fresh stubs per test made them queue there. The 86
tests of 2026-10-26: 22.7 s one by one, 3.4 s now (16 cores), the same results.

Ctrl-C is tested with a real SIGINT, sent while a removal waits for its
Return: exit 130, one line on stderr naming where it stopped, nothing after
it run. A block runs in the background, where dash ignores SIGINT for good,
so that test starts a fresh process through perl, which puts SIGINT back to
its default (skipped without perl).
