# Using Helm

This guide explains the basics of using Helm (and Tiller) to manage
packages on your Kubernetes cluster. It assumes that you have already
[installed](install.md) the Helm client and the Tiller server (typically by `helm
init`).

If you are simply interested in running a few quick commands, you may
wish to begin with the [Quickstart Guide](quickstart.md). This chapter
covers the particulars of Helm commands, and explains how to use Helm.

## Three Big Concepts

A *Chart* is a Helm package. It contains all of the resource definitions
necessary to run an application, tool, or service inside of a Kubernetes
cluster. Think of it like the Kubernetes equivalent of a Homebrew formula,
an Apt dpkg, or a Yum RPM file.

A *Repository* is the place where charts can be collected and shared.
It's like Perl's [CPAN archive](http://www.cpan.org) or the
[Fedora Package Database](https://admin.fedoraproject.org/pkgdb/), but for
Kubernetes packages.

A *Release* is an instance of a chart running in a Kubernetes cluster.
One chart can often be installed many times into the same cluster. And
each time it is installed, a new _release_ is created. Consider a MySQL
chart. If you want two databases running in your cluster, you can
install that chart twice. Each one will have its own _release_, which
will in turn have its own _release name_.

With these concepts in mind, we can now explain Helm like this:

Helm installs _charts_ into Kubernetes, creating a new _release_ for
each installation. And to find new charts, you can search Helm chart
_repositories_.

## 'helm search': Finding Charts

When you first install Helm, it is preconfigured to talk to the official
Kubernetes charts repository. This repository contains a number of
carefully currated and maintained charts. This chart repository is named
`stable` by default.

You can see which charts are available by running `helm search`:

```
$ helm search
NAME                 	VERSION 	DESCRIPTION
stable/drupal   	0.3.2   	One of the most versatile open source content m...
stable/jenkins  	0.1.0   	A Jenkins Helm chart for Kubernetes.
stable/mariadb  	0.5.1   	Chart for MariaDB
stable/mysql    	0.1.0   	Chart for MySQL
...
```

With no filter, `helm search` shows you all of the available charts. You
can narrow down your results by searching with a filter:

```
$ helm search mysql
NAME               	VERSION	DESCRIPTION
stable/mysql  	0.1.0  	Chart for MySQL
stable/mariadb	0.5.1  	Chart for MariaDB
```

Now you will only see the results that match your filter. MySQL is
listed, of course, but so is MariaDB. Why? Because its full description
relates it to MySQL:

Why is
`mariadb` in the list? Because its package description relates it to
MySQL. We can use `helm inspect chart` to see this:

```
$ helm inspect stable/mariadb
Fetched stable/mariadb to mariadb-0.5.1.tgz
description: Chart for MariaDB
engine: gotpl
home: https://mariadb.org
keywords:
- mariadb
- mysql
- database
- sql
...
```

Search is a good way to find available packages. Once you have found a
package you want to install, you can use `helm install` to install it.

## 'helm install': Installing a Package

To install a new package, use the `helm install` command. At its
simplest, it takes only one argument: The name of the chart.

```
$ helm install stable/mariadb
Fetched stable/mariadb-0.3.0 to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
happy-panda
Last Deployed: Wed Sep 28 12:32:28 2016
Namespace: default
Status: DEPLOYED

Resources:
==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         0         0            0           1s

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         1s

==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   1s


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

Now the `mariadb` chart is installed. Note that installing a chart
creates a new _release_ object. The release above is named
`happy-panda`. (If you want to use your own release name, simply use the
`--name` flag on `helm install`.)

During installation, the `helm` client will print useful information
about which resources were created, what the state of the release is,
and also whether there are additional configuration steps you can or
should take.

Helm does not wait until all of the resources are running before it
exits. Many charts require Docker images that are over 600M in size, and
may take a long time to install into the cluster.

To keep track of a release's state, or to re-read configuration
information, you can use `helm status`:

```
$ helm status happy-panda
Last Deployed: Wed Sep 28 12:32:28 2016
Namespace: default
Status: DEPLOYED

Resources:
==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   4m

==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         1         1            1           4m

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         4m


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

The above shows the current state of your release.

### Customizing the Chart Before Installing

Installing the way we have here will only use the default configuration
options for this chart. Many times, you will want to customize the chart
to use your preferred configuration.

To see what options are configurable on a chart, use `helm inspect
values`:

```console
helm inspect values stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
## Bitnami MariaDB image version
## ref: https://hub.docker.com/r/bitnami/mariadb/tags/
##
## Default: none
imageTag: 10.1.14-r3

## Specify a imagePullPolicy
## Default to 'Always' if imageTag is 'latest', else set to 'IfNotPresent'
## ref: http://kubernetes.io/docs/user-guide/images/#pre-pulling-images
##
# imagePullPolicy:

## Specify password for root user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#setting-the-root-password-on-first-run
##
# mariadbRootPassword:

## Create a database user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-user-on-first-run
##
# mariadbUser:
# mariadbPassword:

## Create a database
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-on-first-run
##
# mariadbDatabase:
```

You can then override any of these settings in a YAML formatted file,
and then pass that file during installation.

```console
$ echo 'mariadbUser: user0` > config.yaml
$ helm install -f config.yaml stable/mariadb
```

The above will set the default MariaDB user to `user0`, but accept all
the rest of the defaults for that chart.

### More Installation Methods

The `helm install` command can install from several sources:

- A chart repository (as we've seen above)
- A local chart archive (`helm install foo-0.1.1.tgz`)
- An unpacked chart directory (`helm install path/to/foo`)
- A full URL (`helm install https://example.com/charts/foo-1.2.3.tgz`)

## 'helm upgrade' and 'helm rollback': Upgrading a Release, and Recovering on Failure

When a new version of a chart is released, or when you want to change
the configuration of your release, you can use the `helm upgrade`
command.

An upgrade takes an existing release and upgrades it according to the
information you provide. Because Kubernetes charts can be large and
complex, Helm tries to perform the least invasive upgrade. It will only
update things that have changed since the last release.

```console
$ helm upgrade -f panda.yaml happy-panda stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
happy-panda has been upgraded. Happy Helming!
Last Deployed: Wed Sep 28 12:47:54 2016
Namespace: default
Status: DEPLOYED
...
```

In the above case, the `happy-panda` release is upgraded with the same
chart, but with a new YAML file:

```yaml
mariadbUser: user1
```

We can use `helm get values` to see whether that new setting took
effect.

```console
$ helm get values happy-panda
mariadbUser: user1
```

The `helm get` command is a useful tool for looking at a release in the
cluster. And as we can see above, it shows that our new values from
`panda.yaml` were deployed to the cluster.

Now, if something does not go as planned during a release, it is easy to
roll back to a previous release.

```console
$ helm rollback happy-panda --version 1
```

The above rolls back our happy-panda to its very first release version.
A release version is an incremental revision. Every time an install,
upgrade, or rollback happens, the revision number is incremented by 1.
The first revision number is always 1.

## 'helm delete': Deleting a Release

When it is time to uninstall or delete a release from the cluster, use
the `helm delete` command:

```
$ helm delete happy-panda
```

This will remove the release from the cluster. You can see all of your
currently deployed releases with the `helm list` command:

```
$ helm list
NAME           	VERSION	UPDATED                        	STATUS         	CHART
inky-cat       	1      	Wed Sep 28 12:59:46 2016       	DEPLOYED       	alpine-0.1.0
```

From the output above, we can see that the `happy-panda` release was
deleted.

However, Helm always keeps records of what releases happened. Need to
see the deleted releases? `helm list --deleted` shows those, and `helm
list --all` shows all of the releases (deleted and currently deployed,
as well as releases that failed):

```console
???  helm list --all
NAME           	VERSION	UPDATED                        	STATUS         	CHART
happy-panda   	2      	Wed Sep 28 12:47:54 2016       	DELETED        	mariadb-0.3.0
inky-cat       	1      	Wed Sep 28 12:59:46 2016       	DEPLOYED       	alpine-0.1.0
kindred-angelf 	2      	Tue Sep 27 16:16:10 2016       	DELETED        	alpine-0.1.0
```

Because Helm keeps records of deleted releases, a release name cannot be
re-used. (If you _really_ need to re-use a release name, you can use the
`--replace` flag, but it will simply re-use the existing release and
replace its resources.)

Note that because releases are preserved in this way, you can rollback a
deleted resource, and have it re-activate.

## 'helm repo': Working with Repositories

So far, we've been installing charts only from the `stable` repository.
But you can configure `helm` to use other repositories. Helm provides
several repository tools under the `helm repo` command.

You can see which repositories are configured using `helm repo list`:

```console
$ helm repo list
NAME           	URL
stable         	http://storage.googleapis.com/kubernetes-charts
local          	http://localhost:8879/charts
mumoshu        	https://mumoshu.github.io/charts
```

And new repositories can be added with `helm repo add`:

```console
$ helm repo add dev https://example.com/dev-charts
```

Because chart repositories change frequently, at any point you can make
sure your Helm client is up to date by running `helm repo update`.

## Creating Your Own Charts

The [Chart Development Guide](charts.md) explains how to develop your own
charts. But you can get started quickly by using the `helm create`
command:

```console
$ helm create deis-workflow
Creating deis-workflow
```

Now there is a chart in `./deis-workflow`. You can edit it and create
your own templates.

As you edit your chart, you can validate that it is well-formatted by
running `helm lint`.

When it's time to package the chart up for distribution, you can run the
`helm package` command:

```console
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

And that chart can now easily be installed by `helm install`:

```console
$ helm install ./deis-workflow-0.1.0.tgz
...
```

Charts that are archived can be loaded into chart repositories. See the
documentation for your chart repository server to learn how to upload.

Note: The `stable` repository is managed on the [Kubernetes Charts
GitHub repository](https://github.com/kubernetes/charts). That project
accepts chart source code, and (after audit) packages those for you.

## Conclusion

This chapter has covered the basic usage patterns of the `helm` client,
including searching, installation, upgrading, and deleting. It has also
covered useful utility commands like `helm status`, `helm get`, and
`helm repo`.

For more information on these commands, take a look at Helm's built-in
help: `helm help`.

In the next chapter, we look at the process of developing charts.
