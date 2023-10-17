# `amd64-gentoo-hpc-cockpit-install.sh` Script

The `amd64-gentoo-hpc-cockpit-install.sh` script installs Cockpit and Performance Co-Pilot (PCP) from source on an ~amd64 systemd init driven Gentoo Linux machine. The script also configures PAM login authentication to properly address problems for Cockpit Web Console login, which has been a major stopping point for source based linux distro users. 

## Background

The lack of a user-friendly interface for managing the HPC cluster can be a significant barrier for users who are not familiar with the command-line interface. This can lead to inefficiencies and errors in managing the cluster, which can impact performance and productivity.

The CIO requested some type of visual oversight of the cluster to be able to monitor power consumption/ power state of the HPC cluster nodes. 

## Discussion - Developer Perspective

There is no official supported ebuild on Gentoo for Cockpit due to several problems. One of the main issues is that Cockpit is a web-based graphical interface for managing Linux servers, which goes against the Gentoo philosophy of minimalism, high degree of performance, and customization. Additionally, Cockpit has many dependencies, including systemd, which is not supported by Gentoo. 

Another issue is that Cockpit is not fully compatible with Gentoo's source-based distribution model. Cockpit is designed to work with binary-based distributions like Red Hat Enterprise Linux, which have pre-built packages available in their repositories. Gentoo, on the other hand, relies on users building packages from source, which can be time-consuming and error-prone.

As a result, there is no official supported ebuild for Cockpit on Gentoo. However, users can still install Cockpit from source using the `amd64-gentoo-hpc-cockpit-install.sh` script, which installs Cockpit and Performance Co-Pilot (PCP) from source and configures PAM to allow login to Cockpit.

I will be submitting this as an ebuild for Gentoo Linux once refined. 

## Dependencies

Cockpit requires the following dependencies during the build from source:

- `net-libs/libssh`

PCP requires the following dependencies during the build from source:

- `dev-vcs/git`
- `dev-qt/qtprintsupport`

## Installation

To install Cockpit and PCP from source on Gentoo Linux, follow these steps:

1. Install the necessary dependencies using `sudo emerge --ask <dependency>`.
2. Clone the PCP source code using `git clone https://github.com/performancecopilot/pcp.git`.
3. Navigate to the PCP source code directory using `cd pcp`.
4. Run `./Makepkgs --verbose` to build PCP from source.
5. Fix the `QtPrintSupport` error by running `sudo emerge -av dev-qt/qtprintsupport`.
6. Create the `pcp` user and group by running `sudo groupadd -r pcp` and `sudo useradd -c "Performance Co-Pilot" -g pcp -d /var/lib/pcp -M -r -s /usr/sbin/nologin pcp`.
7. Continue with the PCP installation by running `./configure`, `make -j8`, and `make -j8 install`.
8. Configure PAM for Cockpit by adding the necessary lines to the `/etc/pam.d/cockpit` file.
9. Restart the Cockpit service using `sudo systemctl restart cockpit.service`.
10. Check the status of the Cockpit service using `sudo systemctl status cockpit.service`.

By following these steps, users can install Cockpit and PCP from source on Gentoo Linux, and configure PAM to allow login to Cockpit.
![Cockpit](https://github.com/alexander-labarge/hpc-optimizations/assets/103531175/0c8450c6-ddb1-4ec7-81b1-0df25493d9df)

# `gentoo-apptainer-offline-mirror.sh` Script
## Demonstration of Apptainer High Performance Computing (HPC) Containerization of Gentoo Linux Source Files

**Author:** Alexander M. La Barge <br>
**Date:** 17 Oct 23

This demonstration showcases the use of Apptainer, a container platform, to create and run containers that package up pieces of software in a way that is portable and reproducible. The specific use case demonstrated is the creation of a Gentoo RSYNC Source Mirror from the Massachusetts Institute of Technology (MIT) for offline distribution of necessary build packages for HPC Cluster in an offline networked environment.

### Requirements

- Apptainer: https://apptainer.org/
- Docker (optional): Apptainer directly fetches from docker registry/hub.
- A Linux system to run Apptainer natively. It's easiest to install if you have root access.

### Steps

1. Convert a base Gentoo stage3 image from Docker to Singularity image format (.sif).
2. Build a container using the converted image.
3. Launch a writable shell with persistent storage.
4. Execute the following commands as root in the container:
   - `emerge --sync`
   - `emerge net-misc/rsync`
   - `emerge bash`
   - `export PS1="\033[1;33mapptainer-root # \w $ \033[0m"`
   - `mkdir -p /mnt/gentoo-source`
   - `mkdir -p /mnt/gentoo-portage`
   - `rsync -av --delete --progress --info=progress2 rsync://mirrors.mit.edu/gentoo-distfiles/ /mnt/gentoo-source`
   - `cp -r /var/db/repos/gentoo/* /mnt/gentoo-portage`
5. Set up an RSYNC server in the container to serve files for the offline networked Gentoo mirror:
   - `emerge rsync`
   - Configure RSYNC server settings in `/etc/rsyncd.conf`.

        ```bash
      cat > /etc/rsyncd.conf <<EOL
      pid file = /var/run/rsyncd.pid
      lock file = /var/run/rsync.lock
      log file = /var/log/rsync.log
      use chroot = yes
      read only = yes
      list = yes
      uid = nobody
      gid = nobody
      max connections = 10
      timeout = 300
      
      [gentoo-portage]
          path = /mnt/gentoo-portage
          comment = Gentoo Portage Tree
      
      [gentoo-source]
          path = /mnt/gentoo-source
          comment = Gentoo Source Files
      EOL
      ```
6. Perform a final sync of the Gentoo Portage Tree and Gentoo Source Files.
7. Test Run the RSYNC server inside the container.
8. Test the RSYNC server from outside the container.

   ![image](https://github.com/alexander-labarge/hpc-developement/assets/103531175/d1b3c278-7f4a-417a-9042-dde6c829f1ae)


### Security

By default, the Apptainer containers are user namespaces, so inside the container, you appear as root, but on the host, you're still your regular user. This means you can modify files owned by your user in the sandbox from inside the container, but you won't be able to modify system files unless you've elevated permissions on the host (not typically recommended due to potential security risks).

### Final Storage Footprint

The final storage footprint inside the container (dev/sda mounted to / in container) will depend on the size of the Gentoo Portage Tree and Gentoo Source Files.

![image](https://github.com/alexander-labarge/hpc-developement/assets/103531175/f7b19868-54f9-48c6-8f6f-a2e3b7e9b64b)

