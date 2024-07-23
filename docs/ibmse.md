# IBM SE

## OpenShift deployment

The following instructions are assuming an OpenShift cluster is running.

### Catalog creation

```bash
oc apply -f - << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
 name:  trustee-operator-catalog
 namespace: openshift-marketplace
spec:
 displayName: Trustee Operator Catalog
 sourceType: grpc
 image:  quay.io/rh_ee_lmilleri/trustee-operator-catalog:v0.1.0-ibmse
 updateStrategy:
   registryPoll:
      interval: 5m
EOF
```

### Namespace creation

This is the default Namespace, where all the relevant Trustee objects will be created.

```bash
oc apply -f - << EOF
apiVersion: v1
kind: Namespace
metadata:
  name: kbs-operator-system
EOF
```

### Operator Group

An Operator group, defined by the OperatorGroup resource, provides multi-tenant configuration to OLM-installed Operators:

```bash
oc apply -f - << EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kbs-operator-system
  namespace: kbs-operator-system
spec:
EOF
```

### Subscription

A subscription, defined by a Subscription object, represents an intention to install an Operator. It is the custom resource that relates an Operator to a catalog source:

```bash
oc apply -f - << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kbs-operator-system
  namespace: kbs-operator-system
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: trustee-operator
  source: trustee-operator-catalog
  sourceNamespace: openshift-marketplace
  startingCSV: trustee-operator.v0.1.0
EOF
```

### Check Trustee Operator installation

Now it is time to check if the Trustee operator has been installed properly, by running the command:

```bash
oc get csv -n kbs-operator-system
```

We should expect something like:

```bash
NAME                      DISPLAY            VERSION   REPLACES   PHASE
trustee-operator.v0.1.0   Trustee Operator   0.1.0                Succeeded
```

## Configuration

The Trustee Operator configuration requires a few steps. Some of the steps are provided as an example, but you may want to customize the examples for your real requirements.

### Authorization key-pair generation

First of all, we’d need to create the key pairs for Trustee authorization. The public key is used by Trustee for client authorization, the private key is used by the client to prove its identity and register keys/secrets.


Create secret for client authorization:

```bash
openssl genpkey -algorithm ed25519 > privateKey
openssl pkey -in privateKey -pubout -out publicKey
oc create secret generic kbs-auth-public-key --from-file=publicKey -n kbs-operator-system
```

### Create secrets

How to create secrets to be shared with the attested clients?
In this example we create a secret *kbsres1* with two entries. These resources (key1, key2) can be retrieved by the Trustee clients.
You can add more secrets as per your requirements.

```bash
oc create secret generic kbsres1 --from-literal key1=res1val1 --from-literal key2=res1val2 -n kbs-operator-system
```

### Create IBM SE Volume

## Prepare certificates/keys
Here, the instructions for building an OpenShift volume based on a local directory.
This is for development/test purposes.

The directory should look like the following:

```
├── certs
│   ├── ibm-z-host-key-signing-gen2.crt
|   └── DigiCertCA.crt
├── crls
│   └── ibm-z-host-key-gen2.crl
│   └── DigiCertTrustedRootG4.crl
│   └── DigiCertTrustedG4CodeSigningRSA4096SHA3842021CA1.crl
├── hdr
│   └── hdr.bin
├── hkds
│   └── HKD-3931-0275D38.crt
└── rsa
    ├── encrypt_key.pem
    └── encrypt_key.pub
```

More details are available [here](https://github.com/confidential-containers/trustee/blob/main/deps/verifier/src/se/README.md#deployment-of-kbs-with-ibm-se-verifier).


## Create PersistentVolume (PV)

Example of perstistent volume:

```bash
oc apply -f - << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ibmse-pv
  namespace: kbs-operator-system
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadOnlyMany
  storageClassName: ""
  local:
    path: /opt/confidential-containers/ibmse
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-role.kubernetes.io/worker
              operator: Exists
EOF
```

Note: the *path* must refer to a local directory in the worker node, prepared in the previous section.

## Create PersistentVolumeClaim (PVC)

Example:

```bash
oc apply -f - << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ibmse-pvc
  namespace: kbs-operator-system
spec:
  accessModes:
    - ReadOnlyMany
  storageClassName: ""
  resources:
    requests:
      storage: 100Mi
EOF
```

### Create KbsConfig CRD

Finally, create the CRD for the operator:

```bash
oc apply -f - << EOF
apiVersion: confidentialcontainers.org/v1alpha1
kind: KbsConfig
metadata:
  labels:
    app.kubernetes.io/name: kbsconfig
    app.kubernetes.io/instance: kbsconfig-sample
    app.kubernetes.io/part-of: trustee-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: trustee-operator
  name: kbsconfig-sample
  namespace: kbs-operator-system
spec:
  kbsConfigMapName: kbs-config-sample
  kbsAuthSecretName: kbs-auth-public-key
  kbsDeploymentType: AllInOneDeployment
  kbsRvpsRefValuesConfigMapName: rvps-reference-values-sample
  kbsSecretResources: ["kbsres1"]
  kbsResourcePolicyConfigMapName: resource-policy-sample
  ibmSEConfigSpec:
    configPvcName: ibmse-pvc
EOF
```

### Set default project

```bash
oc project kbs-operator-system
```

### Check if the PODs are running

```bash
oc get pods -n kbs-operator-system
NAME                                                   READY   STATUS    RESTARTS   AGE
trustee-deployment-7bdc6858d7-bdncx                    1/1     Running   0          69s
trustee-operator-controller-manager-6c584fc969-8dz2d   2/2     Running   0          4h7m
```

Also, the log should report something like:

```bash
POD_NAME=$(kubectl get pods -l app=kbs -o jsonpath='{.items[0].metadata.name}' -n kbs-operator-system)
oc logs -n kbs-operator-system $POD_NAME
[2024-06-10T13:38:01Z INFO  kbs] Using config file /etc/kbs-config/kbs-config.json
[2024-06-10T13:38:01Z WARN  attestation_service::rvps] No RVPS address provided and will launch a built-in rvps
[2024-06-10T13:38:01Z INFO  attestation_service::token::simple] No Token Signer key in config file, create an ephemeral key and without CA pubkey cert
[2024-06-10T13:38:01Z INFO  api_server] Starting HTTPS server at [0.0.0.0:8080]
[2024-06-10T13:38:01Z INFO  actix_server::builder] starting 12 workers
[2024-06-10T13:38:01Z INFO  actix_server::server] Tokio runtime found; starting in existing Tokio runtime
```
