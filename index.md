# Installation Manual for OAI all in one

[HSS Install](#hss-installation-configuration)

[HSS Configure](#hss-configuration)

[MME Install](#Install-MME)

[MME Configure](#mme-configuration)

[SPGW-C](#spgw-c-installation-configuration)

[SPGW-U](#spgw-u-installation-configuration)

[eNB](#enb---installation---configuration)

[Programming Sim](#programming-sim-card)

[Run](#hss-mme-spgwc-spgwu-and-enb)

[Configuration Files](#configuration-files)

[Troubleshooting](#troubleshooting)

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
cd ~
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
cd ~
git clone git://github.com/EttusResearch/uhd.git
cd uhd; mkdir host/build; cd host/build
cmake -DCMAKE_INSTALL_PREFIX=/usr ..
make -j4
sudo make install
sudo ldconfig
sudo /usr/lib/uhd/utils/uhd_images_downloader.py
```
## openair-cn 

### HSS Installation Configuration

#### Before installing Cassandra 

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

#### Installing Cassandra

```sh
cd ~/openair-cn/scripts/
./build_cassandra --check-installed-software --force
```

#### Configuring Cassandra

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
#### HSS Configuration

#### HSS Configuration (1/5) : **Prework**

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
sudo chmod 777 /usr/local/etc/oai/freeDiameter/acl.conf /usr/local/etc/oai/freeDiameter/hss_rel14_fd.conf

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
#### HSS Configuration (2/5) : File **acl.conf**

Change @REALM@ with your realm 
Exp. If your realm is "ng4T.com"

`ALLOW_OLD_TLS  *.ng4T.com *.localdomain *.test3gpp.net`

```sh
sudo gedit /usr/local/etc/oai/freeDiameter/acl.conf
```
#### HSS Configuration (3/5) : File **hss_rel14_fd.conf**

```sh
hostname --fqdn
sudo gedit /usr/local/etc/oai/freeDiameter/hss_fd.conf.conf
```

Change **Line 7** to match your hss fqdn with realm : `Identity = "hss.ng4T.com"`

Change **Line 11** to match your realm : `Identity = "hss.ng4T.com"`

Change **Line 16-17** @PREFIX@ to match your oai install dir : `TLS_Cred = "/usr/local/etc/oai/freeDiameter/hss.cert.pem", "/usr/local/etc/oai/freeDiameter/hss.key.pem";
TLS_CA = "/usr/local/etc/oai/freeDiameter/cacert.pem";`

Change **Line 75** @PREFIX@ to match your oai install dir : `LoadExtension = "acl_wl.fdx" : "/usr/local/etc/oai/freeDiameter/acl.conf";`

#### HSS Configuration (4/5) : File **hss_rel14.conf**


```sh
sudo gedit /usr/local/etc/oai/hss_rel14.conf
```

Change **Line 24** to the cassandra server IP `cassandra_Server_IP = @cassandra_Server_IP@;`

#### HSS Configuration (5/5) : File **hss_rel14.json**

```sh
sudo gedit /usr/local/etc/oai/hss_rel14.json
```

Change **Line 2** @PREFIX@ to match your oai install dir : `"fdcfg": /usr/local/etc/oai/freeDiameter/hss_rel14_fd.conf",`

Change **Line 3** @HSS_FQDN@ baed on hostname --fqdn command : ` "originhost": "hss.ng4T.com",`

Change **Line 4** @REALM@ to match your realm : `"originrealm": "ng4T.com"`

Change **Line 11** @cassandra_Server_IP@ to match your cassandra server IP : `    "casssrv": "127.0.0.1", `

Change **Line 20** @OP_KEY@ to match your operator key : `    "optkey" : "1006020f0a478bf6b699f15c062e42b3",`

Change **Line 22** @ROAMING_ALLOWED@ to true : `    "roamallow"  : true,`

Change **Line 25** to `    "logname": "/usr/local/etc/oai/logs/hss.log",`

Change **Line 29** to `    "statlogname": "/usr/local/etc/oai/logs/hss_stat.log",`

Change **Line 32** to `    "auditlogname": "/usr/local/etc/oai/logs/hss_audit.log",`

Change **Line 36** to `    "ossfile": "/usr/local/etc/oai/conf/oss.json"`

### MME - Installation - Configuration

#### Install MME

```sh
cd ~/openair-cn/scripts
./build_mme --check-installed-software --force
./build_mme --clean
```

#### Create MME configuration files

```sh
sudo openssl rand -out ~/.rnd 128
cd ~/openair-cn/scripts
cp ../etc/mme_fd.sprint.conf  /usr/local/etc/oai/freeDiameter/mme_fd.conf
cp ../etc/mme.conf  /usr/local/etc/oai
sudo chmod 777 /usr/local/etc/oai/freeDiameter/mme_fd.conf /usr/local/etc/oai/mme.conf
```
#### MME Configuration

#### MME Configuration (1/2) : File **mme.conf**

Change the following lines in the file /usr/local/etc/oai/mme.conf

``sh
sudo gedit /usr/local/etc/oai/mme.conf
``

Change **Line 3** @REALM@ to your realm (mine is "ng4T.com") : `    REALM                                     = "ng4T.com";`

Change **Line 4** @INSTANCE@ : `    INSTANCE                                  = 1; `

Change **Line 5** @PID_DIRECTORY@ : `    PID_DIRECTORY                             = "/var/run";`

Change **Line 35** @PREFIX@ to match your oai install dir : `        S6A_CONF                   = "/usr/local/etc/oai/freeDiameter/mme_fd.conf";`

Change **Line 36** @HSS_HOSTNAME@ from /etc/hosts : `         HSS_HOSTNAME               = "hss";`

Change **Line 51** @MCC@, @MNC@, @MME_GID@, @MME_CODE : `         {MCC="208" ; MNC="93"; MME_GID="32768" ; MME_CODE="3"; } `

Change **Line 57** @MCC@, @MNC@, @TAC_2@ : `        {MCC="208" ; MNC="93";  TAC = "1"; } `

Delete **Lines 55** 

Delete **Lines 56** to set only one TAI example below.

```markdown
TAI_LIST = ( 
         {MCC="208" ; MNC="93";  TAC = "1"; }                       # YOUR TAI CONFIG HERE
    );
```

Change **Line 79** @MME_INTERFACE_NAME_FOR_S1_MME@ to loopback interface : `lo`

Change **Line 80** @MME_IPV4_ADDRESS_FOR_S1_MME@ : `127.0.1.1/24`

Change **Line 83** @MME_INTERFACE_NAME_FOR_S11@ to loopback interface : `lo`

Change **Line 84** @MME_IPV4_ADDRESS_FOR_S11@ : `127.0.11.1/24`

Change **Line 89** @MME_INTERFACE_NAME_FOR_S10@ to loopback interface : `lo`

Change **Line 90** @MME_IPV4_ADDRESS_FOR_S10@ : `127.0.10.1/24`

Change **Line 99** @OUTPUT@ : `CONSOLE`

Change **Line 120** @TAC-LB_SGW_TEST_0@, @TAC-HB_SGW_TEST_0@, @SGW_IPV4_ADDRESS_FOR_S11_TEST_0@ : `{ID="tac-lb01.tac-hb00.tac.epc.mnc093.mcc208.3gppnetwork.org" ; SGW_IPV4_ADDRESS_FOR_S11="127.0.11.2/24";},`

Change **Line 123** @TAC-LB_MME_1@, @TAC-HB_MME_1@, "@PEER_MME_IPV4_ADDRESS_FOR_S10_1@ : `{ID="tac-lb01.tac-hb00.tac.epc.mnc093.mcc208.3gppnetwork.org" ; PEER_MME_IPV4_ADDRESS_FOR_S10="0.0.0.0/24";}`

Delete **Line 121** 

Delete **Line 122** example below.

```markdown
WRR_LIST_SELECTION = (
        {ID="tac-lb01.tac-hb00.tac.epc.mnc093.mcc208.3gppnetwork.org" ; SGW_IPV4_ADDRESS_FOR_S11="127.0.11.2/24";},
        {ID="tac-lb01.tac-hb00.tac.epc.mnc093.mcc208.3gppnetwork.org" ; PEER_MME_IPV4_ADDRESS_FOR_S10="0.0.0.0/24";}
    );
```

#### MME Configuration (2/2) : File **mme_fd.conf**

Change the following lines in the file /usr/local/etc/oai/freeDiameter/mme_fd.conf

```sh
hostname
/usr/local/etc/oai/freeDiameter/mme_fd.conf
```

Change **Line 4** @MME_FQDN@ to hostname.realm : `oai.ng4T.com`

Change **Line 5** @REALM@ to match your realm : `ng4T.com`

Change **Line 8-10** @PREFIX@ : 
```markdown
TLS_Cred = "/usr/local/etc/oai/freeDiameter/mme.cert.pem",
           "/usr/local/etc/oai/freeDiameter/mme.key.pem";
TLS_CA   = "/usr/local/etc/oai/freeDiameter/mme.cacert.pem";
```

Change **Line 53** @MME_S6A_IP_ADDR@ : `ListenOn = "127.0.0.11";`

```sh
hostname --fqdn
```

Change **Line 103** @HSS_FQDN@, @HSS_IP_ADDR@, @REALM@" to match your configuration : `ConnectPeer= "hss.ng4T.com" { ConnectTo = "127.0.33.1"; No_SCTP ; No_IPv6; Prefer_TCP; No_TLS; port = 3868;  realm = "ng4T.com";};`

### Generate freeDiameter certificates for both HSS and MME

If you have different hostname (oai) and realm (ng4T.com) just change them here

```sh
cd ~/openair-cn/scripts
./check_hss_s6a_certificate /usr/local/etc/oai/freeDiameter/ hss.ng4T.com
./check_mme_s6a_certificate /usr/local/etc/oai/freeDiameter/ oai.ng4T.com
```

### Configure hosts

```sh
hostname --fqdn
sudo gedit /etc/hosts
```

Modify to suit your hostname fqdn
Exp. If your hostname fqdn is "oai"

```markdown
127.0.0.1	localhost
127.0.1.1 oai.ng4T.com oai
127.0.33.1 hss.ng4T.com hss 
```

## openair-cn-cups

### SPGW-C Installation Configuration

#### Install SPGW-C

```sh
cd ~/openair-cn-cups/build/scripts
sudo ./build_spgwc -I -f
sudo ./build_spgwc -c -V -b Debug -j
```

Create configuration files of SPGW-C

```sh
cp ../../etc/spgw_c.conf  /usr/local/etc/oai
```

#### Configuration of SPGW-C : File **spgw_c.conf**

Change the following lines in the file /usr/local/etc/oai/spgw_c.conf

Change **Line 23** from @INSTANCE@ to : `1`

Change **Line 24** from @PID_DIRECTORY@ to : `/var/run`

Change **Line 71** from @SGW_INTERFACE_NAME_FOR_S11@ to : `lo`

Change **Line 72** from read to : `127.0.11.2/24`

Change **Line 84** from @SGW_INTERFACE_NAME_FOR_S5_S8_CP@ read to : `lo`

Change **Line 85** from read to : `127.0.13.1/24`

Change **Line 99** from @INSTANCE@ to : `1`

Change **Line 100** from @PID_DIRECTORY@ to : `/var/run`

Change **Line 147** from @PGW_INTERFACE_NAME_FOR_S5_S8_CP@ to : `lo`

Change **Line 148** from read to : `127.0.13.2/24`

Change **Line 160** from @PGW_INTERFACE_NAME_FOR_SX@ to : `lo`

Change **Line 161** from read to : `127.0.12.1/24`

Change **Line 195** from default to the apn you setup at data_provisioning_users when configuring cassandra : `default.ng4T.com`

Change **Line 203** from @DEFAULT_DNS_IPV4_ADDRESS@ to : `8.8.8.8`

Change **Line 204** from @DEFAULT_DNS_SEC_IPV4_ADDRESS@ to : `8.8.4.4`

### SPGW-U Installation Configuration

#### Install SPGW-U

```sh
cd openair-cn-cups/build/scripts
sudo ./build_spgwu -I -f
sudo ./build_spgwu -c -V -b Debug -j
```

Create configuration files of SPGW-U

```sh
cp ../../etc/spgw_u.conf  /usr/local/etc/oai
```

#### Configure SPGW-U

Change the following lines in the file /usr/local/etc/oai/spgw_u.conf

Change **Line 23** from @INSTANCE@ to : `1`

Change **Line 24** from @PID_DIRECTORY@ to : `/var/run`

Change **Line 59** from @SGW_INTERFACE_NAME_FOR_S1U_S12_S4_UP@ to : `lo`

Change **Line 60** from read to : `127.0.14.2/24`

Change **Line 72** from @SGW_INTERFACE_NAME_FOR_SX@ to : `lo`

Change **Line 73** from read to : `127.0.12.2/24`

Change **Line 85** from @SGW_INTERFACE_NAME_FOR_SGI@ to the interface you get access to internet, run ifconfig : `eno1`

Change **Line 105** from 192.168.160.100 to : `127.0.12.1`

## openairinterface5g

### eNB - Installation - Configuration

#### Install eNB
Clone the repository and change to develop branch, then proceed with installation

```sh
cd ~
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git
cd ~/openairinterface
git branch
git checkout develop
source oaienv
cd cmake_targets
./build_oai -I --eNB -x --install-system-files -w USRP
```

#### Configure eNB

Change the following lines in the file ~/openairinterface5g/targets/PROJECTS/GENERIC-LTE-EPC/CONF/enb.band7.tm1.50PRB.usrpb210.conf

```sh
sudo gedit ~/openairinterface5g/targets/PROJECTS/GENERIC-LTE-EPC/CONF/enb.band7.tm1.50PRB.usrpb210.conf
```

Change values at **Line 18** of mcc and mnc to match your configuration : `plmn_list = ( { mcc = 208; mnc = 93; mnc_length = 2; } );`

Change value at **Line 34** to : `2625000000L`

Change value at **Line 34** to : `25;`

Change value at **Line 43** to : `115;`

Change value at **Line 175** ipv4 to : `127.0.1.1`

Change value at **Line 191** ENB_INTERFACE_NAME_FOR_S1_MME to : `lo`;

Change value at **Line 192** ENB_IPV4_ADDRESS_FOR_S1_MME to : `127.0.1.3/24`;

Change value at **Line 193** ENB_INTERFACE_NAME_FOR_S1U to : `lo`;

Change value at **Line 194** ENB_IPV4_ADDRESS_FOR_S1U to : `127.0.14.3/24`;

Change value at **Line 278** parallel_config to : `PARALLEL_SINGLE_THREAD`;


## Programming Sim Card

For programming the sim cards I use [UICC v1.7](https://open-cells.com/d5138782a8739209ec5760865b1e53b0/uicc-v1.7.tgz)
Download it, extract it then run 

```sh
sudo ./program_uicc --adm=28385289 --opc=8e27b6af0e692e750f32667a3b14605d --key=fec86ba6eb707ed08905757b1bb44b8f --spn=openairinterface --isdn=001011234561000 --imsi=208930000000002 --authenticate
```

Copy the hss sqn next value as we going to update in the the database.
Lets say sqn is 37587.
We use the user 208930000000002.

```sh
cqlsh
select imsi,key,msisdn,opc,rand,sqn from vhss.users_imsi where imsi ='208930000000002';
UPDATE vhss.users_imsi SET sqn=37527 WHERE imsi='208930000000002';
```

## Running HSS, MME, SPGW-C, SPGW-U and eNB

In different terminals type 

```sh
oai_hss -j /usr/local/etc/oai/hss_rel14.json
~/openair-cn/scripts/run_mme --config-file /usr/local/etc/oai/mme.conf
sudo spgwc -oc /usr/local/etc/oai/spgw_c.conf
sudo spgwu -oc /usr/local/etc/oai/spgw_u.conf
sudo ~/openairinterface5g/cmake_targets/ran_build/build/lte-softmodem -O ~/openairinterface5g/targets/PROJECTS/GENERIC-LTE-EPC/CONF/enb.band7.tm1.50PRB.usrpb210.conf
sudo iptables -A FORWARD -i eno1 -o pdn -j ACCEPT 
```
A the iptables command replace eno1 with the interface that you access internet from ifconfig

## Configuration Files

[acl.conf](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/acl.conf.txt)

[cassandra.conf](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/cassandra.conf.txt)

[hss_rel14_fd.conf](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/hss_rel_14_fd.conf.txt)

[hss_rel14.conf](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/hss_rel14.conf.txt)

[hss_rel14.json](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/hss_rel14.json.txt)

[mme.conf](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/mme.conf.txt)

[mme_fd.conf](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/mme_fd.conf.txt)

[spgw-c.conf](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/spgw_c.conf.txt)

[spgw-u.conf](https://raw.githubusercontent.com/angelo-ath/oai/gh-pages/conf_files/spgw_u.conf.txt)

## TROUBLESHOOTING 

### 1. If there is a problem connecting to cassandra try the following
```sh
sudo gedit /etc/cassandra/cassandra-env.sh
```
Uncomment line 268
`JVM_OPTS="$JVM_OPTS -Djava.rmi.server.hostname=127.0.0.1"`
```sh
systemctl restart cassandra
```
### 2. There is still a problem connecting to cassandra try the following
```sh
sudo apt-get purge openjdk-\* icedtea-\* icedtea6-\*
```

### 3. When trying to run hss you get assertion error
Look for syntax errors in hss config files

### 4. When trying to run hss you get missing files
Just create the missing folder and file with the touch command
