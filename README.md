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
    useradd -m "${NEW_USER}" --shell /bin/bash
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
    useradd --system --shell /bin/false --user-group authorized_keys_command_user
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

# ... also test logging in as ${NEW_USER}!
```
