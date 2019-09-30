## To allow kubelet to create CSR

```
cat <<EOF | kubectl create -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
EOF
```

## CSR auto signing for bootstrapper
```
cat <<EOF | kubectl create -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: auto-approve-csrs-for-group
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
```

## Certificates self renewal
```
cat <<EOF | kubectl create -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: auto-approve-renewals-for-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
```


## bootstrap-token
```
cat <<EOF | tee /etc/systemd/system/bootstrap-token.yaml
apiVersion: v1
kind: Secret
metadata:
  # Name MUST be of form "bootstrap-token-<token id>"
  name: bootstrap-token-80a6ee
  namespace: kube-system

# Type MUST be 'bootstrap.kubernetes.io/token'
type: bootstrap.kubernetes.io/token
stringData:
  # Human readable description. Optional.
  description: "The default bootstrap token."

  # Token ID and secret. Required.
  token-id: 07401b
  token-secret: f395accd246ae52d

  # Expiration. Optional.
  expiration: 2019-12-05T12:00:00Z

  # Allowed usages.
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"

  # Extra groups to authenticate the token as. Must start with "system:bootstrappers:"
  auth-extra-groups: system:bootstrappers:node03,system:bootstrappers:ingress
```
```
kubectl create -f bootstrap-token.yaml
```

## Get Cluster-Info
```
kubectl cluster-info
```

```
kubectl config set-cluster bootstrap \
  --kubeconfig=bootstrap-kubeconfig-public  \
  --server=https://172.17.0.94:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true
```

## delete existing and create new one.
```
kubectl delete configmap cluster-info  -n kube-public  
```


```
kubectl -n kube-public create configmap cluster-info \
  --from-file=kubeconfig=bootstrap-kubeconfig-public
```

```
kubectl -n kube-public get configmap cluster-info -o yaml
```

## RBAC to allow anonymous users to access the cluster-info ConfigMap
```
kubectl create role anonymous-for-cluster-info --resource=configmaps --resource-name=cluster-info --namespace=kube-public --verb=get,list,watch
kubectl create rolebinding anonymous-for-cluster-info-binding --role=anonymous-for-cluster-info --user=system:anonymous --namespace=kube-public  
```

## Create bootstrap-kubeconfig for worker nodes. Run this command in Master
```
kubectl config set-cluster bootstrap \
  --kubeconfig=bootstrap-kubeconfig \
  --server=https://172.17.0.94:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true  
```

##
```
kubectl config set-credentials kubelet-bootstrap \
  --kubeconfig=bootstrap-kubeconfig \
  --token=07401b.f395accd246ae52d
```

##
```
kubectl config set-context bootstrap \
  --kubeconfig=bootstrap-kubeconfig \
  --user=kubelet-bootstrap \
  --cluster=bootstrap
```

##
```
kubectl config --kubeconfig=bootstrap-kubeconfig use-context bootstrap
```


## Copy the bootstrap-kubeconfig to worker node and then execute below steps from worker node.
```
scp -rp bootstrap-kubeconfig root@node03:~/bootstrap-kubeconfig
```

###SSH to worker
```
mkdir /var/lib/kubelet/
mv bootstrap-kubeconfig /var/lib/kubelet/  
```

## create service
```
cat <<EOF | tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/bin/kubelet \\
 --bootstrap-kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \\
 --cert-dir=/var/lib/kubelet/ \\
 --kubeconfig=/var/lib/kubelet/kubeconfig \\
 --rotate-certificates=true \\
 --runtime-cgroups=/systemd/system.slice \\
 --kubelet-cgroups=/systemd/system.slice
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
systemctl daemon-reload
systemctl start kubelet
systemctl status kubelet
```
