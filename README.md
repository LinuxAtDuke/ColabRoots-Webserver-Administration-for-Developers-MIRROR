Web Server Administration for WebApp Developers
==============================================

Version: 2015-11-08

[https://gitlab.oit.duke.edu/colabroots/Webserver-Administration-for-Webapp-Developers](https://gitlab.oit.duke.edu/colabroots/Webserver-Administration-for-Webapp-Developers)

Prerequisites
-------------

Students taking this class should be comfortable with Linux and the Linux command line.  A minimum proficiency is provided by taking and understanding the Linux@Duke "Introduction to Linux" class.

Students should also have created a VM-Manage "RHEL 6 Basic" VM:

1. Using a web browser, go to *https://vm-manage.oit.duke.edu*
2. Login using your Duke NetId.
3. Create a new project for this class.
4. Select *RHEL 6 Basic* for the Server.

The vm-manage web page will tell you the name for your VM. The web site will also tell you the initial username and password. You should connect via ssh.

Example: `ssh bitnami@colab-sbx-87.oit.duke.edu`

Once logged in via ssh, enter the `passwd` command to set a unique password.

Example:

```
$ passwd
Changing password for bitnami.
(current) UNIX password:
Enter new UNIX password:
Retype new UNIX password:
```

Basic Server Configuration
--------------------------

In this section, you'll learn:

1. How to update the software on your system
2. How to setup automatic updates, so you don't have to keep track of them
3. How to setup SSH Key Authentication to avoid the use of passwords for login
4. How to setup a host-based firewall with `iptables`

### Updating the Software on Your VM ###

So you're in your new VM-Manage RHEL6 VM.  Or you've installed the OS yourself, or you're using an image from a popular cloud provider.  The very first thing to do is patch.  In almost every case here, there will be updates, bug fixes and critical security patches available for your system.

Most Linux distributions use packages to install software on your system.  Think of packages like zip files that contain everything that's needed to run a given piece of software, as well as information about any dependencies that are required.  Packages, and updates to packages, are easy to install when using the build-in package manager for your Linux distribution.  On RHEL-based systems, the package manager is **rpm**, and rpm packages and updates are generally managed by **yum**.  (On Debian-based systems, the same are accomplished using `dpkg` and `apt`)

** A Brief Aside **

_"The versions of software in these packages is always old!  Why not just build it from source in the grand-old tradition of Unix software installation?"_

Yes, the software in these packages tends to lag behind the actual software projects.  Sometimes by quite a lot.  Package maintainers for RHEL and Debian, and most other distributions, will usually update packages between major releases or when certain milestones are hit.  In the case of RHEL systems, Red Hat freezes the package's major version and updates only between RHEL's major versions.

For example, in this class we're using RHEL6.  The version of Apache's HTTP Server bundled with RHEL6 is httpd-2.2.  In RHEL7, it's httpd-2.4.  Red Hat does this to maintain backward compatibility for each version of RHEL.

_However,_ Red Hat does continue to maintain the version of the httpd package for each version of RHEL that's still supported.  If a new vulnerability is discovered in httpd-2.2, Red Hat patches and releases the fix.  If there's a bug discovered, Red Hat will patch it.  What Red Hat WILL NOT do, is add new features from new versions.

_"Why does this matter to me?  I could get the latest and greatest by compiling it myself!"_

That's true.  And for your own laptop, that's fine.  Or if you are part of a large department maintaining a very specific piece of software, and watch the upstream project for changes, that's probably fine too.  But you probably aren't, and you probably don't want to check every day to see if there's a new patch or bug in your bleeding-edge web server software.  That's where Red Hat, and Debian, and other package maintainers come into play.  They have groups that do this, daily.  If you use their packages, you'll get these patches and they will be applied to your system with very little effort on your part.  And because these packages are stable across versions, your updates will be unlikely to break anything.  Your system will be more secure, and less prone to unexpected outages, if you use packages maintained by an upstream provider.

You don't necessarily have to rely on Red Hat or Debian, either.  In the Red Hat world, some rpm packages are built by trusted third parties who maintain their own repositories of newer software.  Ubuntu has the concept of PPAs, or "Personal Package Archives", that are sometimes maintained by the software vendor themselves.  In these cases, you can get newer software more frequently, but continue to get the benefits from using packages to install them.  A major caveat with this approach, though, is **you must be able to trust the source of these packages**.  Untrusted sources could contain malicious software.

From a simply lazy perspective, yum or apt will also gather all the dependencies needed for your software when you install it, and install it alongside your package.

### Actually Updating the Software on your System ###

Since the VM you're using in this class is a RHEL-based system, we'll use `yum` to update the software that's installed on it.  Like so:

`sudo yum update -y`

That's it!  All of your packages have the latest patches for your OS.

_Note:_ Due to the way your VM is configured, you may have trouble installing updates because the GPG key for the "satyr" package is missing. I'm working to get this fixed, but for now, just use: `yum update -y --exclude=libreport* --exclude=abrt*`

### Repositories ###

`yum` gets it's packages from "repositories", or collections of packages and metadata available online for `yum` to read and download.  (The Debian world calls their repositories "sources", but practically, they work the same way).  In this class, your VM-Manage VM comes pre-configured to use OIT's RHEL repositories, but later we'll need to add another to install a special package, so we'll talk about them now.

Your server has special files on it that tell `yum` what repositories to look at for when it installs or updates packages.  You'll find these files in `/etc/yum.repos.d/`.  Let's look at one now:

```
$ cd /etc/yum.repos.d/
$ ls
cobbler-config.repo  rhel-source.repo
```

These two .repo files actually contain information about a number of repositories.  Let's look at rhel-source.repo (the other is a really long file).

```
$ cat rhel-source.repo
[rhel-source]
name=Red Hat Enterprise Linux $releasever - $basearch - Source
baseurl=ftp://ftp.redhat.com/pub/redhat/linux/enterprise/$releasever/en/os/SRPMS/
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[rhel-source-beta]
name=Red Hat Enterprise Linux $releasever Beta - $basearch - Source
baseurl=ftp://ftp.redhat.com/pub/redhat/linux/beta/$releasever/en/os/SRPMS/
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-beta,file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

Notice that there are two repositories in this file, rhel-source and rhel-soure-beta.  These files tell `yum` to look at some FTP sites hosted by Red Hat when it installs or updates.  There are also some GPG Keys listed so `yum` can verify the packages it's getting were really made by Red Hat (an extra security measure).

### Automatic Updates via Cron###

Updating your system is all well and good, but within days there will again be out-of-date software.  There are a number of ways to manage available updates, but if you are going to be working with a single server or a small amount of servers, it's easiest to just setup a `cron` job to perform your updates for you.  

`cron` is a daemon used to execute scheduled commands.  The easiest way to interact with `cron` as a system administrator is via configuration files.  (See also `crontab`: https://help.ubuntu.com/community/CronHowto for managing tasks as a regular user on a system.)

Below is an example `cron` configuration file:

```
# Example Cron Job
10 1 * * * root echo 'Hello World'
```

That's a pretty simple example, but let's break it down.

```
# ┌───────────── min (0 - 59)
# │ ┌────────────── hour (0 - 23)
# │ │ ┌─────────────── day of month (1 - 31)
# │ │ │ ┌──────────────── month (1 - 12)
# │ │ │ │ ┌───────────────── day of week (0 - 6) (0 to 6 are Sunday to Saturday, or use names; 7 is Sunday, the same as 0)
# │ │ │ │ │
# │ │ │ │ │
# * * * * *  user-to-run-as command-to-execute
```
_(Slightly modified) source: [https://en.wikipedia.org/wiki/Cron](https://en.wikipedia.org/wiki/Cron)_

So, from left to right, the asterisks stand for minute, hour, day of month, month and day of week.  If there is an asterisk instead of a number, it means "every time". So:

```
* 10 * * *    <-- Every minute from 10 AM to 10:59 AM
01 14 15 * *  <-- At 2:01 PM on the 15th of the Month
*/30 * * * *  <-- Tricky one!  Every 30 minutes
```

And finally that means our Example cron job:


```
# Example Cron Job
10 1 * * * root echo 'Hello World'
```

...is "Run at 1:10 AM every day, as root, and echo 'Hello World'".


`cron` config files reside in a set of directories in the file system:

```
$ ls -ld /etc/cron.*
drwxr-xr-x. 2 root root 4096 Aug  7  2014 /etc/cron.d
drwxr-xr-x. 2 root root 4096 Aug  7  2014 /etc/cron.daily
-rw-------. 1 root root    0 Sep 12  2013 /etc/cron.deny
drwxr-xr-x. 2 root root 4096 Aug  7  2014 /etc/cron.hourly
drwxr-xr-x. 2 root root 4096 Jun  3  2011 /etc/cron.monthly
drwxr-xr-x. 2 root root 4096 Jun  3  2011 /etc/cron.weekly
```

_Note:_ Ignore the cron.deny file, or read up on it later.

The cron.hourly, cron.daily, cron.weekly and cron.monthly directories do pretty much what you think they would do: run a given command every hour, day, week or month, respectively.  The cron.d directory is a bit more fluid, and lets you add commands to run on arbitrary schedules.  In these directories, there are a number of `cron` configuration files already, setup for maintenance of certain systems.

Let's look at the *system-update.cron* file included in this repo.

1. What command does it run?
2. What user does the command run as?
3. When does the command run?

You can modify this file to do your system updates however you would like.

Place the file in the `/etc/cron.d` directory and make sure it's executable.

_Note:_ Given the consistency of packages from upstream, it is rarely a problem to have these updates done without human intervention, but if you are running a critical system, you may wish to look into other options.  For what it is worth, OIT used this method to patch servers until relatively recently.

_Note:_ Another consideration for automatic updates include reboots.  A reboot is required to apply new version of the Linux kernel, as well as a few low-level system packages, like `gcc`.  You may not wish to automatically reboot after every set of patches, but you definitely need to do so regularly.  OIT patches and reboots servers every two weeks.

### SSH Key Authentication ###

SSH Key Authentication is a method for using a public and private key pair, as opposed to a password, to identify yourself when logging into a system.  This is convenient (no password!), but more importantly, it is also more secure than using password authentication, especially if the computer is available online.  This is very obvious if you have ever looked at logs of login attempts on a computer.  It is not uncommon to have thousands of login brute-force attempts a week.

With SSH Key Authentication, a user will generate a private key and a public key.  The private key is a securely-stored key only the owner should have access to.  The public key is just that, a publicly viewable key that can be given to other people or servers to authenticate the owner of the private key.  When the public key is placed on the server and SSH Key authentication is used, only someone who possesses the private key is able to login to the server.  For added security, you can require a passphrase in order to use your private key, so an attacker still cannot use it, even if they were to somehow steal a copy.

### How to setup SSH Key Authentication ###

#### Generating Keys ####

**On Your Laptop:**

```
$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/home/bitnami/.ssh/id_rsa):
Created directory '/home/bitnami/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/bitnami/.ssh/id_rsa.
Your public key has been saved in /home/bitnami/.ssh/id_rsa.pub.
The key fingerprint is:
08:58:22:08:19:1e:ae:16:4b:de:6a:cf:12:1c:1c:b8 bitnami@rhel6-generic-template-01.oit.duke.edu
The key's randomart image is:
+--[ RSA 4096]----+
|B+. .            |
|*o.+             |
|.*o .            |
|Eo+  . .         |
|o+..  . S        |
|.o.              |
| o.              |
|..o              |
|  .o             |
+-----------------+
```

Let's start with the command to generate the key itself: `ssk-keygen -t rsa -b 4096`. This command is generating a new key (`ss-keygen`), using the RSA algorithm (`-t rsa`), and a bit size of 4096 (`-b 4096`).  

_Note:_ The RSA (Rivest-Shamir-Aldeman) algorithm is considered more secure than the alternative algorithm, DSA (Digital Signature Algorithm).

The bit size is important too.  The default is 2048, but increasing the bits in the key to 4096 makes it harder decrypt the key using brute-force methods.


```
Enter file in which to save the key (/home/bitnami/.ssh/id_rsa):
Created directory '/home/bitnami/.ssh'.
```

In this section, the `ssh-keygen` command is asking where you want to store your new private key, and what to name it.  If you've never created a key before, you can accept the default.  It's also creating the .ssh directory in this example, because it didn't yet exist.

```
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

This is important as well.  You should choose a good passphrase for your key.  If your private key was ever stolen, someone could impersonate - in this case by logging into your server as you, without a password.

If you are unfamiliar with how to choose a truly strong passphrase, this [Strong Passwords Guide](https://help.ubuntu.com/community/StrongPasswords) on the Ubuntu Community Wiki is an excellent place to start.

```
Your identification has been saved in /home/bitnami/.ssh/id_rsa.
Your public key has been saved in /home/bitnami/.ssh/id_rsa.pub.
```

Finally, `ssh-keygen` is letting you know where it stored your new private (id_rsa) and public (id_rsa.pub) keys.

#### Enabling SSH Key Authentication ####

So, how do you use your new keys to login to your system?

The first thing to do is copy your **public** key to the server.  There are commands that can do this for you (`ssh-copy-id`), but to understand what they are doing, we'll copy the key manually.

**On the server:**

```
$mkdir ~/.ssh
$chmod 700 ~/.ssh
$touch ~/.ssh/authorized_keys
$chmod 600 ~/.ssh/authorized_keys
```

_Note:_ The permissions are important.  SSH authentication will fail if your .ssh directory, private key, or authorized_keys files have permissions that are too open.

**On your laptop:**

```
$cat ~/id_rsa.pub >> /tmp/authorized_keys
$scp /tmp/authorized_keys <username>@<hostname>:~/.ssh/authorized_keys
```

This is effectively copying the contents of your public key into the ~/.ssh/authorized_keys file on the server.

_Note:_ You can have multiple keys in your authorized_keys file, one per line.

On your VM-Manage VM, SSH Key Authentication is already turned on, but in other cases you may need to modify your /etc/ssh/sshd_config file.

#### Disabling Password Authentication ####

Once you have SSH Key Authentication working, it's a good idea to disable password-based authentication.

Open your /etc/ssh/sshd_config file in the editor of your choice, and change the line `PasswordAuthentication yes` to `PasswordAuthentication no`.  When this change has been made, reload the SSH daemon with `sudo service sshd reload`, to apply the changes.

### Host-based Firewalls with iptables ###

Another way to protect your system is to disallow access with a firewall.  RHEL-based (and Debian-based) distributions have a service called `iptables` that implements a host-based firewall on your system.  `iptables` is an extremely dense subject and it can do a huge number of things, but for basic firewall setup, it's not too complicated.

`iptables` is already running on your VM.  Let's look at what it's doing:

```
$ sudo iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

Well, nothing, at the moment.  What you see here are the three default `iptables` chains, INPUT, FORWARD, and OUTPUT.  We are most concerned in this class with INPUT - or traffic coming into the server.  We will ignore FORWARD and OUTPUT, but again, the Ubuntu Community has an excellent [primer on using IPTABLES](https://help.ubuntu.com/community/IptablesHowTo).

So there are no rules in your `iptables` output, and you have been able to successfully SSH in this whole time, so we know `iptables` isn't blocking anything.  We are going to add some ACCEPT rules.  It may seem redundant to add these rules when `iptables` isn't blocking anything, but this brings up an important point about how `iptables` applies it's configuration: from "top" to "bottom".  Each line in the `iptables` output has a number associated, starting from 1 at the top.  When traffic is checked against it's chain (remember INPUT for incoming traffic), `iptables` will scan from line 1 at the top, down to the bottom of the file looking for a matching rule.  The first rule it matches is applied.

What does this mean for you?  If you were to apply a DROP rule, in an effort to prevent attackers from getting into your server, without adding ACCEPT rules for yourself first, you'd be irrevocably locked out of your VM!  So remember, order matters!

So first, we'll allow traffic on port 22, the port used for SSH.  Since this is an example VM, we'll just allow traffic on port 22 from anywhere, but in the real world, you'll want to restrict it to some trusted IP addresses or subnets; say, Duke-only, or the IP address of a server operating as a bastion (jump-box).

```
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

The `-A INPUT` means "add on to the end of the INPUT chain", `-p tcp` is the tcp protocol, `--dport 22` is port 22, the port used by SSH, and `-j ACCEPT` means jump to ACCEPT, and stop processing rules in this chain.

Let's look at our `iptables` rules now, and add the line numbers this time:

```
$ sudo iptables -L -n --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:22

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
```

Now you can see line number 1 in the INPUT chain is our rule to allow SSH access.  

We'll probably also want to allow already established sessions to receive traffic, and we want to put this ahead of all the other rules.  

```
$ sudo iptables -I INPUT 1 -m state --state ESTABLISHED,RELATED -j ACCEPT
$ sudo iptables -L -n --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED
2    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:22

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination    
```

Using the `-I INPUT 1` parameters, we _inserted_ into the INPUT chain at line number 1, displacing our SSH rule to line number 2.  We also matched traffic by state (`-m state`), looking for connections that are ESTABLISHED or RELATED, and accepted them.

Now comes the fun part! BANNING!  Be careful with this.  You could accidentally end up locking yourself out.

```
$ sudo iptables -A INPUT -j DROP
$ sudo iptables -L -n --line-numbersChain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED
2    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:22
3    DROP       all  --  0.0.0.0/0            0.0.0.0/0           

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination         
```

So, what did this do?  At the end of our chain, we appended (`-A INPUT`) a rule to DROP (`-j DROP`) any traffic.  This will match everything that hasn't already been matched by a line above, and drop their connection, effectively looking as though there is no server there responding to requests - like a black hole.

It's important to note that the `iptables` rules we just created are saved into memory.  If the system reboots, or the `iptables` service restarts, these rules will be lost.  On RHEL-based systems, the easiest way to save the rules currently in memory is with the command `sudo service iptables save`.  This actually executes the `iptables-save` program and saves the configuration to /etc/sysconfig/iptables.  This file is read when `iptables` restarts, and the stored rules are loaded back into memory.

On non-RHEL based systems, like Debian, you may be required to save and load the rules manually, or with a script that you write yourself.  On these systems, you can save rules in memory with:

```
$ iptables-save > <filename>
```

And you can reload them into memory with:

```
$ iptables-load < <filename>
```

It's impractical to remember to do this each time your server or  `iptables` service restarts, so you will probably want to script some automatic way to re-load rules on startup.

That's it for `iptables` for now. We'll be adding more `iptables` rules later in the class, as we start up more services.  You are encouraged to go do some out-of-class reading about `iptables` to understand all that it can do.


Basic Web Server Configuration
------------------------------

In this section, you'll learn:

* How to install Apache's HTTP server
* How to start and stop the server, and set it to start upon boot
* How to setup Virtual Hosts, and answer to specific Server names
* How to setup logging, and log rotation

Apache is currently the most-used web server in the world.  It's slowly losing ground to a relatively new server, Nginx (pronounced Engine X), but Apache accounts for almost 50% of all the web servers in the world.  The benefit to using Apache is ease-of-use, but the trade of is (sometimes) performance.  We'll see how to get the most performance out of your Apache server, though, and how to easily configure it to serve web content.

### Installing Apache's HTTP Server ###

On RHEL-based systems, the package name for Apache is "httpd".  Installing httpd on RHEL is pretty easy:

`$ sudo yum install -y httpd`

And that's it.  Apache is installed on your server.  Class over.

Well, not quite yet.  It's not actually doing anything yet, but it is configured to serve a simple web page out of the box. Let's go ahead an get a sample page setup, and start the server.

### Starting and Stopping the Apache server ###

As we did above with the SSH daemon, you will manage the Apache daemon with the "service" command.  To start the web server, run:

`$ sudo service httpd start`

(If you receive the error: "service: command not found", use `sudo /sbin/service httpd start` instead, or add `/sbin` to you PATH).

Stopping, restating, reloading, etc. all use the same syntax:

```
$ sudo service httpd restart
$ sudo service httpd reload
```

A useful trick to catch syntax errors or other issues that may cause the Apache to fail to restart is the built-in configuration test command:

```
$ sudo service httpd configtest
Syntax OK
```

This will parse all of the Apache configs and loaded modules and look for issues that would prevent it from starting, or restarting, and allow you to troubleshoot before bringing down the service.

### Setting Apache to Startup when the Server Boots ###

By default, the httpd package does not automatically set Apache to start when the server boots up.  If your system reboots, for example, after patching, Apache would need to be manually started back up.  That's not really a practical approach for most admins.  

The `chkconfig` command is used to enable or disable services for a given runlevel.  Most Linux servers are usually running in runlevel 3. (Runlevels are out of the scope of this class, but I encourage you to read [http://www.tldp.org/LDP/sag/html/run-levels-intro.html](http://www.tldp.org/LDP/sag/html/run-levels-intro.html) to learn more.)

After installing the httpd package, you can check which runlevels the service is configured to start under:

```
$ chkconfig --list httpd
httpd          	0:off	1:off	2:off	3:off	4:off	5:off	6:off
```

You can see in the output above that the httpd service is set to be "off" by default in all 7 runlevels.  That's not really useful for a web server, so it would make sense to enable it for runlevel 3.

```
$ chkconfig --level 3 httpd on
$ chkconfig --list httpd
chkconfig --list httpd
httpd          	0:off	1:off	2:off	3:on	4:off	5:off	6:off
```

That's better.  You can see the httpd service has been enabled to run when the server enters runlevel 3.  Generally, though, we just enable httpd for all the standard runlevels that make sense for a service:

```
$ chkconfig httpd on
$ chkconfig --list httpd
httpd          	0:off	1:off	2:on	3:on	4:on	5:on	6:off
```

Your system is now set to start httpd automatically when it enters the "usual" runlevels for a server.

_Note:_ Runlevels 0, 1 and 6 are "halt", "single-user mode", and "reboot", respectively.  It does not really make sense to be running the httpd service in those states.

### Check your Handiwork ###

At this point, you have a running Apache server.  Let's check it out.

Copy the "index.html" file from the git repo you cloned earlier to `/var/www/html`, and then open your browser and navigate to "http://<name of your colab server>".

### Configuring Apache ###

Apaches primary configuration file is `/etc/httpd/conf/httpd.conf`.  This file contains a large number of global configurations and enables a basic server that serves content out of "/var/www/html", and places log files in "/var/log/httpd".  It's also configured to include files in "/etc/httpd/conf.d" if they end with ".conf".

Virtual Hosts
- Root Virtual Hosts

- Multiple Virtual Hosts


### Log Rotation ###
- Logging
- Log Formats
    https://httpd.apache.org/docs/2.2/logs.html
- Log Rotation
- - Rotatelogs + tmpwatch
    RotateLogs https://httpd.apache.org/docs/2.2/programs/rotatelogs.html
    tmpwatch http://thelinuxfaq.com/100-how-to-use-tmpwatch-command-in-linux
- - Logrotate http://www.rackspace.com/knowledge_center/article/understanding-logrotate-utility

### Performance Tuning ###

A lot of Apache's default configuration is designed to be either easy to use, or not really configured for a modern web hosting environment, at least out of the box.  This can lead to poor performance for your application, even with a small amount of load.  However, by tuning some settings, you can easily achieve significantly better performance.

The quickest way to tune your server for best performance is to adjust the number of threads Apache uses to fit within the amount of memory available to your system.  This can be a balancing act.  If there are too few threads, visitors to your site will be stuck waiting for an open thread before the page can even begin to be rendered by their browser.  On the other hand, if there are too many threads, they can easily take up all the available memory on your server, and cause Apache to start writing memory to disk, and since the disk storage is much slower than memory, this will quickly lead to very poor performance from your server.  In severe cases, this can even cause the server to become completely unresponsive.

Let's look at this a bit more closely.  The following is a snippet from the `httpd.conf` file showing the default configuration of Apache threads, using the default Multi-Processing Module (MPM):

```
<IfModule prefork.c>
StartServers       8
MinSpareServers    5
MaxSpareServers   20
ServerLimit      256
MaxClients       256
MaxRequestsPerChild  4000
</IfModule>
```

The "Prefork MPM" (noted above as `prefork.c`) is a non-threaded, pre-forking web server, and is the module that's used if you don't make any changes to your Apache startup.  Apache is generally pretty good at self-managing process creation, so you likely will not need to adjust the `StartServers`, `MinSpareServers`, or `MaxSpareServers` settings.  StartServers is the number of processes Apache creates on startup.  MinSpareServers and MaxSpareServers are the minimum and maximum allowed idle processes, respectively.  (Idle processes are ones that are not handling any requests.)

The more important settings to be aware of are the `ServerLimit`, `MaxClients`, and `MaxRequestsPerChild`.

There are threaded MPMs as well ("Worker", and in Apache 2.4 "Event").  Threaded MPMs generally offer better performance, but can only be used if other modules in use are thread-safe.  For example, mod_php, the module for handling PHP processing, is not thread safe.  In order to use a threaded MPM with PHP, the PHP processing has to be offloaded to another service using something like FastCGI.



---

Day Three start Thursday, November 12.
