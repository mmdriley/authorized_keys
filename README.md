# Manage SSH with Google Docs

Steps to use a Google Doc as your `authorized_keys` file across some servers.

Probably most appropriate in a deployment with a small numbers of users and servers.

## One-time setup

### Create the `authorized_keys` Google Doc

1.  Create a new [Google Doc](https://docs.google.com/document).

2.  Add the `authorized_keys` contents, e.g.:

    ```
    # computer1
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC1LQMZJkoWNJrai7qJ6ZI7yqXTZijQd9E/onI01dR2bA1Mvrmbz/BL0tJIrwNxVpNCUn9Os4svPy9ITIrkKg6rlxHMwW1D9oEc7grrFaM2jvhaF/GrMKuD1gC+kRYW5eaZqdcP7njRO8+ciwVImb3sw+mSAvSKUcIvHby8yGEVU2I+p3I35YRSSN1KH+BFPQRE/jd0U4Qm1a5ZI5LWL6cUbFLv5OzHp8nun+BNQStxMe6bjHcXJRtH+8LxXs5meTTo/MOUSUgPIFSYlUF1ujHJio02NXJatlWn6t1IbHMm86JAc6uSOvQNUmEB0PbUdAbV8QCS9k84xz7AzpAJC/U3

    # computer2
    ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAGDUSry9MFpslgCZhingWShnvszp9Aw7KuDlutVi+bl
    ```

3. Change the sharing settings for the document so "anyone with the link can **view**".

## Per-machine bringup

### Create a non-root user with passwordless `sudo`

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

### Install `authorized_keys` script

1.  Download the script

    ```bash
    curl -L "${URL_FOR_SCRIPT}" -o /usr/local/download_authorized_keys

    # The script must be owned and only writable by root.
    chmod 755 /usr/local/download_authorized_keys
    ```

2.  Configure the script

    In `download_authorized_keys`, replace `GOOGLE_DOC_ID` with the ID of the document you created. For example, if the document URL is:
    ```
    https://docs.google.com/document/d/4oVJ6K5g2LOhqlgrblto5WYTasVebsPJGbsHSmVXNyQe/edit`
    ```

    then set `GOOGLE_DOC_ID` as:
    ```
    GOOGLE_DOC_ID=4oVJ6K5g2LOhqlgrblto5WYTasVebsPJGbsHSmVXNyQe
    ```

3.  Create a user to run the script

    ```bash
    # Add a system user (and associated group) with no login privilege.
    useradd authorized_keys_command_user --system --shell /bin/false --user-group
    ```

4.  Configure `sshd` to use the script

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

### Configure `sshd` to use `publickey` authentication

Add to `/etc/ssh/sshd_config`:

```
AuthenticationMethods publickey
ChallengeResponseAuthentication no
PasswordAuthentication no

PermitRootLogin prohibit-password
```

### Read new `sshd` configuration

Make sure to keep an `ssh` session open in case `sshd` can't read the new configuration.

```bash
# Restart sshd and read sshd_config
service ssh restart

# Check that sshd_config was read successfully
service ssh status

# ... also test logging in as ${NEW_USER}.
```

## Belt and suspenders

If you rely entirely on `AuthorizedKeysCommand` to download `authorized_keys` from Google Docs, you might lose SSH access to your hosts if:
- you lose access to your Google account
- Google is down
- Google changes the URL scheme for exporting documents as text

To account for this, you may want to create a "fallback" key and put it in `authorized_keys` on each host. This key should probably stay offline or be rooted at a Yubikey since it will be so hard to revoke.
