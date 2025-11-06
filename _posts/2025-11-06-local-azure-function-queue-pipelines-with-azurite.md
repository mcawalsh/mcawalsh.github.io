---
layout: post
title: "Building a Local Azure Function Queue Pipeline with Azurite"
date: 2025-11-06 10:00:00 +0000
categories: [Azure, Functions]
tags: [azure-functions, azurite, queue-trigger, dotnet, local-development]
description: >
  A hands-on walkthrough of building a local Azure Function pipeline that processes queue messages
  using Azurite for storage emulation — no cloud resources required.
---
[Source code on GitHub → mcawalsh/functionapp-purchase-processor](https://github.com/mcawalsh/functionapp-purchase-processor)


## Getting Started

As part of my prep for the AZ-204 exam I wanted to play around a little with Azure Function Apps to refresh my memory. It's been a long time since I worked with Function Apps and I'm sure a lot of things have changed so I thought it was going to be a fun little playground.

The first thing I did was settle on an idea and what I settled on was a mini purchase processing pipeline. It's a little contrived and at some points you have to just accept that this is a learning experience and isn't going to fully match the real world flows.

So, the flow:
1. Input queue message - Purchase Created
2. Function processes the message  
Logs receipt of the purchase  
3. Sends new message to output queue, `purchase-processed`

To do all of this and because in this instance I do not want to host everything in Azure I'm going to emulate Storage Queues using Azurite and CosmosDB using the Azure Cosmos DB emulator. Since I've never used these emulators before I thought, "Why not? What's another new thing in the list?"


### 🧱 Architecture Overview

```text
Azurite Queue (Purchases)
        ↓
   Azure Function
        ↓
Azurite Queue (ProcessedPurchases)
```

## Function App

One of the first decisions I had to make was to choose what type of Function App model I was going to run with. In-Process or Isolated.

In-Process the function runs in the same process as the Azure Functions runtime, you use attributes like `[FunctionName]` and `[QueueTrigger]` directly on methods, and I believe it is more likely to be questioned on the AZ-204 exam.

Isolated functions run in a separate process. It uses DI and Program.cs sets up the services. The attribute model is similar but class bassed, e.g. `[Function("FuncName")]`

I decided to use the In-Process model because it runs directly inside the Azure Functions runtime and aligns closely with how triggers and bindings are described in the AZ-204 exam.

## Prereqs
1. VSCode Extensions
    - Azurite
2. Azure Storage Explorer

## Step 1. Set Up Azurite (Azure Storage Emulator)

1. After installing the Azurite as a VSCode extension, CTRL+SHIFT+P -> `Azurite: Start`
2. Get Azure Storage Explorer.
3. [Connect to Azure Storage Explorer](https://learn.microsoft.com/en-us/azure/storage/common/storage-connect-azurite?tabs=blob-storage#connect-the-azure-storage-explorer)


## Step 2. Run the QueueMessageSender

The QueueMessageSender is a console app that will ask for a customer name, number of items, and then a name, quantity, and price for each item. Then it will send it to the products queue.

```bash
dotnet build
dotnet run
```

![Example Purchases Message ](./assets/img/20251106/example-purchases-message.png)

# Step 3. Function App

Next I created the Azure Functions project and added a queue trigger.

```bash
func init PurchaseProcessor --worker-runtime dotnet --target-framework net8.0
cd PurchaseProcessor

func new --name ProcessPurchase --template "Queue trigger"
```

This created the ProcessPurchase.cs file which I updated with the following. 

This attribute sets the return queue
```csharp
[return: Queue("purchases-processed", Connection = "AzureWebJobsStorage")]
```

And then updating the `QueueTrigger` attribute to target the `purchases` queue.

```csharp
public void Run([QueueTrigger("purchases", Connection = "AzureWebJobsStorage")] string queueMessage, ILogger log)
```

One thing to note here is that because I'm not base64 encoding my strings when sending them with my console app I needed to make sure I updated the host.json file with the following so that it could accept it when triggering the function.

```json
"extensions": {
  "queues": {
    "messageEncoding": "none"
  }
},
```

> [!NOTE]
> If something goes wrong when you're trying to process the message then by default after 5 attempts the message will be sent to a -poison queue. So in this instance if a message fails from the purchases queue then it will be moved to a purchases-poison queue.

Run the app:

```bash
func start
```

![Example of Purchases Processed](./assets/img/20251106/purchases-processed.png)

Flow Summary:  
QueueMessageSender  →  "purchases" queue  
Azure Function (ProcessPurchase)  →  "purchases-processed" queue

🏁 Wrapping Up

With Azurite and the Azure Functions Core Tools, we can simulate real Azure event-driven flows locally — no cloud resources required.
This setup gives a great sandbox for experimenting with triggers, bindings, and message processing, and it mirrors the kind of architecture covered in the AZ-204 exam.

I will say that getting Azurite to work as expected was not an easy process. I flipped between a Docker version and the VSCode extension managed version before getting it to work. (I think I spent more time with Azurite than I did with the function app) It's all worth it in the end and I think I have refreshed my memory enough.