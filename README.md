Web Server Administration for WebApp Developers
==============================================

Version: 2015-11-09

**Class Notes**

[https://gitlab.oit.duke.edu/colabroots/Webserver-Administration-for-Webapp-Developers](https://gitlab.oit.duke.edu/colabroots/Webserver-Administration-for-Webapp-Developers)

**Linux@Duke Public Mirror**

Clone URL:
[https://github.com/LinuxAtDuke/ColabRoots-Webserver-Administration-for-Developers-MIRROR.git](https://github.com/LinuxAtDuke/ColabRoots-Webserver-Administration-for-Developers-MIRROR.git)

Web URL:
[https://github.com/LinuxAtDuke/ColabRoots-Webserver-Administration-for-Developers-MIRROR](https://github.com/LinuxAtDuke/ColabRoots-Webserver-Administration-for-Developers-MIRROR)


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

Apaches primary configuration file is `/etc/httpd/conf/httpd.conf`.  This file contains a large number of global configurations and enables a basic server that serves content out of "/var/www/html", and places log files in "/var/log/httpd".  It's also configured to include files in "/etc/httpd/conf.d" if they end with ".conf". The conf.d directory is the recommended place to put customizations to the Apache configuration.  This is where we will configure our Virtual Host.

Virtual hosts are a way to run more than one website on a single server.  This may sound redundant if you are only planning on running a singe site anyway, but in combination with the conf.d directory, this allows you to setup and see at a glance the configurations that apply directly to your website.  This also allows you to decouple the website's name from the server's name, so you can move sites around more easily.  In practice, it's  helpful to have the server respond to it's own name for monitoring, statistics (like the server-status page we'll talk about later) and a default landing page for identification.  The virtual host (or hosts) can then be dedicated to running the websites.

There are two types of virtual hosts: `IP-based` and `name-based`.  Ip-based virtual hosts tie a virtual host to a particular IP address on your server, and each virtual host requires it's own IP.  Name-based virtual hosts can share a single IP address and use the HOST header in the incoming request from the browser to direct traffic to the correct virtual host.

Until relatively recently, IP-based virtual hosts were needed for any SSL-encrypted websites, but with the wider adoption of Server Name Indication (SNI), SSL-encrypted sites can use name-based virtual hosts, negating the need for multiple IP addresses.  

We will use name-based virtual hosts for our class.

### Setting up Your Virtual Host ###

A very basic virtual host can consist of just a few lines:

```
<VirtualHost *:80>
  ServerName example.duke.edu
  DocumentRoot /var/www/example.duke.edu
</VirtualHost>
```

This basically says "Listening on port 80, for any request for 'example.duke.edu', serve content from 'var/www/example.duke.edu'".  All other configurations are inherited from the global config file in /etc/httpd/conf/httpd.conf.  You'll probably want something a bit more individual, though.

Copy the contents of the `http_vhost.tmpl` file from the git repository you cloned into a new vhost file: /etc/httpd/conf.d/example.conf

Looking at that file, let's look at each section in order:

```
ServerName example.duke.edu
ServerAlias www.example.duke.edu
ServerAlias another.example.duke.edu
```
This section tells Apache what the name of the virtual host is - ie. the URL at which your site will be viewed.  The `ServerName` directive is the primary name for the site, and when the server itself references the site (e.g. the %{SERVER_NAME} variable), it will use this name.  `ServerAliases` tell Apache to listed for other, alternate names, and to direct them to this virtual host as well.

```
DocumentRoot /srv/example.duke.edu/html
```

The `DocumentRoot` is the directory where your website's files live - the root of the website.  In the example above, `/srv/example.duke.edu/html/index.html` is the file served when you browse to 'http://example.duke.edu/index.html'

```
DirectoryIndex index.php index.html
```

The DirectoryIndex directive tells Apache which files to look for, and in what order, to serve as the default page for a directory if it's not specified.  Using the example from above, if you browse to 'http://example.duke.edu/', you are not specifiying a file to be served.  The DirectoryIndex will tell Apache to first look for an index.php file, and if it exists, serve that, and if there is none, to try to find an index.html to serve.  

```
<Directory "/srv/example.duke.edu/html">
  Options -Indexes +SymlinksIfOwnerMatch
  AllowOverride None
</Directory>
```

The ``<Directory>`` stanza tells Apache to apply a set of rules to that directory, and each directory inside of it will inherit these rules unless specifically overridden.  In this example, we've explicitly disabled the `Indexes` option, and enabled the `SymLinksIfOwnerMatch` option.  

The `AllowOverride` directive tells Apache whether or not the `Options` directive can be overridden for a directory by using a ".htaccess" file - a common way to give website owners more control over the behavior of their site.

```
<Location "/protected">
  Order Allow,Deny
  Deny from All
</Location>
```

Unlike `<Directory>`, which referes to a **physical location on the filesystem of the webserver**, the `<Location>` stanza refers to a **URI**, or a location relative to the website itself.  In this example, it refers to "example.duke.edu/protected" (and www.example..., and another.example...).  In this stanza, we're introducing some access control.  `Order Allow,Deny` tells Apache which order to evaluate `Allow` and `Deny` directives.  In this case, it looks for any `Allow` directives.  If there are none the request is denied.  Then it looks for any `Deny` directives.  If there are any, the request is rejected.  There are other options here for both `Order`, and the `Allow` and `Deny` directives, including options to authorize by IP address, DNS name, etc., and they can be mixed and matched for a robust system of control.  Check out the documentation for mod_authz_host [https://httpd.apache.org/docs/2.2/mod/mod_authz_host.html](https://httpd.apache.org/docs/2.2/mod/mod_authz_host.html) for more information about access control.

```
CustomLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/access_log-%Y%m%d 86400" combined
ErrorLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/error_log-%Y%m%d 86400"
```

These two lines define separate logs for your virtual host separate from the base server's logs.  We'll focus on how these work in the next section.

_Note:_ Adding multiple virtual hosts is as easy as adding multiple `<VirtualHost>` stanzas, especially with name-based virtual hosts.  You can copy the virtual host template as many times as you like, modifying the specific settings (ie: wherever example.duke.edu appears in the template, and any configuration options) to suit the virtual host you're creating, and then reload the Apache service to make the changes live.

### Logging ###

Logging provides you with a lot of useful information. For one, the request logs will tell you about the request traffic coming into the server and the content it is serving in return.  The error logs are helpful in tracking down problems with the server configuration, code problems, missing pages, etc..  Together, the logs can help you improve the performance of your server, and identify malicious traffic.  In this section, we'll talk about how to setup logging, and how to rotate logs to better organize the data and prevent them from taking too much space on the server.

You can output data from the server in a huge number of formats.  You can explicitly tell Apache how the logs it writes should look, to assist with parsing or making them more human-readable.  However, Apache has been around for a very long time, and much of the world has actually standardized on Apache's log formats, so it's likely that any additional tools you use will support one of Apache's default formats. For this class, and on many webservers, the standard "Combined" log format is used:

"<IP Address> <RFC 1413 Identity Information> <UserID> <Date/time> <Requested URI> <Status Code> <Size of Response> <Referer> <User-Agent>"

An example of this request log format from the Apache website:

```
127.0.0.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326 "http://www.example.com/start.html" "Mozilla/4.08 [en] (Win98; I ;Nav)"
```

There can be multiple request logs for a single virtual host, formated in different ways to allow for different tools.

In addition to regular logs, `conditional` logs can be setup using _environmental variables_:

```
SetEnvIf Request_URI "^/lb\.html$" lb_log
CustomLog "|bin/rotatelogs /var/log/httpd/example.duke.edu/lb_log-%Y%m%d" combined env=lb_log
```

The section above illustrates checking if the REQUEST_URI is "/lb.html".  If so, it's tagged with the environment variable "lb_log".  The second line defines a specific log file to use for any request tagged as "lb_log".  This could be used to split out load balancer health checks from the regular traffic, and make it available separately.

_Note:_ More information about log formats available, and logging in general for Apache can be found at [https://httpd.apache.org/docs/2.2/logs.html](https://httpd.apache.org/docs/2.2/logs.html)

### Log Rotation ###

There is a huge amount of data sent to the server logs on a webserver.  According to Apache's documentation, every 10,000 requests or so increases the size of a log file by 1MB.  Because of this, it's important to "rotate" logs periodically, usually every 24 hours (although other rules can be set, both by time and logfile size).  It's important to note that the default log configuration doesn't necessarily apply to new logs setup with each virtual host.  Our of the box (on RHEL servers at least), log rotation only happens for the base server logs in /var/log/httpd.

There are a number of different ways to achieve log rotation.  At it's very basic, log rotation consists of:

1. Stopping Apache
2. Moving existing logs (renaming)
3. Creating new, empty logs
4. Starting Apache

It's important to note that Apache will continue to write to open file handles as long as the server is running, even if you delete or move the file.  If the file is deleted without restarting Apache, the space is not freed from disk until the server is restarted.  Because of this, though, you can move the existing files and issue a _graceful_ restart (`$ service httpd graceful`) to rotate the logs without losing any existing connections (ie. transparently to users).  Apache will need to keep those files open for pending connections, though.  Apache suggests waiting five minutes before doing any further processing when rotating this way.

Easier ways of rotating logs can be achieved with other tools.  Out of the box, on RHEL servers, the `logrotate` command is used to manage the default Apache logs, and can be used for other virtual host logs as well.  

Logrotate makes use of cron to execute log rotations on a schedule, during the cron.daily tasks.  Logrotate is managed by a set of configuration files.  Let's look at "/etc/logrotate.d/httpd":

```
/var/log/httpd/*log {
    missingok
    notifempty
    sharedscripts
    delaycompress
    postrotate
        /sbin/service httpd reload > /dev/null 2>/dev/null || true
    endscript
}
```

Notice that this config is applies to all files in "/var/log/httpd" ending in "log".  The important bits here are how the files are matched on the first line, `postrotate` (and it's delimiter `endscript`, which tells logrotate to reload the Apache service after rotating the logs, and `sharedscripts`, meaning the postrotate script is only run once, after all processing, and not once per log.  Finally `delaycompress` prevents old log files from being immediately compressed (they're compressed during the next logrotate run).  This accomplishes the delay while Apache continues to write to the old log files until the established and pending connections are closed.

Logrotation can be added for the new virtual host by adding rotate commands for the /var/log/httpd/example.duke.edu/ directory.  It is also possible to list the exact names of the logs to be rotated, if desired:

```
/var/log/httpd/example.duke.edu/access.log /var/log/httpd/example.duke.edu/error.log {
...
```

There are a lot of options for how to manage logs with logrotate. [http://www.rackspace.com/knowledge_center/article/understanding-logrotate-utility](http://www.rackspace.com/knowledge_center/article/understanding-logrotate-utility) is a good write-up on more advanced features of logrotate.

Probably the most common logrotation scheme is `rotatelogs`, in combination with the `tmpwatch` command.  (Yes, I know: logrotate/rotatelogs. They are unfortunately similar.)

Rotatelogs can actually be implemented in the virtual host configuration.  Notice the examples in the template above:

```
CustomLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/access_log-%Y%m%d 86400" combined
ErrorLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/error_log-%Y%m%d 86400"
```

In this case, the logs are being _piped_ directly to the rotatelogs program.  Rotatelogs will write out a logfile to the specificed location - here appending the date in YYYYMMDD format.  Every 24 hours (86400 seconds), the logs are rotated to the next day.  Rotatelogs can also be told to rotate when the logs reach a certain size, or execute another program to handle the logs.  It is versitile and easy to setup.

Rotatelogs will not remove old logs after a certain period the way logrotate can, however.  To do that, it can be paired with `tmpwatch`, a command that does just that.  In fact tmpwatch is already handling the removal of log files in the "/tmp" directory.  It is executing a script in "/etc/cron.daily/tmpwatch". The basic usage for tmpwatch is:

```
/usr/sbin/tmpwatch <number of days to keep> <directory to clean>
```

So, to remove logs over 14 days old in your /var/log/httpd/example.duke.edu/ directory, you can setup a cron job to run:

```
/usr/sbin/tmpwatch 14d /var/log/httpd/example.duke.edu
```

There's a tmpwatch template in this repo: tmpwatch.tmpl.

_Note:_ Your tmpwatch job should be a bash script and be executable if you plan to use cron.daily to execute it.

Read more about rotatelogs and tmpwatch here:
    RotateLogs [https://httpd.apache.org/docs/2.2/programs/rotatelogs.html](https://httpd.apache.org/docs/2.2/programs/rotatelogs.html)
    tmpwatch [http://thelinuxfaq.com/100-how-to-use-tmpwatch-command-in-linux]([http://thelinuxfaq.com/100-how-to-use-tmpwatch-command-in-linux)

### Performance Tuning ###

A lot of Apache's default configuration is designed to be either easy to use, or not really configured for a modern web hosting environment, at least out of the box.  This can lead to poor performance for your application, even with a small amount of load.  However, by tuning some settings, you can easily achieve significantly better performance.

The quickest way to tune your server for best performance is to adjust the number of processes Apache uses to fit within the amount of memory available to your system.  This can be a balancing act.  If there are too few processes, visitors to your site will be stuck waiting for an open thread before the page can even begin to be rendered by their browser.  On the other hand, if there are too many processes, they can easily take up all the available memory on your server.  When this happens, Apache will start writing memory to disk. and since the disk storage is much slower than memory, this will quickly lead to very poor performance from your server.  In severe cases, this can even cause the server to become completely unresponsive.

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

The "Prefork MPM" (noted above as `prefork.c`) is a non-threaded, pre-forking web server, and is the module that's used if you don't make any changes to your Apache startup. There are threaded MPMs as well ("Worker", and in Apache 2.4 "Event").  Threaded MPMs generally offer better performance, but can only be used if other modules in use are thread-safe.  For example, mod_php, the module for handling PHP processing, is not thread safe.  In order to use a threaded MPM with PHP, the PHP processing has to be offloaded to another service using something like FastCGI.

Apache is generally pretty good at self-managing process creation, so you likely will not need to adjust the `StartServers`, `MinSpareServers`, or `MaxSpareServers` settings.  StartServers is the number of processes Apache creates on startup.  MinSpareServers and MaxSpareServers are the minimum and maximum allowed idle processes, respectively.  (Idle processes are ones that are not handling any requests.)

The more important settings to be aware of are the `MaxClients` and `ServerLimit`, and `MaxRequestsPerChild`.  MaxClients is the maximum number of processes that Apache is allowed to spawn to handle requests.  Adjusting MaxClients is how a server can be tuned to allow the maximum number of processes to handle requests, while still fitting within the available memory on the system.  `ServerLimit` sets the maximum value for MaxClients (in the Prefork MPM), and should be set close to the same number as MaxClients.  Setting it much higher will allocate shared memory that will never be used.

MaxRequestsPerChild sets how many requests a child process will handle before the process is killed off by the server and replaced with a new one.  A low number will kill child processes frequently.  This can be desirable if there are problems with memory leaks or memory retention, but it does take some resources to create new processes.

Let's see how all this comes together.  Consider this output of the `top` command:

```
$ top -M -b -n 1 -u apache
top - 10:27:41 up 625 days, 0 min,  1 user,  load average: 0.34, 0.38, 0.36
Tasks: 111 total,   1 running, 110 sleeping,   0 stopped,   0 zombie
Cpu(s):  7.6%us,  1.5%sy,  0.0%ni, 90.6%id,  0.2%wa,  0.0%hi,  0.1%si,  0.0%st
Mem:  2010.512M total, 1758.734M used,  251.777M free,  108.590M buffers
Swap: 3999.992M total,   28.770M used, 3971.223M free,  974.949M cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND            
20797 apache    15   0  402m  87m  40m S  0.0  4.3   0:06.58 /usr/sbin/httpd    
20812 apache    15   0  380m  56m  28m S  0.0  2.8   0:06.43 /usr/sbin/httpd    
20801 apache    15   0  379m  53m  27m S  0.0  2.7   0:04.19 /usr/sbin/httpd    
20793 apache    15   0  377m  52m  28m S  0.0  2.6   0:06.06 /usr/sbin/httpd    
22233 apache    15   0  441m  52m  28m S  0.0  2.6   0:05.56 /usr/sbin/httpd    
20799 apache    15   0  376m  51m  28m S  0.0  2.6   0:05.37 /usr/sbin/httpd    
20803 apache    16   0  376m  50m  27m S  0.0  2.5   0:07.06 /usr/sbin/httpd    
20795 apache    15   0  376m  50m  27m S  0.0  2.5   0:05.52 /usr/sbin/httpd    
20790 apache    15   0  376m  50m  27m S  0.0  2.5   0:05.66 /usr/sbin/httpd    
22538 apache    15   0  376m  50m  27m S  0.0  2.5   0:05.59 /usr/sbin/httpd    
20810 apache    15   0  374m  49m  26m S  0.0  2.5   0:05.59 /usr/sbin/httpd    
20814 apache    15   0  374m  49m  26m S  0.0  2.4   0:05.60 /usr/sbin/httpd    
 4700 apache    15   0  438m  47m  25m S  0.0  2.4   0:01.28 /usr/sbin/httpd    
 4704 apache    15   0  373m  47m  26m S  0.0  2.4   0:01.33 /usr/sbin/httpd
 ```

The important things to see here are the total memory available to the system (`Mem: 2010.512M`), and the general Resident Size of the Apache processes (represented by the `RES` column).  The resident size is an accurate representation of the actual amount of system memory used by the process.  Notice above that the total amount of memory available to the server is 2G, and the average resident size for the Apache processes is about 50M.  Dividing the total memory by the average resident size gives us an approximation of the total number of Apache threads that can be running without causing the server to swap, or write memory to disk, in this case 40 (2000M / 50M per process = 40 processes).  In reality, though, the server will need memory for other things, unrelated to Apache.  It's helpful to reserve about 256M for other processes, so a better number of MaxClients for this server is 35 ( 1744M / 50M per process = 34.88 processes).

With that information, we can decide how to effectively tune Apache on the server:

```
<IfModule prefork.c>
StartServers       8
MinSpareServers    5
MaxSpareServers   20
ServerLimit       35
MaxClients        35
MaxRequestsPerChild  4000
</IfModule>
```

The setting above will prevent the server from running completely out of memory and becoming unresponsive, but 35 total processes for an application is very low.  Consider that a modern browser will often open multiple connections to a server, to download the content files in parallel.  A single user can easily use six processes per request.  For the server above, that means only six people can be browsing the site at the same time.

In this case, having a low MaxRequestsPerChild will help some.  Brand new Apache threads (using mod_php) tend to be about 10M on startup, and increase their allocated memory as needed for the requests that they handle.  However, they will never relase that memory - they continue to hang onto the allocated memory regardless of the memory requirements for subsequent requests.  In the example above, the 87m process (pid 20797), has expanded from it's default size up to 87M, probably to handle a php request with a lot of data.  It's very next request could be for an 8k favicon.ico file.  Setting a low MaxRequestsPerChild will allow that thread to die off sooner, and be replaced by a new one.  This is not a perfect solution, though.  Chances are the new thread will to something similar relatively quickly. There are only two real solutions to the problem:  add more memory, or fix the PHP code to use less.

In this case, this is a server with a WordPress website installed on it.  The developer is not particularly skilled at memory management, and the site owner cannot afford more memory, so the server's performance is poor.  The threads are tuned, however, to prevent the server from becoming completely unresponsive.

(There are other ways to handle this as well.  If this server were newer, it could be using PHP-FPM and Apache's Worker or Event MPMs.  Passing off the PHP processing to PHP-FPM is helpful, because it handles memory management better, so there can be more PHP-FPM processes than equivalent Apache+mod_php processes.  This also frees Apache up to handle only static content, making it more efficient as well, and allowing more total Apache threads.


Webserver Security
------------------

This section will focus on HTTPS (TLS/SSL encrpytion), how to generate TLS certificates, configuring secure TLS settings, and how to configure virtual hosts to use SSL certificates.  Also included in this section is installing the Shibboleth service, configuring a Shibboleth Service Provider, and registering with Duke's Shibboleth Identity Provider.

### HTTPS & TLS ###

HTTPS is HTTP over TLS or SSL. It allows you to encrypt traffic between a browser and web server, providing privacy and security for clients.  While the name "SSL" is more widely known, TLS has relatively recently supplanted SSL as the more secure protocol, and in fact there is no version of SSL that is considered secure as of the time of this writing.

Over all, the world is moving ever closer to HTTPS everywhere - not just encrypting secure data, but _all_ data, and there has been a push for more privacy and security overall.  Major companies like Google and Mozilla are securing traffic by default for all their applications.  Cloudflare is offering free HTTPS encryption between clients and their severs, and Let’sEncrypt, a new Certificate Authority, has started offering free, secure TLS certificates.

Given the processing power available to even the cheapest hosting services, it is no longer the case that TLS encryption is an expensive operation.  Given the benefits, there's really no go reason to not use it everywhere.  This section will show how to install mod_ssl, configure the default settings for a secure setup, and enable encryption for an entire website.

TLS works by generating a secret key, and a certificate request.  The key remains secret, installed on your server.  The certificate request contains information about you, your organization and your domain.   The certificate request is submitted to a Certificate Signing Authority (CA), who then creates a certificate and signs it, certifying that they have trust you are the owner of the domain requested.  They, in turn, are trusted by client's browsers, forming a trust chain between your server and your client.

### Generate a Key, Certificate Request, and Self-Signed Certificate ###

**On Your Server**

Generate a key:
```
# Generate a key, named example.duke.edu.key, with 4096 bits
$ openssl genrsa -out example.duke.edu.key 4096
Generating RSA private key, 4096 bit long modulus
.......................................................................................................................................................................................................................................................++
...........................................................++
e is 65537 (0x10001)
```

Using the key, generate a certificate request:
```
# Generate a certificate request with contact information
$ openssl req -new -nodes -key example.duke.edu.key -out example.duke.edu.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:NorthCarolina
Locality Name (eg, city) [Default City]:Durham
Organization Name (eg, company) [Default Company Ltd]:Duke University
Organizational Unit Name (eg, section) []:Co-Lab Class
Common Name (eg, your name or your server's hostname) []:example.duke.edu
Email Address []:christopher.collins@duke.edu

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

You now have a key and certificate request.  You can provide your certificate request to a certificate authority, and they will sign a certificate for you to use.  Duke University has an agreement with the Internet2 Certificate Authority, InCommon, and can provide free certificates for "duke.edu" domains.  You can request a certificate by filling out this form:

[https://duke.service-now.com/nav_to.do?uri=/com.glideapp.servicecatalog_cat_item_view.do?sysparm_id=fe0bd9dfa851a1009de75b2cea15c4c0](https://duke.service-now.com/nav_to.do?uri=/com.glideapp.servicecatalog_cat_item_view.do?sysparm_id=fe0bd9dfa851a1009de75b2cea15c4c0)

While the above process is fast, it does take up to a day or two to complete, so for this class, we're going to continue with a self-signed certificate.

**For a real service, NEVER use a self-signed certificate.  It is extremely easy to perform a man-in-the-middle attack if self-signed certificates are in use.**

For this class ONLY, generate a self-signed certificate:
```
# Generate an UNTRUSTED self-signed certificate using your csr
$ openssl x509 -req -days 90 -in example.duke.edu.csr -signkey example.duke.edu.key -out example.duke.edu.crt
Signature ok
subject=/C=US/ST=NorthCarolina/L=Durham/O=Duke Univesity/OU=Co-Lab Class/CN=example.duke.edu/emailAddress=christopher.collins@duke.edu
Getting Private key
```

You should now have a key, certificate request, and self-signed certificate:

1. example.duke.edu.key
2. example.duke.edu.csr
3. example.duke.edu.crt

### Install and Configure mod_ssl for Apache ###

Installing mod_ssl for Apache is done in a similar manner as we've done before:

```
$ sudo yum install -y mod_ssl
```

In addition to the module itself, it will place a file, `ssl.conf`, into your /etc/httpd/conf.d directory.  This is the server-wide configuration for SSL/TLS.  As with the other default configuration files, it is not in the best state initially.  Specifically, there are two **very important** changes that need to be made:  the SSL Protocol list and the SSL Cipher Suite list.

Open the ssl.conf file and modify the following lines as shown.

1. Remove the Virtual Host tags.  Most of the configurations for the ssl.conf, by default, apply only to the default vhost, as is.  Commenting out these two lines will apply them to all the virtual hosts.

```
<VirtualHost _default_:443>
```
```
</VirtualHost>
```

2. The SSLEngine should not be enabled globally, either:

Comment out the SSLEngine line:

```
SSLEngine On
```

3. Next, harden the SSL Protocols in use.

Mozilla.org maintains a page with recommended, secure SSL Protocols and Cipher Suites to use: [https://wiki.mozilla.org/Security/Server_Side_TLS#Modern_compatibility](https://wiki.mozilla.org/Security/Server_Side_TLS#Modern_compatibility)

Specifically, the Modern compatibility recommendations are the ones to use for the most secure option.  You're looking for the "Ciphersuite" and "Versions" items, for the Cipher Suite and Protocols, respectively.

In the ssl.conf, change the SSLProtocol directive to be:

```
SSLProtocol TLSv1.1 TLSv1.2
```

This enables only TLS versions 1.1 and 1.2; SSL version 1, 2 and 3 are all known to have vulnerabilities, and TLS 1.0 is officially deprecated in June 2016 due to it's own vulnerabilities.

4. Change the SSLCipherSuite to use stronger ciphers:

Change the SSLCipherSuite directive to use the ciphers recommended by Mozilla on the page above:

```
SSLCipherSuite ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK
```

We also want to tell browsers to honor the cipher order we are presenting above.  Add the line:

```
SSLHonorCipherOrder on
```

Making these changes harden the SSL configuration of your server, and make sure it's using the most secure protocols and cipher suites.  It's important to check back every few months to see if any changes have been made to the recommendations.  The Mozilla site is extremely useful for tracking changes as new vulnerabilities are discovered and new technologies are released.

### Configure an HTTPS Virtual Host ###

Within the git repo, you'll find a template for an HTTPS virtual host, named "https_vihost.tmpl".

You'll see it's identical to the http_vhost.tmpl, with the exception of the following:

```
diff http_vhost.tmpl https_vhost.tmpl
1c1
< <VirtualHost *:80>
---
> <VirtualHost *:443>
10a11,15
>   SSLEngine on
>   SSLCertificateFile /etc/pki/tls/certs/example.duke.edu.crt
>   SSLCertificateKeyFile /etc/pki/tls/private/example.duke.edu.key
>   SSLCACertificateFile /etc/pki/tls/certs/example-intermediate-CA.crt
>
21,22c26,27
<   CustomLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/access_log-%Y%m%d 86400" combined
<   ErrorLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/error_log-%Y%m%d 86400"
---
>   CustomLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/ssl_access_log-%Y%m%d 86400" combined
>   ErrorLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/ssl_error_log-%Y%m%d 86400"
```

Notice that the https_vhost file is listening on port 80 instead of port 443, but that like the http vhost, it's using name-based virtual hosting. The error logs are also  slightly modified, allowing us to separate HTTPS traffic into a different set of logs for easy troubleshooting.

The big change is this stanza:

```
SSLEngine on
SSLCertificateFile /etc/pki/tls/certs/example.duke.edu.crt
SSLCertificateKeyFile /etc/pki/tls/private/example.duke.edu.key
SSLCACertificateFile /etc/pki/tls/certs/example-intermediate-CA.crt
```

The `SSLEngine on` directive is explicitly turning on encryption for this virtual host.  `SSLCertificateFile` and `SSLCertificateKeyFile` tell Apache where to look for the certificate and key to use for the encryption.

Finally the `SSLCACertificateFile` is the path to the intermediate certificate for the Certificate Authority.  In some cases, the certificate authority may not have it's own intermediate certificate included in the general ca-bundle store on the server or client.  This is the case for Incommon, the certificate authority used by Internet2 and Duke.  Without the inclusion of the intermediate certificate, the certificate chain is incomplete and unable to validate the trust for the server certificate.  The `SSLCaCertfile` directive tells Apache where the intermediate cert is, in order to complete the chain.

The paths `/etc/pkl/tls/certs` and `/etc/pki/tls/private` are the standard locations for certs and keys, respectively, on RHEL servers.  

_Note:_ The certificates and key above should be in place before attempting to restart the httpd service.

### Rewriting Traffic to HTTPS ###

As mentioned above, there is ample good reason to encrypt all traffic in transit, regardless of what it is.  This can be easily accomplished with a single RewriteRule.  The example snippet below is included in the "rewrite_to_https.tmpl" file as well.

```
  RewriteEngine On
  RewriteCond %{HTTPS} !=on
  RewriteRule .* https://%{SERVER_NAME}%{REQUEST_URI} [R=301,L]
```

This is a generic snippet that can be included in any virtual host to perform a 1-to-1 redirect of all traffic to the HTTPS virtual host matching the ServerName of the virtual host.  In fact, by including this RewriteRule, the http_vhost.tmpl can be significantly reduced, since the HTTP virtual host will no longer be doing anything except redirecting:

```
<VirtualHost *:80>

  ServerName example.duke.edu
  ServerAlias www.example.duke.edu
  ServerAlias another.example.duke.edu

  RewriteEngine On
  RewriteCond %{HTTPS} !=on
  RewriteRule .* https://%{SERVER_NAME}%{REQUEST_URI} [R=301,L]

  CustomLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/access_log-%Y%m%d 86400" combined
  ErrorLog "|bin/rotatelogs -l /var/logs/httpd/example.duke.edu/error_log-%Y%m%d 86400"

</VirtualHost>
```

### Shibboleth Authentication ###

Shibboleth [https://shibboleth.net/](https://shibboleth.net/) is a centralized authentication service that can provide authentication information about individuals to applications.  The applications (either a webapp, or a service running on the server, like Apache or Tomcat) can use this information to make authorization decisions for access to protected resources.

Shibboleth is made up of two main components, an Identity Provider (IDP) run by a central authority, and a Service Provider (SP), the Shibboleth service on your server.  Duke's Office of Information Technology maintains a central IDP for the campus, which has information about all Duke people.  You will be learning how to setup an SP on your server.

For this, you will re-use the TLS certificate and key you generated for the webserver.  You can either use the same pair or generate a new pair.  It does not really matter, practically speaking.

First install the Shibboleth package, and enable and start the service, as done with the httpd package.  This will also install mod_shib, allowing Apache to work with the Shibboleth service.

Note the `--nogpgcheck` argument to the `yum` command.  This is acceptable only for this class.  In the real world, you should go through the process of importing the appropriate GPG keys for the repositories you're using.

```
$ sudo yum install -y shibboleth --nogpgcheck
$ sudo chkconfig shibboleth on
$ sudo service shibd start
```

After installing the package, copy your cert and key to "/etc/shibboleth" (purely for ease of use).

The primary Shibboleth configuration file is the /etc/shibboleth/shibboleth2.xml file.  It's a somewhat complicated and intimidating file, so a template is included.  This template has a default setup, using your base server name as the primary shibboleth application.

The key items here are the -

Request Mapper section:
* #HOST NAME# - the domain name of your site, ie: "example.duke.edu"

Application Defaults section:
* #ENTITY ID# - the entityID you registered with the IDP; it's suggested to use the full base URL of your site, ie: "https://example.duke.edu"
* #HOME URL# - the base url of the site, ie "https://example.duke.edu"

Credential Resolver section:
* #DEFAULT APP KEY# - the full path to your server's TLS key
* #DEFAULT APP CRT# - the full path to your server's TLS cert

It is possible to register more than one application and serve it with the same Shibboleth service.  If you would like to do this, look into adding more "<Host>" sections to the <RequestMapper>,
and adding an "<ApplicationOverride>" section for each additional host.  The Shibboleth documentation online is helpful for this, and OIT will be happy to help provide examples as well.

You can register your SP with OIT's IDP using an online regisration form: https://idms-web.oit.duke.edu/spreg/sps

By default, students do not have direct access to this form, but those of you in the class have been given access to use it.  Normally, you should contact the OIT Service Desk to work with the Identity Management team on this process.

Once at that URL , choose "New Registration".  

On this form, you'll have to use the same information as you entered in the Shibboleth2.xml file.

All fields are required, the important ones are:

* Entity - the entityID you are registering
* Certificate - the TLS certificate you are using with Shibboleth
* Assertion Consumer Service - Change the "Location" field, replacing "your.host" with your actual host.  Leave the rest alone.  This would look like: "https://example.duke.edu/Shibboleth.sso/SAML2/POSThttps://your.host/Shibboleth.sso/SAML2/POST

Registration usually takes just a moment before it is functional.

It was mentioned above that Shibboleth can send information about the users as part of the authentication process.  This is done with "Attributes".  By default, the "eduPersonScopedAffiliation" and "EduPersonPrincipleName" are sent in reply to the SP.  The former is "staff@duke.edu, or student@duke.edu, etc."  The latter, referred to as "eppn", is the scoped "netid@duke.edu".  There are other attributes that can be passed, including mail, given name, title, etc.  

For each attribute that is passed by the IDP, the Shibboleth SP needs to know how to map them to Apache Server Headers.  A good default one to use is provided in the "attribute-map.xml.tmpl".  Replace the existing attribute-map.xml file in your /etc/shibboleth directory with this one.

Let's look at an example from this file:

```
<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.6" id="eppn">
```

This is the attribute mapping for the eppn.  It's actually coming from the IDPs as "urn:oid:1.3.6.1.4.1.5923.1.1.1.6".  By setting the id as "eppn", the Apache HTTP headers will map the value for the eppn to the "eppn" variable.

Another example:

```
<Attribute name="urn:oid:0.9.2342.19200300.100.1.3" id="mail"/>
```

In this case Shibboleth will map the email address for the user to the Apache HTTP header variable "mail".

These attributes can be used by your application to identify the user and make authorization decisions based on the information.  Alternatively, you can have Apache manage authorization using Shibboleth.

The following stanza enables Shibboleth for the entire site, and requires that the authenticated user be a Duke staff member or student:

```
  <Location />
      AuthType Shibboleth
      ShibRequestSetting applicationId default
      ShibRequestSetting requireSession True
      ShibUseHeaders On
      require staff@duke.edu student@duke.edu
  </Location>
```

This is an example of a forced Shibboleth authentication.  By setting "requireSession True", when you visit the page, Apache will immediately direct you to the Shibboleth IDP to authenticate, unless you have already done so.  In this example, the "Affiliation" (staff@duke.edu, student@duke.edu) are being used for authorization.  You can use other attributes for authorization as well:

```
# Authorize based on NetID
require user netid@duke.edu netid2@duke.edu

# Authorize based on group membership
require ismemberof "urn:mace:duke.edu:groups:oit:sysadmins:all"
```

In some cases you may not want to force a user to login to Shibboleth until they attempt to login to your webapp.  In these cases, it's appropriate to use "Lazy Sessions":

```
  <Location />
      AuthType Shibboleth
      ShibRequestSetting applicationId default
      ShibRequestSetting requireSession False
      ShibUseHeaders On
      require shibboleth
  </Location>
```

Here we're setting the "requireSession False".  This leave Shibboleth active, but awaiting a signal from  your application to start the authentication process.  In this case your application will have to decide if the authentication information is or is not available, and if not, kick off a Shibboleth session.  More information about this, and how to direct users to a login url, is available in the Shibboleth documentation.  Good examples to use as a starting point are the WordPress Shibboleth plugin and the Drupal Shibboleth plugin - each handles authorization, and works with lazy sessions.  They are available free on their respective websitess.
