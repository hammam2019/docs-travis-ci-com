---
title: Travis CI Enterprise Troubleshooting Guide
layout: en_enterprise
redirect_from:
  - /user/enterprise/operations-manual/
---

This document provides guidelines and suggestions for troubleshooting your Travis CI Enterprise installation. Each topic contains a common problem and strategies for solving it. If you have questions about a specific scenario or have an issue that is not covered then please contact us at [enterprise@travis-ci.com](mailto:enterprise@travis-ci.com) for assistance.

Throughout this document we'll be using the following terms to refer to the two components of your Travis CI Enterprise installation:

- `Platform machine`: The instance that runs the main Travis components, including the web frontend.
- `Worker machine`: The worker instance(s) that process and run the builds.

> Please note that this guide is geared towards non-High Availability (HA) setups. Please contact us at [enterprise@travis-ci.com](mailto:enterprise@travis-ci.com) if you require support for your HA setup.

## Builds are not starting

### The problem

In the Travis CI Web UI you see no builds are starting. The builds either have no visible state or have a state of `queued`. Canceling and restarting builds has no effect.

### Strategies

There are a few different potential approaches which may help get builds running again. Please try each one in order.

#### Connection to RabbitMQ was lost

The Enterprise Platform uses RabbitMQ to communicate with worker machine(s) in order to process builds. In certain circumstances it is possible for the the worker machine(s) to lose connection with RabbitMQ and therefore become unable to process builds successfully. This is a known problem and we're working to deliver a permanent solution.

In the meantime, to return everything back to a normal working state you can manually restart the worker machine(s). This can be done by connecting to the worker(s) via `ssh` and running the following command:

```bash
$ sudo shutdown -r 0
```

This will immediately restart the machine. The program that processes worker builds (`travis-worker`) is configured to start automatically on system startup and should reestablish its connection to RabbitMQ.

#### Configuration Issues

Please check if the worker machine has all relevant configuration in order. To do so, please use ssh to login to the worker machine(s) and open `/etc/default/travis-enterprise`. This is the main configuration file `travis-worker` uses to connect to the platform machine.

Here is an example:

```
# Default ENV variables for Travis Enterprise
# Uncomment and set, then restart `travis-worker` for
# them to take effect.

export TRAVIS_ENTERPRISE_BUILD_ENDPOINT="__build__"
export TRAVIS_ENTERPRISE_HOST="travisci.example.com"
export TRAVIS_ENTERPRISE_SECURITY_TOKEN="abc12345"
# export TRAVIS_WORKER_DOCKER_PRIVILEGED="true"
```

The relevant variables are `TRAVIS_ENTERPRISE_HOST` and `TRAVIS_ENTERPRISE_SECURITY_TOKEN`. The former needs to contain your primary domain you use to access the Travis CI Enterprise Web UI. The `travis-worker` process uses this domain to reach the platform machine. The value of the latter needs to match the `RabbitMQ Password` on `https://your-travis-ci-enterprise-domain:8800/settings`.

If you have made changes to this file, please run the following so they take effect:

```bash
$ sudo restart travis-worker
```

#### Ports are not open in Security groups / Firewall

A source for the problem could be that the worker machine is not able to communicate with the platform machine.

Here we're distinguishing between an AWS EC2 installation and an installation running on other hardware. For the former, security groups need to be configured per machine. To do so, please follow our installation instructions [here](/user/enterprise/setting-up-travis-ci-enterprise/#1-setting-up-enterprise-platform-virtual-machine). If you're not using AWS EC2, please make sure that the ports listed [in the docs](/user/enterprise/setting-up-travis-ci-enterprise/#1-setting-up-enterprise-platform-virtual-machine) are open in your firewall.

#### Docker Version Mismatch

This issue sometimes occurs after maintenance on workers that were originally installed before November 2017 or on systems running a `docker version` before `17.06.2-ce`. When this happens, the `/var/log/upstart/travis-worker.log` file will contain the following line: `Error response from daemon:client and server don't have same version`. For this issue, we recommend [re-installing each worker from scratch](/user/enterprise/setting-up-travis-ci-enterprise/#2-setting-up-the-enterprise-worker-virtual-machine) on a fresh instance. Please note: the default build environment images will be pulled and you may need to apply customizations again as well.

#### You are running Enterprise v2.2 or higher

By default the Enterprise Platform v2.2 or higher will attempt to route builds to the `builds.trusty` queue. This could lead to build issues if you are not running a Trusty worker to process those builds or if you are targeting a different distribution (e.g. `xenial`).

To address this, either:

- Ensure that you have installed a Trusty worker on a new virtual machine instance: [Trusty installation guide](/user/enterprise/trusty/)
- Override the default queuing behavior to specify a new queue. To override the default queue you must access the Admin Dashboard at `https://<your-travis-ci-enterprise-domain>:8800/settings#override_default_dist_enable` and toggle the the "Override Default Build Environment" button. This will allow you to specify the new default based on your needs and the workers that you have available.

## Enterprise container fails to start due to 'context deadline exceeded' error

### The problem

After a fresh installation or configuration change the Enterprise container doesn't start and the following error is visible in the admin dashboard found at `https://<your-travis-ci-enterprise-domain>:8800/dashboard`:

```
Ready state command canceled: context deadline exceeded
```

### Strategies

#### GitHub OAuth app configuration

The above mentioned error can be caused by a configuration mismatch in [the GitHub OAuth Application](/user/enterprise/setting-up-travis-ci-enterprise/#prerequisites). Please check that _both_ website and callback URL contain the Travis CI Enterprise's hostname. If you have discovered a mismatch here, please restart the Travis container from within the admin dashboard.


## travis-worker on Ubuntu 16.04 does not start

### The problem

`travis-worker` was installed on a fresh installation of Ubuntu 16.04 (Xenial) with no issues. However, the command `sudo systemctl status travis-worker` shows that it is not running.

In addition, the command `sudo journalctl -u travis-worker` contains the following error:

```
/usr/local/bin/travis-worker-wrapper: line 20: /var/tmp/travis-run.d/travis-worker: No such file or directory
```

### Strategy

One possible reason that travis-worker is not running is that `systemctl` cannot create a temporary directory for environment files. To fix this, please create the directory `/var/tmp/travis-run.d/travis-worker` and assign write permissions via:

```sh
$ mkdir -p /var/tmp/travis-run.d/
$ chown -R travis:travis /var/tmp/travis-run.d/
```

## Builds fail with curl certificate errors

### The problem

A build fails with a long `curl` error message similar to:

```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

This can have various causes, including an automatic nvm update or a caching error.

### Strategy

This error is most likely caused by a self-signed certificate. During the build, the worker container attempts to fetch different files from the platform machine. If the server was originally provisioned with a self-signed certificate, curl doesn't trust this certificate and therefore fails. While we're working on resolving this in a permanent way, currently the only solution is to install a certificate issued by a trusted Certificate Authority (CA). This can be a free Let's Encrypt certificate or any other trusted CA of your choice. We have a section in our [Platform Administration Tips](/user/enterprise/platform-tips/#use-a-lets-encrypt-ssl-certificate) page that walks you through the installation process using Let's Encrypt as an example.

## User accounts are stuck in syncing state

### The problem

One or more user accounts are stuck in the `is_syncing = true` state. When you query the database, the number of users which are currently syncing does not decrease over the time. Example:

```sql
travis_production=> select count(*) from users where is_syncing=true;
 count
-------
  1027
(1 row)
```

### Strategy

Log into the platform machine via ssh. You can reset the `is_syncing` flag for user accounts that are stuck by running:

```bash
$ travis console
>> User.where(is_syncing: true).count
>> ActiveRecord::Base.connection.execute('set statement_timeout to 60000')
>> User.update_all(is_syncing: false)
```

It can happen that organizations are also stuck in the syncing state. Since an organization itself does not have a `is_syncing` flag, all users that do belong to it have to be successfully synced.
