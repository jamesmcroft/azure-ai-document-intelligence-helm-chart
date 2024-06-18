# Azure AI Document Intelligence Custom Extraction Connected Container Helm Chart

This repository contains the Helm chart for deploying the [Azure AI Document Intelligence Connected Containers for use with custom extraction scenarios](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/containers/install-run?view=doc-intel-3.1.0&tabs=custom).

The templates in this Helm chart will deploy the following components to a Kubernetes cluster:

- [nginx (alpine)](https://hub.docker.com/_/nginx/) - Reverse proxy for the Layout and Custom Template services.
- [Azure AI Document Intelligence - Layout](https://mcr.microsoft.com/en-us/product/azure-cognitive-services/form-recognizer/layout-3.1/tags) - Service to perform layout analysis on documents to extract text, tables, and forms.
- [Azure AI Document Intelligence - Custom Template](https://mcr.microsoft.com/en-us/product/azure-cognitive-services/form-recognizer/custom-template-3.0/tags) - Service to create custom extraction models for specific document types.
- [Azure AI Document Intelligence - Studio](https://mcr.microsoft.com/product/azure-cognitive-services/form-recognizer/studio/tags) - Web-based Studio UI to create and manage custom extraction models.

This Helm chart also includes a custom `StorageClass` used for provisioning persistent storage as Azure File Shares with `nobrl` mount option enabled to prevent file locking issues with the Azure AI Document Intelligence Studio sqlite database file.

## Getting Started

To deploy the Azure AI Document Intelligence Connected Containers, you will need:

### Pre-requisites

- Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).
- Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
- Install [Helm](https://helm.sh/docs/intro/install/).
- Install [Azure Kube Login](https://azure.github.io/kubelogin/) if you are authenticating kubectl commands using Entra ID authentication with RBAC.
- An Azure Kubernetes Service (AKS) cluster. [Create a new AKS cluster](https://portal.azure.com/#create/microsoft.aks) if you don't have one.
- An Azure AI Document Intelligence service (required for billing purposes only). [Create a new DI service](https://portal.azure.com/#create/Microsoft.CognitiveServicesFormRecognizer) if you don't have one.

### Deploy the Helm Chart

The following steps will guide you through deploying the Azure AI Document Intelligence Connected Containers to your Kubernetes cluster.

> [!NOTE]
> The templates have been pre-configured with default values except for the required `documentIntelligence.env.billing` and `documentIntelligence.env.apikey` values. You can override these values using the `--set` option or a `values.yaml` file as described below. To customize the deployment further, you can modify the chart's default [`values.yaml`](./ai-document-intelligence/values.yaml) file in the `ai-document-intelligence` directory.

```bash
kubectl create namespace di
helm install di-extraction ai-document-intelligence --namespace di --set documentIntelligence.env.billing.value=your-document-intelligence-endpoint-value --set documentIntelligence.env.apikey.value=your-document-intelligence-apikey-value
```

When using secret values, you can configure the billing endpoint and API key with the `--set documentIntelligence.env.billing.valueFrom.secretKeyRef` and `--set documentIntelligence.env.apikey.valueFrom.secretKeyRef` options with both the required `name` and `key` values.

Alternatively, you can use a `values.yaml` file to configure the deployment. See more on [Helm values files](https://helm.sh/docs/chart_template_guide/values_files/).

### Access the Azure AI Document Intelligence Studio

By default, this Helm chart deploys only the containers required and does not expose any services to the public internet. To access the Azure AI Document Intelligence Studio and nginx proxy, you can use `kubectl port-forward` to forward the service ports to your local machine.

```bash
kubectl port-forward svc/di-extraction-ai-document-intelligence-nginx 5000:5000 --namespace di
kubectl port-forward svc/di-extraction-ai-document-intelligence-studio 5001:5001 --namespace di
```

You can then access the Studio UI at `http://localhost:5001`. When creating a new custom extraction project, the Form Recognizer Service Endpoint value will be set to the nginx proxy URL `http://localhost:5000`.

> [!NOTE]
> In a real-world scenario, you would expose the services using an ingress controller, service mesh, or other application gateway to control access to the services and provide a secure connection. This is not covered in this Helm chart as you may already be implementing this. Please refer to the [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/gateway/) for more information.

## FAQ

### How do I configure tolerations, affinity, and node selectors?

Although not included in the default `values.yaml` file, you can configure these settings using the Helm `--set` option or a `values.yaml` file for each of the deployment configurations. The values include `tolerations`, `affinity`, and `nodeSelector` for each of the deployments.

Here is an example `values.yaml` for how to configure the `layout` deployment with tolerations, affinity, and node selectors:

```yaml
layout:
  tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: "key"
                operator: "In"
                values:
                  - "value"
  nodeSelector:
    key: "value"
```
