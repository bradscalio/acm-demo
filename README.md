## Setup
1. Deploy ACM to OpenShift using the Operator. This will be your **hub cluster**.
2. Setup managed clusters in ACM (including local cluster). Apply the label `role=feature-candidate` to one cluster, and `role=stable` to all other clusters.
3. Deploy Ansible Tower (tested with 3.7.3), ensuring that the `openshift` `google-auth` pip modules are installed. One way to do this would be use my custom task-runner image for an openshift deployment, specifying `kubernetes_task_image: quay.io/akrohg/ansible-task` in the `group_vars` of your [openshift installer](https://releases.ansible.com/ansible-tower/setup_openshift/). I installed Tower in this way on the **ACM Hub cluster**.
4. Edit the `vars` in `ansible-hook/setup/main.yml` to provide creds and endpoints for Ansible Tower, your Hub cluster, and your spoke clusters.
5. Setup for Ansible Tower. This performs the following:
* Create an API key for Tower
* Import this repo as a *Project* for Tower
* Create a Job Template in Tower to run as a post hook to the ACM Application Deployment
* Create a Secret in your **hub cluster** to authenticate to Tower
* Subscribe to the **Ansible Automation Platform Resource Operator** in your **hub cluster**
* Create `ClusterRoleBindings` required for the app service account to read node info and report what cloud it's running on. I previously included this in the app workload itself, but the Reconciliation failed from ACM - i suspect insufficient privileges.
```bash
ansible-playbook ansible-hook/setup/main.yml
```
6. Deploy an Apache HTTPD load balancer to your **hub cluster**. This will load balance between app deployments across managed clusters. No apps have been created yet, so this should initially return a 503.
```bash
oc apply -k load-balancer
```
Grab the route:
```bash
oc get route load-balancer -n acm-demo
```

## Performing the Demo
> NOTE: Run all commands against your **hub cluster**.
1. Add the app subscription:
```bash
oc apply -k subscription
```
This will apply `v1` of the app to your `feature-candidate` cluster. The `AnsibleJob` should run and update the `ConfigMap` of the load balancer to target your newly deployed app. This probably takes about 4 minutes. Refresh the load balancer route in your browser to see that it's now resolving.
2. Release `v1` to stable clusters:
```bash
oc apply -f subscription/subscription-v1.yaml
```
This will apply `v1` of the app to all managed clusters (both `feature-candidate` and `stable`). A new `AnsibleJob` resource should be created which will update the load balancer to distribute traffic among all clusters. This may take another 4 minutes or so - refresh the load balancer a few times to see (based on the zone/provider reported by the app) that traffic distributed using a round robin strategy.
3. Release `v2` to your feature candidate cluster:
```bash
oc apply -f subscription/subscription-v2-alpha.yaml
```
After reconciliation, refreshing on the load balancer should show that your feature candidate cluster has been upgraded to `v2` of the app.
4. Release `v2` to stable clusters:
```bash
oc apply -f subscription/subscription-v2.yaml
```