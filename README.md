





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
  family     				= "aurora-postgresql11"
  depends_on 				= [aws_rds_global_cluster.main]
}

module "aurora_secondary" {
  source = ""

  providers = {
    aws = aws.east
  }
  name = 					"aurora-secondary"
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
  tags 						= map("Created-By", "Emmanuel Torrado")
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
  family     				= "aurora-postgresql11"
  depends_on 				= [aws_rds_global_cluster.main]
  existing_cidr_block       = ["10.0.0.0/8"]
}

module "aurora_secondary" {
  source = ""

  providers = {
    aws = aws.east
  }
  name = 					"aurora-secondary"
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
  tags 						= map("Created-By", "Emmanuel Torrado")
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
| deletion\_protection | If the DB instance should have deletion protection enabled | `number` | `300` | no |
| autoscaling\_scale\_out\_cooldown | The amount of time, in seconds, after a scaling activity completes and before the next scaling up activity can start. Default is 300s | `number` | `300` | no |
| autoscaling\_target\_metrics | The metrics type to use. If this value isn't provided the default is CPU utilization | `string` | `"RDSReaderAverageCPUUtilization"` | no |
| autoscaling\_target\_value | The target value to scale with respect to target metrics | `number` | `75` | no |
| backtrack\_window | The target backtrack window, in seconds. Only available for aurora engine currently. Must be between 0 and 259200 (72 hours) | `number` | `0` | no |
| backup\_window | Daily time range during which the backups happen | `string` | `"07:00-09:00"` | no |
| cluster\_dns\_name | Name of the cluster CNAME record to create in the parent DNS zone specified by `zone_id`. If left empty, the name will be auto-asigned using the format `master.var.name` | `string` | `""` | no |
| cluster\_family | The family of the DB cluster parameter group | `string` | `"aurora5.6"` | no |
| cluster\_identifier | The RDS Cluster Identifier. Will use generated label ID if not supplied | `string` | `""` | no |
| cluster\_parameters | List of DB cluster parameters to apply | <pre>list(object({<br>    apply_method = string<br>    name         = string<br>    value        = string<br>  }))</pre> | `[]` | no |
| cluster\_size | Number of DB instances to create in the cluster | `number` | `2` | no |
| context | Single object for setting entire context at once.<br>See description of individual variables for details.<br>Leave string and numeric variables as `null` to use default value.<br>Individual variable settings (non-null) override settings in context object,<br>except for attributes, tags, and additional\_tag\_map, which are merged. | <pre>object({<br>    enabled             = bool<br>    namespace           = string<br>    environment         = string<br>    stage               = string<br>    name                = string<br>    delimiter           = string<br>    attributes          = list(string)<br>    tags                = map(string)<br>    additional_tag_map  = map(string)<br>    regex_replace_chars = string<br>    label_order         = list(string)<br>    id_length_limit     = number<br>  })</pre> | <pre>{<br>  "additional_tag_map": {},<br>  "attributes": [],<br>  "delimiter": null,<br>  "enabled": true,<br>  "environment": null,<br>  "id_length_limit": null,<br>  "label_order": [],<br>  "name": null,<br>  "namespace": null,<br>  "regex_replace_chars": null,<br>  "stage": null,<br>  "tags": {}<br>}</pre> | no |
| copy\_tags\_to\_snapshot | Copy tags to backup snapshots | `bool` | `false` | no |
| db\_name | Database name (default is not to create a database) | `string` | `""` | no |
| db\_port | Database port | `number` | `3306` | no |
| deletion\_protection | If the DB instance should have deletion protection enabled | `bool` | `false` | no |
| delimiter | Delimiter to be used between `namespace`, `environment`, `stage`, `name` and `attributes`.<br>Defaults to `-` (hyphen). Set to `""` to use no delimiter at all. | `string` | `null` | no |
| enable\_http\_endpoint | Enable HTTP endpoint (data API). Only valid when engine\_mode is set to serverless | `bool` | `false` | no |
| enabled | Set to false to prevent the module from creating any resources | `bool` | `null` | no |
| enabled\_cloudwatch\_logs\_exports | List of log types to export to cloudwatch. The following log types are supported: audit, error, general, slowquery | `list(string)` | `[]` | no |
| engine | The name of the database engine to be used for this DB cluster. Valid values: `aurora`, `aurora-mysql`, `aurora-postgresql` | `string` | `"aurora"` | no |
| engine\_mode | The database engine mode. Valid values: `parallelquery`, `provisioned`, `serverless` | `string` | `"provisioned"` | no |
| engine\_version | The version of the database engine to use. See `aws rds describe-db-engine-versions` | `string` | `""` | no |
| enhanced\_monitoring\_role\_enabled | A boolean flag to enable/disable the creation of the enhanced monitoring IAM role. If set to `false`, the module will not create a new role and will use `rds_monitoring_role_arn` for enhanced monitoring | `bool` | `false` | no |
| environment | Environment, e.g. 'uw2', 'us-west-2', OR 'prod', 'staging', 'dev', 'UAT' | `string` | `null` | no |
| global\_cluster\_identifier | ID of the Aurora global cluster | `string` | `""` | no |
| iam\_database\_authentication\_enabled | Specifies whether or mappings of AWS Identity and Access Management (IAM) accounts to database accounts is enabled | `bool` | `false` | no |
| iam\_roles | Iam roles for the Aurora cluster | `list(string)` | `[]` | no |
| id\_length\_limit | Limit `id` to this many characters.<br>Set to `0` for unlimited length.<br>Set to `null` for default, which is `0`.<br>Does not affect `id_full`. | `number` | `null` | no |
| instance\_availability\_zone | Optional parameter to place cluster instances in a specific availability zone. If left empty, will place randomly | `string` | `""` | no |
| instance\_parameters | List of DB instance parameters to apply | <pre>list(object({<br>    apply_method = string<br>    name         = string<br>    value        = string<br>  }))</pre> | `[]` | no |
| instance\_type | Instance type to use | `string` | `"db.t2.small"` | no |
| kms\_key\_arn | The ARN for the KMS encryption key. When specifying `kms_key_arn`, `storage_encrypted` needs to be set to `true` | `string` | `""` | no |
| label\_order | The naming order of the id output and Name tag.<br>Defaults to ["namespace", "environment", "stage", "name", "attributes"].<br>You can omit any of the 5 elements, but at least one must be present. | `list(string)` | `null` | no |
| maintenance\_window | Weekly time range during which system maintenance can occur, in UTC | `string` | `"wed:03:00-wed:04:00"` | no |
| name | Solution name, e.g. 'app' or 'jenkins' | `string` | `null` | no |
| namespace | Namespace, which could be your organization name or abbreviation, e.g. 'eg' or 'cp' | `string` | `null` | no |
| performance\_insights\_enabled | Whether to enable Performance Insights | `bool` | `false` | no |
| performance\_insights\_kms\_key\_id | The ARN for the KMS key to encrypt Performance Insights data. When specifying `performance_insights_kms_key_id`, `performance_insights_enabled` needs to be set to true | `string` | `""` | no |
| publicly\_accessible | Set to true if you want your cluster to be publicly accessible (such as via QuickSight) | `bool` | `false` | no |
| rds\_monitoring\_interval | The interval, in seconds, between points when enhanced monitoring metrics are collected for the DB instance. To disable collecting Enhanced Monitoring metrics, specify 0. The default is 0. Valid Values: 0, 1, 5, 10, 15, 30, 60 | `number` | `0` | no |
| rds\_monitoring\_role\_arn | The ARN for the IAM role that permits RDS to send enhanced monitoring metrics to CloudWatch Logs | `string` | `null` | no |
| reader\_dns\_name | Name of the reader endpoint CNAME record to create in the parent DNS zone specified by `zone_id`. If left empty, the name will be auto-asigned using the format `replicas.var.name` | `string` | `""` | no |
| regex\_replace\_chars | Regex to replace chars with empty string in `namespace`, `environment`, `stage` and `name`.<br>If not set, `"/[^a-zA-Z0-9-]/"` is used to remove all characters other than hyphens, letters and digits. | `string` | `null` | no |
| replication\_source\_identifier | ARN of a source DB cluster or DB instance if this DB cluster is to be created as a Read Replica | `string` | `""` | no |
| restore\_to\_point\_in\_time | List point-in-time recovery options. Only valid actions are `source_cluster_identifier`, `restore_type` and `use_latest_restorable_time` | <pre>list(object({<br>    source_cluster_identifier  = string<br>    restore_type               = string<br>    use_latest_restorable_time = bool<br>  }))</pre> | `[]` | no |
| retention\_period | Number of days to retain backups for | `number` | `5` | no |
| s3\_import | Restore from a Percona Xtrabackup in S3. The `bucket_name` is required to be in the same region as the resource. | <pre>object({<br>    bucket_name           = string<br>    bucket_prefix         = string<br>    ingestion_role        = string<br>    source_engine         = string<br>    source_engine_version = string<br>  })</pre> | `null` | no |
| scaling\_configuration | List of nested attributes with scaling properties. Only valid when `engine_mode` is set to `serverless` | <pre>list(object({<br>    auto_pause               = bool<br>    max_capacity             = number<br>    min_capacity             = number<br>    seconds_until_auto_pause = number<br>    timeout_action           = string<br>  }))</pre> | `[]` | no |
| security\_groups | List of security groups to be allowed to connect to the DB instance | `list(string)` | `[]` | no |
| skip\_final\_snapshot | Determines whether a final DB snapshot is created before the DB cluster is deleted | `bool` | `true` | no |
| snapshot\_identifier | Specifies whether or not to create this cluster from a snapshot | `string` | `null` | no |
| source\_region | Source Region of primary cluster, needed when using encrypted storage and region replicas | `string` | `""` | no |
| stage | Stage, e.g. 'prod', 'staging', 'dev', OR 'source', 'build', 'test', 'deploy', 'release' | `string` | `null` | no |
| storage\_encrypted | Specifies whether the DB cluster is encrypted. The default is `false` for `provisioned` `engine_mode` and `true` for `serverless` `engine_mode` | `bool` | `false` | no |
| subnets | List of VPC subnet IDs | `list(string)` | n/a | yes |
| tags | Additional tags (e.g. `map('BusinessUnit','XYZ')` | `map(string)` | `{}` | no |
| timeouts\_configuration | List of timeout values per action. Only valid actions are `create`, `update` and `delete` | <pre>list(object({<br>    create = string<br>    update = string<br>    delete = string<br>  }))</pre> | `[]` | no |
| vpc\_id | VPC ID to create the cluster in (e.g. `vpc-a22222ee`) | `string` | n/a | yes |
| vpc\_security\_group\_ids | Additional security group IDs to apply to the cluster, in addition to the provisioned default security group with ingress traffic from existing CIDR blocks and existing security groups | `list(string)` | `[]` | no |
| zone\_id | Route53 parent zone ID. If provided (not empty), the module will create sub-domain DNS records for the DB master and replicas | `string` | `""` | no |

## Outputs

| Name | Description |
|------|-------------|
| arn | Amazon Resource Name (ARN) of the cluster |
| cluster\_identifier | Cluster Identifier |
| cluster\_resource\_id | The region-unique, immutable identifie of the cluster |
| cluster\_security\_groups | Default RDS cluster security groups |
| database\_name | Database name |
| dbi\_resource\_ids | List of the region-unique, immutable identifiers for the DB instances in the cluster |
| endpoint | The DNS address of the RDS instance |
| master\_host | DB Master hostname |
| master\_username | Username for the master DB user |
| reader\_endpoint | A read-only endpoint for the Aurora cluster, automatically load-balanced across replicas |
| replicas\_host | Replicas hostname |
| security\_group\_arn | Security Group ARN |
| security\_group\_id | Security Group ID |
| security\_group\_name | Security Group name |

<!-- markdownlint-restore -->



## Share the Love

Like this project? Please give it a ★ on [our GitHub](https://github.com/cloudposse/terraform-aws-rds-cluster)! (it helps us **a lot**)

Are you using this project or any of our other projects? Consider [leaving a testimonial][testimonial]. =)


## Related Projects

Check out these related projects.

- [terraform-aws-rds](https://github.com/cloudposse/terraform-aws-rds) - Terraform module to provision AWS RDS instances
- [terraform-aws-rds-cloudwatch-sns-alarms](https://github.com/cloudposse/terraform-aws-rds-cloudwatch-sns-alarms) - Terraform module that configures important RDS alerts using CloudWatch and sends them to an SNS topic



## Help

**Got a question?** We got answers.

File a GitHub [issue](https://github.com/cloudposse/terraform-aws-rds-cluster/issues), send us an [email][email] or join our [Slack Community][slack].

[![README Commercial Support][readme_commercial_support_img]][readme_commercial_support_link]

## DevOps Accelerator for Startups


We are a [**DevOps Accelerator**][commercial_support]. We'll help you build your cloud infrastructure from the ground up so you can own it. Then we'll show you how to operate it and stick around for as long as you need us.

[![Learn More](https://img.shields.io/badge/learn%20more-success.svg?style=for-the-badge)][commercial_support]

Work directly with our team of DevOps experts via email, slack, and video conferencing.

We deliver 10x the value for a fraction of the cost of a full-time engineer. Our track record is not even funny. If you want things done right and you need it done FAST, then we're your best bet.

- **Reference Architecture.** You'll get everything you need from the ground up built using 100% infrastructure as code.
- **Release Engineering.** You'll have end-to-end CI/CD with unlimited staging environments.
- **Site Reliability Engineering.** You'll have total visibility into your apps and microservices.
- **Security Baseline.** You'll have built-in governance with accountability and audit logs for all changes.
- **GitOps.** You'll be able to operate your infrastructure via Pull Requests.
- **Training.** You'll receive hands-on training so your team can operate what we build.
- **Questions.** You'll have a direct line of communication between our teams via a Shared Slack channel.
- **Troubleshooting.** You'll get help to triage when things aren't working.
- **Code Reviews.** You'll receive constructive feedback on Pull Requests.
- **Bug Fixes.** We'll rapidly work with you to fix any bugs in our projects.

## Slack Community

Join our [Open Source Community][slack] on Slack. It's **FREE** for everyone! Our "SweetOps" community is where you get to talk with others who share a similar vision for how to rollout and manage infrastructure. This is the best place to talk shop, ask questions, solicit feedback, and work together as a community to build totally *sweet* infrastructure.

## Discourse Forums

Participate in our [Discourse Forums][discourse]. Here you'll find answers to commonly asked questions. Most questions will be related to the enormous number of projects we support on our GitHub. Come here to collaborate on answers, find solutions, and get ideas about the products and services we value. It only takes a minute to get started! Just sign in with SSO using your GitHub account.

## Newsletter

Sign up for [our newsletter][newsletter] that covers everything on our technology radar.  Receive updates on what we're up to on GitHub as well as awesome new projects we discover.

## Office Hours

[Join us every Wednesday via Zoom][office_hours] for our weekly "Lunch & Learn" sessions. It's **FREE** for everyone!

[![zoom](https://img.cloudposse.com/fit-in/200x200/https://cloudposse.com/wp-content/uploads/2019/08/Powered-by-Zoom.png")][office_hours]

## Contributing

### Bug Reports & Feature Requests

Please use the [issue tracker](https://github.com/cloudposse/terraform-aws-rds-cluster/issues) to report any bugs or file feature requests.

### Developing

If you are interested in being a contributor and want to get involved in developing this project or [help out](https://cpco.io/help-out) with our other projects, we would love to hear from you! Shoot us an [email][email].

In general, PRs are welcome. We follow the typical "fork-and-pull" Git workflow.

 1. **Fork** the repo on GitHub
 2. **Clone** the project to your own machine
 3. **Commit** changes to your own branch
 4. **Push** your work back up to your fork
 5. Submit a **Pull Request** so that we can review your changes

**NOTE:** Be sure to merge the latest changes from "upstream" before making a pull request!


## Copyright

Copyright © 2017-2020 [Cloud Posse, LLC](https://cpco.io/copyright)



## License

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

See [LICENSE](LICENSE) for full details.

```text
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
```









## Trademarks

All other trademarks referenced herein are the property of their respective owners.

## About

This project is maintained and funded by [Cloud Posse, LLC][website]. Like it? Please let us know by [leaving a testimonial][testimonial]!

[![Cloud Posse][logo]][website]

We're a [DevOps Professional Services][hire] company based in Los Angeles, CA. We ❤️  [Open Source Software][we_love_open_source].

We offer [paid support][commercial_support] on all of our projects.

Check out [our other projects][github], [follow us on twitter][twitter], [apply for a job][jobs], or [hire us][hire] to help with your cloud strategy and implementation.



### Contributors

<!-- markdownlint-disable -->
|  [![Erik Osterman][osterman_avatar]][osterman_homepage]<br/>[Erik Osterman][osterman_homepage] | [![Igor Rodionov][goruha_avatar]][goruha_homepage]<br/>[Igor Rodionov][goruha_homepage] | [![Andriy Knysh][aknysh_avatar]][aknysh_homepage]<br/>[Andriy Knysh][aknysh_homepage] | [![Sarkis Varozian][sarkis_avatar]][sarkis_homepage]<br/>[Sarkis Varozian][sarkis_homepage] | [![Mike Crowe][mike-zipit_avatar]][mike-zipit_homepage]<br/>[Mike Crowe][mike-zipit_homepage] | [![Sergey Vasilyev][s2504s_avatar]][s2504s_homepage]<br/>[Sergey Vasilyev][s2504s_homepage] | [![Todor Todorov][tptodorov_avatar]][tptodorov_homepage]<br/>[Todor Todorov][tptodorov_homepage] | [![Lee Huffman][leehuffman_avatar]][leehuffman_homepage]<br/>[Lee Huffman][leehuffman_homepage] |
|---|---|---|---|---|---|---|---|
<!-- markdownlint-restore -->

  [osterman_homepage]: https://github.com/osterman
  [osterman_avatar]: https://img.cloudposse.com/150x150/https://github.com/osterman.png
  [goruha_homepage]: https://github.com/goruha
  [goruha_avatar]: https://img.cloudposse.com/150x150/https://github.com/goruha.png
  [aknysh_homepage]: https://github.com/aknysh
  [aknysh_avatar]: https://img.cloudposse.com/150x150/https://github.com/aknysh.png
  [sarkis_homepage]: https://github.com/sarkis
  [sarkis_avatar]: https://img.cloudposse.com/150x150/https://github.com/sarkis.png
  [mike-zipit_homepage]: https://github.com/mike-zipit
  [mike-zipit_avatar]: https://img.cloudposse.com/150x150/https://github.com/mike-zipit.png
  [s2504s_homepage]: https://github.com/s2504s
  [s2504s_avatar]: https://img.cloudposse.com/150x150/https://github.com/s2504s.png
  [tptodorov_homepage]: https://github.com/tptodorov
  [tptodorov_avatar]: https://img.cloudposse.com/150x150/https://github.com/tptodorov.png
  [leehuffman_homepage]: https://github.com/leehuffman
  [leehuffman_avatar]: https://img.cloudposse.com/150x150/https://github.com/leehuffman.png

[![README Footer][readme_footer_img]][readme_footer_link]
[![Beacon][beacon]][website]

  [logo]: https://cloudposse.com/logo-300x69.svg
  [docs]: https://cpco.io/docs?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=docs
  [website]: https://cpco.io/homepage?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=website
  [github]: https://cpco.io/github?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=github
  [jobs]: https://cpco.io/jobs?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=jobs
  [hire]: https://cpco.io/hire?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=hire
  [slack]: https://cpco.io/slack?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=slack
  [linkedin]: https://cpco.io/linkedin?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=linkedin
  [twitter]: https://cpco.io/twitter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=twitter
  [testimonial]: https://cpco.io/leave-testimonial?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=testimonial
  [office_hours]: https://cloudposse.com/office-hours?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=office_hours
  [newsletter]: https://cpco.io/newsletter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=newsletter
  [discourse]: https://ask.sweetops.com/?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=discourse
  [email]: https://cpco.io/email?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=email
  [commercial_support]: https://cpco.io/commercial-support?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=commercial_support
  [we_love_open_source]: https://cpco.io/we-love-open-source?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=we_love_open_source
  [terraform_modules]: https://cpco.io/terraform-modules?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=terraform_modules
  [readme_header_img]: https://cloudposse.com/readme/header/img
  [readme_header_link]: https://cloudposse.com/readme/header/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=readme_header_link
  [readme_footer_img]: https://cloudposse.com/readme/footer/img
  [readme_footer_link]: https://cloudposse.com/readme/footer/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=readme_footer_link
  [readme_commercial_support_img]: https://cloudposse.com/readme/commercial-support/img
  [readme_commercial_support_link]: https://cloudposse.com/readme/commercial-support/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-rds-cluster&utm_content=readme_commercial_support_link
  [share_twitter]: https://twitter.com/intent/tweet/?text=terraform-aws-rds-cluster&url=https://github.com/cloudposse/terraform-aws-rds-cluster
  [share_linkedin]: https://www.linkedin.com/shareArticle?mini=true&title=terraform-aws-rds-cluster&url=https://github.com/cloudposse/terraform-aws-rds-cluster
  [share_reddit]: https://reddit.com/submit/?url=https://github.com/cloudposse/terraform-aws-rds-cluster
  [share_facebook]: https://facebook.com/sharer/sharer.php?u=https://github.com/cloudposse/terraform-aws-rds-cluster
  [share_googleplus]: https://plus.google.com/share?url=https://github.com/cloudposse/terraform-aws-rds-cluster
  [share_email]: mailto:?subject=terraform-aws-rds-cluster&body=https://github.com/cloudposse/terraform-aws-rds-cluster
  [beacon]: https://ga-beacon.cloudposse.com/UA-76589703-4/cloudposse/terraform-aws-rds-cluster?pixel&cs=github&cm=readme&an=terraform-aws-rds-cluster
