# Flexisip server with flexisip-account-manager on ubuntu 23.10 by docker compose file

## Table of contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
  - [Specifications](#specifications)
  - [Build `flexisip` docker image from source
    code](#build-flexisip-docker-image-from-source-code)
- [Setup and configurations](#setup-and-configurations)
  - [Setup `flexisip-account-manager`
    repository](#setup-flexisip-account-manager-repository)
  - [Configuration files](#configuration-files)
    - [Base configuration files](#base-configuration-files)
    - [Update base ip address](#update-base-ip-address)
    - [Reference configuration files](#reference-configuration-files)
    - [Get the default `flexisip.conf` configuration
      file](#get-the-default-flexisipconf-configuration-file)
- [Deploy the services](#deploy-the-services)
  - [Run docker compose file](#run-docker-compose-file)
  - [Setup the Flexisip Account Manager
    server](#setup-the-flexisip-account-manager-server)
  - [Check service ports](#check-service-ports)
- [Access services](#access-services)
  - [Access phpMyAdmin](#access-phpmyadmin)
  - [Access the Flexisip Account Manager](#access-the-flexisip-account-manager)
  - [Access the MariaDB database](#access-the-mariadb-database)
- [Other documentations](#other-documentations)
- [FAQ](#faq)

## Introduction

This repository is a forked version of the original repository:
https://github.com/capitalfuse/docker-flexisip, which is updated with required
configurations so you can test on `ubuntu 23.10`.

Support features:

- [x] Flexisip proxy server.
  - [x] Flexisip proxy server authentication.
  - [ ] Flexisip proxy server TLS.
- [x] Flexisip account manager.
  - [x] Flexisip account manager TLS (Check [here](./docs/setup-tls.md)).
- [x] phpMyAdmin web interface.
- [ ] Flexisip conference server.
- [ ] Flexisip presence server.

![schema](docker-flexisip-system.png)

## Prerequisites

### Specifications

This project is tested on the following specifications:

- **OS**: `Ubuntu 23.10`.
- **Docker**: `26.0.0`.
- **flexisip** ([GitHub](https://github.com/BelledonneCommunications/flexisip)):
  `8501f5b` (build Docker image).

### Build `flexisip` docker image from source code

We will build the `flexisip` docker image from the source code using the file
`flexisip/docker/flex-from-src`.

1. Clone the flexisip source code from the git repository.

> [!IMPORTANT]
> You have to clone the repository **"recursive"** to get the submodules.

- From Github:

  ```bash
  git clone https://github.com/BelledonneCommunications/flexisip.git --recursive
  ```

- From Gitlab:

  ```bash
  git clone https://gitlab.linphone.org/BC/public/flexisip.git --recursive
  ```

> [!NOTE]
> The clone process may take sometimes depending on your internet speed.

2. Update the Dockerfile:

- Add the following lines to the file `flexisip/docker/flex-from-src`:

  ```diff
  # flexisip/docker/flex-from-src

  ...
  # Get source code
  COPY --chown=bc:bc . flexisip/
  +
  + RUN cd flexisip && ./docker/cmake_install.sh 3.22.6
  +
  # Configure & build
  RUN cd flexisip \
      && rm -rf work && mkdir work \
      && cmake -S . -B work -G Ninja -DCMAKE_BUILD_TYPE=${build_type} -DENABLE_SANITIZERS=${sanitizer} -DCMAKE_INSTALL_PREFIX=/opt/belledonne-communications -DSYSCONF_INSTALL_DIR=/etc \
      && cmake --build work \
      && sudo cmake --build work --target install
  ...
  ```

  This will update the cmake version to `3.22.6` to pass the `cmake` version
  requirements defined in the `flexisip/CMakeLists.txt` file.

3. Build the `flexisip` docker image:

```bash
docker build -t flexisip --build-arg='njobs=<njobs>' -f docker/flex-from-src .
```

## Setup and configurations

### Setup `flexisip-account-manager` repository

1. Clone the `flexisip-account-manager` repository from the git repository.

- From Github:

  ```bash
  git clone https://github.com/BelledonneCommunications/flexisip-account-manager.git
  ```

- From Gitlab:

  ```bash
  git clone https://gitlab.linphone.org/BC/public/flexisip-account-manager.git
  ```

2. Move to the `ubuntu23-10` directory:

```bash
mv flexisip-account-manager ./ubuntu23-10/
```

3. Change file permissions:

> [!IMPORTANT]
> Prevent 500 error from:
> https://stackoverflow.com/questions/45206228/500-internal-server-error-with-laravel-docker

```bash
cd ubuntu23-10/flexisip-account-manager/flexiapi

sudo chmod -R 777 storage && sudo chmod -R 777 bootstrap/cache
```

### Configuration files

#### Base configuration files

The following configuration files are required to run the `docker-compose` file:

- [`ubuntu23-10/redis/redis.conf`](./ubuntu23-10/redis/redis.conf).
- [`ubuntu23-10/flexisip_conf/flexisip.conf`](./ubuntu23-10/flexisip_conf/flexisip.conf).
- [`ubuntu23-10/.env.flexiapi`](./ubuntu23-10/.env.flexiapi)
  (flexisip-account-manager,Laravel).
- [`ubuntu23-10/nginx/nginx.conf`](./ubuntu23-10/nginx/nginx.conf).


#### Update base ip address

By default, all the configuration files are configured with the IP address
`192.168.1.17`, with subdomains. You have to update the IP address to your own
IP address manually in the following files:

- [`ubuntu23-10/flexisip_conf/flexisip.conf`](./ubuntu23-10/flexisip_conf/flexisip.conf).
- [`ubuntu23-10/.env.flexiapi`](./ubuntu23-10/.env.flexiapi).

Or you can use the `perl` command to update the IP address with your own IP
address (**recommended**):

```bash
cd ubuntu23-10

REPLACE_IP="192.168.1.127"
LOCAL_IP="192.168.1.17"

perl -i -pe 's/'"$LOCAL_IP"'/'"$REPLACE_IP"'/g' ./.env.flexiapi ./flexisip_conf/flexisip.conf
```

Or you can use the `perl` command to update the IP address with your own IP
address and **remove the subdomains**:

```bash
cd ubuntu23-10

REPLACE_IP="192.168.1.127"
LOCAL_IP="192.168.1.17"

perl -i -pe 's/(?<=[=\"])(\w*\.)?'"$LOCAL_IP"'\.nip\.io/'"$REPLACE_IP"'/g' ./.env.flexiapi ./flexisip_conf/flexisip.conf
```

#### Reference configuration files

- [`ubuntu20-04/docker_files/redis/redis.conf`](./ubuntu20-04/docker_files/redis/redis.conf).
- [`docker/flexisip.conf.sample`](./docker/flexisip.conf.sample).
- [`ubuntu20-04/.env`](./ubuntu20-04/.env) (for docker compose file).
- [`flexisip-account-manager/flexiapi/.env`](https://gitlab.linphone.org/BC/public/flexisip-account-manager/-/blob/master/flexiapi/.env.example)
  (flexisip-account-manager,Laravel)`.
- [`ubuntu20-04/docker_files/nginx/nginx_default.conf`](./ubuntu20-04/docker_files/nginx/nginx_default.conf).

#### Get the default `flexisip.conf` configuration file

To get the default `flexisip.conf` [config
file](https://wiki.linphone.org/xwiki/wiki/public/view/Flexisip/Configuration/),
you can run the following command:

```bash
docker exec ubuntu-flexisip flexisip --dump-all-default > flexisip.conf
```

## Deploy the services

### Run docker compose file

> [!NOTE]
> If you are using different `flexisip` image, you have to update the image name
> for `ubuntu-flexisip` service in the `docker-compose.yml` file.

1. Move to the `ubuntu23-10` directory:

```bash
cd ubuntu23-10
```

2. Run the `docker-compose` file:

```bash
docker-compose up
```

### Setup the Flexisip Account Manager server

1. Enter the `flexisip-account-manager` container:

```bash
docker exec -it php-fpm-laravel /bin/bash
```

2. Run the following commands to setup the `flexisip-account-manager` server:

```bash
cd flexiapi

composer install

php artisan key:generate

php artisan migrate
```

3. **(Optional)**: Serve the `flexisip-account-manager` server directly:

> [!NOTE]
> Please make sure to finish step 2 before running the following commands.

```bash
docker exec -it php-fpm-laravel /bin/bash

php artisan serve --host=0.0.0.0
```

Then you can access the `flexisip-account-manager` server routes using the URL:

```bash
http://localhost:8000

# or

http://localhost:8000/documentation

# or

http://localhost:8000/api
```

### Check service ports

After running the `docker-compose` file, you can check the service ports using
the following command:

```bash
docker ps
```

You will get the following output:

```bash
CONTAINER ID   IMAGE                              COMMAND                  CREATED         STATUS         PORTS                                                                      NAMES
c36887f00b26   flexisip                           "/flexisip-entrypoin…"   2 minutes ago   Up 2 minutes   0.0.0.0:5060-5061->5060-5061/tcp, :::5060-5061->5060-5061/tcp              ubuntu-flexisip
dfb4f558e7d3   nginx:alpine                       "/docker-entrypoint.…"   9 hours ago     Up 2 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   nginx
b533a69ea894   phpmyadmin/phpmyadmin:fpm-alpine   "/docker-entrypoint.…"   34 hours ago    Up 2 minutes   9000/tcp                                                                   phpmyadmin-fpm
9d285f04e50c   ubuntu23-10-php-fpm-laravel        "docker-php-entrypoi…"   34 hours ago    Up 2 minutes   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 9000/tcp                        php-fpm-laravel
2421c502e2ea   redis:alpine                       "docker-entrypoint.s…"   34 hours ago    Up 2 minutes   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp                                  redis
8cd52def74d5   mariadb                            "docker-entrypoint.s…"   34 hours ago    Up 2 minutes   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp                                  flexisip-mariadb
```

Since the `network_mode` is [not compatible with other
OS](https://docs.docker.com/network/drivers/host/#:~:text=The%20host%20networking%20driver%20only%20works%20on%20Linux%20hosts%2C%20and%20is%20not%20supported%20on%20Docker%20Desktop%20for%20Mac%2C%20Docker%20Desktop%20for%20Windows%2C%20or%20Docker%20EE%20for%20Windows%20Server.),
it's changed to expose the ports to the host machine.

> [!NOTE]
> Remove the `network_mode: "host"` means you can't access the `flexisip` server
> from the host machine, so you **no longer can use `wireshark` to capture the SIP
> packets**.

<details>
<summary>Debug when using <code>network_mode: "host"</code></summary>

For the `ubuntu-flexisip` container, it configured with `network_mode: "host"`,
it's running in the host network, so you can't see the ports in the `docker ps`
command.

As the `flexisip` server should be listen on the port `5060`, you can check the
port using the following command:

```bash
sudo netstat -ntlp | grep LISTEN | grep flexisip
```

You will get the following output:

```bash
tcp        0      0 127.0.0.1:5060          0.0.0.0:*               LISTEN      61046/flexisip
tcp        0      0 172.17.0.1:5060         0.0.0.0:*               LISTEN      61046/flexisip
tcp        0      0 172.18.0.1:5060         0.0.0.0:*               LISTEN      61046/flexisip
tcp        0      0 192.168.1.17:5060       0.0.0.0:*               LISTEN      61046/flexisip
tcp6       0      0 2402:800:631d:23a9:5060 :::*                    LISTEN      61046/flexisip
tcp6       0      0 ::1:5060                :::*                    LISTEN      61046/flexisip
tcp6       0      0 2402:800:631d:23a9:5060 :::*                    LISTEN      61046/flexisip
```

</details>

> [!IMPORTANT]
> The `flexisip` server is listening on the port `5060`, but **it won't receive
> any `REGISTER` requests from these domains**, only if you configure the
> **`reg-domains`** in file
> [`ubuntu23-10/flexisip_conf/flexisip.conf`](./ubuntu23-10/flexisip_conf/flexisip.conf).
>
> However, we still can register with **`localhost`**, which is the default value of `reg-domains`.

## Access services

### Access phpMyAdmin

You can access the `phpMyAdmin` using the following URL:

```bash
http://localhost/phpmyadmin
```

With the credentials defined for `MariaDB` in the `docker-compose.yml` file:

- Root user:
  - **Username**: `root`.
  - **Password**: `mysql`.

- Normal user:
  - **Username**: `mysql`.
  - **Password**: `mysql`.

### Access the Flexisip Account Manager

You can access the `Flexisip Account Manager` using the following URL:

- API documentation:

  ```bash
  http://account.<YOUR-LOCAL-IP>.nip.io/api
  ```

- Swagger documentation:

  ```bash
  http://account.<YOUR-LOCAL-IP>.nip.io/documentation
  ```

- Login page:

  ```bash
  http://account.<YOUR-LOCAL-IP>.nip.io/login
  ```

- Register page:

  ```bash
  http://account.<YOUR-LOCAL-IP>.nip.io/register
  ```

### Access the MariaDB database

You can access the `MariaDB` database using [Dbeaver](https://dbeaver.io/), with
`phpMyAdmin` as mentioned [above](#access-phpmyadmin), or any other database
client tool with these credentials:

- **Host**: `localhost`.
- **Port**: `3306`.
- Root user:
  - **Username**: `root`.
  - **Password**: `mysql`.

- Normal user:
  - **Username**: `mysql`.
  - **Password**: `mysql`.

## Other documentations

- [SIP Authentication Process](./docs/sip-auth-process.md).
- [Setup Softphone](./docs/setup-softphone.md).
- [Setup TLS](./docs/setup-tls.md).
- [Setup flexisip Proxy Server Authentication](./docs/setup-flexisip-auth.md).

## FAQ

- I can't register my account with softphone clients

  - Wrong password.

  - Check the email domain registered from the `flexisip-account-manager`
    server whether it is included in **BOTH** `reg-domains` and `auth-domains` in file
    [`ubuntu23-10/flexisip_conf/flexisip.conf`](./ubuntu23-10/flexisip_conf/flexisip.conf).

  - Softphone client doesn't support hash algorithm configured in the
    `flexisip.conf` file. If you want the `flexisip` server to be compatible
    with most of the softphone clients, you can switch the
    `available-algorithms` to `MD5` in `env.flexiapi`. Also, you have to update
    the algorithm in the `flexisip.conf` server.

> [!CAUTION]
> Changing the hash algorithm **will make the user no longer access to their account**.

- I can't access the account manager web page

  - Make sure you update the IP address in the `.env.flexiapi` and `flexisip.conf`
    files as mentioned in the [Update base ip address](#update-base-ip-address)
    section.

  - Make sure you have setup the flexisip-account-manager server as mentioned in
    the [Setup the Flexisip Account Manager server](#setup-the-flexisip-account-manager-server)
    section.

- I can't build the `flexisip` docker image

  - Make sure you cloned the `flexisip` source code with the submodules (recursive).

  - Make sure you have followed the steps in the [Build `flexisip` docker image
    from source code](#build-flexisip-docker-image-from-source-code) section.

  - Check the `cmake` version in the `flexisip/CMakeLists.txt` file. If it is
    higher than the version defined in the `flexisip/docker/flex-from-src`
    file, you have to update the `cmake` version in the `flexisip/docker/flex-from-src`
    file.

  - Try to clone the `flexisip` source code that matches the `flexisip` version
    mentioned in the [Prerequisites](#prerequisites) section.

- On MacOS, I can't register my account with the softphone client

  - You have to use `TCP` as the transport protocol, because the `UDP` is not
    working properly. But on Linux, you can use `UDP` without any issues.
