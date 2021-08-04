# How to Install Jitsi Meet with Multi Server Configuration
This tutorial is for jitsi-meet installation using 2 server or more. The main server will contain jitsi-meet react source code, prosody, nginx, and jicofo. The videobridge will be installed seperatelly on the second server and so on.

## Prerequisite
1. Minimum 2 server with 1 IP Public each
2. Ubuntu 18.04

## Sudo Privileges
Before start we make sure that we will have no permission issue on the installation.
````sh
sudo su
````

## Main Server
On this server we will run the nginx, prosody, and jicofo.

### Update apt repo
````sh
apt-get update
````

### Install NGINX
````sh
apt-get install nginx
````
note that we need to install nginx before installing jitsi-meet so the jitsi-meet not using jetty as the web server

### Install Jitsi-Meet with latest prosody
````sh
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -
sudo sh -c "echo 'deb https://download.jitsi.org stable/' > /etc/apt/sources.list.d/jitsi-stable.list"
echo deb http://packages.prosody.im/debian $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list
wget https://prosody.im/files/prosody-debian-packages.key -O- | sudo apt-key add -
sudo apt-get update
sudo apt-get install --no-install-recommends jitsi-meet lua-sec
````
you will be asked the domain, and ssl option when installing jitsi-meet. For me, I prefer use let's encrypt and then change it later.

### Cleaning Not Used Files & Service

remove default site config files on nginx
````sh
rm /etc/nginx/site-enabled/default
rm /etc/nginx/site-available/default
````

stop jitsi videbridge as it will be running on the second server, not in the main server
````sh
/etc/init.d/jitsi-videobridge stop
````

### Install Nodejs
````sh
curl -sL https://deb.nodesource.com/setup_10.x -o nodesource_setup.sh
bash nodesource_setup.sh
apt install nodejs
````

### Install make
````sh
apt-get install make
````

### Prosody Global Config
open global prosody configuration file ``/etc/prosody/prosody.cfg.lua`` and add this statement:
````su
component_interface = "0.0.0.0"
compinent_ports = { 5347 }
network_backend = "epoll"
````
and if you still using the old configuration file, you will find
````su
daemonize = true;
````
if you find this, change it to ``false``, if you dont find it, just leave it

### Prosody Virtual Host Config
Open your config ``/etc/prosody/conf.d/<your.domain.com>.cfg.lua`` and adjust your file like this
````sh
consider_bosh_secure = true;

VirtualHost "<your.domain.com>"
        -- enabled = false -- Remove this line to enable this host
        authentication = "anonymous"
        -- Properties below are modified by jitsi-meet-tokens package config
        -- and authentication above is switched to "token"
        --app_id="example_app_id"
        --app_secret="example_app_secret"
        -- Assign this host a certificate for TLS, otherwise it would use the one
        -- set in the global section (if any).
        -- Note that old-style SSL on port 5223 only supports one certificate, and will always
        -- use the global one.
        ssl = {
                key = "/etc/prosody/certs/<your.domain.com>.key";
                certificate = "/etc/prosody/certs/<your.domain.com>.crt";
        }
        -- we need bosh
        modules_enabled = {
            "bosh";
            "pubsub";
            "ping"; -- Enable mod_ping
        }
        c2s_require_encryption = false

Component "conference.<your.domain.com>" "muc"
    storage = "memory"
    admins = { "focus@auth.<your.domain.com>" }

-- internal muc component
Component "internal.auth.<your.domain.com>" "muc"
    storage = "memory"
    modules_enabled = {
      "ping";
    }
    muc_room_cache_size = 1000
    admins = { "focus@auth.<your.domain.com>", "jvb@auth.<your.domain.com>" }

VirtualHost "auth.<your.domain.com>"
    ssl = {
        key = "/etc/prosody/certs/auth.<your.domain.com>.key";
        certificate = "/etc/prosody/certs/auth.<your.domain.com>.crt";
    }
    authentication = "internal_plain"

Component "focus.<your.domain.com>"
    component_secret = "<jicofo focus secret>"

````
You can comment or delete other line if there are any different with your file.

### Restart Prosody
````sh
/etc/init.d/prosody restart
````

### Look at JVB Password in Prosody
You can check the password by execute this command
````sh
cat /var/lib/prosody/auth%2e<your%2edomain%2ecom>/accounts/jvb.dat
````
This password will be used by jvb on the second server to connect to prosody

## Second Server
On this server we will run the jitsi videobridge2.

### Install Jitsi-Videobridge
````sh
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -
sudo sh -c "echo 'deb https://download.jitsi.org stable/' > /etc/apt/sources.list.d/jitsi-stable.list"
sudo apt-get update
sudo apt-get install jitsi-videobridge2
````
you will be asked the domain, input the domain of the main server. This JVB server doesn't need a domain.

### Config file
open ``/etc/jitsi/videobridge/config`` make sure you the config is already right:
````sh
# Jitsi Videobridge settings

# sets the XMPP domain (default: none)
JVB_HOSTNAME= --> leave this blank

# sets the hostname of the XMPP server (default: domain if set, localhost otherwise)
JVB_HOST= --> leave this blank

# sets the port of the XMPP server (default: 5275)
JVB_PORT=5347 --> adjust this value to prosody listen port 

# sets the shared secret used to authenticate to the XMPP server
JVB_SECRET=6iHFEg3U --> it doesn't matter anymore

# extra options to pass to the JVB daemon
JVB_OPTS="--apis=rest,"

# adds java system props that are passed to jvb (default are for home and logging config file)
JAVA_SYS_PROPS="-Dnet.java.sip.communicator.SC_HOME_DIR_LOCATION=/etc/jitsi -Dnet.java.sip.communicator.SC_HOME_DIR_NAME=videobridge -Dnet.java.sip.communicator.SC_LOG$
````
### sip-communicator.properties file
open ``/etc/jitsi/videobridge/sip-communicator.properties`` and then add these statement
````sh
org.jitsi.videobridge.ENABLE_STATISTICS=true
org.jitsi.videobridge.STATISTICS_TRANSPORT=muc,colibri
org.jitsi.videobridge.xmpp.user.shard.HOSTNAME=<your.domain.com>
org.jitsi.videobridge.xmpp.user.shard.DOMAIN=auth.<your.domain.com>
org.jitsi.videobridge.xmpp.user.shard.USERNAME=jvb
org.jitsi.videobridge.xmpp.user.shard.PASSWORD=<your jvb password> --> from jvb.dat
org.jitsi.videobridge.xmpp.user.shard.MUC_JIDS=JvbBrewery@internal.auth.<your.domain.com>
org.jitsi.videobridge.xmpp.user.shard.MUC_NICKNAME=<unique name for the jvb>
org.jitsi.videobridge.xmpp.user.shard.DISABLE_CERTIFICATE_VERIFICATION=true
````
If your JVB server is behind NAT, also add this 2 statement on this file
````sh
org.ice4j.ice.harvest.NAT_HARVESTER_LOCAL_ADDRESS=<Local.IP.Address>
org.ice4j.ice.harvest.NAT_HARVESTER_PUBLIC_ADDRESS=<Public.IP.Address>
````

## Restart All Services

on main server
````sh
/etc/init.d/prosody restart
/etc/init.d/jicofo restart
````
on jvb server
````sh
/etc/init.d/jitsi-videobridge2 restart
````

## Checking
To make sure all setup is success, you can check jicofo log
````sh
grep 'Added new videobridge' /var/log/jitsi/jicofo.log
````
You should see your videobridge nickname there.

## Extra
To add more Videobridge, just install the jitsi-videobridge on the new server and follow the **Second Server** installation.
