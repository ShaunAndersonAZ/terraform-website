---
layout: "cloud"
page_title: "tfconfig/v2 - Imports - Sentinel - Terraform Cloud"
description: |-
  The tfconfig/v2 import provides access to a Terraform configuration.
---

-> **NOTE:** This is documentation for the next version of the `tfconfig`
Sentinel import, designed specifically for Terraform 0.12. This import requires
Terraform 0.12 or higher, and must currently be loaded by path, using an alias,
example: `import "tfconfig/v2" as tfconfig`.

~> **NOTE:** The Sentinel v2 imports are currently publicly available as a
**technology preview**. They are supported by HashiCorp, but the API is not yet
stable and breaking changes could be made. Watch this documentation for any
changes!

# Import: tfconfig/v2

The `tfconfig/v2` import provides access to a Terraform configuration.

The Terraform configuration is the set of `*.tf` files that are used to
describe the desired infrastructure state. Policies using the `tfconfig`
import can access all aspects of the configuration: providers, resources,
data sources, modules, and variables.

Some use cases for `tfconfig` include:

* **Organizational naming conventions**: requiring that configuration elements
  are named in a way that conforms to some organization-wide standard.
* **Required inputs and outputs**: organizations may require a particular set
  of input variable names across all workspaces or may require a particular
  set of outputs for asset management purposes.
* **Enforcing particular modules**: organizations may provide a number of
  "building block" modules and require that each workspace be built only from
  combinations of these modules.
* **Enforcing particular providers or resources**: an organization may wish to
  require or prevent the use of providers and/or resources so that configuration
  authors cannot use alternative approaches to work around policy
  restrictions.

The data in the `tfconfig/v2` import is sourced from the JSON configuration file
that is generated by the [`terraform show
-json`](/docs/commands/show.html#json-output) command. For more information on
the file format, see the [JSON Output Format](/docs/internals/json-format.html)
page.

## Import Overview

The `tfconfig/v2` import is structured as a series of _collections_, keyed as a
specific format, such as resource address, module address, or a
specifically-formatted provider key.

The collections are:

* [`providers`](#the-providers-collection) - The configuration for all provider
  instances across all modules in the configuration.
* [`resources`](#the-resources-collection) - The configuration of all resources
  across all modules in the configuration.
* [`variables`](#the-variables-collection) - The configuration of all variable
  definitions across all modules in the configuration.
* [`outputs`](#the-outputs-collection) - The configuration of all output
  definitions across all modules in the configuration.
* [`module_calls`](#the-variables-collection) - The configuration of all module
  calls (individual [`module`](/docs/configuration/modules.html) blocks) across
  all modules in the configuration.

These collections are specifically designed to be used with the
[`filter`](https://docs.hashicorp.com/sentinel/language/collection-operations/#filter-expression)
quantifier expression in Sentinel, so that one can collect a list of resources
to perform policy checks on without having to write complex module or
configuration traversal. As an example, the following code will return all
`aws_instance` resource types within the configuration, regardless of what
module they are in:

```
all_aws_instances = filter tfconfig.resources as _, r {
	r.mode is "managed" and
		r.type is "aws_instance"
}
```

You can add specific attributes to the filter to narrow the search, such as the
module address. The following code would return resources in a module named
`foo` only:

```
all_aws_instances = filter tfconfig.resources as _, r {
	r.module_address is "module.foo" and
		r.mode is "managed" and
		r.type is "aws_instance"
}
```

### "Approximate" Addresses Explained

Throughout this documentation, the term _approximate address_ will be used. This
term refers an address that is constructed by `tfconfig/v2` to, as closely as
possible, match the absolute addresses found in the
[`tfplan/v2`](./tfplan-v2.html) and [`tfstate/v2`](./tfstate-v2.html) imports.

These addresses may be equal to the addresses found in those imports, or, in the
event where resources are scaled with
[`count`](/docs/configuration/resources.html#count-multiple-resource-instances-by-count)
and
[`for_each`](/docs/configuration/resources.html#for_each-multiple-resource-instances-defined-by-a-map-or-set-of-strings),
they will be the same as if those resources lacked indexes.

As an example, consider a resource named `null_resource.foo` with a count of `2`
located in a module named `bar`. While there will possibly be entries in the
other imports for `module.bar.null_resource.foo[0]` and
`module.bar.null_resource.foo[1]`, in `tfconfig/v2`, there will only be a
`module.bar.null_resource.foo`. This is because configuration actually _defines_
this scaling, whereas _expansion_ actually happens when the resource graph is
built, which happens as a natural part of the refresh and planning process.

To summarize, generally, the approximate address is an absolute resource
address without indexes.

For now, approximate addresses generally will only see a removal of the resource
index, but in the future, this could include modules. This will be the case once
`count` and `for_each` are implemented for modules, see [Other
Meta-arguments](/docs/configuration/modules.html#other-meta-arguments) in the
Terraform module documentation for more details.

The `approximate_address` helper function, found in this import, can assist in
removing the indexes from addresses found in the `tfplan/v2` and `tfstate/v2`
imports so that data from those imports can be used to reference data in this
one.

## The `approximate_address` Function

The `approximate_address` helper function can be used to produce approximate
addresses from complete addresses found in [`tfplan/v2`](./tfplan-v2.html) and
[`tfstate/v2`](./tfstate-v2.html), by removing the indexes from each resource.

This can be used to help facilitate cross-import lookups for data between plan,
state, and config.

```
import "tfconfig/v2" as tfconfig
import "tfplan/v2" as tfplan

main = rule {
	all filter tfplan.resource_changes as _, rc {
		rc.mode is "managed" and
			rc.type is "aws_instance"
	} as _, rc {
		tfconfig.resources[tfconfig.approximate_address(rc.address)].expressions.ami.constant_value is "ami-abcdefgh012345"
	}
}
```

## Expression Representations

Most collections in this import will have one of two kinds of _expression
representations_. This is a verbose format for expressing a (parsed)
configuration value independent of the configuration source code, which is not
100% available to a policy check in Terraform Cloud.

There are two major parts to an expression representation:

* Any _strictly constant value_ is expressed as an expression with a
  `constant_value` field.
* Any expression that requires some degree of evaluation to generate the final
  value - even if that value is known at plan time - is not expressed in
  configuration. Instead, any particular references that are made are added to
  the `references` field. More details on this field can be found in the
  [expression
  representation](/docs/internals/json-format.html#expression-representation)
  section of the JSON output format documentation.

So to, for example, determine if an output is working off of a particular
resource value, one could do:

```
import "tfconfig/v2" as tfconfig

main = rule {
	tfconfig.outputs["instance_id"].expressions.references is ["aws_instance.foo"]
}
```

-> **NOTE:** The expression representation currently does not account for
complex interpolations or other expressions that combine constants with other
expression data. An example would be something like `"foo${var.bar}"` or `"foo"
+ var.bar`.  In this situation, the partially constant data would be lost. We
will be working to resolve this within future versions of the `tfconfig/v2`
import.

### Block Expression Representation

Expanding on the above, a multi-value expression representation (such as the
kind found in a [`resources`](#the-resources-collection) collection element) is
similar, but the root value is a keyed map of expression representations. This
is repeated until a "scalar" expression value is encountered, ie: a field that
is not a block in the resource's schema.

As an example, one can validate expressions in an
[`aws_instance`](/docs/providers/aws/r/instance.html) resource using the
following:

```
import "tfconfig/v2" as tfconfig

main = rule {
	tfconfig.resources["aws_instance.foo"].expressions.ami.constant_value is "ami-abcdefgh012345"
}
```

Note that _nested blocks_, sometimes known as _sub-resources_, will be nested in
configuration as as list of blocks (reflecting their ultimate nature as a list
of objects). An example would be the `aws_instance` resource's
[`ebs_block_device`](/docs/providers/aws/r/instance.html#block-devices) block:

```
import "tfconfig/v2" as tfconfig

main = rule {
	tfconfig.resources["aws_instance.foo"].expressions.ebs_block_device[0].volume_size < 10
}
```

## The `providers` Collection

The `providers` collection is a collection representing the configurations of
all provider instances across all modules in the configuration.

This collection is indexed by an opaque key. This is currently
`module_address:provider.alias`, the same value as found in the
`provider_config_key` field. `module_address` and the colon delimiter are
omitted for the root module.

The `provider_config_key` field is also found in the `resources` collection and
can be used to locate a provider that belongs to a configured resource.

The fields in this collection are as follows:

* `provider_config_key` - The opaque configuration key, used as the index key.
* `name` - The name of the provider, ie: `aws`.
* `alias` - The alias of the provider, ie: `east`. Empty for a default provider.
* `module_address` - The address of the module this provider appears in.
* `expressions` - A [block expression
  representation](#block-expression-representation) with provider configuration
  values.
* `version_constraint` - The defined version constraint for this provider.

## The `resources` Collection

The `resources` collection is a collection representing all of the resources
found in all modules in the configuration.

This collection is indexed by the approximate resource address. For more
information, see ["Approximate" Addresses
Explained](#quot-approximate-quot-addresses-explained).
              

The fields in this collection are as follows:

* `address` - The approximate resource address. This is the index of the
  collection.
* `module_address` - The module address that this resource was found in.
* `mode` - The resource mode, either `managed` (resources) or `data` (data
  sources).
* `type` - The type of resource, ie: `null_resource` in `null_resource.foo`.
* `name` - The name of the resource, ie: `foo` in `null_resource.foo`.
* `provider_config_key` - The opaque configuration key that serves as the index
  of the [`providers`](#the-providers-collection) collection.
* `provisioners` - The ordered list of provisioners for this resource. The
  syntax of the provisioners matches those found in the
  [`provisioners`](#the-provisioners-collection) collection, but is a list
  indexed by the order the provisioners show up in the resource.
* `expressions` - The [block expression
  representation](#block-expression-representation) of the configuration values
  found in the resource.
* `count_expression` - The [expression data](#expression-representations) for
  the `count` value in the resource.
* `for_each_expression` - The [expression data](#expression-representations) for
  the `for_each` value in the resource.
* `depends_on` - The contents of the `depends_on` config directive, which
  declares explicit dependencies for this resource.

## The `provisioners` Collection

The `provisioners` collection is a collection of all of the provisioners found
across all resources in the configuration.

While normally bound to a resource in an ordered fashion, this collection allows
for the filtering of provisioners within a single expression.

This collection is indexed with a key following the format
`resource_address:index`, with each field matching their respective field in the
particular element below:

* `resource_address`: The [approximate resource
  address](#approximate-addresses-explained) the
  [`resources`](#the-resources-collection) collection.
* `type`: The provisioner type, ie: `local_exec`.
* `index`: The provisioner index as it shows up in the resource provisioner
  order.
* `expressions`: The [block expression
  representation](#block-expression-representation) of the configuration values
  in the provisioner.

## The `variables` Collection

The `variables` collection is a collection of all variables across all modules
in the configuration.

Note that this tracks variable definitions, not values. See the [`tfplan/v2`
`variables` collection](./tfplan-v2.html#the-variables-collection) for variable
values set within a plan.

This collection is indexed by the key format `module_address:name`, with each
field matching their respective name below. `module_address` and the colon
delimiter are omitted for the root module.

* `module_address` - The address of the module the variable was found in.
* `name` - The name of the variable.
* `default` - The defined default value of the variable.
* `description` - The description of the variable.

## The `outputs` Collection

The `outputs` collection is a collection of all outputs across all modules in
the configuration.

Note that this tracks variable definitions, not values. See the [`tfstate/v2`
`outputs` collection](./tfstate-v2.html#the-outputs-collection) for the final
values of outputs set within a state. The [`tfplan/v2` `output_changes`
collection](./tfplan-v2.html#the-output_changes-collection) also contains a more
complex collection of planned output changes.

This collection is indexed by the key format `module_address:name`, with each
field matching their respective name below. `module_address` and the colon
delimiter are omitted for the root module.

* `module_address` - The address of the module the output was found in.
* `sensitive` - Indicates whether or not the output was marked as
  [`sensitive`](/docs/configuration/outputs.html#sensitive-suppressing-values-in-cli-output).
* `expression` - An [expression representation](#expression-representations) for the output.
* `description` - The description of the output.
* `depends_on`: A list of resource names that the output depends on. These are
  the hard-defined output dependencies as defined in the
  [`depends_on`](/docs/configuration/outputs.html#depends_on-explicit-output-dependencies)
  field in an output declaration, not the dependencies that get derived from
  natural evaluation of the output expression (these can be found in the
  `references` field of the expression representation).

## The `module_calls` Collection

The `module_calls` collection is a collection of all module declarations at all
levels within the configuration.

Note that this is the
[`module`](/docs/configuration/modules.html#calling-a-child-module) stanza in
any particular configuration, and not the module itself. Hence, a declaration
for `module.foo` would actually be declared in the root module, which would be
represented by a blank field in `module_address`.

This collection is indexed by the key format `module_address:name`, with each
field matching their respective name below. `module_address` and the colon
delimiter are omitted for the root module.

* `module_address` - The address of the module the declaration was found in.
* `name` - The name of the module.
* `source` - The contents of the `source` field.
* `expressions` - A [block expression
  representation](#block-expression-representation) for all parameter values
  sent to the module.
* `count_expression` - An [expression
  representation](#expression-representations) for the `count` field (not
  currently in use).
* `for_each_expression` - An [expression
  representation](#expression-representations) for the `for_each` field (not
  currently in use).
* `version_constraint` - The string value found in the `version` field of the
  module declaration.

-> **NOTE:** As `count` and `for_each` are currently not implemented in modules,
the `count_expression` and `for_each_expression` fields will always be blank.