# How to Install Portainer using TLS on Ubuntu 16.04

### Introduction

The Transport Layer Security (TLS) is a cryptographic protocol which facilitates secure connections over a network. It allows the bi-directional exchange of encrypted data between a client and a server using certified digital keys for authentication.
[Portainer](https://portainer.io/) is an open-source web UI designed to manage your Docker hosts and Swarm clusters in a friendly way. Some of its core features include the management of Docker containers, images, networks and volumes, swarm cluster overview, access to container's console and the ability to use pre-configured container templates or custom created templates.

The preferred method of using Portainer is as a container because is simpler to implement and offers more flexibility. You can choose to run the application from your Docker host or from your local machine. In both cases, a secure connection is highly recommended.

In this tutorial you will install Portainer on your local machine and establish a secure TLS connection with a remote Docker host that runs Ubuntu 16.04.

![Secure TLS Portainer Communication](https://i.imgur.com/x0r9kxj.png)

## Prerequisites

In order to complete this tutorial you will need:

* One Droplet with a minimum of 1 GB of RAM running Ubuntu 16.04 configured using [this initial server setup tutorial,](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04) including a sudo non-root user and a firewall.
* A local client PC with Internet connection running Ubuntu 16.04.
* Docker-ce installed in both ends (client and server). You will find detailed instructions for Docker installation in our guide [How To Install and Use Docker on Ubuntu 16.04.](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04)

## Step One - Create a Self-Signed SSL Certificate 

In order to establish a secure connection between your local and remote Docker daemons you will need:

* A trusted commercial certificate bundle issued by a Certificate Authority (CA) or a self-signed SSL certificate with its accompanying private key.
* A private key and certified public key for your server.
* A private key and certified public key for your client.

This guide will use a self-signed SSL certificate as its Certificate Authority (CA). If you already have a commercial certificate you can skip this step entirely. In case you are interested in purchasing a certificate please read our guide [How To Install an SSL Certificate from a Commercial Certificate Authority.](https://www.digitalocean.com/community/tutorials/how-to-install-an-ssl-certificate-from-a-commercial-certificate-authority)

In order to simplify this process, the required keys and certificates will be generated from your client.

The first step is creating a new directory to store the necessary files, for the purpose of this tutorial we will call it `certs`. 

```command
[environment local]
mkdir ~/certs
```

Move to the directory:

```command
[environment local]
cd ~/certs
```
    
Now generate your first key pair using `openssl`. The command below will create a 4096-bit RSA private key called `ca-key.pem` and a 64-bit hashed public certificate called `ca-cert.pem` with a validity of 365 days. The keys will be created using the `X.509` format what is the most widely used TLS/SSL format:

```command
[environment local]
openssl req -newkey rsa:4096 -keyout ca-key.pem -x509 -days 365 -sha512 -out ca-cert.pem
```

You will be prompted for a password phrase during the process, keep that information in a safe place. Try to fill the rest of the certificate information as accurate as you can paying special attention to the **Common Name** (CN) field which must include your server Fully Qualified Domain Name (in case you have one) or its public IP address. 

<$>[note]
**Note:**  This guide will intentionally use the server public IP address for the CN field. If you own a domain a better fit to obtaining CA keys could be using **Let's Encrypt**, read our tutorial [How To Use Certbot Standalone Mode to Retrieve Let's Encrypt SSL Certificates](https://www.digitalocean.com/community/tutorials/how-to-use-certbot-standalone-mode-to-retrieve-let-s-encrypt-ssl-certificates) for further instructions. 
<$> 

Once you issue the `openssl` command the you will see an output similar to this:

```
[secondary_label SSL Certificate Output]
Generating a 4096 bit RSA private key
.....................................................................++
..............................++
writing new private key to 'ca-key.pem'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:<^>US<^>
State or Province Name (full name) [Some-State]:<^>California<^>
Locality Name (eg, city) []:<^>San Francisco<^>
Organization Name (eg, company) [Internet Widgits Pty Ltd]:<^>My Company<^>
Organizational Unit Name (eg, section) []:<^>DevOps<^>
Common Name (e.g. server FQDN or YOUR name) []:<^>your_server_ip<^>
Email Address []:<^>sammy@example.com<^>
```

As mentioned before, once the process is finished you will have two new keys:

* **ca-key.pem**: this is your CA private key (that uses a phrase password for further security). 
* **ca-cert.pem**: this is your self-signed CA public certificate.

<$>[warning]
**Warning:** Keep in mind that these keys will be used to sign all your certificates, take the necessary actions to protect them because anyone with access to these keys could sign and authenticate with your applications.
<$>

## Step Two - Create Server Keys

While you are still in `certs` directory generate your server 4096-bit RSA private key:

```command
[environment local]
openssl genrsa -out server-key.pem 4096
```

Next, using the server's private key create a signing request `server.csr`. Be careful to use the same CN (common name) that you filled during the CA creation.

```command
[environment local]
openssl req -subj "/CN=<^>your_server_ip<^>" -sha512 -new -key server-key.pem -out server.csr
```

Before signing the public key is necessary to create a file that contains the certificate's extensions:

```
[label ~/certs/server-extfile.cnf]
extendedKeyUsage = serverAuth 
subjectAltName = DNS:<^>your_server_ip<^>,IP:<^>your_server_ip<^>,IP:127.0.0.1
```

Let's break down the above file. The first line ensures that the key can be used for server authentication. The second line is very important because creates an alternative CN which allows connections to your server using its public IP address (necessary if you are not using FQDN).

<$>[note]
**Note:** The advantage of using the `subjectAltName` variable is that you won't need a FQDN for authentication. However, keep in mind that if the public IP address changes you will need a new certificate.
<$>

Now you can generate your server certificate and sign it using your previously created CA keys, you will need the CA password phrase as well:

```command
[environment local]
openssl x509 -req -days 365 -sha512 -in server.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -extfile server-extfile.cnf -out server-cert.pem
```

You will find this command quite similar to the one used for public CA creation. The main difference is the flags used for specifying the input parameters:

* **-in**: points to the certificate signing request.
* **-CA**: indicates the location of your public CA certificate.
* **-CAkey**: indicates the path of your private CA key.
* **-extfile**: points to the certificate extension file location.
* **-CAcreateserial**: indicates the creation of a serial for the certificate.

After completing this section you will have the required key pair for your Docker host:

* **server-key.pem**: server's private key.
* **server-cert.pem**: server's public certificate.
* **ca.srl**: this is your certificate signing serial.

## Step Three - Create Client Keys

The procedure to create your client keys is similar to the one used before. You can generate as many keys as necessary while they are signed by your CA. Then you can use those keys to authenticate with your remote Docker host that will verify them against the same CA.

Similar than before, let's start with the private key:

```command
[environment local]
openssl genrsa -out client-key.pem 4096
```

Next create the signing request. This time the common name can be different from your CA we will use **client** to differentiate it from the previous one:

```command
[environment local]
openssl req -subj '/CN=<^>client<^>' -sha512 -new -key client-key.pem -out client.csr
```

Add a certificate extension file `client-extfile.cnf` to allow client authentication:

```command
[environment local]
echo extendedKeyUsage = clientAuth >> client-extfile.cnf 
```

Once you have all set you can finally generate the certified public key for your client (remember that you will need the CA password phrase):

```command
[environment local]
openssl x509 -req -days 365 -sha512 -in client.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -extfile client-extfile.cnf -out client-cert.pem
```

Similar to the last section you will end up with two new keys:

* **client-key.pem**: client's private key.
* **client-cert.pem**: client's public certificate.

Now that you have all the required keys it's time to clean-up your `certs` directory, enforce security changing keys permissions and transfer the server keys to your remote Docker host.

Move to the `certs` directory (if you are not already there):

```command
[environment local]
cd ~/certs
```

Remove all unnecessary files like signing requests and certificate extension files:

```command
[environment local]
rm -v *.csr *.cnf
```

Now change the permissions of your keys for enhanced security:

```command
[environment local]
chmod -v 0400 *key*
chmod -v 0444 *cert*
```

The first command gives your private keys read-only permission for your current user. The second command gives your certificates a world-readable permission. Disabling the writing permissions is a secure way to avoid accidental (or intentional) damage to your keys.

Copy the server's keys to their appropriate location. To avoid mistakes we'll use the same convention saving the server keys in a new directory `~/certs`:

Connect to your remote Docker host and create the new directory:

```command
mkdir ~/certs
```

Move to the directory:

```command
cd ~/certs
```

The easier method to move the server keys is using `scp` which stands for *Secure Copy Protocol*. From your local client run this command:

```command
[environment local]
scp server* ca-cert.pem <^>sammy<^>@<^>your_server_ip<^>:/home/<^>sammy<^>/certs
```

That initiates an SSH connection and copies the server's keys and CA certificate from the local client to the server `~/certs` directory.

Please remember to change the example user name `sammy` and your server public IP `your_server_ip` address accordingly.

## Step Four - Configure Docker Daemon in your Remote Server

Your next action is to properly set up the Docker host to listen at port `2376` using a secure TLS connection. But before editing any server configuration is a good idea to test your secure connection. 

The first step is opening your firewall for incoming requests from your client:

```command
sudo ufw allow 2376
```

Now make sure that your Docker service is stopped on your server:

```command
sudo systemctl stop docker.service
```

This is necessary because you'll test the connection directly through Docker's daemon `dockerd`. Run this command while you are inside the `certs` directory to start the Docker daemon manually:

```command
sudo dockerd --tlsverify --tlscacert=ca-cert.pem --tlscert=server-cert.pem --tlskey=server-key.pem -H=0.0.0.0:2376
```

The command above is instructing `dockerd` to listen secure TLS connections on port `2376` using your current keys. 

Now that your server is ready and waiting for connections you can send a request from your local machine. For example, you can ask the Docker version of your server running from `certs` directory: 

```command
[environment local]
docker --tlsverify --tlscacert=ca-cert.pem --tlscert=client-cert.pem --tlskey=client-key.pem -H=<^>your_server_ip<^>:2376 version
```

If everything goes ask expected the output will be similar to this:

```
[secondary_label TLS connection test output]
Client:
 Version:       17.12.0-ce
 API version:   1.35
 Go version:    go1.9.2
 Git commit:    c97c6d6
 Built: Wed Dec 27 20:10:36 2017
 OS/Arch:       linux/amd64

Server:
 Engine:
  Version:      17.12.0-ce
  API version:  1.35 (minimum version 1.12)
  Go version:   go1.9.2
  Git commit:   c97c6d6
  Built:        Wed Dec 27 20:09:53 2017
  OS/Arch:      linux/amd64
  Experimental: false
```

Both versions (client and server) should be listed. To finish the test stop the daemon on your server pressing **Ctrl+C** you will see a message informing that the daemon was gracefully shutdown.

```
[secondary_label TLS connection shutdown output]
INFO[2018-02-27T06:29:50.879463699Z] stopping event stream following graceful shutdown  error="context canceled" module=libcontainerd namespace=plugins.moby
INFO[2018-02-27T06:29:50.879960739Z] stopping event stream following graceful shutdown  error="context canceled" module=libcontainerd namespace=moby
INFO[2018-02-27T06:29:50.988661535Z] stopping healtcheck following graceful shutdown  module=libcontainerd
```

Knowing that your connection is working you can now start the real configuration on your Docker host. It's a two-step process first, you will add the new listening port and second you will configure the daemon flags.

By default, the Docker daemon listens for connections on a UNIX socket. To accept requests from TCP ports you need to edit the service manually. The safer method to do it is using the command:

```command
sudo systemctl edit docker.service
```

You will be presented with a blank file, add the following content:

```
[label docker.service]
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2376
```

Editing the service is mandatory in Ubuntu 16.04 because the `docker.service` unit overrides the `hosts` parameter included in any other configuration file.

On the other hand, to include the location of your TLS keys it's not necessary to edit the service file. You can follow [Docker documentation suggestion](https://docs.docker.com/config/daemon/systemd/#start-the-docker-daemon) using a `daemon.json` file to configure the daemon flags and environment variables.

Using your favorite editor create the `daemon.json` file and add the appropriate configuration values to enable TLS verification as well as certificates location:

```
[label /etc/docker/daemon.json]
{
"tls": true,
"tlsverify": true,
"tlscacert": "/home/<^>sammy<^>/certs/ca-cert.pem",
"tlscert": "/home/<^>sammy<^>/certs/server-cert.pem", 
"tlskey": "/home/<^>sammy<^>/certs/server-key.pem" 
}
```

Once you save the changes reload the Docker daemon:

```command
sudo systemctl daemon-reload
```

Now you can start your Docker service as usual:

```command
sudo systemctl start docker.service
```

To check your new configuration you can verify the `dockerd` listening port:

```command
sudo ps aux |grep dockerd
```

Your console output may vary but should be similar to:

```
[secondary_label TLS connection shutdown output]
root     30754  4.5  5.0 427456 51780 ?        Ssl  06:33   0:00 /usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2376
sammy    30861  0.0  0.0  12944   968 pts/0    S+   06:33   0:00 grep --color=auto dockerd
```

The first line shows how the server is now listening to its local UNIX socket (default behavior) as well as tcp port `2376`. 

Finally, to enable the Docker service during startup run:

```command
sudo systemctl enable docker.service
```

Up to this point you have successfully configured the remote Docker host which is now ready to accept **certified** TLS connections from your client. In the next step you will set up Portainer to communicate with the server.

## Step Five - Installing Portainer

In this section you will deploy Portainer in your local client and then you will configure it to connect with your remote Docker host using a secure TLS connection.

The first thing you need is creating a Docker volume that stores Portainer persistent data. From your local client run the command:

```command
[environment local]
sudo docker volume create portainer_data
```

Portainer installation can't be easier, just run its image using the following flags:

```command
[environment local]
docker run -d -p 9000:9000 --name portainer --restart unless-stopped -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

Let's take a closer look to the above command:

* **-d**: run the container in detached mode.
* **-p 9000:9000**: open the port `9000` for incoming connections.
* **--name**: assing the name **portainer** to your container.
* **--restart unless-stopped**: as the name implies keep the container running unless you decide to manually stop it.
* **-v /var/run/docker.sock:/var/run/docker.sock**: allows communication with the local Docker daemon from the container.
* **-v portainer_data:/data**: mounts your Docker volume for persistent data.
* **portainer/portainer**: downloads the official image.

Once the image is downloaded and the container is started you can use your browser to complete Portainer configuration at the address: `http://localhost:9000`. 

<$>[note]
**Note:** Don't confuse the secure TLS connection between Docker daemons with the `http` connection of the Portainer web UI. Remember that you are using `http` to access a **local** container that behind scenes is communicating through TLS to the remote server. 
<$>

Create your Administrative password and click the **Create user** button.

![Portainer First Login](https://i.imgur.com/VxUeaTZ.png)

The next screen will present you the option to connect Portainer locally or with a remote server. Choose **Remote** and enable TLS options using the TLS switch. Then fill the required information: a name for this connection, the endpoint URL or FQDN and the appropriate keys. You will end with something similar to this:

<$>[note]
**Note:** Remember to add the port `2376` at the end of your public IP address. Your URL should look similar to: `<^>your_server_ip<^>:2376`.
<$>

![Portainer Remote Connection](https://i.imgur.com/6OuMCCt.png)

Once you are ready press the **Connect** button. You will be see Portainer Dashboard which includes an overview of your remote server information as shown in the following image:

![Portainer Dashboard](https://i.imgur.com/ueWBpy1.png)

You are now in control of your remote Docker host using the local Portainer installation. In the next section you will learn how to add more nodes to your client.

## Step Six - Configure Additional Nodes with Portainer

Under **PORTAINER SETTINGS** in the side menu select **Endpoints**. You can add as many nodes as needed from this screen. The minimal information you must provide is the name of the endpoint and its URL (including listening port). 

![Portainer Endpoints](https://i.imgur.com/lQUgPLr.png)

You can also configure a secure connection, just toggle the TLS switch and you will see the additional information needed:

![Portainer TLS Endpoint](https://i.imgur.com/agBgTu8.png)

Using Portainer UI you have the option to configure any combination that suit your needs:

* **TLS with server and client verification (default)**: this is the same type of connection you configured in this tutorial.
* **TLS with client verification only**: only checks the client private key and CA certificate.
* **TLS with server verification only**: only checks the CA certificate.
* **TLS only**: is the less secure option that requires no verification of keys or certificates.

Depending on your selection the UI will ask for the required key files. Click the **Select file** button and choose the appropriate key/certificate. For example, if you want to configure a connection with server and client verification you need to provide the following files:

![Portainer new TLS endpoint](https://i.imgur.com/jBPEFs2.png)

Using the procedures described in this tutorial you can create the new client/server keys and certificates. Depending on your needs you can use the same CA certificate or generate a new one.

<$>[note]
**Note:** Remember that you are running Portainer locally, the new endpoint configuration only applies to the application. In order to establish a successful connection with a new endpoint you'll need to complete beforehand the steps two **Create Server Keys** and four **Configure Docker in your Remote Server.**
<$>

## Conclusion

You now can enjoy the convenient benefits of Portainer to manage your Docker hosts from your local machine using a secure TLS connection.

This guide only covered a small part of Portainer features, for more information about its other functions read the official [documentation.](https://portainer.readthedocs.io/en/stable/)
