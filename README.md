## This is not an official Google product

![Poncho](http://www.gifss.com/videojuegos/pacman/pacman4.gif)

![CIVIR](https://img.shields.io/badge/elastic_ansible_gce_lab-0.1-red.svg) ![ELK-Stack](https://img.shields.io/badge/Elastic_Stack-6.5.2-blue.svg?&style=flat) ![ZooKeeper](https://img.shields.io/badge/ZooKeeper-3.4.13-green.svg?&style=flat) ![Kafka](https://img.shields.io/badge/Kafka-2.11-orange.svg?&style=flat)

# Is not ready to deploy !!!

# Introduction

This tutorial will take you through the provisioning and testing of a multinode
Elasticsearch STACK cluster on GCE with ZooKeeper and Kafka.


# Pre-requisites

## Create service account

First you will need to download some credentials and set some environment
variables in order to authenticate with GCP.

    gcloud iam service-accounts create ansible-lab --display-name ansible-lab
    export SA_EMAIL=$(gcloud iam service-accounts list --filter="displayName:ansible-lab" --format='value(email)')
    export PROJECT=$(gcloud info --format='value(config.project)')
    gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.storageAdmin --member serviceAccount:$SA_EMAIL
    gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.instanceAdmin.v1 --member serviceAccount:$SA_EMAIL
    gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.networkAdmin --member serviceAccount:$SA_EMAIL
    gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.securityAdmin --member serviceAccount:$SA_EMAIL
    gcloud iam service-accounts keys create ansible-lab-sa.json --iam-account $SA_EMAIL

## Add SSH Key to project metadata

You will also need to ensure that you have your local machine's SSH key uploaded
to your project's metadata.

    ls ~/.ssh/id_rsa.pub || ssh-keygen -N ""
    gcloud compute project-info describe --format=json | jq -r '.commonInstanceMetadata.items[] | select(.key == "sshKeys") | .value' > sshKeys.pub
    echo "$USER:$(cat ~/.ssh/id_rsa.pub)" >> sshKeys.pub
    gcloud compute project-info add-metadata --metadata-from-file sshKeys=sshKeys.pub

In order to run the playbook you will need to install Ansible and Libcloud as
follows:

    pip install --user ansible apache-libcloud

Once those dependencies have been installed, clone the repository and enter the
directory:

    git clone https://github.com/ocuil/elastic-ansible-gce-lab
    cd elastic-ansible-gce-lab

# Cluster Configuration

In the repository you will find a `lab.yml` file that contains parameters at
the top for configuring your cluster.

- `machine_type`: The size of your each machine in the cluster, more cores will give you
  better network performance. The full list of machine types can be found
  [here](https://cloud.google.com/compute/docs/machine-types).
- `hosts`: This list defines both the names of your host and your total cluster
  size
- `disk_type`: The type of disk to use for lab's data volume. This can be
  either pd-ssd or pd-standard.
- `disk_size`: The size of the data disk that will be attached to each instance
  in the cluster. Keep in mind that the IOPS and throughput performance of each
  disk scales with its size. More info on persistent disk performance can be
  found [here](https://cloud.google.com/compute/docs/disks/#pdperformance).
- `zone`: This defines the specific zone with a region that the cluster will be
  provisioned in.

# lab volume configuration

In order to create lab volumes at provisioning time you can define the
`volumes` list variable. Each element in the list is a dictionary with the
following keys that describe the type of volume to create in the cluster.

- `name`: Identifier for your volume
- `type`: lab volume type. Examples are `stripe 3`, `replica 3`, or `stripe 3 replica 3`. Can also be left blank for a distributed volume (default).
- `parameters`: List of parameters for the volume.
- `hosts`: The list of hosts that this volume will be created on.

Volumes will be mounted at /mnt/NAME by default. The mount path can be changed
by setting the global variable `mount_point`.

# Deploying

Once you have configured your cluster specifications you can provision it by
running the Ansible playbook as follows:

    export GCE_EMAIL=$SA_EMAIL
    export GCE_PROJECT=$PROJECT
    export GCE_CREDENTIALS_FILE_PATH=../ansible-lab-sa.json
    export ANSIBLE_HOST_KEY_CHECKING=False
    export PATH=~/.local/bin:$PATH
    ansible-playbook -i hosts lab.yml
    cat /tmp/lab-client-*/*/tmp/* # view your results

# Testing your setup

Make a reference with https://github.com/elastic/rally


# Destroying your cluster and clients

You can use the Ansible playbooks to turn down your machines by running the
following:

    ansible-playbook -i hosts lab.yml -e state=absent
