# TODO

* Write logs to some server on each invocation so we know what servers have
  run this and with what arguments.

  Recall the script can be configured to take a number of arguments, like:
  ```
           %%    A literal ‘%’.
           %f    The fingerprint of the key or certificate.
           %h    The home directory of the user.
           %k    The base64-encoded key or certificate for authentication.
           %t    The key or certificate type.
           %u    The username.
  ```
