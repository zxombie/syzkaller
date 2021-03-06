# Setup

Instructions for running OpenBSD host, OpenBSD vm, amd64 kernel.
In addition, the host must be running `-current`.

Variables used throughout the instructions:

- `$KERNEL` - Custom built kernel, see [Compile Kernel](#compile-kernel).
              Defaults to `/sys/arch/amd64/compile/GENERIC/obj/bsd` if the
              instructions are honored.
- `$SSHKEY` - Public SSH key ***without a passphrase*** used to connect to the
              VMs, it's advised to use a dedicated key.
- `$USER`   - The name of the user intended to run syzkaller.
- `$VMDIR`  - Directory containing VM disk images.
- `$VMID`   - The numeric ID of last started VM.

## Install syzkaller

1. Install dependencies:

   ```sh
   # pkg_add bash git gmake go
   ```

2. Clone repository:

   ```sh
   $ mkdir -p ~/go/src/github.com/google
   $ cd ~/go/src/github.com/google
   $ git clone git@github.com:google/syzkaller.git
   $ cd syzkaller
   $ gmake all
   ```

## Compile Kernel

A `GENERIC` kernel must be compiled with
[kcov(4)](https://man.openbsd.org/kcov.4)
option enabled:

```sh
$ cd /sys
$ echo 'pseudo-device kcov 1' >arch/amd64/conf/KCOV
$ echo 'include "arch/amd64/conf/KCOV" >>arch/amd64/conf/GENERIC
$ make -C arch/amd64/compile/GENERIC config
$ make -C arch/amd64/compile/GENERIC
```

## Create VM

1. [vmd(8)](https://man.openbsd.org/vmd.8)
   must be configured to allow non-root users to create VMs since it removes the
   need to run syzkaller as root:

   ```sh
   $ cat /etc/vm.conf
   vm "syzkaller" {
     disable
     disk "${VMDIR}/syzkaller.img"
     local interface
     owner $USER
     allow instance { boot, disk, memory }
   }
   ```

2. Create disk image:

   ```sh
   $ vmctl create "${VMDIR}/syzkaller.img" -s 4G
   ```

3. Install VM:

   ```sh
   $ vmctl start syzkaller-1 -c -t syzkaller -b /bsd.rd -d "${VMDIR}/syzkaller.img"
   ```

   Answers to questions that deviates from the defaults:

   ```
   Password for root account? ******
   Which speed should com0 use? 115200
   Allow root ssh login? yes
   ```

4. Restart the newly created VM and copy the SSH-key:

   ```sh
   $ vmctl stop syzkaller-1 -w
   $ vmctl start syzkaller
   $ ssh "root@100.64.${VMID}.3" 'cat >~/.ssh/authorized_keys' <$SSHKEY
   $ vmctl stop syzkaller -w
   ```

## Configure and run syzkaller

```sh
$ pwd
~/go/src/github.com/google/syzkaller
$ cat openbsd.cfg
{
  "name": "openbsd",
  "target": "openbsd/amd64",
  "http": ":10000",
  "workdir": "$HOME/go/src/github.com/google/syzkaller/workdir",
  "kernel_obj": "/sys/arch/amd64/compile/GENERIC/obj",
  "kernel_src": "/",
  "syzkaller": "$HOME/go/src/github.com/google/syzkaller",
  "image": "$VMDIR/syzkaller.img",
  "sshkey": "$SSKEY",
  "sandbox": "none",
  "procs": 2,
  "type": "vmm",
  "vm": {
    "count": 4,
    "mem": 512,
    "kernel": "$KERNEL"
  }
}
$ ./bin/syz-manager -config openbsd.cfg
```
