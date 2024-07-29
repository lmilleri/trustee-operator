# Intel Trust Domain Extensions (TDX)

## PCCS configuration file

In order to Trustee to work properly with TDX, the file `sgx_default_qcnl.conf` needs to be mounted in the trustee pod file system.

Config map creation:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tdx-config
  namespace: kbs-operator-system
data:
  sgx_default_qcnl.conf: |
      {
        "collateral_service": "https://api.trustedservices.intel.com/sgx/certification/v4/",
        "pccs_url": "https://api.trustedservices.intel.com/sgx/certification/v4/"
        // "pccs_url": "https://localhost:8081/sgx/certification/v4/",
        // To accept insecure HTTPS certificate, set this option to false
        // "use_secure_cert": false
      }
```

## KBS config CRD

For enabling TDX specific configuration in trustee pod, the `KbsConfig` custom resource should have the `tdxConfigSpec` section populated as in the following example:

```yaml
apiVersion: confidentialcontainers.org/v1alpha1
kind: KbsConfig
metadata:  
  name: kbsconfig-sample
  namespace: kbs-operator-system
spec:
  # omitted all the rest of config
  # ...

  # TDX settings
  tdxConfigSpec:
    kbsTdxConfigMapName: tdx-config
```
