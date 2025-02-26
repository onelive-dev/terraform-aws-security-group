

## Usage


**IMPORTANT:** We do not pin modules to versions in our examples because of the
difficulty of keeping the versions in the documentation in sync with the latest released versions.
We highly recommend that in your code you pin the version to the exact version you are
using so that your infrastructure remains stable, and update versions in a
systematic way so that they do not catch you by surprise.

Also, because of a bug in the Terraform registry ([hashicorp/terraform#21417](https://github.com/hashicorp/terraform/issues/21417)),
the registry shows many of our inputs as required when in fact they are optional.
The table below correctly indicates which inputs are required.


This module is primarily for setting security group rules on a security group. You can provide the
ID of an existing security group to modify, or, by default, this module will create a new security
group and apply the given rules to it.

##### `rules` and `rules_map` inputs
This module provides 3 ways to set security group rules. You can use any or all of them at the same time.

The easy way to specify rules is via the `rules` input. It takes a list of rules. (We will define
a rule [a bit later](#definition-of-a-rule).) The problem is that a Terraform list must be composed
of elements that are all the exact same type, and rules can be any of several
different Terraform types. So to get around this restriction, the second
way to specify rules is via the `rules_map` input, which is more complex.

<details><summary>Why the input is so complex (click to reveal)</summary>

- Terraform has 3 basic simple types: bool, number, string
- Terraform then has 3 collections of simple types: list, map, and set
- Terraform then has 2 structural types: object and tuple. However, these are not really single
types. They are catch-all labels for values that are themselves combination of other values.
(This will become a bit clearer after we define `maps` and contrast them with `objects`)

One [rule of the collection types](https://www.terraform.io/docs/language/expressions/type-constraints.html#collection-types)
is that the values in the collections must all be the exact same type.
For example, you cannot have a list where some values are boolean and some are string. Maps require
that all keys be strings, but the map values can be any type, except again all the values in a map
must be the same type. In other words, the values of a map must form a valid list.

Objects look just like maps. The difference between an object and a map is that the values in an
object do not all have to be the same type.

The "type" of an object is itself an object: the keys are the same, and the values are the types of the values in the object.

So although `{ foo = "bar", baz = {} }` and `{ foo = "bar", baz = [] }` are both objects,
they are not of the same type. This means you cannot put them both in the same list or the same map,
even though you can put them in a single tuple or object.
Similarly, and closer to the problem at hand,

```hcl
cidr_rule = {
  type        = "ingress"
  cidr_blocks = ["0.0.0.0/0"]
}
```
is not the same type as
```hcl
self_rule = {
  type        = "ingress"
  self        = true
}
```
This means you cannot put both of those in the same list.
```hcl
rules = tolist([local.cidr_rule, local.self_rule])
```
Generates the error
```text
Invalid value for "v" parameter: cannot convert tuple to list of any single type.
```

You could make them the same type and put them in a list,
like this:
```hcl
rules = tolist([{
  type        = "ingress"
  cidr_blocks = ["0.0.0.0/0"]
  self        = null
},
{
  type        = "ingress"
  cidr_blocks = []
  self        = true
}])
```
That remains an option for you when generating the rules, and is probably better when you have full control over all the rules.
However, what if some of the rules are coming from a source outside of your control? You cannot simply add those rules
to your list. So, what to do? Create an object whose attributes' values can be of different types.
```hcl
{ mine = local.my_rules, theirs = var.their_rules }
```

That is why the `rules_map` input is available. It will accept a structure like that, an object whose
attribute values are lists of rules, where the lists themselves can be different types.

</summary>

The `rules_map` input takes an object.
- The attribute names (keys) of the object can be anything you want, but need to be known during `terraform plan`,
which means they cannot depend on any resources created or changed by Terraform.
- The values of the attributes are lists of rule objects, each object representing one Security Group Rule. As explained
  above in "Why the input is so complex", each object in the list must be exactly the same type. To use multiple types,
  you must put them in separate lists which are values of separate attributes.

###### Definition of a rule

For this module, a rule is defined as an object.
- The attributes and values of the rule objects are fully compatible (have the same keys and accept the same values) as the
Terraform [aws_security_group_rule resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule),
except
   - The `security_group_id` will be ignored, if present
   - You can include an optional `key` attribute. If present, its value must be unique among all security group rules in the
     security group, and it must be known in the Terraform "plan" phase, meaning it cannot depend on anything being
     generated or created by Terraform.

The `key` attribute value, if provided, will be used to identify the Security Group Rule to Terraform in order to
prevent Terraform from modifying it unnecessarily. If the `key` is not provided, Terraform will assign an identifier
based on the rule's position in its list, which can cause a ripple effect of rules being deleted and recreated if
a rule gets deleted from start of a list, causing all the other rules to shift position.
See ["Unexpected changes..."](#unexpected-changes-during-plan-and-apply) below for more details.


##### `rule_matrix` input
The other way to set rules is via the `rule_matrix` input. This splits the attributes of the `aws_security_group_rule`
resource into two sets: one set defines the rule and description, the other set defines the subjects of the rule.
Again, optional "key" values can provide stability, but cannot contain derived values.

As with `rules` and explained above in "Why the input is so complex", all elements of the list must be the exact same type.
This also holds for all the elements of the `rules_matrix.rules` list. Because `rule_matrix` is already
so complex, we do not provide the ability to mix types by packing object within more objects.
All of the elements of the `rule_matrix` list must be exactly the same type. You can make them all the same
type by following a few rules:

- Every object in a list must have the exact same set of attributes. Most attributes are optional and can be omitted,
  but any attribute appearing in one object must appear in all the objects.
- Any attribute that takes a list value in any object must contain a list in all objects.
  Use an empty list rather than `null` to indicate "no value". Passing in `null` instead of a list
  may cause Terraform to crash or emit confusing error messages (e.g. "number is required").
- Any attribute that takes a value of type other than list can be set to `null` in objects where no value is needed.

The schema for `rule_matrix` is:

```hcl
{
  # these top level lists define all the subjects to which rule_matrix rules will be applied
  key                       = an optional unique key to keep these rules from being affected when other rules change
  source_security_group_ids = list of source security group IDs to apply all rules to
  cidr_blocks               = list of ipv4 CIDR blocks to apply all rules to
  ipv6_cidr_blocks          = list of ipv6 CIDR blocks to apply all rules to
  prefix_list_ids           = list of prefix list IDs to apply all rules to

  self = boolean value; set it to "true" to apply the rules to the created or existing security group, null otherwise

  # each rule in the rules list will be applied to every subject defined above
  rules = [{
    key       = an optional unique key to keep this rule from being affected when other rules change
    type      = type of rule, either "ingress" or "egress"
    from_port = start range of protocol port
    to_port   = end range of protocol port, max is 65535
    protocol  = IP protocol name or number, or "-1" for all protocols and ports

    description = free form text description of the rule
  }]
}
```

##### Create before delete
This module provides a `create_before_delete` option that will, when a security group needs to be replaced,
cause Terraform to create the new one before deleting the old one. We recommend making this `true` for new security groups,
but we default it to `false` because if you import a security group with this setting `true`, that security
group will be deleted and replaced on the first `terraform apply`, which will likely cause a service outage.

### Important Notes

##### Unexpected changes during plan and apply
The way Terraform works and the way this module is implemented causes security group rules without keys
to be dependent on their place in the input lists. If a rule is deleted and the other rules therefore move
closer to the start of the list, those rules will be deleted and recreated. This should have no significant
operational impact, but it can make a small change look like a big one when viewing the output of
Terraform plan.

You can avoid this for the most part by providing the optional keys. Rules with keys will not be
changed if their keys do not change and the rules themselves do not change, except in the case of
`rule_matrix`, where the rules are still dependent on the order of the security groups in
`source_security_group_ids`. You can avoid this by using `rules` instead of `rule_matrix` when you have
more than one security group in the list.

##### WARNINGS and Caveats

**_Setting `inline_rules_enabled` is not recommended and NOT SUPPORTED_**: Any issues arising from setting
`inlne_rules_enabled = true` (including issues about setting it to `false` after setting it to `true`) will
not be addressed, because they flow from [fundamental problems](https://github.com/hashicorp/terraform-provider-aws/issues/20046)
with the underlying `aws_security_group` resource. The setting is provided for people who know and accept the
limitations and trade-offs and want to use it anyway. The main advantage is that when using inline rules,
Terraform will perform "drift detection" and attempt to remove any rules it finds in place but not
specified inline. See [this post](https://github.com/hashicorp/terraform-provider-aws/pull/9032#issuecomment-639545250)
for a discussion of the difference between inline and resource rules,
and some of the reasons inline rules are not satisfactory.

**_KNOWN ISSUE_** ([#20046](https://github.com/hashicorp/terraform-provider-aws/issues/20046)):
If you set `inline_rules_enabled = true`, you cannot later set it to `false`. If you try,
Terraform will [complain](https://github.com/hashicorp/terraform/pull/2376) and fail.
You will either have to delete and recreate the security group or manually delete all
the security group rules via the AWS console or CLI before applying `inline_rules_enabled = false`.

**_Objects not of the same type_**: Any time you provide a list of objects, Terraform requires that all objects in the list
must be [the exact same type](https://www.terraform.io/docs/language/expressions/type-constraints.html#dynamic-types-the-quot-any-quot-constraint).
This means that all objects in the list have exactly the same set of attributes and that each attribute has the same type
of value in every object. So while some attributes are optional for this module, if you include an attribute in any one of the objects in a list, then you
have to include that same attribute in all of them.  In rules where the key would othewise be omitted, include the key with value of `null`,
unless the value is a list type, in which case set the value to `[]` (an empty list), due to [#28137](https://github.com/hashicorp/terraform/issues/28137).




## Examples


See [examples/complete/main.tf](https://github.com/cloudposse/terraform-aws-security-group/examples/complete/main.tf) for
even more examples.

```hcl
module "label" {
  source = "cloudposse/label/null"
  # Cloud Posse recommends pinning every module to a specific version
  # version = "x.x.x"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["public"]
  delimiter  = "-"

  tags = {
    "BusinessUnit" = "XYZ",
    "Snapshot"     = "true"
  }
}

module "vpc" {
  source = "cloudposse/vpc/aws"
  # Cloud Posse recommends pinning every module to a specific version
  # version = "x.x.x"
  cidr_block = "10.0.0.0/16"

  context = module.label.context
}

module "sg" {
  source = "cloudposse/security-group/aws"
  # Cloud Posse recommends pinning every module to a specific version
  # version = "x.x.x"

  # Security Group names must be unique within a VPC.
  # This module follows Cloud Posse naming conventions and generates the name
  # based on the inputs to the null-label module, which means you cannot
  # reuse the label as-is for more than one security group in the VPC.
  #
  # Here we add an attribute to give the security group a unique name.
  attributes = ["primary"]

  # Allow unlimited egress
  allow_all_egress = true

  rules = [
    {
      key         = "ssh"
      type        = "ingress"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      self        = null
      description = "Allow SSH from anywhere"
    },
    {
      key         = "HTTP"
      type        = "ingress"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = []
      self        = true
      description = "Allow HTTP from inside the security group"
    }
  ]

  vpc_id  = module.vpc.vpc_id

  context = module.label.context
}

module "sg_mysql" {
  source = "cloudposse/security-group/aws"
  # Cloud Posse recommends pinning every module to a specific version
  # version = "x.x.x"

  # Add an attribute to give the Security Group a unique name
  attributes = ["mysql"]

  # Allow unlimited egress
  allow_all_egress = true

  rule_matrix =[
    # Allow any of these security groups or the specified prefixes to access MySQL
    {
      source_security_group_ids = [var.dev_sg, var.uat_sg, var.staging_sg]
      prefix_list_ids = [var.mysql_client_prefix_list_id]
      rules = [
        {
          key         = "mysql"
          type        = "ingress"
          from_port   = 3306
          to_port     = 3306
          protocol    = "tcp"
          description = "Allow MySQL access from trusted security groups"
        }
      ]
    }
  ]

  vpc_id  = module.vpc.vpc_id

  context = module.label.context
}

```



<!-- markdownlint-disable -->
## Makefile Targets
```text
Available targets:

  help                                Help screen
  help/all                            Display help for all targets
  help/short                          This help short screen
  lint                                Lint terraform code

```
<!-- markdownlint-restore -->
<!-- markdownlint-disable -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 0.14.0 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >= 3.0 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | >= 3.0 |

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_this"></a> [this](#module\_this) | cloudposse/label/null | 0.25.0 |

## Resources

| Name | Type |
|------|------|
| [aws_security_group.cbd](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [aws_security_group.default](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [aws_security_group_rule.keyed](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) | resource |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_additional_tag_map"></a> [additional\_tag\_map](#input\_additional\_tag\_map) | Additional key-value pairs to add to each map in `tags_as_list_of_maps`. Not added to `tags` or `id`.<br>This is for some rare cases where resources want additional configuration of tags<br>and therefore take a list of maps with tag key, value, and additional configuration. | `map(string)` | `{}` | no |
| <a name="input_allow_all_egress"></a> [allow\_all\_egress](#input\_allow\_all\_egress) | A convenience that adds to the rules specified elsewhere a rule that allows all egress.<br>If this is false and no egress rules are specified via `rules` or `rule-matrix`, then no egress will be allowed. | `bool` | `false` | no |
| <a name="input_attributes"></a> [attributes](#input\_attributes) | ID element. Additional attributes (e.g. `workers` or `cluster`) to add to `id`,<br>in the order they appear in the list. New attributes are appended to the<br>end of the list. The elements of the list are joined by the `delimiter`<br>and treated as a single ID element. | `list(string)` | `[]` | no |
| <a name="input_context"></a> [context](#input\_context) | Single object for setting entire context at once.<br>See description of individual variables for details.<br>Leave string and numeric variables as `null` to use default value.<br>Individual variable settings (non-null) override settings in context object,<br>except for attributes, tags, and additional\_tag\_map, which are merged. | `any` | <pre>{<br>  "additional_tag_map": {},<br>  "attributes": [],<br>  "delimiter": null,<br>  "descriptor_formats": {},<br>  "enabled": true,<br>  "environment": null,<br>  "id_length_limit": null,<br>  "label_key_case": null,<br>  "label_order": [],<br>  "label_value_case": null,<br>  "labels_as_tags": [<br>    "unset"<br>  ],<br>  "name": null,<br>  "namespace": null,<br>  "regex_replace_chars": null,<br>  "stage": null,<br>  "tags": {},<br>  "tenant": null<br>}</pre> | no |
| <a name="input_create_before_destroy"></a> [create\_before\_destroy](#input\_create\_before\_destroy) | Set `true` to enable terraform `create_before_destroy` behavior on the created security group.<br>We recommend setting this `true` on new security groups, but default it to `false` because `true`<br>will cause existing security groups to be replaced.<br>Note that changing this value will always cause the security group to be replaced. | `bool` | `false` | no |
| <a name="input_delimiter"></a> [delimiter](#input\_delimiter) | Delimiter to be used between ID elements.<br>Defaults to `-` (hyphen). Set to `""` to use no delimiter at all. | `string` | `null` | no |
| <a name="input_descriptor_formats"></a> [descriptor\_formats](#input\_descriptor\_formats) | Describe additional descriptors to be output in the `descriptors` output map.<br>Map of maps. Keys are names of descriptors. Values are maps of the form<br>`{<br>   format = string<br>   labels = list(string)<br>}`<br>(Type is `any` so the map values can later be enhanced to provide additional options.)<br>`format` is a Terraform format string to be passed to the `format()` function.<br>`labels` is a list of labels, in order, to pass to `format()` function.<br>Label values will be normalized before being passed to `format()` so they will be<br>identical to how they appear in `id`.<br>Default is `{}` (`descriptors` output will be empty). | `any` | `{}` | no |
| <a name="input_enabled"></a> [enabled](#input\_enabled) | Set to false to prevent the module from creating any resources | `bool` | `null` | no |
| <a name="input_environment"></a> [environment](#input\_environment) | ID element. Usually used for region e.g. 'uw2', 'us-west-2', OR role 'prod', 'staging', 'dev', 'UAT' | `string` | `null` | no |
| <a name="input_id_length_limit"></a> [id\_length\_limit](#input\_id\_length\_limit) | Limit `id` to this many characters (minimum 6).<br>Set to `0` for unlimited length.<br>Set to `null` for keep the existing setting, which defaults to `0`.<br>Does not affect `id_full`. | `number` | `null` | no |
| <a name="input_inline_rules_enabled"></a> [inline\_rules\_enabled](#input\_inline\_rules\_enabled) | NOT RECOMMENDED. Create rules "inline" instead of as separate `aws_security_group_rule` resources.<br>See [#20046](https://github.com/hashicorp/terraform-provider-aws/issues/20046) for one of several issues with inline rules.<br>See [this post](https://github.com/hashicorp/terraform-provider-aws/pull/9032#issuecomment-639545250) for details on the difference between inline rules and rule resources. | `bool` | `false` | no |
| <a name="input_label_key_case"></a> [label\_key\_case](#input\_label\_key\_case) | Controls the letter case of the `tags` keys (label names) for tags generated by this module.<br>Does not affect keys of tags passed in via the `tags` input.<br>Possible values: `lower`, `title`, `upper`.<br>Default value: `title`. | `string` | `null` | no |
| <a name="input_label_order"></a> [label\_order](#input\_label\_order) | The order in which the labels (ID elements) appear in the `id`.<br>Defaults to ["namespace", "environment", "stage", "name", "attributes"].<br>You can omit any of the 6 labels ("tenant" is the 6th), but at least one must be present. | `list(string)` | `null` | no |
| <a name="input_label_value_case"></a> [label\_value\_case](#input\_label\_value\_case) | Controls the letter case of ID elements (labels) as included in `id`,<br>set as tag values, and output by this module individually.<br>Does not affect values of tags passed in via the `tags` input.<br>Possible values: `lower`, `title`, `upper` and `none` (no transformation).<br>Set this to `title` and set `delimiter` to `""` to yield Pascal Case IDs.<br>Default value: `lower`. | `string` | `null` | no |
| <a name="input_labels_as_tags"></a> [labels\_as\_tags](#input\_labels\_as\_tags) | Set of labels (ID elements) to include as tags in the `tags` output.<br>Default is to include all labels.<br>Tags with empty values will not be included in the `tags` output.<br>Set to `[]` to suppress all generated tags.<br>**Notes:**<br>  The value of the `name` tag, if included, will be the `id`, not the `name`.<br>  Unlike other `null-label` inputs, the initial setting of `labels_as_tags` cannot be<br>  changed in later chained modules. Attempts to change it will be silently ignored. | `set(string)` | <pre>[<br>  "default"<br>]</pre> | no |
| <a name="input_name"></a> [name](#input\_name) | ID element. Usually the component or solution name, e.g. 'app' or 'jenkins'.<br>This is the only ID element not also included as a `tag`.<br>The "name" tag is set to the full `id` string. There is no tag with the value of the `name` input. | `string` | `null` | no |
| <a name="input_namespace"></a> [namespace](#input\_namespace) | ID element. Usually an abbreviation of your organization name, e.g. 'eg' or 'cp', to help ensure generated IDs are globally unique | `string` | `null` | no |
| <a name="input_regex_replace_chars"></a> [regex\_replace\_chars](#input\_regex\_replace\_chars) | Terraform regular expression (regex) string.<br>Characters matching the regex will be removed from the ID elements.<br>If not set, `"/[^a-zA-Z0-9-]/"` is used to remove all characters other than hyphens, letters and digits. | `string` | `null` | no |
| <a name="input_revoke_rules_on_delete"></a> [revoke\_rules\_on\_delete](#input\_revoke\_rules\_on\_delete) | Instruct Terraform to revoke all of the Security Group's attached ingress and egress rules before deleting<br>the security group itself. This is normally not needed. | `bool` | `false` | no |
| <a name="input_rule_matrix"></a> [rule\_matrix](#input\_rule\_matrix) | A convenient way to apply the same set of rules to a set of subjects. See README for details. | `any` | `[]` | no |
| <a name="input_rules"></a> [rules](#input\_rules) | A list of Security Group rule objects. All elements of a list must be exactly the same type;<br>use `rules_map` if you want to supply multiple lists of different types.<br>The keys and values of the Security Group rule objects are fully compatible with the `aws_security_group_rule` resource,<br>except for `security_group_id` which will be ignored, and the optional "key" which, if provided, must be unique<br>and known at "plan" time.<br>To get more info see https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule . | `list(any)` | `[]` | no |
| <a name="input_rules_map"></a> [rules\_map](#input\_rules\_map) | A map-like object of lists of Security Group rule objects. All elements of a list must be exactly the same type,<br>so this input accepts an object with keys (attributes) whose values are lists so you can separate different<br>types into different lists and still pass them into one input. Keys must be known at "plan" time.<br>The keys and values of the Security Group rule objects are fully compatible with the `aws_security_group_rule` resource,<br>except for `security_group_id` which will be ignored, and the optional "key" which, if provided, must be unique<br>and known at "plan" time.<br>To get more info see https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule . | `any` | `{}` | no |
| <a name="input_security_group_create_timeout"></a> [security\_group\_create\_timeout](#input\_security\_group\_create\_timeout) | How long to wait for the security group to be created. | `string` | `"10m"` | no |
| <a name="input_security_group_delete_timeout"></a> [security\_group\_delete\_timeout](#input\_security\_group\_delete\_timeout) | How long to retry on `DependencyViolation` errors during security group deletion from<br>lingering ENIs left by certain AWS services such as Elastic Load Balancing. | `string` | `"15m"` | no |
| <a name="input_security_group_description"></a> [security\_group\_description](#input\_security\_group\_description) | The description to assign to the created Security Group.<br>Warning: Changing the description causes the security group to be replaced. | `string` | `"Managed by Terraform"` | no |
| <a name="input_security_group_name"></a> [security\_group\_name](#input\_security\_group\_name) | The name to assign to the security group. Must be unique within the VPC.<br>If not provided, will be derived from the `null-label.context` passed in.<br>If `create_before_destroy` is true, will be used as a name prefix. | `list(string)` | `[]` | no |
| <a name="input_stage"></a> [stage](#input\_stage) | ID element. Usually used to indicate role, e.g. 'prod', 'staging', 'source', 'build', 'test', 'deploy', 'release' | `string` | `null` | no |
| <a name="input_tags"></a> [tags](#input\_tags) | Additional tags (e.g. `{'BusinessUnit': 'XYZ'}`).<br>Neither the tag keys nor the tag values will be modified by this module. | `map(string)` | `{}` | no |
| <a name="input_target_security_group_id"></a> [target\_security\_group\_id](#input\_target\_security\_group\_id) | The ID of an existing Security Group to which Security Group rules will be assigned.<br>The Security Group's description will not be changed.<br>Not compatible with `inline_rules_enabled` or `revoke_rules_on_delete`.<br>Required if `create_security_group` is `false`, ignored otherwise. | `list(string)` | `[]` | no |
| <a name="input_tenant"></a> [tenant](#input\_tenant) | ID element \_(Rarely used, not included by default)\_. A customer identifier, indicating who this instance of a resource is for | `string` | `null` | no |
| <a name="input_vpc_id"></a> [vpc\_id](#input\_vpc\_id) | The ID of the VPC where the Security Group will be created. | `string` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_arn"></a> [arn](#output\_arn) | The created Security Group ARN (null if using existing security group) |
| <a name="output_id"></a> [id](#output\_id) | The created or target Security Group ID |
| <a name="output_name"></a> [name](#output\_name) | The created Security Group Name (null if using existing security group) |
| <a name="output_rules_terraform_ids"></a> [rules\_terraform\_ids](#output\_rules\_terraform\_ids) | List of Terraform IDs of created `security_group_rule` resources, primarily provided to enable `depends_on` |
<!-- markdownlint-restore -->


