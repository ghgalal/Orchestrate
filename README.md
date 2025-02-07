# Watson Orchestrate 5.1
Prepared by: 

Gihad Abdelhamid

Zak Al Hashash


## Orchestrate Architecture
![image](https://github.com/user-attachments/assets/4505cdd2-0f38-4e3c-a2fd-f9a29474b033)

![image](https://github.com/user-attachments/assets/036ba2f9-b0f2-42c7-9f89-f60231fa7bf7)


## Techzone environment
Reserve from techzone using this link

TechZone OpenShift VM (for installation)
Https://techzone.ibm.com/my/reservations/create/63a3a25a3a4689001740dbb3

The product will be installed on 5 worker nodes on openshift 4.16 (despite the documentation saying it is supported starting openshift 4.17)
Make sure you select ODF - 2TB as your storage

![image](https://github.com/user-attachments/assets/0f14af27-cc45-4151-8087-4ad5741ef4a1)

## Link to the CPD documentation in IBM Docs
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=getting-started

## Prepare your Bastion
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=installing-setting-up-client-workstation

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
```
```
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
https://docs.openshift.com/container-platform/4.16/security/cert_manager_operator/cert-manager-operator-install.html

Follow the step to get it installed
- go to openshift console
- go to operator hub
- find cert-manager Operator for Red Hat OpenShift
- get it installed
- make sure you select version 1.13 because the latest version doesn't work with Orchestrate and don't install in all namespaces

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
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cluster-updating-global-image-pull-secret

```
cpd-cli manage add-icr-cred-to-global-pull-secret --entitled_registry_key=${IBM_ENTITLEMENT_KEY}
```

Wait until it is updated and set to True
```
oc get mcp
```
### Create the namespaces
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cluster-manually-creating-projects-namespaces-shared-components

```
$OC_LOGIN}
oc new-project ${PROJECT_LICENSE_SERVICE}
oc new-project ${PROJECT_SCHEDULING_SERVICE}
oc new-project ${PROJECT_CPD_INST_OPERATORS}
oc new-project ${PROJECT_CPD_INST_OPERANDS}
```

### Install the shared cluster components
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cluster-installing-shared-components

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
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cluster-configuring-persistent-storage

Since we provisioned the cluster with ODF, we just need to confirm that the storage is configured
```
oc get sc
```

```
oc get pvc -A
```

### Change the clsuter limits
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=settings-changing-process-ids-limit

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

### Install the node feature discovery
- Go to the operator hub from the console
- search for Node Feature Discovery Operator and install the Redhat operator, not the community operator
- Install the operator
- Wait for it to become ready

### Install the NVIDIA GPU Operator
Note that we don't need this as there are no GPUs in the Techzone environment, but we installed it anyway

- Go to the operator hub from the console
- search for NVIDIA GPU Operator
- Install the operator
- Wait for it to become ready

### Install Redhat Openshift AI
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=software-installing-red-hat-openshift-ai

```
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

- Confirm that the data science cluster is ready
```
oc get datasciencecluster default-dsc -o jsonpath='"{.status.phase}"{"\n"}'
```

- Confirm the pods are running
```
oc get pods -n redhat-ods-applications
```

Edit the inferenceservice-config ConfigMap in the redhat-ods-applications project:
- Log in to the Red Hat OpenShift Container Platform web console as a cluster administrator.
- From the navigation menu, select Workloads > Configmaps.
- From the Project list, select redhat-ods-applications.
- Click the inferenceservice-config resource. Then, open the YAML tab.
- In the metadata.annotations section of the file, add opendatahub.io/managed: 'false':
```
metadata:
annotations:
  internal.config.kubernetes.io/previousKinds: ConfigMap
  internal.config.kubernetes.io/previousNames: inferenceservice-config
  internal.config.kubernetes.io/previousNamespaces: opendatahub
  opendatahub.io/managed: 'false'
```

- Find the following entry in the file:
```
"domainTemplate": "{{ .Name }}-{{ .Namespace }}.{{ .IngressDomain }}",
```
and update the value of the domainTemplate field to "example.com":
```
"domainTemplate": "example.com",
```

- Click Save.


### Install the knative service
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=software-installing-red-hat-openshift-serverless-knative-eventing

knative installation will fail and we will need to manually get into the container and remove one line. 
After one of our colleagues consulted with the development team, the "kafka-broker-dispatcher" will always be waiting for another object to be ready, but the pods are brought up by a statefulset, so it will never complete. should be fixed in the next update for Orchestrate

Folllow the steps below
- find the olm-utils container id
```
podman ps
```
- get into the container
```
podman exec -it <container-id> /bin/bash
```
- go to "bin/deploy-knative-eventing" and remove the line "oc wait deployment -n knative-eventing kafka-broker-dispatcher --for condition=Available=True --timeout=60s". you can use the vi search capability by typing / and then type the search string and press enter, it should take you to the line that is supposed to be removed


Install the service
```
cpd-cli manage deploy-knative-eventing \
--release=${VERSION} \
--block_storage_class=${STG_CLASS_BLOCK} \
--patch_redhat_crd=false
```

Check the knative-eventing is working now
```
oc get all -n knative-eventing
```

### Install Appconnect
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=software-installing-app-connect

- Download the Case Files
```
curl -sSLO https://github.com/IBM/cloud-pak/raw/master/repo/case/ibm-appconnect/${AC_CASE_VERSION}/ibm-appconnect-${AC_CASE_VERSION}.tgz

tar -xf ibm-appconnect-${AC_CASE_VERSION}.tgz
```

- create project
```
oc new-project ${PROJECT_IBM_APP_CONNECT}
```

```
oc patch \
--filename=ibm-appconnect/inventory/ibmAppconnect/files/op-olm/catalog_source.yaml \
--type=merge \
-o yaml \
--patch="{\"metadata\":{\"namespace\":\"${PROJECT_IBM_APP_CONNECT}\"}}" \
--dry-run=client \
| oc apply -n ${PROJECT_IBM_APP_CONNECT} -f -
```

- Create operator group
```
cat <<EOF | oc apply -f -
  apiVersion: operators.coreos.com/v1
  kind: OperatorGroup
  metadata:
    name: appconnect-og
    namespace: ${PROJECT_IBM_APP_CONNECT}
  spec:
    targetNamespaces:
    - ${PROJECT_IBM_APP_CONNECT}
    upgradeStrategy: Default
EOF
```

- Create subscription
```
cat <<EOF | oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: ibm-appconnect-operator
    namespace: ${PROJECT_IBM_APP_CONNECT}
  spec:
    channel: ${AC_CHANNEL_VERSION}
    config:
      env:
      - name: ACECC_ENABLE_PUBLIC_API
        value: "true"
    installPlanApproval: Automatic
    name: ibm-appconnect
    source: appconnect-operator-catalogsource
    sourceNamespace: ${PROJECT_IBM_APP_CONNECT}
EOF
```

- Wait for operator to be ready
```
oc wait csv \
--namespace=${PROJECT_IBM_APP_CONNECT} \
--selector=operators.coreos.com/ibm-appconnect.${PROJECT_IBM_APP_CONNECT}='' \
--for='jsonpath={.status.phase}'=Succeeded
```

### Multicloud Onject Gateway secrets config
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=piish-creating-secrets-services-that-use-multicloud-object-gateway

- Check noobaa-admin and noobaa-s3-servicing-cert secrets
```
oc get secrets --namespace=openshift-storage |grep nooba
```
```
export NOOBAA_ACCOUNT_CREDENTIALS_SECRET=noobaa-admin
export NOOBAA_ACCOUNT_CERTIFICATE_SECRET=noobaa-s3-serving-cert
```
- Configure assistant
```
cpd-cli manage setup-mcg \
--components=watson_assistant \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--noobaa_account_secret=${NOOBAA_ACCOUNT_CREDENTIALS_SECRET} \
--noobaa_cert_secret=${NOOBAA_ACCOUNT_CERTIFICATE_SECRET}
```

- configure orchestrate
```
cpd-cli manage setup-mcg \
--components=watsonx_orchestrate \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--noobaa_account_secret=${NOOBAA_ACCOUNT_CREDENTIALS_SECRET} \
--noobaa_cert_secret=${NOOBAA_ACCOUNT_CERTIFICATE_SECRET}
```

- Check if they are created
```
oc get secrets --namespace=${PROJECT_CPD_INST_OPERANDS} \
noobaa-account-watson-assistant \
noobaa-cert-watson-assistant \
noobaa-uri-watson-assistant
```
```
oc get secrets --namespace=${PROJECT_CPD_INST_OPERANDS} \
noobaa-account-watsonx-orchestrate \
noobaa-cert-watsonx-orchestrate \
noobaa-uri-watsonx-orchestrate
```

### Apply permissions to the projects
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=arppn-applying-required-permissions-by-running-authorize-instance-topology-command

```
cpd-cli manage authorize-instance-topology \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

## Install the platform
### Installation of IBM Software Hub
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=hub-installing-software

This step will take around an hour

```
cpd-cli manage setup-instance \
--release=${VERSION} \
--license_acceptance=true \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--run_storage_tests=false
```

- Check the status
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

### Instance Details
- Get the sofware hub login details
```
cpd-cli manage get-cpd-instance-details --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --get_admin_initial_credentials=true
```

### Apply entitlements
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=entitlements-applying-your-without-node-pinning

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise \
--production=false
```

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-orchestrate \
--production=false
```

### Install the components
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=center-installing-software-hub-control-software

```
cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=${COMPONENTS_CR} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true
```

### Install Appconnect
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=software-installing-app-connect

```
cpd-cli manage setup-appconnect \
--appconnect_ns=${PROJECT_IBM_APP_CONNECT} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--release=${VERSION} \
--components=watsonx_orchestrate \
--file_storage_class=${STG_CLASS_FILE}
```

### Install Orchestrate
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=services-watsonx-orchestrate

- Apply OLM
```
cpd-cli manage apply-olm --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --components=watsonx_orchestrate
```

- Apply CR. This step takes around 90 minutes
```
cpd-cli manage apply-cr --components=watsonx_orchestrate --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true
```

