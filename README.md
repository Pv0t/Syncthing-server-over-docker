# 1.0 Syncthing server over docker
## 1.1 Introduction
Syncthing is an open‑source project that enables peer‑to‑peer file synchronization across devices. Rather than relying on a central server that stores your data, Syncthing routes traffic through public relay and discovery servers. Some of these servers are operated by the Syncthing team, while others are run by community contributors.Because the architecture is decentralized, your files are never stored on a single, centralized location they’re merely passed through the relays. The open‑source nature of the project also lets you host your own relay and discovery servers. You can choose to contribute those resources back to the community, or keep them private for exclusive use.Running your own relay/discovery service can be done on a virtual private server (VPS) or on self‑hosted hardware. In this guide, I’ll show you how to set up the servers using Docker and configure the containers for optimal security.

## 1.2 What is a Syncthing Relay & Discovery Server?
Relaying is enabled by default, but it is only used when two devices cannot establish a direct connection. While a relay is active, Syncthing continuously attempts to create a direct link; once a direct connection succeeds, the relay is dropped and communication proceeds peer‑to‑peer. The relay simply forwards encrypted packets between the peers. 

To locate peers on the Internet, Syncthing contacts a discovery server. Anyone can run a discovery server and configure their Syncthing instances to use it. In addition, the Syncthing project operates a global, publicly‑available discovery cluster that all clients can fall back to.

[Here](https://relays.syncthing.net/) a list of all public relay.

## 1.3 Requirements
What you’ll need to run your own Syncthing relay  &  discovery server:

- **Hardware** – this can be a small single‑board computer such as a Raspberry  Pi 4, a dedicated server, or a virtual private server (VPS).
- **Public IP address** – the relay and discovery services must be reachable from the Internet.

## 1.4 Docker Installation & Setting up Server-Side
Each container has its own isolated filesystem, network stack, and process space, typically dedicated to a single service (e.g., a chat server). By running the service inside a container, you keep it sandboxed from the host system and from other applications, simplifying deployment, scaling, and security.

```bash
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo apt-get update
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo apt-get install ca-certificates curl
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo install -m 0755 -d /etc/apt/keyrings
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo chmod a+r /etc/apt/keyrings/docker.asc
Pv0t-VPS[/Syncthing-server-over-docker]$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo apt-get update
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo systemctl enable docker.service
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo systemctl enable containerd.service
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo systemctl start docker.service
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo systemctl start containerd.service
```

Hardening permission:
```shell
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo groupadd docker
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo usermod -aG docker $USER
Pv0t-VPS[/Syncthing-server-over-docker]$ newgrp docker
```

## 1.5 Installation Server-Side - docker-compose.yml
First, create a directory where the data and certificates will be stored:
```shell
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo mkdir -p /home/$USER/syncthing/strelaysrv/{strelaysrv-certs,strelaysrv-data}
Pv0t-VPS[/Syncthing-server-over-docker]$ sudo mkdir -p /home/$USER/syncthing/discosrv/{discosrv-certs,discosrv-db}
```

Now let's create your docker-compose.yml configuration:
```bash
Pv0t-VPS[/home/$USER/syncthing]$ vim docker-compose.yml
```

Inside the docker‑compose.yml file, update the two placeholders as follows:
- **{$USER}** – replace this with your username (the account that will own the containers).
- **{$TOKEN}** – insert a strong, private token of your choosing. This secret token acts as a password, ensuring that only you are the only one to use your secthing server relay & discovery. 

```shell
networks:
	syncthing-net:
		driver: bridge
		ipam:
			config:
				- subnet: 10.10.20.0/24
				  gateway: 10.10.20.1
services:
  syncthing-relay:
    image: syncthing/relaysrv:latest
    container_name: strelaysrv
    restart: always
    user: "1000:1000"
    environment:
      - DEBUG=false
    command: -pools="" -listen=:22067 -status-srv=:22070 -protocol=tcp4 -token={$TOKEN} -keys=/strelaysrv-certs/
    ports:
      - "22067:22067"
      - "22070:22070"
    volumes:
      - /home/{$USER}/syncthing/strelaysrv/strelaysrv-certs:/strelaysrv-certs
      - /home/{$USER}/syncthing/strelaysrv/strelaysrv-data:/strelaysrv-data
    networks:
      - syncthing-net
	        ipv4_address: 10.10.20.10
	security_opt:
		- no-new-privileges:true
	cap_drop:
		- ALL


  discovery:
    image: syncthing/discosrv
    container_name: discosrv
    user: "1000:1000"
    restart: always
    ports:
      - "8443:8443"
    volumes:
      - /home/{$USER}/syncthing/discosrv/discosrv-db:/db
      - /home/{$USER}/syncthing/discosrv/discosrv-certs:/certs
    command: --db-dir=/db --cert=/certs/cert.pem --key=/certs/key.pem --listen=:8443
    networks:
	    syncthing-net:
		    ipv4_address: 10.10.20.15
	security_opt:
		- no-new-privileges:true
	cap_drop:
		- ALL
```

```bash
Pv0t-VPS[/home/$USER/syncthing]$ docker-compose up -d
Pv0t-VPS[/home/$USER/syncthing]$ docker ps
CONTAINER ID   IMAGE                            COMMAND                  CREATED        STATUS                  PORTS                                                                                          NAMES
6cc614abf7a4   syncthing/discosrv               "/bin/entrypoint.sh …"   2 months ago   Up 2 weeks (healthy)    0.0.0.0:8443->8443/tcp, :::8443->8443/tcp, 19200/tcp                                           discosrv
5d8ac8299b41   syncthing/relaysrv:latest        "/bin/entrypoint.sh …"   2 months ago   Up 2 weeks (healthy)    0.0.0.0:22067->22067/tcp, :::22067->22067/tcp, 0.0.0.0:22070->22070/tcp, :::22070->22070/tcp   strelaysrv
Pv0t-VPS[/home/$USER/syncthing]$ docker logs 5d8ac8299b41
2025-00-00 00:00:00 INF 2025/00/00 00:00:00 main.go:263: URI: relay://0.0.0.0:22067/?id=XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX
```
Make a note of the ID parameter value (XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX). You’ll need this identifier later when configuring the client setup. I will refer to it as the variable **{$ID}**.

## 1.6 Installation - Client-Side

```bash
Pv0t[/]$ sudo mkdir -p /etc/apt/keyrings
Pv0t[/]$ sudo curl -L -o /etc/apt/keyrings/syncthing-archive-keyring.gpg https://syncthing.net/release-key.gpg
Pv0t[/]$ echo "deb [signed-by=/etc/apt/keyrings/syncthing-archive-keyring.gpg] https://apt.syncthing.net/ syncthing stable-v2" | sudo tee /etc/apt/sources.list.d/syncthing.list
Pv0t[/]$ sudo apt-get update
Pv0t[/]$ sudo apt-get install syncthing
Pv0t[/]$ syncthing &
```

Open a web browser and go to http://127.0.0.1:8384.

- Navigate: Actions -> Settings -> Connections.

    **Sync‑Protocol Listen Address**
    Enter the relay URL:
    ```
    relay://{$VPS_IP}:22067/?id={$ID}&token={$TOKEN}
    ```
    **Global Discovery Server**
    Set the discovery URL to:
    ```
    https://{$VPS_IP}:8443/?id={$ID}
    ```

Replace the placeholders as follows:
- **{$VPS_IP}** – the public IP address of the machine hosting your Syncthing relay & discovery services.
- **{$ID}** – the relay ID you recorded earlier (the long string from the Docker logs).
- **{$TOKEN}** – the secret token you defined in the docker‑compose.yml file.

After filling in these values, save the settings. Your client will now be configured to use your personal self‑hosted/VPS relay and discovery server.

# 2.0 Reference
- **Syncthing Relay Server**     | https://docs.syncthing.net/users/strelaysrv.html
- **Public Relay Server**        | https://relays.syncthing.net/
- **Syncthing Discovery Server** | https://docs.syncthing.net/users/stdiscosrv.html
- **Docker 'relaysrv' image**    | https://hub.docker.com/r/syncthing/relaysrv/
- **Docker 'discosrv' image**    | https://hub.docker.com/r/syncthing/discosrv/
