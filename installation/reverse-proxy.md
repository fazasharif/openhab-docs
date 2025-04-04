---
layout: documentation
title: Running openHAB Behind a Reverse Proxy
---

# Running openHAB Behind a Reverse Proxy using NGINX

::: warning Security Warning
It is vitally important that you MUST NOT directly expose your openHAB instance to the Internet (e.g. by opening a port in your firewall)!
:::

[[toc]]

## Setting up NGINX

These are the steps required to use [**NGINX**](https://nginx.org), a lightweight HTTP server, although you can use **Apache HTTP** server or any other HTTP server which supports reverse proxying.

### Installation

NGINX runs as a service in most Linux distributions, installation should be as simple as:

```shell
sudo apt-get update && sudo apt-get install nginx
```

Once installed, you can test to see if the service is running correctly by going to `http://mydomain_or_myip`, you should see the default "Welcome to nginx" page.
If you don't, you may need to check your firewall or ports and check if port 80 (and 443 for HTTPS later) is not blocked and that services can use it.

### Basic Configuration

NGINX configures the server when it starts up based on configuration files.
The location of the default setup is `/etc/nginx/sites-enabled/default`. To allow NGINX to proxy openHAB, you need to change this file (make a backup of it in a different folder first).

The recommended configuration below assumes that you run the reverse proxy on the same machine as your openHAB runtime.
If this doesn't fit for you, you just have to replace `proxy_pass http://localhost:8080/` by your openHAB runtime hostname (such as `http://youropenhabhostname:8080/`).

```json
server {
    listen                                    80;
    server_name                               mydomain_or_myip;

    # Cross-Origin Resource Sharing
    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow_Credentials' 'true' always;
    add_header 'Access-Control-Allow-Headers' 'Authorization,Accept,Origin,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range' always;
    add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT,DELETE,PATCH' always;

    location / {
        proxy_http_version                    1.1;
        proxy_pass                            http://localhost:8080/;
        proxy_set_header Host                 $http_host;
        proxy_set_header X-Real-IP            $remote_addr;
        proxy_set_header X-Forwarded-For      $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto    $scheme;
        proxy_read_timeout                    3600;
    }
}
```

It is also recommended to name the file to something relevant to what it's doing, if you already have a default file in place, then you can rename it via:

```shell
sudo mv /etc/nginx/sites-enabled/default /etc/nginx/sites-enabled/openhab
```

Otherwise, create a new file. **Every file in the `sites-enabled` folder gets processed by NGINX, so make sure you only have one per site.**

After saving over the file but **before you commit** the changes to our server, you should **test** to see if our changes contain any errors; this is done with the command:

```shell
sudo nginx -t
```

If you see that the test is successful, you can restart the NGINX service with...

```shell
sudo service nginx restart
```

...and then go to `http://mydomain_or_myip` to see your openHAB server.

## Authentication with NGINX

For further security, you may wish to ask for a **username and password** before users have access to openHAB.
This is fairly simple in NGINX once you have the reverse proxy setup, you just need to provide the server with a basic authentication user file.

### Creating the First User

You will be using _htpasswd_ to generate a username/password file, this utility can be found in the apache2-utils package:

```shell
sudo apt-get install apache2-utils
```

To generate a file that NGINX can use you use the following command, don't forget to change _username_ to something meaningful!

```shell
sudo htpasswd -c /etc/nginx/.htpasswd username
```

You will receive a prompt to create a password for this username, once finished the file will be created.
You're then free to reference the file to NGINX.

### Referencing the File in the NGINX Configuration

Now the configuration file (`/etc/nginx/sites-enabled/openhab`) needs to be edited to use this password.
Open the configuration file and **add** the following lines underneath the proxy_* settings:

```text
    auth_basic                            "Username and Password Required";
    auth_basic_user_file                  /etc/nginx/.htpasswd;
```

### Add authorization and cookie directives in NGINX Configuration

This is an important new requirement in openHAB 3.0 and later versions.
This is not required prior to openHAB 3.0. You must add the following two directives underneath the `add_header` (in the `server` block) and `proxy_set_header` (in the `location /` block) items respectively:

```text
    add_header Set-Cookie X-OPENHAB-AUTH-HEADER=1;
    proxy_set_header Authorization          "";
```

Once done, **test and restart your NGINX service** and authentication should now be enabled on your server!

### Adding or Removing users

To add new users to your site, you must use following command, **do not** use the `-c` modifier again as this will remove all previously created users:

```shell
sudo htpasswd /etc/nginx/.htpasswd username
```

and to delete an existing user:

```shell
sudo htpasswd -D /etc/nginx/.htpasswd username
```

Once again, any changes you make to these files **must be followed with restarting the NGINX service** otherwise no changes will be made.

### Use a client certificate based authentication

You can find a short tutorial in the community forum on how to do so.
[Using NGINX Reverse Proxy for client certificate authentication](https://community.openhab.org/t/using-nginx-reverse-proxy-for-client-certificate-authentication-start-discussion/43064)

## Making Exceptions for Specific IP addresses

It is often desirable to allow specific IPs (e.g. the local network) to access openHAB without needing to prompt for a password or to block everyone else entirely.
In these cases NGINX can use the `satisfy any` directive followed by `allow` and `deny` lines to make these exceptions.
These lines are placed in the `location{}` block. For example, by adding the lines:

```text
    satisfy  any;
    allow    192.168.0.0/24;
    allow    127.0.0.1;
    deny     all;
```

NGINX will allow anyone within the 192.168.0.0/24 range **and** the localhost to connect without a password.
If you have setup a password following the previous section, then the rest will be prompted for a password for access.

## Setting up a Domain

To generate a trusted certificate, you need to own a domain. To acquire your own domain, you can use one of the following methods:

| Method                        | Example Links                                                                                                                                  | Note                                                                                                                                                                          |
| :---------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Purchasing a domain name      | [GoDaddy](https://www.godaddy.com), [Namecheap](https://www.namecheap.com), [Enom](https://www.enom.com), [Register](https://www.register.com) | You should have an IP address that doesn't change (i.e. fixed), or changes rarely, and then update the DNS _A record_ so that your domain/subdomain to point towards your IP. |
| Obtaining a free domain       | [FreeNom](https://www.freenom.com)                                                                                                             | Setup is the same as above.                                                                                                                                                   |
| Using a "Dynamic DNS" service | [No-IP](https://www.noip.com), [Dyn](https://www.dyn.com/dns), [FreeDNS](https://freedns.afraid.org)                                           | Uses a client to automatically update your IP to a domain of you choice, some Dynamic DNS services (like FreeDNS) offer a free domain too.                                    |

## Enabling HTTPS

Encrypting the communication between client and the server is important because it protects against eavesdropping and possible forgery.
The following options are available depending if you have a valid domain:

If you have a **valid domain and can change the DNS** to point towards your IP, follow the [instructions for Let's Encrypt](#using-lets-encrypt-to-generate-trusted-certificates)
If you need to use an internal or external IP to connect to openHAB, follow the [instructions for OpenSSL](#using-openssl-to-generate-self-signed-certificates)

## Using OpenSSL to Generate Self-Signed Certificates

OpenSSL is also packaged for most Linux distributions, installing it should be as simple as:

```shell
sudo apt-get install openssl
```

Once complete, you need to create a directory where our certificates can be placed:

```shell
sudo mkdir -p /etc/ssl/certs
```

Now OpenSSL can be told to generate a 2048 bit long RSA key and a certificate that is valid for a year:

```shell
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/openhab.key -out /etc/ssl/openhab.crt
```

You will be prompted for some information which you will need to fill out for the certificate, when it asks for a **Common Name**, you may enter your IP Address:
Common Name (e.g. server FQDN or YOUR name) []: xx.xx.xx.xx

### Adding the Certificates to Your Proxy Server

The certificate and key should have been placed in `/etc/ssl/`.
NGINX needs to be told where these files are and then enable the reverse proxy to direct HTTPS traffic.
In the NGINX configuration, place the following underneath your `server_name` variable:

```text
  ssl_certificate                 /etc/ssl/openhab.crt;
  ssl_certificate_key             /etc/ssl/openhab.key;
```

## Using Let's Encrypt to Generate Trusted Certificates

::: tip
Skip this step if you have no domain name or have already followed the instructions for OpenSSL
:::

Let's Encrypt is a service that allows anyone with a valid domain to automatically generate a trusted certificate, these certificates are usually accepted by a browser without any warnings.

### Setting up the NGINX Proxy Server to Handle the Certificate Generation Procedure

Let's Encrypt needs to validate that the server has control of the domain.
The simplest way of doing this is using a **webroot plugin** to place a file on the server, and then access it using a specific url: _/.well-known/acme-challenge_.
Since the proxy only forwards traffic to the openHAB server, the server needs to be told to handle requests at this address differently.

First, **create a directory** that Certbot can be given access to:

```shell
sudo mkdir -p /var/www/mydomain
```

Next add the new location parameter to your NGINX config, this should be **placed above the last brace** in the server setting:

```text
  location /.well-known/acme-challenge/ {
    root                            /var/www/mydomain;
  }
```

### Using Certbot

Certbot is a tool which simplifies the process of obtaining secure certificates.
The tool may not be packaged for some Linux distributions so installation instructions may vary, check out [their website](https://certbot.eff.org/) and follow the instructions **using the webroot mode**.
Don't forget to change the example domain to your own! An example of a valid certbot command (in this case for Debian Jessie) would be:

```shell
sudo certbot certonly --webroot -w /var/www/mydomain -d mydomain
```

### Adding the Certificates to Your Proxy Server

The certificate and key should have been placed in `/etc/letsencrypt/live/mydomain_or_myip`.
NGINX needs to be told where these files are and then enable the reverse proxy to direct HTTPS traffic, using Strict Transport Security to prevent man-in-the-middle attacks.
In the NGINX configuration, place the following underneath your server_name variable:

```text
    ssl_certificate                 /etc/letsencrypt/live/mydomain_or_myip/fullchain.pem;
    ssl_certificate_key             /etc/letsencrypt/live/mydomain_or_myip/privkey.pem;
    add_header                      Strict-Transport-Security "max-age=31536000";
```

## Setting Your NGINX Server to Listen to the HTTPS Port

Regardless of the option you choose, make sure you change the port to listen in on HTTPS traffic.

```text
    listen                          443 ssl;
```

After restarting NGINX service, you will be using a valid HTTPS certificate.
You can check by going to `https://mydomain_or_myip` and confirming with your browser that you have a valid certificate.
**These certificates expire within a few months** so it is important to run the updater in a cron expression (and also restart NGINX) as explained in the Certbot setup instructions.
If you want to keep hold of a HTTP server for some reason, just add `listen 80;` and remove the Strict-Transport-Security line.

## Redirecting HTTP Traffic to HTTPS

You may want to redirect all HTTP traffic to HTTPS, you can do this by adding the following to the NGINX configuration.
This will essentially replace the HTTP url with the HTTPS version!

```json
server {
    listen                          80;
    server_name                     mydomain_or_myip;
    return 301                      https://$server_name$request_uri;
}
```

It might be the case that you can't use standard ports. When trying to access a HTTPS port with HTTP, NGINX will respond with a `400 Bad Request`.
We can redirect this gracefully using a "HTTPS [error page](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page)" for the [non-standard HTTP error code 497](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#errors)

```json
server {
    listen                          8084 ssl;
    server_name                     mydomain_or_myip;
    error_page                      497 =301 https://$host:$server_port$request_uri;
}
```

## Putting it All Together

After following all the steps on this page, you _should_ have a NGINX server configuration (`/etc/nginx/sites-enabled/openhab`) that looks like this:

```json
server {
    listen                          80;
    server_name                     mydomain_or_myip;
    return 301                      https://$server_name$request_uri;
}
server {
    listen                          443 ssl;
    server_name                     mydomain_or_myip;

    # Cross-Origin Resource Sharing.
    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow_Credentials' 'true' always;
    add_header 'Access-Control-Allow-Headers' 'Authorization,Accept,Origin,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range' always;
    add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT,DELETE,PATCH' always;

    # openHAB 3 api authentication
    add_header Set-Cookie X-OPENHAB-AUTH-HEADER=1;

    ssl_certificate                 /etc/letsencrypt/live/mydomain/fullchain.pem; # or /etc/ssl/openhab.crt
    ssl_certificate_key             /etc/letsencrypt/live/mydomain/privkey.pem;   # or /etc/ssl/openhab.key
    add_header                      Strict-Transport-Security "max-age=31536000"; # Remove if using self-signed and are having trouble.

    location / {
        proxy_http_version                      1.1;
        proxy_pass                              http://localhost:8080/;
        proxy_set_header Host                   $http_host;
        proxy_set_header X-Real-IP              $remote_addr;
        proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto      $scheme;
        proxy_set_header Upgrade                $http_upgrade;
        proxy_set_header Connection             "Upgrade";
        proxy_set_header Authorization          "";
        satisfy                                 any;
        allow                                   192.168.0.0/24;
        allow                                   127.0.0.1;
        deny                                    all;
        auth_basic                              "Username and Password Required";
        auth_basic_user_file                    /etc/nginx/.htpasswd;
    }

    #### When using Let's Encrypt Only ####
    location /.well-known/acme-challenge/ {
        root                                    /var/www/mydomain;
    }
}
```

## Configuration on Synology DiskStation

Synology DSM (as of 6.2) has the ability to automatically acquire certificates from Let's Encrypt and renew them every 90 days as required.
The majority of the configuration mentioned above can be completed through the DSM GUI, but SSH access is required to implement authentication (**authentication is essential for remote access to your openHAB instance**).

Before you continue, make sure you have the below conditions:

- A working installation of openHAB on your DiskStation (see the [Synology Installation Guide](https://www.openhab.org/docs/installation/synology.html/))
- Your own domain you can configure the CAA record for (see [Setting up a Domain](#setting-up-a-domain))
- Access to your DiskStation by SSH ([How to login to DSM with root permission via SSH/Telnet](https://www.synology.com/en-global/knowledgebase/DSM/tutorial/General_Setup/How_to_login_to_DSM_with_root_permission_via_SSH_Telnet/))
- Ports 80 and 443 forwarded from your router to your DiskStation (make sure you reconfigure the router web UI to a different port first, so you don't lose access!)

Log into the GUI of your DiskStation as administrator, and open the package center.
Install Apache HTTP Server.
This is needed to generate the password files.

Go to Control Panel > Application Portal > Reverse Proxy. We will set up two reverse proxies, one for HTTP and one for HTTPS.
The HTTP one can be disabled later if desired (not at all essential if you will only use the app remotely, and never a browser).

Create two reverse proxies as follows:

| Parameter             | Value                                                      |
| :-------------------- | :--------------------------------------------------------- |
| Description:          | openHAB HTTPS                                              |
| Source Protocol:      | HTTPS                                                      |
| Source Hostname:      | your-hostname.com                                          |
| Source Port:          | 443                                                        |
| Enable HSTS           | Unchecked                                                  |
| Enable HTTP/2         | Unchecked                                                  |
| Enable access control | Unchecked                                                  |
| Destination Protocol: | HTTPS                                                      |
| Destination Hostname: | localhost                                                  |
| Destination Port:     | 8443 (or whichever HTTPS port your openHAB instance is on) |

| Parameter             | Value                                                     |
| :-------------------- | :-------------------------------------------------------- |
| Description:          | openHAB HTTP                                              |
| Source Protocol:      | HTTP                                                      |
| Source Hostname:      | your-hostname.com                                         |
| Source Port:          | 80                                                        |
| Enable HSTS           | Unchecked                                                 |
| Enable HTTP/2         | Unchecked                                                 |
| Enable access control | Unchecked                                                 |
| Destination Protocol: | HTTP                                                      |
| Destination Hostname: | localhost                                                 |
| Destination Port:     | 8080 (or whichever HTTP port your openHAB instance is on) |

Verify that the reverse proxy is working as expected - try <http://your-hostname.com> and <https://your-hostname.com> - you should end up at the openHAB landing page in both cases, but will get a security warning for the https site.

Next, acquire certificates from Let's Encrypt using the GUI in DSM.

Go to Control Panel > Security > Certificate, and click on 'Add'.
Select the option to 'Add a new Certificate'.
Put in a description, something like 'openHAB SSL Cert' (it doesn't matter).
Select 'Get a certificate from Let's Encrypt' and check the box to set it as default.
Click next.
Put in your domain name and email address.
Add a 'Subject Alternative Name' if you want a different topic in the subject line when Let's Encrypt email you about that certificate (not essential).
Click Apply, and wait a few minutes - your certificate is done!

::: tip Note
Sometimes you may receive an error at the end of the certificate wizard - the first time this happens, click on 'cancel and see if you have a certificate anyway.
If the certificate has been generated, you are good to go.
:::

Select the certificate that has just been created, and click on 'Configure'.
Ensure that the new certificate is listed next to your-hostname.com in the table - something like the below.
If it's not selected, update it.

| Services             | Certificate       |
| :------------------- | :---------------- |
| your-hostname.com    | your-hostname.com |
| FTPS                 | synology.com      |
| Cloud Station Server | synology.com      |
| etc etc              | synology.com      |

Once this is done, update the CAA record for your-hostname.com with your registrar (exact process will vary by registrar).
Within an hour or so, you should not receive the security warning for <https://your-hostname.com>.

Next, you must add authentication to the reverse proxy.
There's no GUI way to do this, so we need to create another small NGINX virtual host on the DiskStation.

Log into your DiskStation by SSH.
Use the admin username and password.
Create a .htpasswd file in your openHAB userdata folder (your userdata location may vary, update accordingly):

```shell
htpasswd -c /volume1/openHAB/userdata/.htpasswd username
```

Next, add a very simple NGINX configuration similar to that created above, but without the SSL parameters.
DSM comes with vi installed by default, but you may wish to [install nano](https://anto.online/other/how-to-install-nano-on-your-synology-nas/)

```shell
sudo nano /usr/local/etc/nginx/sites-enabled/openHAB-auth
```

```json
# openHAB NGINX config

server {
    listen                          2020; #This is simply an unused port, it can be any number

    server_name                     your_domain.com;

    return 301                      https://$server_name$request_uri;
}

server {
    listen                          7443 ssl; #This is simply an unused port, it can be any number
    server_name                     your_domain.com;

    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow_Credentials' 'true' always;
    add_header 'Access-Control-Allow-Headers' 'Authorization,Accept,Origin,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range' always;
    add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT,DELETE,PATCH' always;
    add_header Set-Cookie X-OPENHAB-AUTH-HEADER=1;

    location / {
        proxy_http_version                      1.1;
        proxy_pass                              https://localhost:8443/; #Update the port number if needed
        proxy_set_header Host                   $http_host;
        proxy_set_header X-Real-IP              $remote_addr;
        proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto      $scheme;
        proxy_set_header Upgrade                $http_upgrade;
        proxy_set_header Connection             "Upgrade";
        proxy_set_header Authorization          "";
        satisfy                                 any;
        allow                                   192.168.1.0/24;
        allow                                   127.0.0.1;
        deny                                    all;
        auth_basic                              "Username and Password Required";
        auth_basic_user_file                    /volume1/openHAB/userdata/.htpasswd; #Update with your userdata folder if different
    }

}
```

Once you are done, save the file, restart and test NGINX:

```shell
sudo nginx -s reload && sudo nginx -t
```

As above, the first part of the file redirects any HTTP queries to HTTPS directly.
If you don't get any errors, update the reverse proxy settings in the DSM GUI to point to these new endpoints.
Back in the GUI, go to Control Panel > Application Portal > Reverse Proxy, make the updates below:

| Parameter         | Value                                                     |
| :---------------- | :-------------------------------------------------------- |
| Destination Port: | 7443 (or whatever you set it to in the openHAB-auth file) |

| Parameter         | Value                                                     |
| :---------------- | :-------------------------------------------------------- |
| Destination Port: | 2020 (or whatever you set it to in the openHAB-auth file) |

::: tip Note
We do this 'double' redirect to take advantage of the GUI certificate handling in DSM - this is the equivalent of CertBot for a Linux installation.
:::

Give it a try again - you should now get redirected to `https://your-hostname.com` from `http://your-hostname.com`, and should receive a username and password prompt before you see the openHAB landing page.

If you need to troubleshoot the nginx server, SSH into your DiskStation, and check the NGINX error log:

```shell
sudo tail -f /var/log/nginx/error.log
```

This log will update in real-time, so do whatever it was that you were having issues with again, and you'll see the error.

## Additional HTTPS Security

To test your security settings [SSL Labs](https://www.ssllabs.com/ssltest/) provides a tool for testing your domain against ideal settings (Make sure you check "Do not show the results on the boards" if you don't want your domain seen).

This optional section is for those who would like to strengthen the HTTPS security on openHAB, it can be applied regardless of which HTTPS method you used [above](#enabling-https), **but you need to follow at least one of them first**.

First, we need to generate a stronger key exchange, to do this we can generate an additional key with OpenSSL.

::: tip Note
Depending on your hardware this will take up to few minutes to complete:
:::

```shell
mkdir -p /etc/nginx/ssl
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```

Now we can configure NGINX to use this key, as well as telling the client to use specific cyphers and SSL settings, just add the following under your `ssl_certificate **` settings but above ``location *``.
All of these settings are customizable, but make sure you [read up on](https://nginx.org/en/docs/http/configuring_https_servers.html) what these do first before changing them:

```text
    ssl_protocols                   TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers       on;
    ssl_dhparam                     /etc/nginx/ssl/dhparam.pem;
    ssl_ciphers                     ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:HIGH:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!CBC:!EDH:!kEDH:!PSK:!SRP:!kECDH;
    ssl_session_timeout             1d;
    ssl_session_cache               shared:SSL:10m;
    keepalive_timeout               70;
```

Feel free to test the new configuration again on [SSL Labs](https://www.ssllabs.com/ssltest/).
If you're achieving A or A+ here, then your client-openHAB communication is very secure.

## Further Reading

The setup above is a suggestion for high compatibility with an A+ rating at the time of writing, however flaws in these settings (particularly the cyphers) may become known overtime.
The following articles may be useful when understanding and changing these settings.

- [Better Crypto](https://bettercrypto.org/)
- [SSL Labs - Best Practices](https://www.ssllabs.com/projects/best-practices/)
- [Hynek Schlawack - Hardening Your Web Server’s SSL Ciphers](https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/)
