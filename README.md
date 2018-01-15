# SSH bringup

## Create a non-root user with passwordless `sudo`

1.  Create the passwordless `sudo` group

    ```bash
    groupadd sudo-nopasswd
    ```

2.  Give the group `sudo` permission

    Write to `/etc/sudoers.d/sudo-nopasswd`:

    ```
    # Allow users in the sudo-nopasswd group to run any command without
    # being prompted for a password.
    %sudo-nopasswd ALL=(ALL) NOPASSWD:ALL
    ```

3.  Create a user in the group

    ```bash
    useradd "${NEW_USER}" --shell /bin/bash --create-home
    usermod "${NEW_USER}" --append --groups sudo-nopasswd
    ```

## Install `authorized_keys` script

1.  Download the script

    ```bash
    curl -L "${URL_FOR_SCRIPT}" -o /usr/local/download_authorized_keys

    # The script must be owned and only writable by root.
    chmod 755 /usr/local/download_authorized_keys
    ```

2.  Create a user to run the script

    ```bash
    # Add a system user (and associated group) with no login privilege.
    useradd authorized_keys_command_user --system --shell /bin/false --user-group
    ```

3.  Configure `sshd` to use the script

    Add to `/etc/ssh/sshd_config`:

    ```
    # Tokens:
    #   %f - the fingerprint of the key or certificate
    #   %u - the username
    #   %h - the home directory of the user
    #   %k - the base64-encoded key or certificate for authentication
    #   %t - the key or certificate type 
    AuthorizedKeysCommand /usr/local/download_authorized_keys "%f" "%u" "%h" "%k" "%t"

    # AuthorizedKeyCommandUser is required if AuthorizedKeyCommand is set.
    AuthorizedKeysCommandUser authorized_keys_command_user
    ```

    _(Consider keeping `sshd_config` open for the next step.)_

## Configure `sshd` to use `publickey` authentication

Add to `/etc/ssh/sshd_config`:

```
AuthenticationMethods publickey
ChallengeResponseAuthentication no
PasswordAuthentication no

PermitRootLogin prohibit-password
```

## Read new `sshd` configuration

Make sure to keep an `ssh` session open in case `sshd` can't read the new configuration.

```bash
# Restart sshd and read sshd_config
service ssh restart

# Check that sshd_config was read successfully
service ssh status

# ... also test logging in as ${NEW_USER}.
```

## Other ideas

Assuming we restrict ourselves to publickey authentication, maintaining SSH access breaks down into a few inter-related parts:
1.  What goes in `authorized_keys`
2.  When and how to update `authorized_keys` on servers
3.  When and how to update client keys

Consider the most straightforward approach to SSH access: one keypair, stored in a file on each client and accepted by each server. If an attacker compromises _any_ client, they get access to all servers forever. To repair the breach, all clients and all servers need to be updated.

continued...


These are the scenarios we want to make easy:
- Provision a new server
- Grant client access
- Revoke client access (e.g. if compromised)

Bonus:
- Grant access to diverse clients: OpenSSH, PuTTY, random iOS apps

The most straightforward approach is one keypair, copied to each client and listed in `authenticated_keys` on each server. Setting up a new client or server is really easy, but revoking access requires touching every client and server.

Add server - O(1)
Add client - O(1)
Remove client - O(C + S)

Other considerations:
* We can't use devices that generate the keypair internally and only share the public key.
* The key can be protected by a passphrase, but that doesn't protect against brute-force attacks or keylogging.
* It's not easy to securely transport a new key to all clients.


A slightly more intricate approach uses one key per client, with all client keys listed in `authenticated_keys` on each server. Setting up a new server is still easy. Revoking a client still requires touching every server, but only one client. But now setting up a new client *also* requires touching every server.

Add server - O(1)
Add client - O(S)
Remove client - O(S)


In order to avoid updating `authorized_keys` for every new client, we can use OpenSSH certificates. We create a certificate authority (CA) key and sign client keys with it. Servers use `TrustedUserCAKeys` or a `@cert-authority` line in `authorized_keys` to trust the CA key.

If a client is compromised, its key must be revoked at all servers. SSH keys can be revoked in `sshd_config` (`RevokedKeys`) or `authorized_keys` (by listing the key explicitly with `from="!*"`). This must be done at all servers.

Alternatively, we can set a valid interval for the OpenSSH certificates. Assuming certificates have a short enough expiration, a missing or compromised client doesn't require a response. That said, this depends on a service (like [Netflix's BLESS](https://github.com/Netflix/bless) or [the cloudtools `ssh-cert-authority`](https://github.com/cloudtools/ssh-cert-authority)) to respond to signing requests. That service must hold the CA key online. If that service is compromised, and the CA key is extracted, then all servers must be updated.

Add server - O(1)
Add client - O(1)
Remove client:
- _with expiration_: O(1)
- _without expiration_: O(S)
Roll certificate authority key:
- _with expiration_: O(S)
- _without expiration_: O(C + S)

Other considerations:
- Without the signing service, this provides similar security to one key shared by all clients.
- If using a signing service, that service needs to be kept available and secure. It needs to implement its own authentication. And it needs to have a revocation list for client keys that are compromised and should no longer have certificates issued.
- It's a little weird that the signing service has permanent access to a key that entitles it to connect to any server, even though it should never make such a connection. There are some ways to mitigate this and require the server to cooperate with a previously-authorized client:
  - We can encrypt the CA key to _yet another_ key (symmetric or asymmetric) held by the service, then give the encrypted CA key to clients. Clients will send the encrypted CA key with requests. If the server authorizes the request, it will unseal and instrument the key and then quickly forget it until the next request.
  - We can split the signing key between clients and the service. Then a client and the service need to _cooperate_ in order to create a valid OpenSSH certificate. If the signing key is RSA, we can use additive or multiplicative splitting. If it's an ECDH key, there are trickier (and chattier) solutions involving homomorphic encryption. Each client should have a different keyshare, though, so this demands _another_ service with access to the CA key to create keyshares when provisioning clients. Maybe we could pre-create a lot of keyshares and store them somewhere?
  With each of these solutions, rotating the CA key now requires touching every client.


### Configuration management

Distribute `authorized_keys` with Chef, Ansible, Puppet, Salt, etc.

### OpenSSH certificates and certificate authorities

[Reference](https://blog.habets.se/2011/07/OpenSSH-certificates.html)

These would use `@cert-authority` lines in `authorized_keys`.

*   [Netflix BLESS](https://github.com/Netflix/bless)
*   [Uber's SSH CA](https://medium.com/uber-security-privacy/introducing-the-uber-ssh-certificate-authority-4f840839c5cc)
*   [Cloudtools `ssh-ca`](https://github.com/cloudtools/ssh-ca)

These are specific to OpenSSH, so they won't work with other clients like PuTTY.

### RSA key splitting

