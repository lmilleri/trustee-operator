# Intel Trusted Authority (ITA)

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
 image:  quay.io/rh_ee_lmilleri/trustee-operator-catalog:v0.1.0-ita
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

### Trustee ConfigMap object

This command will create the ConfigMap object that provides Trustee all the needed configuration.

Note: you'd probably need to change the Intel API key. Now it is set to ```api_key": "tBfd5kKX2x9ahbodKV1..."```

```bash
oc apply -f - << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: kbs-config
  namespace: kbs-operator-system
data:
  kbs-config.json: |
    {
        "insecure_http" : true,
        "sockets": ["0.0.0.0:8080"],
        "auth_public_key": "/etc/auth-secret/publicKey",
        "attestation_token_config": {
          "attestation_token_type": "CoCo"
        },
        "repository_config": {
          "type": "LocalFs",
          "dir_path": "/opt/confidential-containers/kbs/repository"
        },
        "as_config": {
          "work_dir": "/opt/confidential-containers/attestation-service",
          "policy_engine": "opa",
          "attestation_token_broker": "Simple",
          "attestation_token_config": {
            "duration_min": 5
          },
          "rvps_config": {
            "store_type": "LocalJson",
            "store_config": {
              "file_path": "/opt/confidential-containers/rvps/reference-values/reference-values.json"
            }
          }
        },
        "policy_engine_config": {
          "policy_path": "/opt/confidential-containers/opa/policy.rego"
        },
        "intel_trust_authority_config" : {
          "base_url": "https://api.trustauthority.intel.com",
          "api_key": "tBfd5kKX2x9ahbodKV1...",
          "certs_file": "/etc/intel-trust-authority-certs.txt"
        }
    }
EOF
```

### ITA ConfigMap object

When the trustee operator is deployed, the configmap object *ita-config-sample* is created for mounting the ITA certificate:

```yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: ita-config-sample
  namespace: kbs-operator-system
data:
  intel-trust-authority-certs.txt: |
    {"keys":[{"alg":"PS384","e":"AQAB","kid":"79d80711b754cceb307d4278dc59957f27eb55a8e33d3b824967975843dcbf21df924eebaf93fce186fd291d36817785","kty":"RSA","n":"yE07D7FRSXLsswdeK7h22kw-Xv2K2r4NFoefWElZ6FWmLvCcd27wGEczNeKrE91SWczPtR279tTasQN6_v8qsswC5rCGYlrRWvE0vuPoUXezlV4PX0tCJJJmxWtXFXW0dChWvR1j-_viOItfR8jrybV2-DyVBgGX1ad4BLJJseglPXcofhnKYcG9gp8J2zPFqs1tu6jTW-He3Xw7ZeQNq0n4ZfrRBM3GEYVVsWGlTlqVidMhbvMXSQgz1x2QjyPC2mSUrT-JyA2xTm84Mv_Lmz6FpHXsjXMyPKCUVUf8LSTAiw3UsHa-7QGUW51hh9lZsbWkdSfwGUGxjrcMNEwYo3KvcF8f9Cv1_bkla396poQhtTIHuV478PobzsCfdkbCF5CfwZN31KbqyD9o9pVyzmmQUmOikIZuiSPRnfIU_P8duM5F6yvxQPITZf1RhOPBNYLiOJge7C89OmsM46UKtAYNTieBH-J8oWUUWfAX3pO38bKIzNwHDelSbaeHterJ","x5c":["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"]},{"alg":"RS256","e":"AQAB","kid":"79d80711b754cceb307d4278dc59957f27eb55a8e33d3b824967975843dcbf21df924eebaf93fce186fd291d36817785","kty":"RSA","n":"yE07D7FRSXLsswdeK7h22kw-Xv2K2r4NFoefWElZ6FWmLvCcd27wGEczNeKrE91SWczPtR279tTasQN6_v8qsswC5rCGYlrRWvE0vuPoUXezlV4PX0tCJJJmxWtXFXW0dChWvR1j-_viOItfR8jrybV2-DyVBgGX1ad4BLJJseglPXcofhnKYcG9gp8J2zPFqs1tu6jTW-He3Xw7ZeQNq0n4ZfrRBM3GEYVVsWGlTlqVidMhbvMXSQgz1x2QjyPC2mSUrT-JyA2xTm84Mv_Lmz6FpHXsjXMyPKCUVUf8LSTAiw3UsHa-7QGUW51hh9lZsbWkdSfwGUGxjrcMNEwYo3KvcF8f9Cv1_bkla396poQhtTIHuV478PobzsCfdkbCF5CfwZN31KbqyD9o9pVyzmmQUmOikIZuiSPRnfIU_P8duM5F6yvxQPITZf1RhOPBNYLiOJge7C89OmsM46UKtAYNTieBH-J8oWUUWfAX3pO38bKIzNwHDelSbaeHterJ","x5c":["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"]}]}
```

You might need to edit the certificate with the command:

```bash
oc edit cm ita-config-sample
```


### Create secrets

How to create secrets to be shared with the attested clients?
In this example we create a secret *kbsres1* with two entries. These resources (key1, key2) can be retrieved by the Trustee clients.
You can add more secrets as per your requirements.

```bash
oc create secret generic kbsres1 --from-literal key1=res1val1 --from-literal key2=res1val2 -n kbs-operator-system
```

### Create KbsConfig CRD

Finally, the CRD for the operator is created:

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
  kbsConfigMapName: kbs-config
  kbsAuthSecretName: kbs-auth-public-key
  kbsDeploymentType: AllInOneDeployment
  kbsRvpsRefValuesConfigMapName: rvps-reference-values-sample
  kbsSecretResources: ["kbsres1"]
  kbsResourcePolicyConfigMapName: resource-policy-sample
  tdxConfigSpec:
    kbsTdxConfigMapName: tdx-config-sample
  itaConfigSpec:
    kbsItaConfigMapName: ita-config-sample
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
