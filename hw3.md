Documentation for HW3
Xiangqian Zhang

Task 1: Install a containerized samba server on the Ubuntu machine.
	a. Install Docker on Ubuntu
		1)sudo apt update
		2)sudo apt install -y docker.io
		3)sudo systemctl start docker
		4)sudo systemctl enable docker
	b. Create a samba share directory
		1)sudo mkdir -p /srv/samba/share
		2)sudo chmod 777 /srv/samba/share
	c. Creae a directory for dockerfile and write a dockerfile for the samba server.
		1)sudo mkdir -p ~/samba-server
		2)sudo nano Dockerfile. docker file:
			# Use the latest Ubuntu base image
			FROM ubuntu:latest

			# Install Samba
			RUN apt-get update && \
    			apt-get install -y samba && \
    			rm -rf /var/lib/apt/lists/*

			# Copy the Samba configuration file
			COPY smb.conf /etc/samba/smb.conf

			# Expose the necessary ports
			EXPOSE 139 445

			# Start the Samba server
			CMD ["smbd", "--foreground", "--no-process-group"]

		
		3)create a 'smb.conf' file:
			[global]
   			workgroup = WORKGROUP
   			security = user
   			map to guest = Bad User
   			dns proxy = no

			[share]
			path = /share
   			browsable = yes
   			writable = yes
   			guest ok = yes
   			read only = no
		4)build and run the docker container
			(might need to install Docker BuildKit with following
			command: export DOCKER_BUILDKIF=1, then sudo systemctl1 
			restart docker, finally type in the below command)	
			sudo docker build -t samba-server .
		5)run the docker container
			sudo docker run -d --name samba -p 139:139 -p 445:445 -v /srv/samba/share:/share samba-server

Task2: Install a containerized Pi-hole DNS Server
	a. create a directory for Pi-hole configuration
		sudo mkdir -p /srv/pi-hole/etc-pihole
		sudo mkdir -p /srv/pi-hole/etc-dnsmasq.d
	b. create a directory for Pi-hole docker configuration
		mkdir -p ~/pi-hole
		cd ~/pi-hole
	c. creat a file named 'docker-compose.yml' with below in the file
		version: "3"

		services:
		pihole:
			container_name: pihole
			image: pihole/pihole:latest
			environment:
			TZ: 'America/Los_Angeles'
			WEBPASSWORD: 'yourpassword' # Set a password for the web interface
			volumes:
			- "/srv/pi-hole/etc-pihole:/etc/pihole"
			- "/srv/pi-hole/etc-dnsmasq.d:/etc/dnsmasq.d"
			ports:
			- "1053:53/tcp"
			- "1053:53/udp"
			- "8080:80/tcp"
			restart: unless-stopped

			d. run Pi-hole with Docker compose:
				(might received command not found error, if so, need to install
				docker compose: sudo curl -L "https://github.com/docker/compose/releases/download/v2.5.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
				then, give permission sudo chmod +x /usr/local/bin/docker-compose) 
				sudo docker-compose up -d

Task3: create a docker compose file for both conatainers
	a. create a directory for docker compose configuration
		mkdir -p ~/docker-compose
		cd ~/docker-compose
	b. creat a file named 'docker-compose.yml':
		version: "3"

		services:
		samba:
			image: samba-server
			build: ./samba-server
			ports:
			- "1139:139"
			- "1445:445"
			volumes:
			- /srv/samba/share:/share

		pihole:
			container_name: pihole
			image: pihole/pihole:latest
			environment:
			TZ: 'America/Los_Angeles'
			WEBPASSWORD: 'yourpassword'
			volumes:
			- "/srv/pi-hole/etc-pihole:/etc/pihole"
			- "/srv/pi-hole/etc-dnsmasq.d:/etc/dnsmasq.d"
			ports:
			- "1053:53/tcp"
			- "1053:53/udp"
			- "8080:80/tcp"
			restart: unless-stopped

			c. run both contianers with docker compose:
				sudo docker-compose up -d

Task4: Add firewall rules on bastion host
	# Allow traffic to Samba ports
	pass in on $ext_if proto tcp from any to any port 1139
	pass in on $ext_if proto tcp from any to any port 1445

	# Allow traffic to Pi-hole ports
	pass in on $ext_if proto tcp from any to any port 1053
	pass in on $ext_if proto udp from any to any port 1053
	pass in on $ext_if proto tcp from any to any port 8080

