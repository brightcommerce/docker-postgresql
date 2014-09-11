# Docker PostgreSQL

Dockerfile to build a PostgreSQL v9.3.5 container image with contribs and postgis which can be linked to other containers.

## Table of Contents

- [Version](#version)
- [Installation](#installation)
- [How To Use](#how-to-use)
- [Configuration](#configuration)
    - [Ports](#ports)
    - [Data Store](#data-store)
    - [Securing The Server](#securing-the-server)
- [Shell Access](#shell-access)
- [Upgrading](#upgrading)
- [Credit](#credit)

## Version

The current release (v9.3.5) contains scripts to install PostgreSQL v9.3.5, and uses the Brightcommerce Ubuntu 14.04 base image. Our version numbers will reflect the version of PostgreSQL being installed.

## Installation

Pull the latest version of the image from the Docker Index. This is the recommended method of installation as it is easier to update the image in the future. These builds are performed by the **Docker Trusted Build** service.

``` bash
docker pull brightcommerce/postgresql:latest
```

or specify a tagged version:

``` bash
docker pull brightcommerce/postgresql:9.3.5
```

Alternately you can build the image yourself:

``` bash
git clone https://github.com/brightcommerce/docker-postgresql.git
cd docker-postgresql
docker build -t="$USER/postgresql" .
```

## How To Use

Run the PostgreSQL image:

``` bash
docker run --name postgresql -d brightcommerce/postgresql:latest
```

By default remote logins are permitted to the PostgreSQL server and a random password is assigned for the `postgres` user. The password set for the `postgres` user can be retrieved from the container logs.

``` bash
docker logs postgresql
```

In the output you will notice the following lines with the password:

``` bash
|------------------------------------------------------------------|
| PostgreSQL User: postgres, Password: xxxxxxxxxxxxxx              |
|                                                                  |
| To remove the PostgreSQL login credentials from the logs, please |
| make a note of password and then delete the file pwfile          |
| from the data store.                                             |
|------------------------------------------------------------------|
```

Assuming you have `psql` installed on the host, you can test if the PostgreSQL server is working properly by connecting to the server:

``` bash
psql -U postgres -h $(docker inspect --format {{.NetworkSettings.IPAddress}} postgresql)
```

If you don't have `psql` installed on the host you will need to shell into the container and perform the `psql` tests there. Since we are running CoreOS at DigitalOcean, we have a readonly filesystem. This means we can't install tools the traditional way.

See [Shell Access](#shell-access) for instructions on how to install `nsenter` and `docker-enter` on a host. Once this is installed you can use `docker-enter` to shell into the container and play with `psql`.

``` bash
docker inspect --format {{.NetworkSettings.IPAddress}} postgresql
172.17.0.28
sudo docker-enter postgresql
root@965a5c5d2e55:~# psql -U postgres -h 172.17.0.28
Password for user postgres: xxxxxxxxxxxxxx
psql (9.3.5)
Type "help" for help.

postgres=# \list
                             List of databases
   Name    |  Owner   | Encoding | Collate | Ctype |   Access privileges
-----------+----------+----------+---------+-------+-----------------------
 postgres  | postgres | UTF8     | C       | C     |
 template0 | postgres | UTF8     | C       | C     | =c/postgres          +
           |          |          |         |       | postgres=CTc/postgres
 template1 | postgres | UTF8     | C       | C     | =c/postgres          +
           |          |          |         |       | postgres=CTc/postgres
(3 rows)

postgres=# \q
root@965a5c5d2e55:~# logout
```

## Configuration

### Ports

This installation exposes port `5432`.

### Data Store

For data persistence a volume should be mounted at `/var/lib/postgresql`.

SELinux users are also required to change the security context of the mount point so that it plays nicely with SELinux.

``` bash
mkdir -p /opt/postgresql/data
sudo chcon -Rt svirt_sandbox_file_t /opt/postgresql/data
```

The updated `run` command looks like this:

``` bash
docker run --name postgresql -d -v /opt/postgresql/data:/var/lib/postgresql brightcommerce/postgresql:latest
```

This will make sure that the data stored in the database is not lost when the image is stopped and started again.

### Securing The Server

By default a randomly generated password is assigned for the `postgres` user. The password is stored in a file named `pwpass` in the data store and is printed in the logs.

If you don't want this password to be displayed in the logs, then please note down the password listed in `/opt/postgresql/data/pwpass` and then delete the file.

``` bash
cat /opt/postgresql/data/pwfile
rm /opt/postgresql/data/pwfile
```

Alternately, you can change the password of the `postgres` user:

``` bash
psql -U postgres -h $(docker inspect --format {{.NetworkSettings.IPAddress}} postgresql) password postgres
```

## Shell Access

For debugging and maintenance purposes you may want access the container shell. Since the container does not allow interactive login over the SSH protocol, you can use the [nsenter](http://man7.org/linux/man-pages/man1/nsenter.1.html) linux tool (part of the util-linux package) to access the container shell.

Some linux distros (e.g. ubuntu) use older versions of the util-linux which do not include the `nsenter` tool. To get around this @jpetazzo has created a nice docker image that allows you to install the `nsenter` utility and a helper script named `docker-enter` on these distros.

To install the `nsenter` tool on your host execute the following command:

``` bash
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
```

If you are running CoreOS on the host and have difficulties using the above command because of the readonly filesystem, execute the following instructions in the `core` users' home directory:

``` bash
mkdir -p tools/bin
docker run --rm jpetazzo/nsenter cat /nsenter > tools/bin/nsenter
curl -L https://raw.githubusercontent.com/jpetazzo/nsenter/master/docker-enter > tools/bin/docker-enter
chmod +x tools/bin/*
```

Then add the toolset binaries to the system path:

``` bash
mv .bashrc .bashrc-original
sudo curl -L https://raw.githubusercontent.com/brightcommerce/coreos/master/bashrc > ~/.bashrc
sudo ln -s /root/.bashrc /root/.bash_profile
source ~/.bashrc
````

Now you can access the container shell using the `docker-enter` command:

``` bash
sudo docker-enter postgresql
```

For more information refer https://github.com/jpetazzo/nsenter.

Another tool named `nsinit` can also be used for the same purpose. Please refer https://jpetazzo.github.io/2014/03/23/lxc-attach-nsinit-nsenter-docker-0-9/ for more information.

## Upgrading

To upgrade to newer releases, simply follow this 3 step upgrade procedure.

- **Step 1**: Stop the currently running image:

``` bash
docker stop postgresql
```

- **Step 2**: Update the docker image:

``` bash
docker pull brightcommerce/postgresql:latest
```

- **Step 3**: Start the image:

``` bash
docker run --name postgresql -d [OPTIONS] brightcommerce/postgresql:latest
```

## Credit

This repository was based on the work of [docker-postgresql by Sameer Naik](https://github.com/sameersbn/docker-postgresql). Updated to install PostgreSQL v9.3.5 with contribs and postgis.
