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
