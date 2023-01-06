# Gloo Mesh with a unique Intermediate CA for each workload cluster

This guide describes how to use unique intermediate certificate
Authorities for each workload cluster for communication between the Gloo
Mesh Management Server deployed on the management cluster and the Relay
Agents deployed on the workload clusters.

### Background summary
The Gloo Mesh management server communicates with the
workload clusters via "Relay Agents" deployed on each workload cluster
as described in [Relay Architecture](https://docs.solo.io/gloo-mesh-enterprise/main/concepts/platform/relay/).
Some certificate architectures alternatives are described in
[Certificate architectures](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/prod/certs/cert-architecture/). This guide describes an additional
architecture using a unique intermediate CA for each workload
cluster and the management cluster.

# Preparation

1. Clone this repository and go to the directory where this `README.md` file is
located.

1. Deploy 3 clusters: 1 management cluster and 2 workload clusters.

1. Set the Kubernetes contexts to `mgmt`, `cluster1` and `cluster2`.

1. Set the Kubernetes context environment variables:

    ```bash
    export MGMT=mgmt
    export CLUSTER1=cluster1
    export CLUSTER2=cluster2
    ```

1.  Create the gloo-mesh namespace.

    The certificates will be created in the gloo-mesh namespace.

    ```bash
    kubectl create ns gloo-mesh --context ${MGMT}
    ```

## Install cert-manager

In this guide, we use cert-manager as an example, but you could use
other CA tools.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.10.1 \
  --set installCRDs=true \
  --kube-context=${MGMT}
```

# It's all about the certificates

## Create the Root CA Issuer

This repo includes a self-signed Root CA that you can use for testing.
Create a secret named `relay-root-ca` that has the cert/key pair
for the Root CA:

```base
kubectl apply -f cert/gloo-mesh-relay-root-ca.yaml --context ${MGMT}
```

The secret is a self-signed certificate with the `Issuer` and `Subject`
both set to `relay-root-ca`.

Next we create a cert-manager `Issuer` Custom Resource which will use
the above secret. This will result in a root CA that can be used to sign
certificates we'll need, such as the intermediate certificates.

```bash
kubectl apply -f cert/gloo-mesh-relay-issuer.yaml --context ${MGMT}
```

## Intermediate CAs for each cluster

The below creates the following for each cluster:
* A cert-manager `Certificate` for the cluster's intermediate CA.
* A cert-manager `Issuer` for the cluster's intermediate CA.

When this finishes, we will be able to create certificates for
the agents that will be signed by the `intermediate` CAs created here.

Note we also create an intermediate CA for the management cluster as it
also needs a certificate that will be signed by its intermediate CA.

```base
# Create an intermediate CA and Issuer for each cluster
for CLUSTER_NAME in ${MGMT} ${CLUSTER1} ${CLUSTER2}
do

# Creates a certificate CR which causes a secret with the intermediate
# CA key pair for the cluster to be created.

kubectl apply --context $MGMT -f - << EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: relay-intermed-ca-cert-${CLUSTER_NAME}
  namespace: gloo-mesh
spec:
  isCA: true                # This will be an (intermediate) CA
  # 1 year life
  duration: 8760h0m0s
  secretName: relay-intermed-ca-${CLUSTER_NAME}   # This secret will be created
  commonName: relay-intermed-ca-${CLUSTER_NAME}
  issuerRef:
    name: relay-root-ca     # Will be signed by the relay-root-ca
    kind: Issuer
    group: cert-manager.io
EOF

# Create a cert-manager Issuer for this cluster's intermediate CA
# certificate that was just created.
# Then we will be able to create certificates signed by this intermediate CA
# for communication between the workload cluster agents and the gloo
# mesh management server.

kubectl apply --context $MGMT -f - << EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: relay-intermed-ca-${CLUSTER_NAME}
  namespace: gloo-mesh
spec:
  ca:
    # This secret was just created above by the creation of the Certificate.
    secretName: relay-intermed-ca-${CLUSTER_NAME}
EOF

done
```

You should now see 3 intermediate cluster secrets and the
root CA that was created earlier. For example:

```bash
$ kubectl get secrets -n gloo-mesh --context ${MGMT}
NAME                         TYPE                DATA   AGE
relay-intermed-ca-cluster1   kubernetes.io/tls   3      23s
relay-intermed-ca-cluster2   kubernetes.io/tls   3      24s
relay-intermed-ca-mgmt       kubernetes.io/tls   3      24s
relay-root-ca                Opaque              2      3m34s
```

The cert-manager `Issuers`:

```bash
$ kubectl get Issuer -n gloo-mesh --context ${MGMT}
NAME                         READY   AGE
relay-intermed-ca-cluster1   True    3m21s
relay-intermed-ca-cluster2   True    3m21s
relay-intermed-ca-mgmt       True    3m21s
relay-root-ca                True    6m14s
```

And the cert-manager `Certificates`:
```bash
$ kubectl get Certificates -n gloo-mesh --context ${MGMT}
NAME                              READY   SECRET                       AGE
relay-intermed-ca-cert-cluster1   True    relay-intermed-ca-cluster1   2m31s
relay-intermed-ca-cert-cluster2   True    relay-intermed-ca-cluster2   2m31s
relay-intermed-ca-cert-mgmt       True    relay-intermed-ca-mgmt       2m32s
```

## Create the relay certificates for the workload clusters

Now that we have an intermediate CA for each cluster, we will create
relay certificates for each workload cluster. The certificates will be signed
by their own intermediate CA which were just created above.

The Gloo Mesh Management server certificate is created later
as it uses different values than the workload cluster's relay agent
certificates created here:


```bash
for CLUSTER_NAME in ${CLUSTER1} ${CLUSTER2}
do

kubectl apply --context $MGMT -f - << EOF
kind: Certificate
apiVersion: cert-manager.io/v1
metadata:
  name: gloo-remote-agent-intermed-${CLUSTER_NAME}-cert
  namespace: gloo-mesh
spec:
  commonName: gloo-mesh-agent
  dnsNames:
    # Must match the cluster name used in the helm chart install
    - "${CLUSTER_NAME}"
  # 1 year
  duration: 8760h0m0s
  issuerRef:
    group: cert-manager.io
    kind: Issuer
    name: relay-intermed-ca-${CLUSTER_NAME}
  renewBefore: 8736h0m0s
  secretName: gloo-agent-tls-cert-${CLUSTER_NAME}   # Creates this secret now
  usages:
    - digital signature
    - key encipherment
    - client auth
    - server auth
  privateKey:
    algorithm: "RSA"
    size: 4096
EOF

done
```

The following shows the 2 new secrets that have been created that contain
the workload relay agent certificates:

```bash
$ kubectl get secret -n gloo-mesh | grep gloo-agent-tls
gloo-agent-tls-cert-cluster1   kubernetes.io/tls   3      64s
gloo-agent-tls-cert-cluster2   kubernetes.io/tls   3      64s
```

This certificate is on the `MGMT` cluster in the secret
`gloo-agent-tls-cert-cluster1` in the `gloo-mesh` namespace with the key
`tls.crt`.

Digging into one of the certificates, the certificate for `cluster1`
contains:

* `Subject: CN=gloo-mesh-agent`
* `Issuer: CN=relay-intermed-ca-cluster1` (Signed by its intermediate CA.)
* `Subject Alternative Name:  DNSname: cluster1`  (The cluster name,
referred to in the `KubernetesCluster` Kubernetes Custom Resource
created later.)

## Copy the workload agent certificates to the workload clusters

The workload agent certificates are now in secrets in the management
cluster.

We will copy those secrets/certificates to the workload clusters as the
workload clusters will need to send them to the Gloo Mesh Management
Relay Server verify who they are.

```bash
# Copy each workload certificate secret to the workload cluster.
for CLUSTER_NAME in ${CLUSTER1} ${CLUSTER2}
do
    kubectl create ns gloo-mesh --context ${CLUSTER_NAME}

    kubectl get secret gloo-agent-tls-cert-${CLUSTER_NAME} \
      --namespace gloo-mesh \
      --output json \
      --context ${MGMT} \
      | jq 'del(.metadata.creationTimestamp,.metadata.resourceVersion,.metadata.uid)' \
      | kubectl apply --context ${CLUSTER_NAME} -f -
done
```

The agent secret is now in each cluster. For example on `cluster1`:

```bash
$ kubectl get secret -n gloo-mesh --context ${CLUSTER1}
NAME                           TYPE                DATA   AGE
gloo-agent-tls-cert-cluster1   kubernetes.io/tls   3      22s
```


## Create the Gloo Mesh Management Server certificate

The workload clusters have their Gloo Mesh Management workload cluster
certificates, but the management server doesn't have its certificate yet.

We will now create the Gloo Mesh Management Server certificate which
will be signed by its intermediate CA:

```bash
kubectl apply --context $MGMT -f - << EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: gloo-relay-server
  namespace: gloo-mesh
spec:
  commonName: gloo-mesh-mgmt-server
  dnsNames:
  - "*.gloo-mesh"
  # 1 year life
  duration: 8760h0m0s
  issuerRef:
    group: cert-manager.io
    kind: Issuer
    name: relay-intermed-ca-mgmt
  renewBefore: 8736h0m0s
  secretName: gloo-server-tls-cert    # Creates this
  usages:
    - server auth
    - client auth
  privateKey:
    algorithm: "RSA"
    size: 4096
EOF
```

# Deploy Gloo Mesh Enterprise

We now have all of the certificates for the workload clusters
and the Gloo Mesh Management Server, and are ready to deploy Gloo Mesh
to the management cluster.

1. Set your Gloo Mesh License Key to this environment variable:

    ```bash
    export GLOO_MESH_LICENSE_KEY=[...insert here...]
    ```

1. Set the version of Gloo Mesh you will be installing:

    ```bash
    export GLOO_MESH_VERSION=v2.1.2
    ```

1. Install Gloo Mesh Enterprise:

    ```bash
    helm upgrade --install \
        gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise \
        --namespace gloo-mesh \
        --kube-context ${MGMT}
        --create-namespace \
        --version ${GLOO_MESH_VERSION} \
        --set-string licenseKey=${GLOO_MESH_LICENSE_KEY} \
        --values helm/mgmt-values.yaml \
    ```

You may want to inspect the `helm/mgmt-values.yaml` file used here.

# Deploy The Workload Agents

# Get the management server endpoint
The workload agents need to know the endpoint of the Gloo Mesh
Management Server. Set the environment variables here and
echo MGMT_SERVER_NETWORKING_ADDRESS to verify it is valid:

```bash
MGMT_INGRESS_ADDRESS=$(kubectl get svc -n gloo-mesh gloo-mesh-mgmt-server --context ${MGMT} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
MGMT_INGRESS_PORT=$(kubectl -n gloo-mesh get service gloo-mesh-mgmt-server --context ${MGMT} -o jsonpath='{.spec.ports[?(@.name=="grpc")].port}')
MGMT_SERVER_NETWORKING_ADDRESS=${MGMT_INGRESS_ADDRESS}:${MGMT_INGRESS_PORT}
echo $MGMT_SERVER_NETWORKING_ADDRESS
```

The helm chart will refer to the above $MGMT_SERVER_NETWORKING_ADDRESS value
as the Gloo Mesh Management Server's `serverAddress`.

## Deploy the workload agents

For each workload cluster, we:
1. Create a `KubernetesCluster` Custom Resource with the name of
the cluster.
1. Install the helm chart for the Gloo Mesh Agent.

```bash
for CLUSTER_NAME in ${CLUSTER1} ${CLUSTER2}
do

# Create the Kubernetescluster custom resource for this cluster.
kubectl apply --context ${MGMT} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: KubernetesCluster
metadata:
  name: ${CLUSTER_NAME}
  namespace: gloo-mesh
spec:
  clusterDomain: cluster.local
EOF

# Install the gloo mesh agent on the workload cluster.
helm upgrade --install gloo-mesh-agent gloo-mesh-agent/gloo-mesh-agent \
  --create-namespace \
  --version ${GLOO_MESH_VERSION} \
  --namespace gloo-mesh \
  --kube-context=${CLUSTER_NAME} \
  --set relay.authority=gloo-mesh-mgmt-server.gloo-mesh \
  --set relay.serverAddress=${MGMT_SERVER_NETWORKING_ADDRESS} \
  --set relay.clientTlsSecret.name=gloo-agent-tls-cert-${CLUSTER_NAME} \
  --set cluster=${CLUSTER_NAME} \
  --set insecure=false

done
```

# Verify the clusters have been registered correctly

You can check the cluster(s) have been registered correctly using the
following commands:


```bash
pod=$(kubectl --context ${MGMT} -n gloo-mesh get pods -l app=gloo-mesh-mgmt-server -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${MGMT} -n gloo-mesh debug -q -i ${pod} --image=curlimages/curl -- curl -s http://localhost:9091/metrics | grep relay_push_clients_connected
```

You should get an output similar to this (it may take a minute or so
to report this):

```bash
# HELP relay_push_clients_connected Current number of connected Relay push clients (Relay Agents).
# TYPE relay_push_clients_connected gauge
relay_push_clients_connected{cluster="cluster1"} 1
relay_push_clients_connected{cluster="cluster2"} 1
```

Alternatively you can bring up the Gloo Mesh dashboard and verify
the workload clusters are shown on the main page:

```bash
meshctl dashboard
```
