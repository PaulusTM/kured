:exclamation: We are currently in the middle of the move to a different github organisation.
If you have an issue with downloading kured images, helm charts, or other information, please
do not hesitate to contact us in slack (see link below) or in the
[Migration issue](https://github.com/kubereboot/kured/issues/633).

# kured - Kubernetes Reboot Daemon

<img src="https://github.com/kubereboot/kured/raw/main/img/logo.png" align="right"/>

- [kured - Kubernetes Reboot Daemon](#kured---kubernetes-reboot-daemon)
  - [Introduction](#introduction)
  - [Kubernetes & OS Compatibility](#kubernetes--os-compatibility)
  - [Installation](#installation)
  - [Configuration](#configuration)
    - [Reboot Sentinel File & Period](#reboot-sentinel-file--period)
    - [Reboot Sentinel Command](#reboot-sentinel-command)
    - [Setting a schedule](#setting-a-schedule)
    - [Blocking Reboots via Alerts](#blocking-reboots-via-alerts)
    - [Blocking Reboots via Pods](#blocking-reboots-via-pods)
    - [Adding node labels before and after reboots](#adding-node-labels-before-and-after-reboots)
    - [Prometheus Metrics](#prometheus-metrics)
    - [Notifications](#notifications)
    - [Overriding Lock Configuration](#overriding-lock-configuration)
  - [Operation](#operation)
    - [Testing](#testing)
    - [Disabling Reboots](#disabling-reboots)
    - [Manual Unlock](#manual-unlock)
    - [Automatic Unlock](#automatic-unlock)
    - [Delaying Lock Release](#delaying-lock-release)
  - [Building](#building)
  - [Frequently Asked/Anticipated Questions](#frequently-askedanticipated-questions)
    - [Why is there no `latest` tag on Docker Hub?](#why-is-there-no-latest-tag-on-docker-hub)
  - [Getting Help](#getting-help)

## Introduction

Kured (KUbernetes REboot Daemon) is a Kubernetes daemonset that
performs safe automatic node reboots when the need to do so is
indicated by the package management system of the underlying OS.

- Watches for the presence of a reboot sentinel file e.g. `/var/run/reboot-required`
  or the successful run of a sentinel command.
- Utilises a lock in the API server to ensure only one node reboots at
  a time
- Optionally defers reboots in the presence of active Prometheus alerts or selected pods
- Cordons & drains worker nodes before reboot, uncordoning them after

## Kubernetes & OS Compatibility

The daemon image contains versions of `k8s.io/client-go` and
`k8s.io/kubectl` (the binary of `kubectl` in older releases) for the purposes of
maintaining the lock and draining worker nodes. Kubernetes aims to provide
forwards and backwards compatibility of one minor version between client and
server:

| kured  | {k8s.io/,}kubectl | k8s.io/client-go | k8s.io/apimachinery | expected kubernetes compatibility |
| ------ | ----------------- | ---------------- | ------------------- | --------------------------------- |
| main   | 0.24.7            | v0.24.7          | v0.24.7             | 1.23.x, 1.24.x, 1.25.x            |
| 1.10.2 | 0.23.6            | v0.23.6          | v0.23.6             | 1.22.x, 1.23.x, 1.24.x            |
| 1.9.2  | 0.22.4            | v0.22.4          | v0.22.4             | 1.21.x, 1.22.x, 1.23.x            |
| 1.8.1  | 0.21.4            | v0.21.4          | v0.21.4             | 1.20.x, 1.21.x, 1.22.x            |
| 1.7.0  | 0.20.5            | v0.20.5          | v0.20.5             | 1.19.x, 1.20.x, 1.21.x            |
| 1.6.1  | 0.19.4            | v0.19.4          | v0.19.4             | 1.18.x, 1.19.x, 1.20.x            |
| 1.5.1  | 0.18.8            | v0.18.8          | v0.18.8             | 1.17.x, 1.18.x, 1.19.x            |
| 1.4.4  | 1.17.7            | v0.17.0          | v0.17.0             | 1.16.x, 1.17.x, 1.18.x            |
| 1.3.0  | 1.15.10           | v12.0.0          | release-1.15        | 1.15.x, 1.16.x, 1.17.x            |
| 1.2.0  | 1.13.6            | v10.0.0          | release-1.13        | 1.12.x, 1.13.x, 1.14.x            |
| 1.1.0  | 1.12.1            | v9.0.0           | release-1.12        | 1.11.x, 1.12.x, 1.13.x            |
| 1.0.0  | 1.7.6             | v4.0.0           | release-1.7         | 1.6.x, 1.7.x, 1.8.x               |

See the [release notes](https://github.com/kubereboot/kured/releases)
for specific version compatibility information, including which
combination have been formally tested.

Versions >=1.1.0 enter the host mount namespace to invoke
`systemctl reboot`, so should work on any systemd distribution.

## Installation

To obtain a default installation without Prometheus alerting interlock
or Slack notifications:

```console
latest=$(curl -s https://api.github.com/repos/kubereboot/kured/releases | jq -r .[0].tag_name)
kubectl apply -f "https://github.com/kubereboot/kured/releases/download/$latest/kured-$latest-dockerhub.yaml"
```

If you want to customise the installation, download the manifest and
edit it in accordance with the following section before application.

## Configuration

The following arguments can be passed to kured via the daemonset pod template:

```console
Kubernetes Reboot Daemon

Usage:
  kured [flags]

Flags:
      --alert-filter-regexp regexp.Regexp   alert names to ignore when checking for active alerts
      --alert-firing-only                   only consider firing alerts when checking for active alerts
      --annotate-nodes                      if set, the annotations 'weave.works/kured-reboot-in-progress' and 'weave.works/kured-most-recent-reboot-needed' will be given to nodes undergoing kured reboots
      --blocking-pod-selector stringArray   label selector identifying pods whose presence should prevent reboots
      --drain-grace-period int              time in seconds given to each pod to terminate gracefully, if negative, the default value specified in the pod will be used (default -1)
      --drain-timeout duration              timeout after which the drain is aborted (default: 0, infinite time)
      --ds-name string                      name of daemonset on which to place lock (default "kured")
      --ds-namespace string                 namespace containing daemonset on which to place lock (default "kube-system")
      --end-time string                     schedule reboot only before this time of day (default "23:59:59")
      --force-reboot                        force a reboot even if the drain fails or times out
  -h, --help                                help for kured
      --lock-annotation string              annotation in which to record locking node (default "weave.works/kured-node-lock")
      --lock-release-delay duration         delay lock release for this duration (default: 0, disabled)
      --lock-ttl duration                   expire lock annotation after this duration (default: 0, disabled)
      --log-format string                   use text or json log format (default "text")
      --message-template-drain string       message template used to notify about a node being drained (default "Draining node %s")
      --message-template-reboot string      message template used to notify about a node being rebooted (default "Rebooting node %s")
      --message-template-uncordon string    message template used to notify about a node being successfully uncordoned (default "Node %s rebooted & uncordoned successfully!")
      --node-id string                      node name kured runs on, should be passed down from spec.nodeName via KURED_NODE_ID environment variable
      --notify-url string                   notify URL for reboot notifications (cannot use with --slack-hook-url flags)
      --period duration                     sentinel check period (default 1h0m0s)
      --post-reboot-node-labels strings     labels to add to nodes after uncordoning
      --pre-reboot-node-labels strings      labels to add to nodes before cordoning
      --prefer-no-schedule-taint string     Taint name applied during pending node reboot (to prevent receiving additional pods from other rebooting nodes). Disabled by default. Set e.g. to "weave.works/kured-node-reboot" to enable tainting.
      --prometheus-url string               Prometheus instance to probe for active alerts
      --reboot-command string               command to run when a reboot is required (default "/bin/systemctl reboot")
      --reboot-days strings                 schedule reboot on these days (default [su,mo,tu,we,th,fr,sa])
      --reboot-delay duration               delay reboot for this duration (default: 0, disabled)
      --reboot-sentinel string              path to file whose existence triggers the reboot command (default "/var/run/reboot-required")
      --reboot-sentinel-command string      command for which a zero return code will trigger a reboot command
      --skip-wait-for-delete-timeout int    when seconds is greater than zero, skip waiting for the pods whose deletion timestamp is older than N seconds while draining a node
      --slack-channel string                slack channel for reboot notifications
      --slack-hook-url string               slack hook URL for reboot notifications [deprecated in favor of --notify-url]
      --slack-username string               slack username for reboot notifications (default "kured")
      --start-time string                   schedule reboot only after this time of day (default "0:00")
      --time-zone string                    use this timezone for schedule inputs (default "UTC")
```

### Reboot Sentinel File & Period

By default kured checks for the existence of
`/var/run/reboot-required` every sixty minutes; you can override these
values with `--reboot-sentinel` and `--period`. Each replica of the
daemon uses a random offset derived from the period on startup so that
nodes don't all contend for the lock simultaneously.

### Reboot Sentinel Command

Alternatively, a reboot sentinel command can be used. If a reboot
sentinel command is used, the reboot sentinel file presence will be
ignored. When the command exits with code `0`, kured will assume
that a reboot is required.

For example, if you're using RHEL or its derivatives, you can
set the sentinel command to `sh -c "! needs-restarting --reboothint"`
(by default the command will return `1` if a reboot is required,
so we wrap it in `sh -c` and add `!` to negate the return value).

```yaml
configuration:
  rebootSentinelCommand: sh -c "! needs-restarting --reboothint"
```

### Setting a schedule

By default, kured will reboot any time it detects the sentinel, but this
may cause reboots during odd hours.  While service disruption does not
normally occur, anything is possible and operators may want to restrict
reboots to predictable schedules.  Use `--reboot-days`, `--start-time`,
`--end-time`, and `--time-zone` to set a schedule.  For example, business
hours on the west coast USA can be specified with:

```console
  --reboot-days=mon,tue,wed,thu,fri
  --start-time=9am
  --end-time=5pm
  --time-zone=America/Los_Angeles
```

Times can be formatted in numerous ways, including `5pm`, `5:00pm` `17:00`,
and `17`.  `--time-zone` represents a Go `time.Location`, and can be `UTC`,
`Local`, or any entry in the standard Linux tz database.

Note that when using smaller time windows, you should consider shortening
the sentinel check period (`--period`).

### Blocking Reboots via Alerts

You may find it desirable to block automatic node reboots when there
are active alerts - you can do so by providing the URL of your
Prometheus server:

```console
--prometheus-url=http://prometheus.monitoring.svc.cluster.local
```

By default the presence of *any* active (pending or firing) alerts
will block reboots, however you can ignore specific alerts:

```console
--alert-filter-regexp=^(RebootRequired|AnotherBenignAlert|...$
```

You can also only block reboots for firing alerts:

```console
--alert-firing-only=true
```

See the section on Prometheus metrics for an important application of this
filter.

### Blocking Reboots via Pods

You can also block reboots of an *individual node* when specific pods
are scheduled on it:

```console
--blocking-pod-selector=runtime=long,cost=expensive
```

Since label selector strings use commas to express logical 'and', you can
specify this parameter multiple times for 'or':

```console
--blocking-pod-selector=runtime=long,cost=expensive
--blocking-pod-selector=name=temperamental
```

In this case, the presence of either an (appropriately labelled) expensive long
running job or a known temperamental pod on a node will stop it rebooting.

> Try not to abuse this mechanism - it's better to strive for
> restartability where possible. If you do use it, make sure you set
> up a RebootRequired alert as described in the next section so that
> you can intervene manually if reboots are blocked for too long.

### Adding node labels before and after reboots

If you need to add node labels before and after the reboot process, you can use `--pre-reboot-node-labels` and `--post-reboot-node-labels`:

```console
      --pre-reboot-node-labels=zalando=notready
      --post-reboot-node-labels=zalando=ready
```

Labels can be comma-delimited (e.g. `--pre-reboot-node-labels=zalando=notready,thisnode=disabled`) or you can supply the flags multiple times.

Note that label keys specified by these two flags should match. If they do not match, a warning will be generated.

### Prometheus Metrics

Each kured pod exposes a single gauge metric (`:8080/metrics`) that
indicates the presence of the sentinel file:

```console
# HELP kured_reboot_required OS requires reboot due to software updates.
# TYPE kured_reboot_required gauge
kured_reboot_required{node="ip-xxx-xxx-xxx-xxx.ec2.internal"} 0
```

The purpose of this metric is to power an alert which will summon an
operator if the cluster cannot reboot itself automatically for a
prolonged period:

```console
# Alert if a reboot is required for any machines. Acts as a failsafe for the
# reboot daemon, which will not reboot nodes if there are pending alerts save
# this one.
ALERT RebootRequired
  IF          max(kured_reboot_required) != 0
  FOR         24h
  LABELS      { severity="warning" }
  ANNOTATIONS {
    summary = "Machine(s) require being rebooted, and the reboot daemon has failed to do so for 24 hours",
    impact = "Cluster nodes more vulnerable to security exploits. Eventually, no disk space left.",
    description = "Machine(s) require being rebooted, probably due to kernel update.",
  }
```

If you choose to employ such an alert and have configured kured to
probe for active alerts before rebooting, be sure to specify
`--alert-filter-regexp=^RebootRequired$` to avoid deadlock!

### Notifications

When you specify a formatted URL using `--notify-url`, kured will notify
about draining and rebooting nodes across a list of technologies.

![Notification](img/slack-notification.png)

Alternatively you can use the `--message-template-drain`, `--message-template-reboot` and `--message-template-uncordon` to customize the text of the message, e.g.

```cli
--message-template-drain="Draining node %s part of *my-cluster* in region *xyz*"
```

Here is the syntax:

- slack:           `slack://tokenA/tokenB/tokenC`

    (`slack://<USERNAME>@tokenA/tokenB/tokenC` - in case you want to [respect username](https://github.com/kubereboot/kured/issues/482))

    (`--slack-hook-url` is deprecated but possible to use)

  For the new slack App integration, use:\
    `slack://xoxb:123456789012-1234567890123-4mt0t4l1YL3g1T5L4cK70k3N@<CHANNEL_NAME>?botname=<BOTNAME>`\
    for more information, [look here](https://containrrr.dev/shoutrrr/v0.5/services/slack/#examples)

- rocketchat:      `rocketchat://[username@]rocketchat-host/token[/channel|@recipient]`

- teams:           `teams://group@tenant/altId/groupOwner?host=organization.webhook.office.com`

- Email:           `smtp://username:password@host:port/?fromAddress=fromAddress&toAddresses=recipient1[,recipient2,...]`

More details here: [containrrr.dev/shoutrrr/v0.5/services/overview](https://containrrr.dev/shoutrrr/v0.5/services/overview)

### Overriding Lock Configuration

The `--ds-name` and `--ds-namespace` arguments should match the name and
namespace of the daemonset used to deploy the reboot daemon - the locking is
implemented by means of an annotation on this resource. The defaults match
the daemonset YAML provided in the repository.

Similarly `--lock-annotation` can be used to change the name of the
annotation kured will use to store the lock, but the default is almost
certainly safe.

## Operation

The example commands in this section assume that you have not
overriden the default lock annotation, daemonset name or namespace;
if you have, you will have to adjust the commands accordingly.

### Testing

You can test your configuration by provoking a reboot on a node:

```console
sudo touch /var/run/reboot-required
```

### Disabling Reboots

If you need to temporarily stop kured from rebooting any nodes, you
can take the lock manually:

```console
kubectl -n kube-system annotate ds kured weave.works/kured-node-lock='{"nodeID":"manual"}'
```

Don't forget to release it afterwards!

### Manual Unlock

In exceptional circumstances, such as a node experiencing a permanent
failure whilst rebooting, manual intervention may be required to
remove the cluster lock:

```console
kubectl -n kube-system annotate ds kured weave.works/kured-node-lock-
```

> NB the `-` at the end of the command is important - it instructs
> `kubectl` to remove that annotation entirely.

### Automatic Unlock

In exceptional circumstances (especially when used with cluster-autoscaler) a node
which holds lock might be killed thus annotation will stay there for ever.

Using `--lock-ttl=30m` will allow other nodes to take over if TTL has expired (in this case 30min) and continue reboot process.

### Delaying Lock Release

Using `--lock-release-delay=30m` will cause nodes to hold the lock for the specified time frame (in this case 30min) before it is released and the reboot process continues. This can be used to throttle reboots across the cluster.

## Building

Kured now uses [Go
Modules](https://github.com/golang/go/wiki/Modules), so build
instructions vary depending on where you have checked out the
repository:

**Building outside $GOPATH:**

```console
make
```

**Building inside $GOPATH:**

```console
GO111MODULE=on make
```

You can find the current preferred version of Golang in the [go.mod file](go.mod).

If you are interested in contributing code to kured, please take a look at
our [development][development] docs.

[development]: DEVELOPMENT.md

## Frequently Asked/Anticipated Questions

### Why is there no `latest` tag on Docker Hub?

Use of `latest` for production deployments is bad practice - see
[here](https://kubernetes.io/docs/concepts/configuration/overview) for
details. The manifest on `main` refers to `latest` for local
development testing with minikube only; for production use choose a
versioned manifest from the [release page](https://github.com/kubereboot/kured/releases/).

## Getting Help

If you have any questions about, feedback for or problems with `kured`:

- Invite yourself to the <a href="https://slack.cncf.io/" target="_blank">CNCF Slack</a>.
- Ask a question on the [#kured](https://cloud-native.slack.com/archives/kured) slack channel.
- [File an issue](https://github.com/kubereboot/kured/issues/new).
- Join us in [our monthly meeting](https://docs.google.com/document/d/1bsHTjHhqaaZ7yJnXF6W8c89UB_yn-OoSZEmDnIP34n8/edit#),
  every first Wednesday of the month at 16:00 UTC.

We follow the [CNCF Code of Conduct](CODE_OF_CONDUCT.md).

Your feedback is always welcome!

## Trademarks

**Kured is a [Cloud Native Computing Foundation](https://cncf.io/) Sandbox project.**

![Cloud Native Computing Foundation logo](img/cncf-color.png)

The Linux Foundation® (TLF) has registered trademarks and uses trademarks. For a list of TLF trademarks, see [Trademark Usage](https://www.linuxfoundation.org/trademark-usage/).
