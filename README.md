# Veeam Kasten K10 Setup on Kubernetes

This repository provides a streamlined process for deploying Kasten K10, setting up a sample workload, configuring AWS S3 for backups, and creating a backup policy. It includes both Kubernetes manifest files and an optional Ansible playbook for AWS resource provisioning.

## Table of Contents

- [Veeam Kasten K10 Setup on Kubernetes](#veeam-kasten-k10-setup-on-kubernetes)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Setup Guide](#setup-guide)
    - [1. Deploy Kasten K10 Operator](#1-deploy-kasten-k10-operator)
    - [2. Create K10 Operand](#2-create-k10-operand)
    - [3. Deploy Sample Workload (Optional)](#3-deploy-sample-workload-optional)
    - [4. Annotate VolumeSnapshotClass and StorageClass](#4-annotate-volumesnapshotclass-and-storageclass)
    - [5. Prepare AWS S3 Bucket and IAM User](#5-prepare-aws-s3-bucket-and-iam-user)
      - [Option A: Using the Provided Ansible Playbook (Recommended)](#option-a-using-the-provided-ansible-playbook-recommended)
      - [Option B: Manual AWS Setup](#option-b-manual-aws-setup)
    - [6. Configure Kasten Bucket Profile](#6-configure-kasten-bucket-profile)
    - [7. Create Kasten Backup Policy](#7-create-kasten-backup-policy)

---

## Prerequisites

Before you begin, ensure you have:
* Access to an **OpenShift/Kubernetes cluster** with `oc` (or `kubectl`) configured.
* **Administrator privileges** on your Kubernetes cluster.
* **Ansible** installed if you plan to use the provided playbook for AWS resource provisioning.
* **`yq`** installed for easy YAML manipulation.
* An **AWS account** with programmatic access keys configured for the Ansible playbook or manual setup.

---

## Setup Guide

Follow these steps to set up Kasten K10 for backup and restore operations.

### 1. Deploy Kasten K10 Operator

First, deploy the Kasten K10 operator to your cluster.

```bash
oc create -f installation/
```

### 2. Create K10 Operand
Next, create the K10 operand which will deploy the Kasten K10 application itself.

```bash
oc create -f k10/
```

### 3. Deploy Sample Workload (Optional)
You can deploy a sample workload to test your backup policies. Skip this step if you have an existing workload.

```bash
oc create -f workload/
```

### 4. Annotate VolumeSnapshotClass and StorageClass
Kasten K10 requires specific annotations on your `VolumeSnapshotClass` and `StorageClass` to function correctly. Replace `<volume-snapshot-class>` and `<storage-class>` with the actual names from your cluster.

```bash
oc annotate volumesnapshotclass <volume-snapshot-class> k10.kasten.io/is-snapshot-class=true --overwrite
oc annotate storageclasses.storage.k8s.io <storage-class> k10.kasten.io/sc-supports-block-mode-exports=true --overwrite
```

### 5. Prepare AWS S3 Bucket and IAM User
You have two options to prepare your AWS S3 bucket and IAM user: using the provided Ansible playbook or setting them up manually.

#### Option A: Using the Provided Ansible Playbook (Recommended)
This Ansible playbook will automate the creation of an AWS S3 bucket, a dedicated IAM user with exclusive access to that bucket, and will save the user's credentials to a file.

1. **Configure Sensitive Variables:**
Fill in the sensitive details in `ansible/vars/sensitive_stub.yml`. This file is a template for your sensitive information.

2. **Encrypt Sensitive Variables:**
Encrypt your sensitive variables using Ansible Vault. This will create `ansible/vars/sensitive.yml`.

```bash
ansible-vault encrypt --output ./ansible/vars/sensitive.yml ./ansible/vars/sensitive_stub.yml
```

3. **Adjust Main Variables:**
Review and modify variables in `ansible/vars/main.yml` as needed, for example, to change the AWS region. **Ensure you change the `bucket_name` as it must be globally unique across all AWS users.**

4. **Install Ansible Requirements:**
Install any required Ansible collections.

```bash
ansible-galaxy install -r requirements.yml
```

5. **Run the Ansible Playbook:**
Execute the playbook to create your AWS resources. You will be prompted for your Ansible Vault password.

```bash
ansible-playbook ansible/playbook.yml -J
``` 

**Note on `BucketAlreadyExists` Error:**
If you encounter a `BucketAlreadyExists` error, it means the chosen bucket name is already taken by another AWS user. Select a different, unique name in `ansible/vars/main.yml` and try again.

Upon successful execution, the playbook will create a file named `<iam_user_name>_aws_credentials.yaml` in your `ansible/` directory. This file will contain the `access_key_id` and `secret_access_key` for the newly created IAM user.

#### Option B: Manual AWS Setup
If you prefer not to use Ansible, manually create an S3 bucket and an IAM user with programmatic access limited to that specific bucket in your AWS console. Ensure you record the `access_key_id` and `secret_access_key`.

### 6. Configure Kasten Bucket Profile
Now, you'll provide Kasten with the AWS credentials and bucket information to create a location profile.

1. **Extract and Encode Credentials:**
Retrieve the `access_key_id` and `secret_access_key` from the `ansible/<iam_user_name>_aws_credentials.yaml` file (created by Ansible, or your manually obtained credentials) and encode them in Base64. Then, use `yq` to inject these encoded values directly into `configuration/2-bucket-secret.yaml`.

```bash
access_key_id=$(yq -r '.access_key_id' ansible/<iam_user_name>_aws_credentials.yaml | tr -d '\n' | base64 -w 0)
secret_access_key=$(yq -r '.secret_access_key' ansible/<iam_user_name>_aws_credentials.yaml | tr -d '\n' | base64 -w 0)
yq -i ".data.aws_access_key_id = \"$access_key_id\"" configuration/2-bucket-secret.yaml
yq -i ".data.aws_secret_access_key = \"$secret_access_key\"" configuration/2-bucket-secret.yaml
```

You will also need to fill the other details (like bucket name, region etc.) in `configuration/3-bucket-profile.yaml` if they are not already set.

2. **Accept EULA and Create Profile:**
Deploy the Kasten location profile, which also implicitly accepts the EULA.

```bash
oc create -f configuration/
```
### 7. Create Kasten Backup Policy
Finally, create your Kasten backup policy to protect your workloads.

```bash
oc create -f backup/
```
You can now manage and run your backup policy directly from the **Kasten UI**. 