# Open Policy Agent and Gatekeeper Installation
```bash
# Create a project for OPA to live
oc new-project opa

# Create a TLS secret for OPA
oc apply -f secret.yaml

# Create OPA resources
oc apply -f admission-controller.yaml
oc apply -f webhook-configuration.yaml

# Install Gatekeeper
oc apply -f gatekeeper.yaml

# Install policy
oc apply -f policy-resources.yaml

```

### ***NOTE:*** for every Advanced Cluster Manager spoke cluster participating in the Open Policy Agent and Gatekeeper integration run the following commands:
```bash
# Install Gatekeeper
oc apply -f gatekeeper.yaml

```
