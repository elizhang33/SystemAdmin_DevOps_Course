HW4 documentation

1. Install a containerized Wireguard server. 
	a. create a directory for wireguard config: 
		mkdir -p ~/wireguard
		cd ~/wireguard
	b. create a docker compose file:
		nano docker-compose.yml
	c. add the following content:
version: "3.8"

services:
  wireguard:
    container_name: wireguard
    image: linuxserver/wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles 
      - SERVERURL=vpn.pdx.edum 
      - SERVERPORT=51820
      - PEERS=1 # Number of clients you want to configure
      - PEERDNS=auto
    volumes:
      - ./config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv6.conf.all.forwarding=1
    restart: unless-stopped

	d. run docker compose:
		sudo docker-compose up -d

	e. verify vpn connectivity: 
		retrieve the peer1 config
		set up a wireguard client on local device and connect to the vpn

2. add wireguard to docker compose file
	a.open the docker-compose file
version: "3.8"

services:
  wireguard_vpn:
    container_name: wireguard_vpn
    image: linuxserver/wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles 
      - SERVERURL=vpn.pdx.edu 
      - SERVERPORT=51820
      - PEERS=1 # Number of clients you want to configure
      - PEERDNS=auto
    volumes:
      - ./wireguard/config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv6.conf.all.forwarding=1
    restart: unless-stopped

	b. run docker compose to start the vpn.

3. install a containerized wazuh server
	a. create directory for wazuh config
	b. add content to docker-compose.yml for wazuh config:

	wazuh:
    container_name: wazuh
    image: wazuh/wazuh:4.2.5
    ports:
      - "1514:1514/udp"
      - "1515:1515/tcp"
    environment:
      - WAZUH_MANAGER_IP=0.0.0.0
    volumes:
      - ./wazuh-data:/var/ossec/data
    restart: unless-stopped

  elasticsearch:
    container_name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.2
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    restart: unless-stopped

  logstash:
    container_name: logstash
    image: docker.elastic.co/logstash/logstash:7.10.2
    environment:
      - LS_JAVA_OPTS="-Xms1g -Xmx1g"
    volumes:
      - ./logstash-config:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"
    restart: unless-stopped

  kibana:
    container_name: kibana
    image: docker.elastic.co/kibana/kibana:7.10.2
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    restart: unless-stopped

volumes:
  esdata:
    driver: local

	c. deploy the wazuh stack:
		sudo docker-compose up -d
		(check running containners- sudo docker ps)

	d. install Wazuh agent on FreeBSD
		sudo pkg install wazuh-agent
	e. edit the Wazuh agent config file
		sudo nano /usr/local/etc/wazuh-agent/ossec.conf
	f. edit IP to 192.168.33.111 inside the  config file at /var/ossec/etc/ossec.conf 
		<client>
    		    <server>
      			<address>192.168.33.111</address>
      			<port>1514</port>
			<protocol>udp</protocol>
		    </server>
		</client>
	g. start wazuh agent
		set wazuh_agent_enable="YES" inside rc.conf file at /etc
		service wazuh-agent start

Task5: write a docker compose filefor git lab. 
	a. open the docker-compose.yml and add the following:
		gitlab:
    container_name: gitlab
    image: gitlab/gitlab-ee:latest
    ports:
      - "80:80"
      - "443:443"
      - "22:22"
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.cecs.pdx.edu/xiangqz'
    volumes:
      - ./config:/etc/gitlab
      - ./logs:/var/log/gitlab
      - ./data:/var/opt/gitlab
    restart: unless-stopped

	b. run docker compose:
		sudo docker-compose up -d

	
Task6: install bitwarden instance.
	a. add following to the docker-compse file:
		bitwarden:
    container_name: bitwarden
    image: bitwardenrs/server:latest
    environment:
      WEBSOCKET_ENABLED: "true" 
      SIGNUPS_ALLOWED: "true"   
      ADMIN_TOKEN: "my_admin_token_fake" 
    ports:
      - "80:80"
      - "3012:3012"
    volumes:
      - ./data:/data
    restart: unless-stopped
	
	b. deploy the bitwarden container:
		sudo docker-compose up -d


Task7: perform security testing with SAST and DAST tools
	a. set up sonarqube for SAST by adding following to docker-compose.yml
		sonarqube:
    container_name: sonarqube
    image: sonarqube:latest
    environment:
      - SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true
    ports:
      - "9000:9000"
    volumes:
      - ./sonarqube_conf:/opt/sonarqube/conf
      - ./sonarqube_data:/opt/sonarqube/data
      - ./sonarqube_logs:/opt/sonarqube/logs
      - ./sonarqube_extensions:/opt/sonarqube/extensions
    restart: unless-stopped

	b. deploy sonarqube:
		sudo docker-compose up -d
	c. set up OWASP ZAp for DAST:
		docker pull owasp/zap2docker-stable
		docker run -u zap -p 8080:8080 owasp/zap2docker-stable zap-webswing.sh

Task8: integrate SAST?DAST tools into gitlab CI/CD pipeline
	a. edit .gitlab-ci.yml file to add following:
		stages:
  - build
  - test
  - deploy
  - sast
  - dast

sast:
  stage: sast
  image: sonarsource/sonar-scanner-cli:latest
  script:
    - sonar-scanner -Dsonar.projectKey=my_project_key -Dsonar.sources=. -Dsonar.host.url=http://sonarqube:9000 -Dsonar.login=your_token

dast:
  stage: dast
  image: owasp/zap2docker-stable
  script:
    - zap-baseline.py -t http://<your-app-url> -r testreport.html
  artifacts:
    paths:
      - testreport.html


	b. push the change to gitlab:
		git pusho

		







