To demonstrate the use of Admission Controllers and Security Context Constraints (SCC) in OpenShift, I'll provide sample YAML configurations and some example commands. These will illustrate how to set up an SCC and use admission controllers to enforce security policies on Kubernetes objects, specifically focusing on image validation and security contexts.

### Example 1: Security Context Constraints (SCC) Configuration

OpenShift's SCCs allow you to define security-related settings that apply to pods. Below is a YAML example of an SCC that restricts which images can be used, limits capabilities, and enforces run-as-user constraints.

#### SCC YAML Configuration

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: restricted-images
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
defaultAddCapabilities: []
requiredDropCapabilities:
  - ALL
runAsUser:
  type: MustRunAsRange
  uidRangeMin: 1000
  uidRangeMax: 2000
seLinuxContext:
  type: MustRunAs
  seLinuxOptions:
    level: "s0:c123,c456"
fsGroup:
  type: MustRunAs
  ranges:
  - min: 1000
    max: 2000
supplementalGroups:
  type: MustRunAs
  ranges:
  - min: 1000
    max: 2000
readOnlyRootFilesystem: true
users: []
groups: []
allowedCapabilities: []
volumes:
  - "configMap"
  - "emptyDir"
  - "persistentVolumeClaim"
  - "projected"
  - "secret"
```

**Key Points in the SCC:**
- **allowPrivilegedContainer**: `false` - Disables the use of privileged containers.
- **runAsUser**: Must run as a user within the UID range 1000-2000.
- **seLinuxContext**: SELinux level enforced.
- **readOnlyRootFilesystem**: Ensures containers run with a read-only root filesystem.
- **allowedCapabilities**: Empty, meaning no extra capabilities are granted.
- **volumes**: Restricts the types of volumes that can be used by pods under this SCC.

#### Applying the SCC

To apply the SCC in OpenShift, use the following command:

```bash
oc apply -f scc-restricted-images.yaml
```

This command will create the SCC `restricted-images` in your OpenShift cluster.

### Example 2: Admission Controller Configuration

OpenShift's admission controllers can be used to enforce policies like image signing and validation, which can prevent unsigned or unauthorized images from being deployed.

#### Admission Controller Configuration for Image Signature

Here's an example of an admission plugin configuration that enforces image signature policies:

1. **Create an Image Policy Configuration:**

   ```yaml
   apiVersion: config.openshift.io/v1
   kind: Image
   metadata:
     name: cluster
   spec:
     allowedRegistriesForImport:
       - domainName: "quay.io"
         insecure: false
     externalRegistryHostnames:
       - "external-registry.example.com"
     registrySources:
       allowedRegistries:
         - "quay.io"
       blockedRegistries:
         - "docker.io"
     signaturePolicy:
       type: "Simple"
       simple:
         rejectIfNoSignatureFound: true
         requireSignatureVerification: true
   ```

   This configuration specifies that images must have a valid signature and only allows images from specific registries.

2. **Apply the Admission Controller Policy:**

   ```bash
   oc apply -f image-policy.yaml
   ```

   This command sets up the image policy to enforce image signature requirements on the cluster.

### Summary

By using **Security Context Constraints (SCC)** and **Admission Controllers** in OpenShift, you can enforce strict security policies that control the conditions under which pods are allowed to run and which images can be used. SCCs allow you to define security requirements like user IDs, capabilities, and volume types, while admission controllers can validate and enforce policies like image signing. This ensures a more secure Kubernetes environment in OpenShift.
