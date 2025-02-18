# Red Hat Device Edge Workshop Provisioner

The `https://github.com/redhat-manufacturing/device-edge-workshops` contains an Ansible Playbook `provision_lab.yml`, which is an automated lab setup for Ansible training on AWS (Amazon Web Services).  Set the `workshop_type` variable below to provision the corresponding workshop.

| Workshop | Workshop Type Var |
| --- | --- |
| Red Hat Device Edge - Any Workload, 120 Minutes | `workshop_type: rhde_aw_120`  |

## Table Of Contents

<!-- TOC titleSize:2 tabSpaces:2 depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 skip:0 title:1 charForUnorderedList:* -->

* [Ansible Automation Workshop Provisioner](#ansible-automation-workshop-provisioner)
  * [Table Of Contents](#table-of-contents)
  * [Requirements](#requirements)
  * [Lab Setup](#lab-setup)
  * [Using Ansible-Navigator](#using-ansible-navigator)
    * [1. AWS Creds for Execution Environments](#1-aws-creds-for-execution-environments)
    * [2. Running Ansible-Navigator from the project root](#2-running-ansible-navigator-from-the-project-root)
    * [Setup (per workshop)](#setup-per-workshop)
    * [The Student Variable](#the-student-variable)
    * [Building Local Resources](#building-local-resources)
    * [Automation controller license](#automation-controller-license)
    * [Additional examples](#additional-examples)
    * [Accessing student documentation and slides](#accessing-student-documentation-and-slides)
    * [Accessing instructor inventory](#accessing-instructor-inventory)
    * [DNS](#dns)
    * [Smart Management](#smart-management)
  * [Developer Mode and understanding collections](#developer-mode-and-understanding-collections)
  * [Lab Teardown](#lab-teardown)
  * [Demos](#demos)
  * [FAQ](#faq)
  * [More info on what is happening](#more-info-on-what-is-happening)
<!-- /TOC -->

## Requirements

* The provisioner has an execution environment available on [quay.io](https://quay.io/repository/device-edge-workshops/provisioner-execution-environment) with all requirements baked in. In addition, `ansible-navigator` is required, which can be installed into a virtual environment locally.

* AWS Account (follow directions in one time setup below)

## Lab Setup

## Using Ansible-Navigator

### 1. AWS Creds for Execution Environments

You need to set your AWS credentials as environment variables.  This is because the execution environment will not have access to your ~/.aws/credentials file.  This is preferred anyway because it matches the behavior in Automation controller.

```
export AWS_ACCESS_KEY_ID=AKIA6ABLAH1223VBD3W
export AWS_SECRET_ACCESS_KEY=zh6gFREbvblahblahblahfXIC5nZr51OgdKECaSIMBi9Kc
```

To make environment variables permanent and persistent you can set this to your `~/.bash_rc`.  See Red Hat Knowledge Base article: [https://access.redhat.com/solutions/157293](https://access.redhat.com/solutions/157293)

### 2. Running Ansible-Navigator from the project root

You must run from the project root rather than the `/provisioner` folder.  This is so all the files in the Git project are mounted, not just the provisioner folder.  This is also best practice because it matches the behavior in Automation controller.

Make sure to install the prerequired collections by running:
```
ansible-galaxy  install -r execution-environment/requirements.yml
```

For example:

```
ansible-navigator run provisioner/provision_lab.yml -e @provisioner/extra_vars.yml
```

```
ansible-playbook provisioner/provision_lab.yml -e @provisioner/extra_vars.yml
```

You can also run the provision_lab playbook using ansible-playbook command, provided you have installed all the required collections.
Also make sure in this case to have enabled passwordless sudo on your local user.

### Setup (per workshop)

* Define the following variables in a file passed in using `-e @extra_vars.yml`

```yaml
---
# where the workshop is being run
run_in_aws: true
run_locally: false

# if local hypervisor nodes should be configured
manage_local_hypervisor: false

# region where the nodes will live
ec2_region: us-east-2

# name prefix for all the VMs
ec2_name_prefix: lab-prefix

# Set the right workshop type
workshop_type: rhde_aw_120

# Set the number of student slots
student_total: 10

# Generate offline token to authenticate the calls to Red Hat's APIs
# Can be accessed at https://access.redhat.com/management/api
offline_token: "your-token-here"

# Required for podman authentication to registry.redhat.io
redhat_username: your-username
redhat_password: your-password

#####OPTIONAL VARIABLES

# turn DNS on for control nodes, and set to type in valid_dns_type
dns_type: aws

# password for Ansible control node
admin_password: lab-admin-password

# Sets the Route53 DNS zone to use for Amazon Web Services
workshop_dns_zone: your-dns-zone-here.com

# Use zeroSSL
use_zerossl: true

zerossl_account:
  kid: your
  key: info
  alg: here

# automatically installs Tower to control node
controllerinstall: true

# forces ansible.workshops collection to install latest edits every time
developer_mode: false

# SHA value of targeted AAP bundle setup files.
provided_sha_value: e3cd033d6a6f5ddcdeb2f5b91b1382127d29b969fb224f260d0a4f1e495b20e6
pre_build: false

# Don't need automation hub
automation_hub: false

builder_pub_key: 'your-key-here'

base64_manifest: your-manifest-here
```

### The Student Variable
In normal Ansible workshops, a student number is defined to control how many student slots are provisioned, however to better reflect the edge mantra (and because, ideally, these workshops will be run with edge devices) that var simply controls the number of available signup slots on the attendance host. If no edge devices are available, or if additional capacity is required, the provisioner can fire up a bare metal instance to host virtualized edge devices.

### Building Local Resources
To demonstrate the capabilities of the device edge stack running in a disconnected environment, the provisioner can also be used to configure a local asset (a laptop, NUC, whatever) to act as the 'edge-manager' box. This behavior is controlled by the `run_locally` variable. In addition to setting the var, you'll also need to set up an inventory, like so:
```yaml
all:
  children:
    edge_management:
      hosts:
        edge-manager-local:
          ansible_host: 192.168.200.10
    controller:
      hosts:
        edge-manager-local:
          ansible_host: 192.168.200.10
    local:
      hosts:
        edge-manager-local:
          ansible_host: 192.168.200.10
      children:
        dns:
          hosts:
            edge-manager-local:
              ansible_host: 192.168.200.10
          vars:
            local_domains:
              controller:
                domain: "controller.your-workshop-domain.lcl"
              cockpit:
                domain: "cockpit.your-workshop-domain.lcl"
              gitea:
                domain:  "gitea.your-workshop-domain.lcl"
              edge_manager:
                domain: "edge-manager.your-workshop-domain.lcl"
  vars:
    ansible_user: ansible
    ansible_password: your-password
    ansible_become_password: your-password
```

The provisioner can build the workshop in aws, locally, or both if desired. Running in both can help avoid issues with networking not under the control of the instructor (think terribad hotel wifi) by having two distinct yet identical systems to run the lab from.

>**Note**
>
> ini-formatted inventories are fine, as well as using ssh keys, this is just a standard inventory file. Since the provisioner isn't creating the host (like for aws instances), connection information must be provided.

>**Note**
>
> For local resources, you are responsible for the vast majority of the initial setup, including having RHEL installed, having connectivity, DNS, etc. This provisioner will not manage these elements in the local environment for you.

### Automation controller license

In order to use Automation controller (i.e. `controllerinstall: true`), which is the default behavior (as seen in group_vars/all.yml) you need to have a valid subscription via a `manifest.zip` file.  To retrieve your manifest.zip file you need to download it from access.redhat.com.  

- Here is a video by Colin McNaughton to help you retrieve your manifest.zip:
 [https://youtu.be/FYtilnsk7sM](https://youtu.be/FYtilnsk7sM).
- If you need to get a temporary license, get a trial here [http://red.ht/try_ansible](http://red.ht/try_ansible).
- Follow the following KCS on how to generate the manifest file https://access.redhat.com/solutions/5586461

**How do you use the manifest.zip with the workshop?**

These are the ways to integrate your license file with the workshop:

1. Put the manifest.zip file into provisioner folder

  The first way is to make sure your license/manifest has the exact name `manifest.zip` and put it into the same folder as the `provision_lab.yml` playbook (e.g.) `<your-path>/workshops/provisioner/manifest.zip`

2. Turn the manifest.zip into a variable

  The second way is to turn the `manifest.zip `into a base64 variable.

  This allows the `manifest.zip` to be treated like an Ansible variable so that it can work with CI systems like Github Actions or Zuul.  This also makes it easier to work with Automation controller, in case you are spinning up a workshop using Automation controller itself.

  To do this use the `base64` command to encode the manifest:

  ```
  base64 manifest.zip > base64_platform_manifest.txt
  ```
  Take the output of this command and set it to a variable `base64_manifest` in your extra_vars file.

  e.g.
  ```
  base64_manifest: 2342387234872dfsdlkjf23148723847dkjfskjfksdfj
  ```

  >**Note**
  >
  >The manifest.zip is substantially larger than the tower.license file, so the base64_manifest base64 might be several hundred lines long if you have text wrapping in your editor.

  >**Note**
  >
  >base64 is not encryption, if you require encryption you need to work within your CI system or Automation controller to encrypt the base64 encoded manifest.zip.

3. Download the manifest.zip from a URL

  If you specify the following variables, the provisioner will download the manifest.zip from an authenticated URL:

  ```
  manifest_download_url: https://www.example.com/protected/manifest.zip
  manifest_download_user: username
  manifest_download_password: password
  ```

### Automating the download of aap.tar.gz 

If you have the aap.tar.gz tarball in a secure URL, you can automate the downloading of it by specifying the following variables.
Note that the tarball specified in the URL must match the SHA value defined in provided_sha_value

  ```
  aap_download_url: https://www.example.com/protected/aap.tar.gz
  aap_download_user: username
  aap_download_password: password
  ```

### Additional examples

For more extra_vars examples, look at the following:

* [sample-vars-rhel.yml](sample_workshops/sample-vars-rhel.yml) - example for the Ansible RHEL Workshop

* Run the playbook:

```bash
ansible-playbook provision_lab.yml -e @extra_vars.yml
```

* Login to the AWS EC2 console and you will see instances being created.  For example:

```yaml
rhde_aw_120-student1-ansible
````

### Accessing student documentation and slides

* Exercises and instructor slides are hosted at [aap2.demoredhat.com](aap2.demoredhat.com)

* Workbench information is stored in two places after you provision:

  * in a local directory named after the workshop (e.g. testworkshop/instructor_inventory)
  * By default there will be a website `ec2_name_prefix.workshop_dns_zone` (e.g. `testworkshop.rhdemo.io`)

    * **NOTE:** It is possible to change the DNS domain (right now this is only supported via a AWS Route 53 Hosted Zone) using the parameter `workshop_dns_zone` in your `extra_vars.yml` file.

### Accessing instructor inventory

* The instructor inventory will be copied to `/tmp` on student1's control_node as part of the control_nodes role.
* The instructor can see all assigned students and what their workbench is by visiting `ec2_name_prefix.workshop_dns_zone/list.php` (e.g. `testworkshop.rhdemo.io/list.php`)

### DNS

The provisioner currently supports creating DNS records per control node with valid SSL certs using [Lets Encrypt](https://letsencrypt.org/).  Right now DNS is only supported via AWS Route 53, however we are building it in a way that this can be more pluggable and take advantage of other public clouds.

This means that each student workbench will get an individual DNS entry.  For example a DNS name will look like this: `https://student1.testworkshop.rhdemo.io`

* **NOTE:** The variable `dns_type` defaults to `aws`.  This can also be set to `dns_type: none`.
* **NOTE:**  If Lets Encrypt fails, the workshop provisioner will still pass, and alert you of errors in the `summary_information` at the end of the `provision_lab.yml` Ansible Playbook.

### Smart Management

The Smart Management Lab relies on a prebuilt AMI for Red Hat Satellite Server. An example for building this AMI can be found [here](https://github.com/willtome/ec2-image-build).

The Smart Management Lab also requires AWS DNS to be enabled. See [sample vars](./sample_workshops/sample-vars-smart_mgmt.yml) for required configuration.

## Developer Mode and understanding collections

The Ansible Workshops are actually a collection.  Every role is called using the FQCN (fully qualified collection name).  For example to setup the control node (e.g. install Automation controller) we call the role

```
- include_role:
    name: ansible.workshops.control_node
```

This installs locally from Git (versus from Galaxy or Automation Hub).  If the galaxy.yml version **matches** your installed version, it will skip the install (speed up provisioning).  Using `developer_mode: true` if your extra_vars will force installation every time.  This is super common when you are editing a role and want to immediately see changes without publishing the collection.

If you want to contribute to the workshops, check out the [contribution guide](../docs/contribute.md).

## Lab Teardown

The `teardown_lab.yml` playbook deletes all the training instances as well as local inventory files.

To destroy all the EC2 instances after training is complete:

* Run the playbook:

```bash
ansible-playbook teardown_lab.yml -e @extra_vars.yml
```

* Optionally you can enable verbose debug output of the information gathered that drives the teardown process by passing the extra optional variable `debug_teardown=true`. Example:

```bash
ansible-playbook teardown_lab.yml -e @extra_vars.yml -e debug_teardown=true
```

Note: Replace `ansible-playbook` with `ansible-navigator run` if using `ansible-navigator`.

## More info on what is happening

The `provision_lab.yml` playbook creates a work bench for each student, configures them for password authentication, and creates an inventory file for each user with their IPs and credentials. An instructor inventory file is also created in the current directory which will let the instructor access the nodes of any student.  This file will be called `instructor_inventory.txt`

What does the AWS provisioner take care of automatically?

* AWS VPC creation (Amazon WebServices Virtual Private Cloud)
* Creation of an SSH key pair (stored at ./WORKSHOPNAME/WORKSHOPNAME-private.pem)
* Creation of a AWS EC2 security group
* Creation of a subnet for the VPC
* Creation of an internet gateway for the VPC
* Creation of route table for VPC (for reachability from internet)
