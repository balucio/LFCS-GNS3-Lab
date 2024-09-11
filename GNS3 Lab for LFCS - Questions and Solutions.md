# LFCS GNS3 Lab: Questions and Solutions

 [Back to main](GNS3%20Lab%20for%20LFCS%20-%20Overview%20and%20Guidelines.md)

> [!CAUTION]
> This document includes both questions and their corresponding detailed answers.

## 1. Users and Groups: User Management

On **Client Workstation (`client`)**, create a new user named `testuser`. Set `password` as password and add the user to the `sudo` group. Create a new group named `nfs-read` and assign the user `testuser` to this group.

<details>
  <summary>Answer</summary>

#### 1. Create the `nfs-read` group

```bash
sudo addgroup nfs-read

# Output
info: Selecting GID from range 1000 to 59999 ...
info: Adding group `nfs-read' (GID 1001) ...
```

#### 2. Create `testuser` and add it to `nfs-read` and `sudo` groups
```bash
sudo adduser testuser
# Output omitted for brevity
new password: [password]
Retype new password: [password]
passwd: password updated successfully
info: Adding user `testuser' to group `users' ...

# Adding testuser to nfs-read
sudo usermod -aG "nfs-read" testuser 
# Adding testuser to sudoers
sudo usermod -aG "sudo" testuser 
```

#### 3. Check
```bash
sudo su - testuser

# Output
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
testuser@client:~$
  
id

# Output
uid=1002(testuser) gid=1002(testuser) groups=1002(testuser),27(sudo),100(users),1001(nfs-read)
sudo su -
[sudo] password for testuser: 
root@client:~# 
```

</details>


## 2. Networking: IPv4/IPv6 Configuration

Configure **Router (`router`)** to route traffic between the Management and Application subnets. Ensure that devices on both subnets can reach each other.

<details>
  <summary>Answer</summary>

#### 1. Enabling forwarding on `router` changing **Kernel** parameters:

```bash
# IPv4
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.d/10-enable-forwarding.conf 2>/dev/null
echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.d/10-enable-forwarding.conf 2>/dev/null
echo "net.ipv4.conf.default.forwarding=1" | sudo tee -a /etc/sysctl.d/10-enable-forwarding.conf 2>/dev/null

# IPv6
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.d/10-enable-forwarding.conf 2>/dev/null
echo "net.ipv6.conf.default.forwarding=1" | sudo tee -a /etc/sysctl.d/10-enable-forwarding.conf 2>/dev/null

# Apply changes
sysctl -p /etc/sysctl.d/10-enable-forwarding.conf
```

#### 2. Check
Check if `ping` works for example from `client` and the `nfs-server`

</details>

## 3. Networking: Packet Filtering and NAT

Configure the **Router (`router`)** system to:

1. Apply NAT for all outgoing traffic towards the Internet.
2. Enable forwarding between `eth0` (Management) and `eth1` (Application) subnets.
3. Allow the external interface (`eth2`) to obtain IPv4 and IPv6 configuration from an external DHCP server.
4. Allow hosts in the Management and Application subnets to query external DNS.

> [!TIP]
> To perform this task, you can use `iptables`, `firewalld`, or another firewall application that you are familiar with. Try to avoid mixing different solutions.

<details>
  <summary>Answer</summary>

To answer this task, we will use **firewalld** due to its simplicity. We'll utilize two predefined *firewalld* zones: `public` for handling Internet traffic, and `internal` for managing Application and Management subnet traffic.

<br><br>

> The initial configuration of firewalld can vary slightly between distributions. To ensure the final configuration meets the requirements of this task, some commands may appear redundant, and firewalld might generate warnings indicating that certain configurations are already in place.

#### 1. **Install and enable `firewalld`**

```
# Install firewalld
dnf install -y firewalld

# Enable and start the firewalld service
systemctl enable firewalld
systemctl start firewalld
```

#### 2. **Configure NAT (Masquerading)**

```
# Optional: ensure eth2 is in the public zone
sudo firewall-cmd --zone=public --add-interface=eth2
sudo firewall-cmd --zone=public --add-interface=eth2 --permanent

# Add masquerading (NAT) for the current session
sudo firewall-cmd --zone=public --add-masquerade

# Make masquerading permanent
sudo firewall-cmd --zone=public --add-masquerade --permanent
```

#### 3. **Enable forwarding between `eth0` and `eth1`**

<br><br>

> Ensure that Task **1: Networking - IPv4/IPv6 Configuration** is completed correctly, as proper Kernel level forwarding configuration is essential for firewalld zone forwarding to work effectively.

We use the **firewalld** `internal` zone to handle traffic for the Application and Management subnets.

```
# Assign eth0 and eth1 to the internal zone
sudo firewall-cmd --zone=internal --add-interface=eth0 --permanent  
sudo firewall-cmd --zone=internal --add-interface=eth1 --permanent

# Enable interface forwarding for the current session
sudo firewall-cmd --zone=internal --add-forward

# Enable interface forwarding permanently
sudo firewall-cmd --zone=internal --add-forward --permanent
```

#### 4. **Allow incoming DHCP and DNS traffic**

```
# Allow eth2 to obtain a DHCP configuration for the current session
sudo firewall-cmd --zone=public --add-service=dhcp
sudo firewall-cmd --zone=public --add-service=dhcpv6-client

# Allow eth2 to obtain a DHCP configuration permanently
sudo firewall-cmd --zone=public --add-service=dhcp --permanent
sudo firewall-cmd --zone=public --add-service=dhcpv6-client --permanent
```

#### 5. **Allow internal hosts to query external DNS**

Since NAT is active on `eth2`, DNS queries should work by default. To verify this, try querying an external DNS from all internal servers:

```
dig www.google.it @8.8.8.8
```

#### 6. Final steps and verifications

- **Save configuration and reload firewalld rules**
  ```
  # Save the configuration permanently
  sudo firewall-cmd --runtime-to-permanent 
  
  # Reload firewalld rules
  sudo firewall-cmd --reload 
  ```

- **Checking configuration**
  - All internal system have now Internet connection
  - All internal system can ping each others
  - Issuing a `sudo dhclient eth2` should works without errors
  - All internal systems can use an external DNS like for example `8.8.8.8`  

</details>

## 4. Security: Reconnaissance Attack

As an ace hacker, use the **Attacker (`attacker`)** system to perform a reconnaissance attack on the router's `eth2` interface. Your objective is to identify which services are exposed on the router. Report any insecure services.

<details>
  <summary>Answer</summary>

To complete this task, we will use `nmap` to perform a network scan from the **Attacker (`attacker`)** system and identify any insecure services exposed by the **Router (`system`)**.

#### 1. Perform a scan to identify the target

This is an optional step if you already know the IP address of the router's `eth2` interface):

```bash
# Get the IP address of the eth0 interface
ip addr show

# Output omitted for brevity
inet 192.168.122.116/24 scope global eth0
  
# Perform a scan on the subnet
nmap -sn 192.168.122.0/24

# Output omitted for brevity
Nmap scan report for router (192.168.122.241)
Host is up (0.0018s latency).
MAC Address: 0C:E7:0A:4A:00:02 (Unknown)
```

#### 2. Perform a full port scan on target

Now we will use `nmap` to scan the IP address of the `router` system**. The `nmap` command offers various options to improve the scan making it more reliable, but here’s a basic command:

```bash
# Bare minimum port scan
nmap -p- 192.168.122.241
 
# Output omitted for brevity
Nmap scan report for router (192.168.122.241)
PORT     STATE SERVICE
22/tcp   open  ssh
9090/tcp open  zeus-admin
MAC Address: 0C:E7:0A:4A:00:02 (Unknown)
```

#### 3. Attack Results:
- Port `22` (ssh) is open.
- Port `9090` is open - identified as `zeus-admin` service enabled on the `router`.

</details>

## 5. Security: Router Hardening

Following a security audit, a critical vulnerability was discovered on the **Router (`router`)**: an unsecured service is publicly exposed on the internet. As a white hat engineer, your task is to identify the service and port, and secure it from external access while maintaining availability for internal users. Additionally, ensure that by default, no external traffic is allowed into the `router`, except for the `ssh` service.

<details>
  <summary>Answer</summary>

To complete this task, we will use the already configured `firewalld` on the **Router (`router`)** system.

#### 1. Identify the service listening on port `9090`

On Rocky Linux, the `netstat` command is deprecated, so we'll use `ss` instead:

```
sudo ss -lptn 'sport = :9090'

# Output
LISTEN 0      128                *:9090            *:*    users:(("systemd",pid=1,fd=38))
```

The `ss` command tells us it's a `systemd` socket. To identify the socket name:

```
# List all systemd sockets
sudo systemctl list-sockets | grep 9090

# Output
[::]:9090                         cockpit.socket                  cockpit.service
```

The `cockpit` service is listening on port `9090`.

#### 2. Block incoming traffic on port `9090`

We can simply remove the `cockpit` service from the `public` zone:

```
# Permanently remove cockpit service from the public zone
sudo firewall-cmd --permanent --zone=public --remove-service=cockpit 
# Output: success

# Remove cockpit service from the public zone at runtime
sudo firewall-cmd --zone=public --remove-service=cockpit 
# Output: success
```

#### 3. Allow SSH and deny all other traffic on `eth2`

- **Enable SSH**: SSH should already be allowed in the `public` zone, but you can double-check:

  ```
  # Allow SSH on eth2 for the current session
  sudo firewall-cmd --zone=public --add-service=ssh
  
  # Allow SSH on eth2 permanently
  sudo firewall-cmd --zone=public --add-service=ssh --permanent
  ```

- **Deny all other traffic**: 
  Set `DROP` as the default behavior for the `public` zone. Alternatively, the `REJECT` policy is also valid but provides feedback to the sender.

  ```bash
  sudo firewall-cmd --set-target=DROP --permanent
  ```

- **Save configuration and reload firewalld rules**:

  ```bash
  # Save the configuration permanently
  sudo firewall-cmd --runtime-to-permanent 
  
  # Reload firewalld rules
  sudo firewall-cmd --reload 
  ```

#### 4. Final check

- **Check the router's configuration**:

  ```
  sudo firewall-cmd --list-all

  # Output for public and internal zones

  internal (active)
    target: default
    interfaces: eth0 eth1
    services: cockpit dhcpv6-client mdns samba-client ssh
    forward: yes
    masquerade: no

  public (active)
    target: DROP
    interfaces: eth2
    services: dhcp dhcpv6-client ssh
    forward: no
    masquerade: yes
  ```

- **Perform an `nmap` scan from the `attacker` system to ensure the issue was mitigated**:

    ```
    nmap -sS -sV -O -p- 192.168.122.241

    # Output
    Nmap scan report for router (192.168.122.241)
    Host is up (0.0016s latency).
    Not shown: 65534 filtered tcp ports (no-response)
    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 8.0 (protocol 2.0)
    MAC Address: 0C:E7:0A:4A:00:02 (Unknown)
    ```
</details>

## 6. Networking: SSH Configuration

On **Web and Application Server (`webapp-services`)**, configure SSH to allow key-based authentication only and disable root login.

<details>
  <summary>Answer</summary>

#### 1. **Remove Custom Configuration**

Since custom SSH configurations in `/etc/ssh/sshd_config.d/` may have higher priority, ensure these are removed or adjusted as needed.

```bash
rm -f /etc/ssh/sshd_config.d/*
```

#### 2. **Update the Main SSH Configuration**

Edit `/etc/ssh/sshd_config` to ensure it allows only *key-based authentication* and disables *root login*. The `sshd_config` file should look like this:

```ssh
Include /etc/ssh/sshd_config.d/*.conf
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
```

#### 3. **Restart `sshd` Service**

Apply the new SSH configuration by restarting the SSH service:

```bash
systemctl restart sshd
```

#### 4. **Verify Configuration**

To verify that the configuration is working correctly, perform the following steps:

- **Test SSH Access Without Key**: From the `infra-services` or the `client`, try to connect via `ssh` with the `rocky` user to the `webapp-services`, ensuring access is denied.

  ```bash
  ssh ubuntu@192.168.20.20
  ubuntu@192.168.20.20: Permission denied (publickey).
  ```

- **Generate and Deploy SSH Key**: Use `ssh-keygen` to generate the key-pair on `infra-services` or the `client`. The command will create the key-pair in the user's `.ssh` directory.

- **Copy Public Key**: Append the generated public key `id_rsa.pub` into the `authorized_keys` file of the `ubuntu` user on `webapp-services`.

- **Test**: The command `ssh ubuntu@192.168.20.20` should work without a password if everything is configured correctly.

</details>

## 7. Operations Deployment: Kernel Parameters Configuration

Modify kernel parameters on **Management Server (`infra-services`)** to increase the maximum number of open files. Ensure this change is persistent across reboots.

<details>
  <summary>Answer</summary>

#### 1. **Change `fs.file-max`**

The `fs.file-max` kernel parameter controls the maximum number of file descriptors that can be allocated by the kernel. To make this change persistent, create a configuration file in `/etc/sysctl.d/`, for example, `10-max-file.conf`.

```bash
# Persistent configuration
echo "fs.file-max=100000" | sudo tee /etc/sysctl.d/10-max-file.conf 2>/dev/null

# Apply the change at runtime
sysctl -w fs.file-max=100000
```

#### 2. **Check the Value**

You can verify that the new value is correctly applied by checking `/proc/sys/fs/file-max`:

```bash
cat /proc/sys/fs/file-max

# Expected output
100000
```

</details>

## 8. Operations Deployment: Process Management

On **Web and Application Server (`webapp-services`)**, start a web service (e.g., `nginx`), monitor its process, and configure it to restart automatically if it crashes.

<details>
  <summary>Answer</summary>

#### 1. **Install and Enable NGINX**:

```bash
# Install NGINX
sudo apt install -y nginx

# Enable NGINX to start on boot
sudo systemctl enable nginx

# Start the NGINX service
sudo systemctl start nginx
```

#### 2. **Configure Process Monitoring and Automatic Restart**

The most straightforward and recommended way to automatically restart a service (NGINX), if it crashes is to create a `systemd` override file.

- **Create a Systemd Override for NGINX**
  see `man systemd.service` for help
    ```bash
    # Create or edit an override file for the NGINX service
    sudo systemctl edit nginx
    ```
    This will open a text editor. Add the following content to configure automatic restarts:
    ```bash
    [Service]
    # Restart the service on failure
    Restart=on-failure
  
    # Optional: attempt to restart up to 5 times within a 10-second window before giving up
    RestartSec=5s
    ```

- **Reload daemon**
  Reload the systemd daemon to apply the changes:
    ```bash
    sudo systemctl daemon-reload
    ```
- **Restart nginx**: Finally restart `nginx` to ensure new configuration is in effect
    ```bash
    sudo systemctl restart nginx
    ```

#### 3. **Test configuration**
Simulate a crash to check that the NGINX service restarts automatically and test access

- **Simulate a crash**: 
    ```bash
    sudo killall -9 nginx
    ```
- **Check the Status**:
    ```bash
    sudo systemctl status nginx
    ```

- **Test access**:
  From the `client` system, open a web browser and navigate to [http://192.168.20.20](http://192.168.20.20). You should see the NGINX welcome page.

</details>

## 9. Operations Deployment: Software Management

On the **Client Workstation (`client`)**, install the `htop` package from a remote repository and verify the installation.

<details>
  <summary>Answer</summary>

####  1. **Install `htop`**:

The `htop` software can be found on the standard Ubuntu repository, so it is possible to install it issuing:
      
```bash
# Install htop
sudo apt install -y htop
```

#### 2. **Verify installation**:

```bash
# check htop
htop
```

</details>

## 10. Operations Deployment: Container Management

On the **NFS/Backup Server (`nfs-server`)**, install Docker, create a container running an `nginx` web server, and verify its operation. The Docker-CE repository can be installed from this url:
- [https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo).

Eventually, you can use documentation at `/usr/share/doc/docker-compose-plugin` to create a working `docker-compose.yml` configuration.

<details>
  <summary>Answer</summary>

#### 1. **Configure Docker-CE repository**

To add the Docker-CE repository on the System the most straightforward way is to use `dnf` command:

```bash
# Download and install Docker-CE repository
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### 2. **Install Docker-CE**:

```bash
 # Install Docker-CE from repository
 sudo dnf -y install docker-ce

# Optionally install related tools if they are not installed as dependencies
 sudo dnf -y install docker-ce-cli containerd.io docker-compose-plugin
```

#### 3. **Configuring Docker-CE**

The following commands enable Docker-CE to start on boot, start the service, and check the daemon status. Additionally, the current `rocky` user is added to the `docker` group, allowing Docker operations without the `sudo` command. After adding the user to the group, it’s advisable to re-login to update the user group membership.

```bash
# Enable Docker to start on boot
sudo systemctl enable docker

# Start Docker service
sudo systemctl start docker

# Check Docker status
sudo systemctl status docker

# Add Rocky user to the Docker group
sudo usermod -aG docker rocky

# Log out to apply group membership changes
logout
```

#### 4. **Setting up a docker-compose project**

While you can directly use the `docker` command to handle the container, it's advisable to set up a simple *compose* project, and create the relative `docker-compose.yml` file:

- **Setup the project directory and pull the image**:
    ```bash
    cd
    mkdir nginx
    cd nginx
    ```
  **Note**: pulling the latest version is not always a good idea; on production systems, it is advisable to use a specific version tag.
    ```bash
    docker pull nginx
    ```

- **Creating the `docker-compose.yml` file**
  A compose file allows better management of the `nginx` service. Whether you choose to use `docker` commands directly or use *compose*, the container must be configured to start when the system boots. You can use documentation in `/usr/share/doc/docker-compose-plugin` for references. The `docker-compose.yml` should look like this:
     ```yml
      services:
        web:
          image: nginx
          hostname: nginx
          container_name: nginx
          restart: always
          ports:
            - "80:80"
            - "443:443"
          networks:
            - nginx
      networks:
         nginx:
           name: nginx
     ```
  
- **Checking the compose project**: use the following command to check the *compose* file:
     ```bash
     docker compose config
     ```
  If the syntax is correct, it will display the full compose configuration.
  
- **Start the project**: Use `-d` flag to put the container in the background
     ```bash
     docker compose up -d
     ```

- **Test the Restart Policy**
    ```bash
    docker inspect -f "{{ .HostConfig.RestartPolicy.Name }}" nginx
    
    # Expected output
    always
    ```

- **Test docker NGINX access**:
  From the **Client Workstation (`client`)**, open a web browser and navigate to http://192.168.20.30. If everything is working correctly, you should see the NGINX welcome page.

</details>

## 11. Networking: Load Balancing

Configure the **Management Server (`infra-services`)** to act as a load balancer for HTTP traffic between the **NFS/Backup Server (`nfs-server`)** and the **Web and Application Server (`webapp-services`)**. Use the documentation in /usr/share/doc/haproxy/ as a reference to create a working HAProxy configuration.

<details>
  <summary>Answer</summary>

On Linux systems, one of the most commonly used solutions for load balancing is HAProxy. We will install and configure this application to balance HTTP requests between the two servers. Other solutions include *NGINX* and the *Apache HTTP Server*, both of which can also function as proxy servers and balance traffic over different nodes.

####  1. **Install HAProxy**:
The `haproxy` package is available in the default Rocky Linux repositories, making installation straightforward::

```bash
sudo dnf install haproxy
```
####  2. **HAProxy Initial setup**
The HAProxy should be configured to automatically starts when system boot.

- **Enable HAProxy on boot and check the default config**
    ```bash
    sudo systemctl enable haproxy
    sudo systemctl start haproxy
    sudo systemctl status haproxy
    ```

#### 3. **Configure HAProxy to load-balance HTTP requests**
By default, *HAProxy* on Rocky Linux comes with a demonstration configuration that binds to port `5000`. In a production environment, this configuration should be removed, but for this exercise, we can leave it as is and create an additional configuration in the `/etc/haproxy/conf.d` directory, such as `/etc/haproxy/conf.d/webapp.cfg`:

- **Create HAProxy configuration**:

    ```haproxy
    frontend webappfe
      bind  :80
      default_backend webappbk

    backend webappbk
      balance roundrobin
                server webapp-services 192.168.20.20:80
                server nfs-server      192.168.20.30:80
    ```

- **Test HAProxy configuration**: To test the configuration, ensure that all relevant configuration files are included with the `-f` option, and use the `-c` flag to check the configuration:

    ```haproxy
            haproxy -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/conf.d/webapp.cfg  -c
    ```

- **Restart HAProxy**: Restart is needed to apply the configuration changes:
    ```bash
    sudo systemctl restart haproxy
    
    ```
#### 4. **Test HAProxy**
From the **Client Workstation (`client`)**, open a web browser and navigate to `http://192.168.10.10` or `http://192.168.20.10`. If everything is configured correctly, you should see the NGINX welcome page.

</details>

## 12. Storage: LVM Management

On **Management Server (`infra-services`)**, create a new logical volume using the `30 GB` disk (`/dev/vdb`) and mount it to `/mnt/storage` with the `ext4` filesystem. Configure it to mount automatically at boot.

<details>
  <summary>Answer</summary>

#### 1. **Install the LVM2 package**
Install the `lvm2` package on Rocky Linux, which provides the *device-mapper* kernel module necessary for handling LVM2 volumes:

```bash
sudo dnf install lvm2
```

#### 2. **Verify the Kernel module is loaded**
The device-mapper module (`dm_mod`) should already be active in the kernel. Verify this using:

```bash
# Check with lsmod
lsmod | grep dm_mod

# Check module info (optional)
modinfo dm_mod
```

#### 3. **Create the Logical Volume**
Before creating a *logical volume*, we need to initialize the disk as an LVM physical volume and create an LVM volume group.

- **Check the device name**: Confirm the correct device name using `fdisk`:
  
```bash
sudo fdisk -l

# Output (truncated for brevity):
Disk /dev/vdb: 30.3 GiB, 32505856000 bytes, 63488000 sectors
```

- **Create the LVM Physical Volume**: Use `pvcreate` to initialize `/dev/vdb` as an LVM physical volume:
  
```bash
sudo pvcreate /dev/vdb

# Output:
Physical volume "/dev/vdb" successfully created.
```

- **Create the LVM Volume Group**: Use `vgcreate` to create a volume group (e.g., `infra_vg`):
  
```bash
sudo vgcreate infra_vg /dev/vdb

# Output:
Volume group "infra_vg" successfully created.
```

- **Create the LVM Logical Volume**: Use `lvcreate` to create a logical volume (e.g., `infra_lv`), allocating `100%` of the volume group:

```bash
sudo lvcreate --name infra_lv -l "100%VG" infra_vg

# Output:
Logical volume "infra_lv" created.
```

#### 4. **Create the ext4 filesystem**
Since no further partitioning is needed on the logical volume, create the `ext4` filesystem directly:

```bash
sudo mkfs.ext4 /dev/mapper/infra_vg-infra_lv
```

#### 5. **Mount the Logical Volume and Configure for Automatic Mounting**

To ensure the logical volume is mounted on boot, add an entry to `/etc/fstab`. When configuring the system to mount a logical volume automatically at boot, you can reference the device either by its device name (e.g., `/dev/mapper/infra_vg-infra_lv`) or by its `UUID` (Universally Unique Identifier). However, it is generally better to use the UUID because it is assigned when the device is formatted and it is a fixed value:

- **Create the mount point**:
  
```bash
sudo mkdir -p /mnt/storage
```

- **Get the LVM volume UUID**: Use `blkid` to retrieve the UUID of the logical volume. Replace the UUID in the example below with your actual UUID:

```bash
sudo blkid | grep "infra_vg-infra_lv"

# Example Output:
/dev/mapper/infra_vg-infra_lv: UUID="b8d6adfd-8be9-49b5-acf2-aae946f9b55f" BLOCK_SIZE="4096" TYPE="ext4"
```

- **Edit `/etc/fstab`**: Add the following line to mount the logical volume on boot. Ensure you replace the UUID with the one from your `blkid` output:
  
```bash
UUID=b8d6adfd-8be9-49b5-acf2-aae946f9b55f  /mnt/storage  ext4  defaults  0 0
```

- **Check `/etc/fstab`**: Verify the `/etc/fstab` configuration is valid before rebooting. Use the `mount -a` command to test the mounts:

```bash
# Reload systemd (optional)
sudo systemctl daemon-reload
     
# Mount all filesystems specified in fstab
sudo mount -a
```
  Check the output of `mount -a` command for errors or warnings and eventually correct the `/etc/fstab` file

- **Reboot and Verify**: After a successful reboot, check if the logical volume is mounted:

```bash
mount | grep infra_vg-infra_lv

# Example Output:
/dev/mapper/infra_vg-infra_lv on /mnt/storage type ext4 (rw,relatime,seclabel)
```

</details>


## 13. Storage: Filesystem Creation

On **NFS/Backup Server (`nfs-server`)**, create an `ext4` filesystem on `/dev/vdb` and set it up for NFS sharing with the **Client Workstation (client)** only.

<details>
  <summary>Answer</summary>

#### 1. **Prepare the NFS share**
Before we can export the NFS mount we have to preparte the share, creating the filesystem and creating a persistent mount point.

- **Checking for the correct device name**: use `fdisk` to check device name
  ```bash
  fdisk -l
  
  # Output omitted for brevity:
  Disk /dev/vdb: 30.3 GiB, 32505856000 bytes, 63488000 sectors
  ```
  
- **Partitioning and format the device**: Create a snigle GPT partition for the full size of the device. Here you can use any partition program you know, then format the partition in `ext4`:
  
  ```bash
  # partitioning the disk
  sudo parted -s /dev/vdb -a optimal \
        mklabel gpt \
        mkpart nfs ext4 0% 100%
  
  # creating the filesystem
  sudo mkfs.ext4 /dev/vdb1
  
  # output omitted for brevity
  Writing superblocks and filesystem accounting information: done
  ```

#### 2. **Mount device and Configure for Automatic Mounting**

To ensure the device is mounted on boot, add an entry to `/etc/fstab`. When configuring the system to mount a logical volume automatically at boot, you can reference the device either by its device name (e.g., `/dev/vdb`) or by its `UUID` (Universally Unique Identifier). However, it is generally better to use the UUID because is assigned when the device is formatted and it is a fixed value:

- **Create the mount point**:
  
  ```bash
  sudo mkdir -p /mnt/share
  ```

- **Get the LVM volume UUID**: Use `blkid` to retrieve the UUID of the logical volume. Replace the UUID in the example below with your actual UUID:

  ```bash
  sudo blkid | grep "vdb1"

  # Example Output:
  /dev/vdb1: UUID="166cbdf6-d588-4724-bcaf-9f4e315b0a5c" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="nfs" PARTUUID="55d36526-46dd-4529-86e4-177e416e31bf"
  ```

- **Edit `/etc/fstab`**: Add the following line to mount the logical volume on boot. Ensure you replace the UUID with the one from your `blkid` output:

  ```bash
  UUID=166cbdf6-d588-4724-bcaf-9f4e315b0a5c /mnt/share  ext4  defaults  0 0
  ```

- **Check `/etc/fstab`**: Verify the `/etc/fstab` configuration is valid before rebooting. Use the `mount -a` command to test the mounts:

  ```bash
  # Reload systemd (optional)
  sudo systemctl daemon-reload

  # Mount all filesystems specified in fstab
  sudo mount -a
  ```
  Check the output of `mount -a` command for errors or warning and eventually correct the `/etc/fstab` file

- **Reboot and Verify**: After a successful reboot, check if the logical volume is mounted:

  ```bash
  sudo mount | grep vdb1
     
  # Example Output:
  /dev/vdb1 on /mnt/share type ext4 (rw,relatime,seclabel)
  ```
  
#### 3. **Install the NFS package**
On Rocky linux the package `nfs-utils` contains NTP tools, and the client/server daemons

```bash
sudo dnf -y install nfs-utils
```
#### 4. **Start and Enable the NFS server at boot**

```bash
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```
#### 5. **Export the NFS mount point**

The share must be available only for the **Client Workstation (`client`)**, see `man exports` for a brief explanation

- **Prepare the export file**
  
   ```bash
   echo '/mnt/share  192.168.10.10/32(rw)' \
     | sudo tee -a /etc/exports.d/nfs-share.exports 2>/dev/null
    
   # Output
   /mnt/share            192.168.10.30/32(rw)
   ```

- **Check if share is properly exported**
  
    ```bash
    # Export all share
    sudo exportfs -a
    
    # Check if the /mnt/share is exported
    sudo exportfs -v
    
    # Output:
    /mnt/share   192.168.10.30/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
    ```

</details>

## 14. Storage: Swap Space Management

Add a `512 MB` swap file on **Web and Application Server (`webapp-services`)** and configure the system to use it automatically at boot. The Web and Application Server (webapp-services) should not have any swap, if it were not so, add the swap file as additional virtual memory.

<details>
  <summary>Answer</summary>

To set up a swap file, first we have to create and empty file and then initialize it with `mkswap` command. See the following manual pages: `mkswap`, `fstab`, `swapon`.

#### 1. **Create the Swap File and initialize it**:

For best performance, swap files should be placed on fast disks (e.g., `NVMe` or `SSD`). In our case, a good place could be /var/swapfile.

- **Checking the actual swap configuration**
  ```bash
  sudo swapon --show
  ```
  If no output is returned, there is no swap space configured. In either cases, proceed with creating a swap file.

- **Creating the swap file**
  ```bash
  sudo dd if=/dev/zero of=/var/swapfile bs=512M count=1
  
  # output
  1+0 records in
  1+0 records out
  536870912 bytes (537 MB, 512 MiB) copied, 1.38528 s, 388 MB/s
  ```

- **Checking swap file permissions**: `mkswap` will warn if permission are insecure
  ```bash
  sudo chmod 0600 /var/swapfile
  ```

- **Initialize the swap file**
  ```bash
  sudo mkswap --verbose /var/swapfile 
  
  # Output
  Setting up swapspace version 1, size = 512 MiB (536866816 bytes)
  no label, UUID=57d5cef8-2882-4826-acaf-be55e64bacad
  ```

#### 2. **Activate the swap file and add it to fstab for persistence**:

- **Activate the new swap file**
  ```bash
  sudo swapon --verbose /var/swapfile
  
  # Output Omitted for brevity
  swapon: /var/swapfile: found signature [pagesize=4096, signature=swap]
  swapon: /var/swapfile: pagesize=4096, swapsize=536870912, devsize=536870912
  ```

- **Add swap file to `/etc/fstab`**
   ```bash
   # Adding the swap file to fstab
   echo "/var/swapfile   none   swap   defaults   0 0" | sudo tee -a /etc/fstab 2>/dev/null
   
   # Check for syntax errors
   sudo mount -a
   ```

- **Verify Swap Space**: this check should be perfomed after system restart
  ```bash
  free -m
  
  # Output should look as
  
                 total        used        free      shared  buff/cache   available
  Mem:             961         396         532           0         179         564
  Swap:            511           0         511
  ```

</details>

## 15. Storage: Remote Filesystem Access

Mount the NFS share you created on step "**#13. Storage: Filesystem Creation**" on the **Client Workstation (`client`)** and ensure it is accessible. Make the share persistent across reboot.

<details>
  <summary>Answer</summary>

#### 1. Check correct permission on NFS share on **NFS/Backup Server (`nfs-server`)**

By default, NFS shares use `sec=sys` as the default authentication mechanism, this means users are identified using the `UID` reported by the client to the server. To avoid permission errors the simplest solution is to set the permission of the NFS share to `1777` aka `a=rwx,o+t` (same as `/tmp` directory). This allows any user to read and write file on such directory, while the `t` flag, or sticky bit, allows only to the owner or the `root` user to rename or delete a file.

On a production system, a better security mechanism should be considered, for example [NFS Kerberos](https://wiki.archlinux.org/title/NFS/Kerberos) authentication (`sec=krb5p`), but this is not required for this task.

Set privileges for NFS share on **NFS/Backup Server (`nfs-server`)**

```bash
sudo chmod a=rwx,o+t /mnt/share/

# The directory permissions should now be
drwxrwxrwt.  3 root root 4096 Sep  5 07:19 share
```

#### 2. Install NFS Client tools on **Client (`client`)

On debian system the NFS client package is `nfs-common` while on Red Hat based system the package is name `nfs-utils`.

```bash
sudo apt install -y nfs-common
```

#### 3. Mount the NFS share

- **Check if the NFS share is properly exported**

  ```bash
  # This will check the nfs-server correctly share the directory
  sudo showmount -e 192.168.20.30

  # Expected output
  Export list for 192.168.20.30:
    /mnt/share 192.168.10.30
  ```

- **Mount the NFS share**

  ```bash
   sudo mkdir -p /mnt/share
   sudo mount -t nfs -o rw 192.168.20.30:/mnt/share /mnt/share
  ```

- **Test the NFS share**
  To test if the NFS share work properly we can create a dummy file on it. If you get a permission error double check if permission has been set on the **NFS/Backup Server (`nfs-server`)**
  ```bash
  cd /mnt/share
  touch simoni
  ```

#### 4. Make share persistent across reboot

Until now, we've used the `fstab` mechanism to persistently mount devices at boot. For this task, an entry like:

```fstab
192.168.20.30:/mnt/share	/mnt/share	nfs rw,_netdev	0 0
```

is sufficient to mark the task as solved. However, systems that use **systemd** provide a better option using `.mount` unit files. While this method might initially seem more complex, it offers better flexibility and manageability. With `systemd`, features such as mounting on demand, automatic remounting in case of failure, and more advanced dependency management make handling mounts more robust and efficient.

- **Create the `mount` unit file**: Systemd is fussy about the descriptor name, because it must be the full mountpoint replacing slash with dash in our case `/mnt/share` will become `mnt-share.mount`:

  ```bash
  cat <<EOT | sudo tee /etc/systemd/system/mnt-share.mount 2>/dev/null
  [Unit]
  # mount -t nfs -o rw,user 192.168.20.30:/mnt/share /mnt/share

  Description=Simple NFS mounting

  [Mount]

  What=192.168.20.30:/mnt/share
  Where=/mnt/share
  Type=nfs
  Options=rw,_netdev

  [Install]

  WantedBy=multi-user.target
  WantedBy=remote-fs.target
  EOT
  ```

- **Reload systemd and activate the mount file**: enabling the mount unit file will allows sytem to mount the share at boot

  ```bash

  # Reload systemd configuration
  sudo systemctl daemon-reload

  # Enable the mount unit file
  sudo systemctl enable mnt-share.mount

  # Output
  Created symlink /etc/systemd/system/multi-user.target.wants/mnt-share.mount → /etc/systemd/system/mnt-share.mount.
  Created symlink /etc/systemd/system/remote-fs.target.wants/mnt-share.mount → /etc/systemd/system/mnt-share.mount.
  ```

- **Mounting the share and test it**:

  ```bash
  # Mounting the share
  sudo systemctl start mnt-share.mount
  
  # Check if share is mounted
  mount | grep share
  
  # Output should be
  192.168.20.30:/mnt/share on /mnt/share type nfs4 (rw,nosuid,nodev,noexec,relatime,vers=4.2,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.10.30,local_lock=none,addr=192.168.20.30,user)
  
  # Testing share creating a dummy file named simoni_bis
  cd /mnt/share/
  touch  simoni_bis
  ls -la
  
  # Output: you should see the simoni_bis file
   -rw-rw-r-- 1 osboxes osboxes     0 Sep  6 08:40 simoni_bis
  ```

</details>

## 16. Essential Commands: Basic Git Operations

  - On **Client (`client`)**, log as `testuser` initialize a Git repository called `test-repo`, create a simple README.md file, and push it using SSH protocol on the remote git repository present on the **Attacker (`attacker`)** system under `/var/git/test-repo`.

<details>
  <summary>Answer</summary>

To perform this task we need the ip address of the **Attacker (`attacker`)** system, you can directly get it using GNS3 interface. For this task we will assume that the ip is `192.168.122.116/24`

#### 1. Install GIT client

```bash
sudo apt install -y git
```

#### 1. Login as `testuser`
```bash
sudo su - testuser

id
# Output
uid=1002(testuser) gid=1002(testuser) groups=1002(testuser),27(sudo),100(users),1001(nfs-read)
```

#### 3. Initialize the new GIT repository `test-repo`

Here we'll create also the `README.md` file and add it to the repository
```bash
cd
mkdir test-repo
cd test-repo
# Init repo
git init
# Setting branch
git branch -m master
touch README.md
git add README.md

# Setting git user data to avoid errors or warnings
git config --global user.name "testuser"
git config --global user.email "testuser@example.com"

# commit work
git commit -a -m "First Commit"

# Output
[master (root-commit) ac20a15] First Commit
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 README.md
```

#### 4. Push on remote GIT server
We have to properly add the remote using the `ssh` protocol and then push the work on `master` branch

```bash
# We add the remote calling it origin
git remote add origin ssh://192.168.122.116:/var/git/test-repo
# Pushing the work on upstream server origin on master branch
git push --set-upstream origin master
testuser@192.168.122.116's password: 
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 210 bytes | 210.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
To ssh://192.168.122.116:/var/git/test-repo
 * [new branch]      master -> master
branch 'master' set up to track 'origin/master'.
```
</details>

## 17. Users and Groups: ACL Configuration

On the **NFS/Backup (`nfs-server`)** server, set up NFS ACLs so that users in the `nfs-read` group should be able to traverse the directory tree of the NFS share, allowing them to navigate through directories, reading files, but not modify or create files. Test this by trying to write a file from the **Client (`client`)** system.

<details>
  <summary>Answer</summary>

#### 1. Verify ACL Support on the NFS/Backup Server

The ACL feature should generally enabled by default on `ext4` file systems. You can double check this with the following command on **NFS/Backup (`nfs-server`)**:

```bash
sudo tune2fs -l /dev/vdb1 | grep "Default mount options:"

# Expected Output
  Default mount options:    user_xattr acl
```

The output should include `acl`. If ACL is not listed, you may need to enable it in the file system mount options.

#### 2. Creating the `nfs-read` Group

The question implies that the GID of the `nfs-read` group on the **NFS/Backup (`nfs-backup`)** server must match the GID used on the **Client (`client`)** system. This ensures consistency across both systems.

```bash
# Creating the nfs-read group with the same GID as the `nfs-read` group on the `client` system
groupadd --gid 1001 nfs-read
```

#### 3. Creating the ACL So `nfs-read` Group Users Can Only Read and Execute
The `execute` perimssion is needed for the user to traverse directories inside the NFS share

```bash
# Set ACL to allow read and execute permissions for the `nfs-read` group
sudo setfacl -R -m g:nfs-read:rx /mnt/share

# Check that the ACLs are set correctly
getfacl /mnt/share

# Expected output
# file: /mnt/share/
# owner: root
# group: root
# flags: --t
# user::rwx
# group::rwx
# group:nfs-read:rx
# mask::rwx
# other::rwx
```

#### 4. Testing

Perform the following tests on the **Client (`client`)** system:

- As the `osboxes` user, you should be able to create and modify existing files:
  ```bash
  osboxes@client:~$ echo "TEST" > /mnt/share/simonette
  ```

- As the `testuser` user, you should only be able to read existing files but not modify them or create new files:
  ```bash
  testuser@client:~$ cat /mnt/share/simonette
  
  # Output
  TEST
  
  touch /mnt/share/plinco
  touch: cannot touch '/mnt/share/plinco': Permission denied
  ```

</details>

## 18. Essential Commands: SSL Certificates

On **Web and Application Server (`webapp-services`)**, generate a self-signed SSL certificate and configure `nginx` to use it.

<details>
  <summary>Answer</summary>

> Use information in directories `/usr/share/doc/openssl` and `/etc/nginx/snippets` to properly complete this task
{.is-info}

#### 1. Generate a Self-Signed SSL Certificate

There is no standardized location for certificates on the system, but common locations include:
- `/etc/ssl/certs/` for certificates
- `/etc/ssl/private/` for private keys

Use the `openssl` command to generate a self-signed certificate:

```bash
# Create cert directories
sudo mkdir -p /etc/ssl/certs/ /etc/ssl/private/

# Generating a private key
sudo openssl genrsa -out /etc/ssl/private/privkey.pem

# Generate a self-signed certificate (interactive prompts)
sudo openssl req -new -x509 -key /etc/ssl/private/privkey.pem -out /etc/ssl/certs/cacert.pem -days 1095

# Interactive prompts
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []: webapp-services
Email Address []: admin@example.it
```

#### 2. Configure NGINX to Use Self-Signed Certificates

In Ubuntu and Debian systems, `nginx` configurations might be split into different parts. Ensure that your SSL configuration is included in the appropriate file, typically found in `/etc/nginx/sites-available/` or `/etc/nginx/nginx.conf`.

Example `nginx` configuration for SSL:

```nginx
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;

    root /var/www/html;

    ssl_certificate /etc/ssl/certs/cacert.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;

    # Add index.php to the list if using PHP
    index index.html index.htm index.nginx-debian.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### 3. Test the Configuration

Restart `nginx` and check the service status:

```bash
sudo nginx -t  # Test nginx configuration for syntax errors
sudo systemctl restart nginx
```

Open Firefox on **Client (`client`)** and connect to [https://192.168.20.20](https://192.168.20.20). If everything works, you should see a warning:

> Warning: Potential Security Risk Ahead
{.is-warning}

Click on the "Advanced" button and then "Accept the Risk and Continue". You should see the NGINX welcome page.

For troubleshooting, check the systemd journal:

```bash
journalctl -xeu nginx.service
```
</details>


## 19. Essential Commands: System Performance Monitoring

Monitor the CPU on **Management Server (`infra-services`)** using `top` or `htop`, and identify and kill a non system process that is using excessive resources.

<details>
  <summary>Answer</summary>

#### 1. **Use `top` or `htop` to identify anomalous processes**

Every 30 seconds, `top` or `htop` should display a `yes` process that uses around 90% of the CPU. Killing this process won't have a lasting effect because it is a subprocess of a bash script located at `/usr/local/cputest.sh`. This script is executed by a `cronjob`. A good way to identify and kill such a process is by using the output of `ps faux`:

```
ps faux

# Output omitted for brevity
rocky       1021  0.0  0.3 222600  3132 ?        S    21:28   0:00 /bin/bash /usr/local/cputest.sh
rocky       1439  0.0  0.1 217156   924 ?        S    21:40   0:00  \_ sleep 30
```

Kill the process using the following command (adjust the process ID to match the one in your case):

```
sudo kill -9 1021
```

You can also remove the cronjob task by using `crontab -e` and deleting or commenting out the `@reboot` task.

</details>


## 20. Operations Deployment: Script deployment and Scheduling

On **NFS/Backup (`nfs-server`)**, create a script that backs up the `/var/www/html` directory from **Web Application Services (`webapp-services`)** into the `/mnt/share` directory. Schedule the script execution every day at midnight. The name of the backup file should follow this format: `webapp-backup-YYYYMMDD`. Up to you the choice of the archive format and eventually the compression algorithm.


<details>
  <summary>Answer</summary>

Linux offers many tools to perform backup activities. To complete this task, you can use different solutions.

A straightforward way is to use the `rsyncd` daemon and the `backup` user to perform a local sync and then use `tar` to compress the archive.

### 1. Install needed tools and set environment on **Web Application Services (`webapp-services`)**

- **Installing tools**
  
  ```bash
  sudo apt install rsync acl -y
  ```

- **Allow the backup user to read the `/var/www/html`**
  
  ```bash
  # Add ACL for current files
  sudo setfacl -R -m "u:backup:rX" /var/www/html
  # Allow access to new files and directories
  sudo setfacl -R -d -m "u:backup:rX" /var/www/html
  ```

- **Create a simple `rsyncd.conf` file for backup (`man rsyncd.conf`)**
  
  ```bash
  cat << "EOT" | sudo tee /etc/rsyncd.conf 2>/dev/null
  uid = backup
  gid = backup
  use chroot = yes
  max connections = 4
  syslog facility = local5
  pid file = /var/run/rsyncd.pid
  
  [html]
     path = /var/www/html/
     comment = whole /var/www/html/
  EOT
  ```

- **Enable and start the rsyncd daemon**
  
  ```bash
  sudo systemctl daemon-reload 
  sudo systemctl enable rsync.service 
  sudo systemctl start rsyncd.service
  sudo systemctl status rsync.service
  ```

### 2. Install `rsync` on **NFS/Backup (`nfs-server`)** and check if it works

  ```bash
  dnf install rsync
  rsync rsync://backup@192.168.20.20
  
  # Output should be
  html           	whole /var/www/html
  ```

### 3. Create a backup script on **NFS/Backup (`nfs-server`)** in the file `/usr/local/bin/backup_webapp.sh`

  ```bash
  #!/bin/bash

  MAIN_DIR=/mnt/share
  BACKUP_TMP="${MAIN_DIR}/.backup"
  BACKUP_NAME="${MAIN_DIR}/webapp-backup-$(date +"%Y-%d").tar.gz"
  LOG_FILE="${MAIN_DIR}/backup.log"

  # Create temp directory if it doesn't exist
  mkdir -p "${BACKUP_TMP}"

  # Syncing the /var/www/html from the remote system
  rsync -a --delete rsync://backup@192.168.20.20/html/ "${BACKUP_TMP}/" 2>> "${LOG_FILE}"

  # Check if rsync was successful before proceeding
  if [ $? -eq 0 ]; then
    # Archiving files into a compressed file
    tar -czf "${BACKUP_NAME}" -C "${BACKUP_TMP}" . 2>> "${LOG_FILE}"

    # Cleaning up temp directory
    rm -rf "${BACKUP_TMP}"

    # Logging successful backup
    echo "$(date) Backup completed: ${BACKUP_NAME}" >> "${LOG_FILE}"
  else
    echo "$(date) rsync failed. Backup aborted." >> "${LOG_FILE}"
  fi
  ```

### 4. Schedule the job at midnight every day

  ```bash
  0 0 * * * /usr/local/bin/backup_webapp.sh
  ```
</details>
