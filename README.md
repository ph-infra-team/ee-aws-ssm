# ee-aws-ssm

AWX execution environment for running Ansible automation against private AWS EC2 instances through AWS Systems Manager Session Manager.

## Purpose

This image is used by AWX job templates, not by GitLab Terraform pipelines.

It provides the runtime required for:

```yaml
ansible_connection: amazon.aws.aws_ssm
```

The image includes:

- AWX-compatible base runtime from `quay.io/ansible/awx-ee:24.6.1`
- `amazon.aws` collection
- `community.aws` collection
- `boto3`
- `botocore`
- AWS Session Manager plugin executable at `/usr/local/bin/session-manager-plugin`
- system packages from `bindep.txt`

## Repository Files

```text
execution-environment.yml   # ansible-builder definition
requirements.yml            # Ansible collections
requirements.txt            # Python dependencies
bindep.txt                  # OS package dependencies
.gitlab-ci.yml              # consumes shared EE pipeline
```

## Build Flow

```text
GitLab CI/CD
  -> includes infra_team/platform-pipelines/ansible/ee.yml
  -> validates EE source files
  -> runs ansible-builder
  -> builds container image
  -> pushes image to GitLab Container Registry
  -> optionally registers/updates AWX Execution Environment
```

The image is pushed to this project container registry:

```text
registry.midhtech.local:5050/infra_team/automation/ee-aws-ssm:<tag>
```

## Shared Pipeline

This repo uses the central shared pipeline:

```yaml
include:
  - project: infra_team/platform-pipelines
    file: ansible/ee.yml
```

The shared pipeline owns build, smoke test, push, and optional AWX registration behavior.

## AWX Registration

To let the pipeline register/update this EE in AWX, set this GitLab CI/CD variable:

```text
EE_REGISTER_IN_AWX=true
```

Required AWX variables:

```text
AWX_USERNAME
AWX_PASSWORD
```

Optional AWX variables:

```text
AWX_HOST=http://awx.midhtech.local
AWX_EE_NAME=ee-aws-ssm
AWX_EE_PULL=always
AWX_EE_REGISTRY_CREDENTIAL_NAME=gitlab-container-registry
```

If the GitLab registry is private, configure an AWX container registry
credential and attach it to the EE in AWX.

Example AWX credential:

```text
Name: gitlab-container-registry
Credential Type: Container Registry
Authentication URL: registry.midhtech.local:5050
Username: <GitLab deploy token username>
Password: <GitLab deploy token with read_registry>
```

The deploy token can be repo-scoped to this project for least privilege, because
this project owns the image:

```text
registry.midhtech.local:5050/infra_team/automation/ee-aws-ssm:<tag>
```

## AWX Usage

In AWX:

```text
Administration -> Execution Environments -> ee-aws-ssm
```

Expected EE settings:

```text
Image: registry.midhtech.local:5050/infra_team/automation/ee-aws-ssm:v1.0.0
Pull: Always
Registry credential: gitlab-container-registry
```

Use this EE in job templates that target AWS EC2 through SSM.

Example job template:

```text
Inventory: maas-ec2-dev
Project: test_awx_aws_connect
Playbook: playbooks/ping.yml
Execution Environment: ee-aws-ssm
Credential: AWS credential with SSM and S3 permissions
```

## AWX Image Pull Requirements

AWX jobs run on the AWX Kubernetes runtime, not on the GitLab runner. A GitLab
pipeline can build and push this image successfully while AWX still fails to
pull it.

For the current lab environment:

```text
AWX install: AWX Operator
Namespace: awx
Kubernetes: k3s
Container runtime: k3s containerd
GitLab registry: http://registry.midhtech.local:5050
GitLab VM IP: 192.168.1.101
AWX VM hostname: awx.midhtech.local
```

Required AWX-side setup:

1. Attach the AWX `gitlab-container-registry` credential to `ee-aws-ssm`.
2. Configure k3s/containerd to use HTTP for the internal GitLab registry.
3. Ensure the AWX VM can resolve `registry.midhtech.local`.

k3s registry config:

```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  "registry.midhtech.local:5050":
    endpoint:
      - "http://registry.midhtech.local:5050"
```

Restart k3s after creating or changing that file:

```bash
systemctl restart k3s
```

The AWX VM also needs this host mapping or equivalent DNS:

```text
192.168.1.101 gitlab.midhtech.local registry.midhtech.local
```

Useful symptoms:

```text
ImagePullBackOff
  AWX could not start the EE container.

failed to do request: Head "http://registry.midhtech.local:5050/..."
  k3s is using the HTTP registry mirror.

lookup registry.midhtech.local: Try again
  AWX VM DNS cannot resolve the registry name.

401 Unauthorized
  Registry is reachable; check the AWX container registry credential.
```

The GitLab runner `docker:dind --insecure-registry` setting only affects CI
build/push. It does not configure AWX job image pulls.

## Session Manager Plugin

The `community.aws.aws_ssm` connection plugin shells out to the AWS Session
Manager plugin. If this executable is missing, AWX jobs fail after the EE starts
with:

```text
failed to find the executable specified /usr/local/bin/session-manager-plugin
```

This EE installs the plugin during the image build and creates this stable path:

```text
/usr/local/bin/session-manager-plugin
```

The shared smoke test checks for it with:

```text
EE_SMOKE_EXECUTABLES=session-manager-plugin
```

After changing the EE definition, build and publish a new semantic version tag,
then rerun the manual AWX EE registration job so AWX points to the new immutable
image tag.

## SSM Transfer Bucket

The `amazon.aws.aws_ssm` connection plugin commonly uses S3 for module transfer. Do not use an audit logging bucket for this.

Use a dedicated encrypted bucket, for example:

```text
awx-ssm-transfer-dev
```

Recommended controls:

- SSE-KMS encryption
- block public access
- lifecycle expiration for temporary objects
- access limited to the AWX automation role/user
