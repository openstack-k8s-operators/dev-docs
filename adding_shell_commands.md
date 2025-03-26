# Adding shell commands to pod containers

Due to the restrictive nature of kubernetes and CoreOS, containerized
OpenStack services do not have access to the same shell commands available
when they run in an Enterprise Linux environment. For example, common debug
tools like `ping` and `ncat` are not available in containers running in a
podified controlplane.

However, it is possible to install commands into podified containers. Here is
an example that installs the `ip` command into the cinder-backup container in
the cinder-backup-0 pod.

## Prerequisites

The command's executable should be installed from a source OS that matches the
one associated with the container. For example, when the container's image is
based on CentOS Stream 9, then you should install CS-9's version of the
command. In the following example, the host machine (on which the installation
instructions are executed) and the cinder-backup container are based on the
same OS. Other OS combinations may work, but your mileage may vary.

## Procedure

1. Find the absolute path of the command you wish to install

```shell
realpath $(which ip)
```

The answer is `/usr/sbin/ip`. Some commands like `nc` are actually symlinks,
and it's necessary to install the absolute file.

2. Use `kubectl` to copy the file into the container

```shell
kubectl cp /usr/sbin/ip cinder-backup-0:/usr/sbin/
```

If necessary, you can include `"-c container"` to specify a particular
container within the pod.

3. Check to see if additional libraries need to be installed

```shell
oc rsh cinder-backup-0 ldd /usr/sbin/ip
```

Look for any "libXXX => not found" lines, seen here:

```
        linux-vdso.so.1 (0x00007fff583f9000)
        libbpf.so.1 => not found
        libelf.so.1 => /lib64/libelf.so.1 (0x00007f459e007000)
        libmnl.so.0 => not found
        libcap.so.2 => /lib64/libcap.so.2 (0x00007f459dffd000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f459ddf4000)
        libz.so.1 => /lib64/libz.so.1 (0x00007f459ddda000)
        libzstd.so.1 => /lib64/libzstd.so.1 (0x00007f459dd01000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f459e18e000)
```

4. Install any missing libraries

In this example, `libbpf.so.1` and `libmnl.so.0` were not found, so the host
files in /lib64 need to be installed. However, these are just symlinks, and
it's necessary to install both the symlink and absolute file from the host.
Note that `kubectl` copies symlinks intact. It creates a corresponding symlink
in the container, and does not follow the source's symlink.

```shell
# Install the symlinks from the ldd output
kubectl cp /lib64/libbpf.so.1 cinder-backup-0:/lib64/
kubectl cp /lib64/libmnl.so.0 cinder-backup-0:/lib64/

# Copy the absolute files referenced by the symlinks
kubectl cp /lib64/libbpf.so.1.2.0 cinder-backup-0:/lib64/
kubectl cp /lib64/libmnl.so.0.2.0 cinder-backup-0:/lib64/
```

The `ip` command is now available for use within in the cinder-backup
container.
