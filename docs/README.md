## Overview
`kube-bench` runs a YAML representation of the CIS Kubernetes Benchmarks against
a kubernetes cluster.

`kube-bench`

Controls > Groups > Sub-groups > checks

### Controls


## Architecture
controls  --> kube-bench <-- configuration files
                        |
                        v
                     results 

## Benchmark Specification

Benchmark specifications are YAML representation of the CIS Kubernetes Benchmark.

### Controls

`controls` is a YAML document that contains checks that must be run on a 
specific kubernetes node type, a kubernetes master or a kubernetes node.

`controls` is the fundamental input to kube-bench.

The following is an example of a basic `controls` document:

```yaml
---
controls:
id: 1
text: "Master Node Security Configuration"
type: "master"
groups:
- id: 1.1
  text: API Server
  checks:
    - id: 1.1.1
      text: "Ensure that the --allow-privileged argument is set (Scored)"
      audit: "ps -ef | grep kube-apiserver | grep -v grep"
      tests:
      bin_op: or
      test_items:
      - flag: "--allow-privileged"
        set: true
      - flag: "--some-other-flag"
        set: false
      remediation: "Edit the /etc/kubernetes/config file on the master node and
        set the KUBE_ALLOW_PRIV parameter to '--allow-privileged=false'"
      scored: true
- id: 1.2
  text: Scheduler
  checks:
    - id: 1.2.1
      text: "Ensure that the --profiling argument is set to false (Scored)"
      audit: "ps -ef | grep kube-scheduler | grep -v grep"
      tests:
        bin_op: or
        test_items:
          - flag: "--profiling"
            set: true
          - flag: "--some-other-flag"
            set: false
      remediation: "Edit the /etc/kubernetes/config file on the master node and
        set the KUBE_ALLOW_PRIV parameter to '--allow-privileged=false'"
      scored: true
```

`controls` is composed of a hierachy of groups, sub-groups and checks. Each of
the components including the `controls` itself has metadata; an id and a text 
description which are displayed in the output of `kube-bench`.

`type` specifies what kubernetes node type this `controls` is for. Possible 
values are `master` and `node`.

kube-bench automatically selects which `controls` to use based on the detected
node type and the version of kubernetes a cluster is running.

This behaviour can be overridden by specifying the `master` or `node` subcommand
and the `--version` flag on the command line.

For example:
run kube-bench against a master node using the auto-detected version `controls`:
```shell
kube-bench master
```

or run kube-bench against a node using the node `controls` for kubernetes 
version 1.12:
```shell
kube-bench node --version 1.12
```

`controls` for the various versions of kubernetes can be found under the `cfg`
directory in directories with same name as the kubernetes versions. For example
`cfg/1.12`.

`controls` are also organized by distribution under the `cfg` directory for
example `cfg/ocp-3.10`.

### Groups

`groups`is made up of subgroups which test various aspects of the node type. 
For example one subgroup checks parameters passed to the apiserver binary, while 
another subgroup checks parameters passed to the controller-manager binary.

```
groups:
- id: 1.1
  text: API Server
  checks:
    - id: 1.1.1
      text: "Ensure that the --allow-privileged argument is set (Scored)"
      audit: "ps -ef | grep kube-apiserver | grep -v grep"
      tests:
        bin_op: or
        test_items:
          - flag: "--allow-privileged"
            set: true
          - flag: "--some-other-flag"
            set: false
      remediation: "Edit the /etc/kubernetes/config file on the master node and
        set the KUBE_ALLOW_PRIV parameter to '--allow-privileged=false'"
      scored: true
- id: 1.2
  text: Scheduler
  checks:
    - id: 1.2.1
      text: "Ensure that the --profiling argument is set to false (Scored)"
      audit: "ps -ef | grep kube-scheduler | grep -v grep"
      tests:
        bin_op: or
        test_items:
          - flag: "--profiling"
            set: true
          - flag: "--some-other-flag"
            set: false
      remediation: "Edit the /etc/kubernetes/config file on the master node and
        set the KUBE_ALLOW_PRIV parameter to '--allow-privileged=false'"
      scored: true
      
```

These subgroups have `id`, `text` fields which serve the same purposes described
in the previous paragraphs. The most important part of the subgroup is the
`checks` field which is the collection of actual `check`s that form the subgroup.

This is an example of a subgroup.

```yaml
id: 1.1
text: API Server
checks:
  - id: 1.1.1
    text: "Ensure that the --allow-privileged argument is set (Scored)"
    audit: "ps -ef | grep kube-apiserver | grep -v grep"
    tests:
    ...
  - id: 1.1.2
    text: "Ensure that the --anonymous-auth argument is set to false (Not Scored)"
    audit: "ps -ef | grep kube-apiserver | grep -v grep"
    tests:
    ...
    
``` 

`kube-bench` supports running a subgroup by specifying the subgroup `id` on the
command line, with the flag `--group`.

### Check

The CIS Kubernetes Benchmark recommend configurations to harden kubernetes 
components. These recommendations are usually configurations option, and can be 
specified by flags to the binaries or changes to configuration files used by
the binaries.

The Benchmark also provides commands to audit a kubernetes installation, 
identify places where the cluster security can be improved, and steps to 
remediate the problems.

In `kube-bench`, `check` objects embody these recommendations. 

This an example `check` object:
```
id: 1.1.1
text: "Ensure that the --anonymous-auth argument is set to false (Not Scored)"
audit: "ps -ef | grep kube-apiserver | grep -v grep"
tests:
  test_items:
  - flag: "--anonymous-auth"
    compare:
      op: eq
      value: false
    set: true
remediation: |
  Edit the API server pod specification file kube-apiserver
  on the master node and set the below parameter.
  --anonymous-auth=false
scored: false
```

A `check` has an `id`, a `text`, an `audit` , a `tests`,`remediation` and 
`scored` fields.

`kube-bench` supports running specific checks by specifying the check's `id` on
the command line with the `--check` flag.

The `audit` field specifies the command to run for a check. The output of this
command is then evaluated for conformance with the CIS Kubernetes Benchmark.

The criteria the audit is evaluated against is specified in by the `tests`
object. `tests` comprises `bin_op` and `test_items`.

`test_items` specify the criteria(s) the `audit` command's output should meet to
pass a check. This criteria is made up of keywords extracted from the output of
the `audit` commands and operations that compare the these keywords against
values expected by the CIS Kubernetes Benchmark. 

The are two ways to extract keywords from the output of the `audit` command,
`flag` and `path`.

`flag` is used when the keyword is a command line flag. The associated `audit`
command is usually a `ps` command and a `grep` for the binary whose flag we are
checking:

```
ps -ef | grep somebinary | grep -v grep
``` 

Here is an example usage of the `flag` option:
```
...
audit: "ps -ef | grep kube-apiserver | grep -v grep"
tests:
  test_items:
  - flag: "--anonymous-auth"
  ...

```

`path` is used when the keyword is an option set in a JSON or YAML config file.
The associated `audit` command is usually `cat /path/to/config-yaml-or-json`.
For example:

```
...

text: "Ensure that the --anonymous-auth argument is set to false (Not Scored)"
audit: "cat /path/to/some/config"
tests:
  test_items:
  - path: "{.someoption.value}"

    ...
```

A `test_item` compares the output of the audit command and keywords using the
`set` and `compare` fields.

```
  test_items:
  - flag: "--anonymous-auth"
    compare:
      op: eq
      value: false
    set: true
```

`set` checks if a keyword is present in the audit command or not. The possible
values `set` are true and false. If `set` is true, `kube-bench` checks that the
keyword is present in the audit and the check fails if it is not. If `set` is 
false, `kube-bench` checks that the keyword is not present in the output and 
the check fails if it is present.

`compare` has two fields `op` and `value`. `op` specifies which operation to 
use in our comparison, and `value` specifies what we are comparing to the
output of the audit command.

> To use `compare`, `set` must true. The comparison will be ignored if `set` is
> false

The `op`erations currently supported by `kube-bench` are:
- `eq`: tests if the keyword is equal to the compared value.
- `noteq`: tests if the keyword is unequal to the compared value.
- `gt`: tests if the keyword is greater than the compared value.
- `gte`: tests if the keyword is greater than or equal to the compared value.
- `lt`: tests if the keyword is less than the compared value.
- `lte`: tests if the keyword is less than or equal to the compared value.
- `has`: tests if the keyword contains the compared value.
- `nothave`: tests if the keyword does not contain the compared value.

### Variables

Kubernetes config and binary file locations and names can vary from 
installation to installation, so these are configurable in the 
`cfg/config.yaml` file.

For each type of node (*master*, *node* or *federated*) there is a list of 
components, and for each component there is a set of binaries (*bins*) and 
config files (*confs*) that kube-bench will look for (in the order they are 
listed). If your installation uses a different binary name or config file 
location for a Kubernetes component, you can add it to `cfg/config.yaml`.

* **bins** - If there is a *bins* list for a component, at least one of these 
  binaries must be running. The tests will consider the parameters for the 
  first binary in the list found to be running.
* **podspecs** - From version 1.2.0 of the benchmark (tests for Kubernetes 
  1.8), the remediation instructions were updated to assume that the 
  configuration for several kubernetes components is defined in a pod YAML 
  file, and podspec settings define where to look for that configuration.
* **confs** - If one of the listed config files is found, this will be 
  considered for the test. Tests can continue even if no config file is found. 
  If no file is found at any of the listed locations, and a *defaultconf* 
  location is given for the component, the test will give remediation advice 
  using the *defaultconf* location.
* **unitfiles** - From version 1.2.0 of the benchmark  (tests for Kubernetes 
  1.8), the remediation instructions were updated to assume that kubelet 
  configuration is defined in a service file, and this setting defines where 
  to look for that configuration.

### Versions and distributions

`kube-bench` supports benchmark specs for multiple Kubernetes versions and 
distributions. The supported versions and distributions can be found under the
`cfg` directory of the root directory.

To see a list of supported kubernetes versions and distributions, run
```shell
cd $GOPATH/src/github.com/aquasecurity/kube-bench
ls -1 cfg
1.11
1.13
1.6
1.7
1.8
config.yaml
ocp-3.10
```

The versions listed in `cfg` are specifically kubernetes versions not CIS
Kubernetes Benchmark versions and they are not the same. Please refer to the 
version matrix below to see how kubernetes versions and release map to CIS 
Kubernetes Benchmarks.

