---
title: "Terraform: Move resource from one state to another (Terraform Cloud)"
date: 2021-02-10T15:12:03Z
draft: true
---

So I wanted to move a database cluster from one Terraform setup to another. I could have used `terraform import` but this can be a pain.

Instead I used `terraform state mv`, this post documents some of the pitfalls.

Reading the docs it explains quite simply how to "export" a resource to be imported elsewhere.

```
terraform state mv -state-out=database.json 'mongodbatlas_cluster.mongodb_cluster' 'mongodbatlas_cluster.mongodb_cluster'
```

The problem came when using Terraform Cloud, the state file isn't local at all. This was also a fresh workspace.

First things first, you may need to run the usual `terraform init` to create that connection to Terraform Cloud

After this we fetch the current (empty) state:

```
terraform state pull > mystate.json
```

Run our state mv again to copy the resource into our new state file:
```
terraform state mv -state=database.json -state-out=mystate.json 'mongodbatlas_cluster.mongodb_cluster' 'mongodbatlas_cluster.mongodb_cluster'
```

Then we push the state back up to Terraform Cloud:
```
terraform state push mystate.json
```


Now, since we had an empty state, perhaps we could have just pushed our `database.json` straight up, however this method will work for pre-existing workspaces.

If you had your state locally, usually in a `terraform.tfstate` file in the root directory, things would be a lot more clear.

