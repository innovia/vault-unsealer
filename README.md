# Vault-unsealer

This project aims to make it easier to automate the secure unsealing of a Vault
server.

## Usage

```
This is a CLI tool to help automate the setup and management of
Hashicorp Vault.

It will continuously attempt to unseal the target Vault instance, by retrieving
unseal keys from a Google Cloud KMS keyring.

Usage:
  vault-unsealer [command]

Available Commands:
  help        Help about any command
  init        Initialise the target Vault instance
  unseal      A brief description of your command

Flags:
      --aws-kms-key-id string                The ID or ARN of the AWS KMS key to encrypt values
      --aws-ssm-key-prefix string            The Key Prefix for SSM Parameter store
      --google-cloud-kms-crypto-key string   The name of the Google Cloud KMS crypt key to use
      --google-cloud-kms-key-ring string     The name of the Google Cloud KMS key ring to use
      --google-cloud-kms-location string     The Google Cloud KMS location to use (eg. 'global', 'europe-west1')
      --google-cloud-kms-project string      The Google Cloud KMS project to use
      --google-cloud-storage-bucket string   The name of the Google Cloud Storage bucket to store values in
      --google-cloud-storage-prefix string   The prefix to use for values store in Google Cloud Storage
  -h, --help                                 help for vault-unsealer
      --mode string                          Select the mode to use 'google-cloud-kms-gcs' => Google Cloud Storage with encryption using Google KMS; 'aws-kms-ssm' => AWS SSM parameter store using AWS KMS encryption (default "google-cloud-kms-gcs")
      --secret-shares int                    Total count of secret shares that exist (default 1)
      --secret-threshold int                 Minimum required secret shares to unseal (default 1)

Use "vault-unsealer [command] --help" for more information about a command.
```

## Setup AWS
1. Create or use a KMS key in the region you want:
https://console.aws.amazon.com/iam/home?region=us-east-1#/encryptionKeys/us-east-1

2.note the alias name of the key for example:
`alias/vault`

3.note the ARN for the key.

3. Add the following IAM permissions to the identity where the vault-unsealer will run 
for example if running on kubernetes via KOPS run `kops edit cluster` and add the permissions under additional node policies

```yaml
spec:
  additionalPolicies:
  node: |
  {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameters"
            ],
            "Resource": [
                "arn:aws:ssm:<region>:<aws-account-id>:parameter/*"
            ]
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "ssm:PutParameter",
                "ssm:DeleteParameter"
            ],
            "Resource": [
                "arn:aws:ssm:<region>:<aws-account-id>:parameter/kubernetes-*"
            ]
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "kms:Get*",
                "kms:ListKeys",
                "kms:ListAliases"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "kms:DescribeKey",
                "kms:Encrypt",
                "kms:Decrypt"
            ],
            "Resource": [
                "arn:aws:kms:<region>:<aws-account-id>:key/4f43307f-bc31-4c32-9333-76cab2eb6cc7"
            ]
        },
```

### Setting up an existing vault (already initialized)
If you have the existing unseal keys and the root token you can use them to setup the AWS SSM Parameter store.

for the following procedeure use awscli.
```$ pip install awscli```

1. ```
$ export AWS_REGION=<region>
```
2. run `aws kms list-aliases` to find the key id for the key you want to encrypt with.
3. using the TargetKeyId, put the root token into the SSM
```
$ aws ssm put-parameter --name <Prefix>-vault-root --description "Vault Root Token" --type SecureString --key-id <target-key-id> --value <vault-root-token>
```
  
Add each key (zero based 0-4 for 5 keys) like so:

WIP

```
$ aws ssm put-parameter --name kubernetes-vault-unseal-0 --description "Vault Unseal <key number>" --type SecureString --key-id <target-key-id>  --value <unseal-key-num>
```



### Initializing a vault.
if your vault is not yet initialized you can initialized it using the parameter store as follow:
2. export AWS_REGION=<region> 
3.run the command `aws kms list-aliases` to get a list of the kms keys you need, you must use the alias name
```
{
    "Aliases": [
        {
            "AliasName": "alias/MyKmsKey",
            "AliasArn": "arn:aws:kms:us-west-2:1234567812:alias/myKMSKey",
            "TargetKeyId": "4e4ad8a2-20cf-4ffe-a55f-edd96ca41bef"
        },
```

```
$ vault-unsealer init --mode aws-kms-ssm --aws-kms-key-id alias/vault --aws-ssm-key-prefix kubernetes- --secret-shares 5 --secret-threshold 3
```

INFO[0015] root token stored in key store                key=vault-root

this will create 6 keys in the AWS SSM:

<your-prefix>-vault-root
<your-prefix>-vault-unsealer-0
<your-prefix>-vault-unsealer-1
<your-prefix>-vault-unsealer-2
<your-prefix>-vault-unsealer-3
<your-prefix>-vault-unsealer-4


## Building from source

```
$ git clone https://github.com/jetstack/vault-unsealer.git
```

```
$ cd vault-unsealer
```

```
$ export GOPATH=`pwd`
$ go get github.com/jetstack/vault-unsealer/cmd
$ export CI_COMMIT_TAG=<version>
$ export CI_COMMIT_SHA=$(git rev-parse HEAD)
$ make build

make Docker image:
```
$ docker build -t vault-unsealer:<version> .
```

