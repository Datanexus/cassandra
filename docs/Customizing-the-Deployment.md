# Customizing the deployment
As was mentioned in [this document](Deployment-Scenarios.md), there are a number of ways that the deployment process can be customized to suit your needs. In this document, we briefly discuss the files used to define the variables that are used during the playbook run, their intended purpose, and how you can override the default values provided in these files in your own playbook runs.

## Files used during the playbook run
There are two files that are used to define the variables and tags used in all of our playbooks; for the `cassandra` repository those files are as follows:

### The [vars/cassandra.yml](../vars/cassandra.yml) file
This **variables file** defines a reasonable set of defaults for the variables used during all [provision-cassandra.yml](../provision-cassandra.yml) playbook runs, regardless of the target environment. Most of the variables defined in this file do not need to be modified from one playbook run to the next, although the values defined here can be overridden by redefining them in the **configuration file** that is used during that playbook run. We will discuss our recommendations for customizing the values found in this file [later in this document](#customization-options).

### The [config.yml](../config.yml) file

This file is the **default configuration file** used during a [provision-cassandra.yml](../provision-cassandra.yml) playbook run if there is no other configuration file provided at runtime. The intent of the configuration file is to provide values for the variables that we expect to change based on the target environment. This file is maintained under version control as part of the `cassandra` repository, and the current released version of this file looks like this:

```yaml
# (c) 2017 DataNexus Inc.  All Rights Reserved
#
# Defaults used if a configuration file is not passed into the playbook run
# using the `config_file` extra variable; deploys to an AWS environment
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
# network configuration used when creating VMs
cidr_block: '10.10.0.0/16'
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# AWS-specific parameters
region: us-west-2
# node type and image ID to deploy; note that the image ID is optional for AWS
# deployments but mandatory for OSP deployments; if left undefined in an AWS
# deployment then then the playbook will search for an appropriate image to use
type: 't2.large'
#image: 'ami-f4533694'
# the default data and root volume sizes
root_volume: 11
data_volume: 40
```

As you can quite clearly see, this default configuration file defines values for the `tenant`, `project`, `domain`, and `cluster` variables that are used during the playbook run as well as a default value for the `cloud`, `region`, and `type` variables that ensure that the playbook runs that use this default configuration file will deploy Cassandra to a set of VMs running in an AWS environment. The `cloud`, `region`, and `type` variables are included in this file rather than the [vars/cassandra.yml](../vars/cassandra.yml) file because they are more only needed when deploying to an AWS or OpenStack environment.

In short, the variables defined in this default configuration file are those that are more likely to change from one playbook run to the next. In the file shown above, you can see that this default configuration file will use the defined `tenant`, `project`, `domain`, `cluster`, and `application` values to search for (and construct if necessary) a set of `t2.large` machines in the AWS, US West (Oregon) region for use in the playbook run.

## Customization options
There are three basic mechanisms for customing a given playbook run:

* Making changes to the values defined in the **variables file** (by editing the [vars/cassandra.yml](../vars/cassandra.yml) file directly)
* Making changes to the values defined in the **default configuration file** (by editing the [config.yml](../config.yml) file directly)
* Constructing your own configuration file and passing that file into the playbook run via the `config_file` variable

While the first two options may seem attractive at first, the files in question are maintained under version control in the `cassandra` repository. As such, edits to these files may create merge conflicts down the line when you attempt to update your clone of the main `cassandra` repository to pull in the latest updates and bug fixes. To avoid this problem, our recommendation is that you add any variables that you would like to override the value for (whether those variables are defined in the [vars/cassandra.yml](../vars/cassandra.yml) file or the [config.yml](../config.yml) file) to a new, custom configuration file of your own and then pass that new configuration file into the playbook at runtime (using the `config_file` extra variable). For example, if you wanted to increase the heap size for your Cassandra VMs to 8GB (a default heap size of 4GB is defined in the [vars/cassandra.yml](../vars/cassandra.yml) file using the `cassandra_jvm_heaps_size` variable), then you might setup a custom configuration file that looks something like the following:

```yaml
$ cat config-aws-custom.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when creating VMs
cidr_block: '10.10.0.0/16'
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# AWS-specific parameters
region: us-west-2
# node type and image ID to deploy
type: 't2.xlarge'
image: 'ami-f4533694'
# the default data and root volume sizes
root_volume: 11
data_volume: 40
# and add a few Cassandra-specific configuration parameters
cassandra_jvm_heaps_size: "8G"
cassandra_disk_optimization_strategy: spinning
```

Note that we have redefined many of the same parameters that are defined in the default configuration file (the [config.yml](../config.yml) file), since we will be replacing the that default configuration file with our new, custom configuration file during the playbook run. Also note that when we increased the heap size (by redefining the `cassandra_jvm_heaps_size` variable in the custom configuration file shown above), we also had to change the `type` that we are using, since the default `type` defined in our default configuration file (`t2.large`) does not have enough memory to start up a Java process with an 8GB heap.

With our new, custom configuration constructed, we could then deploy a cluster using this custom configuration by simply passing that configuration file into our playbook via the `config_file` variable (using either a full or relative path to that file):

```bash
$ AWS_PROFILE=datanexus_west ./provision-cassandra.yml -e "{ \
    config_file: config-aws-custom.yml \
}"
```

Any of the variables that are set in the [vars/cassandra.yml](../vars/cassandra.yml) and [config.yml](../config.yml) files in this repository can be overridden in this manner. Just keep in mind that any variables defined in the default configuration file (the [config.yml](../config.yml) file) must always be redefined in any custom configuration file that you create (unless, of course, you don't need those variables for the deployment that you are performing).

## A short note on customizing Cassandra
We would like to close with a brief discussion of how the playbook in this repository modifies the default configuration file provided as part of the Cassandra distribution and the discuss how you can take advantage of that process to customize a Cassandra cluster configuration using a custom configuration file.

In short, if there is a parameter that appears in the `conf/cassandra.yaml` file under the top-level directory of the Cassandra distribution (the `concurrent_reads` parameter, for example, which defaults to `32` in the configuration file that is included in the Cassandra distribution), then you can easily override the default value defined for that variable by defining an Ansible variable in your custom configuration file that has the same name but with a `cassandra_` prefix (in this example, you would define a value for the `cassandra_concurrent_reads` variable and that value would be used instead of the default value defined in the configuration file included in the distribution). If you were watching, we actually took advantage of this feature to define a new value for the `disk_optimization_strategy` configuration parameter used in our Cassandra cluster configuration in the custom configuration file that was shown previously:

```yaml
$ cat config-aws-custom.yml
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when creating VMs
cidr_block: '10.10.0.0/16'
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# AWS-specific parameters
region: us-west-2
# node type and image ID to deploy
type: 't2.xlarge'
image: 'ami-f4533694'
# the default data and root volume sizes
root_volume: 11
data_volume: 40
# and add a few Cassandra-specific configuration parameters
cassandra_jvm_heaps_size: "8G"
cassandra_disk_optimization_strategy: spinning
```

This custom configuration file sets up the Cassandra cluster that is being deployed so that the disk reads are optimized for a `spinning` disk (the default is to optimize for an `ssd` instead).

Once you understand this pattern, it is quite simple to construct your own custom configuration files to optimize the Cassandra configuration you are using for the platform you are deploying it to. Most parameters will never need to be reset, but to truly optimize the performance of your Cassandra cluster on the platform you are deploying it to there are a few tweaks that you might have to make. This mechanism should provide a quick and easy way to perform such optimizations by customizing your Cassandra configuration based on the platform you are deploying it to.
