# Greenbone-Installation
How to install latest greenbone

# as root (!)

sudo locale-gen en_US.UTF-8 ;\\
export LC_ALL="C"


# requirements

sudo apt update ;\
sudo apt -y dist-upgrade ;\
sudo apt -y autoremove ;\
sudo apt install -y software-properties-common ;\
sudo apt install -y cmake pkg-config libglib2.0-dev libgpgme-dev libgnutls28-dev uuid-dev libssh-gcrypt-dev \
libldap2-dev doxygen graphviz libradcli-dev libhiredis-dev libpcap-dev bison libksba-dev libsnmp-dev \
gcc-mingw-w64 heimdal-dev libpopt-dev xmltoman redis-server xsltproc libical-dev postgresql \
postgresql-contrib postgresql-server-dev-all gnutls-bin nmap rpm nsis curl wget fakeroot gnupg \
sshpass socat snmp smbclient libmicrohttpd-dev libxml2-dev python-polib gettext rsync xml-twig-tools \
python3-paramiko python3-lxml python3-defusedxml python3-pip python3-psutil virtualenv vim git libunistring-dev ;\
sudo apt install -y texlive-latex-extra --no-install-recommends ;\
sudo apt install -y texlive-fonts-recommended ;\
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add - ;\
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list ;\
sudo apt update ;\
sudo apt -y install yarn

# new user

echo 'export PATH="$PATH:/opt/gvm/bin:/opt/gvm/sbin:/opt/gvm/.local/bin"' | sudo tee -a /etc/profile.d/gvm.sh ;\
sudo chmod 0755 /etc/profile.d/gvm.sh ;\
source /etc/profile.d/gvm.sh ;\
sudo bash -c 'cat << EOF > /etc/ld.so.conf.d/gvm.conf
# gmv libs location
/opt/gvm/lib
EOF'

sudo mkdir /opt/gvm ;\
sudo adduser gvm --disabled-password --home /opt/gvm/ --no-create-home --gecos '' ;\
sudo usermod -aG redis gvm  # This is for ospd-openvas can connect to redis.sock.. If you have a better idea here, pls write in the comments :) ;\
sudo chown gvm:gvm /opt/gvm/ ;\
sudo su - gvm

# as gvm (!)

mkdir src ;\
cd src ;\
export PKG_CONFIG_PATH=/opt/gvm/lib/pkgconfig:$PKG_CONFIG_PATH

# download stuff

git clone -b main --single-branch https://github.com/greenbone/gvm-libs.git ;
git clone -b main --single-branch https://github.com/greenbone/openvas-smb.git ;
git clone -b stable --single-branch https://github.com/greenbone/openvas.git ;
git clone -b stable --single-branch https://github.com/greenbone/ospd.git ;
git clone -b stable --single-branch https://github.com/greenbone/ospd-openvas.git ;
git clone -b stable --single-branch https://github.com/greenbone/gvmd.git ;
git clone -b stable --single-branch https://github.com/greenbone/gsad.git ;
git clone -b stable --single-branch https://github.com/greenbone/gsa.git

# gvm-libs

cd gvm-libs ;\
 export PKG_CONFIG_PATH=/opt/gvm/lib/pkgconfig:$PKG_CONFIG_PATH ;\
 mkdir build ;\
 cd build ;\
 cmake -DCMAKE_INSTALL_PREFIX=/opt/gvm -DLOCALSTATEDIR=/opt/gvm/var -DSYSCONFDIR=/opt/gvm/etc -DGVM_DATA_DIR=/opt/gvm/var -DGVM_RUN_DIR=/opt/gvm/var/run .. ;\
 make ;\
 make doc ;\
 make install ;\
 cd /opt/gvm/src
 
# openvas-smb
 
 cd openvas-smb ;\
 export PKG_CONFIG_PATH=/opt/gvm/lib/pkgconfig:$PKG_CONFIG_PATH ;\
 mkdir build ;\
 cd build/ ;\
 cmake -DCMAKE_INSTALL_PREFIX=/opt/gvm -DLOCALSTATEDIR=/opt/gvm/var -DSYSCONFDIR=/opt/gvm/etc -DGVM_DATA_DIR=/opt/gvm/var -DGVM_RUN_DIR=/opt/gvm/var/run .. ;\
 make ;\
 make install ;\
 cd /opt/gvm/src
 
# scanner
 
 cd openvas ;\
 export PKG_CONFIG_PATH=/opt/gvm/lib/pkgconfig:$PKG_CONFIG_PATH ;\
 mkdir build ;\
 cd build/ ;\
 cmake -DCMAKE_INSTALL_PREFIX=/opt/gvm -DLOCALSTATEDIR=/opt/gvm/var -DSYSCONFDIR=/opt/gvm/etc -DGVM_DATA_DIR=/opt/gvm/var -DGVM_RUN_DIR=/opt/gvm/var/run .. ;\
 make ;\
 make doc ;\
 make install ;\
 cd /opt/gvm/src
 
# as root (!)
 
export LC_ALL="C" ;\
ldconfig ;\
cp /etc/redis/redis.conf /etc/redis/redis.orig ;\
cp /opt/gvm/src/openvas/config/redis-openvas.conf /etc/redis/ ;\
chown redis:redis /etc/redis/redis-openvas.conf ;\
echo "db_address = /run/redis-openvas/redis.sock" > /opt/gvm/etc/openvas/openvas.conf ;\
systemctl enable redis-server@openvas.service ;\
systemctl start redis-server@openvas.service


sysctl -w net.core.somaxconn=1024
sysctl vm.overcommit_memory=1
echo "net.core.somaxconn=1024"  >> /etc/sysctl.conf
echo "vm.overcommit_memory=1" >> /etc/sysctl.conf

cat << EOF > /etc/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages (THP)

[Service]
Type=simple
ExecStart=/bin/sh -c "echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled && echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag"

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload ;\
systemctl start disable-thp ;\
systemctl enable disable-thp ;\
systemctl restart redis-server

visudo

Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/gvm/sbin"
### Allow the user running ospd-openvas, to launch openvas with root permissions
gvm ALL = NOPASSWD: /opt/gvm/sbin/openvas

# as gvm (!)

greenbone-nvt-sync
openvas -u

# manager

cd gvmd ;\
 export PKG_CONFIG_PATH=/opt/gvm/lib/pkgconfig:$PKG_CONFIG_PATH ;\
 mkdir build ;\
 cd build/ ;\
 cmake -DCMAKE_INSTALL_PREFIX=/opt/gvm -DLOCALSTATEDIR=/opt/gvm/var -DSYSCONFDIR=/opt/gvm/etc -DGVM_DATA_DIR=/opt/gvm/var -DGVM_RUN_DIR=/opt/gvm/var/run .. ;\
 make ;\
 make doc ;\
 make install ;\
 cd /opt/gvm/src
 
# as postgres (!)

sudo -u postgres bash
export LC_ALL="C"
createuser -DRS gvm
createdb -O gvm gvmd

psql gvmd
create role dba with superuser noinherit;
grant dba to gvm;
create extension "uuid-ossp";
create extension "pgcrypto";
exit
exit

# as gvm (!)

# certificats

gvm-manage-certs -a

# new user , Super Admin if you wish

gvmd --create-user=admin --password=admin ( --role="Super Admin" )

gvm@gvm2008-lab:/opt/gvm/src$ gvmd --get-users --verbose
admin 41f853e4-fecf-423f-85b7-18fa3396bac5 ««« This uuid

gvmd --modify-setting 78eceaec-3385-11ea-b237-28d24461215b --value 41f853e4-fecf-423f-85b7-18fa3396bac5

# feeds

greenbone-feed-sync --type GVMD_DATA
greenbone-feed-sync --type SCAP
greenbone-feed-sync --type CERT

# gsad

cd gsad ;\
 export PKG_CONFIG_PATH=/opt/gvm/lib/pkgconfig:$PKG_CONFIG_PATH ;\
 mkdir build ;\
 cd build/ ;\
 cmake -DCMAKE_INSTALL_PREFIX=/opt/gvm -DLOCALSTATEDIR=/opt/gvm/var -DSYSCONFDIR=/opt/gvm/etc -DGVM_DATA_DIR=/opt/gvm/var -DGVM_RUN_DIR=/opt/gvm/var/run .. ;\
 make ;\
 make doc ;\
 make install ;\
 touch /opt/gvm/var/log/gvm/gsad.log ;\
 cd /opt/gvm/src
 
# gsa

cd gsa ;\
 export PKG_CONFIG_PATH=/opt/gvm/lib/pkgconfig:$PKG_CONFIG_PATH ;\
 yarn
 yarn build
 cp -r build/* /opt/gvm/share/gvm/gsad/web/ ### probably as root
 cd /opt/gvm/src

# ospd-openvas

cd /opt/gvm/src ;\
export PKG_CONFIG_PATH=/opt/gvm/lib/pkgconfig:$PKG_CONFIG_PATH ;\
virtualenv --python python3.9  /opt/gvm/bin/ospd-scanner/ ;\   ### check current version of python
source /opt/gvm/bin/ospd-scanner/bin/activate

mkdir /opt/gvm/var/run/ospd/ ;\
cd ospd ;\
pip3 install . ;\  ### probably as root
cd /opt/gvm/src

cd ospd-openvas ;\
pip3 install . ;\ ### probably as root
cd /opt/gvm/src

# as root (!)

cat << EOF > /etc/systemd/system/gvmd.service
[Unit]
Description=Open Vulnerability Assessment System Manager Daemon
Documentation=man:gvmd(8) https://www.greenbone.net
Wants=postgresql.service ospd-openvas.service
After=postgresql.service ospd-openvas.service

[Service]
Type=forking
User=gvm
Group=gvm
PIDFile=/opt/gvm/var/run/gvmd.pid
WorkingDirectory=/opt/gvm
ExecStart=/opt/gvm/sbin/gvmd --osp-vt-update=/opt/gvm/var/run/ospd.sock
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
Restart=on-failure
RestartSec=2min
KillMode=process
KillSignal=SIGINT
GuessMainPID=no
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF


cat << EOF > /etc/systemd/system/gsad.service
[Unit]
Description=Greenbone Security Assistant (gsad)
Documentation=man:gsad(8) https://www.greenbone.net
After=network.target
Wants=gvmd.service

[Service]
Type=forking
PIDFile=/opt/gvm/var/run/gsad.pid
WorkingDirectory=/opt/gvm
ExecStart=/opt/gvm/sbin/gsad --drop-privileges=gvm
Restart=on-failure
RestartSec=2min
KillMode=process
KillSignal=SIGINT
GuessMainPID=no
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF


cat << EOF > /etc/systemd/system/ospd-openvas.service 
[Unit]
Description=Job that runs the ospd-openvas daemon
Documentation=man:gvm
After=network.target redis-server@openvas.service
Wants=redis-server@openvas.service

[Service]
Environment=PATH=/opt/gvm/bin/ospd-scanner/bin:/opt/gvm/bin:/opt/gvm/sbin:/opt/gvm/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Type=forking
User=gvm
Group=gvm
WorkingDirectory=/opt/gvm
PIDFile=/opt/gvm/var/run/ospd-openvas.pid
ExecStart=/opt/gvm/bin/ospd-scanner/bin/python /opt/gvm/bin/ospd-scanner/bin/ospd-openvas --pid-file /opt/gvm/var/run/ospd-openvas.pid --unix-socket=/opt/gvm/var/run/ospd.sock --log-file /opt/gvm/var/log/gvm/ospd-scanner.log --lock-file-dir /opt/gvm/var/run/
Restart=on-failure
RestartSec=2min
KillMode=process
KillSignal=SIGINT
GuessMainPID=no
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF


systemctl daemon-reload ;\
systemctl enable gvmd ;\
systemctl enable gsad ;\
systemctl enable ospd-openvas

# sic !

cat << EOF > /etc/tmpfiles.d/gvm.conf
d /run/gvm 1755 gvm gvm
L /run/gvmd - - - - /opt/gvm/var/run
L /run/gsad - - - - /opt/gvm/var/run
EOF

# start 

systemctl start ospd-openvas ;\
systemctl start gvmd ;\
systemctl start gsad

# check

systemctl status gvmd
systemctl status gsad
systemctl status ospd-openvas

# as gvm (!)

# modify  scanner

gvmd --get-scanners
08b69003-5fc2-4037-a479-93b440211c73  OpenVAS  /var/run/ospd/ospd.sock  0  OpenVAS Default «««««««««« THIS UUID
6acd0832-df90-11e4-b9d5-28d24461215b  CVE    0  CVE

gvmd --modify-scanner=08b69003-5fc2-4037-a479-93b440211c73 --scanner-host=/opt/gvm/var/run/ospd.sock
Scanner modified.

# goto https://your_ip/


# update feeds

nano /etc/crontab

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

0 1 * * * gvm	greenbone-nvt-sync
0 2 * * * gvm	greenbone-feed-sync --type GVMD_DATA
0 3 * * * gvm	greenbone-feed-sync --type SCAP
0 4 * * * gvm	greenbone-feed-sync --type CERT


