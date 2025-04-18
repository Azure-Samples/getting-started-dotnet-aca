# Cost optimization for ACA

Let's face it, we all want to save money. And there are steps we can take to optimize the spend on Azure Container Apps. In this chapter we'll take a look at some of those strategies.

Before we dive into the fine-tuning, it's important to understand what scenarios are best suited for Azure Container Apps, and understand how the cost is calculated. Then we can look at the cost optimization strategies, and how to monitor our apps and stay informed about the costs.

## A service for each scenario

When looking at [Azure Container Apps documentation](https://learn.microsoft.com/azure/container-apps/) in the section **About Azure Container Apps** there is a page [**Compare Container options in Azure**](https://learn.microsoft.com/azure/container-apps/compare-options)

Going through all the different services where containers can be used we can learn that:

- Azure Container Apps is 
  - Optimized to run general purpose containers, especially for applications that span many microservices deployed in containers.
  - Powered by Kubernetes and open-source technologies like Dapr, KEDA, and envoy.
  - Supports Kubernetes-style apps and microservices with features like service discovery and traffic splitting.
  - Enables event-driven application architectures by supporting scale based on traffic and pulling from event sources like queues, including scale to zero.
  - Supports running on demand, scheduled, and event-driven jobs.

- Azure Container Apps is not/does not 
  - Does not provide direct access to the underlying Kubernetes APIs. If you require access, you should use Azure Kubernetes Service (AKS). 
  - Is not the less "opinionated" choice when you need only one isolated container that doesn't require scaling, load balancing, or certificates. In those cases, Azure Container Instances (ACI) could be preferable.
  - When more ephemeral code or containers are needed, Azure Functions could be a better choice.

So Azure Container Apps is a good choice for microservices and event-driven applications. It is not the best choice for isolated containers or when you need direct access to Kubernetes APIs.

## Understanding the cost

Looking at those two pages we can understand better how the cost is calculated. 

- [Billing in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/billing)
- [Azure Container Apps Pricing](https://azure.microsoft.com/pricing/details/container-apps/)

There is a consumption plan and a dedicated plan. The consumption plan is the default one, and it is based on the resources consumed by the container apps. The dedicated plan is based on the workload profile instances, not by individual applications.

Let's look at each a bit more.

### Consumption plan

By default, Azure Container Apps runs in the Consumption plan and there are two types of charges:

- Resource consumption: The amount of resources allocated to your container app on a per-second basis, billed in vCPU-seconds and GiB-seconds.
- HTTP requests: The number of HTTP requests your container app receives.
  
The following resources are free during each calendar month, per subscription:

- The first 180,000 vCPU-seconds
- The first 360,000 GiB-seconds
- The first 2 million HTTP requests

The number of running replicas will also impact the cost. We will see how we can change this, but before we do it's important to mention another plan.

### Dedicated plan

Billing for apps and jobs running in the Dedicated plan is based on workload profile instances, not by individual applications. 

## Let's start optimizing!

You can reuse the code and deployed solution from the previous chapters. If you deleted the resources, you can redeploy the solution using the following steps.

1. Getting the Repository Root

	To simplify the copy paste of the commands that sometimes required an absolute path, we will be using the variable `REPOSITORY_ROOT` to keep the path of the root folder where you cloned/ downloaded this repository. The command `git rev-parse --show-toplevel` returns that path.

	```bash
	# Bash/Zsh
	REPOSITORY_ROOT=$(git rev-parse --show-toplevel)
	```

	```powershell
	# PowerShell
	$REPOSITORY_ROOT = git rev-parse --show-toplevel
	```

1. Move to the **6-cost-optimization/sample** directory.

    ```bash
    cd $REPOSITORY_ROOT/cost-optimization/sample
    ```

1. Initialize the Azure Developer CLI (azd) in the current directory.

    ```bash
    azd init
    ```

1. Provision and deploy the microservice apps to ACA.

    ```bash
    azd up
    ```

1. Open the browser and navigate to the deployed app to validate that it's working as expected.
   

## Change the MIN and MAX number of replicas

You can change the number of replicas and the scaling directly in the Azure portal, but doing so would be overridden each time you deploy using the AZD CLI or the CI/CD pipeline. A better way would be to modify the bicep file and redeploy the app.

Open the **infra/resources.bicep** file and look for the `scaleMinReplicas` and `scaleMaxReplicas` for each of the containers. 

1. Change the minimum values from **1** to **0**. This will allow the container to scale to zero when there is no traffic. When the App is accessed again, the container will be started again. This will save cost when the app is not being used. This is a good strategy for development, testing environments, and even for production environments that are not used 24/7.
1. Change the maximum value to **1**. This will limit the number of replicas to 1. This will save cost as it limits the number of replicas that can be started. This can be a good strategy for development or when the app doesn't need to scale.

```bicep
module eshopliteProducts 'br/public:avm/res/app/container-app:0.8.0' = {
  name: 'eshopliteProducts'
  params: {
    name: 'eshoplite-products'
    ingressTargetPort: 8080
    // scaleMinReplicas: 1
    // 👇👇👇 Change the number of Replicas below
    scaleMinReplicas: 0
    scaleMaxReplicas: 1
    // 👆👆👆 Change the number of Replicas above
    // scaleMaxReplicas: 10
```

```bicep
module eshopliteStore 'br/public:avm/res/app/container-app:0.8.0' = {
  name: 'eshopliteStore'
  params: {
    name: 'eshoplite-store'
    ingressTargetPort: 8080
    // scaleMinReplicas: 1
    // 👇👇👇 Change the number of Replicas below
    scaleMinReplicas: 0
    scaleMaxReplicas: 1
    // 👆👆👆 Change the number of Replicas above
    // scaleMaxReplicas: 10
```

```bicep
module eshopliteStoreinfo 'br/public:avm/res/app/container-app:0.8.0' = {
  name: 'eshopliteWeather'
  params: {
    name: 'eshoplite-storeinfo'
    ingressTargetPort: 8080
    // scaleMinReplicas: 1
    // 👇👇👇 Change the number of Replicas below
    scaleMinReplicas: 0
    scaleMaxReplicas: 1
    // 👆👆👆 Change the number of Replicas above
    // scaleMaxReplicas: 10
```

1. Redeploy the app. If you are using the CI/CD pipeline from Chapter 5, you can push the changes to the repository and the pipeline will deploy the changes. If you are using the AZD CLI, you can run the following command.

    ```bash
    azd up
    ```

1. You can visualize the number of replicas in the Azure portal. Go to the Azure Container Apps resource, and click on one of the Container Apps (ex: eshoplite-store). From the left menu, in the `Application` section, click on `Scale`, to see the number of replicas.

There are many other things you can do to optimize the cost, like using the dedicated plan, using the right size of the container, and using the right number of replicas and custom Scaling rules. Keep in mind to do those changes in the bicep file, so you can keep track of the changes and redeploy the app when needed.

## Monitoring the Cost

Optimizing the cost is not a one-time task, it's an ongoing process. It's a good idea to monitor your applications and stay informed about the cost. There are many ways and levels of detail you can monitor the cost of resources in Azure. Let's create a budget for the Azure Container Apps resource we just deployed.

1. Go to the Azure portal and navigate to the Resource Group where the Azure Container Apps resource is deployed (ex: rg-cost-opt-chapter). From the left menu, expand the **Cost Management** section and click on **Budgets**.
   
1. Click Add to create a new budget. Fill the form and specify the date, period, and amount you want to set for the budget. You can also set alerts with different thresholds (ex: 50%) to be notified by email when those are reached.

1. Click on **Create** to finalize the budget.

## Learn more

Learn more about [Azure Monitor cost and usage](https://learn.microsoft.com/azure/azure-monitor/cost-usage).

## Clean up the deployed resources

To clean up the resources, run the following command:

```bash
azd down --force --purge
```

### Video

[![Getting Started with .NET on ACA - Part 6](images/ep6_thumb_yt_small.jpg)](https://www.youtube.com/watch?v=JMgQ_RAI5Ao&list=PLI7iePan8aH6jQxDupYUvgQsP3G7WGM0b&index=6)


## Up next

There's nothing worse than your application going down and you not knowing about it. (Well - maybe if your boss knows about it before you do). In the next chapter we'll look at your options for monitoring your application in Azure Container Apps.

👉 [Monitoring ACA](../7-monitoring/)