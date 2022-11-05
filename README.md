# Terraform-AWS-Secrets-Manager
## Create Secrets in AWS Secrets Manager using Terraform in Amazon account

main.tf creates the below components: 
Creates random password for user adminaccount in AWS secret(Masteraccoundb) 
Creates a secret named Masteraccoundb 
Creates a secret version that will contain AWS secret(Masteraccoundb) 
````
# Firstly create a random generated password to use in secrets.
 
resource "random_password" "password" {
  length           = 16
  special          = true
  override_special = "_%@"
}
 
# Creating a AWS secret for database master account (Masteraccoundb)
 
resource "aws_secretsmanager_secret" "secretmasterDB" {
   name = "Masteraccoundb"
}
 
# Creating a AWS secret versions for database master account (Masteraccoundb)
 
resource "aws_secretsmanager_secret_version" "sversion" {
  secret_id = aws_secretsmanager_secret.secretmasterDB.id
  secret_string = <<EOF
   {
    "username": "adminaccount",
    "password": "${random_password.password.result}"
   }
EOF
}
 
# Importing the AWS secrets created previously using arn.
 
data "aws_secretsmanager_secret" "secretmasterDB" {
  arn = aws_secretsmanager_secret.secretmasterDB.arn
}
 
# Importing the AWS secret version created previously using arn.
 
data "aws_secretsmanager_secret_version" "creds" {
  secret_id = data.aws_secretsmanager_secret.secretmasterDB.arn
}
 
# After importing the secrets storing into Locals
 
locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.creds.secret_string)
}
````

provider.tf allows Terraform to interact with AWS cloud using AWS API.

````
provider "aws" {
  region = "us-east-2"
}
````

## Creating Postgres database using Terraform with AWS Secrets in AWS Secret Manager
After secret keys and values are successfully added using Terraform, the next step is to use these AWS secrets as credentials for the database master account while creating the database.

Below code creates the database cluster using the AWS secrets master_username = local.db_creds.username and master_password = local.db_creds.password.

````
resource "aws_rds_cluster" "main" {
  cluster_identifier = "democluster"
  database_name = "maindb"
  master_username = local.db_creds.username
  master_password = local.db_creds.password
  port = 5432
  engine = "aurora-postgresql"
  engine_version = "13.7"
  db_subnet_group_name = "dbsubntg"  # Make sure you create this before manually
  storage_encrypted = true
}


resource "aws_rds_cluster_instance" "main" {
  count = 2
  identifier = "myinstance-${count.index + 1}"
  cluster_identifier = "${aws_rds_cluster.main.id}"
  instance_class = "db.r5.large"
  engine = "aurora-postgresql"
  engine_version = "13.7"
  db_subnet_group_name = "dbsubntg"
  publicly_accessible = true
}
````
<details>
<summary>Output</summary>
<pre>$ 
````
alp@master1:~/Terraform-AWS-Secrets-Manager$ terraform apply
random_password.password: Refreshing state... [id=none]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:

  # data.aws_secretsmanager_secret.secretmasterDB will be read during apply
  # (config refers to values not yet known)
 <= data "aws_secretsmanager_secret" "secretmasterDB" {
      + arn                 = (known after apply)
      + description         = (known after apply)
      + id                  = (known after apply)
      + kms_key_id          = (known after apply)
      + name                = (known after apply)
      + policy              = (known after apply)
      + rotation_enabled    = (known after apply)
      + rotation_lambda_arn = (known after apply)
      + rotation_rules      = (known after apply)
      + tags                = (known after apply)
    }

  # data.aws_secretsmanager_secret_version.creds will be read during apply
  # (config refers to values not yet known)
 <= data "aws_secretsmanager_secret_version" "creds" {
      + arn            = (known after apply)
      + id             = (known after apply)
      + secret_binary  = (sensitive value)
      + secret_id      = (known after apply)
      + secret_string  = (sensitive value)
      + version_id     = (known after apply)
      + version_stages = (known after apply)
    }

  # aws_rds_cluster.main will be created
  + resource "aws_rds_cluster" "main" {
      + allocated_storage               = (known after apply)
      + apply_immediately               = (known after apply)
      + arn                             = (known after apply)
      + availability_zones              = (known after apply)
      + backup_retention_period         = 1
      + cluster_identifier              = "rampupcluster"
      + cluster_identifier_prefix       = (known after apply)
      + cluster_members                 = (known after apply)
      + cluster_resource_id             = (known after apply)
      + copy_tags_to_snapshot           = false
      + database_name                   = "rampupdb"
      + db_cluster_parameter_group_name = (known after apply)
      + db_subnet_group_name            = "dbsubntg"
      + enable_global_write_forwarding  = false
      + enable_http_endpoint            = false
      + endpoint                        = (known after apply)
      + engine                          = "aurora-postgresql"
      + engine_mode                     = "provisioned"
      + engine_version                  = "13.7"
      + engine_version_actual           = (known after apply)
      + hosted_zone_id                  = (known after apply)
      + iam_roles                       = (known after apply)
      + id                              = (known after apply)
      + kms_key_id                      = (known after apply)
      + master_password                 = (sensitive value)
      + master_username                 = (known after apply)
      + network_type                    = (known after apply)
      + port                            = 5432
      + preferred_backup_window         = (known after apply)
      + preferred_maintenance_window    = (known after apply)
      + reader_endpoint                 = (known after apply)
      + skip_final_snapshot             = false
      + storage_encrypted               = true
      + tags_all                        = (known after apply)
      + vpc_security_group_ids          = (known after apply)
    }

  # aws_rds_cluster_instance.main[0] will be created
  + resource "aws_rds_cluster_instance" "main" {
      + apply_immediately                     = (known after apply)
      + arn                                   = (known after apply)
      + auto_minor_version_upgrade            = true
      + availability_zone                     = (known after apply)
      + ca_cert_identifier                    = (known after apply)
      + cluster_identifier                    = (known after apply)
      + copy_tags_to_snapshot                 = false
      + db_parameter_group_name               = (known after apply)
      + db_subnet_group_name                  = "dbsubntg"
      + dbi_resource_id                       = (known after apply)
      + endpoint                              = (known after apply)
      + engine                                = "aurora-postgresql"
      + engine_version                        = "13.7"
      + engine_version_actual                 = (known after apply)
      + id                                    = (known after apply)
      + identifier                            = "myinstance-1"
      + identifier_prefix                     = (known after apply)
      + instance_class                        = "db.r5.large"
      + kms_key_id                            = (known after apply)
      + monitoring_interval                   = 0
      + monitoring_role_arn                   = (known after apply)
      + network_type                          = (known after apply)
      + performance_insights_enabled          = (known after apply)
      + performance_insights_kms_key_id       = (known after apply)
      + performance_insights_retention_period = (known after apply)
      + port                                  = (known after apply)
      + preferred_backup_window               = (known after apply)
      + preferred_maintenance_window          = (known after apply)
      + promotion_tier                        = 0
      + publicly_accessible                   = true
      + storage_encrypted                     = (known after apply)
      + tags_all                              = (known after apply)
      + writer                                = (known after apply)
    }

  # aws_rds_cluster_instance.main[1] will be created
  + resource "aws_rds_cluster_instance" "main" {
      + apply_immediately                     = (known after apply)
      + arn                                   = (known after apply)
      + auto_minor_version_upgrade            = true
      + availability_zone                     = (known after apply)
      + ca_cert_identifier                    = (known after apply)
      + cluster_identifier                    = (known after apply)
      + copy_tags_to_snapshot                 = false
      + db_parameter_group_name               = (known after apply)
      + db_subnet_group_name                  = "dbsubntg"
      + dbi_resource_id                       = (known after apply)
      + endpoint                              = (known after apply)
      + engine                                = "aurora-postgresql"
      + engine_version                        = "13.7"
      + engine_version_actual                 = (known after apply)
      + id                                    = (known after apply)
      + identifier                            = "myinstance-2"
      + identifier_prefix                     = (known after apply)
      + instance_class                        = "db.r5.large"
      + kms_key_id                            = (known after apply)
      + monitoring_interval                   = 0
      + monitoring_role_arn                   = (known after apply)
      + network_type                          = (known after apply)
      + performance_insights_enabled          = (known after apply)
      + performance_insights_kms_key_id       = (known after apply)
      + performance_insights_retention_period = (known after apply)
      + port                                  = (known after apply)
      + preferred_backup_window               = (known after apply)
      + preferred_maintenance_window          = (known after apply)
      + promotion_tier                        = 0
      + publicly_accessible                   = true
      + storage_encrypted                     = (known after apply)
      + tags_all                              = (known after apply)
      + writer                                = (known after apply)
    }

  # aws_secretsmanager_secret.secretmasterDB will be created
  + resource "aws_secretsmanager_secret" "secretmasterDB" {
      + arn                            = (known after apply)
      + force_overwrite_replica_secret = false
      + id                             = (known after apply)
      + name                           = "Masteraccoundb"
      + name_prefix                    = (known after apply)
      + policy                         = (known after apply)
      + recovery_window_in_days        = 30
      + rotation_enabled               = (known after apply)
      + rotation_lambda_arn            = (known after apply)
      + tags_all                       = (known after apply)

      + replica {
          + kms_key_id         = (known after apply)
          + last_accessed_date = (known after apply)
          + region             = (known after apply)
          + status             = (known after apply)
          + status_message     = (known after apply)
        }

      + rotation_rules {
          + automatically_after_days = (known after apply)
        }
    }

  # aws_secretsmanager_secret_version.sversion will be created
  + resource "aws_secretsmanager_secret_version" "sversion" {
      + arn            = (known after apply)
      + id             = (known after apply)
      + secret_id      = (known after apply)
      + secret_string  = (sensitive value)
      + version_id     = (known after apply)
      + version_stages = (known after apply)
    }

Plan: 5 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_secretsmanager_secret.secretmasterDB: Creating...
aws_secretsmanager_secret.secretmasterDB: Creation complete after 6s [id=arn:aws:secretsmanager:us-east-2:647065240363:secret:Masteraccoundb-mvsX5n]
data.aws_secretsmanager_secret.secretmasterDB: Reading...
aws_secretsmanager_secret_version.sversion: Creating...
data.aws_secretsmanager_secret.secretmasterDB: Read complete after 1s [id=arn:aws:secretsmanager:us-east-2:647065240363:secret:Masteraccoundb-mvsX5n]
data.aws_secretsmanager_secret_version.creds: Reading...
aws_secretsmanager_secret_version.sversion: Creation complete after 1s [id=arn:aws:secretsmanager:us-east-2:647065240363:secret:Masteraccoundb-mvsX5n|B0C56CB7-7E70-4BAC-9A09-05429FCC8BBB]
data.aws_secretsmanager_secret_version.creds: Read complete after 0s [id=arn:aws:secretsmanager:us-east-2:647065240363:secret:Masteraccoundb-mvsX5n|AWSCURRENT]
aws_rds_cluster.main: Creating...
aws_rds_cluster.main: Still creating... [10s elapsed]
aws_rds_cluster.main: Still creating... [20s elapsed]
aws_rds_cluster.main: Still creating... [30s elapsed]
aws_rds_cluster.main: Still creating... [40s elapsed]
aws_rds_cluster.main: Creation complete after 43s [id=rampupcluster]
aws_rds_cluster_instance.main[0]: Creating...
aws_rds_cluster_instance.main[1]: Creating...
aws_rds_cluster_instance.main[0]: Still creating... [10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [20s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [20s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [30s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [40s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [1m0s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [1m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [1m10s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [1m10s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [1m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [1m20s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [1m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [1m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [1m40s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [1m40s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [1m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [1m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [2m0s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [2m0s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [2m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [2m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [2m20s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [2m20s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [2m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [2m30s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [2m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [2m40s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [2m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [2m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [3m0s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [3m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [3m10s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [3m10s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [3m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [3m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [3m30s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [3m30s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [3m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [3m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [3m50s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [3m50s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [4m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [4m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [4m10s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [4m10s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [4m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [4m20s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [4m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [4m30s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [4m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [4m40s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [4m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [4m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [5m0s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [5m0s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [5m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [5m10s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [5m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [5m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [5m30s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [5m30s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [5m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [5m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [5m50s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [5m50s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [6m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [6m0s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [6m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [6m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [6m20s elapsed]
aws_rds_cluster_instance.main[0]: Still creating... [6m20s elapsed]
aws_rds_cluster_instance.main[0]: Creation complete after 6m21s [id=myinstance-1]
aws_rds_cluster_instance.main[1]: Still creating... [6m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [6m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [6m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [7m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [7m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [7m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [7m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [7m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [7m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [8m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [8m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [8m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [8m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [8m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [8m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [9m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [9m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [9m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [9m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [9m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [9m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [10m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [10m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [10m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [10m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [10m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [10m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [11m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [11m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [11m20s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [11m30s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [11m40s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [11m50s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [12m0s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [12m10s elapsed]
aws_rds_cluster_instance.main[1]: Still creating... [12m20s elapsed]
aws_rds_cluster_instance.main[1]: Creation complete after 12m21s [id=myinstance-2]

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
alp@master1:~/Terraform-AWS-Secrets-Manager$
````
</pre>
</details>
