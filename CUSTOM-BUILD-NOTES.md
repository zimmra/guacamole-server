# Custom guacd build notes

This fork carries local changes needed for VNC connections to modern macOS
Screen Sharing. The custom build is maintained on the
`dev/1.6.1-custom` branch and is intended to be built directly on the Debian
VM where `guacd` runs.

The branch is based on Apache Guacamole's `staging/1.6.1` branch. When rebasing
onto a newer Apache release, reapply and verify each change described below.

## Local changes

### Restrict VNC authentication

Commit `73f8f0a6` configures LibVNCClient with an explicit authentication
allow-list containing:

* `rfbNoAuth`
* `rfbVncAuth` (standard VNC password authentication, security type 2)

Apple Remote Desktop authentication (`rfbARD`, security type 30) is
deliberately omitted. LibVNCClient's ARD implementation does not work with the
modern macOS Screen Sharing server involved here. Omitting it allows
LibVNCClient to select standard VNC password authentication when the server
offers it.

A successful negotiation should include messages similar to:

```text
We have 5 security types
30, 33, 36, 2, 35
Selecting security type 2
Selected Security Scheme 2
VNC authentication succeeded
Connected using RFB 3.8, security: VNC (2).
```

This allow-list also excludes every other VNC authentication scheme, including
TLS-based schemes. If support for another scheme is later required, review and
extend `guac_vnc_allowed_auth_schemes` in `src/protocols/vnc/vnc.c`.
`rfbNoAuth` means this client will accept an unauthenticated VNC server if one
is intentionally configured.

### Detect LibVNCServer's current Gcrypt macro

Commit `c4209504` supports both names used by LibVNCServer to indicate that its
VNC client was compiled with Libgcrypt:

* `LIBVNCSERVER_WITH_CLIENT_GCRYPT` (older)
* `LIBVNCSERVER_HAVE_LIBGCRYPT` (current)

Without this change, Guacamole can compile out its Gcrypt initialization even
though the installed `libvncclient` uses Libgcrypt. The relevant diagnostics
on the Debian VM were:

```text
#define LIBVNCSERVER_HAVE_LIBGCRYPT 1
libgcrypt.so.20 => /lib/x86_64-linux-gnu/libgcrypt.so.20
```

After the fix, debug logging should include:

```text
GCrypt initialization started.
GCrypt initialization completed.
```

### Ignore resize events until VNC initialization completes

Commit `f5dcd2a7` prevents the VNC size handler from dereferencing the
`rfbClient` before the VNC connection thread has assigned it.

Termix can send a Guacamole `size` instruction immediately after its handshake.
This can race with `rfbInitClient()`. Authentication may succeed, but the
`user-input` thread then crashes while processing that early size instruction.
GDB identified the failure as:

```text
Thread "user-input" received SIGSEGV
#0 rfbClientGetClientData()
#1 guac_vnc_display_set_size(client=0x0, ...)
#2 guac_vnc_user_size_handler(...)
#3 guac_user_input_thread(...)
```

The handler now follows the existing mouse and keyboard handlers and sends the
resize only when `vnc_client->rfb_client` is non-NULL. The initial requested
display size is retained by the Guacamole handshake and applied after VNC
initialization, so ignoring this premature event does not lose the initial
size.

## Full Debian build and installation

These commands assume:

* The source checkout is `/opt/guacamole-server`.
* Commands run as `root`, so `sudo` is unnecessary.
* An existing systemd unit already manages `guacd`.
* The desired installation prefix is `/usr/local`.
* Dependencies for RDP, terminal, SSH, telnet, and VNC are installed.

```bash
cd /opt/guacamole-server

git remote set-url origin https://github.com/zimmra/guacamole-server.git
git fetch origin
git switch dev/1.6.1-custom
git pull --ff-only origin dev/1.6.1-custom

make distclean 2>/dev/null || true
autoreconf -fi

export CPPFLAGS="-Wno-error=deprecated-declarations"

./configure \
    --prefix=/usr/local \
    --with-rdp \
    --with-terminal \
    --with-ssh \
    --with-telnet \
    --with-vnc \
    --disable-kubernetes \
    --disable-guacenc \
    --disable-guaclog

make -j"$(nproc)"

systemctl stop guacd
make install
ldconfig
systemctl start guacd

systemctl status guacd --no-pager -l
journalctl -u guacd -f
```

The `CPPFLAGS` setting avoids treating deprecation warnings from the installed
FreeRDP headers as fatal errors. It does not disable the RDP protocol.

It is expected for the configure summary to report both `Init scripts: no` and
`Systemd units: no` in this installation. The VM already has
`/etc/systemd/system/guacd.service`, so this build should not install or replace
the service unit.

`make install` replaces `/usr/local/sbin/guacd` and installs the matching
libraries and protocol plugins under `/usr/local/lib`. Running `ldconfig`
afterward ensures the dynamic linker sees those new libraries.

## Incremental rebuild after a source-only update

If `configure.ac` and other build-system inputs have not changed since the last
full build:

```bash
cd /opt/guacamole-server

git fetch origin
git switch dev/1.6.1-custom
git pull --ff-only origin dev/1.6.1-custom

export CPPFLAGS="-Wno-error=deprecated-declarations"
make -j"$(nproc)"

systemctl stop guacd
make install
ldconfig
systemctl start guacd

systemctl status guacd --no-pager -l
journalctl -u guacd -f
```

Run the full build instead whenever `configure.ac`, `Makefile.am`, dependencies,
or compiler options have changed.

## Troubleshooting notes

Messages like these can result from a health check opening the guacd socket and
disconnecting without completing the Guacamole protocol handshake:

```text
Guacamole connection closed during handshake
Error reading "select": End of stream reached while reading instruction
```

They are not, by themselves, evidence that the VNC connection failed. Locate
the connection ID for the real attempt and follow its subsequent authentication
and process messages.

These messages following a connection attempt were a consequence of the client
child process crashing, not the original cause:

```text
Connection "$..." removed.
Unable to request termination of client process: No such process
```

If the problem returns after an upstream rebase, first confirm that all three
custom commits (or equivalent changes) remain present:

```bash
git log --oneline --decorate -10
git grep -n "guac_vnc_allowed_auth_schemes"
git grep -n "GUAC_VNC_WITH_CLIENT_GCRYPT"
git grep -n "Send display update only if finished connecting"
```

Then verify which installed binaries and libraries are active:

```bash
command -v guacd
readlink -f /usr/local/lib/libguac.so.25
readlink -f /usr/local/lib/libguac-terminal.so.2
ldd /usr/local/lib/libvncclient.so.1 | grep -E 'gcrypt|gnutls|ssl|crypto'
systemctl cat guacd
```
