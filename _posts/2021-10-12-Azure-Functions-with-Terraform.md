---
layout: posts
title: Azure Functions with Terraform
author: Michal Stovicek
---

It has been some time since I wrote a blog post. A lot of work and lazyness, you know... (-:
Today I want to show a way how to deploy Azure Functions with Terraform. You know, you can use VS Code to deploy the functions, but if you need to upload the same code to many functions, it may get very frustrating and you may eventually end up with different versions accross your functions. You can also use a script, but why would you do that if you can use Terraform, right?

The function I am going to use is written in *Python 3.8* and it is intended to start and stop *Azure Kubernetes Service* during after-hours. This may be usefull to save some $, right?

So, first things first. Let's create directory structure for the Terraform module as well as for the Azure function and open VS Code in the directory which will hold the Function's code:

```bash
mkdir -p aks-stop-start-function/files/aks-stop-starter
cd aks-stop-start-function/files/aks-stop-starter
code .
```

For the function initialization, development and also for the upload of the code to the Azure Function service we need some prerequisites installed:

* [Terraform](https://www.terraform.io/downloads.html)
* [Azure Functions Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v3%2Clinux%2Ccsharp%2Cportal%2Cbash%2Ckeda#install-the-azure-functions-core-tools)
* [Azure Functions plugin for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions)

>**Note:** The Azure Functions plugin for VS Code is optional, but I am going to use it to initialize the function. You can use Azure Functions Core Tools as well.

In the VS Code, press `Ctrl + Shift + P` to open the commands menu and select `Azure Functions: Create New Project...`. You will be asked couple of questions where you select the directory, function programming language (Python in this case) and then you'll be also asked to select Python interpreter to use for the Python Virtual Environment which is created for you. Last three questions are about how the function should be triggered (TimerTrigger in this case), function name (I've used `aks_stop_starter`) and the last one is the cron format for the Time Trigger which can be left default.

>**Note:** I use `*/5 * * * * *` for development as I want to test the function locally before it gets deployed to Azure and don't really want to wait 5 minutes between runs which is the default.

Perfect, so now I have the project initiated with some sample code. Because the code is going to interact with *Azure Kubernetes Service* I need to include two more Python modules to the project - one for the interaction with the container services and second for login to Azure. The function is going to use *System Assigned Identity* for that.

```bash
cat >>./requirements.txt<<-EOF
azure-mgmt-containerservice
azure-identity
EOF
```

Azure Functions use `local.settings.json` file for the local function runs. To make it working, I need a connection string to an Azure Storage Account where the Azure Function puts some files. The connection string needs to be put as a value to the `AzureWebJobsStorage` key in the `local.settings.json` file. The function will not run locally if you don't set that up.

That's all what needs to be set up. Alright, so now I'll replace the content of the `./aks_stop_starter/__init__.py` file with this one:

```python
import logging
import os
import azure.functions as func
from azure.identity import DefaultAzureCredential
from aks_stop_starter.start_stop_class import AksStartStopper

SUBSCRIPTION_ID = os.environ.get(key="AZURE_SUBSCRIPTION_ID")


def main(timer: func.TimerRequest) -> None:
    if timer.past_due:
        logging.info("Function run is past due!")
    if not SUBSCRIPTION_ID:
        logging.error("AZURE_SUBSCRIPTION_ID is not set!")
        return 1

    logger = logging.getLogger("azure")
    logger.setLevel(logging.WARNING)
    creds = DefaultAzureCredential()

    aks_clusters_stop_start = AksStartStopper(
        creds, SUBSCRIPTION_ID)

    logging.info("Running start/stop")
    aks_clusters_stop_start.run()

    logging.info("Finished")
```

>**Note:** The `timer` parameter is mandatory for the `main` function. You can just ignore it, but I've used the condition from the sample function.

See that I've set logger severity for `azure` to *WARNING* so that there is not as many useless information from the Azure SDK.

Also notice that the code imports a class `AksStartStopper` from the `aks_stop_starter/start_stop_class.py` file which does not exist yet. I'll create that file with the following content (and yes, I know, the code is a mess (-: ).

```python
import datetime
import logging
from typing import Optional
import azure.mgmt.containerservice as aks
from azure.core.credentials import TokenCredential


class AksStartStopper():
    states = {"stop": "Stopped", "start": "Running"}
    state_actions = {"stop": "Stopping", "start": "Starting"}
    action_tags = {"stop": "AutoStop", "start": "AutoStart"}

    def __init__(self, credentials: TokenCredential, subscription_id: str) -> None:
        self.subscription_id = subscription_id
        self.aks_client = aks.ContainerServiceClient(
            credentials, self.subscription_id)
        self.utc_current_datetime = datetime.datetime.utcnow().replace(
            tzinfo=datetime.timezone.utc)
        self.business_days = [d for d in range(0, 5)]

    def _get_aks_clusters(self) -> list:
        return self.aks_client.managed_clusters.list()

    def _check_aks_state(self, cluster: object, cluster_resource_group_name: str, action: str) -> bool:
        if cluster.power_state.code == self.states[action]:
            logging.info(
                f"Cluster {cluster.name} - rg {cluster_resource_group_name} - already {self.states[action]}.")
            return True
        return False

    def _check_values(self, values: list) -> bool:
        if not values or len(values) != 3:
            return False

        return True

    def _set_datetime(self, values: list) -> Optional(datetime.datetime):
        if self._check_values(values):
            return datetime.datetime.utcnow().replace(
                hour=int(values[1]),
                minute=int(values[2]),
                tzinfo=datetime.timezone.utc
            )
        return None

    def _aks_clusters_action(self, action: str):
        for cluster in self._get_aks_clusters():
            cluster_resource_group_name = cluster.id.split('/')[4]
            start_values = self._get_aks_cluster_action_tags(cluster, "start")
            stop_values = self._get_aks_cluster_action_tags(cluster, "stop")
            action_values = self._get_aks_cluster_action_tags(cluster, action)
            current_day = datetime.datetime.utcnow().weekday()

            utc_start_datetime = self._set_datetime(start_values)
            utc_stop_datetime = self._set_datetime(stop_values)
            utc_action_datetime = self._set_datetime(action_values)

            if not utc_action_datetime:
                continue

            if action_values[0] == "bd" and current_day not in self.business_days and self.states["stop"]:
                if self._check_aks_state(cluster, cluster_resource_group_name, "stop"):
                    continue

                self._perform_aks_action(
                    action, cluster, cluster_resource_group_name)
                continue

            perform_action = False
            if ((action_values[0] == "w"
                    or (action_values[0] == "bd" and current_day in self.business_days))
                    and utc_action_datetime < self.utc_current_datetime):
                if (action == "start"
                        and utc_start_datetime
                        and utc_stop_datetime
                        and utc_start_datetime < utc_stop_datetime
                        and self.utc_current_datetime < utc_stop_datetime):
                    perform_action = True
                elif (action == "start"
                        and not utc_stop_datetime):
                    perform_action = True
                elif action == "stop":
                    perform_action = True
                else:
                    continue

            if perform_action:
                if self._check_aks_state(cluster, cluster_resource_group_name, action):
                    continue
                self._perform_aks_action(
                    action, cluster, cluster_resource_group_name)
            else:
                logging.info(
                    f"{action.capitalize()} action not needed for cluster {cluster.name} - rg {cluster_resource_group_name}")

    def _perform_aks_action(self, action: str, cluster: object, cluster_resource_group_name: str):
        logging.info(
            f"{self.state_actions[action]} cluster {cluster.name} - rg {cluster_resource_group_name}.")

        if action == "stop":
            self.aks_client.managed_clusters.begin_stop(
                resource_group_name=cluster_resource_group_name,
                resource_name=cluster.name)
        else:
            self.aks_client.managed_clusters.begin_start(
                resource_group_name=cluster_resource_group_name,
                resource_name=cluster.name)

    def _get_aks_cluster_action_tags(self, cluster: dict, action: str) -> list:
        cluster_tags: dict = cluster.tags
        tag = cluster_tags.get(self.action_tags[action])

        if isinstance(tag, str):
            return tag.split('-')
        else:
            return []

    def run(self):
        '''
        Stops AKS clusters based on tags.

        Tag structure parsed by the class:
        [bd|w]-hours-minutes

        If 'bd' is used the AKS clusters will be stopped/started at hours and minutes during
        business days and will stay turned off during the weekend.
        '''
        self._aks_clusters_action(action="start")
        self._aks_clusters_action(action="stop")
```

Now that the code is ready, you may want to test it. Just press `F5` and it will run. You may want to add some break points in the VS Code so that the the code is stopped before it actually stops/starts the clusters or maybe update the `_perform_aks_action` function to just log what it is going to do without actually doing it.

So now the function is ready and so I can go ahead and write some Terraform code (by the way the official Terraform plugin for VS Code works pretty well as it suggests references to variables, locals, resources, data and even modules outputs now!). I like to split the code into more files where one is for *providers* configuration, one for *variables and locals*, one for *data sources* and one or more for the *resources*. Here is what goes into each of the files:

`provider.tf`:

```hcl
provider "azurerm" {
  features {}
}

terraform {
  required_providers {
    azurerm = {
      version = "~>2.74"
      source  = "hashicorp/azurerm"
    }
    null = {
      version = "~>3.0"
      source  = "hashicorp/null"
    }
  }
}
```

`variables.tf`:

```hcl
locals {
  name_suffix_long = "${var.project_name}-${var.environment}-${var.location}"
}

variable "project_name" {
  type        = string
  description = "Name of the project (i.e. energiq)"
}

variable "environment" {
  type        = string
  description = "Environment name (i.e. dev/uat/prod)"
}

variable "location" {
  type        = string
  description = "Region into which to deploy the resources"
  default     = "westeurope"
}

variable "aks_scheduler_version" {
  type        = string
  description = "Version of the scheduler func (any change will trigger function re-deployment)"
  default     = "v1"
}

variable "tags" {
  type        = map(string)
  description = "Tags to apply to the resources"
  default     = {}
}
```

`data.tf`:

```hcl
data "azurerm_subscription" "this" {}
```

`main.tf`:

```hcl
resource "azurerm_resource_group" "aks_scheduler" {
  name     = "rg-aksscheduler-${local.name_suffix_long}"
  location = var.location

  tags = merge(
    {
      Source         = "Terraform"
      DeploymentName = var.project_name
      Environment    = var.environment
      Location       = var.location
    },
    var.tags
  )
}

resource "azurerm_storage_account" "aks_scheduler" {
  name                     = "akssched-${local.name_suffix_long}"
  location                 = var.location
  account_tier             = "Standard"
  resource_group_name      = azurerm_resource_group.aks_scheduler.name
  account_replication_type = "LRS"

  tags = merge(
    {
      Source         = "Terraform"
      DeploymentName = var.project_name
      Environment    = var.environment
      Location       = var.location
    },
    var.tags
  )
}

resource "azurerm_app_service_plan" "aks_scheduler" {
  name                = "ap-aksscheduler-${local.name_suffix_long}"
  location            = var.location
  resource_group_name = azurerm_resource_group.aks_scheduler.name
  kind                = "FunctionApp"
  reserved            = true

  sku {
    size = "Y1"
    tier = "Dynamic"
  }

  tags = merge(
    {
      Source         = "Terraform"
      DeploymentName = var.project_name
      Environment    = var.environment
      Location       = var.location
    },
    var.tags
  )
}

resource "azurerm_function_app" "aks_scheduler" {
  name                = "af-aksscheduler-${local.name_suffix_long}"
  location            = var.location
  resource_group_name = azurerm_resource_group.aks_scheduler.name

  enabled                = true
  enable_builtin_logging = true
  https_only             = true
  os_type                = "linux"

  app_service_plan_id        = azurerm_app_service_plan.aks_scheduler.id
  storage_account_name       = azurerm_storage_account.aks_scheduler.name
  storage_account_access_key = azurerm_storage_account.aks_scheduler.primary_access_key
  version                    = "~3"

  app_settings = {
    FUNCTIONS_WORKER_RUNTIME              = "python"
    WEBSITE_RUN_FROM_PACKAGE              = "1"
    APPLICATIONINSIGHTS_CONNECTION_STRING = azurerm_application_insights.aks_scheduler.connection_string
    APPINSIGHTS_INSTRUMENTATIONKEY        = azurerm_application_insights.aks_scheduler.instrumentation_key
    AZURE_SUBSCRIPTION_ID                 = data.azurerm_subscription.this.subscription_id
  }

  site_config {
    linux_fx_version = "PYTHON|3.8"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = merge(
    {
      Source         = "Terraform"
      DeploymentName = var.project_name
      Environment    = var.environment
      Location       = var.location
    },
    var.tags
  )

  lifecycle {
    ignore_changes = [
      app_settings["WEBSITE_RUN_FROM_PACKAGE"]
    ]
  }
}

resource "azurerm_application_insights" "aks_scheduler" {
  name                = "af-aksscheduler-${local.name_suffix_long}"
  location            = var.location
  resource_group_name = azurerm_resource_group.aks_scheduler.name

  application_type     = "other"
  daily_data_cap_in_gb = 1
  retention_in_days    = 30

  tags = merge(
    {
      Source         = "Terraform"
      DeploymentName = var.project_name
      Environment    = var.environment
      Location       = var.location
    },
    var.tags
  )
}

resource "azurerm_role_definition" "aks_scheduler" {
  name        = "AKS Start Stop ${var.environment} Role"
  description = "Allows to read AKS clusters and perform Start and Stop operations on them"
  scope       = data.azurerm_subscription.this.id

  assignable_scopes = [data.azurerm_subscription.this.id]

  permissions {
    actions = [
      "Microsoft.ContainerService/managedClusters/read",
      "Microsoft.Resources/deployments/*",
      "Microsoft.ContainerService/managedClusters/start/action",
      "Microsoft.ContainerService/managedClusters/stop/action"
    ]
  }
}

resource "azurerm_role_assignment" "aks_scheduler_subscription" {
  description        = "Used to list AKS clusters in subscription and perform stop/start operations."
  role_definition_id = azurerm_role_definition.aks_scheduler.id
  scope              = data.azurerm_subscription.this.id
  principal_id       = azurerm_function_app.aks_scheduler.identity.0.principal_id
}

resource "null_resource" "aks_scheduler_publish" {
  provisioner "local-exec" {
    command = "sleep 30 && cd ${path.module}/files/aks-stop-starter && func azure functionapp publish ${azurerm_function_app.aks_scheduler.name} --python"
  }

  triggers = {
    func_change = var.aks_scheduler_version
  }

  depends_on = [
    azurerm_function_app.aks_scheduler
  ]
}
```

In the above Terraform code you may notice that I am using `null_resource` to actually deploy the Python function into the Azure function. There is currently no other way than using a `provisioner` with some script to do that as far as I am aware of. So, the `null_resource` triggers every time the variable `aks_scheduler_version` is updated. It means that you may update the Terraform or Python code, but the new version of the Python code won't be uploaded to Azure until the variable `aks_scheduler_version` is changed. That variable can contain literally anything. The `null_resource` also depends on the Azure Function and waits until the Function is deployed/updated and even then if it is triggered, it waits 30 seconds before it starts with the upload. This is to make sure that the Azure Function resource is really ready and the code can be uploaded.

Only remaining thing is to set up variables for example this way (most of the variables have some default values):

```bash
cat >>terraform.tfvars<<EOF
project_name = "myproject"
environment  = "dev"
EOF
```

And there we go! Just run the `terraform init` and `terraform apply`. This will create all the necessary resources in Azure and also uploads the Python code.

I don't really like the fact that another tool (the Azure Functions Core Tools) is involved, but the solution works pretty well. What I found annoying though, is that sometimes when you upload updated code it is not loaded by the Azure Function (the old code is still used). It can be usually resolved by stop and start of the Azure Function.

Anyway, I hope you found it usefull and enjoyed this post! (-:
