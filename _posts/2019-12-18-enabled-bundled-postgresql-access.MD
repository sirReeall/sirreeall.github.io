---
title:  "How to use [pgAdmin](https://www.pgadmin.org/) to access the default Alfresco Postgresql Database"
categories: 
    - blog
    - alfresco
    - howto
    - postgresql
tags: 
    - alfresco
    - acs
    - postgresql
    - pgAdmin
---

In the following blog we will configure the bundled, [Alfresco Content Services](https://www.alfresco.com/ecm-software), [Postgresql](https://www.postgresql.org/) database for access via [pgAdmin](https://www.pgadmin.org/) in a lab environment. 

You should note that this is **not** a secure or hardened method. It is likely not be suitable for production environments.

## Why would you need pgAdmin?

Sometimes accessing a database via the command line is great. It's quick it's fast.

Other times it can be awkward and convoluted. Not necessarily complex, just not as feature rich as you need. Fortunately most RDBMS come with some form of GUI. This is the case with Postgresql which can be used with pgAdmin.

## Out of the Box Setup


### Remote Client and pgAdmin 

My remote client is running Windows 10 and already has pgAdmin installed. You can find pgAdmin on [pgAdmin's download page](https://www.pgadmin.org/download/). I'm using pgAdmin 4 which is really easy to install. Just download and run the installer. When you start up pgAdmin it will open up in a web browser. 

We will configure the connection from pgAdmin to the Postgresql database later in this post. For now, you just need to make sure it's installed.

### Alfresco Server

My lab is running Centos 7 without any GUI installed. This means I can **only** access Postgresql via the command line or a remote GUI.

When you install ACS 5.2.6 (or anything before ACS 6) using the [installer](https://docs.alfresco.com/5.2/tasks/simpleinstall-enterprise-lin.html), you can choose to install a bundled Postgresql server. This is pretty cool because you are up and running in a matter of minutes. 

You can see the installed database location is _acs\_install\_dir_/postgresql:

```
acs526
├── alf_data
├── amps
├── amps_share
├── bin
├── common
├── java
├── libreoffice
├── licenses
├── modules
├── postgresql
├── scripts
├── solr4
├── tmp
└── tomcat
```

This database is setup with a **user** `postgres` which will have the **password** you specify for the admin user during installation. In my case, and as this is a lab, I simply use `admin`

We will run all commands from the **acs_install_dir**, which in my case is **/opt/acs526**

On a side topic, _if_ you wanted to access this database via the command line, you can use the supplied psql binary, which is run using the following command:

```
./postgresql/bin/psql postgres postgres
```

### Vagrant, remember to forward them PORTS!

You can skip this section if you are not using vagrant.

In my case I'm using vagrant on virtualbox. I need the ports forwarded so I can remotely access the postgresql server. This is straightforward as the default port is 5432 as per Postgresql's [documentation](https://www.postgresql.org/docs/9.4/app-postgres.html). I've added this to my vagrant file using the following snippet:

```
# Postgresql
config.vm.network "forwarded_port", guest: 5432, host: 5432, auto_correct: false
```

### ~~Don't~~ Disable the firewall

As this is a lab, I have the firewall disabled. If you have a need to have the firewall enabled, make sure you allow access to the postgresql port (default is 5432) through the firewall.

## Configuring Postgresql

There are few files that need be modified to allow access to Postgresql from the outside work. These are not located in the postgresql directory. Instead, they can be found in `acs_install_dir/alf_data/postgresql`. The files we need to modify are as follows:

### postgresql.conf

By default, Postgresql will listen on localhost only. This means that any incoming connection from external interfaces will never make it to the binary. We can use `postgresql.conf` and add an external facing interface that the Postresql server can listen on.

Open up `postgresql.conf` and locate the string `#listen_addresses = 'localhost'`

Remove the `#` character and change `localhost` to the ip address on the ACS server that your want to listen on, in my lab I am not particularly concerned about security and hence I'm using '*' which means all interfaces. I've added the following:

```
listen_addresses = '*'
```

### pg_hba.conf

By default, Postgresql will only allow connections from within the server via the loopback adapter. This is fine if your server has a desktop GUI installed and you are planning on using pgAdmin "locally" on the ACS server.

In my lab, my server does not have a GUI, this means I need to configure Postgresql to be accessible from other IP address. We will use `pg_hba.conf` to allow access to Postgresql from any ip address. **Make sure this is appropriate for your security (it probably isn't in a production environment**

At the end of this file you will see the OOTB configuration section. Add the following to the end of the file:

```
host    all             all             0.0.0.0/0        md5
```

### The Restart

That's it for the Postgresql configuration. You now need to restart Postgresql for the changes to be loaded. In my lab, I'm using the bundled alfresco.sh script to restart everything using `./alfresco.sh restart`

Typically in production environments you would down ACS first, then restart the Postgresql, once up confirm that you can access Postgresql using pgAdmin, then bring up ACS again, and finally confirm the application is functioning correctly. This is obviosuly dependent on the internally policies 

## Configure pgAdmin

Start up pgAdmin, right click on Servers and select Create -> Server...

![create server](/assets/files/2019-12-18-enable-bundled-protgresql-access/pgAdmin-create-server.png)

Give the server a name, acs in my case

![server name](/assets/files/2019-12-18-enable-bundled-protgresql-access/pgAdmin-servername.png)

Switch to the `Connection` tab and provide the hostname/ ip address where the database lives. In my case, I'm using vagrant which allows me to use 127.0.0.1

Add a password if you don't like prompts :)

![connection tab](/assets/files/2019-12-18-enable-bundled-protgresql-access/pgAdmin-connection-tab.png)

Hit save and you'll find that you can see all the coolness of Postgresql and pgAdmin working together

![connection tab](/assets/files/2019-12-18-enable-bundled-protgresql-access/pgAdmin-connected.png)

