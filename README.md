Installation Documentation for BigBlueButton
============================================

This guide augments [the official BigBlueButton installation guide](https://docs.bigbluebutton.org/2.2/install.html#installation) which has a few errors.
Note that like the official guide this is for Ubuntu 16.04 64bit.
Not for anything else.

```sh
apt update
apt install -y haveged software-properties-common wget net-tools openjdk-8-jdk

export LANG=C.UTF-8; export LC_ALL=C.UTF-8

add-apt-repository ppa:bigbluebutton/support -y
add-apt-repository ppa:rmescandon/yq -y

apt update
apt dist-upgrade -y

wget -qO - https://www.mongodb.org/static/pgp/server-3.4.asc | apt-key add -
echo "deb [ arch=amd64,arm64 ] http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.4 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-3.4.list
apt update
apt install -y mongodb-org curl
curl -sL https://deb.nodesource.com/setup_8.x | bash -
apt install -y nodejs gcc g++ make

wget https://ubuntu.bigbluebutton.org/repo/bigbluebutton.asc -O- | apt-key add -
echo "deb https://ubuntu.bigbluebutton.org/xenial-220/ bigbluebutton-xenial main" | tee /etc/apt/sources.list.d/bigbluebutton.list
apt update

apt install -y bigbluebutton bbb-html5
# will throw an error touch missing fine and run again:
sudo -u etherpad touch /usr/share/etherpad-lite/APIKEY.txt
apt install -y bigbluebutton bbb-html5

# configure html5 client
sed -i 's/^attendeesJoinViaHTML5Client.*=.*$/attendeesJoinViaHTML5Client=true/' /usr/share/bbb-web/WEB-INF/classes/bigbluebutton.properties
sed -i 's/^moderatorsJoinViaHTML5Client.*=.*$/moderatorsJoinViaHTML5Client=true/' /usr/share/bbb-web/WEB-INF/classes/bigbluebutton.properties

apt install -y bbb-demo

bbb-conf --restart
bbb-conf --check
bbb-conf --status

# Remove demo to remove public room
apt remove -y bbb-demo

bbb-conf --restart
bbb-conf --check
bbb-conf --status



# TLS stuff here
# https://docs.bigbluebutton.org/2.2/install.html#configure-ssl-on-your-bigbluebutton-server

# configure domain name
bbb-conf --setip bigbluebutton.example.com

mkdir /etc/nginx/ssl
# deploy certificates
openssl dhparam -out /etc/nginx/ssl/dhp-4096.pem 4096
vim /etc/nginx/sites-available/bigbluebutton
vim /etc/bigbluebutton/nginx/sip.nginx
sed -i 's/^bigbluebutton.web.serverURL=http:/bigbluebutton.web.serverURL=https:/' /usr/share/bbb-web/WEB-INF/classes/bigbluebutton.properties
sed -i 's/^jnlpUrl=http:/jnlpUrl=https:/' /usr/share/red5/webapps/screenshare/WEB-INF/screenshare.properties
sed -i 's/^jnlpFile=http:/jnlpFile=https:/' /usr/share/red5/webapps/screenshare/WEB-INF/screenshare.properties
sed -e 's|http://|https://|g' -i /var/www/bigbluebutton/client/conf/config.xml
sed -i 's/wsUrl: ws:/wsUrl: wss:/' /usr/share/meteor/bundle/programs/server/assets/app/config/settings.yml
sed -i 's/url: http:/url: https:/' /usr/share/meteor/bundle/programs/server/assets/app/config/settings.yml
sed -i 's/playback_protocol: http/playback_protocol: https/' /usr/local/bigbluebutton/core/scripts/bigbluebutton.yml
sed -i 's_http://_https://_' /var/lib/tomcat7/webapps/demo/bbb_api_conf.jsp
bbb-conf --restart

# Firewall see https://docs.bigbluebutton.org/2.2/customize.html#restrict-access-to-specific-ports
apt install -y ufw
ufw allow OpenSSH
ufw allow "Nginx Full"
ufw allow 16384:32768/udp
ufw --force enable
```

Greenlight
----------

Greenlight is an admin frontend for BBB.
We do not need this if we use e.g. Stud.IP to create rooms.

This part basically following the guide at https://docs.bigbluebutton.org/greenlight/gl-install.html

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt update
apt install -y docker-ce docker-ce-cli containerd.io
mkdir /opt/greenlight && cd /opt/greenlight
# ... see docs
# execute docker commands

# make sure to not use docker-compose from the Ubuntu repository
# install current docker-compose
curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# make sure Greenlight starts automatically
sed -i 's/restart: .*$/restart: always/g' /opt/greenlight/docker-compose.yml

# ...

# Make sure to add at the bottom of /opt/greenlight/.env
DEFAULT_REGISTRATION=approval

# create admin user
# make sure to immediately log-in and change the password
docker exec greenlight-v2 bundle exec rake admin:create
```

Configure Clones Server
-----------------------

This isn't necessary unless you clone a complete server in which case you would need to switch to another domain name.

```bash
# switch from webconf01 to webconf04
bbb-conf --setip webconf04.example.com
sed -i 's/webconf01.e/webconf04.e/g' /etc/nginx/sites-available/bigbluebutton
systemctl reload nginx.service
apt install pwgen
bbb-conf --setsecret "$(pwgen -n1 40)"
sed -i 's/webconf01/webconf04/g' /opt/greenlight/.env
sed -i "s/^BIGBLUEBUTTON_SECRET=.*$/BIGBLUEBUTTON_SECRET=$(sudo bbb-conf --secret | grep Secret: | awk '{print $2;}')/" /opt/greenlight/.env
cd /opt/greenlight
docker-compose down
docker-compose up -d
sed -i "s/131.173.22.186/$(hostname -i | awk '{print $2}')/" /opt/freeswitch/etc/freeswitch/vars.xml
sed -i "s/131.173.22.186/$(hostname -i | awk '{print $2}')/" /usr/share/red5/webapps/sip/WEB-INF/bigbluebutton-sip.properties
sed -i "s/131.173.22.186/$(hostname -i | awk '{print $2}')/" /etc/bigbluebutton/nginx/sip.nginx
sed -i "s/131.173.22.186/$(hostname -i | awk '{print $2}')/" /usr/local/bigbluebutton/bbb-webrtc-sfu/config/default.yml
bbb-conf --restart
```

Configure sipgate
-----------------

To configure sipgate and make it possible to connect to a BBB video conference via telephone you have to
add the [my_provider.xml](https://github.com/virtUOS/bigbluebutton-install/blob/master/sipgate_configs/my_provider.xml)
to the folder

```bash
/opt/freeswitch/etc/freeswitch/dialplan/public/my_provider.xml
```

and the [sipgate.xml](https://github.com/virtUOS/bigbluebutton-install/blob/master/sipgate_configs/sipgate.xml) to the folder

```bash
/opt/freeswitch/etc/freeswitch/sip_profiles/external/sipgate.xml
```
