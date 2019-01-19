
# Lab

GitHub repo: https://aka.ms/apim/labready2019 

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
2. Add new HTTP trigger Function. Name it **Order** and leave authorization level as **Function**. Click **Create**.
3. Navigate to **Integrate** tab.
4. Unselect **GET** in **Selected HTTP methods** and click **Save**.
5. Click on **+ New Output**. Pick **Azure Service Bus** and click on **Select**.
6. In the **Extensions not Installed** section, click **Install**.
7. Click **new** in the **Service Bus connection** section. Choose **store-msg-...** as the namespace and click **Select**.
8. Choose **Service Bus queue** in the **Message type** section. 
9. Set **Queue name** to **store-msg-queue-1**.
10. Click **Save**.
11. Go to the Function implementation. Replace the code of the Function with:

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
12. Click **Save and run** to see if the Function works. You should the response:
    ```JSON
    {
        "name": "Azure"
    }
    ```

### Implement Function App store-product

1. Navigate to the Function App **store-product-...**.
2. Add new HTTP trigger Function. Name it **Product** and leave authorization level as **Function**. Click **Create**.
3. Navigate to **Integrate** tab.
4. Unselect **POST** in **Selected HTTP methods** and click **Save**.
5. Click **Save**.
6. Go to the Function implementation. Replace the code of the Function with:

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
7. Click **Save and run** to see if the Function works.
8. On the right sidebar, in the **Query** section, click **+ Add parameter**. Set name to **name** and value to **Microsoft Surface**.
9. Click **Run** in the right sidebar and you should see the response:
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

### Test the API

1. Go to the **Test** tab in the top menu.
2. Select **GET Product** from the API operations list.
3. Click **+ Add parameter**. Set name to **name**, value to **Microsoft Surface**.
4. Click **Send**.
5. You should a **200 OK** response:

    ```
    HTTP/1.1 200 OK
    
    cache-control: private
    ...
    x-powered-by: ASP.NET,ASP.NET

    {"Name":"Microsoft Surface","Description":"Coming soon..."}
    ```

## Exercise 3: Manage and protect the Store API

### Implement a throttling policy

1. Go to the **Design** view of the **Store API** in your API Management instance.
2. Click the **</>** icon in the **Inbound processing** section.
3. Add the **rate-limit** line in the ```<inbound>...</inbound>``` section as follows:

    ```XML
    <inbound>
        <base />
        <rate-limit calls="5" renewal-period="30" />
    </inbound>
    ```
4. Click **Save**.

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
2. Click **...** next to your API name on the left sidebar. Select **Add version**.
3. Set name to **store-api-v2**.
4. Set version identifier to **v2**.
5. Click **Create**.
6. Make sure **v2** is selected on the left side bar.
7. Click **...** next to the **POST Product** API operation.
8. Select **Delete** and click **Yes**.
9. Compare the versions. **Original** should have the **POST Product** operation, while **v2** shouldn't.

### Add a CORS policy

1. Go to the **Design** view of the **Store API** in your API Management instance.
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
4. You should a **200 OK** response:

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

### Generate your API usage

1. Go to the **Test** tab in the top menu.
2. Select **GET Product** from the API operations list.
3. Click **Send** a few times.

### See the logs in Application Insights

1. Navigate to the Application Insights instance **store-logs** located in your resource group.
2. Look at the dashboards in the overview.
3. Go to **Application map** from the menu on the left.
4. Click on the **storeapi...** node, and then on the **Investigate performance** on the right.
5. Click on the **Drill into (xx) samples** button in the bottom right corner.
6. Select a sample from the list and view the properties of the request.

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
