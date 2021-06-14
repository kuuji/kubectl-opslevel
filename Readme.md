<p align="center">
    <a href="https://github.com/OpsLevel/kubectl-opslevel/blob/main/LICENSE" alt="License">
        <img src="https://img.shields.io/github/license/OpsLevel/kubectl-opslevel.svg" /></a>
    <a href="http://golang.org" alt="Made With Go">
        <img src="https://img.shields.io/github/go-mod/go-version/OpsLevel/kubectl-opslevel?filename=src%2Fgo.mod" /></a>
    <a href="https://GitHub.com/OpsLevel/kubectl-opslevel/releases/" alt="Release">
        <img src="https://img.shields.io/github/v/release/OpsLevel/kubectl-opslevel" /></a>  
    <a href="https://GitHub.com/OpsLevel/kubectl-opslevel/issues/" alt="Issues">
        <img src="https://img.shields.io/github/issues/OpsLevel/kubectl-opslevel.svg" /></a>  
    <a href="https://github.com/OpsLevel/kubectl-opslevel/graphs/contributors" alt="Contributors">
        <img src="https://img.shields.io/github/contributors/OpsLevel/kubectl-opslevel" /></a>
    <a href="https://github.com/OpsLevel/kubectl-opslevel/pulse" alt="Activity">
        <img src="https://img.shields.io/github/commit-activity/m/OpsLevel/kubectl-opslevel" /></a>
    <a href="https://dependabot.com/" alt="Dependabot">
        <img src="https://badgen.net/badge/Dependabot/enabled/green?icon=dependabot" /></a>
</p>

`kubectl-opslevel` is a command line tool that enables you to import & reconcile services with [OpsLevel](https://www.opslevel.com/) from your Kubernetes clusters.  You can also run this tool inside your Kubernetes cluster as a job to reconcile the data with OpsLevel periodically.  If you opt for this please read our [service aliases](#aliases) section as we use these to properly find and reconcile the data so it is important you choose something unique.

### Quickstart

Follow the [installation](#installation) instructions before running the below commands

```bash
# Generate a config file
kubectl opslevel config sample > opslevel-k8s.yaml

# Like Terraform, generate a preview of data from your Kubernetes cluster
# NOTE: this step does not validate any of the data with OpsLevel
kubectl opslevel service preview

# Import (and reconcile) the found data with your OpsLevel account
 OL_APITOKEN=XXXX kubectl opslevel service import
```

![](demo.gif)


<blockquote>This tool is still in beta.  It's sufficently stable for production use and has successfully imported & reconciled multiple OpsLevel accounts</blockquote>

#### Current Sample Configuration

```yaml
version: "1.0.0"
service:
  import:
    - selector: # This limits what data we look at in Kubernetes
        kind: deployment # supported options ["deployment", "statefulset", "daemonset", "service", "ingress", "job", "cronjob", "configmap", "secret"]
        namespace: 
          include: # if set only these namespaces will be inspected
            - ""
          exclude: # if set these namespaces will be excluded from inspection
            - "kube-system"
            - "local-path-storage"
        labels: {}
      opslevel: # This is how you map your kubernetes data to opslevel service
        name: .metadata.name
        description: .metadata.annotations."opslevel.com/description"
        owner: .metadata.annotations."opslevel.com/owner"
        lifecycle: .metadata.annotations."opslevel.com/lifecycle"
        tier: .metadata.annotations."opslevel.com/tier"
        product: .metadata.annotations."opslevel.com/product"
        language: .metadata.annotations."opslevel.com/language"
        framework: .metadata.annotations."opslevel.com/framework"
        aliases: # This are how we identify the services again during reconciliation - please make sure they are very unique
          - '"k8s:\(.metadata.name)-\(.metadata.namespace)"'
        tags:
          assign: # tag with the same key name but with a different value will be updated on the service
            - '{"imported": "kubectl-opslevel"}'
            # find annoations with format: opslevel.com/tags.<key name>: <value>
            - '.metadata.annotations | to_entries |  map(select(.key | startswith("opslevel.com/tags"))) | map({(.key | split(".")[2]): .value})'
            - .metadata.labels
          create: # tag with the same key name but with a different value with be added to the service
            - '{"environment": .spec.template.metadata.labels.environment}'
        tools:
          - '{"category": "other", "displayName": "my-cool-tool", "url": .metadata.annotations."example.com/my-cool-tool"} | if .url then . else empty end'
          # find annotations with format: opslevel.com/tools.<category>.<displayname>: <url> 
          - '.metadata.annotations | to_entries |  map(select(.key | startswith("opslevel.com/tools"))) | map({"category": .key | split(".")[2], "displayName": .key | split(".")[3], "url": .value})'
        repositories: # attach repositories to the service using the opslevel repo alias - IE github.com:hashicorp/vault
          - '{"name": "My Cool Repo", "directory": "/", "repo": .metadata.annotations.repo} | if .repo then . else empty end'
          # if just the alias is returned as a single string we'll build the name for you and set the directory to "/"
          - .metadata.annotations.repo
          # find annotations with format: opslevel.com/repo.<displayname>.<repo.subpath.dots.turned.to.forwardslash>: <opslevel repo alias> 
          - '.metadata.annotations | to_entries |  map(select(.key | startswith("opslevel.com/repos"))) | map({"name": .key | split(".")[2], "directory": .key | split(".")[3:] | join("/"), "repo": .value})'

```

Table of Contents
=================

   * [Prerequisite](#prerequisite)
   * [Installation](#installation)
      * [Validate Install](#validate-install)
   * [The Configuration File Explained](#the-configuration-file-explained)
      * [Service Aliases](#service-aliases)
      * [Lifecycle &amp; Tier &amp; Owner](#lifecycle--tier--owner)
      * [Tags](#tags)
      * [Tools](#tools)
   * [Preview](#preview)
   * [Import](#import)
   * [Deploy](#deploy)

## Prerequisite

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [jq](https://stedolan.github.io/jq/download/)
- [OpsLevel API Token](https://app.opslevel.com/api_tokens)

## Installation

#### MacOS

```sh
TOOL_VERSION=$(curl -s https://api.github.com/repos/opslevel/kubectl-opslevel/releases/latest | grep tag_name | cut -d '"' -f 4)
curl -Lo kubectl-opslevel.tar.gz https://github.com/opslevel/kubectl-opslevel/releases/download/${TOOL_VERSION}/kubectl-opslevel-darwin-amd64.tar.gz
tar -xzvf kubectl-opslevel.tar.gz  
rm kubectl-opslevel.tar.gz
sudo mv kubectl-opslevel /usr/local/bin/kubectl-opslevel
```

#### Linux

```sh
TOOL_VERSION=$(curl -s https://api.github.com/repos/opslevel/kubectl-opslevel/releases/latest | grep tag_name | cut -d '"' -f 4)
curl -Lo kubectl-opslevel https://github.com/opslevel/kubectl-opslevel/releases/download/${TOOL_VERSION}/kubectl-opslevel-linux-amd64.tar.gz
tar -xzvf kubectl-opslevel.tar.gz  
rm kubectl-opslevel.tar.gz
sudo mv kubectl-opslevel /usr/local/bin/kubectl-opslevel
```

#### Docker

The docker container is hosted on [AWS Public ECR](https://gallery.ecr.aws/e1n4f2i6/kubectl-opslevel)

The following downloads the container and creates a shim at `/usr/local/bin` so you can use the binary like its downloaded natively - it will just be running in a docker container. *NOTE: you may need to adjust how your kube config is mounted and set inside the container*

```
TOOL_VERSION=$(curl -s https://api.github.com/repos/opslevel/kubectl-opslevel/releases/latest | grep tag_name | cut -d '"' -f 4)
docker pull public.ecr.aws/e1n4f2i6/kubectl-opslevel:${TOOL_VERSION}
docker tag public.ecr.aws/e1n4f2i6/kubectl-opslevel:${TOOL_VERSION} kubectl-opslevel:latest 
cat << EOF > /usr/local/bin/kubectl-opslevel
#! /bin/sh
docker run -it --rm -v \$(pwd):/app -v ${HOME}/.kube:/.kube -e KUBECONFIG=/.kube/config --network=host kubectl-opslevel:latest \$@
EOF
chmod +x /usr/local/bin/kubectl-opslevel
```

<!---
TODO: Implement other methods


#### Homebrew


TODO: Need to Publish to Homebrew


```
brew update && brew install kubectl-opslevel
```

#### Windows


TODO: Chocolately?


1. Get `kubectl-opslevel-windows-amd64` from our [releases](https://github.com/opslevel/kubectl-opslevel/releases/latest).
2. Rename `kubectl-opslevel-windows-amd64` to `kubectl-opslevel.exe` and store it in a preferred path.
3. Make sure the location you choose is added to your Path environment variable.

-->

### Validate Install

Once you have the binary on your Path you can validate it works by running:

```sh
kubectl opslevel version
```

Example Output:

```sh
v0.1.1-0-gc52681db6b33
```

The log format default is more human readable but if you want structured logs you can set the flag `--logFormat=JSON`

```json
{"level":"info","time":1620251466,"message":"Ensured tag 'imported = kubectl-opslevel' assigned to service: 'db'"}
```

## The Configuration File Explained

The tool is driven by a configuration file that allows you to map data from kubernetes resource into OpsLevel fields. 

Here is a simple example that maps a deployment's metadata name to an OpsLevel service name:

<!---
TODO: Would be great to read this from a static file in the repo/wiki?
-->

```yaml
service:
  import:
  - selector:
      kind: deployment
    opslevel:
      name: .metadata.name
```

In the sample configuration there are number of sane default jq expression setup that should help you get started quickly. You can generate a sample config to act as a starting point with the following:

```sh
kubectl opslevel config sample > ./opslevel-k8s.yaml
```

Here are some additional examples of things you can potentially do.

Filter out unwanted keys from your labels to tags mapping (in this case excluding labels that start with "flux"):

```yaml
service:
  import:
  - selector:
      kind: deployment
    opslevel:
      name: '"\(.metadata.name)-\(.metadata.namespace)"'
      tags:
      - .metadata.labels | to_entries | map(select(.key |startswith("flux") | not)) | from_entries
```

Target `Ingress` Resources where each host rule is attached as a tool to cataloge the domains used:

```yaml
service:
  import:
  - selector:
      kind: ingress
    opslevel:
      name: '"\(.metadata.name)-\(.metadata.namespace)"'
      tags:
      - {"imported": "kubectl-opslevel"}
      tools:
      - '.spec.rules | map({"cateogry":"other","displayName":.host,"url": .host})'
```

#### Service Aliases

Aliases on services are very import to get right so that we can effectively look up the service later for reconciliation. Because of that we have only included 1 example which should be fairly unique to most kubernetes setup.  

*This example is also generated by default if no aliases are configured so that we can support lookup for reconcilation when you provide no other aliases.*

```yaml
      aliases:
      - 'k8s:"\(.metadata.name)-\(.metadata.namespace)"'
```

The example concatenates togeather the resource name and namespace with a prefix of `k8s:` to create a unique alias.

#### Lifecycle & Tier & Owner

The `Lifecycle`, `Tier` and `Owner` fields in the configuration are only validated upon `service import` not during `service preview`.  Valid values for these fields need to match the `Alias` for these resources in OpsLevel.  To view the valid aliases for these resources you can run the commands `account lifecycles`, `account tiers` and `account teams`.

#### Tags

In the tags section there are 4 example expressions that show you different ways to build the key/value payload for attaching tag entries to your service

```yaml
      tags:
      - '{"imported": "kubectl-opslevel"}'
      - '.metadata.annotations | to_entries |  map(select(.key | startswith("opslevel.com/tags"))) | map({(.key | split(".")[2]): .value})'
      - .metadata.labels
      - .spec.template.metadata.labels
```

The first example shows how to hardcode a tag entry.  In this case we are denoting that the imported service came from this tool.

The second example leverages a convention to capture `1..N` tags.  The jq expression is looking for kubernetes annotations using the following format `opslevel.com/tags.<key>: <value>` and here is an example:

```yaml
  annotations:
    opslevel.com/tags.hello: world
```

The third and fourth examples extract the `labels` applied to the kubernetes resource directly into OpsLevel service's tags

#### Tools

In the tools section there are 2 example expressions that show you how to build the necessary payload for attaching tools entries to your service.

```yaml
      tools:
      - '{"category": "other", "displayName": "my-cool-tool", "url": .metadata.annotations."example.com/my-cool-tool"}
        | if .url then . else empty end'
      - '.metadata.annotations | to_entries |  map(select(.key | startswith("opslevel.com/tools")))
        | map({"category": .key | split(".")[2], "displayName": .key | split(".")[3],
        "url": .value})'
```

The first example shows you the 3 required fields - `category` , `displayName` and `url` - but the expression hardcodes the values for `category` and `displayName` leaving you to specify where the `url` field comes from.

The second example leverages a convention to capture `1..N` tools.  The jq expression is looking for kubernetes annotations using the following format `opslevel.com/tools.<category>.<displayName>: <url>` and here is a example:

```yaml
  annotations:
    opslevel.com/tools.logs.datadog: https://app.datadoghq.com/logs
```

## Preview

The primary iteration loop of the tool resides in tweaking the configuration file and running the `service preview` command to view data that represents what the tool will do (think of this as a dry-run or terraform plan)

```sh
kubectl opslevel service preview -c ./opslevel-k8s.yaml
```

Once you are happy with the full output you can move onto the actual import process.

*NOTE: this step does not validate any of the data with OpsLevel - fields that are references to other things (IE: Tier, Lifecycle, Owner, etc) are not validated at this point and might cause a warning message during import* 

## Import

Once you are ready to import data into OpsLevel run the following:
*(insert your OpsLevel API Token)*:

```sh
 OL_APITOKEN=XXXX kubectl opslevel service import -c ./opslevel-k8s.yaml
```

This command may take a few minutes to run so please be patient while it works.  In the meantime you can open a browser to [OpsLevel](https://app.opslevel.com/) and view the newly generated/updated services.

## Deploy

Once you are happy with how the tool runs locally we do recommend deploying this tool inside your kubernetes cluster so that it can perform reconciliation periodically.  For this we have a [helm chart](https://github.com/OpsLevel/helm-charts) you can use to get started so head over there to get started.
