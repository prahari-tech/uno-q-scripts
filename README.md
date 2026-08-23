# uno-q-scripts

Two bash scripts for working with an [Arduino UNO Q](https://store.arduino.cc/products/uno-q):
one to provision a freshly flashed board, one to deploy an app to it.

They came out of setting up a 4GB UNO Q for an eldercare robot. The write-up,
including the five things a fresh board gets wrong and why each script does what
it does, is here:

**[Setting Up My 4GB Arduino UNO Q: Two Scripts and Five Traps](https://prahari.net/blog/setting-up-my-4gb-arduino-uno-q-two-scripts-and-five-traps)**

## `bin/q-setup`

Takes a board straight from `arduino-flasher-cli flash latest` to one you can
reach over SSH. Everything runs over ADB, because a freshly flashed board has no
network yet.

```bash
bin/q-setup doctor   # report what is and isn't ready
bin/q-setup all      # do everything
```

It handles, in order: waiting for ADB, setting the board name, joining WiFi,
generating SSH host keys, installing your public key, setting the account
password, dropping stale entries from `known_hosts`, synchronising the clock,
refreshing the package and library indexes, upgrading the Arduino packages,
pre-pulling app-bricks images, reclaiming disk, and verifying key auth works.

Five things it exists to work around:

1. **No SSH host keys.** The image ships without any, so `sshd` refuses to start
   until `dpkg-reconfigure openssh-server` generates them.
2. **No account password.** The account ships `NP` with a forced-change flag, so
   sshd authenticates your key and then PAM refuses the session — which looks
   exactly like a broken key.
3. **The clock is months behind.** No network means no NTP, so the board boots
   believing it is the image build date, and every TLS handshake fails on
   "not yet valid" certificates.
4. **mDNS is off.** `system set-name` reports success, but `avahi-daemon` ships
   disabled, so `<name>.local` resolves nowhere.
5. **The friendly wrappers prompt for a password.** The underlying commands are
   already `NOPASSWD` in the board's sudoers, so driving those directly is what
   makes unattended provisioning possible.

## `bin/q-deploy`

Pushes an app to the board and starts it, over SSH if the board answers and ADB
over USB otherwise.

```bash
bin/q-deploy deploy q-bridge
```

It encodes two things that are easy to get wrong: `TMPDIR=/tmp` (without it the
build looks for an Android path on a Debian board), and pushing an app
directory's *children* rather than the directory — `adb push` nests the source
inside the destination when the destination already exists.

## Configuration

```bash
cp bin/board.env.example bin/board.env && chmod 600 bin/board.env
```

`bin/board.env` holds the WiFi passphrase and the board password. **It is
gitignored and must stay that way.** Anything left empty is prompted for at run
time instead, so filling it in is optional — it only makes unattended reruns
possible.

Secrets are only ever piped over stdin, never passed as arguments, so they do
not appear in the board's process list.

## Requirements

`adb`, `ssh`, and a board reachable over USB. Tested against image 2026-05-28
(kernel 7.0.0, Zephyr core 0.55.2) on a 4GB UNO Q.

These are the scripts I actually use, not a polished tool. Corrections and
issues welcome.
