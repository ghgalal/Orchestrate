# Watson Orchestrate 5.1
## Techzone environment
Reserve from techzone using this link

TechZone OpenShift VM (for installation)
Https://techzone.ibm.com/my/reservations/create/63a3a25a3a4689001740dbb3

The product will be installed on 5 worker nodes on openshift 4.16 (despite the documentation saying it is supported starting openshift 4.17)
Make sure you select ODF - 2TB as your storage

![image](https://github.com/user-attachments/assets/0f14af27-cc45-4151-8087-4ad5741ef4a1)

## Prepare your Bastion
ssh into your bastion. Use the details provided in your provisioned environment
```
ssh itzuser@<host> -p <port>
```

Setup oc
```
export OPENSHIFT_BASE_DOMAIN=679159d7e4b091eaa609cf5b.ocp.techzone.ibm.com
wget --no-check-certificate https://downloads-openshift-console.apps.${OPENSHIFT_BASE_DOMAIN}/amd64/linux/oc.tar
tar -xvf oc.tar
chmod +x oc
sudo mv oc /usr/local/bin/oc
```

Setup podman
```
sudo yum install -y podman
```
setup cpd-cli
```
#cpd-cli /home/itzuser/cpd-cli-linux-EE-14.1.0-1189:
sudo bash

wget https://github.com/IBM/cpd-cli/releases/download/v14.1.0/cpd-cli-linux-EE-14.1.0.tgz
tar -xzf cpd-cli-linux-EE-14.1.0.tgz
export PATH="$(pwd)/cpd-cli-linux-EE-14.1.0-1189":$PATH

cpd-cli manage restart-container
```

Prepare your cpd_vars.sh
```
vi [cpd_vars.sh](https://github.com/ghgalal/Orchestrate/blob/main/cpd_vars.sh)
```

