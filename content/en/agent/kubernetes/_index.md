---
title: Kubernetes
kind: documentation
aliases:
    - /guides/basic_agent_usage/kubernetes
    - /agent/basic_agent_usage/kubernetes
    - /tracing/kubernetes/
    - /tracing/setup/kubernetes
    - /integrations/faq/using-rbac-permission-with-your-kubernetes-integration
    - /integrations/faq/gathering-kubernetes-events
    - /agent/kubernetes/event_collection
    - /agent/kubernetes/daemonset_setup
    - /agent/kubernetes/helm
further_reading:
- link: "agent/kubernetes/metrics"
  tag: "documentation"
  text: "Metrics collected by the Agent"
- link: "agent/autodiscovery"
  tag: "documentation"
  text: "Autodiscovery with the Agent"
---

Run the Datadog Agent in your Kubernetes cluster directly in order to start collecting your cluster and applications metrics, traces, and logs. You can deploy it with a [Helm chart](?tab=helm) or a [DaemonSet](?tab=daemonset).

**Note**: Agent version 6.0 and above only support versions of Kubernetes higher than 1.7.6. For prior versions of Kubernetes, consult the [Legacy Kubernetes versions section][1].

## Installation

{{< tabs >}}
{{% tab "Helm" %}}

To install the chart with a custom release name, `<RELEASE_NAME>` (e.g. `datadog-agent`):

1. [Install Helm][1].
2. Download the [Datadog `value.yaml` configuration file][2].
3. Retrieve your Datadog API key from your [Agent installation instructions][3] and run:

- **Helm v3+**

    ```bash
    helm install <RELEASE_NAME> -f datadog-values.yaml  --set datadog.apiKey=<DATADOG_API_KEY> stable/datadog
    ```

- **Helm v1/v2**

    ```bash
    helm install -f datadog-values.yaml --name <RELEASE_NAME> --set datadog.apiKey=<DATADOG_API_KEY> stable/datadog
    ```

This chart adds the Datadog Agent to all nodes in your cluster via a DaemonSet. It also optionally deploys the [kube-state-metrics chart][4] and uses it as an additional source of metrics about the cluster. A few minutes after installation, Datadog begins to report hosts and metrics.

Next, enable the Datadog features that you'd like to use: [APM][5], [Logs][6]

**Note**: For a full list of the Datadog chart's configurable parameters and their default values, refer to the [Datadog Helm repository README][7].

### Upgrading from chart v1.x

The Datadog chart has been refactored in v2.0 to regroup the `values.yaml` parameters in a more logical way.

If your current chart version deployed is earlier than `v2.0.0`, follow the [migration guide][8] to map your previous settings with the new fields.

[1]: https://v3.helm.sh/docs/intro/install/
[2]: https://github.com/helm/charts/blob/master/stable/datadog/values.yaml
[3]: https://app.datadoghq.com/account/settings#api
[4]: https://github.com/helm/charts/tree/master/stable/kube-state-metrics
[5]: /agent/kubernetes/apm?tab=helm
[6]: /agent/kubernetes/log?tab=helm
[7]: https://github.com/helm/charts/tree/master/stable/datadog
[8]: https://github.com/helm/charts/blob/master/stable/datadog/docs/Migration_1.x_to_2.x.md
{{% /tab %}}
{{% tab "DaemonSet" %}}

Take advantage of DaemonSets to deploy the Datadog Agent on all your nodes (or on specific nodes by [using nodeSelectors][1]).

To install the Datadog Agent on your Kubernetes cluster:

1. **Configure Agent permissions**: If your Kubernetes has role-based access control (RBAC) enabled, configure RBAC permissions for your Datadog Agent service account. From Kubernetes 1.6 onwards, RBAC is enabled by default. Create the appropriate ClusterRole, ServiceAccount, and ClusterRoleBinding with the following command:

    ```shell
    kubectl apply -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/clusterrole.yaml"

    kubectl apply -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/serviceaccount.yaml"

    kubectl apply -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/clusterrolebinding.yaml"
    ```

    **Note**: Those RBAC configurations are set for the `default` namespace by default. If you are in a custom namespace, update the `namespace` parameter before applying them.

2. **Create a secret that contains your Datadog API Key**. Replace the `<DATADOG_API_KEY>` below with [the API key for your organization][2]. This secret is used in the manifest to deploy the Datadog Agent.

    ```shell
    kubectl create secret generic datadog-secret --from-literal api-key="<DATADOG_API_KEY>" --namespace="default"
    ```

    **Note**: This create a secret in the `default` namespace. If you are in a custom namespace, update the `namespace` parameter of the command before running it.

3. **Create the Datadog Agent manifest**. Create the `datadog-agent.yaml` manifest out of one of the following templates:

    - [Manifest with Logs, APM, process, metrics collection enabled][3].
    - [Manifest with Logs, APM, and metrics collection enabled][4].
    - [Manifest with Logs and metrics collection enabled][5].
    - [Manifest with APM and metrics collection enabled][6].
    - [Vanilla Manifest with just metrics collection enabled][7].

     To enable trace collection completely, [extra steps are required on your application pod configuration][8]. Refer also to the [logs][9], [APM][10], and [processes][11] documentation pages to learn how to enable each feature individually.

     **Note**: Those manifests are set for the `default` namespace by default. If you are in a custom namespace, update the `metadata.namespace` parameter before applying them.

4. Optional - **Set your Datadog site**. If you are using the Datadog EU site, set the `DD_SITE` environment variable to `datadoghq.eu` in the `datadog-agent.yaml` manifest.

5. **Deploy the DaemonSet** with the command:

    ```shell
    kubectl apply -f datadog-agent.yaml
    ```

6. **Verification**: To verify the Datadog Agent is running in your environment as a DaemonSet, execute:

    ```shell
    kubectl get daemonset
    ```

     If the Agent is deployed, you will see output similar to the text below, where `DESIRED` and `CURRENT` are equal to the number of nodes running in your cluster.

    ```shell
    NAME            DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    datadog-agent   2         2         2         2            2           <none>          10s
    ```

7. Optional - **Setup Kubernetes State metrics**: Download the [Kube-State manifests folder][12] and apply them to your Kubernetes cluster to automatically collects [kube-state metrics][13]:

    ```shell
    kubectl apply -f <NAME_OF_THE_KUBE_STATE_MANIFESTS_FOLDER>
    ```

[1]: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector
[2]: https://app.datadoghq.com/account/settings#api
[3]: /resources/yaml/datadog-agent-all-features.yaml
[4]: /resources/yaml/datadog-agent-logs-apm.yaml
[5]: /resources/yaml/datadog-agent-logs.yaml
[6]: /resources/yaml/datadog-agent-apm.yaml
[7]: /resources/yaml/datadog-agent-vanilla.yaml
[8]: /agent/kubernetes/apm/#setup
[9]: /agent/kubernetes/log
[10]: /agent/kubernetes/apm
[11]: /infrastructure/process/?tab=kubernetes#installation
[12]: https://github.com/kubernetes/kube-state-metrics/tree/master/examples/standard
[13]: /agent/kubernetes/data_collected/#kube-state-metrics
{{% /tab %}}
{{% tab "Operator" %}}
[The Datadog Operator][2] is in public beta. The Datadog Operator is a way to deploy the Datadog Agent on Kubernetes and OpenShift. It reports deployment status, health, and errors in its Custom Resource status, and it limits the risk of misconfiguration thanks to higher-level configuration options. To get started, check out the [Getting Started page][1] in the [Datadog Operator repo][2] or install the operator from the [OperatorHub.io Datadog Operator page][3].

[1]: https://github.com/DataDog/datadog-operator/blob/master/docs/getting_started.md
[2]: https://github.com/DataDog/datadog-operator
[3]: https://operatorhub.io/operator/datadog-operator
{{% /tab %}}
{{< /tabs >}}

## Integrations

Once the Agent is up and running in your cluster, use [Datadog's Autodiscovery feature][2] to collect metrics and logs automatically from your pods.

## Further Reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /agent/faq/kubernetes-legacy
[2]: /agent/autodiscovery
