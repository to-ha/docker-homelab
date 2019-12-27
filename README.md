# Docker homelab using docker-compose
## Purpose
A homelab based on docker containers using docker-compose for orchestration.
This repo and guide are for an x86_64 Intel based server running Ubuntu 18.04 LTS.
The setup is for educational purposes inside a private LAN, not for a production environment or a setup exposed to public networks.

## Prerequisites
### Install Docker
I.e. for Ubuntu server 18.04 LTS follow the instruction at https://docs.docker.com/install/linux/docker-ce/ubuntu/
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt update

sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
Check the fingerprint of the image against the one on the webpage:
```bash
sudo apt-key fingerprint 0EBFCD88
    
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```
When adding the repository make sure to add the correct architecture (in my case x86_64)
```bash
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
sudo docker run hello-world
```

### Install docker-compose
Follow instructions at https://docs.docker.com/compose/install/

Use latest stable release, find associated URL at https://github.com/docker/compose/releases
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Generate TLS certificate
My installation is not exposed to the public internet. It is LAN only. This means Let's Encrypt is not suitable to sign certificates.
For use in a LAN environment self-signed certificates can be used.

<ol>
<li>
Generate private key rootTLS.key
<pre><code>
mkdir ~/TLS
cd ~/TLS
openssl genrsa -des3 -out rootTLS.key 2048
</code></pre>
Enter a passphrase when prompted.
</li>

<li>
Create rootTLS.pem certificate file using the key from step 1 (you will be asked for the passphrase you entered earlier)
<pre><code>
openssl req -x509 -new -nodes -key rootTLS.key -sha256 -days 3650 -out rootTLS.pem

You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.

Country Name (2 letter code) [AU]:AU
State or Province Name (full name) [Some-State]:State
Locality Name (eg, city) []:City
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Organization
Organizational Unit Name (eg, section) []:OrganizationUnit
Common Name (e.g. server FQDN or YOUR name) []:localhost
Email Address []:yourname@yourdomain
</code></pre>

Note: If you get the following error:

_random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/home/<yourusername>/.rnd_

you can create the ~/.rnd file yourself:
<pre><code>
dd if=/dev/urandom of=~/.rnd bs=256 count=1
</code></pre>
then repeat the step.
</li>

<li>
Trust your new own Certificate Authority (CA) by copying it into the Ubuntu trust store.
<pre><code>
sudo mkdir /usr/local/share/ca-certificates/extra
sudo cp ~/TLS/rootTLS.pem /usr/local/share/ca-certificates/extra/rootTLS.crt
sudo update-ca-certificates
</code></pre>
</li>

<li>
Create a new private key for your host and an associated certificate signing request.
Again you need to specifiy some information. The important part is the common name (CN) in the subject line. This needs to be localhost or the correct hostname.
<pre><code>
openssl req \
 -new -sha256 -nodes \
 -out localhost.csr \
 -newkey rsa:2048 -keyout localhost.key \
 -subj "/C=AU/ST=State/L=City/O=Organization/OU=OrganizationUnit/CN=localhost/emailAddress=yourname@yourdomain"
</code></pre>
</li>

<li>
Sign the key with your own root CA:
<pre><code>
openssl x509 \
 -req \
 -in localhost.csr \
 -CA rootTLS.pem -CAkey rootTLS.key -CAcreateserial \
 -out localhost.crt \
 -days 500 \
 -sha256 \
 -extfile <(echo " \
 [ v3_ca ]\n \
 authorityKeyIdentifier=keyid,issuer\n \
 basicConstraints=CA:FALSE\n \
 keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment\n \
 subjectAltName=DNS:localhost \
 ")
</code></pre>
</li>

<li>
Copy your private key and associated certificate to where the traefik reverse proxy can find it.
The location is specified in the file docker-homelab/my-docker-data/traefik/dynamic_conf.toml (in this case /etc/traefik/).
There is a bind mount for the traefik container in the docker-compose.yaml ${MY_DOCKER_DATA_DIR}/traefik:/etc/traefik

So you need to copy the two files to docker-homelab/my-docker-data/traefik.
The code block assumes you cloned the repo into your home directory under ~/docker-homelab
<pre><code>
cp ~/TLS/localhost.crt ~/docker-homelab/my-docker-data/traefik/
cp ~/TLS/localhost.key ~/docker-homelab/my-docker-data/traefik/
</code></pre>
</li>
</ol>
