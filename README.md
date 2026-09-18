

## Architecture decisions

### Why Fargate and not EC2 here
There is no ASG and no launch template in this stack — `ecs.tf` creates the cluster (Container Insights enabled) and a capacity provider set, and nothing else provisions compute. The EC2 path is visibly *not* taken: `templates/user-data.tpl` and the `nodes_ami` / `node_instance_type` / `node_volume_*` / `cluster_on_demand_*` / `cluster_spot_*` variables are still declared, and `environment/dev/terraform.tfvars` still assigns them (a pinned AMI, `c5.large`, 50 GiB gp3), but no resource consumes any of it. I rejected EC2 capacity providers because the cost is permanent: AMI patching, ECS agent upgrades, managed termination protection and instance draining on scale-in, plus paying for idle headroom so a deploy has somewhere to land.

### FARGATE_SPOT is allowed on the cluster but kept out of the default strategy
`capacity_providers` defaults to `["FARGATE", "FARGATE_SPOT"]`, but `default_capacity_provider_strategy` pins `base = 1`, `weight = 100` on `FARGATE` alone. Spot is available, never inherited. I rejected weighting spot in the cluster default because every service that ships without its own strategy would silently accept two-minute reclamation notices — including the ones I never intended to make interruptible.

### An NLB in front of the internal ALB, not one NLB per service
`aws_api_gateway_vpc_link` is the REST-API VPC Link, which accepts only NLB ARNs. Instead of putting services behind NLBs to satisfy that, I stood up one internal NLB (`vpc_link.tf`) and registered the internal ALB in an `alb`-type target group behind it. Rejecting this means one NLB per exposed service: an hourly bill each, and L4 only, so host/path routing disappears and every routing change moves into API Gateway.

### Three DNS planes instead of one shared namespace
Cloud Map (`<project>.discovery.com`), Service Connect (`<project>.local`) and a Route53 private zone `<project>.internal.com`, whose `*` record aliases the internal ALB, are separate on purpose. Collapsing Cloud Map and Service Connect into one namespace is the tempting shortcut; the cost is that a half-finished migration leaves two resolution paths answering the same hostname, and Cloud Map will not let you delete a namespace that still has services registered.

### The cluster publishes a contract to SSM, not to its state file
Both load balancer and listener ARNs, both discovery namespace IDs, the Service Connect name and the VPC Link ID are written to flat `/aws/ecs/*` parameters, and the ingress listener's default action is a fixed `200` rather than a target group. Service stacks read those parameters and attach their own rules. `terraform_remote_state` was the alternative: it would grant every service stack read access to this entire state file and make each deploy a cross-stack refresh. Two costs I accepted: the flat parameter paths allow exactly one cluster per account/region, and an unmatched request returns `200`, so a naive uptime check on the ALB root passes with zero healthy services.

<!-- BEGIN_TF_DOCS -->
## Requirements

No requirements.

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | 5.43.0 |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [aws_ecs_cluster.main](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecs_cluster) | resource |
| [aws_ecs_cluster_capacity_providers.main](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecs_cluster_capacity_providers) | resource |
| [aws_lb.main](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb) | resource |
| [aws_lb_listener.main](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener) | resource |
| [aws_security_group.lb](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [aws_security_group.main](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [aws_security_group_rule.ingress_443](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_security_group_rule.ingress_80](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_security_group_rule.subnet_ranges](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |
| [aws_ssm_parameter.lb_arn](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter) | resource |
| [aws_ssm_parameter.lb_listener](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter) | resource |
| [aws_ssm_parameter.subnet_private_1a](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter) | data source |
| [aws_ssm_parameter.subnet_private_1b](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter) | data source |
| [aws_ssm_parameter.subnet_private_1c](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter) | data source |
| [aws_ssm_parameter.subnet_public_1a](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter) | data source |
| [aws_ssm_parameter.subnet_public_1b](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter) | data source |
| [aws_ssm_parameter.subnet_public_1c](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter) | data source |
| [aws_ssm_parameter.vpc](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ssm_parameter) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_capacity_providers"></a> [capacity\_providers](#input\_capacity\_providers) | The list of capacity providers that will be allowed in the fargate cluster | `list` | <pre>[<br>  "FARGATE",<br>  "FARGATE_SPOT"<br>]</pre> | no |
| <a name="input_cluster_on_demand_desired_size"></a> [cluster\_on\_demand\_desired\_size](#input\_cluster\_on\_demand\_desired\_size) | The desired number of on-demand instances in the ECS cluster. | `number` | n/a | yes |
| <a name="input_cluster_on_demand_max_size"></a> [cluster\_on\_demand\_max\_size](#input\_cluster\_on\_demand\_max\_size) | The maximum size of the ECS cluster for on-demand instances. | `number` | n/a | yes |
| <a name="input_cluster_on_demand_min_size"></a> [cluster\_on\_demand\_min\_size](#input\_cluster\_on\_demand\_min\_size) | The minimum size of the ECS cluster for on-demand instances. | `number` | n/a | yes |
| <a name="input_cluster_spot_desired_size"></a> [cluster\_spot\_desired\_size](#input\_cluster\_spot\_desired\_size) | The desired number of spot instances in the ECS cluster. | `number` | n/a | yes |
| <a name="input_cluster_spot_max_size"></a> [cluster\_spot\_max\_size](#input\_cluster\_spot\_max\_size) | The maximum size of the ECS cluster for spot instances. | `number` | n/a | yes |
| <a name="input_cluster_spot_min_size"></a> [cluster\_spot\_min\_size](#input\_cluster\_spot\_min\_size) | The minimum size of the ECS cluster for spot instances. | `number` | n/a | yes |
| <a name="input_load_balancer_internal"></a> [load\_balancer\_internal](#input\_load\_balancer\_internal) | Defines whether the Load Balancer should be internal (true) or external (false). | `bool` | n/a | yes |
| <a name="input_load_balancer_type"></a> [load\_balancer\_type](#input\_load\_balancer\_type) | The type of Load Balancer to be created (e.g.: 'application' or 'network'). | `string` | n/a | yes |
| <a name="input_node_instance_type"></a> [node\_instance\_type](#input\_node\_instance\_type) | The EC2 instance type to be used by the ECS nodes. | `string` | n/a | yes |
| <a name="input_node_volume_size"></a> [node\_volume\_size](#input\_node\_volume\_size) | The volume size, in GiB, to be used by the ECS nodes. | `number` | n/a | yes |
| <a name="input_node_volume_type"></a> [node\_volume\_type](#input\_node\_volume\_type) | The EBS volume type to be used by the ECS nodes (e.g.: 'gp2', 'io1'). | `string` | n/a | yes |
| <a name="input_nodes_ami"></a> [nodes\_ami](#input\_nodes\_ami) | The AMI to be used by the ECS cluster nodes. | `string` | n/a | yes |
| <a name="input_project_name"></a> [project\_name](#input\_project\_name) | The project name, used to name resources in the scope of this Terraform. | `string` | n/a | yes |
| <a name="input_region"></a> [region](#input\_region) | The AWS region where resources will be created. | `string` | n/a | yes |
| <a name="input_ssm_private_subnet_1"></a> [ssm\_private\_subnet\_1](#input\_ssm\_private\_subnet\_1) | The ID of the first private subnet in the VPC for SSM resources. | `string` | n/a | yes |
| <a name="input_ssm_private_subnet_2"></a> [ssm\_private\_subnet\_2](#input\_ssm\_private\_subnet\_2) | The ID of the second private subnet in the VPC for SSM resources. | `string` | n/a | yes |
| <a name="input_ssm_private_subnet_3"></a> [ssm\_private\_subnet\_3](#input\_ssm\_private\_subnet\_3) | The ID of the third private subnet in the VPC for SSM resources. | `string` | n/a | yes |
| <a name="input_ssm_public_subnet_1"></a> [ssm\_public\_subnet\_1](#input\_ssm\_public\_subnet\_1) | The ID of the first public subnet in the VPC for SSM resources. | `string` | n/a | yes |
| <a name="input_ssm_public_subnet_2"></a> [ssm\_public\_subnet\_2](#input\_ssm\_public\_subnet\_2) | The ID of the second public subnet in the VPC for SSM resources. | `string` | n/a | yes |
| <a name="input_ssm_public_subnet_3"></a> [ssm\_public\_subnet\_3](#input\_ssm\_public\_subnet\_3) | The ID of the third public subnet in the VPC for SSM resources. | `string` | n/a | yes |
| <a name="input_ssm_vpc_id"></a> [ssm\_vpc\_id](#input\_ssm\_vpc\_id) | The VPC ID where SSM-related resources will be created. | `string` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_lb_ssm_arn"></a> [lb\_ssm\_arn](#output\_lb\_ssm\_arn) | The Amazon Resource Name (ARN) of the AWS Systems Manager (SSM) parameter that stores the Load Balancer ARN. This value can be used to reference the Load Balancer ARN in IAM policies, security rules, or anywhere else that requires the Load Balancer ARN. |
| <a name="output_lb_ssm_listener"></a> [lb\_ssm\_listener](#output\_lb\_ssm\_listener) | The ID of the AWS Systems Manager (SSM) parameter that stores the Load Balancer Listener. This value can be used to reference the Listener in automations, scripts, or within other AWS configurations that require the Listener ID. |
| <a name="output_load_balancer_dns"></a> [load\_balancer\_dns](#output\_load\_balancer\_dns) | The DNS name of the created Load Balancer. This value can be used to access the Load Balancer within the network or from the internet, depending on the configuration. |
<!-- END_TF_DOCS -->
