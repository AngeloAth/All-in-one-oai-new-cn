# Installation Manual for OAI all in one
## Hardware Setup

Intel® Core™ i7-9700 @ 3.00GHz

32 GB RAM

USRP B210

LTE Band7 Duplexer

Open Cells Card Reader

Open Cells Sim Cards


## Software Setup

### OS Installation
Install Ubuntu [18.04](https://releases.ubuntu.com/18.04/ubuntu-18.04.5-desktop-amd64.iso) on your computer

### Clone the core network's repositories

```sh
git clone https://github.com/OPENAIRINTERFACE/openair-cn.git
git clone https://github.com/OPENAIRINTERFACE/openair-cn-cups.git
```

### Switch to develop branch in both directories that have been created

```sh
cd ~/openair-cn
git branch
git checkout develop

cd ~/openair-cn-cups
git branch
git checkout develop
```
### Install USRP B210 Drivers (Open Cells Tutorial)

```sh
sudo apt-get install libboost-all-dev libusb-1.0-0-dev python-mako doxygen python-docutils python-requests python3-pip cmake build-essential
sudo pip3 install mako numpy
git clone git://github.com/EttusResearch/uhd.git
cd uhd; mkdir host/build; cd host/build
cmake -DCMAKE_INSTALL_PREFIX=/usr ..
make -j4
sudo make install
sudo ldconfig
sudo /usr/lib/uhd/utils/uhd_images_downloader.py
```

### HSS - Installation - Configuration

- Before installing Cassandra 

Add the public key and install OpenJRE version 8 

```sh
echo "deb http://www.apache.org/dist/cassandra/debian 21x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list
curl https://downloads.apache.org/cassandra/KEYS | sudo apt-key add -
sudo apt install openjdk-8-jdk openjdk-8-jre
```

Change broken link of apache keys

```sh
cd openair-cn/build/tools
sudo gedit ~/build_helper.cassandra
```
Change link in line 64 to https://downloads.apache.org/cassandra/KEYS

- Installing Cassandra

```sh
cd ~/openair-cn/scripts/
./build_cassandra --check-installed-software --force
```

- Configuring Cassandra

Set the JRE version 8 as default one

```sh
sudo service cassandra stop
update-java-alternatives -l
sudo update-alternatives --config java
```
Type selection number that equals java-8-openjdk exp. here is 2 then press Enter
 
![alt text](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/screenshots/1.cass-java-jre8.png "jdk-8")

Verify that Cassandra is properly installed and running

```sh
sudo service cassandra start
nodetool status
```

Stop Cassandra and cleanup the log files before modifying the configuration

```sh
sudo service cassandra stop
sudo rm -rf /var/lib/cassandra/data/system/*
sudo rm -rf /var/lib/cassandra/commitlog/*
sudo rm -rf /var/lib/cassandra/data/system_traces/*
sudo rm -rf /var/lib/cassandra/saved_caches/*
```

Update the Cassandra configuration 

```sh
sudo gedit /etc/cassandra/cassandra.yaml
```
Edit the following lines
 
```sh
- LINE 272 comment this line
# seeds is actually a comma-delimited list of addresses.
- LINE 273
seeds: "127.0.0.1"
- LINE 705
endpoint_snitch: GossipingPropertyFileSnitch
```
Verify that you can access Cassandra DB

```sh
cqlsh
exit
```
Create cassandra db keyspaces

```sh
cqlsh --file /home/oai/openair-cn/src/hss_rel14/db/oai_db.cql 127.0.0.1 
```
Populate users table
Modify the below command to suit your setup.
Exp. If your hostname is abcd and realm is ng4T.com modify --mme-identity abcd.ng4T.com , in my case the hostname is "oai" and realm is "ng4T.com"

```sh
hostname --fqdn
```
Then
```sh
~/openair-cn/scripts/data_provisioning_users --apn default.ng4T.com --apn2 internet --key fec86ba6eb707ed08905757b1bb44b8f --imsi-first 208930000000002 --msisdn-first 001011234561000 --mme-identity oai.ng4T.com --no-of-users 10 --realm ng4T.com --truncate True  --verbose True --cassandra-cluster 127.0.0.1
```

Add the MME info in the database
Again here use your hostname and realm
Exp. --mme-identity abcd.ng4T.com

```sh
~/openair-cn/scripts/data_provisioning_mme --id 3 --mme-identity oai.ng4T.com --realm ng4T.com --ue-reachability 1 --truncate True  --verbose True -C 127.0.0.1
```

- HSS Configuration (1/4) : **Prework**

Create the folders

```sh
sudo mkdir -p /usr/local/etc/oai
sudo chmod 777 /usr/local/etc/oai
sudo mkdir  /usr/local/etc/oai/freeDiameter
sudo chmod 777 /usr/local/etc/oai/freeDiameter
sudo mkdir -p /usr/local/etc/oai/conf
sudo chmod 777 /usr/local/etc/oai/conf
sudo mkdir -p /usr/local/etc/oai/logs
sudo chmod 777 /usr/local/etc/oai/logs
```

Create the freeDiameter files

```sh
cd ~/openair-cn/scripts

sudo cp ../etc/acl.conf ../etc/hss_rel14_fd.conf /usr/local/etc/oai/freeDiameter
sudo chmod 777 /usr/local/etc/oai/freeDiameter/hss_rel14.conf /usr/local/etc/oai/freeDiameter/hss_rel14.json

sudo cp ../etc/hss_rel14.conf ../etc/hss_rel14.json /usr/local/etc/oai
sudo chmod 777 /usr/local/etc/oai/hss_rel14.conf /usr/local/etc/oai/hss_rel14.json

sudo cp ../etc/oss.json /usr/local/etc/oai/conf
sudo chmod 777 /usr/local/etc/oai/conf/oss.json
```

Create freeDiameter log folders and files

```sh
touch /usr/local/etc/oai/logs/hss.log
touch /usr/local/etc/oai/logs/hss_stat.log
touch /usr/local/etc/oai/logs/hss_audit.log
sudo chmod 777 /usr/local/etc/oai/logs/hss.log /usr/local/etc/oai/logs/hss_stat.log /usr/local/etc/oai/logs/hss_audit.log
```

Generate the certificates

```sh
../src/hss_rel14/bin/make_certs.sh hss ng4T.com /usr/local/etc/oai
oai_hss -j /usr/local/etc/oai/hss_rel14.json --onlyloadkey
```
- HSS Configuration (2/4) : File **acl.conf**

Change @REALM@ with your realm 
Exp. If your realm is "ng4T.com"

`ALLOW_OLD_TLS  *.ng4T.com *.localdomain *.test3gpp.net`

```sh
sudo gedit /usr/local/etc/oai/freeDiameter/acl.conf
```
- HSS Configuration (3/4) : File **hss_fd.conf**

Change **Line 7** to match your hss and realm : `Identity = "hss.ng4T.com"`
Change **Line 7** to match your hss and realm : `Identity = "hss.ng4T.com"`









### MME - Installation - Configuration

- Install MME

```markdown
cd ~/openair-cn/scripts
./build_mme --check-installed-software --force
./build_mme --clean
```

- Create MME configuration files

```markdown
sudo openssl rand -out ~/.rnd 128
cd ~/openair-cn/scripts
cp ../etc/mme_fd.sprint.conf  /usr/local/etc/oai/freeDiameter/mme_fd.conf
cp ../etc/mme.conf  /usr/local/etc/oai
sudo chmod 777 /usr/local/etc/oai/freeDiameter/mme_fd.conf /usr/local/etc/oai/mme.conf
```

- Configuration of MME

1. Change the following lines in the file /usr/local/etc/oai/mme.conf

```markdown
REALM = "ng4T.com";
INSTANCE = 1; 
PID_DIRECTORY = "/var/run";  
S6A_CONF = "/usr/local/etc/oai/freeDiameter/mme_fd.conf";
HSS_HOSTNAME = "hss"; 
{MCC="208" ; MNC="93"; MME_GID="32768" ; MME_CODE="3"; } 
{MCC="208" ; MNC="93";  TAC = "1"; }
{ID="tac-lb01.tac-hb00.tac.epc.mnc093.mcc208.3gppnetwork.org" ; SGW_IPV4_ADDRESS_FOR_S11="127.0.11.2/24";},
{ID="tac-lb01.tac-hb00.tac.epc.mnc093.mcc208.3gppnetwork.org" ; PEER_MME_IPV4_ADDRESS_FOR_S10="0.0.0.0/24";}
```

2. Change the following lines in the file /usr/local/etc/oai/freeDiameter/mme_fd.conf  %%

```markdown
Identity = "oai.ng4T.com";
Realm = "ng4T.com";
TLS_Cred = "/usr/local/etc/oai/freeDiameter/mme.cert.pem",
           "/usr/local/etc/oai/freeDiameter/mme.key.pem";
TLS_CA   = "/usr/local/etc/oai/freeDiameter/mme.cacert.pem";
ListenOn = "127.0.0.11";
ConnectPeer= "hss.ng4T.com" { ConnectTo = "127.0.0.1"; No_SCTP ; No_IPv6; Prefer_TCP; No_TLS; port = 3868;  realm = "ng4T.com";};
```

### Check the certificates for both HSS and MME

```markdown
cd ~/openair-cn/scripts
./check_hss_s6a_certificate /usr/local/etc/oai/freeDiameter/ hss.ng4T.com
./check_mme_s6a_certificate /usr/local/etc/oai/freeDiameter/ oai.ng4T.com
```

### Configure hosts

```markdown
hostname --fqdn
sudo gedit /etc/hosts
```

Modify to suit your hostname fqdn
Exp. If your hostname fqdn is ABCD

127.0.0.1	localhost
127.0.1.1 ABCD.ng4T.com ABCD
127.0.33.1 hss.ng4T.com hss

### SPGW-C - Installation - Configuration

Install SPGW-C

```markdown
cd ~/openair-cn-cups/build/scripts
sudo ./build_spgwc -I -f
sudo ./build_spgwc -c -V -b Debug -j
```

Create configuration files of SPGW-C

```markdown
cp ../../etc/spgw_c.conf  /usr/local/etc/oai
```

Configure SPGW-C

- 1. Change the following lines in the file /usr/local/etc/oai/spgw_c.conf
INSTANCE                       = 1;           
PID_DIRECTORY                  = "/var/run";

### SPGW-U - Installation - Configuration

Install SPGW-U

```markdown
cd openair-cn-cups/build/scripts
sudo ./build_spgwu -I -f
sudo ./build_spgwu -c -V -b Debug -j
```

Create configuration files of SPGW-U

```markdown
cp ../../etc/spgw_u.conf  /usr/local/etc/oai
```

Configure SPGW-U

- 1. Change the following lines in the file /usr/local/etc/oai/spgw_u.conf
INSTANCE                       = 1;           
PID_DIRECTORY                  = "/var/run";

### eNB - Installation - Configuration

Clone the repository and change to develop branch, then proceed with installation

```markdown
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git
cd ~/openairinterface
git branch
git checkout develop
source oaienv
cd cmake_targets
./build_oai -I --eNB -x --install-system-files -w USRP
```

Configure eNB

```markdown
sudo gedit ~/openairinterface5g/targets/PROJECTS/GENERIC-LTE-EPC/CONF/enb.band7.tm1.50PRB.usrpb210.conf
```
### Programming Sim Card

For programming the sim cards I use [UICC v1.7](https://open-cells.com/d5138782a8739209ec5760865b1e53b0/uicc-v1.7.tgz)
Download it, extract it then run 

```markdown
sudo ./program_uicc --adm=28385289 --opc=8e27b6af0e692e750f32667a3b14605d --key=fec86ba6eb707ed08905757b1bb44b8f --spn=openairinterface --isdn=001011234561000 --imsi=208930000000002 --authenticate
```

Copy the hss sqn next value as we going to update in the the database.
Lets say sqn is 37587.
We use the user 208930000000002.

```markdown
cqlsh
select imsi,key,msisdn,opc,rand,sqn from vhss.users_imsi where imsi ='208930000000002';
UPDATE vhss.users_imsi SET sqn=37527 WHERE imsi='208930000000002';
```

### Running HSS,MME,SPGW-C,SPGW-U and eNB

In different terminals type 

```markdown
oai_hss -j /usr/local/etc/oai/hss_rel14.json
~/openair-cn/scripts/run_mme --config-file /usr/local/etc/oai/mme.conf
sudo spgwc -oc /usr/local/etc/oai/spgw_c.conf
sudo spgwu -oc /usr/local/etc/oai/spgw_u.conf
sudo ~/openairinterface5g/cmake_targets/ran_build/build/lte-softmodem -O ~/openairinterface5g/targets/PROJECTS/GENERIC-LTE-EPC/CONF/enb.band7.tm1.50PRB.usrpb210.conf
```

## TROUBLESHOOTING SECTION

### 1. If there is a problem connecting to cassandra try the following
```markdown
sudo gedit /etc/cassandra/cassandra-env.sh
```
Uncomment line 268
`JVM_OPTS="$JVM_OPTS -Djava.rmi.server.hostname=127.0.0.1"`
```markdown
systemctl restart cassandra
```
### 2. If there is still a problem connecting to cassandra try the following
```markdown
sudo apt-get purge openjdk-\* icedtea-\* icedtea6-\*
```
