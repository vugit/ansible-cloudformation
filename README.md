# Ansible Role: Create cloudformation stacks

**[Motivation](#motivation)** |
**[Installation](#installation)** |
**[Variables](#variables)** |
**[Usage](#usage)** |
**[Templates](#templates)** |
**[Dependencies](#dependencies)** |
**[Requirements](#requirements)** |
**[License](#license)**

[![Build Status](https://travis-ci.org/cytopia/ansible-cloudformation.svg?branch=master)](https://travis-ci.org/cytopia/ansible-cloudformation)
[![Ansible Galaxy](https://img.shields.io/ansible/role/d/23347.svg)](https://galaxy.ansible.com/cytopia/cloudformation/)

Ansible role to render an arbitrary number of Jinja2 templates into cloudformation files and create any number of stacks.


## Motivation

This role overcomes the shortcomings of Cloudformation templates itself as well as making heavy use of Ansible's features.

1. **Cloudformation limitations** - The Cloudformation syntax is very limited when it comes to programming logic such as conditions, loops and complex variables such as arrays or dictionaries. By wrapping your Cloudformation template into Ansible, you will be able to use Jinja2 directives within the Cloudformation template itself, thus having all of the beauty of Ansible and still deploy via Cloudformation stacks.
2. **Environment agnostic** - By being able to render Cloudformation templates with custom loop variables you can finally create fully environment agnostic templates and re-use them for production, testing, staging and other environments.
3. **Dry run** - Another advantage of using Ansible to deploy your Cloudformation templates is that Ansible supports a dry-run mode (`--check`) for Cloudformation deployments (since Ansible 2.4). During that mode it will create Change-sets and let you know **what would change** if you actually roll it out. This way you can safely test your stacks before actually applying them.

This role can be used to either only generate your templates via `cloudformation_generate_only` or also additionally deploy your rendered templates. So when you have your deployment infrastructure already in place, you can still make use of this role, by only rendering the templates and afterwards hand them over to your existing infrastructure.

When templates are rendered, a temporary `build/` directory is created inside the role directory. This can either persist or be re-created every time this role is run. Specify the behaviour with `cloudformation_clean_build_env`.


## Installation
```bash
$ ansible-galaxy install cytopia.cloudformation
```

## Variables

### Overview

The following variables are available in `defaults/main.yml` and can be used to setup your infrastructure.

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `cloudformation_clean_build_env` | bool | `False` | Clean build directory of Jinja2 rendered Cloudformation templates on each run. |
| `cloudformation_generate_only` | bool | `False` | Specify this variable via ansible command line arguments to only render the Cloudformation files from Jinja2 templates and do not deploy them on AWS. |
| `cloudformation_defaults` | dict | `{}` | Dictionary of default values to apply to every cloudformation stack. Note that those values can still be overwritten on a per stack definition. |
| `cloudformation_stacks` | dict | `[]` | Array of cloudformation stacks to deploy. |

### Details

This section contains a more detailed describtion about available dict or array keys.

#### `cloudformation_defaults`

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `aws_access_key` | string | optional | AWS access key to use |
| `aws_secret_key` | string | optional | AWS secret key to use |
| `security_token` | string | optional | AWS security token to use |
| `profile` | string | optional | AWS boto profile to use |
| `notification_arns` | string | optional | Publish stack notifications to these ARN's |
| `region` | string | optional | AWS region to deploy stack to |

#### `cloudformation_stacks`

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `stack_name` | string | required | Name of the cloudformation stack |
| `template` | string | required | Path to the cloudformation template to render and deploy (Does not need to be rendered) |
| `aws_access_key` | string | optional | AWS access key to use (overwrites default) |
| `aws_secret_key` | string | optional | AWS access key to use (overwrites default)  |
| `security_token` | string | optional | AWS security token to use (overwrites default) |
| `profile` | string | optional | AWS boto profile to use (overwrites default) |
| `notification_arns` | string | optional | Publish stack notifications to these ARN's (overwrites default) |
| `region` | string | optional | AWS region to deploy stack to (overwrites default) |
| `template_parameters` | dict | optional | Required cloudformation stack parameters |
| `tags` | dict | optional | Tags associated with the cloudformation stack |

### Examples

Define default values to be applied to all stacks (if not overwritten on a per stack definition)
```yml
cloudformation_defaults:
  profile: testing
  region: eu-central-1
```


Define cloudformation stacks to be rendered and deployed
```yml
cloudformation_stacks:
  - stack_name: stack-s3
    template: files/cloudformation/s3.yml.j2
    profile: production
    template_parameters:
      bucketName: my-bucket
    tags:
      env: production
  - stack_name: stack-lambda
    template: files/cloudformation/lambda.yml.j2
    profile: production
    template_parameters:
      lambdaFunctionName: lambda
      handler: lambda.run_handler
      runtime: python2.7
      s3Bucket: my-bucket
      s3Key: lambda.py.zip
    tags:
      env: production
```

Only render your Jinja2 templates, but do not deploy them to AWS. Rendered cloudformation files will be inside the `build/` directory of this role.
```bash
$ ansible-playbook play.yml -e cloudformation_generate_only=True
```

## Usage

### Simple

Basisc usage example:

`playbook.yml`
```yml
- hosts: localhost
  connection: local
  roles:
    - cloudformation
```

`group_vars/all.yml`
```yml
cloudformation_defaults:
  profile: testing
  region: eu-central-1

cloudformation_stacks:
  - stack_name: stack-s3
    template: files/cloudformation/s3.yml.j2
    template_parameters:
      bucketName: my-bucket
    tags:
      env: "{{ cloudformation_defaults.profile }}"
  - stack_name: stack-lambda
    template: files/cloudformation/lambda.yml.j2
    template_parameters:
      lambdaFunctionName: lambda
      handler: lambda.run_handler
      runtime: python2.7
      s3Bucket: my-bucket
      s3Key: lambda.py.zip
    tags:
      env: "{{ cloudformation_defaults.profile }}"
```

### Advanced

Advanced usage example calling the role independently in different *virtual* hosts.

`inventory`
```ini
[my-group]
infrastructure  ansible_connection=local
application     ansible_connection=local
```

`playbook.yml`
```yml
# Infrastructure part
- hosts: infrastructure
  roles:
    - cloudformation
  tags:
    - infrastructure

# Application part
- hosts: application
  roles:
    - some-role
  tags:
    - some-role
    - application

- hosts: application
  roles:
    - cloudformation
  tags:
    - application
```

`group_vars/my-group.yml`
```yml
stack_prefix: testing
boto_profile: testing
s3_bucket: awesome-lambda

cloudformation_defaults:
  profile: "{{ boto_profile }}"
  region: eu-central-1
```

`host_vars/infrastructure.yml`
```yml
cloudformation_stacks:
  - stack_name: "{{ stack_prefix }}-s3"
    template: files/cloudformation/s3.yml.j2
    template_parameters:
      bucketName: "{{ s3_bucket }}"
    tags:
      env: "{{ stack_prefix }}"
```

`host_vars/application.yml`
```yml
cloudformation_stacks:
  - stack_name: "{{ stack_prefix }}-lambda"
    template: files/cloudformation/lambda.yml.j2
    template_parameters:
      lambdaFunctionName: lambda
      handler: lambda.run_handler
      runtime: python2.7
      s3Bucket: "{{ s3_bucket }}"
      s3Key: lambda.py.zip
    tags:
      env: "{{ stack_prefix }}"
```


## Templates

This section gives a brief overview about what can be done with Cloudformation templates using Jinja2 directives.

### Example: Subnet definitions

The following template can be rolled out to different staging environment and is able to include a different number of subnets.

Ansible variables
```yml
---
# file: staging.yml
vpc_subnets:
  - directive: subnetA
    az: a
    cidr: 10.0.10.0/24
    tags:
      - name: Name
        value: staging-subnet-a
      - name: env
        value: staging
  - directive: subnetB
    az: b
    cidr: 10.0.20.0/24
    tags:
      - name: Name
        value: staging-subnet-b
      - name: env
        value: staging
```

```yml
---
# file: production.yml
vpc_subnets:
  - directive: subnetA
    az: a
    cidr: 10.0.10.0/24
    tags:
      - name: Name
        value: prod-subnet-a
      - name: env
        value: production
  - directive: subnetB
    az: b
    cidr: 10.0.20.0/24
    tags:
      - name: Name
        value: prod-subnet-b
      - name: env
        value: production
  - directive: subnetC
    az: b
    cidr: 10.0.30.0/24
    tags:
      - name: Name
        value: prod-subnet-c
      - name: env
        value: production
```

Cloudformation template
```jinja
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC Template
Resources:
  vpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: {{ vpc_cidr_block }}
      EnableDnsSupport: true
      EnableDnsHostnames: true
{% if vpc_tags %}
      Tags:
{% for tag in vpc_tags %}
        - Key: {{ tag.name }}
          Value: {{ tag.value }}
{% endfor %}
{% endif %}
{% for subnet in vpc_subnets %}
  {{ subnet.directive }}:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: {{ subnet.az }}
      CidrBlock: {{ subnet.cidr }}
      VpcId: !Ref vpc
{% if subnet.tags %}
      Tags:
{% for tag in subnet.tags %}
        - Key: {{ tag.name }}
          Value: {{ tag.value }}
{% endfor %}
{% endif %}
```


### Example: Security groups

Defining security groups with IP-specific rules is very difficult when you want to keep environment agnosticity and still use the same Cloudformation template for all environments. This however can easily be overcome by providing environment specific array definitions via Jinja2.

Ansible variables
```yml
---
# file: staging.yml
# Staging is wiede open, so that developers are able to
# connect from attached VPN's
security_groups:
  - protocol:  tcp
    from_port: 3306
    to_port:   3306
    cidr_ip:   10.0.0.1/32
  - protocol:  tcp
    from_port: 3306
    to_port:   3306
    cidr_ip:   192.168.0.15/32
  - protocol:  tcp
    from_port: 3306
    to_port:   3306
    cidr_ip:   172.16.0.0/16
```

```yml
---
# file: production.yml
# The production environment has far less rules as well as other
# ip ranges.
security_groups:
  - protocol:  tcp
    from_port: 3306
    to_port:   3306
    cidr_ip:   10.0.15.1/32
```

Cloudformation template
```jinja
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC Template
Resources:
  rdsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: RDS security group
{% if security_groups %}
      SecurityGroupIngress:
{% for rule in security_groups %}
        - IpProtocol: "{{ rule.protocol }}"
          FromPort: "{{ rule.from_port }}"
          ToPort: "{{ rule.to_port }}"
          CidrIp: "{{ rule.cidr_ip }}"
{% endfor %}
{% endif %}
```


## Dependencies

None


## Requirements

Use at least **Ansible 2.4** in order to also have `--check` mode for cloudformation.


## License

[MIT License](LICENSE.md)

Copyright (c) 2017 [cytopia](https://github.com/cytopia)
