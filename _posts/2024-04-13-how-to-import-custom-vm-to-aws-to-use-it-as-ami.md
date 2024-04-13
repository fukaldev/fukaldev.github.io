---
layout: post
title: How to Import a Custom VM into AWS to Use as an AMI
date: '2024-04-13 18:19:04 +0300'
categories: [AWS]
tags: [aws, image]
img_path: /img/aws-import/
---

![Meme](meme.png)

Hi everyone, today I want to discuss importing a custom image to AWS for use as an AMI (Amazon Machine Image). You might wonder why we need this when AWS already offers various OS images. While AWS provides a wide range of OS images tailored for their environment, these may include custom modifications, which can be cumbersome to deal with. For example, I encountered a situation where I needed to use `dnsmasq` as my DNS server, but the Debian image provided by AWS used `systemd-resolve` by default, which is not standard for a base installation.

Of course, there are other reasons why you might want to import your custom image to AWS:
- Your image may not be available among AWS images and marketplace offerings.
- You may encounter issues due to custom modifications applied to images provided by AWS.
- You may want to distribute the same image to other cloud platforms or deploy it on a physical machine.

## Booting up the VM
I'll be using VirtualBox to create my custom image, but you can use any virtualization tool. Ultimately, we only need a VHD (Virtual Hard Disk) file. By providing this VHD to AWS, we can create a snapshot of our custom machine, which can then be turned into an AMI.
I won't delve into the details of creating a virtual machine here, except for the Virtual Hard Disk. When setting up virtual machines, we define a Virtual Hard Disk size. This is what the virtual machine sees, assuming its storage is only as much as the Virtual Hard Disk size. In reality, it utilizes the host system's storage, claiming space from it whenever needed until it reaches its limit or exhausts the host system's storage.

What's crucial for us here is that when we import our image to AWS, it attempts to allocate space equal to the Virtual Hard Disk size. Typically, when creating virtual machines for personal use, I allocate the entire Virtual Hard Disk size. However, this approach isn't cost-effective for AWS because it preallocates the full disk size, which isn't necessary when storing snapshots and volumes for creating AMIs. Let's keep it at 20 GiB, which is sufficient for our purposes and cost-effective for storage.

![Modifying VHD size](vhd-size.png)

### In AWS Terminology
- **EBS**: the AWS service itself.
- **EBS Volume**: think of it like a hard drive you can attach to an EC2 instance.
- **Snapshot**: a point-in-time copy of your volume.
- **AMI**: a copy of a full instance.
- **EC2**: the service that RUNS the VM compute.

This means every Snapshot and Volume occupies some memory and uses your quota.

Now, let's proceed with setting up your OS in the usual way.

## Tuning the VM
In AWS documentation about [VM Import/Export](https://docs.aws.amazon.com/vm-import/latest/userguide/prerequisites.html), you can find a list of prerequisites. Most distributions satisfy these requirements, but we'll focus on the trickier ones in this article.

- **Enable Secure Shell (SSH) for remote access**: Ensure there is an SSH connection with your VM machine by verifying connectivity.
- **Ensure that your Linux VM is not using predictable network interface device names**: We'll address this later in the article.

Additionally, although not mentioned in the documentation, modifying console access settings is crucial. This allows debugging in case of networking or SSH issues.

### SSH access
Set your network adapter as **Bridge** in your VM settings and your VM get an IP address within the same subnet of host machine.
Use your credentials and make sure there is an succesful SSH connection. You can use `systemctl status sshd` command to check SSH service status. If service not available, you can install SSH with your OS's package manager.
![Network settings](network-settings.png)

```bash
$ ssh fukal@192.168.1.108
The authenticity of host '192.168.1.108 (192.168.1.108)' can't be established.
ED25519 key fingerprint is SHA256:*******************************************.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.108' (ED25519) to the list of known hosts.
fukal@192.168.1.108's password: 
Linux debian 6.1.0-20-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.85-1 (2024-04-11) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
fukal@debian:~$ systemctl status sshd
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Sat 2024-04-13 14:15:57 EDT; 16min ago
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 501 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 510 (sshd)
      Tasks: 1 (limit: 9475)
     Memory: 8.1M
        CPU: 188ms
     CGroup: /system.slice/ssh.service
             └─510 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Warning: some journal files were not opened due to insufficient permissions.
```
### Console Access

* Edit the `/etc/default/grub` file:
    * Remove `resume=` from the kernel parameters.
    * Modify `GRUB_TERMINAL` to read `GRUB_TERMINAL="console serial"`.
    * Add `GRUB_SERIAL_COMMAND="serial --unit=0 --speed=115200"` to configure GRUB's serial connection.
    * Edit `GRUB_CMDLINE_LINUX` to include `console=tty0 console=ttyS0,115200` for adding the serial console to the Linux kernel boot parameters.
    * Then, update GRUB using either:
        ```
        grub-mkconfig -o /boot/grub/grub.cfg
        ```
        or
        ```
        update-grub
        ```
    Ensure you switch to the `root` user.
    * Reboot the machine and verify the kernel boot commands, which should include `console=ttyS0`.
        ```
        dmesg | grep console=ttyS0
        ```

### Network Settings

* Execute the following command to clear any persistent network interface naming:
    ```
    > /etc/udev/rules.d/70-persistent-net.rules
    ```
    This will empty the file, but refrain from rebooting immediately as it can be regenerated.
* Add `net.ifnames=0` and `biosdevname=0` parameters to `GRUB_CMDLINE_LINUX` to prevent predictable network interface device names.

---

Power off your device gracefully using the `halt -p` command. \
 ***We've completed the VM setup.***

## Getting the VHD and Uploading to S3

![Get VHD](get-vhd.png)

VirtualBox stores machines in VDI format. To use an image in AWS, we need to ensure it's in a supported format, such as VHD. Use the **Virtual Media Manager** tool in VirtualBox to convert the image. We don't need to allocate the full size, so proceed accordingly.

Then, basically upload your VHD file to one of your S3 buckets. Again, I'm not going to get into the details of that; you can find many tutorials and ways about uploading files to S3.

## Importing Image from AWS Console

Navigate to the **EC2 Image Builder** service and click on **Images** from the left pane. You'll see **Import Image** on the right.

![Images](images.png)

Then, you can select your base OS and image name. If your OS is not listed, you can go with Amazon Linux. Since Debian is compatible with Ubuntu, I'll go with that.

![Debian](debian.png)

In this section, select the uploaded VHD file from your S3 Bucket.

![S3](s3.png)

Then, select the related and authorized IAM role.

**Your image will be ready in minutes.**

