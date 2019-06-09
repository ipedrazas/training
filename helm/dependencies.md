# Chart Dependencies

In Helm, one chart may depend on any number of other charts.
These dependencies can be dynamically linked through the `requirements.yaml`
file or brought in to the `charts/` directory and managed manually.

Although manually managing your dependencies has a few advantages some teams need,
the preferred method of declaring dependencies is by using a
`requirements.yaml` file inside of your chart.


### Managing Dependencies with `requirements.yaml`

A `requirements.yaml` file is a simple file for listing your
dependencies.

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```

- The `name` field is the name of the chart you want.
- The `version` field is the version of the chart you want.
- The `repository` field is the full URL to the chart repository. Note
  that you must also use `helm repo add` to add that repo locally.

Once you have a dependencies file, you can run `helm dependency update`
and it will use your dependency file to download all the specified
charts into your `charts/` directory for you.

```console
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete.
Saving 2 charts
Downloading apache from repo http://example.com/charts
Downloading mysql from repo http://another.example.com/charts
```

When `helm dependency update` retrieves charts, it will store them as
chart archives in the `charts/` directory. So for the example above, one
would expect to see the following files in the charts directory:

```
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

Managing charts with `requirements.yaml` is a good way to easily keep
charts updated, and also share requirements information throughout a
team.