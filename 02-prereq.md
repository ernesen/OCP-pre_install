Summary: OCP installation - prerequisites
id: lab02-ocp-prereq

# OpenShift Container Platform Installation - Prerequisites

## Introduction
Duration: 00:01:00

During the installation process, a couple of DNS records need to be resolved. Here we will use a simple DNS solution, DNSMasq, to run as a DNS service. We then configure the DNS records for the installation.

We will use software based LB, haproxy, for the LB required for OCP

We use Apache HTTP server to serve the ignition request.

Lastly we will setup and populate the mirrored registry.

## Table of Nodes
duration: 05:00

Create a table for your nodes. A sample is listed as below, Update the hostname and IP based on the cluster assigned to you. 

| Node        | Hostname           | IP  |
| ------------|:-------------:| -----:|
| Bastion    | bastion  .< cluster name >.< domain name > | 192.168.xx.xx |
| Bootstrap    | bootstrap.< cluster name >.< domain name > | 192.168.xx.xx |
| Master 1     | master1.< cluster name >.< domain name > | 192.168.xx.xx |
| Master 2     | master2.< cluster name >.< domain name > | 192.168.xx.xx |
| Master 3     | master3.< cluster name >.< domain name > | 192.168.xx.xx |
| Worker 1     | worker1.< cluster name >.< domain name > | 192.168.xx.xx |
| Worker 2     | worker2.< cluster name >.< domain name > | 192.168.xx.xx |
| Worker 3     | worker3.< cluster name >.< domain name > | 192.168.xx.xx |

## Configure DNS
duration: 15:00

Install the dnsmasq and the dns utils.
```
yum -y install dnsmasq bind-utils
```

Create the following configuration file based on your cluster.

```console
address=/api.< cluster name >.< domain name >/192.168.10.79
address=/api-int.< cluster name >.< domain name >/192.168.10.79
address=/.apps.< cluster name >.< domain name >/192.168.10.79

address=/master1.< cluster name >.< domain name >/192.168.10.11
ptr-record=11.10.168.192.in-addr.arpa,master1.< cluster name >.< domain name >
...

```

In the above sample, we point the api and api-int to the bastion node where the load balancer will be running. The wildcard *.apps is set to the load balancer. 

For each node, create the A record and the PTR record as shown in the above example.

After the file is created, save it under /etc/dnsmasq.d/ folder named as <your cluster name>.conf, restart the service and check the status of the service

```
systemctl restart dnsmasq
systemctl status dnsmasq
```

positive
: Optionally, you can update the bastion nodes' DNS server to point to the DNSmasq server. 
Edit /etc/sysconfig/network-scripts/ifcfg-eth0, update the DNS record to the bastion server itself. Restart the NetworkManager 

Now you validate the records with nslookup or dig. Validate all the hostname of the nodes can be resolved to the respective IP address. On the other hand, all the IPs can be resolved to the respective hostname using the `dig -x` command.


## Configure LB
duration: 15:00

We will haproxy as the LB on the bastion node.

Install haproxy first, 
```
yum -y install haproxy
```

backup the /etc/haproxy/haproxy.cfg. Create a new haproxy.cfg with the following sample content.
```console
global
  log         127.0.0.1 local2
  chroot      /var/lib/haproxy
  pidfile     /var/run/haproxy.pid
  maxconn     4000
  user        haproxy
  group       haproxy
  daemon

  stats socket /var/lib/haproxy/stats

defaults
  mode                     http
  log                      global
  option                   httplog
  option                   dontlognull
  option http-server-close
  # option forwardfor      except 127.0.0.0/8
  option                   redispatch
  retries                  3
  timeout http-request     10s
  timeout queue            1m
  timeout connect          10s
  timeout client           1m
  timeout server           1m
  timeout http-keep-alive  10s
  timeout check            10s
  maxconn                  3000

frontend stats
  bind *:9000
  stats enable
  stats uri /stats
  stats refresh 10s

frontend openshift-api-server
  bind *:6443
  default_backend openshift-api-server
  mode tcp
  option tcplog

backend openshift-api-server
  balance source
  mode tcp
  server dev-47-bootstrap 192.168.10.11:6443 check
  server dev-47-master1 192.168.10.11:6443 check
  server dev-47-master2 192.168.10.12:6443 check
  server dev-47-master3 192.168.10.13:6443 check


frontend machine-config-server
  bind *:22623
  default_backend machine-config-server
  mode tcp
  option tcplog

backend machine-config-server
  balance source
  mode tcp
  server dev-47-bootstrap 192.168.10.11:22623 check
  server dev-47-master1 192.168.10.11:22623 check
  server dev-47-master2 192.168.10.12:22623 check
  server dev-47-master3 192.168.10.13:22623 check


frontend ingress-http
  bind *:80
  default_backend ingress-http
  mode tcp
  option tcplog

backend ingress-http
  balance source
  mode tcp
  server dev-47-master1 192.168.10.11:80 check
  server dev-47-master2 192.168.10.12:80 check
  server dev-47-master3 192.168.10.13:80 check
  server dev-47-worker1 192.168.10.14:80 check
  server dev-47-worker2 192.168.10.15:80 check
  server dev-47-worker3 192.168.10.16:80 check


frontend ingress-https
  bind *:443
  default_backend ingress-https
  mode tcp
  option tcplog

backend ingress-https
  balance source
  mode tcp
  server dev-47-master1 192.168.10.11:443 check
  server dev-47-master2 192.168.10.12:443 check
  server dev-47-master3 192.168.10.13:443 check
  server dev-47-worker1 192.168.10.14:443 check
  server dev-47-worker2 192.168.10.15:443 check
  server dev-47-worker3 192.168.10.16:443 check
```

Here we defined the following LB services:
- open-shift-api-server 6443
- machine-config-server 22623
- ingress-http 80
- ingress-https 443

Each record has a backend and a frontend. frontend defined which TCP port to listen on, the backend defines the servers to serve the request. 

It's noticed during the installation, the bootstrap server is used for the API server and machine config server. After the cluster is up, the bootstrap backend can be removed.

Update the config based on your cluster's configuration. 

Restart the haproxy, and validate the service status.

```
systemctl restart haproxy
systemctl status haproxy
```

## Configure of HTTP server
duration: 5:00

Install Apache http server
```
yum -y install httpd
systemctl enable httpd
systemctl start httpd
systemctl status httpd
```

Validate the wwwroot is set to /var/www/html,
```
grep DocumentRoot /etc/httpd/conf/httpd.conf
```

Create a test test file, validate you can get the content correctly through the HTTP server.
```
echo 123 > /var/www/html/abc.txt
curl http://localhost/abc.txt
rm /var/www/html/abc.txt
```

## Create Local Registry
duration: 20:00

### Create Certificate for Registry
First let's create the certificate required for the registry.

Copy the cfssl tools from the lab-files folder
```
cp cfssl_1.5.0_linux_amd64 /usr/local/bin/cfssl
cp cfssljson_1.5.0_linux_amd64 /usr/local/bin/cfssljson
chmod a+rx /usr/local/bin/cfssl
chmod a+rx /usr/local/bin/cfssljson
```

Switch to the normal user as you created in the first lab,
```
su - user1
```

Create the following json file to generate the CA cert, save it in the directory certs/myca.json
```json
{
   "CN": "ca",
   "hosts": [
      "ca"
   ],
   "key": {
      "algo": "rsa",
      "size": 2048
   },
   "names": [
      {
         "C": "SG",
         "ST": "SG",
         "L": "Singapore"
      }
   ]
}
```

Create self signed CA cert,
```
cd certs
cfssl gencert -initca myca.json | cfssljson -bare myca
```

Lets add this CA cert the system trusted certs. Switch to root
```
cp /home/user1/certs/myca.pem /etc/pki/ca-trust/source/anchor/myca.crt
update-ca-trust
```

Switch back to the normal user.

Create cert for registry server. Create a json file as certs/serverRequest.json,
```json
{
   "CN": "registry",
   "hosts": [ "registry","bastion.ocp4.ibmcloud.io.cpak" ],
   "key": {
      "algo": "rsa",
      "size": 2048
   }
}
```
Notice the hosts is the list of the hostname you want to add in the cert SAN extensions. Modify it based on your registry's hostname.

```
cfssl gencert -ca=myca.pem -ca-key=myca-key.pem serverRequest.json | cfssljson -bare registry
```

ls the directory, you should see file `registry.pem` and `registry-key.pem` created, which are the cert and the key files respectively.

### Create htpasswd authentication

Install htpasswd package with root account or sudo privilege. 
```
sudo yum install -y httpd-tools
```
You may already have the package already installed. 

Switch to normal user, create the following passwd file,
```
htpasswd -bBc htpasswd admin password
```
It will create a encrypted "password" for "admin" in the file named "htpasswd" in the current directory

### Load Registry Container Image

Load the docker registry image from lab-files into the container storage with `skopeo`
```console
skopeo copy dir:///data/lab-files/registry-images containers-storage:registry
```

Validate you have the images "registry" available with `podman images`.

### Run Registry Container

Choose a large enough filesystem to host the images (> 50GB). Create a directory. For example, `/data/registry/`. 
_The following example will use `/data/registry` as an example, change accordingly if you need_

Assign the proper ownership for this directory if needed such as,
```
chown user1:user1 /data/registry
```

Switch to normal user if have not.
```
cd /data/registry
mkdir data auth certs
```

Copy the certs, htpasswd generated in the above steps to the respective directory created.
```
cp $HOME/htpasswd /data/registry/auth
cp $HOME/certs/registry.pem /data/registry/certs
cp $HOME/certs/registry-key.pem /data/registry/certs
```

Now run the container,
```
podman run -d --name mirror-registry -p 5000:5000 \
  -v /data/registry/data:/var/lib/registry:z \
  -v /data/registry/auth:/auth:z \
  -v /data/registry/certs:/certs:z \
  -e REGISTRY_AUTH=htpasswd \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.pem \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry-key.pem \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  registry
```

### Validate the Registry
Use the podman to login to the local registry, `podman login <hostname>:5000 -u <user> -p <password>`

You should able to see "Login Succeeded!"

We can also validate the content by running a curl call, replace with your actual username, password, and hostname.
```
curl -u admin:password https://{{ .hostname }}:5000/v2/_catalog 
```

You should see a empty list as the registry now is empty
```
{"repositories":[]}
```

