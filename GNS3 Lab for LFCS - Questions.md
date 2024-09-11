# GNS3 Lab for LFCS: Questions

 [Back to main](GNS3%20Lab%20for%20LFCS%20-%20Overview%20and%20Guidelines.md)

> [!TIP]
> This document includes only questions.

## 1. Users and Groups: User Management

On **Client Workstation (`client`)**, create a new user named `testuser`. Set `password` as password and add the user to the `sudo` group. Create a new group named `nfs-read` and assign the user `testuser` to this group.

## 2. Networking: IPv4/IPv6 Configuration

Configure **Router (`router`)** to route traffic between the Management and Application subnets. Ensure that devices on both subnets can reach each other.


## 3. Networking: Packet Filtering and NAT

Configure the **Router (`router`)** system to:

1. Apply NAT for all outgoing traffic towards the Internet.
2. Enable forwarding between `eth0` (Management) and `eth1` (Application) subnets.
3. Allow the external interface (`eth2`) to obtain IPv4 and IPv6 configuration from an external DHCP server.
4. Allow hosts in the Management and Application subnets to query external DNS.

> [!TIP]
> To perform this task, you can use `iptables`, `firewalld`, or another firewall application that you are familiar with. Try to avoid mixing different solutions.


## 4. Security: Reconnaissance Attack

As an ace hacker, use the **Attacker (`attacker`)** system to perform a reconnaissance attack on the router's `eth2` interface. Your objective is to identify which services are exposed on the router. Report any insecure services.


## 5. Security: Router Hardening

Following a security audit, a critical vulnerability was discovered on the **Router (`router`)**: an unsecured service is publicly exposed on the internet. As a white hat engineer, your task is to identify the service and port, and secure it from external access while maintaining availability for internal users. Additionally, ensure that by default, no external traffic is allowed into the `router`, except for the `ssh` service.

## 6. Networking: SSH Configuration

On **Web and Application Server (`webapp-services`)**, configure SSH to allow key-based authentication only and disable root login.


## 7. Operations Deployment: Kernel Parameters Configuration

Modify kernel parameters on **Management Server (`infra-services`)** to increase the maximum number of open files. Ensure this change is persistent across reboots.

## 8. Operations Deployment: Process Management

On **Web and Application Server (`webapp-services`)**, start a web service (e.g., `nginx`), monitor its process, and configure it to restart automatically if it crashes.

## 9. Operations Deployment: Software Management

On the **Client Workstation (`client`)**, install the `htop` package from a remote repository and verify the installation.

## 10. Operations Deployment: Container Management

On the **NFS/Backup Server (`nfs-server`)**, install Docker, create a container running an `nginx` web server, and verify its operation. The Docker-CE repository can be installed from this url:
- [https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo)

Eventually, you can use documentation at `/usr/share/doc/docker-compose-plugin` to create a working `docker-compose.yml` configuration.

## 11. Networking: Load Balancing

Configure the **Management Server (`infra-services`)** to act as a load balancer for HTTP traffic between the **NFS/Backup Server (`nfs-server`)** and the **Web and Application Server (`webapp-services`)**. Use the documentation in /usr/share/doc/haproxy/ as a reference to create a working HAProxy configuration.

## 12. Storage: LVM Management

On **Management Server (`infra-services`)**, create a new logical volume using the `30 GB` disk (`/dev/vdb`) and mount it to `/mnt/storage` with the `ext4` filesystem. Configure it to mount automatically at boot.

## 13. Storage: Filesystem Creation

On **NFS/Backup Server (`nfs-server`)**, create an `ext4` filesystem on `/dev/vdb` and set it up for NFS sharing with the **Client Workstation (client)** only.

## 14. Storage: Swap Space Management

Add a `512 MB` swap file on **Web and Application Server (`webapp-services`)** and configure the system to use it automatically at boot. The Web and Application Server (webapp-services) should not have any swap, if it were not so, add the swap file as additional virtual memory.

## 15. Storage: Remote Filesystem Access

Mount the NFS share you created on step "**#13. Storage: Filesystem Creation**" on the **Client Workstation (`client`)** and ensure it is accessible. Make the share persistent across reboot.

## 16. Essential Commands: Basic Git Operations

  - On **Client (`client`)**, log as `testuser` initialize a Git repository called `test-repo`, create a simple README.md file, and push it using SSH protocol on the remote git repository present on the **Attacker (`attacker`)** system under `/var/git/test-repo`.

## 17. Users and Groups: ACL Configuration

On the **NFS/Backup (`nfs-server`)** server, set up NFS ACLs so that users in the `nfs-read` group should be able to traverse the directory tree of the NFS share, allowing them to navigate through directories, reading files, but not modify or create files. Test this by trying to write a file from the **Client (`client`)** system.

## 18. Essential Commands: SSL Certificates

On **Web and Application Server (`webapp-services`)**, generate a self-signed SSL certificate and configure `nginx` to use it.

## 19. Essential Commands: System Performance Monitoring

Monitor the CPU on **Management Server (`infra-services`)** using `top` or `htop`, and identify and kill a non system process that is using excessive resources.

## 20. Operations Deployment: Script deployment and Scheduling

On **NFS/Backup (`nfs-server`)**, create a script that backs up the `/var/www/html` directory from **Web Application Services (`webapp-services`)** into the `/mnt/share` directory. Schedule the script execution every day at midnight. The name of the backup file should follow this format: `webapp-backup-YYYYMMDD`. Up to you the choice of the archive format and eventually the compression algorithm.
