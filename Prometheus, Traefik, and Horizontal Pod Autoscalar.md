# Prometheus, Traefik, and Horizontal Pod Autoscalar

Let's try to setup a kubernetes (k8s) cluster with in-cluster monitoring.

## Prerequisite

The topics covered are fairly advanced and I assume basic knowledge of kubernetes plus helm. I also assume you've read this note (TODO).

## Installing k8s

For a quick test, we want something that can be push-button deployed. I settled on `microk8s` on ubuntu. Although `k3s` also works and has the advantage of being lightweight, I believe there are difference in the implementation that may increase the risk that something doesn't work out. `microk8s` is picked because it uses "pure upstream" k8s and doesn't change things from the official release. (TODO: concept of k8s *distribution*)

Just follow instruction on the official web sites. We will use a cluster with 1 master and 1 worker node - install it as usual on both nodes, then run `microk8s add-node` on the master node to obtain the command to run on the worker to have it *join* the cluster. I've also chosen to add user to the `microk8s` group so that commands can be run with `sudo`.

Then, instead addons. I find that while they are convinient, their maturity/robustness differ, and if you need to be fully in control, you'd probably be better off just installing those manually - hence that's what I went with. However, some base addons are important enough to be considered a "given" in any cluster:

`microk8s enable dns dashboard helm3 hostpath-storage host-access`

Finally, `microk8s` has scoped the kubernetes command line client locally to prevent conflicts. Since we're on an empty machine, we can just `alias kubectl="microk8s kubectl"` (and similarly for `helm` and `helm3`) to make life much saner (when you need to frantically enter commands after commands).

## A few words on the Monitoring Stack

Prometheus/Grafana is a lightweight monitoring tech stack. Prometheus looks like a time-series database to store metrics, but there are [important differences](https://iximiuz.com/en/posts/prometheus-is-not-a-tsdb/). Prometheus Open Source uses a pull based approach - it has various jobs to scrape data from endpoints/targets regularly. It does expects the data to be in a specific formats. "Exporter" can be used to convert non-prometheus conformant system's metric data into a form that can be read by Prometheus successfully.

Grafana is the UI/dashboard. It basically query against Prometheus. You can create your own visualizations, choosing from a variety of forms offered, and supplying the data using "prometheus query" (think SQL but in its own language). You can export/save the dashboard design as json file, and even share it with the world through Grafana Cloud.

There are other components available too. For instance, `alertmanager` and so on let you setup a watch to trigger alerts based on the metrics - perhaps connecting to an on-call system like PagerDuty.

## A note on kubernetes/helms/operator

Although Prometheus (as well as Traefik) are in fact a standalone software and can be installed as just a binary, we will be installing them in-cluster. There are three general approach:

- Through raw k8s manifests
- Using a helm chart (which is templated k8s manifest)
- Using an operator (it is supposed to crystalize "operational expertise" - the complex know-how of configuring and operating the software correctly - and allow normal people to do a production-strength deployment themselves)

Each represent a rise in level of abstraction and so you can expect things to get more complicated. Again, I do advise understanding which layer is which to avoid lots of confusion and pain down the line. Also - read the source code! It can be useful for troubleshooting.

Generally speaking:

- A k8s manifest will need some mechanism to interact with the binary to supply the configurations. It can be through `args` in Pod for command line argument, `env` for environmental variables (which can come from ConfigMap or Secrets etc), or through a file by a `volumeMounts`, with the actual data/file supplied through `volumes` -> ConfigMap -> special syntax in a yaml file. ([link](https://carlos.mendible.com/2019/02/10/kubernetes-mount-file-pod-with-configmap/)) (I hope this covers the most common cases)
- A Helm chart can defines its own tree of values, which are then feed into various places in a manifest. As the manifest is in the language of kubernetes, while the `values.yaml` is more designed from the end-user's perspective, this makes reading the source code all the more important. (Especially in case the documentation is lacking)
- Operator often relies on CustomResourceDefinition (CRD). A CRD let you extend k8s by defining your own resources. This let you design your own DSL in the domain of k8s, allowing more nature expression of intent using the same declarative model + synchronization loop paradigm. (Behind the scene custom programs need to be written by someone to act on those data)

> Tips: To list CRD, run `kubectl get crd`. But to *see* specific *instances of* CRD you've created, run `kubectl get <name of crd type> <name of crd>`. For example, for `servicemonitor.monitoring.coreos.com`, if we created one named "backend", then run `kubectl get ServiceMonitor backend`.


## Installing Prometheus

(TODO)

We will use the [Prometheus Operator](https://prometheus-operator.dev/). Be careful to apply the [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) stack instead of the plain operator as it contains a bunch of scraping targets etc already built in to monitor k8s from the get go.

## What is Traefik?

TODO

It is a cloud-native ingress/fronting server. Think counterpart of `nginx` (which is a reverse proxy/HTTP server).

What makes it *cloud native* is that it is designed from day one with cloud integration in mind and so can seamlessly work with/inside k8s. This explains some design choice that may look strange otherwise, such as using inversion of control/pull instead of push - instead of you specifying the routes in a master config file, you specify them at the "application side" and Traefik will pick it up and apply the right configurations automatically. This is known as "dynamic configuration" in Traefik Doc. To make it extensible, various "Provider" means that the source can be in different formats and Traefik will still work.

In this article we will stick to CRD IngressRoute Provider - since the pattern above is already very similar to the declarative + sync loop paradigm of k8s, why not just go all the way and fully embrace it. Besides, using CRD have the benefit that the syntax/semantics can be tailored to be more concise than the counterpart for the old k8s ingress provider.


## Installing Traefik

TODO

(Quick Note: Traefik version 1 and 2 are completely different. Although many still love the version 1, for the future choosing version 2 is necessary. Also be careful that the "official" helm chart still points to version 1, so you'd need to specially source the repo from what the doc provide.)

Traefik's Documentation can be tricky to get around. Reading the Traefik helm chart source code helped a lot to understand what's going on.

In short:

- Traefik as a binary lives inside a Pod/isolated env. The "Static Configuration" is the usual configuration in the sense of a plain program - it can be sourced from command line argument, env. variable, config file, etc. However due to Traefik's special nature this config is limited in scope - essentially to global/boostrapping configs.
- The helm chart already comes with a set of command line arguments supplied that should be enough for normal use case. In particular, prometheus metrics are enabled, dashboard is also enabled.
  - Should you need to, you can add more to it via `additionalCommandLineArguments`. If doing quick one-liner on `helm` command line, use the TODO syntax.
- A set of default container ports are defined: metrics (for prometheus), traefik (internal), web (the actual ingress), websecure (https counterpart of web). From `traefik` program's point of view these are the only things it know. Then, the helm chart define k8s services to expose these port (selectively - switched by the argument TODO), but may map them to a different port on the outside.

TODO explain why the official commands given to enable metrics and dashboard "works"

> Hint: In Traefik's doc, choose the `yaml` format - the other formats are for other use cases and not applicable to us.

## Letting Prometheus Monitor Traefik metrics

Because we installed Prometheus using an operator, we should configure it by creating new CRD instead of changing the configuration ourselves - the operator will scan it and automatically update the configs. Because the default recommendation is not to expose the metric service, we will use `PodMonitor` instead (which is only recently supported).

The one I used is this:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  labels:
    team: monitoring
  name: traefik-pod-monitor
  namespace: traefik-v2
spec:
  namespaceSelector:
    matchNames:
      - traefik-v2
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 15s
  selector:
    matchLabels:
      app.kubernetes.io/instance: traefik
```

Take special care to ensure that the pod selectors is correct - e.g. I had to look up and change the `matchLabels` from `app` to what you see above!

After applying, go to the Prometheus UI, Service Discovery (or targets), and wait until the targets are active.

You can then go to Grafana and import a Traefik dashboard. I used [this one](https://grafana.com/grafana/dashboards/11462-traefik-2/).

## HPA on custom metrics


## Bonus: Node Autoscaling


## Conclusion


## Reference

https://github.com/jeremyrickard/rps-demo-prometheus - A sample app that uses Prometheus and Kubernetes Custom Metrics to drive an HPA

https://github.com/mmatur/prometheus-traefik - Treafik and prometheus integration

https://fabianlee.org/2022/07/07/prometheus-monitoring-a-custom-service-using-servicemonitor-and-prometheusrule/

https://fabianlee.org/2022/07/08/prometheus-monitoring-services-using-additional-scrape-config-for-prometheus-operator/

https://traefik.io/blog/capture-traefik-metrics-for-apps-on-kubernetes-with-prometheus/

https://traefik.io/blog/install-and-configure-traefik-with-helm/

https://www.robert-jensen.dk/posts/2021-microk8s-with-traefik-and-metallb/

https://github.com/zalando-incubator/kube-metrics-adapter - General purpose metrics adapter for Kubernetes HPA metrics

https://livewyer.io/blog/2019/05/28/horizontal-pod-autoscaling/

https://www.civo.com/learn/monitoring-k3s-with-the-prometheus-operator-and-custom-email-alerts

https://docs.fuga.cloud/how-to-monitor-your-traefik-ingress-with-prometheus-and-grafana

https://dev.to/karvounis/advanced-traefik-configuration-tutorial-tls-dashboard-ping-metrics-authentication-and-more-4doh

https://www.civo.com/learn/application-performance-monitoring-with-prometheus-and-grafana-on-kubernetes



--

- Example of what operator offers that are not in a helm chart: ability to upgrade it correctly.
