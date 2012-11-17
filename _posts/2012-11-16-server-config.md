---
layout: post
title: "Server Config"
description: ""
category: 
tags: []
---
{% include JB/setup %}




### Create New VM ###
"Ubuntu 12.04 LTS MIN" as your OS. 

Set hostname and root password.



## Basic Ubuntu Setup

To set up your new server, execute the following commands.

### Set the hostname

Set the server hostname. Any name will do &mdash; just make it memorable. In this example, I chose "future".

{% highlight bash %}
echo "future" > /etc/hostname
hostname -F /etc/hostname
{% endhighlight %}

Let's verify that it was set correctly:

{% highlight bash %}
hostname
{% endhighlight %}


### Set the fully-qualified domain name

Set the <abbr title="Fully-qualified domain name">FQDN</abbr> of the server by making sure the following text is in the `/etc/hosts` file:

{% highlight bash %}
    127.0.0.1          localhost.localdomain   localhost
    127.0.1.1          ubuntu
    <your server ip>   future.<your domain>.net       future
{% endhighlight %}

It is useful if you add an A record that points from some domain you control (in this case I used "future.&lt;your domain&gt;.net") to your server IP address. This way, you can easily reference the IP address of your server when you SSH into it, like so:

{% highlight bash %}
ssh future.<your domain>.net
{% endhighlight %}

If you're curious, you can [read more](http://en.wikipedia.org/wiki/Hosts_\(file\)) about the `/etc/hosts` file.


### Set the time

Set the server timezone:

{% highlight bash %}
dpkg-reconfigure tzdata
{% endhighlight %}

Verify that the date is correct:

{% highlight bash %}
date
{% endhighlight %}


### Update the server

Check for updates and install:

{% highlight bash %}
apt-get update
apt-get upgrade
{% endhighlight %}


## Basic Security Setup

### Create a new user

The `root` user has a lot of power on your server. It has the power to read, write, and execute any file on the server. It's not advisable to use `root` for day-to-day server tasks. For those tasks, use a user account with normal permissions.

Add a new user:

{% highlight bash %}
adduser <your username>
{% endhighlight %}

Add the user to the `sudoers` group:

{% highlight bash %}
usermod -a -G sudo <your username>
{% endhighlight %}

This allows you to perform actions that require `root` priveledge by simply prepending the word `sudo` to the command. You may need to type your password to confirm your intentions.

Login with new user:

{% highlight bash %}
exit
ssh <your username>@<your server ip>
{% endhighlight %}


### Set up SSH keys

SSH keys allow you to login to your server without a password. For this reason, you'll want to set this up on your main computer. SSH keys are very convenient and don't make your server any less secure.

If you've already generated SSH keys before (maybe for your GitHub account?), then you can skip the next step.

#### Generate SSH keys

Generate SSH keys with the following command:

{% highlight bash %}
ssh-keygen -t rsa -C "<your username>@<your username>.org"
{% endhighlight %}

When prompted, just accept the default locations for the keyfiles. Also, you'll want to choose a nice, strong password for your key. If you're on Mac, you can save the password in your keychain so you won't have to type it in repeatedly.

Now you should have two keyfiles, one public and one private, in the `~/.ssh` folder.

If you want more information about SSH keys, GitHub has a [great guide](https://help.github.com/articles/generating-ssh-keys).

#### Copy the public key to server

Now, copy your public key to the server. This tells the server that it should allow anyone with your private key to access the server. This is why we set a password on the private key earlier.

From your local machine, run:

{% highlight bash %}
scp ~/.ssh/id_rsa.pub <your username>@<your server ip>:
{% endhighlight %}

On your Linode, run:

{% highlight bash %}
mkdir .ssh
mv id_rsa.pub .ssh/authorized_keys
chown -R <your username>:<your username> .ssh
chmod 700 .ssh
chmod 600 .ssh/authorized_keys
{% endhighlight %}


### Disable remote root login and change the SSH port

Since all Ubuntu servers have a `root` user and most servers run SSH on port 22 (the default), criminals often try to guess the `root` password using automated attacks that try many thousands of passwords in a very short time. This is a common attack that nearly all servers will face.

We can make things substantially more difficult for automated attackers by preventing the `root` user from logging in over SSH and changing our SSH port to something less obvious. This will prevent the vast majority of automatic attacks.

Disable remote root login and change SSH port:

{% highlight bash %}
sudo nano /etc/ssh/sshd_config
Set "PermitRootLogin no"
Set "Port 44444"
sudo service ssh restart
{% endhighlight %}

In this example, we changed the port to 44444. So, now to connect to the server, we need to run:

{% highlight bash %}
ssh <your username>@future.<your domain>.net -p 44444
{% endhighlight %}

Update: Someone posted this useful note about choosing an SSH port on Hacker News:

> Make sure your SSH port is below 1024 (but still not 22). Reason being if your Linode is ever compromised a bad user may be able to crash sshd and run their own rogue sshd as a non root user since your original port is configured >1024. (More info [here](http://unix.stackexchange.com/questions/16564/why-are-the-first-1024-ports-restricted-to-the-root-user-only))


## Advanced Security Setup



### Add a firewall

We'll add an [iptables](http://en.wikipedia.org/wiki/Iptables) firewall to the server that blocks all incoming and outgoing connections except for ones that we manually approve. This way, only the services we choose can communicate with the internet.

The firewall has no rules yet. Check it out:

{% highlight bash %}
sudo iptables -L
{% endhighlight %}

Setup firewall rules in a new file:

{% highlight bash %}
sudo nano /etc/iptables.firewall.rules
{% endhighlight %}

The following firewall rules will allow HTTP (80), HTTPS (443), SSH (44444), ping, and some other ports for testing. All other ports will be blocked.

Paste the following into `/etc/iptables.firewall.rules`:

{% highlight bash %}
*filter

#  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT

#  Accept all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

#  Allow all outbound traffic - you can modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

#  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

#  Allow ports for testing
-A INPUT -p tcp --dport 8080:8090 -j ACCEPT

#  Allow ports for MOSH (mobile shell)
-A INPUT -p udp --dport 60000:61000 -j ACCEPT

#  Allow SSH connections
#  The -dport number should be the same port number you set in sshd_config
-A INPUT -p tcp -m state --state NEW --dport 44444 -j ACCEPT

#  Allow ping
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

#  Log iptables denied calls
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

#  Reject all other inbound - default deny unless explicitly allowed policy
-A INPUT -j REJECT
-A FORWARD -j REJECT

COMMIT
{% endhighlight %}

Activate the firewall rules now:

{% highlight bash %}
sudo iptables-restore < /etc/iptables.firewall.rules
{% endhighlight %}

Verify that the rules were installed correctly:

{% highlight bash %}
sudo iptables -L
{% endhighlight %}

Activate the firewall rules on startup:

{% highlight bash %}
sudo nano /etc/network/if-pre-up.d/firewall
{% endhighlight %}

Paste this into the `/etc/network/if-pre-up.d/firewall` file:

{% highlight bash %}
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.firewall.rules
{% endhighlight %}

Set the script permissions:

{% highlight bash %}
sudo chmod +x /etc/network/if-pre-up.d/firewall
{% endhighlight %}



## Improve Server Stability

VPS servers can easily run out of memory during traffic spikes.

For example, most people don't change Apache's default setting which allows 150 clients to connect simultaneously. This is way too large a number for a typical VPS server. Let's do the math. Apache's processes are typically ~25MB each. If our website gets a temporary traffic spike and 150 processes launch, we'll need 3750MB of memory on our server. If we don't have this much (and we don't!), then the OS will grind to a halt as it swaps memory to disk to make room for new processes, but then immediately swaps the stuff on disk back into memory.

No useful work gets done once swapping happens. The server can be stuck in this state for hours, even after the traffic rush has subsided. During this time, very few web requests will get serviced.

It's very important to configure your applications so memory swapping does not occur. If you use Apache, you should set `MaxClients` to something more reasonable like 20 or 30. There are many other optimizations to make, too. Linode has a Library article with [optimizations](http://library.linode.com/hosting-website) for Apache, MySQL, and PHP.


### Reboot server on out-of-memory condition

Still, in cases where something goes awry, it is good to automatically **reboot your server when it runs out of memory**. This will cause a minute or two of downtime, but it's better than languishing in the swapping state for potentially hours or days.

You can leverage a couple kernel settings and Lassie to make this happen on Linode.

Adding the following two lines to your `/etc/sysctl.conf` will cause it to reboot after running out of memory:

{% highlight bash %}
vm.panic_on_oom=1
kernel.panic=10
{% endhighlight %}

The vm.panic_on_oom=1 line enables panic on OOM; the kernel.panic=10 line tells the kernel to reboot ten seconds after panicking.

[Read more](http://www.linode.com/wiki/index.php/Rebooting_on_OOM) about rebooting when out of memory on Linode's wiki.




## Install Useful Server Software

At this point, you have a pretty nice server setup. Congrats! But, your server still doesn't do anything useful. Let's install some software.


### Install a compiler

A compiler is often required to install Python packages and other software, so let's just install one up-front.

{% highlight bash %}
sudo aptitude install build-essential
{% endhighlight %}


### Install MySQL

Install MySQL:

{% highlight bash %}
sudo aptitude install mysql-server libmysqlclient-dev
{% endhighlight %}

Set root password when prompt asks you.

Verify that MySQL is running.

{% highlight bash %}
sudo netstat -tap | grep mysql
{% endhighlight %}

For connecting to MySQL, instead of the usual PHPMyAdmin, I now use [Sequel Pro](http://www.sequelpro.com/), a free app for Mac.

#### Improve MySQL security

Before using MySQL in production, you'll want to improve your MySQL installation security. Run:

{% highlight bash %}
mysql_secure_installation
{% endhighlight %}

This will help you set a password for the root account, remove anonymous-user accounts, and remove the test database.


#### Keep your MySQL tables in tip-top shape

Over time your MySQL tables will get fragmented and queries will take longer to complete. You can keep your tables in top shape by regularly running [OPTIMIZE TABLE](http://dev.mysql.com/doc/refman/5.1/en/optimize-table.html) on all your tables. But, since you'll never remember to do this regularly, we should set up a cron job to do this.

Open up your crontab file:

{% highlight bash %}
crontab -e
{% endhighlight %}

Then, add the following line:

{% highlight bash %}
@weekly mysqlcheck -o --user=root --password=<your password here> -A
{% endhighlight %}

Also, you can try manually running the above command to verify that it works correctly.


#### Backup your MySQL databases

The excellent `automysqlbackup` utility can automatically make daily, weekly, and monthly backups of your MySQL database.

Install it:

{% highlight bash %}
sudo aptitude install automysqlbackup
{% endhighlight %}

Now, let's configure it. Open the configuration file:

{% highlight bash %}
sudo nano /etc/default/automysqlbackup
{% endhighlight %}

By default, your database backups get stored in `/var/lib/automysqlbackup` which isn't very intuitive. I recommend changing it to a folder within your home directory. To do this, find the line that begins with `BACKUPDIR=` and change it to `BACKUPDIR="/home/<your username>/backups/"`

You also want to get an email if an error occurs, so you'll know if automatic backups stop working for some reason. Find the line that begins with `MAILADDR=` and change it to `MAILADDR="<your email address>"`.

Close and save the file. That's it!

# Install LAMP #

## Installing Apache ##

Install Apache on your Linode by entering the following command: ::

    sudo apt-get install apache2

Your Linode will download, install, and start the Apache web server.

Optimizing Apache for a Linode 512
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Installing Apache is easy, but if you leave it running with the default settings, your server could run out of memory. That's why it's important to optimize Apache *before* you start hosting a website on your Linode. Here's how to optimize the Apache web server for a Linode 512:

.. note:: These guidelines are designed to optimize Apache for a Linode 512, but you can use this information for any size Linode. The values are based on the amount of memory available, so if you have a Linode 1024, multiply all of the values by 2 and use those numbers for your settings.

1. Just to be safe, make a copy of Apache's configuration file by entering the following command. You can restore the duplicate (``apache2.backup.conf``) if anything happens to the configuration file. ::

	sudo cp /etc/apache2/apache2.conf /etc/apache2/apache2.backup.conf

2. Open Apache's configuration file for editing by entering the following command: ::

	sudo nano /etc/apache2/apache2.conf
	
3. Make sure that the following values are set:

.. colorize:: apache

	<IfModule mpm_prefork_module>
	StartServers 1
	MinSpareServers 3
	MaxSpareServers 6
	MaxClients 24
	MaxRequestsPerChild 3000
	</IfModule>
	
4. Save the changes to Apache's configuration file by pressing ``Control`` + ``x`` and then pressing ``y``.

5. Restart Apache to incorporate the new settings. Enter the following command: ::

	sudo service apache2 restart

Good work! You've successfully optimized Apache for your Linode, increasing performance and implementing safeguards to prevent excessive resource consumption. You're almost ready to host websites with Apache.

Configuring Name-based Virtual Hosts
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now that Apache is optimized for performance, it's time to starting hosting one or more websites. There are several possible methods of doing this. In this section, you'll use *name-based virtual hosts* to host websites in your home directory. Here's how:

.. note:: You should *not* be logged in as ``root`` while executing these commands. To learn how to create a new user account and log in as that user, see `Adding a New User </securing-your-server#sph_adding-a-new-user>`_. 

1. Disable the default Apache virtual host by entering the following command: ::

	sudo a2dissite default
	
2. Navigate to your home directory by entering the following command: ::

	cd ~
	
3. Create a folder to hold your website by entering the following command: ::

	mkdir public
	
4. Create a set of folders inside ``public`` to store your website's files, logs, and backups. Enter the following command, replacing ``example.com`` with your domain name: ::

	mkdir -p public/example.com/{public,log,backup} 
	
5. Set your home directory to be readable and accessible to all users on the system by entering the following command: ::

	sudo chmod a+rx ~
	
6. Set the ``public`` directory (and all of the files in it) to be readable and accessible to all users on the system by entering the following command: ::

	sudo chmod -R a+rx ~/public

7. Create the virtual host file for your website by entering the following command. Replace ``example.com`` with your domain name: ::

	sudo nano /etc/apache2/sites-available/example.com

8. Now it's time to create a configuration for your virtual host. We've created some basic settings to get your started. Copy and paste the settings shown below in to the virtual host file you just created. Replace ``example_user`` with your username, and ``example.com`` with your domain name.

.. colorize:: apache

	# domain: example.com
	# public: /home/example_user/public/example.com/

	<VirtualHost *:80>
	  # Admin email, Server Name (domain name), and any aliases
	  ServerAdmin webmaster@example.com
	  ServerName  www.example.com
	  ServerAlias example.com
	  
	  # Index file and Document Root (where the public files are located)
	  DirectoryIndex index.html index.php
	  DocumentRoot /home/example_user/public/example.com/public

	  # Log file locations
	  LogLevel warn
	  ErrorLog  /home/example_user/public/example.com/log/error.log
	  CustomLog /home/example_user/public/example.com/log/access.log combined
	</VirtualHost>
	
9. Save the changes to the virtual host configuration file by pressing ``Control + x`` and then pressing ``y``.
	
10. Create a symbolic link to your new ``public`` directory by entering the following command. Replace ``example.com`` with your domain name: ::

	    sudo a2ensite example.com
	
11. Restart Apache to save the changes. Enter the following command: ::	
 
		sudo service apache2 restart

12. Repeat steps 1-11 for every other website you want to host on your Linode.
	
Congratulations! You've configured Apache to host one or more websites on your Linode. After you `upload files <#sph_id3>`_ and `add DNS records <#sph_adding-dns-records>`_ later in this guide, your websites will be accessible to the outside world.

Database
--------

Databases store data in a structured and easily accessible manner, serving as the foundation for hundreds of web and server applications. A variety of open source database platforms exist to meet the needs of applications running on your Linux VPS. This section will help you get started with *MySQL*, one of the most popular database platforms.

Installing MySQL
~~~~~~~~~~~~~~~~

Here's how to install and configure MySQL:

1. Install MySQL by entering the following command. Your Linode will download, install, and start the MySQL database server. ::

	sudo apt-get install mysql-server
	
2. You will be prompted to enter a password for the MySQL ``root`` user. Enter a password.
	
3. Secure MySQL by entering the following command to open ``mysql_secure_installation`` utility::

	sudo mysql_secure_installation
	
4. The ``mysql_secure_installation`` utility appears. Follow the instructions to remove anonymous user accounts, disable remote root login, and remove the test database. 

That's it! MySQL is now installed and running on your Linode.

Optimizing MySQL for a Linode 512
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MySQL consumes a lot of memory when using the default configuration. To set resource constraints, you'll need to edit the MySQL configuration file. Here's how to optimize MySQL for a Linode 512:

.. note:: These guidelines are designed to optimize MySQL for a Linode 512, but you can use this information for any size Linode. If you have a larger Linode, start with these values and modify them while carefully watching for memory and performance issues.

1. Open the MySQL configuration file for editing by entering the following command: ::

	sudo nano /etc/mysql/my.cnf
	
2. Make sure that the following values are set:

.. colorize:: ini

    max_connections = 75
    key_buffer = 16M
    max_allowed_packet = 1M
    thread_stack = 64K
    table_cache = 32

3. Save the changes to MySQL's configuration file by pressing ``Control + x`` and then pressing ``y``.
	
4. Restart MySQL to save the changes. Enter the following command: ::	
 
	sudo service mysql restart
	
Now that you've edited the MySQL configuration file, you're ready to start creating and importing databases.

Creating a Database
~~~~~~~~~~~~~~~~~~~

The first thing you'll need to do in MySQL is create a *database*. (If you already have a database that you'd like to import, skip to `Importing a Database <#sph_id1>`_.) Here's how to create a database in MySQL:

1. Log in to MySQL by entering the following command and then entering the MySQL root password: ::

	mysql -u root -p
	
2. Create a database and grant a user permission to it by entering the following command. Replace ``exampleDB`` with your own database name: ::

	create database exampleDB;

3. Create a new user in MySQL and then grant that user permission to access the new database by issuing the following command. Replace ``example_user`` with your username, and ``5t1ck`` with your password: ::
	
	grant all on exampleDB.* to 'example_user' identified by '5t1ck';
	
.. note:: MySQL usernames and passwords are only used by scripts connecting to the database. They do not need to represent actual user accounts on the system.	

4. Tell MySQL to reload the grant tables by issuing the following command: ::

	flush privileges;
	
5. Now that you've created the database and granted a user permissions to the database, you can exit MySQL by entering the following command: ::

	quit
	
Now you have a new database that you can use for your website. If you don't need to import a database, go ahead and skip to `PHP <#sph_id2>`_.

Importing a Database
~~~~~~~~~~~~~~~~~~~~

If you have an existing website, you may want to import an existing database in to MySQL. It's easy, and it allows you to have an established website up and running on your Linode in a matter of minutes. Here's how to import a database in to MySQL:

1. Upload the database file to your Linode. See the instructions in  `Uploading Files <#sph_id3>`_.
	
2. Import the database by entering the following command. Replace ``username`` with your MySQL username, ``password`` with your MySQL password, and ``database_name`` with your own: ::

	mysql -u username -ppassword database_name < FILE.sql
	
Your database will be imported in to MySQL.

PHP
---

PHP is a general-purpose scripting language that allows you to produce dynamic and interactive webpages. Many popular web applications and content management systems, like WordPress and Drupal, are written in PHP. To develop or host websites using PHP, you must first install the base package and a couple of modules.

Installing PHP
~~~~~~~~~~~~~~

Here's how to install PHP with MySQL support and the Suhosin security module:

1. Install the base PHP package by entering the following command: ::

	sudo apt-get install php5 php-pear

2. Add MySQL support by entering the following command: ::

	sudo apt-get install php5-mysql
	
3. Secure PHP with Suhosin by entering the following command: ::

	sudo apt-get install php5-suhosin

Optimizing PHP for a Linode 512
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After you install PHP, you'll need to enable logging and tune PHP for better performance. The setting you'll want to pay the most attention to is ``memory_limit``, which controls how much memory is allocated to PHP. Here's how to enable logging and optimize PHP for performance:

.. note:: These guidelines are designed to optimize PHP for a Linode 512, but you can use this information as a starting point for any size Linode. If you have a larger Linode, you could increase the memory limit to a larger value, like 128M.

1. Open the PHP configuration files by entering the following command: ::

	sudo nano /etc/php5/apache2/php.ini

2. Verify that the following values are set. All of the lines listed below should be uncommented. Be sure to remove any semi-colons (;) at the beginning of the lines.

.. colorize:: ini

	max_execution_time = 30
	memory_limit = 64M
	error_reporting = E_COMPILE_ERROR|E_RECOVERABLE_ERROR|E_ERROR|E_CORE_ERROR
	display_errors = Off
	log_errors = On
	error_log = /var/log/php.log
	register_globals = Off

.. note:: The 64M setting for ``memory_limit`` is a general guideline. While this value should be sufficient for most websites, larger websites and some web applications may require 128 megabytes or more. 

3. Save the changes by pressing ``Control`` + ``x`` and then pressing ``y``.

4. Restart Apache to load the PHP module by entering the following command: ::

	sudo service apache2 restart
	
Congratulations! PHP is now installed on your Linode and configured for optimal performance.

Uploading Files
---------------

You've successfully installed Apache, MySQL, and PHP. Now it's time to upload a website to your Linode. This is one of the last steps before you "flip the switch" and publish your website on the Internet. Here's how to upload files to your Linode:

1. If you haven't done so already, download and install an FTP client on your desktop computer. We recommend using `Filezilla </networking/file-transfer/transfer-files-filezilla-ubuntu-9.10>`_ on Linux systems, `Cyberduck </networking/file-transfer/transfer-files-cyberduck>`_ on Mac OS X, and `WinSCP </networking/file-transfer/transfer-files-winscp>`_ on Windows.

2. Follow the instructions in the guides listed above to connect to your Linode.

3. Upload your website's files to the ``~/public/example.com/public`` directory. Replace ``example.com`` with your domain name.

.. note:: If you configured name-based virtual hosts, don't forget to upload the files for the other websites to their respective directories. 

If you're using a content management system like WordPress or Drupal, you may need to configure the appropriate settings file to point the content management system at the MySQL database.

Testing
-------

It's a good idea to test your website(s) before you add the DNS records. This is your last chance to check everything and make sure that it looks good before it goes live. Here's how to test your website:

1. Enter your Linode's IP address in a web browser (e.g., type ``http://123.456.78.90`` in the address bar, replacing the example IP address with your own.) Your website should load in the web browser.

2. If you plan on hosting multiple websites and you're using Linux or Mac OS X, you can test the virtual hosts by editing the ``/etc/hosts`` file on your desktop computer. Add two new lines to the ``hosts`` file with your IP address and the domain names, as shown below: ::

	123.456.78.90	domain1.com
	123.456.78.90	domain2.com
	
3. Test the name-based virtual hosts by entering the domain names in the address bar of the web browser on your desktop computer. Your websites should load in the web browser.

This is important: Remember to remove the entries for the name-based virtual hosts from your ``/etc/hosts`` file before you add the DNS records. When everything looks good, you're ready to add a DNS record to point your domain name at your website!

Adding DNS Records
------------------

Now you need to point your domain name(s) at your Linode. This process can take a while, so please allow up to 24 hours for DNS changes to be reflected throughout the Internet. Here's how to add DNS records:

1. Log in to the `Linode Manager <https://manager.linode.com>`_.

2. Click the **DNS Manager** tab.

3. Select the **Add a domain zone** link. The form shown below appears.

.. image:: /assets/910-hosting-1-small.png
	:alt: Create a domain zone.
	:target: /assets/909-hosting-1.png

4. In the **Domain** field, enter your website's domain name in the **Domain** field.

5. In the **SOA Email** field, enter the administrative contact email address for your domain.

6. Select the **Yes, insert a few records to get me started** button.

7. Click **Add a Master Zone**. Several DNS records will be created for your domain, as shown below.

.. image:: /assets/911-hosting-2-small.png
	:alt: The DNS records created for the domain.
	:target: /assets/912-hosting-2.png

8. Make sure that your domain name is set to use our DNS server. Use your domain name registrar's interface to set the name servers for your domain to the following:

	- ``ns1.linode.com``
	- ``ns2.linode.com``
	- ``ns3.linode.com``
	- ``ns4.linode.com``
	- ``ns5.linode.com``

9. Repeat steps 1-8 for every other name-based virtual host you created earlier.

You've added DNS records for your website(s). Remember, DNS changes can take up to 24 hours to propagate through the Internet. Be patient! Once the DNS changes are completed, you will be able to access your website by typing the domain name in to your browser's address bar.

Setting Reverse DNS
-------------------

You're almost finished! The last step is setting reverse DNS for your domain name. Here's how:

1. Log in to the `Linode Manager <https://manager.linode.com>`_.

2. Click the **Linodes** tab.

3. Select your Linode.

4. Click the  **Remote Access** tab.

5. Select the **Reverse DNS** link, as shown below.

.. image:: /assets/951-hosting-3-1.png
	:alt: Select Reverse DNS link.
	:target: /assets/951-hosting-3-1.png

6. Enter the domain in the **Hostname** field, as shown below.

.. image:: /assets/914-hosting-4-small.png
	:alt: Enter domain in Hostname field.
	:target: /assets/915-hosting-4.png

7. Click **Look up**. A message appears indicating that a match has been found.

8. Click **Yes**. 

You have set up reverse DNS for your domain name. 