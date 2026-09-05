# How to deploy MkDocs on nginx Web Server



## 1. Preparation for Installation

Check the installed versions of Python and PIP (3.8.2+ and 23.3.1+ required respectively) using the following commands:

```bash
python --version
pip --version
```

To upgrade PIP, run:

```
python -m pip install --upgrade pip
```

If upgrading PIP over the internet is not possible, download it from pypi.org/project/pip/#files on another PC, transfer it to the target machine, unzip and install:

```
sudo python setup.py install
```

## 2. Installing MkDocs

To install MkDocs, run:

```
pip install mkdocs
```

If the server firewall blocks downloading Python packages, you can use PIP on another computer to download the package and its dependencies without installing them:

```
python -m pip download --destination-directory <download_location> mkdocs
```

After transferring the .whl files to the target server, install the packages with:

```
pip install --no-index --find-links=. /path_to_files/*.whl
```

To create a new project, run:

```
mkdocs new my-project
cd my-project
```

The created project will contain a configuration file named `mkdocs.yml` and a folder named `docs`, which will hold the documentation source files (`docs` is the default value for the `docs_dir` configuration setting). Currently, the `docs` folder contains only one documentation page named `index.md`. MkDocs comes with a built-in development server that lets you preview your documentation as you work. Make sure you are in the same directory as the `mkdocs.yml` configuration file, then start the server with:

```
mkdocs serve
```

To run the server on a specific IP and port, use:

```
mkdocs serve --dev-addr <ip.port>
```

## 3. Installing nginx

Install nginx on your RHEL server:

```
sudo dnf install nginx
```

If nginx cannot be downloaded on the server, use the "downloadonly" plugin on another PC:

```
sudo dnf install dnf-downloadonly
```

After installing the plugin, download the rpm package and its dependencies (replace the equals signs and their contents with the appropriate values):

```
sudo dnf install --downloadonly --downloaddir=<directory> <package>
```

Open the nginx configuration file:

```
sudo nano /etc/nginx/nginx.conf
```

Locate the server section and add the following configuration parameters:

```
server {
   listen 80;
   server_name yourdomain.com;

   root /путь/до/mkdocs/site;
   index index.html;

   location / {
       try_files $uri $uri/ =404;
   }
}
```

Restart nginx to apply the changes:

```
sudo systemctl restart nginx
```

Or:

```
sudo service nginx restart
```

Your MkDocs site should now be accessible. Verify this by visiting `yourdomain.com` in your web browser.

## 4. Installing MkDocs Material Theme

Install the theme using:

```
pip install mkdocs-material
```

If you encounter the following error during installation:

```
ERROR: Could not find a version that satisfies the requirement setuptools>=40.8.0 (from versions: none)
ERROR: No matching distribution found for setuptools>=40.8.0
```

Then install the `wheel` package, upgrade `setuptools` to the latest version, and then manually download and install the packages:

```
python -m pip download --destination-directory download_location mkdocs-material
pip install --no-build-isolation --find-links=. /path_to_files/*.whl
```

## 5. Configuring HTTP to HTTPS Redirect

### 5.1 Creating an SSL Certificate

SSL uses a combination of a public certificate and a private key. The private key is stored on the server and must not be disclosed. The SSL certificate is public and available to all users requesting content.

To create a self-signed certificate and key, run the following command. Ensure the required directories exist beforehand:

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```

The command consists of the following parts:

```
openssl: basic command-line tool for creating and managing certificates, keys, and other OpenSSL files.
req: this subcommand indicates that an X.509 certificate signing request (CSR) is to be used. X.509 is a public key infrastructure standard that SSL and TLS adhere to for managing keys and certificates. This command allows you to create a new X.509 certificate.
-x509: this option modifies the previous subcommand, telling the utility to create a self-signed certificate instead of generating a CSR.
-nodes: skips the option of securing the certificate with a passphrase. This is necessary so that the Nginx server can read the file without user intervention. Setting a passphrase would require entering it after every reboot.
-days 365: sets the validity period of the certificate (in this case, one year).
-newkey rsa:2048: creates a new certificate and a new 2048-bit RSA key simultaneously.
-keyout: tells OpenSSL where to place the generated key file.
-out: tells OpenSSL where to place the created certificate.
```

When creating the key, you will need to enter the following information:

```
Country Name (2 letter code) [AU]:RU
State or Province Name (full name) [Some-State]:Saint Petersburg
Locality Name (eg, city) []:SPb
Organization Name (eg, company) [Internet Widgits Pty Ltd]:RZD
Organizational Unit Name (eg, section) []:IVC
Common Name (e.g. server FQDN or YOUR name) []:PTK
Email Address []:admin@your_domain.com
```

When using OpenSSL, you should also generate Diffie-Hellman keys, which are needed for Perfect Forward Secrecy (PFS). This process may take a few minutes.

```
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

### 5.2 Configuring nginx for SSL Support

Now edit the nginx configuration. It is recommended to name the file according to its purpose (e.g., `self-signed.conf`):

```
sudo nano /etc/nginx/snippets/self-signed.conf
```

Add the `ssl_certificate` directive pointing to the certificate and the `ssl_certificate_key` directive pointing to the corresponding private key:

```
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

### 5.3 SSL Configuration

Create another snippet for SSL settings. This will allow nginx to use strong encryption and enable some additional security features. These settings can be reused in future nginx configurations, so give the file a generic name:

```
sudo nano /etc/nginx/snippets/ssl-params.conf
```

Copy all the suggested parameters. You only need to add a DNS resolver for upstream requests (this guide uses Google's DNS). Also add the `ssl_dhparam` parameter to configure Diffie-Hellman key support.

```
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
ssl_ecdh_curve secp384r1;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
ssl_dhparam /etc/ssl/certs/dhparam.pem;
```

> Note: Since the certificate is self-signed, SSL stapling will not be used. nginx will issue a warning, disable stapling for this certificate, and continue operating.

### 5.4 Configuring nginx for SSL Support

Open the server block file in a text editor:

```
sudo nano /etc/nginx/nginx.conf
```

Now edit the settings so that unencrypted HTTP requests are automatically redirected to HTTPS.

```
server {
	listen 80 default_server;
	server_name server_domain_or_IP;
	root /var/www/site/;
	index index.html;

	return 301 https://$server_name$request_uri;
}

server {
	listen 443 ssl default_server;
	http2 on;
	root /var/www/site/;
	index index.html;

	include snippets/self-signed.conf;
	include snippets/ssl-params.conf;
}
```

> Note: During configuration, it is recommended to use a temporary 302 redirect. After verifying the settings are correct, switch to a permanent 301 redirect.

Save and close the file.

> Note: The file may contain only one `listen` directive with the `default_server` modifier for a given IP and port combination. If the `default_server` modifier appears in multiple `server` blocks, keep it in only one block and remove it from the others.

### 5.5 Updating nginx Settings

First, check the syntax for errors:

```
sudo nginx -t
```

If no errors are found, the command will return:

```
nginx: [warn] "ssl_stapling" ignored, issuer certificate not found
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Note the warning at the beginning. The web server will issue this warning when using a self-signed certificate. Connections are still encrypted correctly.

If syntax errors are found, fix them. Then restart the web server:

```
sudo systemctl restart nginx
```

### 5.6 Testing the Configuration

Now verify that traffic between the server and client is encrypted. Open the following link in your browser:

```
https://server_domain_or_IP
```

Since the certificate is self-signed, your antivirus or browser may warn about its untrustworthiness:

```
Your connection is not secure. Attackers might be trying to steal your information. It is recommended not to proceed to the site.
```

This is normal behavior in such a situation, as the browser cannot verify the host's authenticity. However, in this case, you only need to encrypt traffic, which the self-signed certificate handles perfectly, so you can bypass the warning. After that, you will have access to your site.
