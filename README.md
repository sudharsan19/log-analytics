# JFrog Log Analytics

This project integrates Jfrog logs into various log analytic providers through the use of fluentd as a common logging agent.

The goal of this project is to provide Jfrog customers with robust log analytic solutions that they could use to monitor the Jfrog unified platform microservices.

## Table of Contents

   * [Fluentd](#fluentd)
     * [Root Installation](#root-installation)
     * [User Installation](#user-installation)
     * [Logger Agent](#logger-agent)
     * [Config Files](#config-files)
     * [Running As A Service](#running-as-a-service)
     * [Running As A Service As A Regular User](#running-as-a-service-as-a-regular-user)
   * [Splunk](#splunk)
     * [Splunk Config](#splunk-config)
     * [Splunk Demo](#splunk-demo)
   * [Elasticsearch - Kibana](elastic-fluentd-kibana/README.md)
   * [Prometheus-Grafana](prometheus-fluentd-grafana/README.md)
   * [Datadog](datadog/README.md)
   * [Tools](#tools)
   * [Contributing](#contributing)
   * [Versioning](#versioning)
   * [Contact](#contact)

## Fluentd 

Fluentd is a required component to use this integration.

Fluentd has an logger agent called td-agent which will be required to be installed into each node you wish to monitor logs on.

For more details on how to install Fluentd into your environment please visit:

[Fluentd installation guide](https://docs.fluentd.org/installation)

#### Root Installation

Install the td-agent agent on Redhat UBI we need to run the below command:

```
$ curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent3.sh | sh
```

Root access will be required as this will use yum to install td-agent

#### User Installation

Non-root users to make life easier for we have provided a tar.gz containing everything you need to run fluentd.

Follow these steps:

* Download the tar from this Github: fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz

* Explode the tar into /opt/jfrog/artifactory and run:

``` 
/opt/jfrog/artifactory/fluentd-1.11.0-linux-x86_64/fluentd <conf_file>
```

Updating fluentd to future releases is simple as well:

``` 
/opt/jfrog/artifactory/fluentd-1.11.0-linux-x86_64/lib/ruby/bin/gem install fluentd
```

Adding any fluentd plugins like Datadog as works in the same fashion:

``` 
/opt/jfrog/artifactory/fluentd-1.11.0-linux-x86_64/lib/ruby/bin/gem install fluent-plugin-datadog
```

#### Logger Agent

* Package Manager installations only.

The default configuration file for td-agent is located at:

```
/etc/td-agent/td-agent.conf
```

You should update this configuration file and run td-agent as a service.

If you wish to only run td-agent against a test configuration file you can also run:

```
td-agent -c fluentd.conf
```

Once td-agent has been installed on an Artifactory or Xray node you will also need to install the relevant plugin if you are using Splunk or Datadog:

Splunk:
```
td-agent-gem install fluent-plugin-splunk-enterprise
```

Datadog:
``` 
td-agent-gem install fluent-plugin-datadog
```

Elastic:
``` 
td-agent-gem install fluent-plugin-elasticsearch
```

#### Config Files

Fluentd requires configuration file to know which logs to tail and how to ship them to the correct log provider.

Our configurations are saved into each log provider's folder.

###### Splunk:

[Artifactory 7.x+](https://github.com/jfrog/log-analytics/blob/master/splunk/fluent.conf.rt)

[Xray 3.x+](https://github.com/jfrog/log-analytics/blob/master/splunk/fluent.conf.xray)

[Artifactory 6.x](https://github.com/jfrog/log-analytics/blob/master/splunk/fluent.conf.rt6)

###### EFK:

[Artifactory 7.x+](https://github.com/jfrog/log-analytics/blob/master/elastic-fluentd-kibana/fluent.conf.rt)

[Xray 3.x+](https://github.com/jfrog/log-analytics/blob/master/elastic-fluentd-kibana/fluent.conf.xray)

[Artifactory 6.x](https://github.com/jfrog/log-analytics/blob/master/elastic-fluentd-kibana/fluent.conf.rt6)

We will need to store these configurations into the correct location per our installer type.

#### Running as a service

By default td-agent will run as the td-agent user however the JFrog logs folder only has file permissions for the artifactory or xray user.

* Fix the group and file permissions issue in Artifactory as root:

``` 
usermod -a -G artifactory td-agent
chmod 0770 /opt/jfrog/artifactory/var/log/*
```

* Fix the group and file permissions issue in Xray as root:

``` 
usermod -a -G xray td-agent
chmod 0770 /opt/jfrog/xray/var/log/*
```

* Run td-agent and check it's status

```
systemctl start td-agent
systemctl status td-agent
```

#### Running as a service as a regular user

Using systemd:

* Create a service unit configuration file

```
mkdir -p ~/.config/systemd/user/
touch ~/.config/systemd/user/jfrogfluentd.service
```

* Copy paste below snippet, update the configuration file location, and save into the file:

```
[Unit]
Description=JFrog_Fluentd

[Service]
ExecStart=/opt/jfrog/artifactory/fluentd-1.11.0-linux-x86_64/fluentd <conf_file>
Restart=always

[Install]
WantedBy=graphical.target
See man systemd.service and man systemd.unit for more options.
```

* Enable service in userspace

``` 
systemctl --user enable jfrogfluentd
```

* Start it and check it's status

```
systemctl --user start jfrogfluentd
systemctl --user status jfrogfluentd
```

* Enjoy!


## Splunk

### Splunk Config

Fluentd setup must be completed prior to Splunk.

To use the integration an administrator of Splunk will need to install the Jfrog Logs Application into Splunk from Splunkbase.

The next step will be to configure the Splunk HEC.

Our integration uses the [Splunk HEC](https://dev.splunk.com/enterprise/docs/dataapps/httpeventcollector/) to send data to Splunk.

Users will need to configure the HEC to accept data (enabled) and also create a new token. Save this token.

Users will also need to specify the HEC_HOST, HEC_PORT and if ssl is enabled the ca_file to be used.

``` 
<match jfrog.**>
  @type splunk_hec
  host HEC_HOST
  port HEC_PORT
  token HEC_TOKEN
  format json
  # buffered output parameter
  flush_interval 10s
  # ssl parameter
  use_ssl true
  ca_file /path/to/ca.pem
</match>
```

### Splunk Demo

To run this integration for Splunk users can create a Splunk instance with the correct ports open in Kubernetes by applying the yaml file:

``` 
kubectl apply -f splunk/splunk.yaml
```

This will create a new Splunk instance you can use for a demo to send your Jfrog logs over to.

Once they have a Splunk up for demo purposes they will need to configure the HEC and then update fluent config files with the relevant parameters for HEC_HOST, HEC_PORT, & HEC_TOKEN.

They can now access Splunk to view the Jfrog dashboard as new data comes in.

## Tools
* [Fluentd](https://www.fluentd.org) - Fluentd Logging Aggregator/Agent
* [Splunk](https://www.splunk.com/) - Splunk Logging Platform
* [Splunk HEC](https://dev.splunk.com/enterprise/docs/dataapps/httpeventcollector/) - Splunk HEC used to upload data into Splunk
* [Elasticsearch](https://www.elastic.co/) - Elastic search log data platform
* [Kibana](https://www.elastic.co/kibana) - Elastic search visualization layer

## Contributing
Please read CONTRIBUTING.md for details on our code of conduct, and the process for submitting pull requests to us.

## Versioning
We use [SemVer](http://semver.org/) for versioning. For the versions available, see the [tags on this repository](https://github.com/your/project/tags).

## Contact
* Github
