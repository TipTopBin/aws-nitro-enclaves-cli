## Ubuntu 20.04 - How to install Nitro CLI from GitHub sources

The following steps are an option of how to install the Nitro CLI from the
GitHub sources on an Ubuntu 20.04 instance.

```sh
$ cat /etc/os-release
NAME="Ubuntu"
VERSION="20.04.2 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.2 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```

Check if the Nitro Enclaves kernel driver is included in the Ubuntu kernel. If
it isn't included, it will be built from the Nitro CLI GitHub sources using the
setup tooling (e.g. make install, nitro-cli-config) that is mentioned later.

Install the "linux-modules-extra-aws" package that includes extra modules for
AWS systems. Insert the Nitro Enclaves kernel driver.

Note: The path to the driver can be updated depending on the kernel version
installed on the system e.g. /usr/lib/modules/\<kernel-version\>/kernel/drivers.
Similarly for the kernel config e.g. /boot/config-\<kernel-version\>.

```sh
$ sudo apt-get install build-essential

$ uname -r
5.11.0-1016-aws

$ grep /boot/config-$(uname -r) -e NITRO_ENCLAVES
CONFIG_NITRO_ENCLAVES=m

$ sudo apt-get install linux-modules-extra-aws

$ ls -l /usr/lib/modules/$(uname -r)/kernel/drivers/virt/nitro_enclaves/nitro_enclaves.ko
-rw-r--r-- 1 root root 63825 Aug 12 04:21 /usr/lib/modules/5.11.0-1016-aws/kernel/drivers/virt/nitro_enclaves/nitro_enclaves.ko

$ sudo insmod /usr/lib/modules/$(uname -r)/kernel/drivers/virt/nitro_enclaves/nitro_enclaves.ko

$ lsmod | grep nitro_enclaves
nitro_enclaves         45056  0
```

The information about the Nitro Enclaves kernel driver availability can be found
at [Driver information](https://github.com/aws/aws-nitro-enclaves-cli#driver-information).

Setup the Docker dependency. You need to add to the docker group the non-root
user that is chosen for setting up enclaves, corresponding to your system design
and permissions.

```sh
$ sudo apt-get install docker.io

$ sudo systemctl start docker

$ sudo systemctl enable docker

$ sudo usermod -aG docker <username>
```

For the changes to take effect, log out of the instance and then reconnect to it.

Setup the Nitro CLI. The [nitro-cli-config](../bootstrap/nitro-cli-config)
script uses the current user for the setup. Update the user reference, if the
script is not run as the non-root user that is chosen for setting up enclaves.

```sh
$ git clone https://github.com/aws/aws-nitro-enclaves-cli.git

$ cd aws-nitro-enclaves-cli/
```

Part of the "nitro-cli-config" script:

```sh
# The current user.
THIS_USER="$(whoami)"
```

If the Nitro Enclaves kernel driver is included in the Ubuntu kernel, add the
following changes to not build it from the GitHub sources.

```sh
$ git diff bootstrap/nitro-cli-config
diff --git a/bootstrap/nitro-cli-config b/bootstrap/nitro-cli-config
index 35d424b..8099975 100755
--- a/bootstrap/nitro-cli-config
+++ b/bootstrap/nitro-cli-config
@@ -450,19 +450,6 @@ function driver_insert {
     local log_file="/var/log/$RES_DIR_NAME/nitro_enclaves.log"
     local loop_idx=0

-    # Remove an older driver if it is inserted.
-    if [ "$(lsmod | grep -cw $DRIVER_NAME)" -gt 0 ]; then
-        driver_remove
-    fi
-
-    print "Inserting the driver..."
-
-    # Insert the new driver.
-    sudo_run "insmod $DRIVER_NAME.ko" || fail "Failed to insert driver."
-
-    # Verify that the new driver has been inserted.
-    [ "$(lsmod | grep -cw $DRIVER_NAME)" -eq 1 ] || fail "The driver is not visible."
-
     print "Configuring the device file..."

     # Create the NE group if it doesn't already exist.
```

```sh
$ git diff bootstrap/env.sh
diff --git a/bootstrap/env.sh b/bootstrap/env.sh
index 1ebcabd..9df6331 100755
--- a/bootstrap/env.sh
+++ b/bootstrap/env.sh
@@ -9,8 +9,5 @@ then
     return -1
 fi

-lsmod | grep -q nitro_enclaves || \
-    sudo insmod ${NITRO_CLI_INSTALL_DIR}/lib/modules/extra/nitro_enclaves/nitro_enclaves.ko
-
 export PATH=${PATH}:${NITRO_CLI_INSTALL_DIR}/usr/bin/:${NITRO_CLI_INSTALL_DIR}/etc/profile.d/
 export NITRO_CLI_BLOBS=${NITRO_CLI_INSTALL_DIR}/usr/share/nitro_enclaves/blobs
```

```sh
$ git diff Makefile
diff --git a/Makefile b/Makefile
index dff654c..76f3b1a 100644
--- a/Makefile
+++ b/Makefile
@@ -318,10 +318,7 @@ install-tools:
        $(CP) -r examples/${HOST_MACHINE}/* ${NITRO_CLI_INSTALL_DIR}${DATA_DIR}/nitro_enclaves/examples/

 .PHONY: install
-install: install-tools nitro_enclaves
-       $(MKDIR) -p ${NITRO_CLI_INSTALL_DIR}/lib/modules/$(uname -r)/extra/nitro_enclaves
-       $(INSTALL) -D -m 0755 drivers/virt/nitro_enclaves/nitro_enclaves.ko \
-               ${NITRO_CLI_INSTALL_DIR}/lib/modules/$(uname -r)/extra/nitro_enclaves/nitro_enclaves.ko
+install: install-tools
        $(INSTALL) -D -m 0644 bootstrap/env.sh ${NITRO_CLI_INSTALL_DIR}${ENV_SETUP_DIR}/nitro-cli-env.sh
        $(INSTALL) -D -m 0755 bootstrap/nitro-cli-config ${NITRO_CLI_INSTALL_DIR}${ENV_SETUP_DIR}/nitro-cli-config
        sed -i "2 a NITRO_CLI_INSTALL_DIR=$$(readlink -f ${NITRO_CLI_INSTALL_DIR})" \

```

Continue the setup for the Nitro CLI.

```sh
$ export NITRO_CLI_INSTALL_DIR=/

$ make nitro-cli

$ make vsock-proxy

$ sudo make NITRO_CLI_INSTALL_DIR=/ install

$ source /etc/profile.d/nitro-cli-env.sh

$ echo source /etc/profile.d/nitro-cli-env.sh >> ~/.bashrc

$ nitro-cli-config -i
```

For the changes to take effect, log out of the instance and then reconnect to it.

Start the Nitro Enclaves allocator service.

```sh
$ cat /etc/nitro_enclaves/allocator.yaml
---
# Enclave configuration file.
#
# How much memory to allocate for enclaves (in MiB).
memory_mib: 512
#
# How many CPUs to reserve for enclaves.
cpu_count: 2
#
# Alternatively, the exact CPUs to be reserved for the enclave can be explicitly
# configured by using `cpu_pool` (like below), instead of `cpu_count`.
# Note: cpu_count and cpu_pool conflict with each other. Only use exactly one of them.
# Example of reserving CPUs 2, 3, and 6 through 9:
# cpu_pool: 2,3,6-9

$ sudo systemctl start nitro-enclaves-allocator.service

$ sudo systemctl enable nitro-enclaves-allocator.service
Created symlink /etc/systemd/system/multi-user.target.wants/nitro-enclaves-allocator.service → /lib/systemd/system/nitro-enclaves-allocator.service.
```

As a sanity check for the Nitro CLI setup, run the hello world enclave.

https://docs.aws.amazon.com/enclaves/latest/user/getting-started.html

```sh
$ nitro-cli build-enclave --docker-dir /usr/share/nitro_enclaves/examples/hello --docker-uri hello:latest --output-file hello.eif

$ nitro-cli run-enclave --cpu-count 2 --memory 512 --enclave-cid 16 --eif-path hello.eif --debug-mode
```
