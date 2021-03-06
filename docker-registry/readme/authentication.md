
### Registry Server: Set up authentication

#### Create User
`sudo apt-get -y install apache2-utils`
>use htpasswd to create password hash 

1. `cd ~/docker-registry/nginx`
2. `htpasswd -c registry.password bo` 
3. `htpasswd  registry.password bo` to check (without `-c` , c for create)

```
>  curl http://localhost:5043/v2/
<html>
<head><title>401 Authorization Required</title></head>
<body bgcolor="white">
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx/1.9.15</center>
</body>
</html>
bochen2014@bchen:/opt/docker-registry/nginx$ curl http://bo:bo@localhost:5043/v2/
{}
bochen2014@bchen:/opt/docker-registry/nginx$ 
```

#### Signing Your Own Certificate

Since Docker currently doesn't allow you to use self-signed SSL certificates this is a bit more complicated than usual — we'll also have to set up our system to act as our own certificate signing authority.

1. `cd ~/docker-registry/nginx`
2. Generate a new root key: `openssl genrsa -out devdockerCA.key 2048`
3. Generate a root certificate (enter whatever you'd like at the prompts):  
`openssl req -x509 -new -nodes -key devdockerCA.key -days 10000 -out devdockerCA.crt` (the crt here is x509 which is PEM essentially; DER is binary format; crt could be either binary DER or ascII PEM ---BEGIN )
4. Then generate a key for your server (this is the file referenced by `ssl_certificate_key` in `/ect/nginx/config.d/registry.conf`):
`openssl genrsa -out domain.key 2048`
5.  Now we have to make a certificate signing request.
`openssl req -new -key domain.key -out dev-docker-registry.com.csr`
After you type this command, OpenSSL will prompt you to answer a few questions. Write whatever you'd like for the first few, but when OpenSSL prompts you to enter the "Common Name" make sure to type in the domain or IP of your server.

For example, if your Docker registry is going to be running on the domain `www.example.com`, then your input should look like this:
```sh
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:www.example.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
# Do not enter a challenge password.
A challenge password []:
An optional company name []:

```
6. sign the certificate request:
`openssl x509 -req -in dev-docker-registry.com.csr -CA devdockerCA.crt -CAkey devdockerCA.key -CAcreateserial -out domain.crt -days 10000`

by now, you have both `domain.crt` and `domain.key` generated;

```sh
server {
  listen 443;
  server_name registry.bambora.co.nz;

  # SSL
  ssl on;
  ssl_certificate /etc/nginx/conf.d/domain.crt;
  ssl_certificate_key /etc/nginx/conf.d/domain.key;

  location /api/ {
     root /usr/share/nginx/html;
     index index.html index.htm;
  }
}  
```
#### Registry Client (docker daemon): Trust the certicifcate

Since the certificates we just generated aren't verified by any known certificate authority (e.g., VeriSign), we need to tell any clients that are going to be using this Docker registry that this is a legitimate certificate. Let's do this locally on the host machine so that we can use Docker from the Docker registry server itself:
```bash
sudo mkdir /usr/local/share/ca-certificates/docker-dev-cert
sudo cp devdockerCA.crt /usr/local/share/ca-certificates/docker-dev-cert
sudo update-ca-certificates
```
if you don't have `devdockerCA.crt` as you didn't set it up, you can do `openssl s_client -connect example.com.au:443 -showcerts`. this will print out the full certifcate chain.

Restart the Docker daemon so that it picks up the changes to our certificate store:
`sudo service docker restart`

>Warning: You'll have to repeat this step for every machine that connects to this Docker registry! Instructions for how to do this for Ubuntu 14.04 clients are listed in Step 9 — Accessing Your Docker Registry from a Client Machine.

Here are the examples tested on centos and tool-box;
#### on CentOS (tested on centOS 7.2)
1. Install the ca-certificates package:
`yum install ca-certificates`
2. Enable the dynamic CA configuration feature:
`update-ca-trust force-enable`
3. Add it as a new file to `/etc/pki/ca-trust/source/anchors/`  
  3.1 `cp foo.crt /etc/pki/ca-trust/source/anchors/`   
  3.2 `sudo update-ca-trust`  to update ca trust list    
  3.3 `sudo service docker restart`

#### on docker-toolbox
1. `openssl s_client -connect examle.com.au:443 -showcerts` and save `cert-1.crt` and `cert-2.crt` (certification chain)
2. the `boot2docker` VM only has access to  your `%HOME%` by default. if you need to add more shared folders, you need to configure as you normally do on vbox (`c:\Workspace`->/c/Workspace)
3. NOTE: `boot2docker` VM doesn't matain state. all you changes are gone after reboot. 
   > https://github.com/boot2docker/boot2docker#installing-secure-registry-certificates


[deprecated] Normally, your trusted CA are stored at `/usr/local/share`, so you need to add the cr file there (verified 👌)   
```bash
sudo -i
# /usr/local won't be persisted, as boot2docker won't persit files (Core Linux, Core OS)
#  /etc/docker/certs.d/example.com.au:8080/ is the correct target
cd /usr/local/share/ca-certificates # cd /etc/docker/certs.d/example.com.au:8080/ 
mkdir artifactory && cd artifactory
cp /c/Workspace/cert-* .
update-ca-certificates
/etc/init.d/docker restart
```

but since `docker-toolbox` runs on memory only os (Core linux), it can't persist your file.   verified👌

```bash
docker-machine scp certfile default:ca.crt # https://github.com/docker/machine/pull/4388
docker-machine ssh default
sudo mv ~/ca.crt /etc/docker/certs.d/examle.com.au:8080/ca.crt
exit
docker-machine restart
```


```
4. docker pull example.com.au:8080/your-image:latest