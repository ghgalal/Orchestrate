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
ssh itzuser@api.679d326900f27b33a46a02b5.ocp.techzone.ibm.com -p 40222
```

Setup podman
```
sudo yum install -y podman
```

setup cpd-cli
```
#cpd-cli /home/itzuser/cpd-cli-linux-EE-14.1.0-1189:
sudo su

wget https://github.com/IBM/cpd-cli/releases/download/v14.1.0/cpd-cli-linux-EE-14.1.0.tgz
tar -xzf cpd-cli-linux-EE-14.1.0.tgz
```

Get the path of cpd-cli
```
pwd
```

edit ~/.bashrc and add the path of cpd-cli
- vi ~/.bachrc
- insert the path the came out of pwd at the end of the file
  export PATH=<cpd-cli-path>:$PATH
- save and close using :wq
- source ~/.bashrc

If don't want to do that, you can export the path everytime you ssh to the bastion

#export PATH="$(pwd)/cpd-cli-linux-EE-14.1.0-1189":$PATH

Restart the olm-utils container
```
cpd-cli manage restart-container
```

oc is already installed on the bastion, but in case you are using a different host, use the following steps to setup oc
```
export OPENSHIFT_BASE_DOMAIN=679159d7e4b091eaa609cf5b.ocp.techzone.ibm.com
wget --no-check-certificate https://downloads-openshift-console.apps.${OPENSHIFT_BASE_DOMAIN}/amd64/linux/oc.tar
tar -xvf oc.tar
chmod +x oc
sudo mv oc /usr/local/bin/oc
```


Prepare your [cpd_vars.sh](https://github.com/ghgalal/Orchestrate/blob/main/cpd_vars.sh)
```
vi cpd_vars.sh
```
```
bash ./cpd_vars.sh
chmod 700 ./cpd_vars.sh
source ./cpd_vars.sh
```
Login to oc and cpd-cli
```
${OC_LOGIN}
```
```
${CPDM_OC_LOGIN}
```
## Install Redhat certmanager
Follow the step to get it installed
- go to openshift console
- go to operator hub
- find redhat certmanager
- get it installed
- make sure you select version 1.13 because the latest version doesn't work with Orchestrate

Check if it is installed and working
```
oc get subscription -n cert-manager-operator
```
```
oc get csv -n cert-manager-operator
```
```
oc get pods -n cert-manager
```

## Prepare the Cluster
### Update the global pull image secret
```
cpd-cli manage add-icr-cred-to-global-pull-secret \ --entitled_registry_key=${IBM_ENTITLEMENT_KEY}
```

Wait until it is updated and set to True
```
oc get mcp
```
### Create the namespaces
```
$OC_LOGIN}
oc new-project ${PROJECT_LICENSE_SERVICE}
oc new-project ${PROJECT_SCHEDULING_SERVICE}
oc new-project ${PROJECT_CPD_INST_OPERATORS}
oc new-project ${PROJECT_CPD_INST_OPERANDS}
```

### Install the shared cluster components
Installing Foundational Services, Licensing service and scheduling service

- Licensing Service
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

- Scheduler
```
cpd-cli manage apply-scheduler \
--release=${VERSION} \
--license_acceptance=true \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE}
```

### Persistent storage
Since we provisioned the cluster with ODF, we just need to confirm that the storage is configured
```
oc get sc
```

```
oc get pvc -A
```

### Change the clsuter limits
Confirm there is no kubeletconfig running
```
oc get kubeletconfig
```

In case you get an empty response, apply the following config
```
oc apply -f - << EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: cpd-kubeletconfig
spec:
  kubeletConfig:
    podPidsLimit: 16384
  machineConfigPoolSelector:
    matchExpressions:
    - key: pools.operator.machineconfiguration.openshift.io/worker
      operator: Exists
EOF
```

In case there is a kubeletconfig already running, change the values
- get the name and export it
```
export KUBELET_CONFIG=<kubeletconfig-name>
```
- apply the change
```
oc patch kubeletconfig ${KUBELET_CONFIG} \
--type=merge \
--patch='{"spec":{"kubeletConfig":{"podPidsLimit":16384}}}'
```

Make sure kubletconfig is now running
```
oc get kubeletconfig
```

## Install the node feature discovery
- Go to the operator hub from the console
- search for node feature discovery and install the Redhat operator, not the community operator
- Install the operator
- Wait for it to become ready

## Install the NVIDIA GPU Operator
Note that we don't need this as there are no GPUs in the Techzone environment, but we installed it anyway

- Go to the operator hub from the console
- search for NVIDIA GPU Operator
- Install the operator
- Wait for it to become ready

## Install Redhat Openshift AI
```
${OC_LOGIN}

oc new-project redhat-ods-operator
```

- Install the operator group
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
EOF
```

- Install the operator
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  name: rhods-operator
  channel: stable-2.13
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  config:
     env:
        - name: "DISABLE_DSC_CONFIG"
EOF
```

wait until the operator is installed

- Create a Dev Space CLI
```
cat <<EOF |oc apply -f -
apiVersion: dscinitialization.opendatahub.io/v1
kind: DSCInitialization
metadata:
  name: default-dsci
spec:
  applicationsNamespace: redhat-ods-applications
  monitoring:
    managementState: Managed
    namespace: redhat-ods-monitoring
  serviceMesh:
    managementState: Removed
  trustedCABundle:
    managementState: Managed
    customCABundle: ""
EOF
```

- Confirm that the status of the following pods in the redhat-ods-applications project are Running: • kserve-controller-manager-* pod • kubeflow-training-operator-* pod • odh-model-controller-* pod
```
oc get dscinitialization
```

```
cat <<EOF |oc apply -f -
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    codeflare:
      managementState: Removed
    dashboard:
      managementState: Removed
    datasciencepipelines:
      managementState: Removed
    kserve:
      managementState: Managed
      defaultDeploymentMode: RawDeployment
      serving:
        managementState: Removed
        name: knative-serving
    kueue:
      managementState: Removed
    modelmeshserving:
      managementState: Removed
    ray:
      managementState: Removed
    trainingoperator:
      managementState: Managed
    trustyai:
      managementState: Removed
    workbenches:
      managementState: Removed
EOF
```

- Confirm the pods are running 
```
oc get pods -n redhat-ods-applications
```

Edit the inferenceservice-config configuration map in the redhat-ods-applications project:

-Log in to the Red Hat OpenShift Container Platform web console as a cluster administrator.
-From the navigation menu, select Workloads > Configmaps.
-From the Project list, select redhat-ods-applications.
-Click the inferenceservice-config resource. Then, open the YAML tab.
-In the metadata.annotations section of the file, add opendatahub.io/managed: 'false':
```
metadata:
annotations:
  internal.config.kubernetes.io/previousKinds: ConfigMap
  internal.config.kubernetes.io/previousNames: inferenceservice-config
  internal.config.kubernetes.io/previousNamespaces: opendatahub
  opendatahub.io/managed: 'false'
```

-Find the following entry in the file:
```
"domainTemplate": "{{ .Name }}-{{ .Namespace }}.{{ .IngressDomain }}",
```
and update the value of the domainTemplate field to "example.com":
```
"domainTemplate": "example.com",
```

- Click Save.
