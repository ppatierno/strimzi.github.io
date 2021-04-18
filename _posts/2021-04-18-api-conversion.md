---
layout: post
title:  "API versions conversion"
date: 2021-04-18
author: jakub_stejskal
---
Upgrade process of Strimzi and Kafka is not trivial.
In Strimzi 0.22 we introduced new API version `v1beta2` which will replace `v1beta1` / `v1alpha1` APIs and make Strimzi ready for Kubernetes 1.22.
This blog post will show you how to perform the conversion and get ready for next Strimzi releases.

<!--more-->

Even with these changes, there are still some disadvantages:
* For upgrade, roll out the new brokers while first using the older `log.message.format.version` or `inter.broker.protocol.version` and only afterwards change to the new versions for message format and inter-broker protocol.
* Downgrade won't be executed when new `log.message.format.version` or `inter.broker.protocol.version` are already used.
  You need to change configuration to use same `log.message.format.version` and `inter.broker.protocol.version` as older Kafka.

You can find further information in our [documentation](https://strimzi.io/docs/operators/0.22.1/full/deploying.html#proc-upgrade-cli-tool-crds-str).

### New API versions

As we stated before, we introduced new API version `v1beta2` for Strimzi resources.
Every user of Strimzi will have to the migration from old `v1beta1` to `v1beta2`.
This migration is not necessarily needed right after upgrade to Strimzi 0.22, but has to be done before upgrade to Strimzi 0.23 or later.
The main reason why Strimzi users had to do this migration are preparations of Strimzi to use [Kubernetes CRD `apiextensions/v1`](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#customresourcedefinition-v122).

Custom resources are edited and controlled using APIs added to Kubernetes by CRDs.
Put another way, CRDs extend the Kubernetes API to allow the creation of custom resources.
CRDs are themselves resources within Kubernetes.
They are installed in a Kubernetes cluster to define the versions of API for the custom resource.
Each version of the custom resource API can define its own schema for that version.
Kubernetes clients, including the Strimzi Operators, access the custom resources served by the Kubernetes API server using a URL path (API path), which includes the API version

In 0.21, we supported custom resource version `v1alpha1` for all custom resources and `v1beta1` for `Kafka`, `KafkaConnect`, `KafkaConnectS2I`, `KafkaMirrorMaker`, `KafkaUser` and `KafkaTopic`.
These versions are currently deprecated and will be dropped in the following Strimzi 0.23.

One another change is, that with new API version we remove some deprecated fields so users need to migrate them to new format.
For instance `v1beta2` supports only new array based listener configuration and new metrics configuration formats from config maps.
In additional some old unused fields like `tlsSidecar` were removed as well.
All changes in field are described in our [documentation](https://strimzi.io/docs/operators/latest/full/deploying.html#proc-upgrade-cli-tool-files-str).

For smooth migration to new CRs and CRDs version we created `api-conversion` tool. _Note that for proper conversion you need to have installed Strimzi 0.22 with all its CRDs._

### API conversion tool

The tool is shipped as part of Strimzi 0.22 in [zip](https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.22.1/api-conversion-0.22.1.zip) and [tar.gz](https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.22.1/api-conversion-0.22.1.tar.gz) archives.
After un-archive, you will find there all libs needed by the tool in `libs` directory and two scripts for running the tool in folder `bin`.
Script `api-conversion.sh` is for linux/mac users and `api-conversion.cmd` is for Windows users.

The tool can operate in two modes:
* convert your resources to `v1beta2`. It can convert resources in yaml files or resource already deployed into Kubernetes cluster.
* convert CRDs to use `v1beta2` as a stored version and use `apiextensions/v1`.

You can use the tool directly from un-archived directory as it's shown in the example bellow.

![api-conversion tool help](/assets/images/posts/2021-04-18-api-conversion.png)

#### convert-file

`convert-file` (`cf`) command basically takes input yaml file passed via option `-f` or `--file` and convert CR to new API version and migrate deprecated fields.
The output will be printed to _stdout_ or you can specify output file with `-o` or `--output` options.
You can also use `--in-file` option to save changes directly to input file.
For more info run `bin/api-conversion.sh cr --help` command.

#### convert-resources

`convert-resources` (`cr`) allow you to convert custom resources directly in your Kubernetes cluster.
You can use `-a`,`--all-namespaces` or `-n`,`--namespace` options to specify namespace where the resources should be converted.
With `-k`,`--kind` the tool will convert only a specific kind of custom resources.
This option can be passed multiple times to convert multiple kinds.
For instance to convert `Kafka` and `KafkaTopic` you just need to pass `--kind Kafka --kind KafkaTopic` to `cr` command.

The last option is `-n`,`--name` where you can specify a name of the custom resource.
This option can be used only with `--namespace` and single `--kind` options.
For more info run `bin/api-conversion.sh cf --help` command.

#### convert-crd

`convert-crd` updates `spec.versions` in the Strimzi CRDs to declare `v1beta2` as the storage API version.
The command also updates existing custom resources where it updates API version and stores them as `v1beta2`.

Once you convert CRDs to API version `apiextensions/v1`, you should use only `v1beta2` version in Strimzi Custom resources.


### Conclusion

In this blog post we take a brief look to API conversion, which will be needed for future Strimzi versions.
You can find further information in our [documentation](https://strimzi.io/docs/operators/0.22.1/full/deploying.html#proc-upgrade-cli-tool-files-str).
