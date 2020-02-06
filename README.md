# ACME Corporation - Vault Enterprise Implementation

This implementation provisions multiple Vault Enterprise clusters over the following regions:

## Background

## Architecture 

| AWS Region        | Role                                  | Size (Consul/Vault)   | Notes                                               |
| ----------------- | ------------------------------------- | --------------------- | --------------------------------------------------- |
| us-west-1         | Primary                               | (6/3)                 |                                                     |
| us-west-2         | DR Replica of Primary                 | (1/1)*                |                                                     |
| eu-central-1      | Performance Replica                   | (1/1)*                | Contains EU PII-related database decryption keys**  |
| eu-west-1         | DR Replica of EU Performance Replica  | (1/1)*                | Contains EU PII-related database decryption keys**  |
| ap-southeast-1    | Performance Replica                   | (1/1)*                |                                                     |

## Dynamic Database Credentials - MySQL Database Secrets Engine
Reference: [MySQL Database Secrets Engine](https://www.vaultproject.io/docs/secrets/databases/mssql/)

Prior to the Vault Enterprise implementation, ACME Corporation's website and ERP application relied on static credentials to connect to an RDS MySQL database. These credentials were scheduled for rotation during a downtime window every three months. Sometimes these downtime windows were missed, leading to back-to-back weeks of downtime windows. Furthermore, the static credentials being valid for three months also created a large window of of opportunity for a malicious actor to retrieve them and use them without authorization.

Use of Vault's Database Secrets Engine allows for the ERP and Website application to request dynamic database credentials from Vault with the help of [Vault Agent](https://www.vaultproject.io/docs/agent/): the agent performs [auto-auth](https://www.vaultproject.io/docs/agent/autoauth/) to the primary Vault cluster using the [EC2 Authentication Method](https://www.vaultproject.io/docs/auth/aws/) using an instance-id-bound-role (see fixtures and scripts in [use-cases/auth-methods/ec2](use-cases/auth-methods/ec2)). Then, a template declaration in the Vault Agent configuration references the secret path for the MySQL Database Secrets Engine. Vault Agent will request this secret from Vault, causing Vault to create a temporary user and password for MySQL for the length of the lease's period, revoking it once the lease has expired.

See the [use-cases/vault-agent](use-cases/vault-agent) fixtures and scripts to see how Vault Agent is set up.

In order to set up the policies for, and the Database Secret Engine itself, see the [use-cases/database-secrets-engine](use-cases/database-secrets-engine) fixtures and scripts.

## Database Encryption - Transit Secrets Engine
Reference: [Transit Secrets Engine](https://www.vaultproject.io/docs/secrets/transit/index.html)

Prior to the Vault Enterprise implementation, ACME Corporation's website and ERP application wrote unencrypted data to the same database. This meant that all data, including EU data would reside in plaintext. The [Transit Secrets Engine](https://www.vaultproject.io/docs/secrets/transit/index.html) is used to use a Vault-managed key to encrypt and decrypt data without persisting it into Vault, but rather persisting it to the database, and decrypting it using Vault when the data is needed again.

See [use-cases/transit-secrets-engine](use-cases/transit-secrets-engine) for the fixtures and scripts used to set up the Transit Secrets Engine.

## EU Data Protection - Mount Filters
Reference: [Mount Filters](https://www.vaultproject.io/guides/operations/mount-filter/)

With the Transit Secrets Engine, the ERP application is able to delegate on-the-fly encryption to Vault, such that sensitive data stored in the database is encrypted prior to write transactions and decrypted after read transactions. However, for EU data, a key reserved for encryption and decryption is written to the European Performance-Replica cluster (see [use-cases/transit-secrets-engine/script-vault-eu.sh](use-cases/transit-secrets-engine/script-vault-eu.sh)). [Mount Filters](https://www.vaultproject.io/guides/operations/mount-filter/) can then be used to ensure that these keys do not get replicated to non-european clusters, thus ensuring that only the applications connected via Vault Agent to the European Vault cluster will be able to decrypt this data.

## Systems Access Management - SSH Secrets Engine
Reference: [SSH Secrets Engine](https://www.vaultproject.io/docs/secrets/ssh/index.html)

Prior to the Vault Enterprise implementation, ACME Corporation's infrastructure relied on static SSH passwords to manage access to machines. This makes access difficult to manage in case an employee leaves. Even with SSH key-pairs, the allowed public keys on machines need to be managed in case of access management changes. Many organizations end up using a single key-pair because of the overhead required to effectively manage access management via SSH key-pairs (read [Why We need Dynamic Secrets by Armon Dadgar](https://www.hashicorp.com/blog/why-we-need-dynamic-secrets/)).

A binary that takes the role of a PAM module - [Vault SSH Helper](https://github.com/hashicorp/vault-ssh-helper) - can be used in tandem with Vault to manage access to servers by performing a challenge requesting a One-Time-Password created by Vault for the specific machine beforehand. The helper then reaches out to Vault to finalize the challenge, and will allow the authenticated entity on success. These OTP, like other Vault secrets, will eventually expire if they are unused. And as their name suggests, can only be used once, before the individual is required to return to Vault and generate a new OTP.

## Applying the Terraform configuration

Ensure that a `stable.tfvars` file exists, with the following keys set:

```
vault_ent_license="[ENTER VAULT ENT LICENSE HERE]"
consul_ent_license="[ENTER CONSUL ENT LICENSE HERE]"
```

Then, ensure you are passing the `stable.tfvars` file when performing a `terraform apply`:

```
terraform apply -var-file=stable.tfvars
```

## Accessing Provisioned Consul and Vault Instances

The provisioned EC2 Instances are pre-configured with the [AWS Systems Manager Agent (SSM Agent)](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html), and thus a secure shell can be accessed without using an SSH keypair, from the [SSM managed instances console](https://console.aws.amazon.com/systems-manager/managed-instances).

## Auto-Unseal

This implementation uses [AWS KMS Auto-Unseal Mechanism](https://www.vaultproject.io/docs/configuration/seal/awskms/) to encrypt the master key and auto-unseal the underlying storage. When initializing a cluster, use the script located in [use-cases/auto-unseal/](use-cases/auto-unseal/).
