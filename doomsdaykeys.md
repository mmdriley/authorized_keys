# Doomsday SSH keys

We want to configure hosts with "last resort" SSH keys that can always be used to log on, even if other methods of SSH key management have failed.

Since the whole point of these keys is they "always work", we don't want to have do anything clever to manage or rotate them. But if we can't change them easily, then the consequences of key compromise are much more dire.

If we use **hardware-backed** keys, we can be sure the keys are safe as long as we're in possession of the hardware token. And unlike more onerous solutions like offline/paper keys, we can test that the keys work without impacting their confidentiality and forcing a key rotation.

## More than FIDO?

OpenSSH has supported FIDO/U2F authenticators since [version 8.2](https://www.openssh.com/txt/release-8.2). This is relatively straightforward to set up and it works with most Yubikeys, including Yubico's most affordable "Security Key" line.

However, OpenSSH authentication with a security key requires both the client and server to be running at least OpenSSH 8.2. For Ubuntu installs, this means running at least 20.04. There are also some obstacles to getting this working for Windows or Mac clients.

## Smartcard authentication

A smaller segment of the Yubikey product line can act as PIV-compatible smartcards. One example is the Yubikey 5.

Smartcard-rooted SSH credentials are harder to set up and use, but are compatible with more software. For the best mix of compatibility and usability, consider setting up **both** security key and PIV logins. They can both use the same Yubikey.

### Generating the certificate

First, make sure the Yubikey's smartcard PIN is set. You can use the Yubikey manager for this. Remember the default PIN is `123456`.

Next, get the `yubico-piv-tool`. At time of writing it was available [here](https://www.yubico.com/products/services-software/download/smart-card-drivers-tools/).

Use this command to generate, self-sign, and import an ECDH certificate:

```bash
yubico-piv-tool --slot 9a --action generate --action verify-pin --action selfsign-certificate --action import-certificate --algorithm ECCP256 --subject "/CN=example.com/O=ssh" --valid-days 1 --verbose
```

The certificate is placed in slot `9a`, which is [used for PIV authentication](https://developers.yubico.com/PIV/Introduction/Certificate_slots.html).

The validity interval (`--valid-days`) **does not** affect use of the key. We use an interval of one day to make it obvious we're not worried about certificate expiration.

The certificate subject (`--subject`) is likewise not relevant to use of the key.

Yubikeys support RSA and ECC options for `--algorithm`. We choose `ECCP256` for good security with small keys.

## Getting the SSH public key

Use `yubico-piv-tool` to export the certificate, `openssl` to get certificate public key, and `ssh-keygen` to reformat it for use in `authorized_keys`:

```bash
yubico-piv-tool.exe --slot 9a --action read-cert | \
    openssl x509 -inform PEM -noout -pubkey | \
    ssh-keygen -i -m pkcs8 -f /dev/stdin
# prints "ecdsa-sha2-nistp256 AAAAE2V..."
```

## Allowing login with the key

Put the doomsday keys in `~/.ssh/authorized_keys` for the user they should authorize. Note OpenSSH will check this file first before using more exotic mechanisms like `AuthorizedKeysCommand`.

## Authenticating with PIV

[This page](https://piv.idmanagement.gov/engineering/ssh/) from the US General Services Administration is a great collection of resources for authenticating with PIV over SSH on Windows and Mac.

On Mac, add the key to `ssh-agent`:

```bash
ssh-add -s /usr/lib/ssh-keychain.dylib
```

On Windows, use [PuTTY-CAC](https://github.com/NoMoreFood/putty-cac/releases).

## Checking the right key is being used

If you've logged on to an OpenSSH server and want to make sure the Yubikey's certificate was used, you can pull the key fingerprint from the logs.

```bash
journalctl -u ssh.service --since=-10m
```

Look for a line like:

```
Aug 15 15:15:15 server-hostname sshd[3185]: Accepted publickey for sphinx from 2607:f8b0:400a:800::200e%eth0 port 53861 ssh2: ECDSA SHA256:PLFBQDIxWZ4H1xQtYIUBUxlEyThFxHgv7DTMpWyjc+M
```

After `SHA256:` on that line is the base64-encoded SHA-256 hash of the SSH public key.

Assuming the public part of the SSH key you created above was something like:

```
ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPqrQKgI7uD+3MYNLhmfOZoyjifmv0SjvvGtZGcRiKb5g39oyreutg9OCOppWSyjD2GyrN3KfEmky+s6CRRcvAY= user@server
```

You can get the expected fingerprint as:

```bash
$ echo 'AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPqrQKgI7uD+3MYNLhmfOZoyjifmv0SjvvGtZGcRiKb5g39oyreutg9OCOppWSyjD2GyrN3KfEmky+s6CRRcvAY=' | base64 -d | shasum -a256 | cut -f1 -d' ' | xxd -r -p | base64
PLFBQDIxWZ4H1xQtYIUBUxlEyThFxHgv7DTMpWyjc+M=
```

... which matches what we saw in the log, modulo the trailing `=`.

## Other references

- [Yubico PIV introduction](https://developers.yubico.com/yubico-piv-tool/YubiKey_PIV_introduction.html) -- also describes default PIV parameters and setup steps
- [`yubico-piv-tool` docs](https://developers.yubico.com/yubico-piv-tool/)
