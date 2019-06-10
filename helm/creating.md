# Authoring Charts

Helm uses a packaging format called charts. A chart is a collection of files that describe a related set of Kubernetes resources. A single chart might be used to deploy something simple, like a memcached pod, or something complex, like a full web app stack with HTTP servers, databases, caches, and so on.

Charts are created as files laid out in a particular directory tree, then they can be packaged into versioned archives to be deployed.

This document explains the chart format, and provides basic guidance for building charts with Helm.

## Charts and Versioning

Every chart must have a version number. A version must follow the SemVer 2 standard. Unlike Helm Classic, Kubernetes Helm uses version numbers as release markers. Packages in repositories are identified by name plus version.

For example, an nginx chart whose version field is set to version: 1.2.3 will be named:

```
nginx-1.2.3.tgz
```

More complex SemVer 2 names are also supported, such as version: `1.2.3-alpha.1+ef365`. But non-SemVer names are explicitly disallowed by the system.

The version field inside of the `Chart.yaml` is used by many of the Helm tools, including the CLI and the Tiller server. When generating a package, the `helm package` command will use the version that it finds in the `Chart.yaml` as a token in the package name. The system assumes that the version number in the chart package name matches the version number in the `Chart.yaml`. Failure to meet this assumption will cause an error.

### The appVersion field

Note that the `appVersion` field is not related to the `version` field. It is a way of specifying the version of the application. For example, the `drupal` chart may have an `appVersion`: 8.2.1, indicating that the version of Drupal included in the chart (by default) is` 8.2.1`. This field is informational, and has no impact on chart version calculations.

## Creating a Chart

The command to write a new Helm chart is very simple

```
helm create mynewchart
```

This creates all the files and folders needed to define a chart.

## Exercise

1.- In this exercise we are going to practice everything we have learned so far. We're going to create a Chart for our application and deploy it.

* Install that chart without modifying it.
* Create a simple application and package it with Docker.
* Push that docker image to a docker registry.
* Re-deploy the chart using that image that you have just created. Do not modify any file in the Chart.

## Templates

Helm templates are basically Go Templates. Not the best, not the worst.

Templates generate manifest files, which are YAML-formatted resource descriptions that Kubernetes can understand.

The structure of a Chart is:

```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

The `templates/` directory is for template files. When Tiller evaluates a chart, it will send all of the files in the `templates/` directory through the template rendering engine. Tiller then collects the results of those templates and sends them on to Kubernetes.

Let's look at on the templates the `create` command has generated, for example, the `deployment.yaml` starts with:

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: {{ include "mynewapp.fullname" . }}
  labels:
    app: {{ include "mynewapp.name" . }}
    chart: {{ include "mynewapp.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
spec:
```

The template directive `{{ .Release.Name }}` injects the release name into the template. The values that are passed into a template can be thought of as namespaced objects, where a dot (`.`) separates each namespaced element.

The Release object is one of the [built-in objects](built-in.md) for Helm and it will display the release name that Tiller assigns to our release.

**Hard-coding the name:** into a resource is usually considered to be bad practice. Names should be unique to a release. So we might want to generate a name field by inserting the release name.

Also, the `name`: field is limited to 63 characters because of limitations to the DNS system. For that reason, `release names` are limited to 53 characters.

A template directive is enclosed in `{{` and `}}` blocks.

### Arguments

Sometimes we want to transform the supplied data in a way that makes it more usable to us. For example, when injecting strings from the .Values object into the template, we ought to quote these strings. We can do that by calling the `quote` function in the template directive:


```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```


Template functions follow the syntax `functionName arg1 arg2....` In the snippet above, quote `.Values.favorite.drink` calls the `quote` function and passes it a single argument.

Helm has over 60 available functions. Some of them are defined by the [Go template language](https://godoc.org/text/template) itself. Most of the others are part of the [Sprig template library](https://godoc.org/github.com/Masterminds/sprig). 

### Pipelines

In a very similar way we have Pipelines in Linux, we have the same concept in our Helm templates. Pipelines are an efficient way of getting several things done in sequence. Let’s rewrite the above example using a pipeline.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

In this example, instead of calling `quote ARGUMENT`, we inverted the order. We “sent” the argument to the function using a pipeline (`|`): `.Values.favorite.drink | quote`. Using pipelines, we can chain several functions together:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

### Using the `DEFAULT` function

One function frequently used in templates is the `default` function: `default DEFAULT_VALUE GIVEN_VALUE`. This function allows you to specify a default value inside of the template, in case the value is omitted. Let’s use it to modify the drink example above:

```
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

### Operators are functions

Operators are implemented as functions that return a boolean value. To use `eq`, `ne`, `lt`, `gt`, `and`, `or`, `not` etcetera place the operator at the front of the statement followed by its parameters just as you would a function. 

To chain multiple operations together, separate individual functions by surrounding them with parentheses.

```
{{ if and .Values.fooString (eq .Values.fooString "foo") }}
    {{ ... }}
{{ end }}
```

### Flow control

Control structures (called “actions” in template parlance) provide you, the template author, with the ability to control the flow of a template’s generation. Helm’s template language provides the following control structures:

* `if/else` for creating conditional blocks
* `with` to specify a scope
* `range`, which provides a “for each”-style loop

In addition to these, it provides a few actions for declaring and using named template segments:

* `define` declares a new named template inside of your template
* `template` imports a named template
* `block` declares a special kind of fillable template area


#### If/else

```
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

#### Controlling Whitespaces

While we’re looking at conditionals, we should take a quick look at the way whitespace is controlled in templates. Let’s take this example and format it to be a little easier to read:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
    mug: true
  {{end}}
 ``` 

 If we render this template we will generate the following:

 ```
 # Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
    mug: true
```

`mug` is incorrectly indented. Let’s simply out-dent that one line, and re-run:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{end}}
```

When we sent that, we’ll get YAML that is valid, but still looks a little funny:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: telling-chimp-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: true
```

First, the curly brace syntax of template declarations can be modified with special characters to tell the template engine to chomp whitespace. `{{-` (with the dash and space added) indicates that whitespace should be chomped left, while `-}}` means whitespace to the right should be consumed. Be careful! **Newlines are whitespace!**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{- end}}
```

### Using `With`

The next control structure to look at is the with action. This controls variable scoping. Recall that . is a reference to the current scope. So `.Values` tells the template to find the Values object in the current scope.

The syntax for with is similar to a simple if statement:

```
{{ with PIPELINE }}
  # restricted scope
{{ end }}
```

Scopes can be changed. with can allow you to set the current scope (`.`) to a particular object. For example, we’ve been working with `.Values.favorites`. Let’s rewrite our ConfigMap to alter the `.` scope to point to `.Values.favorites`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

# Loops with `Range`

Many programming languages have support for looping using `for` loops, `foreach` loops, or similar functional mechanisms. In Helm’s template language, the way to iterate through a collection is to use the `range` operator.

To start, let’s add a list of pizza toppings to our `values.yaml` file:

```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

We can now do:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```

# `Define` and `Template`

The `define` action allows us to create a named template inside of a template file. Its syntax goes like this:

```
{{ define "MY.NAME" }}
  # body of template here
{{ end }}
```

For example, we can define a template to encapsulate a Kubernetes block of labels:

```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

Now we can embed this template inside of our existing ConfigMap, and then include it with the `template` action:

```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

Conventionally, Helm charts put these templates inside of a partials file, usually `_helpers.tpl`. Let’s move this function there:

```
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

**Template names are global.** As a result of this, if two templates are declared with the same name the last occurrence will be the one that is used. Since templates in subcharts are compiled together with top-level templates, it is best to name your templates with chart specific names.

### The `Include` function

Remember that `template` is an action, and not a function, there is no way to pass the output of a template call to other functions; the data is simply inserted inline.

To work around this case, Helm provides an alternative to `template` that will import the contents of a template into the present pipeline where it can be passed along to other functions in the pipeline: the `include` function.

It is considered preferable to use `include` over `template` in Helm templates simply so that the output formatting can be handled better for YAML documents.

