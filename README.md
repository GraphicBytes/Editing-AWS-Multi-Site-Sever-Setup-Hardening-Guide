- [The Stack We Are Installing](#markdown-header-the-stack-we-are-installing)
- [Part One – Installing stuff](#markdown-header-part-one-installing-stuff)
- [Part Two – Server Hardening](#markdown-header-part-two-server-hardening)
- [Part Three – Setting Up A vHost](#markdown-header-part-three-setting-up-a-vhost)
- [Part Four – Optimisation](#markdown-header-part-four-optimisation)
- [Why no Nginx?](#markdown-header-why-no-nginx)

_Last Update: **Feb 2022**_

In this guide we are going to set up a server to host multiple websites running Ubuntu 20.04 LTS. We will be installing the tools needed to host, secure and isolate multiple websites on one server and covering key performance configurations. 

This guide will need a full review after April 2022 and amended where need to respond to any changes for example a new PHP release, or a new major LTS release of the operating system.

# The Stack We Are Installing
**Apache 2.0 – PHP8.0-FPM – Maria DB**

For those familiar with server set ups and are only here for the copy and paste, you maybe asking why we aren’t going to use an Nginx reverse proxy, that’s the performative choice, right? Yes, it is, technically, but I will explain why I opt out of using Nginx at the end. I will also detail some theory with the configuration as we progress through the setup. 

# Part One – Installing stuff

**Starting point**

I’m going to assume that the server is set up and you have just connected via SSH to a fresh new operating system. The first thing we want to do is switch to super user:

    sudo su
### 

Refresh the index and check for updates

    apt update
### 

Do any updates, and say yes to all the questions with the ‘-y’ flag

    apt upgrade -y
### 

The default repositories usually have the traditional CGI versions of PHP, and while there are no problems whatsoever using this version, the FPM (Fast CGI) version is better. The FPM version of PHP uses a more modern and threaded system to render, and it consumes less system resources and memory… it can handle more requests.

Let’s add the Repos we will need. 

    apt install software-properties-common
### 
    add-apt-repository ppa:ondrej/php
### 

Hit Enter if prompted and say “yes” if asked any questions

Do a fresh index update so our new repos are included.

     apt update
### 
     apt upgrade -y
### 

Install Apache2, this will be our web server, and we add the “yes to all” -y flag.

     apt install apache2 -y
### 

Once complete, visit the IP of your server inside a web browser and you should see the Apache default Page.

We will need mpm-itk so we can assign each vHost to their own user

     apt-get install libapache2-mpm-itk
### 

     a2enmod mpm_itk <- probably already enabled
### 

Now we need to configure Apache with PHP-FPM. 

     apt install php8.0-fpm libapache2-mod-fcgid -y
### 

Enable our configuration with Apache. 

     a2enconf php8.0-fpm
### 

     a2enmod proxy_fcgi setenvif
### 

Now restart Apache so our new configs boot up with it:

     service apache2 restart
### 

We are going to come back and install some more modules and change a few things, but first we need to make sure PHP-fpm and Apache are being nice to each other.

     cd /var/www/html
### 

     nano info.php
### 

In the Nano editor use the phpinfo() function.

     <?php phpinfo(); ?>
### 

Save the file

`CRTL + O or CMD + O`

Hit enter to confirm

Exit the nano editor

`CRTL + X or CMD + X`

Let’s install some useful tools, starting with PHP Curl, used for server-to-server connections.

     apt install php8.0-curl -y
### 

GD and Imagick for server-side image processing

     apt install php8.0-gd php-imagick -y
### 

Multibyte String and Internationalization for encoding, and a spell check utilised by WordPress.

     apt install php8.0-mbstring php8.0-intl php-pspell -y
### 

Add file archiving to PHP

     apt install php8.0-zip -y
### 

Add SQL support for PHP

     apt install php8.0-mysqli -y
### 

Now we’re going to install MariaDB which will be our database server. The reason we use MariaDB over the default SQL Server installed with the command “apt install sql” is because MariaDB is the “true” SQL. By that I mean MariaDB is the open-source continuation of the original SQL. 

Keeping the history lesson short, SQL was sold with the agreement an open-source fork could continue. That fork kept the loyalty of the original community who made SQL what it was, and they continue to keep MariaDB up to speed with modern standards. But the proprietary fork of SQL has been poorly funded and neglected and lags in development.  

The bit we want specifically are MariaDB’s more modern storage engines that perform better and use less system resources… it can handle more requests.

     apt install mariadb-server -y
### 

Enable Maria DB

     systemctl enable mariadb
### 

Check the status and make sure MariaDB running

     systemctl status mariadb
### 

Look for “Active: active (running)”.

`CTRL + C or CMD + C`

`CTRL + C or CMD + C`

That double Command or Control + C should get you out of the SQL status screen if you’re stuck. 
We now need to secure our SQL server; we’ll kick start that process with:

     mysql_secure_installation
### 

`Enter current password for root (enter for none): [ENTER]`

`Set root password? [Y/n] Y`

`New password: [SOMETHING SUPER SECURE]`

`Re-enter new password: [SOMETHING SUPER SECURE]`

`Remove anonymous users? [Y/n] Y`

`Disallow root login remotely? [Y/n] Y`

`Remove test database and access to it? [Y/n] Y`

`Reload privilege tables now? [Y/n] Y`

That is our MariaDB server set up and secured. At this point the only user is Root and by default you can only use Root via SSH. Credentials for Root will not work for any web project or with any database manager like phpMyAdmin, and that’s how we will keep it.

We will come back to adding a user to our SQL server when we reach the Virtual Host section, but first we are going to Harden the server and make it more secure.

But, before we get to the server hardening, 2 things left to install, Certbot. Certbot will install and auto-renew our Let’s Encrypt SSL certificates.

     snap install core; sudo snap refresh core
### 

     snap install --classic certbot
### 

     ln -s /snap/bin/certbot /usr/bin/certbot
### 

Those 3 commands install Certbot and that’s everything we need installed on the system.

**This is a good time to get a tea or coffee because the next part is the one typo, and you brick the server part.**


# Part Two – Server Hardening

I know that some people would probably do this bit first before then installing PHP etc, but after bricking a few servers myself I’ve found that it can be helpful to have that front-end output available. It’s an extra point of confirmation to find out if the server is alive or not. For example, it might show you the Apache default page but reject SSH connections, this is handy feedback if you’re debugging and accidently screw up the SSH ciphers you’re going to be updating during this part of this guide… if this does happen by the way, there’s no way back, we start again 😊

The first thing we’re going to do is amend the server’s IP tables. These are a set of rules that decide what ports the server will accept connections on. On AWS this step is somewhat redundant as the server’s connection types need to be enabled/disabled within AWS, BUT!!! Our pen testers like to ignore the AWS IP firewall during their internal testing and want to see these rules set in order to give us a pass. If we do move host however, these rules would then become a vital step for server security.

First, we need to install a tool to make our changes here persistent upon reboot

    apt install iptables-persistent -y
### 

`Save current IPv4 rules? Yes`

`Save current IPv6 rules? Yes`

From this point forward, DO NOT CLOSE YOUR SSH CONNECTION! If you disconnect during the following steps, there will be no way to connect to the server and it will need a full operating system reinstall.

We first want to reject all connections by default, that way we only open what the server needs. To do that punch in the following:

    iptables -A INPUT -i lo -j ACCEPT
### 

    iptables -A OUTPUT -o lo -j ACCEPT
### 

We want the system to maintain established connection information in memory:

    iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
### 

    iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED -j ACCEPT
### 

    iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
### 

We need SFTP and SSH access. For this one we need to set rules for the sport and dport

    iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
### 

    iptables -A INPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT
### 

    iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT
### 

    iptables -A OUTPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
### 

Next comes our HTTP/HTTPS ports

    iptables -A INPUT -p tcp -m multiport --dports 80,443,8080 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
### 

    iptables -A OUTPUT -p tcp -m multiport --dports 80,443,8080 -m conntrack --ctstate ESTABLISHED -j ACCEPT
### 

Those are enough to save knowing we’ll be able to reconnect via SSH and SFTP, and the ports to accept web requests are open. Every other port will be closed unless we explicitly add rules to open them.

Let’s save that so the server boots with those rules enabled.

    netfilter-persistent save
### 

THIS NEXT BIT IS VERY IMPORTANT! DO NOT CLOSE YOUR SSH WINDOW!!!

Open a new Terminal or Command Prompt window and connect to the server via SSH, if you connect without any problem, we haven’t bricked the server.

For some reason I’ve noticed that the reboot after these rule updates is an extra-long reboot, and it might scare you into thinking you’ve bricked it. So, if the server doesn’t let you log in after about nighty seconds, give it 5 minutes then try again.

    reboot
### 

Reconnect via SSH and make sure the IP Table rules loaded with the reboot and remember to go back to super user.

    sudo su
### 

    iptables -L -v
### 

Now we’re moving onto Fail-2-ban, this adds to the built-in security of the operating system and is especially effective against botnet and guessing attacks.

An interesting article I read on this subject pointed out something about the false sense of security with using a keypair file. What I mean is that if you consider the content of the key file, it’s a sting of characters that could eventually be guessed correctly.

What fail-2-ban will do is make the highly improbable near on impossible.

    apt install fail2ban -y
### 

With that installed we need to set up the configs for it.

    cd /
### 

    cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
### 

Fail-to-ban is good to go out the box, but if you do need to add any custom configurations, you would make them in the jail.conf file you just created, but we don’t need to add any as the default is already very strict. 

From now on Fail-2-Ban will look out for bad behaviour and block IP addresses with measures that escalate exponentially in severity. This means a dev making an honest mistake will find themselves locked out for about 10 minutes, but a bot attempting a guessing attack will quickly find themselves blocked for years.

Moving on, we have one last scary part with potential to brick the server.

We need to disable weak SSH ciphers, and all it takes here is one typo and we have a brick to reset.

    nano /etc/ssh/sshd_config
### 

At the bottom of this file, add the following lines:

    ciphers aes128-ctr,aes128-gcm@openssh.com,aes192-ctr,aes256-ctr,aes256-gcm@openssh.com
    MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-256,hmac-sha2-512-etm@openssh.com,hmac-sha2-512
    KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group14-sha256
    HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ssh-rsa
### 

These directives instruct the server to use a series of ciphers that make our server more secure. I’ll admit now that I have no idea how this works. The boffins said to use these, and when we did, we passed our pen test… so just use these until the cryptography experts say otherwise. 
Save the config:

`CMD + O or CTRL + O`

Hit enter to confirm

`CMD + X or CTRL + X`

    reboot
### 

Connecting after this reboot might present a warning, the server will now be using one of the new ciphers which will make your Terminal or Command Prompt treat this as suspicious.

On a windows machine you can clear this warning with the following:

`$ ssh-keygen -R SERVER-IP` <- do this on your local machine

I’m not sure what the Mac equivalent is if there is an error, but get around it and log back in via SSH.

We need to make sure we can restrict shell access the way we want too by making sure the rbash system link is set. 

    ln -s /bin/bash /bin/rbash <- will probably fail but it’s ok
### 

If this returns a failed message saying “file exists” it’s ok, if that file didn’t exist you would have just created it. But, either way it’s there and we can move on,

Our server is now hardened, and the weakest point to attack will now be our own code, but we are great coders so that’s not a problem 😊 

# Part Three – Setting Up A vHost

If you’re new to setting up a server, a vHost is a set of per-domain directives that may look familiar as a lot of them can also be used inside .htaccess files. It’s one vHost for each domain and we are going to cover setting those up. 

Performance Note
One thing to consider with regards to server performance is that if you’re hosting a very busy website, even if only in short spikes, but enough traffic to cause your server to reach 80% or higher utilisation, you can get a small but noticeable performance boost by moving any directives inside .htaccess files and placing them directly in the Apache level vhost config.

This will spare some I/O pressure not needing to read .htaccess files on each request. But, this is a tiny boost, and this would be one of the last performance optimisations made after other optimisations were exhausted.


Setting up “devserver.bright-publishing.com“

Obviously replace devserver.bright-publishing.com with the domain you’re using.

We need a unique username, for this I’ll use the subdomain “devserver” as the username, replace it with your own.

The default Apache root directory is /var/www/html/, but were giving all project their own individual root directories.

    sudo su
### 

    cd /
### 

I don’t think there is a right and wrong way to structure your root directories, but my personal preference and a good way to keep navigation easy when hosting multiple projects is to structure it like so.

/var/www/html/ <- default

/var/www/example.com/www/public/ <- files for example.com & www.example.com

/var/www/example.com/subdomain/public/  <- files for subdomain.example.com

/var/www/bright-publishing.com/www/public/ <- files for bright-publishing & www.bright-publishing

/var/www/bright-publishing.com/gumbo/public/ <- files for gumbo.bright-publishing

With that structure in mind, and I hope you agree it makes sense, /var/www/bright-publishing.com/devserver/ will be the root directory for this vHost example.

Create the directories and user for our vHost with the following, amend where appropriate.

    mkdir -p /var/www/bright-publishing.com/devserver/public
### 

    mkdir -p /var/www/bright-publishing.com/devserver/.ssh
### 

    touch /var/www/bright-publishing.com/devserver/.ssh/authorized_keys
### 

    useradd -d /var/www/bright-publishing.com/devserver devserver
### 

Now lets give our new user the permissions it needs.

    chown -R devserver:devserver /var/www/bright-publishing.com/devserver 
### 

    chmod 700 /var/www/bright-publishing.com/devserver/.ssh
### 

    chmod 644 /var/www/bright-publishing.com/devserver/.ssh/authorized_keys
### 

We’re going to create a new keypair for our user devserver, to do this we need to jump over to that user.

    su - devserver
### 

    ssh-keygen -b 4096 -f devserver -t rsa
### 

`Enter file in which to save the key: [ENTER]` <- default location is fine

`Enter passphrase (empty for no passphrase): [ENTER]`

`Enter same passphrase again: [ENTER]`

If you only saw 2 of those 3 prompts, it’s ok, ignore and carry on.

In the /var/www/bright-publishing.com/devserver directory there will be 2 files “devserver” and “devserver.pub”. The first file contains what we need for our new .pem key file.

On your local machine create a file called devserver.pem and have it ready

First thing we will do is rename that file

    mv devserver devserver.pem
### 

You won’t have permission to download that file with FileZilla using the default SSH login, so you need to copy and paste the contents into your local file. To view the pem contents: 

    cd /var/www/bright-publishing.com/devserver
### 

    cat devserver.pem
### 

Copy everything in the output to your local devserver.pem file, including the --BEGIN-- and --END-- parts
.Once done, keep it safe as that is now the key for our new devserver user and will be used to isolate this vHost. But there is more to do.
We need to make a note of the devserver.pub file contents

    cd /
### 

    cd /var/www/bright-publishing.com/devserver
### 

    cat devserver.pub
### 

Copy and paste the contents somewhere like notpad.exe

    nano .ssh/authorized_keys
### 

Put what we just copied into this file

`CMD + O or CTRL + O`

Hit enter to confirm

`CMD + X or CTRL + X`

    exit
### 

    sudo su
### 

    service ssh restart
### 

    service sshd restart
### 

Using that new devserver.pem file and the username devserver you can now log into the server, but you only have privileges to alter files within the /var/www/bright-publishing.com/devserver/ root directory we set.

    reboot
### 

After that reboot check to make sure your new key file works via SFTP, and log back into the server.

    sudo su
### 

We can safely delete those 2 left over files

    rm /var/www/bright-publishing.com/devserver/devserver.pem
### 

    rm /var/www/bright-publishing.com/devserver/devserver.pub
### 

Our new user will come in really handy as it will let us restrict our vHost, so in the unlikely event of a successful hack, the attacker will be confined and unable to get to the rest of the system.

Lets move on

    cd /etc/apache2/sites-available
### 

We are going to create a brand-new config file

    nano devserver.bright-publishing.com.conf
### 

Copy the following into the nano editor

    <VirtualHost *:80>
        
        ServerName devserver.bright-publishing.com
        ServerAdmin digital@bright-publishing
        DocumentRoot /var/www/bright-publishing.com/devserver/public

        <IfModule mpm_itk_module>
            AssignUserId devserver devserver
        </IfModule>
        
        <Directory /var/www/bright-publishing.com/devserver/public>
            Options -Indexes +FollowSymLinks +MultiViews
            AllowOverride All
            Require all granted
        </Directory>

        <FilesMatch \.php$>
            SetHandler "proxy:fcgi://127.0.0.1:9001"
        </FilesMatch>
    </VirtualHost>
### 

Key Notes here: 

* SetHandler "proxy:fcgi://127.0.0.1:9001" – that port ‘9001’ needs to be unique per vHost, so we use 9001 for this one, but the next vHost would use ‘9002’, then ‘9003’ etc.
* AssignUserId devserver devserver – this is the user and usergroup, in that order. This will make Apache create processes as this user especially and exclusively for our vHost, isolated from all others.
* -Indexes – this stops public listing of directories.
* AllowOverride All – this enables use of .htaccess files.
* Don’t leave trailing forward slash / at the end of the directories

`CMD + O or CTRL + O`

Hit enter to confirm

`CMD + X or CTRL + X`

enable the site in Apache.

    a2ensite devserver.bright-publishing.com.conf
### 

    systemctl reload apache2
### 

We need PHP to create some processes that are accessible to devserver user.

    cd /etc/php/8.0/fpm/pool.d
### 

    nano devserver.conf
### 

We will revisit this file during the performance part of the guide, but for now copy the following into the nano editor. 

    [devserver]
        user = devserver
        group = devserver
        listen = 127.0.0.1:9001
        listen.owner = devserver
        listen.group = devserver
        pm = dynamic
        pm.max_children = 1000
        pm.start_servers = 2
        pm.min_spare_servers = 1
        pm.max_spare_servers = 3
### 


Notes here, it should be clear where to set your user and group, but also remember to match the listen = 127.0.0.1:9001 to pair up with the Apache vHost.

`CMD + O or CTRL + O`

Hit enter to confirm

`CMD + X or CTRL + X`

    service php8.0-fpm reload
### 

That should be our vhost set up to use it’s own isolated Apache and PHP processes.

Now we need to secure the site with an SSL certificate, if your domain changes haven’t propagated and you’re not seeing the Apache default page when visiting the domain or subdomain, wait until you can see that default page before doing the following.

Each domain only gets a certain amount of requests to Let’s Encrypt and the timeouts are in weeks, so really make sure you can visit the domain and see a result that isn’t an error. 

    certbot --apache --agree-tos --redirect --hsts --staple-ocsp --must-staple -d devserver.bright-publishing.com --email brightpublishing@gmail.com
### 

On the first certificate you’ll be asked to share the email.

`(Y)es/(N)o: N`

When visiting your domain/subdomain you should now see that it is secured and if you were on http it should have bumped you to https. Certbot will have made a copy of your vHost conf file and generated an SSL version of it. You don’t need to do anymore but remember if you’re making changes to the vHost to make them to both the file you created and the SSL version Certbot created.

Also note, that CertBot will auto renew any certificate you create this way.


# Part Four – Optimisation

[still to be written]


# Why no Nginx?

Nginx is faster at delivering media files than Apache, but you end up with a more complicated server to maintain and the benefits of using Nginx I believe are overstated, kind of.

The difference on a quiet or moderately stressed server between Apache and Nginx is literally unnoticeable to the end user. Where Nginx thrives is on a busy server, one receiving 100s or even 1000s of concurrent connections. In that circumstance an Apache server using an Nginx reverse proxy takes advantage of both Apache and Nginx’s strengths, and this delivers the best possible server performance. 

But another way to look at it is this. You’ve now reached the point where you’re busy enough to notice the performance improvements of NginX, so why aren’t you already using a CDN? That CDN will completely relive your main server from the responsibility of serving the very static files you would be using NginX for.

This is why I opt out from using Nginx, if you’re busy enough to see the benefits, you’re certainly busy enough to be using a CDN.
