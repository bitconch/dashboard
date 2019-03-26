

Use Grafana to Setup a Dashboard
=======
* * *

## Install Grafana on Ubuntu

```bash
wget https://dl.grafana.com/oss/release/grafana_6.0.1_amd64.deb 

sudo apt-get install -y adduser libfontconfig

sudo dpkg -i grafana_6.0.1_amd64.deb 
```

* * *

## Configure Grafana

Semicolons (the ; char) are the standard way to comment out lines in a .ini file.
Change the port number to 9530
1. Open configure file grafana.ini with vim editor
```bash
vim /etc/grafana/grafana.ini
```
2. change port number to 9530, in grafana ini file, comment starts with ```#```, value comments with ```;```

```bash
# The http port  to use
http_port = 9530

# The full public facing url you use in browser, used for redirects and emails
# If you use reverse proxy and sub path specify full url (with sub path)
root_url = http://localhost:9530

```
3. save and restart grafana service

```
systemctl daemon-reload
systemctl start grafana-server
systemctl status grafana-server （"Press 'Q' to quit"）
systemctl restart grafana-server
```
check if grafana works: open browser to http://localhost:9530

[more about grafana configuration](http://docs.grafana.org/installation/configuration/)

* * *

## Set Grafana to HTTPS

### 1. generate certification file and key save to /etc/ssl/grafana-key.pem, /etc/ssl/grafana-cert.pem

1. move to grafana configuration directory
```bash
cd /etc/grafana
```
-    create a temporary self-signed certificate

```bash
openssl genrsa -out selfsigned-grafana.key 2048
openssl req -new -key selfsigned-grafana.key -out selfsigned-grafana.csr
openssl x509 -req -days 365 -in selfsigned-grafana.csr -signkey selfsigned-grafana.key -out selfsigned-grafana.crt
```

2.  Enter passphrase, and answers some stupid questions like 

    * Country Name likes US
    * State likes CA 
    * Locality likes LA
    * Orgnization  likes Bitconch PTE Ltd.
    * Unit Name Dev-Dashboard

3.  change the certification and key file permission
```bash
chown grafana:grafana selfsigned-grafana.crt
chown grafana:grafana selfsigned-grafana.key
chmod 400 selfsigned-grafana.crt 
chmod 400 selfsigned-grafana.key
```

### 2. change grafana configuration file and restart service

```bash
vim /etc/grafana/grafana.ini
```

```shell
[server]
# Protocol (http, https, socket)
protocol = https

# The http port  to use
http_port = 9530

# enable gzip
enable_gzip = true

# https certs & key file
cert_file = "/etc/grafana/selfsigned-grafana.crt"
cert_key = "/etc/grafana/selfsigned-grafana.key"

# set to true if you host Grafana behind HTTPS. default is false.
cookie_secure = true

```

### 3. restart the grafana service 

```bash
systemctl restart grafana-server
systemctl status grafana-server （"Press 'Q' to quit"）
```

### 4. check the grafana logging
```bash
vim /var/log/grafana/grafana.log
```
### 5. verify https works, open browser and go to https://localhost:9530


* * *



 Install Influxdb
=======

**You can install influxdb in following ways:**

* download and manually install

```
wget https://dl.influxdata.com/influxdb/releases/influxdb_1.5.0_amd64.deb
sudo dpkg -i influxdb_1.5.0_amd64.deb

```
* add apt repo from the [offical website](https://docs.influxdata.com/influxdb/v1.7/introduction/installation/)
```

wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list

```

*  install from apt repo

```
sudo apt-get update && sudo apt-get install influxdb
sudo service influxdb start
```
**everything looks fine if output looks like below when influxd service launched:**
**Open a influxdb client by type "influx" in terminal**
```
influx
```
**If everything goes well The output should look like this:**
```
Connected to http://localhost:8086 version 1.7.4
InfluxDB shell version: 1.7.4
Enter an InfluxQL query

```
**Exit from client by type "exit" and press "Enter" key**
```
> exit
```
##  Configure Influxdb

The configuration file for InfluxDB is ```influxdb.conf```, it has different location per different OS

* Linux: ```/etc/influxdb/influxdb.conf```

* MacOS: ```/usr/local/etc/influxdb.conf```

### enable https, and set the endpoint to https://metrics.bitconch.org
1. **Generate Root SSL certificate** 
```bash 
openssl genrsa -des3 -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
```
Enter passphrase, and answers some stupid questions like 
```
* Country Name: US
* State: CA
* Locality: LA
* Orgnization: Bitconch PTE Ltd.
* Unit Name: Dev-Dashboard-InfluxDB
* Common Name: Bitconch Authority
```
2. **Trust the root SSL certificate**
```bash
mkdir -p /usr/local/share/ca-certificates/influxdb
cp rootCA.pem /usr/local/share/ca-certificates/influxdb/rootCA.crt
update-ca-certificates
```
3. **Generate Domain SSL certificate**
- **Create a new OpenSSL configuration file server.csr.cnf with following contents**
```
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=US
ST=CA
L=LA
O=Bitconch PTE Ltd.
OU=Dev-Dashboard-InfluxDB
emailAddress=hello@example.com
CN = metrics.bitconch.org
```
- **Create a v3.ext file with following contents in order to create a X509 v3 certificate**
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = metrics.bitconch.org
```
- **Create a certificate key for metrics.bitconch.org using the configuration settings stored in server.csr.cnf. This key is stored in server.key.**
```
openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key -config <( cat server.csr.cnf )
```
- **A certificate signing request is issued via the root SSL certificate we created earlier to create a domain certificate for metrics.bitconch.org. The output is a certificate file called server.crt.**
```
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 500 -sha256 -extfile v3.ext
```
4. **Place the private key file (.key) and the signed certificate file (.crt) in the /etc/ssl directory.**
```bash
cp server.key /etc/ssl/
cp server.crt /etc/ssl/
```
5. **Set certificate file permissions**
```bash
sudo chown influxdb:influxdb /etc/ssl/server.crt
sudo chmod 644 /etc/ssl/server.crt
sudo chmod 600 /etc/ssl/server.key
```
6. **Enable Https in the influxdb configuration file**
```bash
vim /etc/influxdb/influxdb.conf
```
- https-enabled to true
- https-certificate to /etc/ssl/server.crt
- https-private-key to /etc/ssl/server.key
```

[http]

  [...]

  # Determines whether HTTPS is enabled.
  https-enabled = true

  [...]

  # The SSL certificate to use when HTTPS is enabled.
  https-certificate = "server.crt"

  # Use a separate private key location.
  https-private-key = "server.key"
```
7. **enable authentication**
- auth-enabled to true
```
# Determines whether user authentication is enabled over HTTP/HTTPS.
  auth-enabled = true

```
8. **Setting Global Variable**
Add the following contents to $HOME/.profile
```bash
vim $HOME/.profile
```
```
export INFLUX_HOST="https://metrics.bitconch.org"
export INFLUX_DATABASE="<DataBase Name>"
export INFLUX_USERNAME="<user name>"
export INFLUX_PASSWORD="<password>"
```
```bash
source $HOME/.profile
```
## check the influxdb 
- start the influxdb
```
chown -R influxdb:influxdb /var/lib/
systemctl start influxdb
systemctl status influxdb
```

- stop the influxdb 

```
systemctl stop influxdb
```

- resetart the infludb 

```
sudo systemctl restart influxdb
```

- connect to https enabled influxdb, since we are using selfsigned certificate, make sure the ```unsafeSsl``` is used.

```
 influx  -ssl -unsafeSsl -username admin -password 'password'

```


- create another user with admin access, and log in using the new user
```bash
CREATE USER caesar WITH PASSWORD '<password>' WITH ALL PRIVILEGES
exit
influx  -ssl -unsafeSsl  -username 'caesar' -password '<password>'
```
try new user's access, a dummy db is created and dropped.

```bash 
> SHOW  USERS

> SHOW DATABASES

> CREATE DATABASE dummy1

> SHOW DATABASES

name: databases
name
----
_internal
dummy1

> DROP DATABASE dummy1
> SHOW DATABASES
name: databases
name
----
_internal

> CREATE DATABASE dashboard01
```

## Install  InfluxDB rootCA to your localhost

* Ubuntu 
1. Download the /usr/local/share/ca-certificates/influxdb/rootCA.crt
```powershell
scp -r -P 22 user@remote_host:/usr/local/share/ca-certificates/influxdb/rootCA.crt rootCA.crt
```
2. install certificate
```powershell
cp rootCA.crt /usr/local/share/ca-certificates/influxdb
update-ca-certificates
```
* Windows

    Download the /etc/influxdb/selfsigned-influxdb.crt
```powershell
scp -r -P 22 user@remote_host:/usr/local/share/ca-certificates/influxdb/rootCA.crt rootCA.crt
```

    Double click the crt file and resave to cer file 

    Open and Run MMC

    Import the crt file






### Update the Grafana Dashboard Configuration

- grafana dash board is defined in ```testnet-dashboard-stable.json```

- set Grafna API Token 

    create an orgnization in the dashboard
```bash

curl -k -X POST -H "Content-Type: application/json" -d '{"name":"dev-org"}' https://admin:<password>@localhost:9530/api/orgs

{"message":"Organization created","orgId":3}

```
an orgnization with id ``3`` is created

```
curl -k -X POST https://admin:<password>@localhost:9530/api/user/using/3

{"message":"Active organization changed"}

```

create the API token 

```
curl -k -X POST -H "Content-Type: application/json" -d '{"name":"apikeycurl", "role": "Admin"}' https://admin:zaq12wsx@localhost:9530/api/auth/keys

>{"name":"apikeycurl","key":"eyJrIjoiUWJmTk9hRkx2anVCekVRSWU3UGN3ZkhiM2kxV1I0bnEiLCJuIjoiYXBpa2V5Y3VybCIsImlkIjozfQ=="}
>{"name":"apikeycurl2","key":"eyJrIjoiTVdRaEFiRDF0SThCTjJQTTdOckR2SElndjZ5ZThxdnAiLCJuIjoiYXBpa2V5Y3VybDIiLCJpZCI6M30="}
>{"name":"apikeycurl3","key":"eyJrIjoieG5IbmllOGxJWW9MY3o1dGlmMmNPU0dERzVzYzFKR0EiLCJuIjoiYXBpa2V5Y3VybDMiLCJpZCI6M30="}
>{"name":"dummy1","key":"eyJrIjoid2cyYUlrUnF2OHpIRlNLcW1sMzFYS3MwOEpWUzF6dXgiLCJuIjoiZHVtbXkxIiwiaWQiOjN9"}
```

set the GRAFANA_API_TOKEN

```
$ export GRAFANA_API_TOKEN="eyJrIjoiUWJmTk9hRkx2anVCekVRSWU3UGN3ZkhiM2kxV1I0bnEiLCJuIjoiYXBpa2V5Y3VybCIsImlkIjozfQ=="
- set the configuration parameter 

```bash
export DASHBOARD_DB_CONFIG="host=localhost,db=dashboard02,u=caesar,p=<password>"
```
- run ```init.sh``` to initilize metrics database.

    This will set env var ```DASHBOARD_DB_CONFIG```,```

- publish the dashboard

    * install python3 venv
```
sudo apt-get install python3-venv
```
    * make sure you have the access to avoid ```git@github.com: Permission denied (publickey)``
```bash
./publish-metrics-dashboard.sh
```

### Install Rust
```bash
$ curl https://sh.rustup.rs -sSf | sh
$ source $HOME/.cargo/env
$ rustup component add rustfmt-preview
```
```bash
$ rustup update
```
```bash
$ sudo apt-get install libssl-dev pkg-config zlib1g-dev llvm clang
```
```bash
$ cargo build --all
```
Run the test suite:

```bash
$ cargo test --all
```

To emulate all the tests that will run on a Pull Request, run:

```bash
$ ./ci/run-local.sh
```

### malformed entry error

if erros like happened:
```bash
E: Malformed entry 63 in list file /etc/apt/sources.list (URI parse)
E: The list of sources could not be read.
E: Malformed entry 63 in list file /etc/apt/sources.list (URI parse)
E: The list of sources could not be read.
```
This error message told us the 63 line of the file ```/etc/apt/sources.list``` is invalid, so just commented out the line would be ok. 
use vim to open /etc/apt/sources.list, and comment out the 63 line, or any other number of line

### Solutions for error likes: git@github.com: Permission denied (publickey).

A "Permission denied" error means that the server rejected your connection. on your local machine, you haven't made any SSH keys.

Here's how to fix:

Open git bash (Use the Windows search. To find it, type "git bash") or the Mac Terminal. Pro Tip: You can use any *nix based command prompt (but not the default Windows Command Prompt!)
Type cd ~/.ssh. This will take you to the root directory for Git (Likely C:\Users\[YOUR-USER-NAME]\.ssh\ on Windows)
Within the .ssh folder, there should be these two files: id_rsa and id_rsa.pub. These are the files that tell your computer how to communicate with GitHub, BitBucket, or any other Git based service. Type ls to see a directory listing. If those two files don't show up, proceed to the next step. NOTE: Your SSH keys must be named id_rsa and id_rsa.pub in order for Git, GitHub, and BitBucket to recognize them by default.
To create the SSH keys, type ssh-keygen -t rsa -C "your_email@example.com". This will create both id_rsa and id_rsa.pub files.
Now, go and open id_rsa.pub in your favorite text editor (you can do this via Windows Explorer or the OSX Finder if you like, tpying open . will open the folder).
Copy the contents--exactly as it appears, with no extra spaces or lines--of id_rsa.pub and paste it into GitHub and/or BitBucket under the Account Settings > SSH Keys. NOTE: I like to give the SSH key a descriptive name, usually with the name of the workstation I'm on along with the date.
Now that you've added your public key to Github and/or BitBucket, try to git push again and see if it works. It should!
