---
layout: post
title: Migrating TeamCity to docker with data containers
date: '2016-07-22T17:45:00+02:00'
tags: teamcity, docker
---
Using separate data container gives you much more flexibility in how you upgrade, test, or backup the services. You then have a pure service that you can kill restart throw away and upgrade without worrying about the configuration. The configiuration will be provided by the long-live data container instead.
you dont need to backup the service. only the data container associated to it, etc...

We'll have 4 pieces :  

 1. Teamcity server  
 2. Teamcity data  
 3. Postgres instance  
 4. Postgres data  

## Let's start with the data containers

We'll use an empty image to create empty data containers : `tianon/true`
(If you want to know more, I prefer [this reasoning, using an empty image](http://jdemarks.azurewebsites.net/2015/04/1083/) to [this one that uses the same base image as the app](http://container42.com/2014/11/18/data-only-container-madness/), but maybe I'm wrong, let me know :))  

    docker create -v /teamcity --name teamcity-data tianon/true echo 'teamcity data'
    docker create -v /postgres --name postgres-data tianon/true echo 'postgres data'

That gives you a container with nothing but 1 folder named `/teamcity` or `/postgres`.
you can take a peak at it by doing the following :

    C:\> docker run -it --rm --volumes-from teamcity-data ubuntu ls
    bin   core  etc   lib    media  opt   root  sbin  sys       tmp  var
    boot  dev   home  lib64  mnt    proc  run   srv   ``teamcity``  usr

We map the volumes from our newly created teamcity-data container into a new ubuntu container and we run ls. we can see the `teamcity` folder, mounted from our teamcity-data container.  

## Postgres

Postgres is quite straight forward. we'll start the postgres conainter, and take the volume from our postgres-data container :

    docker run --volumes-from postgres-data \
      --name tc-postgres \
      -e POSTGRES_PASSWORD=<password_here> \
      -e PGDATA=/postgres/pgdata postgres

postgres image uses some environment variables during setup to define the password and the data folder for our db. the user is postgres by default.

that's about it. Now we get :

    C:\> docker ps
    CONTAINER ID     IMAGE       COMMAND                  CREATED       STATUS       PORTS       NAMES
    7f43642f2c2e     postgres    "/docker-entrypoint.s"   2 hours ago   Up 2 hours   5432/tcp    tc-postgres


## Teamcity server

### Restoring backup

I took a backup of the existing teamcity config folders, from the teamcity UI, and unzipped it on my local machine :

    C:\teamcityupgrade> ls

        Directory: C:\teamcityupgrade

    Mode                LastWriteTime         Length Name
    ----                -------------         ------ ----
    d-----       2016-07-22  01:05 PM                teamcity
    -a----       2016-07-22  01:01 PM      124890616 TeamCity_Backup.zip

    C:\teamcityupgrade> ls .\teamcity\

        Directory: C:\teamcityupgrade\teamcity

    Mode                LastWriteTime         Length Name
    ----                -------------         ------ ----
    d-----       2016-07-22  01:01 PM                config
    d-----       2016-07-22  01:01 PM                lib
    d-----       2016-07-22  01:01 PM                metadata
    d-----       2016-07-22  01:01 PM                plugins
    d-----       2016-07-22  01:01 PM                system
    -a----       2016-07-22  10:56 AM              6 charset
    -a----       2016-07-22  10:56 AM            681 export.report
    -a----       2016-07-22  10:56 AM             87 version.txt

We'll want to restore the database, and you can do that with the maintainDB.sh script from teamcity. that's in the TC container. Let's see what we need :

 1. Use the volumes from our teamcity-data container
 2. Tell teamcity to use our volume folder (env variable)
 3. link it to postgres
 4. restore backup
 6. run teamcity

We'll start with the db restore. We will bash into a teamcity container to get access to the maintainDB.sh script. we call the container `restore-tc`.  
The script needs the `database.properties` file, an empty config folder, backup zip file and a connection to postgres :

    docker run -it --name restore-tc --rm \
      --link tc-postgres \ # gives us a connection to postgres
      --volumes-from teamcity-data \ # gives us the config folder
      jetbrains/teamcity-server /bin/bash

Then we will copy the backup and the `database.properties` and the driver libraries into the container. We'll use [docker cp](https://docs.docker.com/engine/reference/commandline/cp/) to do that :

So from another console, we run :

     docker cp C:\teamcityupgrade\TeamCity_Backup.zip restore-tc:/backup.zip

We edit the database.properties to connect to our postgres, and copy it too :

    connectionProperties.user=postgres
    connectionProperties.password=<password_here>
    connectionUrl=jdbc\:postgresql\://tc-postgres\:5432/  

Note the url uses the postgres container name. this works thanks to the link parameter in the previous command.
Then we copy it over :

    docker cp C:\teamcityupgrade\teamcity\config\database.properties restore-tc:/restore-database.properties

And finally the jdbc drivers for postgres:

     docker cp C:\teamcityupgrade\teamcity\lib\jdbc\ restore-tc:/teamcity/jdbc/

Now, to restore the database, let's go back to the bash console inside our `restore-tc` container and run the `maintainDB.sh` script ([documentation](https://confluence.jetbrains.com/display/TCD9/Restoring+TeamCity+Data+from+Backup#RestoringTeamCityDatafromBackup-Performingfullrestore)):

    :/# /opt/teamcity/bin/maintainDB.sh restore \
        -A /teamcity/ \
        -F /backup.zip \
        -T /restore-database.properties

Voilà, backup is restored!

### Running Teamcity

We could keep that container and run it from there, but we can also kill it and start a new one with about the same parameter, and without the /bin/bash.
all the config data is in the `teamcity-data` data container now.
so we exit the `restore-tc` container (we started it with --rm so it will be stopped and removed).

What we want :  

 1. data from the teamcity-data (where we restored the config)
 2. tell teamcity to use the /teamcity folder as data folder
 2. use db from postgres-tc (we also just restored it)
 3. port exposed
 4. run as daemon

To run it, the full command will be :  

    docker run -d --name teamcity \
       -p80:8111 \
       -e TEAMCITY_DATA_PATH=/teamcity \
       --link tc-postgres \
       --volumes-from teamcity-data \
       jetbrains/teamcity-server

Aaand... it works (I mapped it to port 8111 in my test) :)

![it works!](/blog/assets/article_images/2016-07-22-migrating-teamcity/tc-works.png)
