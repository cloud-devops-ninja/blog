---
layout: post
title:  "Bicep: a more intuative way to deploy Azure resources"
date:   2023-02-26 12:00:00 +0100
authors: [cdo-ninja]
image: assets/images/posts/bicep-logo-730x250.png
tags: [blog]
hidden: false
toc: false
---

# Bicep: a more intuative way to deploy Azure resources

When it comes to deploying resources on Azure, defining infrastructure can be a complex and time-consuming process. Fortunately, there's a new tool available that simplifies the process and saves time and resources in the process: Bicep.

## What is Bicep?
Bicep is a domain-specific language (DSL) used to describe and deploy Azure resources. It is a declarative language that provides a simplified way of defining Azure resources in comparison to traditional JSON or YAML templates.

Bicep is open source and is developed and maintained by Microsoft.

## Why use Bicep?

Bicep simplifies the process of deploying Azure resources by allowing users to write more concise code. It also provides a simplified syntax that is easier to read and understand than traditional JSON or YAML templates.

In addition, Bicep provides features like parameterization, reusable modules, and better error messages that make it easier to build and maintain infrastructure.

## How does Bicep work?

Bicep files have a .bicep file extension and are compiled into ARM templates before deployment. This means that any Azure resource that can be defined using ARM templates can also be defined using Bicep.

Here's an example of what Bicep code looks like:

```bicep
param storageAccountName string
param location string

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-04-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}
```

This code creates an Azure storage account using the `Microsoft.Storage/storageAccounts@2021-04-01` resource type. The `param` statements at the top of the file define two parameters that can be passed in when the Bicep file is deployed.

## How to get started with Bicep?

To get started with Bicep, you'll need to install the Bicep CLI. The CLI is available for Windows, macOS, and Linux.

Once you have the Bicep CLI installed, you can create a new Bicep file using your favorite text editor or IDE. 

*Note: I personally recommend using VS Code, combined with the [Bicep Extension for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep), as your code editor to use the extension supporting features, like scaffolding, snippets and linter to make create your first .bicep file a breeze!*

You can also use the `bicep new` command to generate a basic Bicep file.

After you've created your Bicep file, you can use the `bicep build` command to compile it into an ARM template. You can then use the `az deployment group create` command to deploy the ARM template.

*Note: As of Azure CLI version 2.20.0 and Azure PowerShell 5.6.0 you no longer need to compile the bicep template to an ARM template, but can use the bicep template as direct input for the resource deployment commands. Azure CLI and Azure PowerShell will automatically compile your bicep template before and offer the compiled ARM template to the Azure Resource Manager*

## Conclusion

Bicep is a powerful tool for deploying Azure resources. It simplifies the process of defining infrastructure by providing a more concise syntax and additional features like parameterization and reusable modules.

If you're new to Bicep, I recommend checking out the official Bicep documentation and experimenting with some sample code to get a feel for how it works.

With Bicep, you can create and manage your Azure infrastructure in a more streamlined and efficient way, saving time and resources in the process. So why not give it a try and see how it can help you with your Azure deployments?
