# Nginx Web Server Setup Lab

![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%20Linux-E95420?logo=ubuntu&logoColor=white)
![Nginx](https://img.shields.io/badge/Web%20Server-Nginx-009639?logo=nginx&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Linux%20Fundamentals-blue)

A hands-on lab covering the installation, service management, and server block configuration of **Nginx** on an Ubuntu Linux virtual machine.

---

## Table of Contents

- [Overview](#overview)
- [Skills Demonstrated](#skills-demonstrated)
- [Prerequisites](#prerequisites)
- [Step 1: Installing Nginx](#step-1-installing-nginx)
- [Step 2: Managing the Nginx Service](#step-2-managing-the-nginx-service)
- [Step 3: Setting Up a Server Block](#step-3-setting-up-a-server-block)
- [Server Block Configuration Explained](#server-block-configuration-explained)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Key Takeaways](#key-takeaways)
- [Author](#author)

---

## Overview

Nginx is a high-performance web server widely used as a reverse proxy, load balancer, and caching server. It is built to handle large volumes of traffic efficiently, which makes it a staple of modern web hosting and system administration.

In this lab, you will:

1. Install Nginx on Ubuntu and confirm it is serving the default page.
2. Control the Nginx process using `systemctl`.
3. Host a custom website using a **server block** (Nginx's equivalent of an Apache virtual host).

## Skills Demonstrated

- Linux fundamentals and package management with `apt`
- Service management with `systemd` / `systemctl`
- Web server configuration (Nginx `sites-available` / `sites-enabled` workflow)
- File ownership and permissions
- Configuration validation with `nginx -t`

## Prerequisites

| Requirement | Details |
|---|---|
| Operating system | Ubuntu Linux virtual machine |
| Privileges | Administrative access (`sudo`) |
| Network | Port **80** must be free |

Stop any service that may already be using port 80:

```bash
sudo systemctl stop apache2
sudo systemctl stop nginx
```

> **Note:** If either service is not installed, `systemctl` will report that the unit was not found. This is safe to ignore.

---

## Step 1: Installing Nginx

### 1.1 Update package lists

```bash
sudo apt update -y
```

### 1.2 Install Nginx

```bash
sudo apt install nginx -y
```

This installs Nginx and all required dependencies.

### 1.3 Verify the installation

Nginx starts automatically after installation. Check its status:

```bash
sudo systemctl status nginx
```

A healthy service reports `active (running)`.

Then browse to the server:

```
http://<server_ip_address>
```

You should see the default **Welcome to nginx!** page.

---

## Step 2: Managing the Nginx Service

| Action | Command |
|---|---|
| Stop | `sudo systemctl stop nginx` |
| Start | `sudo systemctl start nginx` |
| Restart | `sudo systemctl restart nginx` |
| Reload (apply config changes without dropping connections) | `sudo systemctl reload nginx` |
| Disable on boot | `sudo systemctl disable nginx` |
| Enable on boot | `sudo systemctl enable nginx` |

---

## Step 3: Setting Up a Server Block

By default, Nginx serves content from `/var/www/html`. To host multiple websites on one server, use server blocks.

> Replace `your_domain` with your actual domain name throughout this section.

### 3.1 Create the website directory

```bash
sudo mkdir -p /var/www/your_domain/html
sudo chown -R $USER:$USER /var/www/your_domain/html
sudo chmod -R 755 /var/www/your_domain
```

### 3.2 Create an index page

```bash
sudo nano /var/www/your_domain/html/index.html
```

```html
<html>
    <head>
        <title>Welcome to Your Domain!</title>
    </head>
    <body>
        <h1>Success! The your_domain server block is working!</h1>
    </body>
</html>
```

### 3.3 Create the server block configuration

```bash
sudo nano /etc/nginx/sites-available/your_domain
```

```nginx
server {
    listen 80;
    listen [::]:80;

    root /var/www/your_domain;
    index index.html index.htm;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 3.4 Enable the server block and validate

Enable the configuration by symlinking it into `sites-enabled`:

```bash
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```

Disable the default site

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Check for configuration errors:

```bash
sudo nginx -t
```

Expected output:

```
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Restart Nginx to apply the changes:

```bash
sudo systemctl restart nginx
```

> **Heads up:** Ubuntu ships a `default` site in `sites-enabled` that claims port 80 as the `default_server`. Because this lab's block uses `server_name _;` without `default_server`, requests may still land on the default page. If so, remove the default symlink (the original file stays in `sites-available`) and reload:
>
> ```bash
> sudo rm /etc/nginx/sites-enabled/default
> sudo nginx -t && sudo systemctl reload nginx
> ```

---

## Server Block Configuration Explained

A server block tells Nginx: *"for requests matching this address and name, serve files from this folder."*

| Directive | Purpose |
|---|---|
| `listen 80;` / `listen [::]:80;` | Listens for HTTP on port 80 over IPv4 and IPv6. |
| `root` | The directory containing the site's files. |
| `index` | Files to try, in order, when a directory is requested. |
| `server_name _;` | A catch-all placeholder that matches any hostname. Replace it with your real domain (e.g. `your_domain www.your_domain`) when hosting multiple sites. |
| `location /` | Handles all request paths under the site root. |
| `try_files $uri $uri/ =404;` | Serves the matching file or directory, and returns a 404 if neither exists. |

---

## Verification

Confirm the configuration is valid and the service is healthy:

```bash
sudo nginx -t
sudo systemctl status nginx
curl -I http://<server_ip_address>
```

A response of `HTTP/1.1 200 OK` and your page content confirms the server block is working.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Nginx fails to start | Port 80 is in use by another service | Run `sudo ss -tlnp \| grep :80` and stop the conflicting service |
| Default Nginx page still shows | Default site is still enabled | Remove `/etc/nginx/sites-enabled/default` and reload |
| `nginx -t` reports an error | Typo or missing semicolon in the config | Check the reported file and line number |
| `403 Forbidden` | Incorrect ownership or permissions on the web root | Re-apply the `chown` and `chmod` commands from Step 3.1 |
| `404 Not Found` | Wrong `root` path or missing `index.html` | Verify the path in the config matches the directory you created |
| Changes not appearing | Config not reloaded | Run `sudo systemctl reload nginx` |
| Logs | Errors and access records | `sudo tail -f /var/log/nginx/error.log` |

---

## Key Takeaways

- Nginx's `sites-available` / `sites-enabled` layout keeps configuration modular: create the file, then enable it with a symlink.
- Always run `nginx -t` before restarting or reloading to catch syntax errors safely.
- Use `reload` instead of `restart` to apply configuration changes without interrupting active connections.
- Only one service can bind port 80, so stop Apache or other web servers before running Nginx.

---

## Author

**Gerald Oti** - DevOps / Platform / SRE Engineer, Berlin

- GitHub: [goti13](https://github.com/goti13)
- LinkedIn: [gerald-oti](https://www.linkedin.com/in/gerald-oti/)
