# Implement, manage and secure a scalable serverless application with API Management

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmikebudzynski%2Fapim-lab-2019-ready%2Fmaster%2Fazuredeploy.json" target="_blank"><img src="http://azuredeploy.net/deploybutton.png"/></a>

Instructions for the technical lab at Microsoft Ready (Winter 2019).

Instructors: 
- Mike Budzynski
- Elizabeth Graham
- David Barkol

## Exercise 1: Create and implement resources

### View your resources

1. Navigate to the Azure Portal. You should see a resource group with the following resources:
    - API Management: store-api-...
    - Application Insights: store-logs
    - Service Bus Namespace: store-msg-...
    - Function App: store-order-...
    - Function App: store-product-...
    - Logic App: store-shipping

### Implement Function App store-order

1. Navigate to the Function App **store-order-...**.
1. Add new HTTP trigger Function. Click on the **+** icon next to **Functions**. Select **In portal -> Continue -> More templates -> Finish**.
1. Click on the **Http Trigger** tile.
1. Name the Function **Order** and leave authorization level as **Function**.
1. Click **Create**.
1. Navigate to **Integrate** tab.
1. Unselect **GET** in **Selected HTTP methods** and click **Save**.
1. Click on **+ New Output**. Pick **Azure Service Bus** and click on **Select**.
1. In the **Extensions not Installed** section, click **Install**.
1. Click **new** in the **Service Bus connection** section. Choose **store-msg-...** as the namespace and click **Select**.
1. Choose **Service Bus queue** in the **Message type** section. 
1. Set **Queue name** to **store-msg-queue-1**.
1. Important: wait at least 1-2 minutes.
1. Click **Save**.
1. Go to the Function implementation. Replace the code of the Function with:

    ```C#
    public static string Run(HttpRequest req, ILogger log, out string outputSbMsg)
    {
        // Read the body from the HTTP request
        var message = new StreamReader(req.Body).ReadToEnd();
        
        // Log for debugging
        log.LogInformation($"C# function processed: {message}");
    
        // Queue message in the Service Bus
        outputSbMsg = message;
    
        // Return HTTP response
        return message;
    }
    ```
1. Click **Save**.

### Implement Function App store-product

1. Navigate to the Function App **store-product-...**.
1. Add new HTTP trigger Function. Name it **Product** and leave authorization level as **Function**. Click **Create**.
1. Go to the Function implementation. Replace the code of the Function with:

    ```C#
    #r "Newtonsoft.Json"
    
    using System.Net;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Extensions.Primitives;
    using Newtonsoft.Json;
    
    public class Product
    {
        public string Name { get; private set; }
        public string Description { get; private set; }
    
        public Product(string name, string description = "")
        {
            this.Name = name;
            this.Description = description;
        }
    }
    
    public static async Task<IActionResult> Run(HttpRequest req, ILogger log)
    {
        // Retrieve product name from the query parameter
        string name = req.Query["name"];
    
        // Create product object (this is a mock, normally it would come from a database).
        // Then, serialize it and return.
        var product = new Product(name ?? string.Empty, "Coming soon...");
        var productSerialized = JsonConvert.SerializeObject(product);
        
        return (ActionResult) new OkObjectResult(productSerialized);
    }
    ```
1. Click **Save and run** to see if the Function works.
1. On the right sidebar, in the **Query** section, click **+ Add parameter**. Set name to **name** and value to **Microsoft Surface**.
1. Select the **GET** operation from the dropdown.
1. Click **Run** in the right sidebar and you should see the response:
    ```JSON
    {"Name":"Microsoft Surface","Description":"Coming soon..."}
    ```

## Exercise 2: Front Azure Function Apps with Azure API Management

### Create an API

1. Navigate to the API Management instance **store-api-...** located in your resource group.
2. Click on **APIs** on the left sidebar.
3. Select **Function App** in the **Add a new API** view.
4. Click **Browse** to locate the Function App. Select the **store-product-...** Function App.
5. Set **Display name** to **Store API**.
6. Set **Name** to **store-api**.
7. Set **API URL suffix** to **api**.
8. Click **Create**.
9. Click **...** next to your API name on the left sidebar. Select **Import**.
10. Browse for the **store-order-...** app and confirm import.

### Make sure the Service Bus output works

Unfortunately, there is a bug in Azure Functions, which most likely prevented your *Order* Function from correct activation. To make sure your Function works, follow the steps below:

1. Navigate to the Function App **store-order-...**.
1. Go to the Function implementation. Click **Run** to see if the Function works. You should see the response:
    ```JSON
    {
        "name": "Azure"
    }
    ```
1. If you do not see the response, you need to recreate the Function's output - continue with the steps below. If you see the correct response, you can skip to the next section *Test the API*.
1. Navigate to **Integrate** tab.
1. Click on the existing Service Bus output.
1. Select **Delete**.
1. Click on **+ New Output**. Pick **Azure Service Bus** and click on **Select**.
1. In the **Extensions not Installed** section, click **Install**.
1. The **Service Bus connection** should already be propagated with **store-msg-...**.
1. Choose **Service Bus queue** in the **Message type** section.
1. Set **Queue name** to **store-msg-queue-1**.
1. Click **Save**.
1. Wait 2 minutes for the changes to propagate.
1. Go to the Function implementation. Click **Run** to see if the Function works. You should see the response:
    ```JSON
    {
        "name": "Azure"
    }
    ```

### Test the API

1. Navigate to the API Management instance **store-api-...** located in your resource group.
1. Click on **APIs** on the left sidebar and go to the **Store API**.
1. Go to the **Test** tab in the top menu.
1. Select **POST Order** from the API operations list.
1. In the **Request body** paste:

    ```
    {
        "id": "14157194",
        "address": "One Microsoft Way, Redmond, WA, USA",
        "products": 
        [
            "31", 
            "145"
        ]
    }
    ```

1. Click **Send**.
1. You should receive a **200 OK** response:

    ```
    HTTP/1.1 200 OK
    
    cache-control: private
    ...
    x-powered-by: ASP.NET,ASP.NET
    {
        "id": "14157194",
        "address": "One Microsoft Way, Redmond, WA, USA",
        "products": 
        [
            "31", 
            "145"
        ]
    }
    ```

## Exercise 3: Manage and protect the Store API

### Implement a throttling policy

1. Go to the **Design** view of the **Store API** in your API Management instance.
1. Make sure you have selected **All operations** on the left bar with API operations.
1. Click the **</>** icon in the **Inbound processing** section.
1. Place a cursor in the `<inbound>` section:
    ```XML
        <inbound>
            <base />
            |
        </inbound>
    ```
1. Select **Limit call rate per key** from the right sidebar. It will automatically add the **rate-limit-by-key** line in the ```<inbound>...</inbound>``` section as follows:

    ```XML
    <inbound>
        <base />
        <rate-limit-by-key calls="number" renewal-period="seconds" counter-key="@()" />
    </inbound>
    ```

1. Fill the placeholders in the policy with the information below:

    ```XML
    <rate-limit-by-key calls="5" renewal-period="30" counter-key="@(context.Subscription.Key)" />
    ```

1. Click **Save**.

### Test the throttling policy

1. Go to the **Test** tab in the top menu.
2. Select **GET Product** from the API operations list.
3. Click **+ Add parameter**. Set name to **name**, value to **Microsoft Surface**.
4. Click **Send**.
5. You should a **200 OK** response.
6. Click **Send** a few more times. You should see a response:

    ```
    HTTP/1.1 429 Too many requests
    
    cache-control: private
    ...
    x-powered-by: ASP.NET
    {
        "statusCode": 429,
        "message": "Rate limit is exceeded. Try again in 28 seconds."
    }
    ```

### Create a new version of the API

1. Go to the **Design** view of the **Store API** in your API Management instance.
1. Click **...** next to your API name on the left sidebar. Select **Add version**.
1. Set name to **store-api-v2**.
1. Set version identifier to **v2**.
1. Click **Create**.
1. Make sure **v2** is selected on the left side bar. Click on **Original** and then **v2** to load it.
1. Click **...** next to the **POST Product** API operation.
1. Select **Delete** and click **Yes**.
1. Go to the **Design** view.
1. Select **...** next to `rate-limit-by-key` in the **Inbound processing** section and click **Delete**.
1. Click **Save** at the bottom.
1. Compare the versions. **Original** should have the **POST Product** operation, while **v2** shouldn't.

### Add a CORS policy

1. Go to the **Design** view of the **Store API** in your API Management instance, and click **All operations**.
2. Click **+ Add policy** in the **Inbound processing** section.
3. Select the **Allow cross-origin resource sharing (CORS)** tile.
4. Click **Save**.

### Add a policy to rewrite query parameters

1. Go to the **Design** view of the **Store API** in your API Management instance.
2. Select the **GET Product** operation.
3. Click **+ Add policy**.
4. Select **Set query parameters**.
5. Set name to **name**, value to **Microsoft Xbox**, action to **skip**.
6. Click **Save**.

### Test the query parameters rewrite policy

1. Go to the **Test** tab in the top menu.
2. Select **GET Product** from the API operations list.
3. Click **Send**.
4. You should see a **200 OK** response:

    ```
    HTTP/1.1 200 OK
    
    cache-control: private
    ...
    x-powered-by: ASP.NET,ASP.NET
    {"Name":"Microsoft Xbox","Description":"Coming soon..."}
    ```

## Exercise 4: Monitor the Store API

### Add an Application Insights logger

1. Navigate to the API Management instance **store-api-...** located in your resource group.
2. Click **Application Insights** from the menu on the left.
3. Click **+ Add**.
4. Select **store-logs** as your Application Insights instance.
5. Click **Create**.

### Enable logging to Application Insights

1. Select **APIs** from the menu on the left and navigate to the **Store API, v2**.
2. Click on the **Settings** tab on the top.
3. Scroll down to the **Diagnostics logs** section and select **Enable** in the Application Insights view.
4. Set **Destination** to **store-logs**.
5. Select **Sampling** to **100%** for testing purposes.
6. Click **Save**.

### See the existing logs in Application Insights

1. Navigate to the Application Insights instance **store-logs** located in your resource group.
2. Look at the dashboards in the overview.
3. Go to **Application map** from the menu on the left.
4. You should be able to see two Function Apps and a Service Bus there.

### Generate your API usage

1. Navigate back to the API Management instance **store-api-...** located in your resource group.
1. Select **APIs** from the menu on the left and navigate to the **Store API, v2**.
1. Make sure at least 2 minutes have passed since you completed the steps in *Enable logging to Application Insights*.
1. Go to the **Test** tab in the top menu.
1. Select **GET Product** from the API operations list.
1. Click **Send** a few times.

### See the logs in Application Insights

1. Navigate back to the Application Insights instance **store-logs** located in your resource group.
1. You need to wait a few minutes since generating the API usage.
1. Look at the dashboards in the overview.
1. Go to **Application map** from the menu on the left.
1. Now you should be able to see another node for API Management: **storeapi...**. Click on it and then on the **Investigate performance** on the right.
1. Click on the **Drill into (xx) samples** button in the bottom right corner.
1. Select a sample from the list and view the properties of the request.

## Exercise 5 (optional): Cache responses

### Add a cache in API Management

1. Navigate to the API Management instance **store-api-...** located in your resource group.
2. Select **External cache** from the menu on the left.
3. Click **+ Add**.
4. Select **store-cache-...** as the cache instance.
5. Click **Save**.

### Add a policy for caching responses

1. Select **APIs** from the menu on the left and navigate to the **Store API, v2**.
2. Select the **GET Product** operation.
3. Click **+ Add policy**.
4. Select the **Cache responses** tile.
5. Specify duration as **3600** seconds.
6. Switch to the **Full** view.
7. Click **+ Add query parameter** and set it to **name**.
8. Click **Save**.

### Test the caching

1. Go to the **Test** tab in the top menu.
2. Select **GET Product** from the API operations list.
3. Click **+ Add parameter**. Set name to **name**, value to **Microsoft Surface**.
4. Click **Send**.
5. You should a **200 OK** response.
6. Go to the **Trace** tab. At the end of the **Inbound** section you should see:
    ```JSON
    cache-lookup (214.292 ms)
    {
        "message": "Cache lookup resulted in a miss. Cache headers listed below were removed from the request to prompt the backend service to send back a complete response.",
        "cacheKey": "...",
        "cacheHeaders": [
            {
                "name": "Cache-Control",
                "value": "no-cache, no-store"
            }
        ]
    }
    ```
7. Click **Send** again.
8. Now in the **Trace** tab you should see:
    ```JSON
    cache-lookup (67.351 ms)
    {
        "message": "Cache lookup resulted in a hit! Cached response will be used. Processing will continue from the step in the response pipeline that is after the corresponding `cache-store`.",
        "cacheKey": "..."
    }
    ```

## Additional resources

If instructed, please visit this link: [https://aka.ms/apim/labready2019](https://aka.ms/apim/labready2019)