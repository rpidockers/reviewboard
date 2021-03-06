docker-reviewboard
==================

Dockerized reviewboard. This container follows Docker's best practices, and DOES NOT include sshd, supervisor, apache2, or any other services except the reviewboard itself which is run with ```uwsgi```.

The requirements are PostgreSQL and memcached, you can use either dockersized versions of them, or external ones, e.g. installed on the host machine, or even third-party machines.

## Quickstart. Run dockerized reviewboard with all dockerized dependencies.

    # Install postgres
    docker run -d --name rb-postgres postgres
    docker run -it --link rb-postgres:postgres --rm postgres sh -c 'exec createuser reviewboard -h "$POSTGRES_PORT_5432_TCP_ADDR" -p "$POSTGRES_PORT_5432_TCP_PORT" -U postgres'
    docker run -it --link rb-postgres:postgres --rm postgres sh -c 'exec createdb reviewboard -O reviewboard -h "$POSTGRES_PORT_5432_TCP_ADDR" -p "$POSTGRES_PORT_5432_TCP_PORT" -U postgres'

    # Install memcached
    docker run --name rb-memcached -d -p 11211 sylvainlasnier/memcached

    # Run reviewboard
    docker run -it --link rb-postgres:pg --link rb-memcached:memcached -v <ssh-dir>:/.ssh -v <media-dir>:/media -p 8000:8000 ikatson/reviewboard

After that, go the url, e.g. ```http://localhost:8000/```, login as ```admin:admin```, change the admin password, and change the location of your SMTP server so that the reviewboard can send emails. You are all set!

For details, read below.

## Build yourself if you want.

If you want to build this yourself, just run this:

    docker build -t 'ikatson/reviewboard' git://github.com/ikatson/docker-reviewboard.git

## Dependencies

### Install PostgreSQL

You can install postgres either into a docker container, or whereever else.

1. Example: install postgres into a docker container, and create a database for reviewboard.

        docker run -d --name some-postgres postgres

        # Create the database and user for reviewboard
        docker run -it --link some-postgres:postgres --rm postgres sh -c 'exec createuser reviewboard -h "$POSTGRES_PORT_5432_TCP_ADDR" -p "$POSTGRES_PORT_5432_TCP_PORT" -U postgres'
        docker run -it --link some-postgres:postgres --rm postgres sh -c 'exec createdb reviewboard -O reviewboard -h "$POSTGRES_PORT_5432_TCP_ADDR" -p "$POSTGRES_PORT_5432_TCP_PORT" -U postgres'

2. Example: install postgres into the host machine

        apt-get install postgresql-server

        # Uncomment this to make postgres listen
        # echo "listen_addresses = '*'" >> /etc/postgresql/VERSION/postgresql.conf
        # invoke-rc.d postgresql restart
        sudo -u postgres createuser reviewboard
        sudo -u postgres createdb reviewboard -O reviewboard
        sudo -u postgres psql -c "alter user reviewboard set password to 'SOME_PASSWORD'"

### Install memcached

1. Example: install into a docker container

        docker run --name memcached -d -p 11211 sylvainlasnier/memcached

1. Example: install locally on Debian/Ubuntu.

        apt-get install memcached

   Don't forget to make it listen on needed addresses by editing /etc/memcached.conf, but be careful not to open memcached for the whole world.

## Run reviewboard

This container has two volume mount-points:

- ```/.ssh``` - The default path to where reviewboard stores it's ssh keys.
- ```/media``` - The default path to where reviewboard stores uploaded media.

The container accepts the following environment variables:

- ```PGHOST``` - the postgres host. Defaults to the value of ```PG_PORT_5432_TCP_ADDR```, provided by the ```pg``` linked container.
- ```PGPORT``` - the postgres port. Defaults to the value of ```PG_PORT_5432_TCP_PORT```, provided by the ```pg``` linked containe, or 5432, if it's empty.
- ```PGUSER``` - the postgres user. Defaults to ```reviewboard```.
- ```PGDB``` - the postgres database. Defaults to ```reviewboard```.
- ```PGPASSWORD``` - the postgres password. Defaults to ```reviewboard```.
- ```MEMCACHED``` - memcache address in format ```host:port```. Defaults to the value from linked ```memcached``` container.
- ```DOMAIN``` - defaults to ```localhost```.
- ```DEBUG``` - if set, the django server will be launched in debug mode.

Also, uwsgi accepts a any environment variables for it's configuration
E.g. ```-e UWSGI_PROCESSES=10``` will create 10 reviewboard processes.

### Example. Run with dockerized postgres and memcached from above, expose on port 8000:

    docker run -it --link some-postgres:pg --link memcached:memcached -v <ssh-dir>:/.ssh -v <media-dir>:/media  -p 8000:8000 ikatson/reviewboard

### Example. Run with postgres and memcached installed on the host machine.

    DOCKER_HOST_IP=$( ip addr | grep 'inet 172.1' | awk '{print $2}' | sed 's/\/.*//')

    docker run -it -p 8000:8080 -v <ssh-dir>:/.ssh -v <media-dir>:/media -e PGHOST="$DOCKER_HOST_IP" -e PGPASSWORD=123 -e PGUSER=reviewboard -e MEMCACHED="$DOCKER_HOST_IP":11211 ikatson/reviewboard

Now, go to the url, e.g. ```http://localhost:8000/```, login as ```admin:admin``` and change the password. The reviewboard is almost ready to use!

### Container SMTP settings.

You should also change SMTP settings, so that the reviewboard can send emails. A good way to go is to set this to docker host's internal IP address, usually, ```172.17.42.1```).

Don't forget to setup you mail agent to accept emails from docker.

For example, if you use ```postfix```, you should change ```/etc/postfix/main.cf``` to contain something like the lines below:

    mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 172.17.0.0/16
    inet_interfaces = 127.0.0.1,172.17.42.1
