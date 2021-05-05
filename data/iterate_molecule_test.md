---
title: "{{ replace .Name "-" " " | }}"
date: {{ .Date }}
draft: true
---

# Testing Molecule Test Code

Openshift ci is enabled on the [mig-operator](https://github.com/konveyor/mig-operator/), and now we should be writing end-to-end tests when a feature or bugfix is added. This document will show you how to iterate and test the molecule test that must be added.

## Creating Molecule Test

The first step is to create your molecule test. These tests should be added in the [openshift directory](https://github.com/konveyor/mig-operator/tree/master/molecule/openshift). To understand how to write molecule tests, you should look at the [docs here](https://molecule.readthedocs.io/en/latest/). You can see an example of an added test [here](https://github.com/konveyor/mig-operator/pull/667).

## Testing Molecule Test

To test your molecule test you should first set up a [virtual environment](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/) with [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-with-pip), [molecule](https://pypi.org/project/molecule/) and [jmespath](https://pypi.org/project/jmespath/) installed. 

The next step is to deploy the migration operator in running clusters. To create running clusters, use [mig-agnosticd](https://github.com/konveyor/mig-agnosticd) to create a 4.x cluster and then use the [build-push-subscribe.sh](https://github.com/konveyor/mig-operator/blob/master/deploy/build-push-subscribe.sh) script to deploy to operator onto the cluster. 

Once this is completely running, the molecule test is as simple as:

```bash
$ molecule test -s openshift
```

Sometimes the log output can overflow a tmux or screen buffer. In this case, use an output file, for example:

```bash
$ molecule test -s openshift > output.txt
```

if this fails, then you can  fix the test and re-run, as the test is idempotent