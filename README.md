





## Usage


**IMPORTANT:** This module has  differences against any other module, since we need a primary zone and a DR zone we need to pass explicit provider within the module, we can't use variables or dynamic providers to create the infraestructure with a single module call.
Also, following the best practices we decided to separate the creation of the aurora global cluster from the core of the module.


[Basic usage](examples/basic)

```hcl
provider "aws" {
  region = "us-west-2"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "east"
  region = "us-east-1"
}


resource "aws_rds_global_cluster" "main" {
  provider                  = aws.west
  engine                    = "aurora-postgresql"
  engine_version            = 11.8
  storage_encrypted         = "true"
  global_cluster_identifier = "aurora-global-database"
  deletion_protection       = false
  lifecycle {
    ignore_changes = [engine_version]
  }
}


module "aurora_primary" {
  source = ""

  providers = {
    aws = aws.west
  }
  name                      = "aurora-primary"
  enable_global_cluster     = true
  global_cluster_identifier = aws_rds_global_cluster.main.id
  secret_name               = "secret-region-west-2"
  vpc_id                    = "vpc-0d205098237fadeb3"
  subnets                   = ["subnet-001765f8fb9f94b92", "subnet-01b0a16127c1c6f2b"]
  replica_count             = 1
  instance_class            = "db.r5.large" 
  family     				        = "aurora-postgresql11"
  depends_on 				        = [aws_rds_global_cluster.main]
}

module "aurora_secondary" {
  source = ""

  providers = {
    aws = aws.east
  }
  name                      = "aurora-secondary"
  enable_global_cluster     = true
  source_region             = "us-west-2"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  create_secret             = false
  vpc_id                    = "vpc-0657d91b8d0f0535b"
  subnets                   = ["subnet-0f4bac0a7ac27db63", "subnet-0467814864e7e5ef2"]
  replica_count             = 1
  username                  = null
  db_master_password        = null
  instance_class            = "db.r5.large"
  family                    = "aurora-postgresql11"
  tags 						          = map("Created-By", "Emmanuel Torrado")
  depends_on                = [module.aurora_primary]
}
```


## Specifing an existing CIDR blocks to security groups

```hcl
provider "aws" {
  region = "us-west-2"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "east"
  region = "us-east-1"
}


resource "aws_rds_global_cluster" "main" {
  provider                  = aws.west
  engine                    = "aurora-postgresql"
  engine_version            = 11.8
  storage_encrypted         = "true"
  global_cluster_identifier = "aurora-global-database"
  deletion_protection       = false
  lifecycle {
    ignore_changes = [engine_version]
  }
}

module "aurora_primary" {
  source = ""

  providers = {
    aws = aws.west
  }
  name                      = "aurora-primary"
  enable_global_cluster     = true
  global_cluster_identifier = aws_rds_global_cluster.main.id
  secret_name               = "secret-region-west-2"
  vpc_id                    = "vpc-0d205098237fadeb3"
  subnets                   = ["subnet-001765f8fb9f94b92", "subnet-01b0a16127c1c6f2b"]
  replica_count             = 1
  instance_class            = "db.r5.large" 
  family     				        = "aurora-postgresql11"
  depends_on 				        = [aws_rds_global_cluster.main]
  existing_cidr_block       = ["10.0.0.0/8"]
}

module "aurora_secondary" {
  source = ""

  providers = {
    aws = aws.east
  }
  name                      = "aurora-secondary"
  enable_global_cluster     = true
  source_region             = "us-west-2"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  create_secret             = false
  vpc_id                    = "vpc-0657d91b8d0f0535b"
  subnets                   = ["subnet-0f4bac0a7ac27db63", "subnet-0467814864e7e5ef2"]
  replica_count             = 1
  username                  = null
  db_master_password        = null
  instance_class            = "db.r5.large"
  family                    = "aurora-postgresql11"
  tags 						          = map("Created-By", "Emmanuel Torrado")
  depends_on                = [module.aurora_primary]
}
```

## Requirements

| Name | Version |
|------|---------|
| terraform | >= 0.13 |
| aws | >= 2.0 |
| null | >= 2.0 |

## Providers

| Name | Version |
|------|---------|
| aws | >= 2.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| apply\_immediately | Determines whether or not any DB modifications are applied immediately, or during the maintenance window. | `bool` | `false` | no |
| create\_cluster | Controls if the RDS cluster should be created | `bool` | `true` | yes |
| create\_secret | Control if you need to create a secret, if false, it will just not store the admin password and user in the secret manager | `bool` | `"true"` | no |
| create\_db\_subnet | Controls if the DB Subnet Group should be created, if set to false you can use an existing one | `bool` | `true` | no |
| create\_db\_parameter\_group | Controls if you needto create a new db parameter group for the instances in the cluster, if set to false you can use an existing one | `bool` | `true` | no |
| create\_db\_cluster\_parameter\_group | Controls if you need to create a new db cluster parameter group for the cluster, if set to false you can use an existing one | `bool` | `true` | no |
| create\_security\_group | Controls if the security group creation is required within the cluster, if not, you can use an existing one, we highly recommend to create new one | `bool` | `true` | no |
| db\_parameter\_group\_name | You can specifiy a db parameter group name that already exist if you dont want to create one new, if empty it will create a new one. | `string` | `""` | no |
| db\_subnet\_group\_name | You can specify a subnet group that already exist if you dont want to create one new, if empty it will create a new on. | `string` | `""` | no |
| db\_master\_password | Controls the master password of the db | `string` | `""` | no |
| database\_name | Tje datanase name, in Aurora Replica you can't specify a custom database name. | `bool` | `false` | no |
| deletion\_protection | If the DB instance should have deletion protection enabled | `bool` | `false` | no |
| db\_port | The port to accept connections | `number` | `5432` | no |
| existing\_security\_groups | List of security groups to be allowed to connect to the DB instance | `list(string)` | `[]` | no |
| existing\_cidr\_blocks | Allowing inbound traffic to the DB for existing CIDR | `list(string)` | `[]` | no |
| existing\_cidr\_blocks | Allowing inbound traffic to the DB for existing CIDR | `list(string)` | `[]` | no |
| engine | The name of the database engine to be used for this database cluster. Allowed values are aurora-postgresql and aurora-mysql | `string` | `aurora-postgresql` | no |
| engine\_version | The database engine version| `string` | `11.8` | no |
| engine\_mode | The database engine mode| `string` | `global` | no |
| family | The family of the DB cluster parameter group| `string` | `aurora-postgresql11` | no|
| final\_snapshot\_identifier | The prefix name to use when creating a final snapshot on cluster destroy, appends a random 8 digits to name to ensure it's unique too.| `bool` | `true` | no|
| global\_cluster\_identifier | The global cluster identifier specified on aws_rds_global_cluster.| `string` | `""` | no|
