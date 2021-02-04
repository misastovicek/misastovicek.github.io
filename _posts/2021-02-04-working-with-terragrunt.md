---
layout: posts
title: Working with Terragrunt
author: Michal Stovicek
---
## Terragrunt in action

I bet most of the people who work with any public or even private cloud knows [Hashicorp's Terraform](https://terraform.io). *Terraform* is an awesome cloud agnostic tool which can be used to deploy Infrastructure as Code (IaC in short). *Terraform* is using a sort of plugins called **providers** for each of the cloud providers. Interesting thing is that not only cloud providers such as Microsoft Azure, Amazon Web Services or Google Cloud are supported, but there are providers for Kubernetes, Helm or even Datadog as well! You can check existing providers at [Terraform Registry](https://registry.terraform.io/browse/providers).

What is usually chalenging though, is how to keep the infrastructure code DRY, reusable and easily maintainable. There are couple of ways where one of them is to use *Terraform* workspaces which alows to use the same code like this for multiple environemnts (for example dev, uat, prod):

```hcl
resource "azurerm_resource_group" "this" {
    name = "rg-${terraform.workspace}"
    location = "westeurope"
}
```

If I run `terraform workspace ls`, I may get output similar to this:

```bash
$ terraform workspace list
  default
* dev
  prod
  uat
```

So from the example above, you see that there will be different resource group created for each of the workspaces. One would be `rg-dev`, second `rg-uat` and so on. This will also make sure that the resources will be saved into separated statefiles. But what happens when you want to have `*.tfvars` file for each of the environments? In such case you would have to write maybe shell script or any other script which will call *terraform* with the variables you need otherwise you would need to provide those manually which won't scale well. Also how would you handle backend setup if the environments are for example in different accounts and you want to store the statefiles within those environments? Let's say I use Azure Blob Storage backend or AWS S3 backend for each of the environments.

What solves the above questions is a tool called [**Terragrunt**](https://terragrunt.gruntwork.io/). It helps to keep the infrastructure code DRY, your provider configurations DRY and more. *Terragrunt* is a wrapper around the *Terraform* so it just calls the *terraform* with proper command line options and environment variables. What is interesting though is that it can generate backend configuration so now you can have a configuration repo with all the `*.tfvars` files stored in the right directory and use *Terragrunt* to generate the backend configuration based on it's configuration files stored along with the `*.tfvars` files. *Terragrunt* is also capable of passing variables to the *terraform* directly via it's configuration block called **inputs** and what's more interesting, it can load variables to such block from YAML files. This is especially usefull with it's support of [Mozilla's SOPS](https://github.com/mozilla/sops) so now you can have your secrets stored directly in Git repository since they are encrypted by SOPS which supports encryption by PGP, Hashicorp Vault, Azure Key Vault, AWS KMS and GCP KMS. Sounds good? So read on! 

With *Terragrunt*, I prefer to have each infrastructure module in a separate repository (i.e. networking, Kubernetes, VMs, etc.) so that they can be versioned easily. Of course, they can then source more modules. The version can then be passed to the *Terragrunt* via it's configuration file `terragrunt.hcl`. This is what *Terraform* itself supports - when you want to use a version of a module in the *Terraform* code, you can say that when sourcing the module. However, with *Terragrunt* you are not sourcing a module into a module. You are pulling a git repository with the module and *Terragrunt* will possibly template the backend configuration in the pulled directory.

I can definitely talk about *Terraform* and *Terragrunt* more but instead, I sugest you to have a look at [terragrunt.gruntwork.io](https://terragrunt.gruntwork.io/) and I will show you some examples and describe how I use *Terragrunt*.

Let's consider a project which is composed of **Azure Kubernetes Service**, **Azure Database for MySQL** and of course **Virtual Network** and let's say **Azure Key Vault** for storing secrets and **Log Analytics Workspace** to store the monitoring data and logs. The project has to have three stages - Development, User Acceptance Test and Production. What I would do is create one *Terraform* module for *common-services* which will create Log Analytics Workspace and Azure Key Vault, one for Virtuel Network with subnets, routing tables and sercurity groups applied to the subnets, one for Azure Kubernetes Service which may also deploy monitoring agent and maybe install an Ingress controller and the last module for MySQL.

With *Terragrunt*, I would have a configuration repository for this specific project and the directory structure may look like this:

```bash
$ mkdir -p terragrunt/{dev,uat,prod}/westeurope/{common-services,networking,kubernetes,mysql}
$ tree
.
└── terragrunt
    ├── dev
    │   └── westeurope
    │       ├── common-services
    │       ├── kubernetes
    │       ├── mysql
    │       └── networking
    ├── prod
    │   └── westeurope
    │       ├── common-services
    │       ├── kubernetes
    │       ├── mysql
    │       └── networking
    └── uat
        └── westeurope
            ├── common-services
            ├── kubernetes
            ├── mysql
            └── networking
```

Alright, so we have *terragrunt* directory in our configuration repo (there may also be directories like kube-manifests, helm, ansible and so on) and that directory contains three environments. Each environment can be deployed to one or more regions but for now let's keep it simple with just one. There are also directories for each of the infrastructure components we want in the specific environment and region.

So, now let's start with *Terragrunt* configuration. First of all, let's place one `terragrunt.hcl` into the root of the *terragrunt* directory. Content of the file may look like this:

```hcl
remote_state {
  backend = "azurerm"
  generate = {
    path = "backend.tf"
    if_exists = "overwrite"
  }
  config = {
    subscription_id      = "00000000-0000-0000-0000-000000000000"
    resource_group_name  = "rg-config"
    storage_account_name = "terragruncfg"
    container_name       = "terraform-statefiles"
    key                  = "${path_relative_to_include()}/terraform.tfstate"
  }
}

inputs = {
  project_name = "myProject"
}
```

The code above is pretty much self-explanatory - use *azurerm* backend for *Terraform* and use the configuration from the `config` code block. The `generate` code block does one very useful thing which is generating of a `backend.tf` file in the *terraform* code directory and overwrite the file if it already exists. Last thing here is `intputs` where we can define project scope variables used in the *terraform* modules. *Terragrunt* will create environment variables from all the inputs in the format of `TF_VAR_project_name` before the *terraform* command is called so these will be available for all the environments in all of the locations. You may also wander what `path_relative_to_include()` function does. It is always set to the relative path from where you ran the *Terragrunt* (it's actually a bit more complicated than that so I suggest to read the docs before using it).

Right now we have base *Terragrunt* configuration file which will be shared by all the others. Let's continue with couple more files. We want to have `env.hcl` in each of the environment directories (dev,uat,prod) and `location.hcl` in each of the location directories (three times westeurope in this case):

```bash
cd terragrunt
for e in `ls -1`; do cat >${e}/env.hcl<<-EOF
locals {
  environment = "$e"
}
EOF
cat >${e}/westeurope/location.hcl<<-EOF
locals {
  location = "westeurope"
}
EOF
done
```

The above command creates the `hcl` files with a content defining the environment and the location for which can then be used by *terragrunt*. Now we can go to the specific components directories and create another `terragrunt.hcl` file in them. I am going to create all of them with some base content and update them later based on the component:

```bash
for e in `ls -1`; do
  for c in `ls -1 $e/westeurope/ | grep -v ".hcl"`; do
    cat >$e/westeurope/$c/terragrunt.hcl<<-'EOF'
include {
  # Find the first 'terragrunt.hcl' file in the parent folders
  path = find_in_parent_folders()
}

locals {
  env = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  location = read_terragrunt_config(find_in_parent_folders("location.hcl"))
  common_vars = yamldecode(file("${get_parent_terragrunt_dir()}/common_vars.yml"))
}

terraform {
  # Source of the Terraform module to be deployed
  source = ""
}

inputs = {
  environment = local.env.locals.environment
  location = local.location.locals.location
  
  tags = merge(
    local.common_vars.tags,
    {"Source" = "*Terragrunt*"}
  )
}
EOF
  done
done
```

Well, now we have the same `terragrunt.hcl` file in each of the components directories but before we start to update them, I would like to point out few things which you may see in the file created by the command above.

First of all, you may see that there is a code block called `include` and uses `find_in_parent_folders()` function. This will search for the first match of `terragrunt.hcl` file in the parent directories and includes them to this child file. This is particulary useful for the *Terraform* backend configuration and common variables in the `inputs` block of the parent file. The input blocks are merged during the *Terragrunt* run.

Another thing is the `locals` block which you may recognize from the *Terraform* itself. Since *Terragrunt* uses the same GO libraries as *Terragrunt* does, it works pretty much the same way. What's interesting about this block is that it is not loaded from the parent *Terragrunt* config files. You have to read them explicitly using `read_terragrunt_config()` function in the "local locals" (weird expression, isn't it? (-: ) and then you can use them however you want (for example in the inputs block). You may also notice that I am using `yamldecode()` function which serves as a YAML files reader and would be especially useful in combination with SOPS. An example might be `yamldecode(sops_decrypt_file(find_in_parent_folders("secrets.yaml")))` which would search for `secrets.yaml` file, decrypts it via SOPS and loads them using the `yamldecode()` function.

Alright, so let's update the files for each of the components. What I usually do is that I use `*.tfvars` files for variables specified for the components so I can re-use the already existing files from days before the *Terragrunt*. Here comes the `terraform` code block from the code above. We can tell in that block that we want to use a `*.tfvars` file with all *Terraform* commands which support it. So, I am going to update the `terragrunt.hcl` file in kubernetes component. Here is how the block will look like after my updates:

```hcl
terraform {
  # Source of the Terraform module to be deployed
  source = "git::https://mygitlab.example.com/terraform-modules/aks.git?ref=v1.0"

  extra_arguments "var_files" {
    commands = get_terraform_commands_that_need_vars()
    required_var_files = [
      "${get_terragrunt_dir()}/aks.tfvars"
    ]
  }
}
```

From the code snippet above you can see that I am using Azure Kubernetes Service module from a Git repository and I am using it's version 1.0. I am also including file `aks.tfvars` from the current directory but only for commands which neeed variables. It means that the file will be passed as `-tfvars aks.tfvars` parameter to the *terraform* command.

I would do the same as above for all the components and probably copy that into other two environments so that I have the same files and variables everywhere. Then it's just about changes in variables files. So, I could have the following in my `aks.tfvars` - dev environment:

```hcl
resource_group_name = "rg-aks-dev"
default_nodes_size  = "Standard_B4ms"
```

I would now copy the `terragrunt.hcl` and `aks.tfvars` files to the other environments and update the files:

```bash
cp dev/westeurope/kubernetes/* uat/westeurope/kubernetes/
cp dev/westeurope/kubernetes/* prod/westeurope/kubernetes/
```

Here is the content for UAT and Prod `aks.tfvars` respectively:

```hcl
resource_group_name = "rg-aks-uat"
default_nodes_size  = "Standard_D4s_v3"
```

```hcl
resource_group_name = "rg-aks-prod"
default_nodes_size  = "Standard_D4s_v3"
```

Alright, so if I do the same for all the files and environments, I'll end up with this structure:

```bash
.
└── terragrunt
    ├── dev
    │   ├── westeurope
    │   │   ├── common-services
    │   │   │   ├── common-services.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── kubernetes
    │   │   │   ├── aks.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── mysql
    │   │   │   ├── mysql.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── networking
    │   │   │   ├── networking.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   └── location.hcl
    │   └── env.hcl
    ├── prod
    │   ├── westeurope
    │   │   ├── common-services
    │   │   │   ├── common-services.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── kubernetes
    │   │   │   ├── aks.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── mysql
    │   │   │   ├── mysql.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── networking
    │   │   │   ├── networking.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   └── location.hcl
    │   └── env.hcl
    ├── uat
    │   ├── westeurope
    │   │   ├── common-services
    │   │   │   ├── common-services.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── kubernetes
    │   │   │   ├── aks.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── mysql
    │   │   │   ├── mysql.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   ├── networking
    │   │   │   ├── networking.tfvars
    │   │   │   └── terragrunt.hcl
    │   │   └── location.hcl
    │   └── env.hcl
    └── terragrunt.hcl
```

So, now we are ready to go! We have our environments ready for our project and *Terraform* modules can be used by any other project! How cool is that?! And there is even more you can do with *Terragrunt*. You can for example create dependencies between the components and run `terragrunt run-all apply` from the root directory and terragrunt will go into each of the children and execute `terragrunt apply` in each of them so you can deploy the whole infrastructure with a single command! There is a catch though. You may be wondering how could you deploy Kubernetes before the Virtual network exists, or how can you deploy monitoring agent which depends on Log Analytics Workspace before the workspace actually exists. *Terragrunt* has a solution for that. You just need to specify `dependencies` code block in your `terragrunt.hcl` file. Let's consider the *kubernetes* component. It depends on *networking* (you know, for networking) as well as *common-services* (for monitoring and possibly secrets from Key Vault). You can add the following block to the `terragrunt.hcl`:

```hcl
dependencies {
  paths = ["../common-services", "../networking"]
}
```

What's even more interesting, you may also use outputs from those dependencies and use them in the `inputs` block of the dependent component! Just define `dependency` block and use the outputs:

```hcl
dependency "networking" {
  config_path = "../networking"
}

inputs = {
  aks_subnet_id = dependency.networking.outputs.aks_subnet_id
  
  environment = local.env.locals.environment
  location = local.location.locals.location
  
  tags = merge(
    local.common_vars.tags,
    {"Source" = "*Terragrunt*"}
  )
}
```

If you are like me then you found the *Terragrunt* to be an AWESOME tool. What I suggest to implement though is some kind of CI/CD pipeline to apply your infrastructure as it still might be error prone when applying manually. For example dev, uat and prod environments live in different Azure Subscription and you may not notice that you are currently using prod subscription while you are running *Terragrunt* from dev environment. Then you will be surprised that it wants to recreate the whole infrastructure.

So, I hope you will give a try to *Terragrunt* and I promiss you won't regret!
