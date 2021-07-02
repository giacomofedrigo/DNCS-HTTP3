# DNCS-HTTP3
## About 🔍

### Team

Team members are **Luigi Dorigatti** (``) and **Giacomo Fedrigo** (`204788`).

### Project

The goal of the project is to build a virtualized framework for analyzing the performance of HTTP/3 + QUIC, with respect to HTTP/2 or TCP.

**Suggested software**: Vagrant, OpenVSwitch, docker, or alternatively mininet+docker (Comnetsemu)
**Reference software**: https://blog.cloudflare.com/experiment-with-http-3-using-nginx-and-quiche/

## Lab Environment 🌍

In order to be as unbiased as possibile, and also to make the performance evaluation replicable by everyone, it is necessary a virtualized lab. More specifically, in the implementation chosen, two softwares are used to set up the environment: **Docker** and **Vagrant**.
In order to replicate a realistic scenario, the setup will be the following: one host, used as a client is connected directly to the only router of the lab. Than there are 2 hosts used as servers and belonging to the same subnet (different from the client one) connected to a switch and so to the router. The fist host will be named `client`, and on top of it will run the software needed for the performance evaluation (that will be discussed in a dedicated section). On the other hand, the second host will be called `web-server` and will run 3 different Docker containers and similarly the last one will be called `video-server` and it will run the remaining 3 containers.

<img src="media/Network-topology.png" width="1000">

For the performance evaluation to be likely realistic, it would have to include both web-page static contents and also video streaming (the most popular medium nowadays). Hence the need of 6 different Docker containers: `(3 protocols to be tested) X (2 kinds of media)`. The containers could run on top the same host, but for the HTTP/3+QUIC container to work, it needs to use port `80` and `443`, and since there are 2 of those, it would be a problem. For this specific reason the containers are divided in 2 different hosts (one dedicated to web static contents, and the other dedicated to video streaming).

| Service         | Protocol      | IP address  | Ports   |
| --------------- | ------------- | ----------- | ------- |
| Web page        | TCP           | 192.168.2.2 | 82, 452 |
| Web page        | HTTP/2        | 192.168.2.2 | 81, 451 |
| Web page        | HTTP/3 + QUIC | 192.168.2.2 | 80, 443 |
| Video streaming | TCP           | 192.168.2.3 | 82, 452 |
| Video streaming | HTTP/2        | 192.168.2.3 | 81, 451 |
| Video streaming | HTTP/3 + QUIC | 192.168.2.3 | 80, 443 |

Regarding the IP addresses assigned by Vagrant to the different hosts, they are the following: `router::eth1` is `192.168.1.1`, `router::eth2` is `192.168.2.1`, `client::eth1` is `192.168.1.2` and `web-server::eth1` is `192.168.2.2` and `video-server::eth1` is `192.168.2.3`.

## Vagrant Configuration 🖥

As showcased earlier, Vagrant is used to manage the VM and networking side of the Lab environment. The imaged used for the OS is `ubuntu/bionic64`.
Some things have to be pointed out: the X11 server is forwarded in order to use performance evaluation tools and browsers form the `client` (it will be necessary for the host machine to run an X-server, like XQuartz for macOS). This is achieved by adding in the Vagrantfile the following lines:

```Ruby
  config.ssh.forward_agent = true
  config.ssh.forward_x11 = true
```

Also, both the `client` and the `server` have 1024 MB of RAM in order to be capable of running Google Chrome (the first) and ffmpeg (the last).
All the provisioning scripts are in the `vagrant` folder and are used mainly for routing and the installation of basic software.
The provisioning script responsible for the Docker deployment (`docker_run.sh`) is also contained in the same folder as the others, but will be discussed later on.
Last but not least, it is important to know that docker images won't be compiled at each `vagrant up`, but instead downloaded from the [Docker Hub](https://hub.docker.com/u/giovannibaccichet) (which automatically builds them every time something is committed to this repository), in order to save time.

## Docker Configuration 🐳

The approach chosen was to build 2 different Docker images, deploying 6 containers: the first one serves the purpose of running a web-server, whereas the last one is used to stream HLS video. The video-streaming image is a mod of the web-server one and they are both based on NGINX server, more specifically NGINX 1.16.1 (this particular version is needed to run the quiche-patch). In fact both images are pre-configured as HTTP/3 capable, but will be limited in the `nginx.conf` configuration file to run on TCP, HTTP/2 and HTTP/3+QUIC as demanded (in order to complete the performance evaluation).

### SSL Certificates

Since QUIC needs encryption in order to work properly, SSL/TLS certificates had to be generated. After a little bit of research, it turned out that self-signed SSL certificates cannot be used with QUIC: only trusted SSL certificates issued by a CA work.
Let's Encrypt can be used to generate valid certificated, associated with a real domain. Using certbot and DNS verification, the command is the following:

```bash
sudo certbot -d HOSTNAME --manual --preferred-challenges dns certonly
```

And the files needed for NGINX can be found in `/etc/letsencrypt/live/HOSTNAME`.
It is useful to create a subdomain redirecting to `192.168.2.2` (in this case **web.bacci.dev**) or `192.168.2.3` (in this case **video.bacci.dev**) associated with said certificates. This is necessary to connect to the docker containers from the `client` inside the Vagrant environment because as highlighted earlier QUIC accepts only encrypted traffic.
For obvious reasons the certificates used in this performance evaluation are not included in the package, however they can be generated with ease using the command above and passed to docker using the `-v` option (this will be explained in the Deployment section).
