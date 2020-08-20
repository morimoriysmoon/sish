# More

## Purchase domain and connect to DNS Provider
To generate wildcard certificates with Letsencrypt, we need to use DNSrobocert and provide DNS API key to DNSrobocert. I recommend cloudflare as DNS Provider.
You can find more DNS providers DNSrobocert supports.

	https://dnsrobocert.readthedocs.io/en/latest/providers_options.html

Please prepare DNS API key for next step.

## DNS record
Please edit DNS record to point all the subdomains to an identical IP address.

## Install Docker and Docker compose

1. Docker
   - https://docs.docker.com/engine/install/ubuntu/

2. Docker compose
   - https://docs.docker.com/compose/install/

## Git clone
	git clone https://github.com/morimoriysmoon/sish.git

## Build
At the moment, sish has an issue with Global Request command. sish does not send reply for Global Request command to client unless errors are found. I hope it will be fixed soon. So as to avoid the issue, we need to build docker image with source.

	sudo docker build -t sish-grr-fix 

## Private key for sish
By default, sish will generate private key unless given by user. The algorithm "Ed25519" will be used by default. However, JSch does not support "Ed25519" algorithm. So we need to create own private key with different algorithm.\
The algorithm "ecdsa-sha2-nistp521" can be a good candidate. Please use passphrase for security. We need the passphrase for next step. The private key should be OpenSSH private key format(not Putty format). After creating, please rename as "ssh_key" and put in the folder "deploy/keys".

## Known Hosts
We need know-hosts string to properly connect to SSH service of sish. We can copy and paste it from other devices after successful connection.

## Configuration file for Docker compose
	./deploy/docker-compose.yml

	version: '3.7'

	services:
	reverseproxy:
		image: nginx
		container_name: nginx
		ports:
		- 443:443
		volumes:
		- ./nginx-preread-protocol.conf:/etc/nginx/nginx.conf:ro
		network_mode: host
	letsencrypt:
		image: adferrand/dnsrobocert:latest
		container_name: letsencrypt-dns
		volumes:
		- /var/run/docker.sock:/var/run/docker.sock
		- ./letsencrypt:/etc/letsencrypt
		- ./le-config.yml:/etc/dnsrobocert/config.yml
		restart: always
	sish:
		image: sish-grr-fix:latest
		container_name: sish
		depends_on: 
		- letsencrypt
		volumes:
		- ./letsencrypt:/etc/letsencrypt
		- ./pubkeys:/pubkeys
		- ./keys:/keys
		- ./ssl:/ssl
		- ./log:/log
		command: |
		--authentication=true
		--authentication-password=YOUR-PASSWORD
		--ssh-address=:2222
		--http-address=:80
		--https-address=:8443
		--https=true
		--https-certificate-directory=/ssl
		--https-ondemand-certificate=false
		--https-ondemand-certificate-accept-terms=true
		--https-ondemand-certificate-email=YOUREMAIL@EMAIL.COM
		--private-key-location=/keys/ssh_key
		--private-key-passphrase=YOUR-PASSPHRASE
		--bind-random-ports=false
		--bind-random-subdomains=true
		--bind-random-subdomains-length=8
		--domain=YOUR.DOMAIN
		--debug=false
		--log-to-file=true
		--log-to-file-path=/log/sish.log
		network_mode: host
		restart: always


## Configuration file for DNSrobocert
	./deploy/le-config.yml

	acme:
	  email_account: AUTH_EMAIL
	certificates:
	- autorestart:
	- containers:
	  - sish
	domains:
	- your.domain
	- '*.your.domain'
	  name: your.domain
	  profile: cloudflare
	profiles:
  	- name: cloudflare
	  provider: cloudflare
	  provider_options:
		auth_token: AUTH_TOKEN
		auth_username: AUTH_EMAIL
		zone_id: ZONE_ID

## Configuration file for NGINX
	./deploy/nginx-preread-protocol.conf

	worker_processes auto;

	events {
		worker_connections 1024;
	}

	stream {
		upstream ssh {
			server localhost:2222;
		}

		upstream web {
			server localhost:8443;
		}

		map $ssl_preread_protocol $upstream {
			default ssh;
			"~*TLSv*.*" web;
		}

		# SSH and SSL on the same port
		server {
			listen 443;

			proxy_pass $upstream;
			ssl_preread on;
		}
	}

## Create symbolic links for wildcard certificates
	ln -s /etc/letsencrypt/live/<your domain>/fullchain.pem deploy/ssl/<your domain>.crt\
	ln -s /etc/letsencrypt/live/<your domain>/privkey.pem deploy/ssl/<your domain>.key

"/etc/letsencrypt/" is a directory of the mounted volume of the container, not a directory on Ubuntu.

## Firewall (with iptables)
TCP 80,443 ports should be allowed.

	sudo iptables -I INPUT LINENO -i NIC_ID -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
	sudo iptables -I INPUT LINENO -i NIC_ID -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT

## Run
	sudo docker-compose -f deploy/docker-compose.yml up -d

## Check log message
	tail -f deploy/log/sish.log
