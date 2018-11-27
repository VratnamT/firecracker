# Getting Started with Firecracker on AWS

[Getting Started with Firecracker](https://github.com/firecracker-microvm/firecracker/blob/master/docs/getting-started.md) provides detailed instructions on how to download the Firecracker binary, start Firecracker with different options, build from the source, and run integration tests. You can run Firecracker in production using the [Firecracker Jailer](https://github.com/firecracker-microvm/firecracker/blob/master/docs/jailer.md). 

Let's take a look at how to get started with using Firecracker on AWS Cloud (these steps can be used on any bare metal machine):

* Create an `i3.metal` instance using Ubuntu 18.04.1.

* Firecracker is built on top of KVM and needs read/write access to `/dev/kvm`. Log in to the host in one terminal and set up that access:
    
    ```
    sudo chmod 777 /dev/kvm
    ```

* Download and start the Firecracker binary:
    
    ```
    curl -L https://github.com/firecracker-microvm/firecracker/releases/download/v0.11.0/firecracker-v0.11.0
    ./firecracker-v0.11.0 --api-sock /tmp/firecracker.sock
    ```

* Each microVM can be managed using a [REST API](https://github.com/firecracker-microvm/firecracker/blob/master/api_server/swagger/firecracker.yaml). In another terminal, query the microVM:
    
    ```
    curl --unix-socket /tmp/firecracker.sock "http://localhost/machine-config"
    ```
    
    This returns a response:
    
    ```
    { "vcpu_count": 1, "mem_size_mib": 128,  "ht_enabled": false,  "cpu_template": "Uninitialized" }
    ```
    
    This starts a VMM process and waits for the microVM configuration. By default, one vCPU and 128 MiB memory are assigned to each microVM. Now this microVM needs to be configured with an uncompressed Linux kernel binary and an ext4 file system image to be used as root filesystem.

* Download a sample kernel and rootfs:
    
    ```
    curl -fsSL -o hello-vmlinux.bin https://s3.amazonaws.com/spec.ccfc.min/img/hello/kernel/hello-vmlinux.bin
    curl -fsSL -o hello-rootfs.ext4 https://s3.amazonaws.com/spec.ccfc.min/img/hello/fsfiles/hello-rootfs.ext4
    ```

* Set up the guest kernel:
    
    ```
    curl --unix-socket /tmp/firecracker.sock -i \
        -X PUT 'http://localhost/boot-source'   \
        -H 'Accept: application/json'           \
        -H 'Content-Type: application/json'     \
        -d '{        "kernel_image_path": "./hello-vmlinux.bin",        "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"    }'
    ```

* Set up the root filesystem:
    
    ```
    curl --unix-socket /tmp/firecracker.sock -i \
        -X PUT 'http://localhost/drives/rootfs' \
        -H 'Accept: application/json'           \
        -H 'Content-Type: application/json'     \
        -d '{        "drive_id": "rootfs",        "path_on_host": "./hello-rootfs.ext4",        "is_root_device": true,        "is_read_only": false    }'
    ```

* Once the kernel and root filesystem are configured, the guest machine can be started:
    
    ```
    curl --unix-socket /tmp/firecracker.sock -i \
        -X PUT 'http://localhost/actions'       \
        -H  'Accept: application/json'          \
        -H  'Content-Type: application/json'    \
        -d '{        "action_type": "InstanceStart"     }'
    ```

* The first terminal now shows a serial TTY prompting you to log in to the guest machine:
    
    ```
    Welcome to Alpine Linux 3.8
    Kernel 4.14.55-84.37.amzn2.x86_64 on an x86_64 (ttyS0)
    
    localhost login: 
    ```

* Login as `root` with password `root` to see the terminal of the guest machine:
    
    ```
    localhost login: root
    Password: 
    Welcome to Alpine!
    
    The Alpine Wiki contains a large amount of how-to guides and general
    information about administrating Alpine systems.
    See <http://wiki.alpinelinux.org>.
    
    You can setup the system with the command: setup-alpine
    
    You may change this message by editing /etc/motd.
    
    login[979]: root login on 'ttyS0'
    localhost:~#
    ```

* You can see the filesystem using ls /
    
    ```
    localhost:~# ls /
    bin         home        media       root        srv         usr
    dev         lib         mnt         run         sys         var
    etc         lost+found  proc        sbin        tmp
    ```

* Terminate the microVM using the reboot command. Firecracker currently does not implement guest power management, as a tradeoff for efficiency. Instead, the reboot command issues a keyboard reset action which is then used as a shutdown switch.
    
Once the basic microVM is created, you can add network interfaces, add more drives, and continue to configure the microVM.

Want to create thousands of microVMs on your bare metal instance?

```
for ((i=0; i<1000; i++)); do
   sudo ./firecracker-v0.10.1 --api-sock /tmp/firecracker-$i.sock &
done
```

Multiple microVMs may be configured with a single shared root file system, and each microVM can then be assigned its own read/write share.
