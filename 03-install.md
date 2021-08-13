Summary: OCP installation
id: lab03-ocp-install

# OpenShift Container Platform Installation

## Introduction
Duration: 00:01:00

We will install the OpenShift Container Platform 4.7 in the lab environment. 

We will load the mirrored images, create the installation configure, prepare the RHCOS OS, and start the installation.

## Mirror the image from internet
Duration: 00:05:00

The mirror part is ignored here. You just need copy the download images directory `lab-files/ocpImages` to your registry server, which is the bastion node in the lab environment. 

The steps can be referred to the OCP document, https://docs.openshift.com/container-platform/4.7/installing/install_config/installing-restricted-networks-preparations.html.

## Extract the installer
Duration: 00:05:00

Extract the OpenShift command line tools from lab-files
```
cd /data/labFiles/ocp
tar zxvf openshift-client-linux.tar.gz
mv oc kubectl /usr/local/bin

tar zxvf openshift-install-linux.tar.gz
mv openshift-install /usr/local/bin

tar zxvf opm-linux.tar.gz
mv opm /usr/local/bin
```

## Create secret JSON
Duration: 05:00

Create the following `secret.json` file,
```json
{
  "auths": {
    "{{ .fqdn }}:5000": {
      "auth": "{{ .base64 }}",
      "email": "anyone@xyz.com"
    }
  }
}
```

Replace {{ .fqdn }} with your fullly qualified domain name of the bastion node, such as bootstrap.<cluster name>.<domain name>.

Get the {{ .base64 }} with encoded username and password pair by running,
```
echo -n "admin:password" | base64 -w0; echo
YWRtaW46cGFzc3dvcmQ=
```
Replace {{ .base64}} with the value obtained, `YWRtaW46cGFzc3dvcmQ=`

## Load image into registry
Duration: 10:00

Run the following command,

```console
export OCP_RELEASE=4.7.5 
export LOCAL_REGISTRY='{{ .fqdn }}:5000' 
export LOCAL_REPOSITORY='ocp4/openshift4' 
export LOCAL_SECRET_JSON='/home/user1/secret.json' 
oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=/data/lab-files/ocpImages "file://openshift/release:${OCP_RELEASE}*" ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}
```

Replace {{ .fqdn }} with your fullly qualified domain name of the bastion node.

Change "/data/lab-files/ocpImages" if your images is located in a different directory.

Validate the images are loaded by running,
```console
curl -s -u admin:password https://{{ .fqdn }}:5000/v2/ocp4/openshift4/tags/list  | jq '.tags'
```

It will return a list of OCP images.

Positive
: Install jq to parse and view the json content by `yum -y install jq` with root.

## Create installation configuration file
Duration: 10:00

Positive
: You may choose to use root or normal user. Use that user for the rest of the tasks.

Create SSH keys for bastion node to access the OCP cluster nodes.
```
cd
ssh-keygen -b 4096 -t rsa -f .ssh/id_rsa -N "" 
```

Use the following as a base, modify it based on your environment.

```
apiVersion: v1
baseDomain: ibmcloud.io.cpak
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}

imageContentSources:
- mirrors:
  - bastion.ocp4.ibmcloud.io.cpak:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - bastion.ocp4.ibmcloud.io.cpak:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
additionalTrustBundle: |
    -----BEGIN CERTIFICATE-----
    MIIDRjCCAi6gAwIBAgIUfXKWV+5dl4PYpJWq0NESF8OfkrIwDQYJKoZIhvcNAQEL
    BQAwOzELMAkGA1UEBhMCU0cxCzAJBgNVBAgTAlNHMRIwEAYDVQQHEwlTaW5nYXBv
    cmUxCzAJBgNVBAMTAmNhMB4XDTIxMDQwNTA4NDYwMFoXDTI2MDQwNDA4NDYwMFow
    OzELMAkGA1UEBhMCU0cxCzAJBgNVBAgTAlNHMRIwEAYDVQQHEwlTaW5nYXBvcmUx
    CzAJBgNVBAMTAmNhMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAt+Rp
    7Oc/f1H9kMBWpphx9pDmMyDo0Ux9O3sqCpMqssFs7FtdiNi0JyHKSpoF75nhEDqC
    nfxBTDlpN0LjnlzSr+6FG0zyfJGe8/kAHFEuCPuAtuZlJty3xbkja1m7ful62YMn
    IzNTEvRnpEOiKTT6pxzQjom//nVUbg0Vud+i7AXiGce0mJnPcLhdokIzHBgtaELI
    mPKAXN9ufjy9yKow+GXHhiuZEH5MXk9yaJAanZC8Yag9qZz0Xc/hrb0QdW/nfUsv
    GtyeAl1KDgNEzDdcCs0cRHXJR0bm92QtwxhfBiuatbLWD94bnzKh/FVAQ+4ujsxs
    5LbslbpcPFnBC8aApwIDAQABo0IwQDAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/
    BAUwAwEB/zAdBgNVHQ4EFgQUNrUGPWwLoB6zpttB1etUVCM6OXQwDQYJKoZIhvcN
    AQELBQADggEBAJI+mVX/e6BsYQZTQIkIxWQEkqITBmC0E9xIYqtSf9kUto6ZQmFG
    k1eTLKQwk6VMCX1JsKMb56AeBeZe6q726JSLxJYSnB2mL4J/DnADiTdY5d+vqKCS
    7bhqa4wMvYWbDO/e3/FxQvGFE12F/V65mvvJBok/6WrRCDP0tCZLOxon4f8EGU8W
    D4UOuouEafQAdPif3WKmkXetpkugRfFNkGF36pB8hfhYqLpdUJsPfEoM6GqjKVa7
    zbRDb6EY5BIiuqCdDZn7fMF8K7Yx9PI0KAUbYH+VpHwG+JeTDA/nXJ2epPg3p9C8
    ESxGEcfShsXouEG/5B4Mvdr5TXxVzbFxTnE=
    -----END CERTIFICATE-----


pullSecret: '{
  "auths": {
    "bastion.ocp4.ibmcloud.io.cpak:5000": {
      "auth": "YWRtaW46cGFzc3dvcmQ=",
      "email": "wenzm@sg.ibm.com"
    }
  }
}
'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDQMu2otYZqNZud83cEqAvJRwSc28GHGx/NSdEBPmCjeIV1VEJb0bi/VMwWZLn079jXbt8HHyea+1JtGT53t+pnapxNxD8K39pSgyhbkdiNG1gNHXkG2yPunmfkep1twuWNOFr0y3cFoKelcp89rd2zsKE6oPESMS8MgVaFh8uNR2n/mV2j6dQm4u1d/B0hZDvNrzLEhQ+o50aYlT/DBRw3OQ/sjdfUGND/SNFvTRNhABBW3dV2QhmZmXCoWePTzyMBpxsWrxepewXJHnXUPQAebGYg8zD3HEpNziZ63ro4QavPji1JS+c0PXUdxiDtbldCVublLul4iW9KANItBAfppr3ranOQTXoMiQe9SBVkIF2kX0Ped93rSIj7co75eCy6uBE/JNCJE0FlOjsKjIB4ZTicAbpsm4KDFWsYRYjACgzUKYY/ZAPRu9GQtciTHHwaWtcO5nH8flc0qIjL4yjh/I7fFjZTNEo5m2IHgSIGyE7n+IIcSfhWUM/Hwmvcte0tkpFIabsl7uZ0FlENunjXEB3EYdEpShFlAoJaCHMJIee9WiF/lpd6RdZMvOARwv9SvQCaGUUdVtdZ+/NQfXbQ66HG/gOwIZYAqHkSoW+ffQ+ZmoSICIknLy6/B8PQzyN7LmDmhA4936s+t2AcnX5SV3vpIQc7M3CxVaYqPZKQWw== root@rhel8'


```

## Create ignition files
Duration: 05:00

```
mkdir ocp-install
cp install-config.yaml ocp-install
cd ocp-install
openshift-install create ignition-configs --dir .
```

Positive
: You may want to keep the install-config.yaml file outside of a working directory in case you need to recreate the ignitions. Because the yaml file will be removed in the directory after ignition files are generated.

Copy the *.ign files to the web server.
```
cp *ign /var/www/html/
chmod a+r /var/www/html/*ign
```

Test you can get the ign file through the web server
```
curl http://{{ .fqdn }}/bootstrap.ign
```

## Bootup the cluster node - bootstrap
Duration: 20:00

Inject the ISO image file in the lab-files/ocp folder to the VM. Bootup the VM, after the 4.6 release, the live CD will boot up into a shell, from which we can install the coreos on to the harddisk.

On the shell prompt, set the static IP address, DNS, and hostname.

Get the connection name first,
```
sudo nmcli co show
```

Then
```
sudo nmcli con mod "Wired connection 1" ipv4.addresses "192.168.10.102/24"
sudo nmcli con mod "Wired connection 1" ipv4.gateway "192.168.10.1"
sudo nmcli con mod "Wired connection 1" ipv4.dns "192.168.10.101"
sudo nmcli con mod "Wired connection 1" ipv4.method manual
sudo nmcli con mod "Wired connection 1" connection.autoconnect yes
sudo nmcli con up "Wired connection 1"
```

Validate the gateway can be reached. DNS name can be resolved. Ignation file can be retrieved.

```
ping 192.168.10.1
dig bootstrap.ocp4.ibmcloud.io.cpak
curl http://bastion.ocp4.ibmcloud.io.cpak:8080/bootstrap.ign
```

Find out the 1st disk's name by running `lsblk`. In my case it is vda. Yours may change.

Now intall the Coreos with the following command
```console
sudo coreos-installer install -n --insecure-ignition \
     -I http://bastion.ocp4.ibmcloud.io.cpak:8080/bootstrap.ign \
     /dev/vda
```

-n stands for --copy-network, -I stands for --ignation-url
You can run `coreos-intaller intall --help` for the options.

After the installation complete, run `sudo reboot` to restart the node.

Validate the node is up.  ssh into the node with the user whose ssh key is used for authentication
```
ssh core@192.168.10.102
```

Check the ip address, hostname are set properly.
Check the bootkube log,
```
journalctl -b -f -u release-image.service -u bootkube.service
```

Validate the containers are running,
```
sudo -i
crictl ps
```

## Bootup the cluster node - masters and workers
Duration: 40:00

Perform the same steps for the masters and workers respectively.

Note the workers may not be fully up when rebooted. It will be up till the masters are ready.


## Install OCP
Duration: 20:00

When the masters are fully up. Run the installation. 

On the bastion node, 
```
cd ocp-install
openshift-install --dir=. wait-for bootstrap-complete --log-level=debug
```

This may take 10 - 30 minutes.

## Approve workers node CSR
Duration: 10:00

Check node status 
```
export KUBECONFIG=$HOME/ocp-install/auth/kubeconfig
oc get nodes
```

You may only see masters node. 

Approve any pending CSR.
```
export KUBECONFIG=$HOME/ocp-install/auth/kubeconfig
oc get csr | grep Pending | awk '{print $1}'| xargs oc adm certificate approve
```

Positive
: You will need to run the above steps several times until all the worker nodes are ready.


## Wait for cluster operator installed - installation complete
Duration: 30:00

When the bootstrap is done, 
```
export KUBECONFIG=$HOME/ocp-install/auth/kubeconfig
oc patch configs.imageregistry.operator.openshift.io cluster --type json  --patch '[{"op": "replace", "path": "/spec/storage", "value": { "emptyDir": {}  }}]'
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"defaultRoute":true}}'
```

If the response is object not found, wait a while and retry the oc patch command.

Run the following command to complete the cluster operators.
```
cd ocp-install
openshift-install --dir=. wait-for install-complete  --log-level=debug
```

You can check the progress in detail by opening another shell using the same user,
```
export KUBECONFIG=$HOME/ocp-install/auth/kubeconfig
oc get co
```
Validate all the cluster operators are completed.

Copy the URL and password listed in the install log. We will use that for OCP web console.


## Login to OCP console
Duration: 05:00

Launch your browser to access the OCP Web console.

If you are using Windows, you may need to update the etc hosts file to point all the required hostnames to the LB node. For example,

```
192.168.10.101 console-openshift-console.apps.dev-airgap47.ibmcloud.io.cpak oauth-openshift.apps.dev-airgap47.ibmcloud.io.cpak
```

## Remove bootstrap node
Duration: 05:00

Now you can stop the bootstrap node and remove it to recycle the resource.

Update the LB configuration, `/etc/haproxy/haproxy.cfg`,  remove the bootstrap node from the backend list. Restart the LB.
```
systemctl restart haproxy
```


