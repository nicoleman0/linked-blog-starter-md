###### [](https://hub.docker.com/r/honeynet/conpot#via-a-pre-built-image)Via a pre-built image

1. Install [Docker](https://docs.docker.com/engine/installation/)
2. Run `docker pull honeynet/conpot`
3. Run `docker run -it -p 80:80 -p 102:102 -p 502:502 -p 161:161/udp --network=bridge honeynet/conpot:latest /bin/sh`
4. Finally run `conpot -f --template default`

Navigate to `http://MY_IP_ADDRESS` to confirm the setup.

###### [](https://hub.docker.com/r/honeynet/conpot#build-docker-image-from-source)Build docker image from source

1. Install [Docker](https://docs.docker.com/engine/installation/)
2. Clone this repo with `git clone https://github.com/mushorg/conpot.git` and `cd conpot/docker`
3. Run `docker build -t conpot .`
4. Run `docker run -it -p 80:8800 -p 102:10201 -p 502:5020 -p 161:16100/udp -p 47808:47808/udp -p 623:6230/udp -p 21:2121 -p 69:6969/udp -p 44818:44818 --network=bridge conpot`

Navigate to `http://MY_IP_ADDRESS` to confirm the setup.

###### [](https://hub.docker.com/r/honeynet/conpot#build-from-source-and-run-with-docker-compose)Build from source and run with docker-compose

1. Install [docker-compose](https://docs.docker.com/compose/install/)
2. Clone this repo with `git clone https://github.com/mushorg/conpot.git` and `cd conpot/docker`
3. Build the image with `docker-compose build`
4. Test if everything is running correctly with `docker-compose up`
5. Permanently run as a daemon with `docker-compose up -d`


---

**Run directly without /bin/sh (easiest)**

```bash
docker run --rm -p 8800:80 -p 102:102 -p 502:502 -p 161:161/udp honeynet/conpot:latest
```

