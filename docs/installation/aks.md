# Install on AKS

<details markdown="1">
<summary><b>TIP</b>: Make sure you have the right quota and permissions.</summary>

- Confirm you have enough quota for GPU-enabled node SKUs (like `Standard_NC48ads_A100_v4`).
- Verify that you have the right permissions and that the account you use with Azure CLI can create AKS clusters, node pools, etc.
</details>

## Installing Prerequisites

Before running this setup, ensure you have:

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [Helm](https://helm.sh/docs/intro/install/)

## 1. Define environment variables

```bash
AZURE_RESOURCE_GROUP="${AZURE_RESOURCE_GROUP:-production-stack}"
AZURE_REGION="${AZURE_REGION:-southcentralus}"
CLUSTER_NAME="${CLUSTER_NAME:-production-stack}"
USER_NAME="${USER_NAME:-azureuser}"
GPU_NODE_POOL_NAME="${GPU_NODE_POOL_NAME:-gpunodes}"
GPU_NODE_COUNT="${GPU_NODE_COUNT:-1}"
GPU_VM_SIZE="${GPU_VM_SIZE:-Standard_NC48ads_A100_v4}"
```

- `AZURE_RESOURCE_GROUP`: The name of the Azure resource group. The default value is `production-stack`.
- `AZURE_REGION`: The Azure location or region where the AKS cluster will be deployed. The default value is `southcentralus`.
- `CLUSTER_NAME`: The name of the AKS cluster. The default value is `production-stack`.
- `USER_NAME`: The username for the AKS cluster. The default value is `azureuser`.
- `GPU_NODE_POOL_NAME`: The name of the GPU node pool. The default value is `gpunodes`.
- `GPU_NODE_COUNT`: The number of GPU nodes in the GPU node pool. The default value is `1`.
- `GPU_VM_SIZE`: The SKU of the GPU VMs in the GPU node pool. The default value is `Standard_NC48ads_A100_v4`. You can find more GPU VM sizes [here](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview#gpu-accelerated).

## 2. Create a Resource Group

```bash
az group create \
  --name "${AZURE_RESOURCE_GROUP}" \
  --location "${AZURE_REGION}"
```

## 3. Create an AKS Cluster (CPU Only)

If you only intend to run CPU-based models, you can create a simple AKS cluster as follows:

```bash
az aks create \
    --resource-group "${AZURE_RESOURCE_GROUP}" \
    --name "${CLUSTER_NAME}" \
    --enable-oidc-issuer \
    --enable-workload-identity \
    --enable-managed-identity \
    --node-count 1 \
    --location "${AZURE_REGION}" \
    --admin-username "${USER_NAME}" \
    --generate-ssh-keys \
    --os-sku Ubuntu
```

## 4. (Optional) Add a GPU Node Pool

If you want to run GPU-backed models, you need a GPU node pool. Below is an example using the `Standard_NC48ads_A100_v4` SKU. Feel free to adjust `--node-vm-size` to match a GPU SKU you have quota for:

```bash
az aks nodepool add \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --cluster-name "${CLUSTER_NAME}" \
  --name "${GPU_NODE_POOL_NAME}" \
  --node-count "${GPU_NODE_COUNT}" \
  --node-vm-size "${GPU_VM_SIZE}" \
  --node-taints sku=gpu:NoSchedule
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3
```

## 5. Get AKS Credentials

Download and merge the kubeconfig for your AKS cluster:

```bash
az aks get-credentials \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --name "${CLUSTER_NAME}"
  --overwrite-existing
```

## 6. Install the NVIDIA Device Plugin (for GPU Clusters)

If you created a GPU node pool, install the NVIDIA device plugin so Pods can request GPUs:

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.1/deployments/static/nvidia-device-plugin.yml
```

Confirm the DaemonSet is running:

```bash
kubectl get daemonset nvidia-device-plugin-daemonset -n kube-system
```

## 7. Install KubeAI

Add the KubeAI Helm repository and update:

```bash
helm repo add kubeai https://www.kubeai.org
helm repo update
```

**(Optional)** If you plan to use private or TOS-protected models from Hugging Face, export your token:

```bash
export HUGGING_FACE_HUB_TOKEN=<YOUR_HF_TOKEN>
```

Install KubeAI:

```bash
helm upgrade --install kubeai kubeai/kubeai \
  --set secrets.huggingface.token="${HUGGING_FACE_HUB_TOKEN}" \
  --wait
```

### Customizing Resource Profiles

If you are running GPU nodes (e.g. `Standard_NC48ads_A100_v4`), you may need to define custom resource profiles or edit existing ones. In your Helm values, you can specify a node selector matching your GPU pool labels:

```yaml
resourceProfiles:
  nvidia-azure-a100:
    nodeSelector:
      kubernetes.io/hostname: <optional-custom-node-label>
    limits:
      nvidia.com/gpu: "1"
    requests:
      nvidia.com/gpu: "1"
```

Then use `resourceProfile: nvidia-azure-a100:1` when installing models. For details, see the [Configure Resource Profiles](../how-to/configure-resource-profiles.md) guide.

## 8. Deploy Models

You can now deploy your models using either:

- The [kubeai/models Helm chart](../how-to/install-models.md#installing-models-with-helm), or
- Directly applying Model manifests with `kubectl apply -f`.

For instance, to enable a CPU-based text generation model and a GPU-based text embedding model, you might create a `kubeai-models.yaml`:

```yaml
catalog:
  llama-3.1-8b-instruct-fp8-l4:
    enabled: true
    engine: VLLM
    resourceProfile: nvidia-azure-a100:1
    minReplicas: 1

  phi4-gpu:
    enabled: true
    features: [TextGeneration]
    url: hf://microsoft/phi-4
    engine: VLLM
    resourceProfile: nvidia-azure-a100:1
    minReplicas: 1
```

Then apply:

```bash
helm install kubeai-models kubeai/models \
    -f kubeai-models.yaml
```

Wait for the Pods to appear and become ready:

```bash
kubectl get pods --watch
```

## 9. Testing

### Port-forward to the KubeAI service

```bash
kubectl port-forward svc/kubeai 8000:80
```

Now you can query the OpenAI-compatible endpoints at `localhost:8000/openai/v1`. For example:

```bash
curl http://localhost:8000/openai/v1/models
```

Or test with a completion:

```bash
curl http://localhost:8000/openai/v1/completions \
   -H "Content-Type: application/json" \
   -d '{
     "model": "phi4-gpu",
     "prompt": "Hello from AKS!",
     "max_tokens": 100
   }'
```

---

**That’s it!** You now have KubeAI running on an AKS cluster. You can further customize your setup by defining resource profiles for specific GPU SKUs, hooking up advanced caching solutions like Azure Files, or layering your own authentication. Check out the rest of the [KubeAI docs](../index.md) for more guidance.
