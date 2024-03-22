# Flexisip server with flexisip-account-manager on ubuntu 23.10 by docker compose file

## Table of contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
  - [Build `flexisip` docker image from source
    code](#build-flexisip-docker-image-from-source-code)
- [Setup and configurations](#setup-and-configurations)
  - [Setup `flexisip-account-manager`
    repository](#setup-flexisip-account-manager-repository)
  - [Configuration files](#configuration-files)
- [Deploy the services](#deploy-the-services)
  - [Run docker compose file](#run-docker-compose-file)
  - [Setup the Flexisip Account Manager
    server](#setup-the-flexisip-account-manager-server)
  - [Check service ports](#check-service-ports)
- [Access services](#access-services)
  - [Access phpMyAdmin](#access-phpmyadmin)
  - [Access the Flexisip Account Manager](#access-the-flexisip-account-manager)
  - [Access the MariaDB database](#access-the-mariadb-database)


## Introduction

This repository is a forked version of the original repository:
https://github.com/capitalfuse/docker-flexisip, which is updated with required
configurations so you can test on `ubuntu 23.10`.

![schema](docker-flexisip-system.png)

## Prerequisites

### Build `flexisip` docker image from source code

We will build the `flexisip` docker image from the source code using the file
`flexisip/docker/flex-from-src`.

1. Clone the flexisip source code from the git repository.

> [!NOTE]
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

The following configuration files are required to run the `docker-compose` file:

- [`ubuntu23-10/redis/redis.conf`](./ubuntu23-10/redis/redis.conf).
- [`ubuntu23-10/flexisip_conf/flexisip.conf`](./ubuntu23-10/flexisip_conf/flexisip.conf).
- [`ubuntu23-10/.env.flexiapi`](./ubuntu23-10/.env.flexiapi)
  (flexisip-account-manager,Laravel).
- [`ubuntu23-10/nginx/nginx.conf`](./ubuntu23-10/nginx/nginx.conf).

> [!NOTE]
> The `ubuntu23-10/docker-compose.yml` is already updated with the required
> MariaDB environment variables, so we don't need to add the file
> `ubuntu23-10/.env`.

> [!TIP]
> You can refer to the files in the repo or use files already defined in folder
> `ubuntu23-10` for reference:
>
> - [`ubuntu20-04/docker_files/redis/redis.conf`](./ubuntu20-04/docker_files/redis/redis.conf).
> - [`docker/flexisip.conf.sample`](./docker/flexisip.conf.sample).
> - [`ubuntu20-04/.env`](./ubuntu20-04/.env) (for docker compose file).
> - [`flexisip-account-manager/flexiapi/.env`](https://gitlab.linphone.org/BC/public/flexisip-account-manager/-/blob/master/flexiapi/.env.example)
>   (flexisip-account-manager,Laravel)`.
> - [`ubuntu20-04/docker_files/nginx/nginx_default.conf`](./ubuntu20-04/docker_files/nginx/nginx_default.conf).

---

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

3. (Optional): Serve the `flexisip-account-manager` server directly:

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
CONTAINER ID   IMAGE                              COMMAND                  CREATED       STATUS          PORTS                                                                      NAMES
07f56cc27000   phpmyadmin/phpmyadmin:fpm-alpine   "/docker-entrypoint.…"   2 hours ago   Up 44 minutes   9000/tcp                                                                   phpmyadmin-fpm
e45146708df9   flexisip                           "/flexisip-entrypoin…"   2 hours ago   Up 44 minutes                                                                              ubuntu-flexisip
ce3fab6e02a4   ubuntu23-10-php-fpm-laravel        "docker-php-entrypoi…"   2 hours ago   Up 44 minutes   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 9000/tcp                        php-fpm-laravel
be1fe1970bb2   nginx:alpine                       "/docker-entrypoint.…"   2 hours ago   Up 44 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   nginx
3193e93fc232   redis:alpine                       "docker-entrypoint.s…"   2 hours ago   Up 44 minutes   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp                                  redis
de0711c1f7ba   mariadb                            "docker-entrypoint.s…"   2 hours ago   Up 44 minutes   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp                                  flexisip-mariadb
```

For the `ubuntu-flexisip` container, it configured with `network_mode: "host"`,
it's running in the host network, so you can't see the ports in the `docker ps`
command.

As the `flexisip` server should be listen on the port `5060`, you can check the
port using the following command:

```bash
sudo netstat -ntlp | grep LISTEN | grep flexisip
```

> [!NOTE]
> You will have to install `net-tools` package to use the `netstat` command:
>
>  ```bash
>  sudo apt-get install net-tools
>  ```

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

> [!IMPORTANT]
> The `flexisip` server is listening on the port `5060`, but **it won't receive
> any `REGISTER` requests from these addresses**, only if you configure the
> **`reg-domains`** in file
> [`ubuntu23-10/flexisip_conf/flexisip.conf`](./ubuntu23-10/flexisip_conf/flexisip.conf).

## Access services

### Access phpMyAdmin

You can access the `phpMyAdmin` using the following URL:

```bash
http://localhost/phpmyadmin
```

With the credentials defined for `MariaDB` in the `docker-compose.yml` file:

- Root:
  - **Username**: `root`.
  - **Password**: `mysql`.

- User:
  - **Username**: `mysql`.
  - **Password**: `mysql`.

### Access the Flexisip Account Manager

You can access the `Flexisip Account Manager` using the following URL:

```bash
http://localhost/api

# or

http://localhost/documentation
```

- Login page:

  ```bash
  http://localhost/login
  ```

- Register page:

  ```bash
  http://localhost/register
  ```

### Access the MariaDB database

You can access the `MariaDB` database using [Dbeaver](https://dbeaver.io/), with
`phpMyAdmin` as mentioned [above](#access-phpmyadmin), or any other database
client tool with these credentials:

- **Host**: `localhost`.
- **Port**: `3306`.
- Root:
  - **Username**: `root`.
  - **Password**: `mysql`.

- User:
  - **Username**: `mysql`.
  - **Password**: `mysql`.
