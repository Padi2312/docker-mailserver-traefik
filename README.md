# Docker Mailserver + Traefik

Docker-Compose setup for running a mailserver with Traefik as a reverse proxy.

> [!WARNING]
> This repository is a work in progress and might not be fully functional. Please use it with caution and report any issues you encounter or suggestions you have.


> [!IMPORTANT]  
> The following setup expects you to have a working Traefik setup. You should also have setup the mailserver with the default configuration before using this setup. In case you haven't configured anything yet, you can use the provided files in this repository.

# Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
  - [Traefik](#traefik)
  - [Mailserver](#mailserver)
    - [Mailserver settings](#mailserver-settings)
      - [`mailserver/config/postfix/main.cf` - Postfix configuration](#mailserverconfigpostfixmaincf---postfix-configuration)
      - [`mailserver/config/postfix-master.cf` - Postfix master configuration](#mailserverconfigpostfix-mastercf---postfix-master-configuration)
      - [`mailserver/config/dovecot/dovecot.conf` - Dovecot configuration](#mailserverconfigdovecotdovecotconf---dovecot-configuration)
  - [DNS Configuration](#dns-configuration)
- [`mailsetup.sh`](#mailsetupsh)
- [License](#license)
- [References](#references)


# Overview

This setup uses **Traefik** as a reverse proxy and **Docker Mailserver** as the mailserver. The mailserver is configured to use **Roundcube** as the webmail client.

**Roundcube** is a modern webmail client that is easy to use and has a lot of features, but this is **optional** and can be removed if you want to use a different webmail client.

# Prerequisites

> [!NOTE]
> In case you don't have a running traefik setup, you need to setup Traefik in the `traefik` directory. I recommend you to have a look at the setup of [wollomatic/traefik-hardened](https://github.com/wollomatic/traefik-hardened). The configuration is also being used in this repository.

- Running Traefik setup with certificate generation over LetsEncrypt (otherwise you can use the provided `traefik` directory in this repository)
- You need a domain that you can use for the mailserver
- You need to have access to the DNS settings of the domain to set the correct records

# Setup

This part will guide you through the setup of the mailserver and Traefik. It will show how to configure running instances of Traefik and the mailserver.

You can also clone this repository and use the given files in case you haven't running anything yet.

## Traefik

We need to add the following ports to the Traefik configuration to route the mailserver traffic:

**Mailserver Ports**
| Port | Type     | Usage                                        |
| ---- | -------- | -------------------------------------------- |
| 25   | SMTP     | Used for sending emails                      |
| 465  | SMTP/SSL | Used for sending emails securely over SSL    |
| 587  | SMTP/TLS | Used for sending emails securely over TLS    |
| 993  | IMAP/SSL | Used for accessing emails over IMAP securely |
| 4190 | Sieve    | Used for managing email filters              |

---

Edit the `traefik/docker-compose.yaml` file:

```diff
   traefik:
     # Remaining configuration
     ports:
       - "10080:80"
       - "10443:443"
+      - "25:25"
+      - "465:465"
+      - "587:587"
+      - "993:993"
+      - "4190:4190"
```

---

Edit the your traefik configuration file (in this repo it's `traefik/config/traefik.yaml`):

```diff
 entryPoints:
   web:
     address: ':10080' # will be routed to port 80, see docker-compose.yaml
     http:
       redirections:   # redirect entire entrypoint to https
         entryPoint:
           to: ':443'
           scheme: https
   websecure:
     address: ':10443' # will be routed to port 443, see docker-compose.yaml
     http3:
       advertisedPort: 443
+  smtp:
+    address: ':25'
+  smtp-ssl:
+    address: ':465'
+  smtp-tls:
+    address: ":587"
+  imap-ssl:
+    address: ':993'
+  sieve:
+    address: ':4190'
```

---

That's it for the Traefik part. Now we can move on to the mailserver setup.

## Mailserver

> [!IMPORTANT]
> You have to change the domain `example.invalid` to your domain in the following steps.

For the mailserver setup we're going to start with the `docker-compose.yaml` file.

Edit the `mailserver/docker-compose.yaml` file:

```diff
mailserver:
  image: mailserver/docker-mailserver:latest
  restart: unless-stopped
  volumes:
+   - ../traefik/config/acme.json:/etc/traefik/acme.json:ro
+ hostname: mail
+ domainname: example.invalid
- ports:
-   - "25:25"
-   - "465:465"
-   - "587:587"
-   - "993:993"
-   - "4190:4190"
+ labels:
+   - "traefik.enable=true"

+   - "traefik.http.routers.mailserver.entrypoints=websecure"
+   - "traefik.http.routers.mailserver.rule=Host(`mail.example.invalid`)"
+   - "traefik.http.routers.mailserver.tls=true"
+   - "traefik.http.routers.mailserver.tls.certresolver=default"
+   - "traefik.http.routers.mailserver.middlewares=secHeaders@file"
+   - "traefik.http.routers.mailserver.service=noop@internal"

+   - "traefik.tcp.routers.smtp.rule=HostSNI(`*`)"
+   - "traefik.tcp.routers.smtp.entrypoints=smtp"
+   - "traefik.tcp.routers.smtp.service=smtp"
+   - "traefik.tcp.services.smtp.loadbalancer.server.port=25"
+   - "traefik.tcp.services.smtp.loadbalancer.proxyProtocol.version=1"

+   - "traefik.tcp.routers.smtp-ssl.rule=HostSNI(`*`)"
+   - "traefik.tcp.routers.smtp-ssl.entrypoints=smtp-ssl"
+   - "traefik.tcp.routers.smtp-ssl.tls.passthrough=true"
+   - "traefik.tcp.routers.smtp-ssl.service=smtp-ssl"
+   - "traefik.tcp.services.smtp-ssl.loadbalancer.server.port=465"
+   - "traefik.tcp.services.smtp-ssl.loadbalancer.proxyProtocol.version=1"

+   - "traefik.tcp.routers.smtp-tls.rule=HostSNI(`*`)"
+   - "traefik.tcp.routers.smtp-tls.entrypoints=smtp-tls"
+   - "traefik.tcp.routers.smtp-tls=true"
+   - "traefik.tcp.routers.smtp-tls.service=smtp-tls"
+   - "traefik.tcp.services.smtp-tls.loadbalancer.server.port=587"
+   - "traefik.tcp.services.smtp-tls.loadbalancer.proxyProtocol.version=1"

+   - "traefik.tcp.routers.imap-ssl.rule=HostSNI(`*`)"
+   - "traefik.tcp.routers.imap-ssl.entrypoints=imap-ssl"
+   - "traefik.tcp.routers.imap-ssl.service=imap-ssl"
+   - "traefik.tcp.routers.imap-ssl.tls.passthrough=true"
+   - "traefik.tcp.services.imap-ssl.loadbalancer.server.port=10993"
+   - "traefik.tcp.services.imap-ssl.loadbalancer.proxyProtocol.version=2"

+   - "traefik.tcp.routers.sieve.rule=HostSNI(`*`)"
+   - "traefik.tcp.routers.sieve.entrypoints=sieve"
+   - "traefik.tcp.routers.sieve.service=sieve"
+   - "traefik.tcp.services.sieve.loadbalancer.server.port=4190"
  environment:
+   - "PERMIT_DOCKER=host"
+   - "SSL_TYPE=letsencrypt"
+   - "SSL_DOMAIN=example.invalid"
+ networks:
+   - traefik-servicenet
```

- **Removal of ports** - We don't need to expose the ports anymore, as Traefik will handle the routing

- **Labels** - We're adding the Traefik labels to the mailserver container

- **Environment** - We're setting the `SSL_DOMAIN` to the domain you're using for the mailserver and the `SSL_TYPE` to `letsencrypt` to use Let's Encrypt for the SSL certificates

- **Environment** - We're setting the `PERMIT_DOCKER` to `host` to allow the mailserver to communicate with the host

- **Networks** - We're adding the `traefik` network to the mailserver container to make it accessible to Traefik

- **Certificate Generation** - Mailserver must have a valid certificate for the domain in order to work properly. 
  - Traefik will generate certificates for the domain and store them in the `traefik/config/acme/acme.json` file.
  - The mailserver will use the certificates from the `acme.json` file to secure the connections.

### Mailserver settings

After the initial start of the mailserver, you can access the configuration files in the `mailserver/config` directory

You have to edit the following files:

#### `mailserver/config/postfix/main.cf` - Postfix configuration

```cf
postscreen_upstream_proxy_protocol = haproxy
smtpd_sasl_auth_enable = yes
smtpd_tls_auth_only = no 
```

`smtptd_sasl_auth_enable` - Enable SASL authentication for the SMTP server
`smtptd_tls_auth_only` - Allow non-TLS connections for the SMTP server (might work also with `yes`, didn't test it)

---

#### `mailserver/config/postfix-master.cf` - Postfix master configuration

```cf
submission/inet/smtpd_upstream_proxy_protocol=haproxy
smtps/inet/smtpd_upstream_proxy_protocol=haproxy
```

---

#### `mailserver/config/dovecot/dovecot.conf` - Dovecot configuration

```cf
haproxy_trusted_networks = <TRAEFIK_SERVICENET_IP>/16
haproxy_timeout = 3 secs
service imap-login {
  inet_listener imaps {
    haproxy = yes
    ssl = yes
    port = 10993
  }
}
```

Add the IP of the `traefik-servicenet` network to the `haproxy_trusted_networks` variable to allow the Dovecot server to communicate with Traefik over the network. 

You can find the IP of the network by running `docker network inspect traefik-servicenet`

---

## DNS Configuration

You need to add the following DNS records to your domain:

**Mailserver DNS Records**
| Type | Name                       | Value                                                                                |
| ---- | -------------------------- | ------------------------------------------------------------------------------------ |
| A    | mail.example.invalid       | <YOUR_SERVER_IP>                                                                     |
| MX   | example.invalid            | mail.example.invalid                                                                 |
| TXT  | mail.\_domainkey           | v=DKIM1; k=rsa; p=\<DKIM_PUBLIC_KEY\>                                                |
| TXT  | \_dmarc.example.invalid    | v=DMARC1; p=none; rua=mailto: [email protected]; ruf=mailto: [email protected]; fo=1 |
| TXT  | \_spf.example.invalid      | v=spf1 mx -all                                                                       |
| PTR  | <REVERSED_IP>.in-addr.arpa | mail.example.invalid                                                                 |

### DKIM Configuration

You will get the DKIM public key from the `mailserver/config/opendkim/keys/<YOUR_DOMAIN>/mail.txt` file the initial start and setup of DKIM. Copy the contents and remove the quotes and the `mail._domainkey	IN	TXT	("` part.

After editing it should look like this:

```txt
v=DKIM1; h=sha256; k=rsa; p=Dmdklfi3o489DJKDdloeu4397dnDFJRdkejnslsdfllsdf343499834...
```

# `mailsetup.sh`

> [!NOTE]
> The script is just a small wrapper around the `setup.sh` from Docker Mailserver

I've created a script that will simplify the usage of the `setup.sh` from Docker Mailserver. It will allow you to run the setup commands inside the mailserver container without the need to type the `docker compose exex mailserver` all the time.

You can use it like this:

```sh
./mailsetup.sh <command>
```

More details for usage with `setup.sh` at [https://docker-mailserver.github.io/docker-mailserver/edge/config/setup.sh/](https://docker-mailserver.github.io/docker-mailserver/edge/config/setup.sh/)


# License

This project is licensed under the MIT License - see the LICENSE file for details.

# References

- [wollomatic/traefik-hardened](https://github.com/wollomatic/traefik-hardened) - Traefik setup
- [docker-mailserver/docker-mailserver](https://github.com/docker-mailserver/docker-mailserver) - Docker Mailserver
- [Mailsever behind Proxy](https://docker-mailserver.github.io/docker-mailserver/latest/examples/tutorials/mailserver-behind-proxy/) - Docker Mailserver behind a proxy setup