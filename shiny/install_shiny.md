
# Install

## Install Docker (see official doc)

```sh
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install  apt-transport-https ca-certificates curl  gnupg  lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo g	pg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo   "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
# Check that docker works
sudo docker run hello-world
sudo service docker status
```

### Modify docker service entry to listen to port 2375

Edit `/etc/systemd/system/multi-user.target.wants/docker.service`
and change the line that starts with `ExecStart` as:
```ini
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock -H fd:// --containerd=/run/containerd/containerd.sock
```
Then reload systemctl & restart the docker service:

```sh
sudo systemctl daemon-reload
sudo systemctl restart docker.service
sudo systemctl status docker.service
```

Check that port 2375 is listened to:
```sh
ss -tunapl | grep 2375
# should output
# tcp    LISTEN  0       4096                 *:2375               *:*
```

## Install caddy  [official doc](https://caddyserver.com/docs/install)

```sh
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo apt-key add -
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
# check that caddy works
caddy version
```
### Edit the Caddyfile

`/etc/caddy/Caddyfile`

```
# The Caddyfile is an easy way to configure your Caddy web server.

shiva.ciml.univ-amu.fr {
	# Set this path to your site's directory.
	# root * /usr/share/caddy

	# Enable the static file server.
	file_server

	# Another common task is to set up a reverse proxy:
	reverse_proxy 127.0.0.1:8080
}

# Refer to the Caddy docs for more information:
# https://caddyserver.com/docs/caddyfile
```


## Install Shiny (need java 8 not > 10)

```sh
java -version
sudo apt install openjdk-8-jre-headless
# Must be 1.8
java -version
```

### Create the `/opt/shinyproxy` directory

```sh
cd opt/
sudo mkdir shinyproxy
cd shinyproxy/
sudo chown $USER .
sudo chgrp docker .
```

### Install the deb package (will setup the daemon)
(see https://shinyproxy.io/documentation/deployment/#deb-package)


```
wget https://www.shinyproxy.io/downloads/shinyproxy_2.5.0_amd64.deb
sudo dpkg -i shinyproxy_2.5.0_amd64.deb
# could be cleaner I know
sudo apt --fix-broken install
```

### Fix the service file

This is `/etc/systemd/system/shinyproxy.service`

```ini
[Unit]
Description=ShinyProxy
After=syslog.target network.target

[Service]
Type=simple
# I changed the username & group
User=admincenturi
Group=docker
WorkingDirectory=/etc/shinyproxy
ExecStart=/usr/bin/java -jar /opt/shinyproxy/shinyproxy.jar
KillMode=process
StandardOutput=journal
StandardError=journal
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

### Configure ShinyProxy


The configuration is done through: `/opt/shinyproxy/application.yml`

```yml
proxy:
  title: CIML Shiny Server
  logo-url: http://www.ciml.univ-mrs.fr/themes/custom/ciml_theme/logo.svg
  landing-page: /
  heartbeat-rate: 10000
  heartbeat-timeout: 60000
  port: 8080
  authentication: simple
  admin-groups: scientists
  # Example: 'simple' authentication configuration
  users:
  - name: admincenturi
    password: XXXX
    groups: scientists
  - name: testuser
    password: XXXX
    groups: scientists
  # Docker configuration
  docker:
    cert-path: /home/none
    url: http://localhost:2375
    port-range-start: 20000
  specs:
  - id: 01_hello
    display-name: Hello Application
    description: Application which demonstrates the basics of a Shiny app
    container-cmd: ["R", "-e", "shinyproxy::run_01_hello()"]
    container-image: openanalytics/shinyproxy-demo
    access-groups: scientists
  - id: 02_shiva
    display-name: Shiva
    container-cmd: ["R", "-e", "shiny::runApp('/root/Shiva', host='0.0.0.0', port=3838)"]
    container-image: shiva
    access-groups: scientists

logging:
  file:
    shinyproxy.log
  level:
    DEBUG
```

####  Test app

```sh
sudo docker pull openanalytics/shinyproxy-demo
```

### Get Shiva archive and load it with docker:
docker load -i shiva.tar

## Start the service

```sh
sudo systemctl daemon-reload
sudo systemctl restart shinyproxy.service
sudo systemctl status shinyproxy.service
```

> That should be it!
