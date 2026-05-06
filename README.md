# build_rhel_iso
This repository contains artifacts to build a RHEL iso for deploying LFEnergy Seapath clusters

## Pre-flight steps

Perform steps listed below

* Provision a host machine with latest version of RHEL 9
* Install [edgeautomation](https://github.com/rprakashg/edgeautomation) collection by running command below

```sh
ansible-galaxy collection install git+https://github.com/rprakashg/edgeautomation.git,main
```

* Create an ansible inventory file and add snippet below, replace with specific parameters.

```yaml
---
all:
  hosts:
    imagebuilder:
      ansible_host: "<replace>"
      ansible_port: 22
      ansible_user: "<replace>"
      ansible_ssh_private_key_file: "<replace>"
```

## Setup Imagebuilder host
Run the [setup_imagebuilder](./ansible/setup_imagebuilder.yml) playbook to configure the imagebuilder host

```sh
ansible-playbook -i inventory setup_imagebuilder.yml 
```

## Build ISO
Build RHEL iso using the blueprint defined in [vpac](./ansible/vars/vpac.yaml) file

```sh
ansible-playbook -i inventory build_iso.yml -e @vars/vpac.yaml
```

Once complete grab the compose job id from the imagebuilder host by running command below:

```sh
composer-cli compose status
```

Create an ansible vault by running command below:

```sh
ansible-vault create ./vars/secrets.yml
```

When prompted enter vault secret. Save the vault secret into an environment variable as shown below

```sh
export VAULT_SECRET=<redacted>
```

Add snippet below to vault `secrets.yml` file

```yaml
admin_user: <redacted>
admin_user_password: <redacted>
admin_user_ssh_pubkey: <redacted>
root_password: <redacted>
```

## Inject custom Kickstart
Create a custom anaconda kickstart file using the definition file [ks](./ansible/vars/ks.yaml) and create a new ISO with the custom kickstart that is used to customize RHEL installation

```sh
ansible-playbook -i inventory --vault-password-file <(echo "$VAULT_SECRET") inject_ks.yml -e @vars/ks.yaml -e compose_job_id=<specify>
```

## Build cloudinit seed ISO
For each host machine in seapath cluster create cloudinit seed iso by running the [build_cloudinit_seed_iso](./ansible/build_cloudinit_seed_iso.yml). 

```sh
ansible-playbook -i inventory --vault-password-file <(echo "$VAULT_SECRET") build_cloudinit_seed_iso.yml -e @vars/cloudinit.yaml -e hostname=<specify> -e iso_name=seed-<specify>.iso
```

At this point we have the base RHEL ISO image and the cloudinit seed ISOs for each host machine in a seapath cluster ready to be used for provisioning the seapath cluster nodes.

