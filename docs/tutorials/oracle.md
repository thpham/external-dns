# Setting up ExternalDNS for Oracle Cloud Infrastructure (OCI)

This tutorial describes how to setup ExternalDNS for usage within a Kubernetes cluster using OCI DNS.

Make sure to use the latest version of ExternalDNS for this tutorial.

## Creating an OCI DNS Zone

Create a DNS zone which will contain the managed DNS records. Let's use
`example.com` as a reference here.  Make note of the OCID of the compartment
in which you created the zone; you'll need to provide that later.

For more information about OCI DNS see the documentation [here][1].

## Deploy ExternalDNS

Connect your `kubectl` client to the cluster you want to test ExternalDNS with.
The OCI provider supports two authentication options: key-based and instance
principals.

### Key-based

We first need to create a config file containing the information needed to connect with the OCI API.

Create a new file (oci.yaml) and modify the contents to match the example
below. Be sure to adjust the values to match your own credentials, and the OCID
of the compartment containing the zone:

```yaml
auth:
  region: us-phoenix-1
  tenancy: ocid1.tenancy.oc1...
  user: ocid1.user.oc1...
  key: |
    -----BEGIN RSA PRIVATE KEY-----
    -----END RSA PRIVATE KEY-----
  fingerprint: af:81:71:8e...
  # Omit if there is not a password for the key
  passphrase: Tx1jRk...
compartment: ocid1.compartment.oc1...
```

Create a secret using the config file above:

```shell
$ kubectl create secret generic external-dns-config --from-file=oci.yaml
```

### OCI IAM Instance Principal

If you're running ExternalDNS within OCI, you can use OCI IAM instance
principals to authenticate with OCI.  This obviates the need to create the
secret with your credentials.  You'll need to ensure an OCI IAM policy exists
with a statement granting the `manage dns` permission on zones and records in
the target compartment to the dynamic group covering your instance running
ExternalDNS.
E.g.:

```
Allow dynamic-group <dynamic-group-name> to manage dns in compartment id <target-compartment-OCID>
```

You'll also need to add the `--oci-auth-instance-principal` flag to enable
this type of authentication. Finally, you'll need to add the
`--oci-compartment-ocid=ocid1.compartment.oc1...` flag to provide the OCID of
the compartment containing the zone to be managed.

For more information about OCI IAM instance principals, see the documentation [here][2].
For more information about OCI IAM policy details for the DNS service, see the documentation [here][3].

### OCI OKE Workloads Identity Principal

A workload running on a Kubernetes cluster is considered a resource in its own right.
A workload resource is identified by the unique combination of cluster, namespace, and service account.
This unique combination is referred to as the workload identity.
You can use the workload identity when defining IAM policies to grant fine-grained access to other OCI resources.

First, you'll also need to add the `--oci-auth-workload-principal` flag to enable
this type of authentication. Finally, you'll need to set the following environment variables
in the deployment.

For example, here are the Helm values to override:

```yaml
provider: oci
extraArgs:
- '--oci-auth-workload-principal'
- '--oci-compartment-ocid=ocid1.compartment.oc1...'
env:
- name: OCI_RESOURCE_PRINCIPAL_VERSION
  value: '2.2'
- name: OCI_RESOURCE_PRINCIPAL_REGION
  value: 'eu-zurich-1'
```


The following is an example of policy statement:

```txt
Allow any-user to manage dns in compartment id <compartment_ocid> where all { request.principal.type = 'workload', request.principal.namespace = 'external-dns', request.principal.service_account = 'external-dns', request.principal.cluster_id = 'ocid1.cluster.oc1...' }
```

For more information about OCI OKE workload identity principal, see the documentation [here][4].

## Manifest (for clusters with RBAC enabled)

Apply the following manifest to deploy ExternalDNS.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.13.4
        args:
        - --source=service
        - --source=ingress
        - --provider=oci
        - --policy=upsert-only # prevent ExternalDNS from deleting any records, omit to enable full synchronization
        - --txt-owner-id=my-identifier
        volumeMounts:
          - name: config
            mountPath: /etc/kubernetes/
      volumes:
      - name: config
        secret:
          secretName: external-dns-config
```

## Verify ExternalDNS works (Service example)

Create the following sample application to test that ExternalDNS works.

> For services ExternalDNS will look for the annotation `external-dns.alpha.kubernetes.io/hostname` on the service and use the corresponding value.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: example.com
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: http
```

Apply the manifest above and wait roughly two minutes and check that a corresponding DNS record for your service was created.

```
$ kubectl apply -f nginx.yaml
```

[1]: https://docs.cloud.oracle.com/iaas/Content/DNS/Concepts/dnszonemanagement.htm
[2]: https://docs.cloud.oracle.com/iaas/Content/Identity/Reference/dnspolicyreference.htm
[3]: https://docs.cloud.oracle.com/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm
[4]: https://docs.cloud.oracle.com/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm

