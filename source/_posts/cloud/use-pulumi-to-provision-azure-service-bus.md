title: Provisions Azure Service Bus Resources, Using Pulumi
date: June 13, 2021 08:47 PM
category: cloud
tags:
  - pulumi
  - azure
  - azure service bus
  - messaging
  - microservices
---

Nowadays it is very common that developers use one of the cloud service providers, such as Azure or AWS. One of the challenges working with those services in an enterprise application is how to manage the resources one may require in the cloud; for instance, Azure has a messaging service, Azure Service Bus, and you could bind Azure Functions to Topics and Queues within the bus. Azure functions will not start though, unless those topics and queues exist. By using pulumi in a CI/CD pipeline you could make sure that you always have the required resources before the deployment happens. Let's see more of pulumi in action.

<!-- more -->

[Pulumi](https://www.pulumi.com/) as they mentioned in their [documentation](https://www.pulumi.com/docs/get-started/) _is a modern infrastructure as code platform that allows you to use familiar programming languages and tools to build, deploy, and manage cloud infrastructure._ That means, a developer can use the programming language of her or his interest to manage cloud resources, and pulumi will also track changes of those resources for you based on your code base and the stack it creates.

In this article I want to show how to manage topics of Azure Service Bus leveraging pulumi and using C#, you could choose other languages. Let's begin. :)

**PS:** Make sure that you have [pulumi cli](https://www.pulumi.com/docs/get-started/install/) and [azure cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed on your machine, and you are logged into your azure subscription.


To start, create an empty folder, and navigate from your preferred command line tool in that directory and run the following command, and follow the instructions, if you want to use default values pass `--yes` or `-y` to the command.

```bash
pulumi new azure-csharp
```

This will create a project that could be used to create azure resources. There are 3 important files, `pulumi.yaml`, `pulumi.dev.yaml` and `MyStack.cs`. the first one includes general and common configurations for your pulumi stack, the second one, includes configurations specific to a special stack, in this case, `dev`. This could differ if you have chosen another stack name during creation. `MyStack.cs` contains the C# code to create pulumi stack and consequently azure resources. Pulumi uses changes in this file to track which cloud resources should be created, deleted, or updated on your cloud platform.

To create any resources on Azure, we need to have a [resource group](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/overview#resource-groups). Currently there should be one in `MyStack.cs` file.

```cs
var resourceGroup = new ResourceGroup("resourceGroup");
```

Keep this line in the file and remove the rest; of course, remove those that will not break the program from compiling :wink:.

# Azure Service Bus Namespace

Next step is to create an [azure service bus namespace](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-create-namespace-portal), this is a place where later on we could create topics and queues there.

```cs
var serviceBusNamespace = new Pulumi.AzureNative.ServiceBus.Namespace("sbns-pulumi-sample",
    new NamespaceArgs
    {
        ResourceGroupName = resourceGroup.Name
    });
```

**PS:** For naming conventions check [this](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming) link.

Now let's get back to the terminal and run `pulumi up`, you should see an output similar to this:

```bash
Previewing update (dev)

View Live: https://app.pulumi.com/shahabganji/PulumiAzureServiceBusSample/dev/previews/97d3a7e1-9507-43d0-b71f-624282ee8bc1

     Type                                     Name                             Plan
 +   pulumi:pulumi:Stack                      PulumiAzureServiceBusSample-dev  create
 +   ├─ azure-native:resources:ResourceGroup  rg-pulumi-sample                 create
 +   └─ azure-native:servicebus:Namespace     sbns-pulumi-sample               create

Resources:
    + 3 to create

Do you want to perform this update?  [Use arrows to move, enter to select, type to filter]
  yes
> no
  details
```

You will see that pulumi wants to `create` two resources, a `ResourceGroup` and a `Namespace` for service bus, choose details if you want to see more details about what will happen on this command, otherwise select `yes` and wait for the command to finish. The following should be the output:

```bash
Do you want to perform this update? yes
Updating (dev)

View Live: https://app.pulumi.com/shahabganji/PulumiAzureServiceBusSample/dev/updates/1

     Type                                     Name                             Status
 +   pulumi:pulumi:Stack                      PulumiAzureServiceBusSample-dev  created
 +   ├─ azure-native:resources:ResourceGroup  rg-pulumi-sample                 created
 +   └─ azure-native:servicebus:Namespace     sbns-pulumi-sample               created

Resources:
    + 3 created

Duration: 1m16s
```

Now we can add our topics and subscriptions to the service bus namespace, use the code below to tell pulumi to create those resources and run `pulumi up`.

```cs
var topic = new Pulumi.AzureNative.ServiceBus.Topic("sbt-pulumi-sample", new TopicArgs
{
    ResourceGroupName = resourceGroup.Name,
    NamespaceName = serviceBusNamespace.Name,
});

var subscription = new Pulumi.AzureNative.ServiceBus.Subscription("sbs-pulumi-sample", new SubscriptionArgs
{
    ResourceGroupName = resourceGroup.Name,
    NamespaceName = serviceBusNamespace.Name,
    TopicName = topic.Name
});
```

This time, you will see that pulumi detects only the changes you made and will only _*create*_ the new resources; you will see an output similar to the one below:


```bash
Previewing update (dev)

View Live: https://app.pulumi.com/shahabganji/PulumiAzureServiceBusSample/dev/previews/4e34159a-5746-41c3-82c2-2c3aaa704390

     Type                                     Name                             Plan
     pulumi:pulumi:Stack                      PulumiAzureServiceBusSample-dev
 +   ├─ azure-native:servicebus:Topic         sbt-pulumi-sample                create
 +   └─ azure-native:servicebus:Subscription  sbs-pulumi-sample                create

Resources:
    + 2 to create
    3 unchanged

Do you want to perform this update?  [Use arrows to move, enter to select, type to filter]
  yes
> no
  details
```

choose `yes` to create all the new resources. then you could navigate to [portal](https://portal.azure.com), there you should be able to see all the resources create by pulumi. If you are interested to see the stack that pulumi has created for you navigate to [pulumi app](https://app.pulumi.com/), log into your account and then on the home page you will see the list of stacks.

To cleanup all the creates resources in this sample run `pulumi destroy`, to delete all the resources create on the cloud, and then, if you want to totally remove the pulumi stack as well run `pulumi stack rm dev`.

# Conclusion

Pulumi facilitates the way developers can manage the resources on their cloud infrastructure, one of the great benefits of that is tracking changes made to the infrastructure and integrate these operations within the CI/CD pipeline, no matter which framework or language you are using, C# or python, Azure or AWS.

If you found this article useful please share it, and I hope you have a bug free coding week ahead :). You could find the code for this blog in [here](https://github.com/shahabisblogging/pulumi-azure-service-bus) on GitHub.
