---
title: "Automating AWS AMI Creation"
description: "  "
layout: post
categories:
  - AWS
  - EC2
  - HashiCorp
  - Packer
---

This article briefly delves into Hashicorp [Packer](https://www.packer.io/) and what it can do for your AMI creation workflow.

Setting up an AMI is a well-defined process that usually follows these steps:

1. Launch an instance from an existing AMI, private or public.

2. SSH into the instance.

3. Execute a series of commands to configure it.

4. Stop the instance.

5. Create an AMI from the instance.

6. Terminate the instance.

7. If you need the AMI in a different region or account, do a manual copy operation or explicitly grant rights to a different account.

8. Repeat steps 1-7 whenever you need a change in the AMI.

Since AMI creation is such an involved and lengthy process, we might sometimes prefer to have an initialization script that runs every time an instance is launched with the AMI and have the script configure the instance with the latest updates to suit our needs. This renders the launched instance unusable for a period of time until the configuration process is complete and might even contribute to higher costs if the workloads are short-lived.

Justifiably, these costs might be more than acceptable considering the alternative, which would be having to follow the lengthy workflow laid out above every time you need your AMI updated.

But what if we could automate this process and treat AMI creation like a git push or a simple configuration change?

**Enter Packer.**

Packer automates every step in the AMI creation workflow using a JSON template that you define, which means that your workflow for setting up an AMI will now look like this:

```
1. Create a JSON template.

2. packer validate template.json

3. packer build template.json
```

When you need to update the AMI, modify your template to make the changes you need, and run steps 2 and 3.

The Packer documentation is comprehensive and can be found [here](https://www.packer.io/docs) and a tutorial can be found [here](https://learn.hashicorp.com/tutorials/packer/getting-started-build-image). But in order to get a quick intuition of what Packer can do, let's explore a simple example.


## Example Use Case

You need to build an AMI with the following properties:

* You have a series of existing AMIs that all start with the prefix `sample_ami_v_`. You will also be following this naming guideline for your new AMI.
  > Versioning the AMI names with predictable prefixes allows you to filter them programmatically.

* The `redis-server` apt package needs to be added to the AMI.

* The AMI needs to be available in the `us-east-1` region primarily, and additionally in the `ap-southeast-1` and `ap-south-1` regions.

* The AMIs should be accessible from a different AWS account with the account ID: `123456789123`.


### JSON Template

```json
{
  "variables": {
    "aws_access_key": "{{env `AWS_ACCESS_KEY_ID`}}",
    "aws_secret_key": "{{env `AWS_SECRET_ACCESS_KEY`}}"
  },
  "builders": [
    {
      "type": "amazon-ebs",
      "access_key": "{{user `aws_access_key`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "region": "us-east-1",
      "source_ami_filter": {
        "filters": {
          "name": "sample_ami_v_*"
        },
        "owners": ["987654321987"],
        "most_recent": true
      },
      "instance_type": "t2.medium",
      "ssh_username": "ubuntu",
      "ami_name": "sample_ami_v_{{timestamp}}",
      "run_tags": {
        "Name": "ami_creator"
      },
      "ami_regions": ["ap-southeast-1", "ap-south-1"],
      "ami_users": ["123456789123"]
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline_shebang": "/bin/bash -e",
      "inline": ["sudo apt install redis-server"]
    }
  ]
}
```


### Breakdown

* Packer uses the AWS access key and the secret key to access and provision AWS resources. 
  * Set environment variables on the command line and define them inside `variables`; they can be now be accessed in your template as user variables.

* After determining an appropriate AMI by using `builders` -> `source_ami_filter`, an instance is launched in "us-east-1" (`builders` -> `region`).
  * A temporary security group and key pair are created.
  * The security group allows access to port 22 (SSH).
  * The tags specified under `builders` -> `run_tags` are added to the launched instance.

* Packer connects to the instance using SSH and executes the commands defined under `provisioners` -> `inline`.
  * The output of these commands will be displayed on your console.
  * The `-e` flag is used when defining `provisioners` -> `inline_shebang` to ensure that errors caused by the commands will trigger the failure of the build process, which prevents your AMIs from being silently built with something that's not according to your specifications.

* Once that's done, the instance is stopped, and an AMI (named `builders` -> `ami_name`) is created using the stopped instance.
  * This might take a few minutes.

* The newly created AMI is then copied over to the regions specified under `builders` -> `ami_regions`.

* Once the copies are done, permissions are granted to these AMIs for the users specified under `builders` -> `ami_users`.
  * So in this case, we will end up with three AMIs across three different regions and each of those AMIs will be available to two AWS accounts -- the original user and user "123456789123".

* The final step is clean-up.
  * The launched instance is terminated.
  * The temporary security group and key pair are deleted.


## Summary

Thus, with a bit of initial setup, Packer automates the entire AMI build process. Here are some take-aways and points to note:

* The configuration of the launched instance can be as simple as installing a single package, as in the example above, or as complex as running an Ansible Playbook. You can find a list of provisioners that you can use to configure your instance [here](https://www.packer.io/docs/provisioners/ansible).

* Commit your packer templates to your code repository to help you version your changes.
  * If possible, have a folder set aside for Packer templates and have one copy for each version of the template there (v1, v2, v3, etc) to give yourself a clear overview of the changes being added to your AMI over time.

* Use `builders` -> `ami_description` to describe the changes you're making to the AMI.

* Packer only creates your AMIs and in no way manages them. So it's important to keep this in mind when setting up an automated pipeline using Packer, especially if you're specifying availability for multiple regions. Over time, these AMIs might invisibly add up in the background, contributing to higher costs.

* You can have multiple builders in a single template, which allows you to define different AMIs in a single file. You can use this feature to create AMIs for multiple platforms at the same time. An example can be found [here](https://learn.hashicorp.com/tutorials/packer/getting-started-parallel-builds).
