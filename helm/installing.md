#  Installing Helm

To install Helm we only have to get the tar file from https://github.com/helm/helm/releases and execute:

```
helm init
```

But there are other options. For example, 

```
helm init --history-max 50
```

Setting `--history-max` on helm init is recommended as configmaps and other objects in helm history can grow large in number if not purged by max limit. Without a max history set the history is kept indefinitely, leaving a large number of records for helm and tiller to maintain.


### Setting up RBAC for tiller 

```
kubectl -n kube-system create sa tiller

kubectl create clusterrolebinding tiller-cluster-rule \
    --clusterrole=cluster-admin \
    --serviceaccount=kube-system:tiller

helm init --skip-refresh --upgrade --service-account tiller
```

### Other Helm options

The `--node-selectors` flag allows us to specify the node labels required for scheduling the Tiller pod.

```
helm init --node-selectors "roles"="admin"
```

`--override` allows you to specify properties of Tiller’s deployment manifest. 

```
helm init \
    --override "spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].weight"="1" \
    --override "spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].preference.matchExpressions[0].key"="my-team-nodes"
```

## Helm and Openshift

Helm runs in Openshift, however, it's important to note that Openshift doesn't allow to run containers as root and many Helm charts in public repos do not comply with this policy.

### Securing Helm

There are 3 aspects to make Helm more secure:

* Using proper RBAC rules.
* Using TLS.
* Using a secure backend for storing release information.

```
helm init \
--override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}' \
--tiller-tls \
--tiller-tls-verify \
--tiller-tls-cert=cert.pem \
--tiller-tls-key=key.pem \
--tls-ca-cert=ca.pem \
--service-account=accountname
```

### Helm Options

Environment:
  $HELM_HOME          set an alternative location for Helm files. By default, these are stored in ~/.helm
  $HELM_HOST          set an alternative Tiller host. The format is host:port
  $HELM_NO_PLUGINS    disable plugins. Set HELM_NO_PLUGINS=1 to disable plugins.
  $TILLER_NAMESPACE   set an alternative Tiller namespace (default "kube-system")
  $KUBECONFIG         set an alternative Kubernetes configuration file (default "~/.kube/config")

Usage:
  helm [command]

Available Commands:
  completion  Generate autocompletions script for the specified shell (bash or zsh)
  create      create a new chart with the given name
  delete      given a release name, delete the release from Kubernetes
  dependency  manage a chart's dependencies
  fetch       download a chart from a repository and (optionally) unpack it in local directory
  get         download a named release
  history     fetch release history
  home        displays the location of HELM_HOME
  init        initialize Helm on both client and server
  inspect     inspect a chart
  install     install a chart archive
  lint        examines a chart for possible issues
  list        list releases
  package     package a chart directory into a chart archive
  plugin      add, list, or remove Helm plugins
  repo        add, list, remove, update, and index chart repositories
  reset       uninstalls Tiller from a cluster
  rollback    roll back a release to a previous revision
  search      search for a keyword in charts
  serve       start a local http web server
  status      displays the status of the named release
  template    locally render templates
  test        test a release
  upgrade     upgrade a release
  verify      verify that a chart at the given path has been signed and is valid
  version     print the client/server version information

Flags:
      --debug                           enable verbose output
      --home string                     location of your Helm config. Overrides $HELM_HOME (default "/Users/ipedrazas/.helm")
      --host string                     address of Tiller. Overrides $HELM_HOST
      --kube-context string             name of the kubeconfig context to use
      --kubeconfig string               absolute path to the kubeconfig file to use
      --tiller-connection-timeout int   the duration (in seconds) Helm will wait to establish a connection to tiller (default 300)
      --tiller-namespace string         namespace of Tiller (default "kube-system")

## Exercises:

1.- Install Helm in your minikube cluster with the following configuration:

    * We want `tiller` to run in the `helm-system` namespace
    * Make sure that we don't keep more than 10 versions of each release.
    * Tiller runs in a node with the `platform` label

2.- Create a set of configurations for:
    * Local (minikube)
    * Lab
    * Development environment (air gap)
