# Airflow on Kubernetes

## Installation

I've made an umbrella chart with several values overrides. Install it as usual:

```sh
cd airflow/helm
helm upgrade --install airflow . --namespace airflow --create-namespace
```

...or point ArgoCD Application to this folder.

## Executor

In k8s context there are two main executor options: `CeleryExecutor` and `KuberntesExecutor` which are essentially "worker pool" vs "fork" models with the respective pros and cons.

`CeleryExecutor` uses a worker pool to process work units. It's suitable for steady stream of small tasks since it will have lower scheduling latency and overhead. Worker pool can be scaled with `KEDA` based on queue length.

`KubernetesExecutor` creates separate pods for each work unit. It's suitable for resource-heavy tasks that exceed worker's resources, especially if those tasks come up in bursts.

`CeleryKubernetesExecutor` is designed to be the "best of both worlds" approach that implements two executors and decised which one to use based on task's queue.

I would start with Helm chart's default `CeleryExecutor` and adjust based on nature and characteristics of the tasks.

## DAGs

There are 3 ways of delivering DAGs to workers:
1. baking them into container images
2. syncing them from Git to shared volume
3. syncing then into local FS of Arflow pods

Option one is a non-starter for obvious reason of utter inconvenience. It requires requires building custom images and delivering them to clusters running Airflow on every change to DAG definitions.

Option two is the most dilligent on paper, since it reduces amount of wasted work on Git syncs, preventing potential throttling from GitHub and friends. However it requires setting up `ReadWriteMany` type of shared volume, which is it's own sort of hassle.

Option three is the most hands-off approach where you only need to point Aifrlow installation to Git repo containing DAGs and they are just synced into every pod that needs them. It comes at a cost of more requests to Git provider, but for a small startup I would start with this option and monitor this aspect. It might never become a real problem and would save on infra complexity at the same time. If it does become a problem - migration path to shared volume is straightforward.

## Logs

Technically, logs are visible in Airflow console with the default Helm chart values. However, to persist the logs one has to provide `ReadWriteMany` volume (notice the theme here).
