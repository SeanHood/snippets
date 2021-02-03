---
title: "Terraform and OPA/Conftest/Rego, what a pain in the arse"
date: 2021-02-03T22:52:13Z
draft: false
---
# Terraform and OPA/Conftest/Rego, what a pain in the arse it is

So this has taken me 24+ hours to figure out. All docs, references to using OPA
with Terraform are about testing the plan. I want some static analysis on raw
`.tf` files. An OPA based linter if you will.

First thing I ran into was iterating over an object, then being able to use
it's key. In Python I'd do: `for k,v in dict`. In Rego, we do hash[key], then
just go and use `key` wherever.  ¯\\\_(ツ)\_/¯ Maybe this will make more sense
the more I use Rego.

Second thing, I think I was being caught out by was: "[Universal Quantification](https://www.openpolicyagent.org/docs/latest/policy-language/#universal-quantification-for-all)".
So in my policy where I have `not key == computed`, I had used `key != computed`
which, maybe this will make sense in the future, but right now. Mind blown.

Anyway, the thing you've been waiting for. Some examples.

main.tf
```terraform
module "foo_app_uk_prod" {
  source = "./modules/foo"

  module  = "foo"
  project = "app"
  region  = "uk"
  env     = "stage"
}

module "foo_database_uk_prod" {
  source = "./modules/foo"

  module  = "foo"
  project = "database"
  region  = "uk"
  env     = "prod"
}
```

policy/module.md
```go
package main

deny[msg] {
    m := input.module[key]

    # Based on the attributes, we compute what the expected module should be named
    computed_module_name := sprintf("%s_%s_%s_%s", [m.module, m.project, m.region, m.env])

    not key == computed_module_name

    msg := sprintf("Module name: %v, does not match computed configuration: %v", [key, computed_module_name])
}
```

So if we take our `main.tf` and `module.rego` and run them like so with the handy
tool called `conftest`. 

```
$ conftest test main.tf -p policy/module.rego
FAIL - main.tf - main - Module name: foo_app_uk_prod, does not match computed configuration: foo_app_uk_stage

1 test, 0 passed, 0 warnings, 1 failure, 0 exceptions
```

We can see that we have one failure. This policy/linter
is trying to ensure the module name matches the configuration passed to it.

Looks like we've messed up when copy and pasting. Our app config, let's update
line 7 to say `prod` and run `conftest` again:

```
$ conftest test main.tf -p policy/module.rego

1 test, 1 passed, 0 warnings, 0 failures, 0 exceptions
```

Much better.



