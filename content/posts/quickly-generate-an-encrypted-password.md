+++
title = "quickly generate an encrypted password"
date = 2012-02-12
+++

Hi everybody !  Here's a quick method for generating encrypted passwords that are suitable for things like `/etc/passwd` .  I realise that this isn't terribly complex, but honestly, I always forget how to do this until I actually need to do it - so here's a reminder for all of us. :)

```bash
#!/bin/bash

if [ "x$1" == 'x' ]; then
echo "USAGE: $0 'password'"
exit 1
fi


# Get an md5sum of the password string; this is used for the SHA seed.
md5=$( echo $1 | md5sum )
extract="${md5:2:8}"


# Calculate the SHA hash of the password string using the extracted seed.
mkpasswd -m SHA-512 "$1" "$extract"
exit $?
```