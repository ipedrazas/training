# Using Helm

This section covers:

* Helm repositories
* Installing Helm charts
* Fetching Releases
* Helm template



Installing a Helm chart is as simple as doing:

```
helm install stable/mysql
```

When we execute this command a few things happen:

* Helm checks if the chart exists in its `cache` directory (~/.helm/cache/archive)
* if the chart is not in the local cache, Helm pulls the chart (tgz file) and saves in the local cache
* Helm executes the install command
    * Merge templates + values + release
    * Sends manifests to Tiller
    * Tiller installs the release

## Helm Repository 

A chart repository is an HTTP server that houses an `index.yaml` file and optionally some packaged charts. When you're ready to share your charts, the preferred way to do so is by uploading them to a chart repository.

To create a repository we need to execute the following command:

```
helm repo add [flags] [NAME] [URL]
```

For example:

```
 helm repo add my-helm-repo https://my-relm-registy.com --ca-file my-helm-registry-ca.cer
helm repo update
```

You can list your helm repos by doing

```
helm repo list
```

Note that when you install a Helm Chart, you're using the name of the registry, for example, if I list my helm repos this is what I get:

```
-> %  helm repo list
NAME       	URL
stable     	https://kubernetes-charts.storage.googleapis.com
local      	http://127.0.0.1:8879/charts
weaveworks 	https://weaveworks.github.io/flux
remote      https://kubernetes-charts.storage.googleapis.com
```

This means that if I do:

```
helm install stable/mysql
```

Helm is going to go to https://kubernetes-charts.storage.googleapis.com to check the latest chart there and check if it's in my local cache, if not, it will download it.

Note that I could do something like

```
helm install remote/mysql
```

And the result would be the same than above since both chart repos are the same.

We could also do a `fetch` to just simply download the helm chart (tgz) without installing it.

```
helm fetch stable/mysql
```

This option is handy when we want to create or re-hydrate a local cache.

## Installing Helm charts

We have seen how to fecth and download charts. Let's see now the different options to install a few charts.

There are 2 options to install a helm chart:

* Using the `install` command
* Using the `upgrade` command plus the `-i` flag.


For example, both instructions will install mysql in our cluster. The difference is that with `upgrade -i` we are setting the `Release` name and if that release is not installed, helm will install it. However, once the release is installed, the `install` command cannot be used anymore. In summary, `upgrade` allows to install and upgrade a release.

```
helm upgrade -i my-release stable/mysql
```

### Values and Values files

Helm has the concept of templates that used with a set of default values will generate a set of manifests.

Values is a built-in Helm object and it provides access to values passed into the chart. Its contents come from four sources:

* The `values.yaml` file in the chart
* If this is a subchart, the `values.yaml` file of a parent chart
* A values file is passed into `helm install` or `helm upgrade` with the `-f` flag (`helm install -f myvals.yaml ./mychart`)
* Individual parameters passed with `--set` (such as `helm install --set foo=bar ./mychart`)


### Deleting a default key

If you need to delete a key from the default values, you may override the value of the key to be `null`, in which case Helm will remove the key from the overridden values merge.

For example, we have a chart that defines a `livenessProbe` as `HTTP` as follows:

```
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

We want now to replace that HTTP liveness probe by a shell script. This is what we would do:

```
helm install stable/drupal  --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

### Values files priority

You can specify the `--values`/`-f` flag multiple times. The priority will be given to the last (right-most) file specified. For example, if both myvalues.yaml and override.yaml contained a key called 'Test', the value set in override.yaml would take precedence:

```
-> % helm upgrade -f myvalues.yaml -f override.yaml redis ./redis
```

## How to retrieve an installed release

To list the releases that we have installed, we can issue the command

```
-> %  helm ls 
NAME          	REVISION	UPDATED                 	STATUS  	CHART      	APP VERSION	NAMESPACE
orbiting-goose	1       	Fri Jun  7 17:26:43 2019	DEPLOYED	mysql-1.1.1	5.7.14     	default
```

To retrieve the installed release we can use the `helm get` command:

```
> %  helm get orbiting-goose
REVISION: 1
RELEASED: Sun Fri  7 17:26:43 2019
CHART: mysql-1.1.1
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
busybox:
  image: busybox
  tag: 1.29.3
configurationFiles: {}
configurationFilesPath: /etc/mysql/conf.d/
extraInitContainers: |
  # - name: do-something
  #   image: busybox
  #   command: ['do', 'something']
extraVolumeMounts: |
  # - name: extras
  #   mountPath: /usr/share/extras
  #   readOnly: true
extraVolumes: |
  # - name: extras
  #   emptyDir: {}
image: mysql
imagePullPolicy: IfNotPresent
imageTag: 5.7.14
initContainer:
  resources:
    requests:
      cpu: 10m
      memory: 10Mi
initializationFiles: {}
livenessProbe:
  failureThreshold: 3
  initialDelaySeconds: 30
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 5
metrics:
  annotations: {}
  enabled: false
  flags: []
  image: prom/mysqld-exporter
  imagePullPolicy: IfNotPresent
  imageTag: v0.10.0
  livenessProbe:
    initialDelaySeconds: 15
    timeoutSeconds: 5
  readinessProbe:
    initialDelaySeconds: 5
    timeoutSeconds: 1
  resources: {}
  serviceMonitor:
    additionalLabels: {}
    enabled: false
nodeSelector: {}
persistence:
  accessMode: ReadWriteOnce
  annotations: {}
  enabled: true
  size: 8Gi
podAnnotations: {}
podLabels: {}
readinessProbe:
  failureThreshold: 3
  initialDelaySeconds: 5
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 1
resources:
  requests:
    cpu: 100m
    memory: 256Mi
securityContext:
  enabled: false
  fsGroup: 999
  runAsUser: 999
service:
  annotations: {}
  port: 3306
  type: ClusterIP
ssl:
  certificates: null
  enabled: false
  secret: mysql-ssl-certs
testFramework:
  image: dduportal/bats
  tag: 0.4.0
tolerations: []

HOOKS:
---
# orbiting-goose-mysql-test
apiVersion: v1
kind: Pod
metadata:
  name: orbiting-goose-mysql-test
  labels:
    app: orbiting-goose-mysql
    chart: "mysql-1.1.1"
    heritage: "Tiller"
    release: "orbiting-goose"
  annotations:
    "helm.sh/hook": test-success
spec:
  initContainers:
    - name: test-framework
      image: "dduportal/bats:0.4.0"
      command:
      - "bash"
      - "-c"
      - |
        set -ex
        # copy bats to tools dir
        cp -R /usr/local/libexec/ /tools/bats/
      volumeMounts:
      - mountPath: /tools
        name: tools
  containers:
    - name: orbiting-goose-test
      image: "mysql:5.7.14"
      command: ["/tools/bats/bats", "-t", "/tests/run.sh"]
      volumeMounts:
      - mountPath: /tests
        name: tests
        readOnly: true
      - mountPath: /tools
        name: tools
  volumes:
  - name: tests
    configMap:
      name: orbiting-goose-mysql-test
  - name: tools
    emptyDir: {}
  restartPolicy: Never
MANIFEST:

---
# Source: mysql/templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: orbiting-goose-mysql
  namespace: default
  labels:
    app: orbiting-goose-mysql
    chart: "mysql-1.1.1"
    release: "orbiting-goose"
    heritage: "Tiller"
type: Opaque
data:

  mysql-root-password: "RVMyR3J1OHhNbQ=="


  mysql-password: "cDNQOTczNklEWg=="
---
# Source: mysql/templates/tests/test-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orbiting-goose-mysql-test
  labels:
    app: orbiting-goose-mysql
    chart: "mysql-1.1.1"
    heritage: "Tiller"
    release: "orbiting-goose"
data:
  run.sh: |-
---
# Source: mysql/templates/pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: orbiting-goose-mysql
  namespace: default
  labels:
    app: orbiting-goose-mysql
    chart: "mysql-1.1.1"
    release: "orbiting-goose"
    heritage: "Tiller"
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "8Gi"
---
# Source: mysql/templates/svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: orbiting-goose-mysql
  namespace: default
  labels:
    app: orbiting-goose-mysql
    chart: "mysql-1.1.1"
    release: "orbiting-goose"
    heritage: "Tiller"
  annotations:
spec:
  type: ClusterIP
  ports:
  - name: mysql
    port: 3306
    targetPort: mysql
  selector:
    app: orbiting-goose-mysql
---
# Source: mysql/templates/deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: orbiting-goose-mysql
  namespace: default
  labels:
    app: orbiting-goose-mysql
    chart: "mysql-1.1.1"
    release: "orbiting-goose"
    heritage: "Tiller"
spec:
  template:
    metadata:
      labels:
        app: orbiting-goose-mysql
    spec:
      initContainers:
      - name: "remove-lost-found"
        image: "busybox:1.29.3"
        imagePullPolicy: "IfNotPresent"
        resources:
          requests:
            cpu: 10m
            memory: 10Mi

        command:  ["rm", "-fr", "/var/lib/mysql/lost+found"]
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      # - name: do-something
      #   image: busybox
      #   command: ['do', 'something']

      containers:
      - name: orbiting-goose-mysql
        image: "mysql:5.7.14"
        imagePullPolicy: "IfNotPresent"
        resources:
          requests:
            cpu: 100m
            memory: 256Mi

        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: orbiting-goose-mysql
              key: mysql-root-password
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: orbiting-goose-mysql
              key: mysql-password
              optional: true
        - name: MYSQL_USER
          value: ""
        - name: MYSQL_DATABASE
          value: ""
        ports:
        - name: mysql
          containerPort: 3306
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "mysqladmin ping -u root -p${MYSQL_ROOT_PASSWORD}"
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "mysqladmin ping -u root -p${MYSQL_ROOT_PASSWORD}"
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 3
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        # - name: extras
        #   mountPath: /usr/share/extras
        #   readOnly: true

      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: orbiting-goose-mysql
      # - name: extras
      #   emptyDir: {}
```

## Helm template

Render chart templates locally and display the output.

This does not require Tiller. However, any values that would normally be
looked up or retrieved in-cluster will be faked locally. Additionally, none
of the server-side testing of chart validity (e.g. whether an API is supported)
is done.

To render just one template in a chart, use `-x`:

```
-> % helm template mychart -x templates/deployment.yaml
```

## Exercises

1.- Deploy mysql using a helm chart

2.- Re-deploy MySql setting the password to whatever you think it would be appropriate.

3.- List the releases and make sure that it's your password the one that it's been used.

4.- List the contents of your `repositories.yaml`