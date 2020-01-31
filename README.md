# Nextcloud Setup

## The Host (DigitalOcean)

1. Create a DigitalOcean account. Add billing details.

2. In the sidebar, go to Account > Security.

3. Add an SSH public key with "Add SSH Key". If you don't have a key, you can create one like this:

   ```shell
   $ ssh-keygen -t rsa -C <your@email.com>
   ```

   And then to get your public key contents which we need for DigitalOcean:

   ```shell
   $ cat ~/.ssh/id_rsa.pub
   ```

4. Go back to Droplets in the sidebar, and click "Create > Droplets" at the top. Select the following options:

   * In the Marketplace tab for "Choose an Image", choose Docker on Ubuntu 18.04 (or a later LTS release like 20.04, but at time of writing the latest is 18.04). If it doesn't appear in "Recommended for you", search for it by clicking "See all Marketplace Apps".
   * Under "Choose a Plan", choose "Standard" and select the $10/mo option with 2GB RAM and 1 CPU.
   * Which datacenter you choose is personal preference, but will affect performance depending on where the expected visitors are located.
   * Under "Select Additional Options", check "Monitoring"
   * For "Authentication", select "SSH Keys", and select the SSH key you saved in step 3.
   * Choose a sensible hostname for the droplet. For small projects, I like the simple naming convention of "sites" for a single droplet containing websites.
   * Finally, at the end, check the box to "Enable backups". This will cost an extra 20% on top of the droplet price, but it will take image snapshots of the instance every week in case of accidents or failure.
   * Create the Droplet

## Setting up Ubuntu

### User Setup

When your Ubuntu droplet spins up, it will have some default configurations we want to change, and software we need to install.

1. Using the IP address provided for your Droplet, the login looks like this:

   ```shell
   $ ssh root@123.456.78.90
   ```

   However, we can make it a little more convenient by naming the droplet with an alias on our system.

   ```shell
   $ sudo vim /etc/hosts
   ```

   In this file, add a new line with the IP address of your droplet, and then an alias for that IP, like so:

   ```
   138.197.49.194 sites
   ```

   In this case, I've added a line that will alias my droplet IP as `sites`. Now at the command line we can reference the droplet this way instead of copying its IP address all the time. So our SSH command looks like:

   ```shell
   $ ssh root@sites
   ```

   Execute this to login to the droplet.

2. We don't want to allow log in as root, but we do want administrative privileges, so first we'll create another user. Note that `username` in the commands below should be replaced with the name you choose for your user:

   ```shell
   $ adduser username
   ```

   After providing a password when prompted, the rest of the information is optional. Just hit "Enter" until you're through it all.

3. Add the new user to the `sudo` group to give it access to administrative commands:

   ```shell
   $ usermod -aG sudo username
   ```

4. While we're at it, add the user to the `docker` group, too. This will give it the ability to run docker commands without needing to `sudo` at all:

   ```shell
$ usermod -aG docker username
   ```
   
   
   
5. To allow us to use `sudo` on the new user without having to type a password all the time, edit the sudoers file with `sudo visudo`.

   There will be a section in the opened file that looks like:

   ```shell
   # Allow members of group sudo to execute any command
   %sudo	ALL=(ALL:ALL) ALL
   ```

   Change the line defining the sudo group privileges to this:

   ```shell
   %sudo	ALL=(ALL) NOPASSWD: ALL
   ```

   Save the file.

6. Set up SSH access for the new user. Since the SSH configuration for us to log in as root is already working, we can just copy it to our new user's home directory:

   ```shell
   $ cp -r /root/.ssh/ /home/username/
   ```

   At this point, you can test the SSH login in another terminal window from your host:

   ```shell
   $ ssh username@sites
   ```

   And make sure you can run administrative commands:

   ```shell
   $ sudo -l
   ```

   If the above commands work, you're good to go!

7. Now it's time to remove the ability to SSH in as root. <u>**IMPORTANT:**</u> **Before executing this step, make sure you have still have an open and working terminal session, and _do not close it_**. We're going to change SSHd settings, and if anything goes wrong, you could lose the ability to log in again. So keep your current terminal session(s) handy until it's done.

   To remove root login ability, edit the `sshd_config` file:
   ```shell
   $ sudo vim /etc/ssh/sshd_config
   ```

   You'll be looking for a line which looks like this:

   ```shell
   PermitRootLogin yes
   ```

   Change it to this:

   ```shell
   PermitRootLogin no
   ```

   And add one more line explicitly allowing the login of your user:

   ```
   AllowUsers username
   ```

   Save and exit the editor, and finally, restart the SSH daemon:

   ```shell
   $ sudo service ssh restart
   ```

   From a new terminal in your host, make sure you can still SSH in as your user:

   ```shell
   $ ssh username@sites
   ```

   And make sure you _cannot_ SSH in as root:

   ```shell
   $ ssh root@sites
   ```

### Installing Packages

Docker and Docker Compose are already installed on this system, so we don't need to worry about getting those. We will be installing a couple things, however. 

7. As a reverse proxy, we'll be installing the webserver Nginx, and git so that we can clone configs we'll use in later steps. 

```shell
$ sudo apt install nginx git
```

8. We also want Let's Encrypt's certbot to help us configure SSL certificates later:

```shell
$ sudo add-apt-repository ppa:certbot/certbot && sudo apt install python-certbot-nginx
```

## Setting up the Site

To install and manage Nextcloud and its dependent services, instead of installing all the software locally, we'll use Docker containers.

9. To start, clone the repository found at https://github.com/miend/nextcloud-nginx-docker to your user's home directory:

```shell
$ cd ~ && git clone git@github.com:miend/nextcloud-nginx-docker.git
```

This contains configurations for our docker-compose setup and environment variables that should be set for the database (in `db.env`, you should set custom values for `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD`). Once you've made the desired changes, move on to step 10.

10. We'll be using an nginx setup that exists outside of docker-compose so that we could potentially serve other sites and services from this same droplet later, if desired. In the end (in my opinion), it simplifies things.

    Start with opening a file in nginx's `sites-available` directory, named with whatever makes the file easy to identify. I like to use the name of the site being served, like so:

    ```shell
    $ sudo vim /etc/nginx/sites-available/my.domain.com
    ```

    Where `my.domain.com` is the domain name you've reserved for your site.

    In this file, paste the following configuration (again, replacing `my.domain.com` with your domain):

    ```nginx
    server {
        listen 80;
        server_name my.domain.com www.my.domain.com;
    
        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP  $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_buffering off;
            proxy_request_buffering off;
            client_max_body_size 0;
            proxy_read_timeout  36000s;
            proxy_redirect off;
            proxy_ssl_session_reuse off;
        }
    }
    ```

    Save the file. There is a lot of content in here to be explained, but it's out of the scope of this documentation. To know what any of these nginx directives do for us, you can visit their documentation at https://nginx.org/en/docs/dirindex.html

11. Create a soft link (like a shortcut in the Linux world) from this file to another directory, `sites-enabled`. The link needs to specify the _full_ path to each of the files:

    ```shell
    $ sudo ln -s /etc/nginx/sites-available/my.domain.com /etc/nginx/sites-enabled/
    ```

    And check to see if the configuration is valid according to Nginx:

    ```shell
    $ sudo nginx -t
    ```

    If so, reload the configuration!

    ```shell
    $ sudo systemctl reload nginx
    ```

12. At this point, we're ready to start some magic. Time to run:

    ```shell
    $ docker-compose up -d
    ```

    This will start `docker-compose`, which will download or build all of our individual containers configured in `docker-compose.yml`, and it will run them in the background. It will also create a docker network and some volumes containers will use.

13. We're almost there. The docker containers serving the app are running, but you might not have a domain name already pointed to your droplet's IP yet. If you do have one, skip to step 15. Otherwise, you can actually test whether the app is functional from the outside by creating an alias in your own machine's `/etc/hosts` file. :

    ```shell
    # (On your own machine)
    $ sudo vim /etc/hosts
    ```

    Now add the following line:

    ```shell
    <ip_address> my.domain.com
    ```

    Where `<ip_address>` is the IP of the droplet you created, and `my.domain.com` is your chosen domain name. Save the file. You can keep this around for as long as you're testing, but it would be a good idea to remove the line you added when you no longer need it.  

14. Now in your browser, you should be able to visit http://my.domain.com and see the setup page for Nextcloud. But don't go through the setup just yet! We want to configure SSL before transmitting any sensitive data like passwords over the network. To do this, we'll use the Let's Encrypt certbot we installed earlier. 

    **NOTE**: Unfortunately, to progress any further in this guide, you'll need to point your desired domain (the "`@`" record) or subdomain (an "`A`" record) at your DigitalOcean droplet. Since different registrars have different processes for this, you'll need to refer to your registrar's documentation on how to do it.

    To configure nginx to serve HTTPS using an SSL certificate, simply run the following command, replacing `my.domain.com` with your domain.

    ```shell
    sudo certbot --nginx -d my.domain.com -d www.my.domain.com
    ```

    Enter a valid email address when prompted (**this is important!** If your certs are about to expire and need to be manually renewed, Let's Encrypt will alert you by email). Certbot will set itself up to answer a challenge for Let's Encrypt to determine if you're actually the owner of `my.domain.com`. If your domain is properly pointed at your droplet, this challenge should succeed, and then Certbot will ask you to make the following choice:

    ```
    Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
    -------------------------------------------------------------------------------
    1: No redirect - Make no further changes to the webserver configuration.
    2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
    new sites, or if you're confident your site works on HTTPS. You can undo this
    change by editing your web server's configuration.
    -------------------------------------------------------------------------------
    Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
    ```

    Select the number 2 to have all HTTP (unencrypted) requests redirected to use HTTPS (encrypted) instead.

    After that's done, check to see if certbot's renewal feature works for your site:

    ```shell
    $ sudo certbot renew --dry-run
    ```

    If this comes back successful, then your certificate should be automatically renewed for you when it's nearing expiry. If that doesn't happen for whatever reason, Let's Encrypt helps out with an email reminder, so just be on the lookout for that.

15. Now you should be able to successfully visit https://my.domain.com and see your Nextcloud setup page once again, this time with an encrypted SSL connection. Go ahead and enter the desired login credentials for the `admin` user where prompted, and let the app finish setup for you.

    At this point, you should have a working Nextcloud installation, but there are still a couple minor tweaks.

## Post-Setup Configuration/Fixes

16. You should be able to navigate around your Nextcloud site, create users, and see content, but when I used this setup myself, there were a couple issues I encountered. Occasionally, a link within nextcloud would cause the user to redirect to a path at `localhost:8080` when clicked (instead of `my.domain.com`). Additionally, links sent to users in emails (like the "Forgot my Password" emails) would do the same thing, and the process of authorizing client apps for a user would fail due to an http protocol issue. Whoops. To prevent these issues, we can run a couple commands.

    First, use the command `docker ps` to see the status of all running containers. It should generate output like this:

    ```
    CONTAINER ID        IMAGE                     COMMAND                  CREATED             STATUS              PORTS                  NAMES
    2467d47458b4        username_web               "nginx -g 'daemon of…"   3 hours ago         Up 3 hours          0.0.0.0:8080->80/tcp   username_web_1
    44492b8a16dc        nextcloud:18-fpm-alpine   "/entrypoint.sh php-…"   3 hours ago         Up 3 hours          9000/tcp               username_app_1
    6cdbf389d2ef        nextcloud:18-fpm-alpine   "/cron.sh"               3 hours ago         Up 3 hours          9000/tcp               username_cron_1
    24d12b2b9f0d        redis:5-alpine            "docker-entrypoint.s…"   3 hours ago         Up 3 hours          6379/tcp               username_redis_1
    a1ae9768a0de        mariadb:10.4              "docker-entrypoint.s…"   3 hours ago         Up 3 hours          3306/tcp               username_db_1
    ```

    We want the name of the container running Nextcloud itself, which should be `username_app_1`. Use that for the following two commands, where `username` is, of course, the name of your user, and `my.domain.com` is the name of your domain.

    ```shell
    $ docker exec --user www-data username_app_1 php occ config:system:set trusted_domains 0 --value my.domain.com
    $ docker exec --user www-data username_app_1 php occ config:system:set overwrite.cli.url --value https://my.domain.com
    $ docker exec --user www-data michael_app_1 php occ config:system:set overwriteprotocol --value https
    ```

    The above will execute three commands provided by `occ` inside the docker container that set some configuration variables for Nextcloud, and should solve the aforementioned issues.

## Configuring Email

Running a mailserver is a major pain. That being the case, I recommend a free account from [MailJet](https://www.mailjet.com/), who have an excellent free-forever starting tier and require no payment details to start.

17. Refer to Mailjet's instructions on how to set up your domain for validated email sending, and when asked how you'll be using Mailjet, select "As a Developer". In the "Send your first email" step, select SMTP Relay. You can use the settings they provide to help you fill in the settings in Nextcloud.

18. In Nextcloud, click the admin's user profile on the top right, and go to Settings. Then in the left-hand sidebar, go to Personal Info. Set the email field here to a valid email address you control, so you'll be able to receive test emails.  

    Then, select Basic Settings in the left-hand sidebar. This is where email is configured. You can fill in the fields as follows (and replace the `mail` sender name with another if you want):

    ```
    Send mode: SMTP
    Encryption: SSL/TLS
    From address: mail@mydomain.com
    Authentication method: Login
    (Check the box that says "Authentication required")
    Server address: <provided by mailjet>
    Port: 465
    Credentials: <provided by mailjet>
    ```

    Now hit "Save", and to test, "Send email". You should receive a test email to the address you just configured for the admin profile.

## Done

Congrats! You have Nextcloud installed and configured for use. Have fun.

