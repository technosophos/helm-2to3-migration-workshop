# Helm 2-to-3 Workshop

This repository contains the instructions and code necessary to do the Oct. 2020 Helm workshop hosted by CNCF.

## Prerequisites


### Kubernetes
You should have a Kubernetes cluster running, and have basically all the privileges.
Do NOT use your production cluster, because we are gonna do some high-risk things.
I recommend Docker Desktop (in Kubernetes mode), Kind, Minikube, or k3s running locally.

## Helm
You should also get a copy of Helm 2. Later you will be replacing this with Helm 3. For my
demos, I have `helm2` aliased to Helm 2, and `helm3` aliased to Helm 3.

If you are just getting Helm 2 setup, you may need to do something like this:

```
$ kubectl apply -f ./tiller-rbac.yaml 
$ helm2 init --service-account tiller
```

## The Workshop

### Install Some Helm 2 Things

We are going to start with Helm 2, install some things, and then perform a migration.

We'll be using a few charts from Bitnami's repository.

```console
$ helm2 repo add bitnami https://charts.bitnami.com/bitnami
```

Next we can install a few charts. We'll do `wordpress` and `drupal` because they are each
reasonably complex. We'll also set a few values just to verify that the migration works.

```console
$ helm2 install bitnami/drupal --set drupalEmail=me@example.com -n drupal 
$ helm2 install bitnami/wordpress --set wordpressEmail me@example.com -n wordpress
```

You can verify that these are installed:

```console
$ helm2 ls
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
drupal          1               Tue Oct 20 14:08:21 2020        DEPLOYED        drupal-9.1.0    9.0.7           default  
wordpress       1               Tue Oct 20 14:09:50 2020        DEPLOYED        wordpress-9.8.0 5.5.1           default  
```

At this point, we have a few things to migrate.

### Install Helm 3 and the Migration Plugin

You will need Helm 3 to continue this workshop.
Check out the [installation guide](https://helm.sh/docs/intro/install/) for instructions on how to get it.
I have installed mine manually, and aliased the `helm3` command to it (`alias helm3=/Users/technosophos/Code/Go/src/helm.sh/helm/bin/helm`).
You don't have to do this.
You can double-check that your `helm` version is 3 using the `helm version` command:

```console
$ helm3 version
version.BuildInfo{Version:"v3.3+unreleased", GitCommit:"caa68e78decd8cdb4e3e6368988f13ec08884070", GitTreeState:"clean", GoVersion:"go1.15.2"}
```

Next, we will install a Helm 3 plugin that allows us to do [Helm 2-to-3 migrations](https://github.com/helm/helm-2to3):

```console
$ helm3 plugin install https://github.com/helm/helm-2to3
Downloading and installing helm-2to3 v0.7.0 ...
https://github.com/helm/helm-2to3/releases/download/v0.7.0/helm-2to3_0.7.0_darwin_amd64.tar.gz
Installed plugin: 2to3
```

We can verify that the plugin was installed:

```console
$ helm3 plugin list
NAME    VERSION DESCRIPTION                                                               
2to3    0.7.0   migrate and cleanup Helm v2 configuration and releases in-place to Helm v3
```

We can read the plugin documentation with the `--help` flag:

```console
$ helm3 2to3 --help
Migrate and Cleanup Helm v2 configuration and releases in-place to Helm v3

Usage:
  2to3 [command]

Available Commands:
  cleanup     cleanup Helm v2 configuration, release data and Tiller deployment
  convert     migrate Helm v2 release in-place to Helm v3
  help        Help about any command
  move        migrate Helm v2 configuration in-place to Helm v3

Flags:
  -h, --help   help for 2to3

Use "2to3 [command] --help" for more information about a command.
```

Now we're ready to do the actual migration.

### (OPTIONAL) Migration Step 1: Moving Your Local Configuration

If you have just installed Helm 3, you will have no configured repositories, plugins, or starters.
But you may have had some of those things configured in your Helm 2 instance.

For example, earlier we configured Helm 2 to use the Bitnami charts repository.
We can use the `helm 2to3 move` command to migrate our configuration.

First, we'll do it in `--dry-run` mode:

```console
$ helm 2to3 move config --dry-run
2020/10/20 14:25:52 NOTE: This is in dry-run mode, the following actions will not be executed.
2020/10/20 14:25:52 Run without --dry-run to take the actions described below:
2020/10/20 14:25:52 
2020/10/20 14:25:52 WARNING: Helm v3 configuration may be overwritten during this operation.
2020/10/20 14:25:52 
[Move config/confirm] Are you sure you want to move the v2 configuration? [y/N]: y
2020/10/20 14:25:57 
Helm v2 configuration will be moved to Helm v3 configuration.
2020/10/20 14:25:57 [Helm 2] Home directory: /Users/technosophos/.helm
2020/10/20 14:25:57 [Helm 3] Config directory: /Users/technosophos/Library/Preferences/helm
2020/10/20 14:25:57 [Helm 3] Data directory: /Users/technosophos/Library/helm
2020/10/20 14:25:57 [Helm 3] Cache directory: /Users/technosophos/Library/Caches/helm
2020/10/20 14:25:57 [Helm 3] Create config folder "/Users/technosophos/Library/Preferences/helm" .
2020/10/20 14:25:57 [Helm 2] repositories file "/Users/technosophos/.helm/repository/repositories.yaml" will copy to [Helm 3] config folder "/Users/technosophos/Library/Preferences/helm/repositories.yaml" .
2020/10/20 14:25:57 [Helm 3] Create cache folder "/Users/technosophos/Library/Caches/helm" .
2020/10/20 14:25:57 [Helm 3] Create data folder "/Users/technosophos/Library/helm" .
2020/10/20 14:25:57 [Helm 2] plugins "/Users/technosophos/.helm/cache/plugins" will copy to [Helm 3] cache folder "/Users/technosophos/Library/Caches/helm/plugins" .
2020/10/20 14:25:57 [Helm 2] plugin symbolic links "/Users/technosophos/.helm/plugins" will copy to [Helm 3] data folder "/Users/technosophos/Library/helm" .
2020/10/20 14:25:57 [Helm 2] starters "/Users/technosophos/.helm/starters" will copy to [Helm 3] data folder "/Users/technosophos/Library/helm/starters" .
```

If you are satisfied with what you see being moved, remove the `--dry-run` flag and try again.

### Migration Step 2: Moving Releases

In Matt Butcher's talk, he shared how releases used to be stored in `kube-system` using one format,
and are now stored in other namespaces using a different format.
In this step, we will run the converter to migrate those releases.

You can start by looking at the releases `helm3` currently knows about:

```console
$ helm3 ls
NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION
```

You should see an empty list. This is because Helm 3 cannot see releases created by Helm 2.
The releases are in different formats (and in different namespaces).

We can use the `2to3` plugin to convert them. Again, we will dry-run first:

```console
$ helm3 2to3 convert wordpress --dry-run
2020/10/20 15:33:51 NOTE: This is in dry-run mode, the following actions will not be executed.
2020/10/20 15:33:51 Run without --dry-run to take the actions described below:
2020/10/20 15:33:51 
2020/10/20 15:33:51 Release "wordpress" will be converted from Helm v2 to Helm v3.
2020/10/20 15:33:51 [Helm 3] Release "wordpress" will be created.
2020/10/20 15:33:51 [Helm 3] ReleaseVersion "wordpress.v1" will be created.
```

Remove the `--dry-run` and perform the actual migration.

```console
$ helm3 2to3 convert wordpress
```

Now running `helm3 ls` should show the Wordpress release:

```console
$ helm3 list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
wordpress       default         1               2020-10-20 20:09:50.7562551 +0000 UTC   deployed        wordpress-9.8.0 5.5.1 
```

Repeat the process for the `drupal` release.

### (OPTIONAL) Migration Step 4: Remove Old Helm 2 Stuff

**WARNING:** This will delete both local and remote data. You may want to skip this step
or just do the `--dry-run` to see how it would work.

As a last step, you can delete old Helm releases and even tiller itself.

To delete old releases, use the `2to3` plugin's `cleanup` command.

```console
$ helm 2to3 cleanup --dry-run
2020/10/20 15:41:36 NOTE: This is in dry-run mode, the following actions will not be executed.
2020/10/20 15:41:36 Run without --dry-run to take the actions described below:
2020/10/20 15:41:36 
WARNING: "Helm v2 Configuration" "Release Data" "Tiller" will be removed. 
This will clean up all releases managed by Helm v2. It will not be possible to restore them if you haven't made a backup of the releases.
Helm v2 may not be usable afterwards.

[Cleanup/confirm] Are you sure you want to cleanup Helm v2 data? [y/N]: y
2020/10/20 15:41:39 
Helm v2 data will be cleaned up.
2020/10/20 15:41:39 [Helm 2] Releases will be deleted.
2020/10/20 15:41:39 [Helm 2] ReleaseVersion "drupal.v1" will be deleted.
2020/10/20 15:41:39 [Helm 2] ReleaseVersion "wordpress.v1" will be deleted.
2020/10/20 15:41:39 [Helm 2] Tiller in "kube-system" namespace will be removed.
2020/10/20 15:41:39 [Helm 2] Home folder "/Users/technosophos/.helm" will be deleted.
```

The output above indicates what will be deleted.

**If you JUST want to uninstall Tiller** and leave the releases alone, you can run `helm2 reset`