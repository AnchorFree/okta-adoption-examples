# Okta + Vault + terraform + AWS dynamic credentials
This how-to will guide you through AWS dynamic secrets configuration based on Vault and STS Assume role. 

## Configure AWS account
### Create vault account and assing policy
First of all create an AWS user and assign following policy:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::*:role/vault-*"
        }
    ]
}
```

In this paritcular example, we believe that Vault should be able to assume roles with `vault-` prefix. Based on your security preferences, you may pick paricular roles which can be assumed. 
This particular policy doesn't enforce any account this role will be assumed in. In case you would like to secure that particular user - may want to set some account id. 


### Create AWS Role with `vault-*` prefix
You can add any permissions you like, the only requirement is to have `vault-*` prefix. In case you decided to use only limited set of predefined roles, don't forget to add the role to vault user policy. 

### Edit trust relationship
Basically, if you configuring it for one account, and you create vault user in this AWS account, you should set something like:
```
arn:aws:iam::{account-id}:user/vault 
```
Otherwise you are free to create any trust relations you like, e.g. you can allow access from Vault user from any other account you know about. By replcaing the `{account-id}` with your account id numbers. 

## Configure Vault

### Create secret backend and backend roles  
```
resource "vault_aws_secret_backend" "aws" {
  access_key  = "..."
  secret_key  = "..."
  path        = "my-unique-path"
  description = "Some description"
  default_lease_ttl_seconds = 900
  max_lease_ttl_seconds = 3600
}

resource "vault_aws_secret_backend_role" "example" {
  backend         = "${vault_aws_secret_backend.aws.path}"
  name            = "example"
  role_arns       = ["arn:aws:iam::{account-id}:role/vault-example"]
  credential_type = "assumed_role"
  max_sts_ttl     = 3600
  default_sts_ttl = 900
}

```

You may configure as many roles as you want. For instance you may want to give read access for secret backen with name `noc` to specific `noc` team. You may add all role_arns you belive they require for their work. You may want to add `sre` role with some extended permissions, etc. 

## Use from terraform

Now you should configure Vault to allow users with specific Okta group to read from the `my-unique-path` specific role, and actually use it from terraform:
```
data "vault_aws_access_credentials" "example" {
  backend = "my-unique-path"
  role    = "example"
  type    = "sts" # this is really important
}

provider "aws" {
  region = "${var.aws_region}"
  token  = "${data.vault_aws_access_credentials.example.security_token}"
}
```

