# SFTP

![Docker Automated build](https://img.shields.io/docker/automated/ocasta/s3-sftp.svg) ![Docker Build Status](https://img.shields.io/docker/build/ocasta/s3-sftp.svg) ![Docker Stars](https://img.shields.io/docker/stars/ocasta/s3-sftp.svg) ![Docker Pulls](https://img.shields.io/docker/pulls/ocasta/s3-sftp.svg)

![OpenSSH logo](https://raw.githubusercontent.com/ocastastudios/s3-sftp/master/openssh.png "Powered by OpenSSH")

# Supported tags and respective `Dockerfile` links

- [`debian-stretch`, `debian`, `latest` (*Dockerfile*)](https://github.com/ocastastudios/s3-sftp/blob/master/Dockerfile) [![](https://images.microbadger.com/badges/image/ocasta/s3-sftp.svg)](http://microbadger.com/images/ocasta/s3-sftp "Get your own image badge on microbadger.com")

# Securely share your files

Easy to use SFTP ([SSH File Transfer Protocol](https://en.wikipedia.org/wiki/SSH_File_Transfer_Protocol)) server with [OpenSSH](https://en.wikipedia.org/wiki/OpenSSH).
This is an automated build linked with the [debian](https://hub.docker.com/_/debian/) repository.

# Usage

- Define users in (1) command arguments, (2) `SFTP_USERS` environment variable
  or (3) in file mounted as `/etc/sftp/users.conf` (syntax:
  `user:pass[:e][:uid[:gid[:dir1[,dir2]...]]] ...`, see below for examples)
  - Set UID/GID manually for your users if you want them to make changes to
    your mounted volumes with permissions matching your host filesystem.
  - Directory names at the end will be created under user's home directory with
    write permission, if they aren't already present.
- Mount volumes
  - The users are chrooted to their home directory, so you can mount the
    volumes in separate directories inside the user's home directory
    (/home/user/**mounted-directory**) or just mount the whole **/home** directory.
    Just remember that the users can't create new files directly under their
    own home directory, so make sure there are at least one subdirectory if you
    want them to upload files.
  - For consistent server fingerprint, mount your own host keys (i.e. `/etc/ssh/ssh_host_*`)
- Each user gets a folder in their home directory called 'uploads' which is mapped to the S3 bucket /$user/uploads

# Examples

## Simplest docker run example

```
docker run -p 22:22 \
    -d --privileged \
    -v /host/secrets/S3_ACCESS_KEY_ID:/run/secrets/S3_ACCESS_KEY_ID \
    -v /host/secrets/S3_SECRET_ACCESS_KEY:/run/secrets/S3_SECRET_ACCESS_KEY \
    -e S3_BUCKET=your-bucket-name \
    -e S3_REGION=eu-west-2 \
    ocasta/s3-sftp foo:pass:::
```

S3 access key and secret should be provided as secrets.

User "foo" with password "pass" can login with sftp and upload files to a folder called "uploads". No mounted directories or custom UID/GID.

The following examples have the S3 information and --privileged omitted to keep the differences easier to see.

### Using Docker Compose:

```
sftp:
    image: ocasta/s3-sftp
    ports:
        - "2222:22"
    command: foo:pass:1001
```

### Logging in

The OpenSSH server runs by default on port 22, and in this example, we are forwarding the container's port 22 to the host's port 2222. To log in with the OpenSSH client, run: `sftp -P 2222 foo@<host-ip>`

## Store users in config

```
docker run \
    -v /host/users.conf:/etc/sftp/users.conf:ro \
    -p 2222:22 -d ocasta/s3-sftp
```

/host/users.conf:

```
foo:123:1001:100
bar:abc:1002:100
baz:xyz:1003:100
```

## Encrypted password

Add `:e` behind password to mark it as encrypted. Use single quotes if using terminal.

```
docker run \
    -p 2222:22 -d ocasta/s3-sftp \
    'foo:$1$0G2g0GSt$ewU0t6GXG15.0hWoOX8X9.:e:1001'
```

Tip: you can use [atmoz/makepasswd](https://hub.docker.com/r/atmoz/makepasswd/) to generate encrypted passwords:
`echo -n "your-password" | docker run -i --rm atmoz/makepasswd --crypt-md5 --clearfrom=-`

## Logging in with SSH keys

Mount public keys in the `/run/secrets/` secrets directory (or as Docker secrets). All keys are automatically appended to `.ssh/authorized_keys`. In this example, we do not provide any password, so the user `foo` can only login with his SSH key.

The secrets must be named in the following format `${user}_ssh_key_*`

```
docker run \
    -v /host/id_rsa.pub:/run/secrets/foo_ssh_key_id_rsa.pub:ro \
    -v /host/id_other.pub:/run/secrets/foo_ssh_key_id_other.pub:ro \
    -p 2222:22 -d ocasta/s3-sftp \
    foo::1001
```

## Providing your own SSH host key (recommended)

This container will generate new SSH host keys at first run. To avoid that your users get a MITM warning when you recreate your container (and the host keys changes), you can mount your own host keys.

```
docker run \
    -v /host/ssh_host_ed25519_key:/run/secrets/ssh_host_ed25519_key \
    -v /host/ssh_host_rsa_key:/run/secrets/ssh_host_rsa_key \
    -v /host/share:/home/foo/share \
    -p 2222:22 -d ocasta/s3-sftp \
    foo::1001
```

Tip: you can generate your keys with these commands:

```
ssh-keygen -t ed25519 -f ssh_host_ed25519_key < /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key < /dev/null
```

## Execute custom scripts or applications

Put your programs in `/etc/sftp.d/` and it will automatically run when the container starts.
See next section for an example.

# What version of OpenSSH do I get?

- [List of `openssh-server` packages on Debian releases](https://packages.debian.org/search?keywords=openssh-server&searchon=names&exact=1&suite=all&section=main)

**Note:** The time when this image was last built can delay the availability of an OpenSSH release. Since this is an automated build linked with [debian](https://hub.docker.com/_/debian/) repo, the build will depend on how often they push changes (out of my control).  Typically this can take 1-5 days, but it can also take longer. You can of course make this more predictable by cloning this repo and run your own build manually.
