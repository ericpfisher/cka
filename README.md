# CKA Resources and Practice

## Usage

Designed to be used with [Multipass](https://multipass.run) and its `--cloud-init` feature.

### Bootstrapping

#### Control Plane

```bash
multipass launch \
--cpus 2 \
--disk 50G \
--memory 4Gb \
--name k8scp \
--cloud-init cloud-init/cp.k8s.cloud-config.yaml
```

Once the control plane is bootstrapped, run commands to gather info for other nodes to join the cluster:

```bash
export K8S_CP_IPV4=$(multipass exec k8scp -- ip --brief address show | grep ens3 | awk '{print $3}' | cut -d / -f 1)
export K8S_JOIN_TOKEN=$(multipass exec k8scp -- sudo kubeadm token create)
export K8S_TOKEN_CA_CERT_HASH=$(multipass exec k8scp -- openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
openssl rsa -pubin -outform der 2>/dev/null | \
openssl dgst -sha256 -hex | \
sed 's/^.* //')
```

#### Worker(s)

Use the above environment variables to populate values in worker cloud-config using `envsubst`:

```bash
multipass launch \
--cpus 2 \
--disk 50G \
--memory 4Gb \
--name k8sworker \
--cloud-init <(envsubst '$K8S_CP_IPV4,$K8S_JOIN_TOKEN,$K8S_TOKEN_CA_CERT_HASH' cloud-init/worker.k8s.cloud-config.yaml)
```

Use the `kubectl get nodes` command to verify the worker(s) connected:

```bash
multipass exec k8scp -- kubectl get nodes
```

If done correctly, the worker(s) will bootstrap and connect to the control plane automatically. The above command should show something similar to:

```bash
NAME     STATUS   ROLES           AGE    VERSION
k8scp       Ready    control-plane   57m    v1.30.2
k8sworker   Ready    <none>          2m3s   v1.30.2
```
