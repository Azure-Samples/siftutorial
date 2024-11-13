# Project Name: SQL database In Fabric Tutorial - Application Development

This application is part of the [SQL database In Fabric tutorial](https://aka.ms/fabric-sql-tutorial). In this step you will create the application for the GraphQL Endpoint you created in the SQL In Fabric tutorial. So far in the tutorial you have created a database which stores the sales and products for Contoso, and added suppliers and joining entities using Transact-SQL (T-SQL). You now wish to allow developers to use the data without having to learn T-SQL, and also enable them to query multiple Microsoft Fabric components in a single interface. 

You have been asked to create an application that will show all affected Suppliers if a Location has a supply chain break, due to natural disasters or other interruptions. This code shows how to create an ASP.NET application that uses a GraphQL Query to access a Query in the SQL database In Fabric GraphQL endpoint you created in the last section of the tutorial. You will need to [install the .NET Framework, version 8.0 or higher](https://dotnet.microsoft.com/en-us/download/dotnet-framework) for this section.

1. **Create a new ASP.NET Core Web Application**:
   - Open a terminal or command prompt.
   - Run the following command to create a new ASP.NET Core web application:
     ```bash
     dotnet new webapp -n GraphQLWebApp
     cd GraphQLWebApp
     ```

2. **Add the necessary packages**:
   - Run the following commands to add the required packages:
     ```bash
     dotnet add package Azure.Identity
     dotnet add package GraphQL
     dotnet add package GraphQL.Client
     dotnet add package GraphQL.Client.Serializer.Newtonsoft
     ```
3. **Add the Azure CLI software and log in to your subscription**:

In some cases you may need to have the Azure CLI tool available and log in using its commands to set the security context properly if you have multiple subscriptions. Note: If you are running on a system other than Microsoft Windows, you can [check this reference for installation steps](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli). 

   - Run the following commands to add the required software and log in:
     ```bash
     winget install -e --id Microsoft.AzureCLI
     az login
     ```
If prompted, select the Subscription ID you used to create the tutorial assets. 

4. **Modify the `Program.cs` file**:
   - Open the `Program.cs` file and replace its content with the following code:
```csharp
/* Program.cs
Purpose: This tuorial program shows how to create a simple web application that uses a GraphQL query to return data from a Power BI dataset. 
Uses the GraphQL.Client library to send the query to the Power BI service, and then displays the results in a web page. 
Uses the DefaultAzureCredential class from the Azure.Identity library to authenticate with Azure Active Directory and 
get an access token to use in the query. 
It uses the WebApplication class from the Microsoft.AspNetCore.Components.WebAssembly.Hosting library to create a web application, 
and the GraphQLHttpClient class from the GraphQL.Client library to send the query. 
It also uses the GraphQLHttpRequest class from the GraphQL.Client library to define the query, and the NewtonsoftJsonSerializer 
class from the GraphQL.Client.Serializer.Newtonsoft library to serialize the response. 
We use the WriteAsync method of the HttpResponse class to write the HTML response to the web page.

Requirements: To run this program, you need to have the .NET SDK installed on your machine. 
You also need to have an Azure subscription and a Power BI dataset with a GraphQL API enabled.
You should install the required libraries with the following commands:

dotnet add package GraphQL.Client --version 3.3.0
dotnet add package GraphQL.Client.Serializer.Newtonsoft --version 3.3.0
dotnet add package Microsoft.AspNetCore.Components.WebAssembly.Hosting --version 5.0.7
dotnet add package Azure.Identity --version 1.4.1

Author: Buck Woody, Microft Corporation 
Last Modified: 2021.09.01
*/

// Set up your libraries
using GraphQL.Client.Http;
using GraphQL.Client.Serializer.Newtonsoft;
using Azure.Identity;

// Make a connection to Azure, and get a token
var credential = new DefaultAzureCredential();
var token = await credential.GetTokenAsync(new Azure.Core.TokenRequestContext(new[] { "https://analysis.windows.net/powerbi/api/.default" }));

// Set up your web application
var builder = WebApplication.CreateBuilder(args);
// Add services to the container.
builder.Services.AddRazorPages();
// Build the application
var app = builder.Build();

// Configure the HTTP request pipeline.
app.UseStaticFiles();
app.UseRouting();

// Add the Razor pages
app.MapRazorPages();
// Add the GraphQL query initial web page
app.MapGet("/", async context =>
{
    var html = @"
<!DOCTYPE html>
<html lang='en'>
<head>
    <meta charset='UTF-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1.0'>
    <title>Product Count by Suppliers</title>
    <link rel='stylesheet' href='https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css'>

    <style>
        .form-inline .form-control {
            width: auto;
            flex: 0 0 50px; /* Adjust the width as needed */
        }
        .form-inline .btn {
            margin-left: 10px; /* Adjust the margin as needed */
        }
    </style>
</head>

<body>
    <div class='container mt-5'>
      <img src='./contoso.png' alt='Contoso Logo' width='200'>
        <h3>Product Count by Suppliers</h3>
        <p><i>Supplier Network Control Center - Supplier Sector Impact Zones by Product (Function NCC-1701)</i></p>
        <p>Enter the impacted Location ID for the Supplier that is facing the outage. The system will return the other Suppliers within that impact area and show the count of items each Supplier provides to Manufacturing.</p>
        <form method='get' action='/graphql' class='form-inline'>
            <label for='locationId' class='mr-2'>Location ID:</label>
            <input type='text' class='form-control' id='locationId' name='locationId' required>
            <button type='submit' class='btn btn-primary'>Search</button>
        </form>
        <div id='result'></div>
    </div>
</body>

</html>";
// Return the HTML
    await context.Response.WriteAsync(html);
});

// Add the GraphQL query
app.MapGet("/graphql", async context =>
{
    var locationId = context.Request.Query["locationId"];
    if (string.IsNullOrEmpty(locationId))
    {
        await context.Response.WriteAsync("Location ID is required.");
        return;
    }

// Set up the GraphQL client - your endpoint goes here
    var graphQLOptions = new GraphQLHttpClientOptions
    {
        EndPoint = new Uri("ReplaceWithYourGraphQLEndpointAddress"),
        MediaType = "application/json",
    };

// Set up the client with headers, use the token
    var graphQLClient = new GraphQLHttpClient(graphQLOptions, new NewtonsoftJsonSerializer());
    graphQLClient.HttpClient.DefaultRequestHeaders.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token.Token);
    graphQLClient.Options.IsValidResponseToDeserialize = response => response.IsSuccessStatusCode;
// Set up the GraphQL query - you could make this a variable if you wanted to
    var query = new GraphQLHttpRequest
    {
        Query = $@"query {{ vProductsbySuppliers(filter: {{ SupplierLocationID: {{ eq: {locationId} }} }}) 
          {{ items 
              {{ CompanyName SupplierLocationID ProductCount }} 
          }} 
        }}"
    };
// Send the query
    var graphQLResponse = await graphQLClient.SendQueryAsync<dynamic>(query);
    var items = graphQLResponse.Data.vProductsbySuppliers.items;

// Build the HTML response from the query
    var html = @"
    <!DOCTYPE html>
    <html>
    <head>
        <link rel='stylesheet' href='https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css'>
    </head>
    <body>
        <div class='container'>
           <img src='./contoso.png' alt='Contoso Logo' width='200'>
            <h2>Product Count by Suppliers</h2>
            <table class='table table-striped'>
                <thead>
                    <tr>
                        <th>Company Name</th>
                        <th>Supplier Location ID</th>
                        <th>Product Count</th>
                    </tr>
                </thead>
                <tbody>";

    foreach (var item in items)
    {
        html += $"<tr><td>{item.CompanyName}</td><td>{item.SupplierLocationID}</td><td>{item.ProductCount}</td></tr>";
    }

    html += @"
                </tbody>
            </table>
        </div>
    </body>
    </html>";
// Return the HTML
    await context.Response.WriteAsync(html);
});
// Run the application
app.Run();
```

5. **Copy Graphics File**:
   - Optionally, you place any PNG graphic with the name *Contoso.png* image file in this solution in the following directory:
     ```bash
     /wwwroot
     ```
     

6. **Run the application**:
   - In the terminal or command prompt, run the following command to start the application:
     ```bash
     dotnet run
     ```

6. **Access the application**:
   - Right-click on the line in your command window that looks similar to `http://localhost:5261/graphql` to see the output of your GraphQL query.

### Code of Conduct
This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

### License
Microsoft and any contributors grant you a license to the Microsoft documentation and other content in this repository under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/legalcode), see [the LICENSE file](https://github.com/MicrosoftDocs/mslearn-tailspin-spacegame-web/blob/master/LICENSE), and grant you a license to any code in the repository under [the MIT License](https://opensource.org/licenses/MIT), see the [LICENSE-CODE file](https://github.com/MicrosoftDocs/mslearn-tailspin-spacegame-web/blob/master/LICENSE-CODE).

Microsoft, Windows, Microsoft Azure and/or other Microsoft products and services referenced in the documentation
may be either trademarks or registered trademarks of Microsoft in the United States and/or other countries.
The licenses for this project do not grant you rights to use any Microsoft names, logos, or trademarks.
Microsoft's general trademark guidelines can be found at http://go.microsoft.com/fwlink/?LinkID=254653.

Privacy information can be found at https://privacy.microsoft.com/en-us/

Microsoft and any contributors reserve all other rights, whether under their respective copyrights, patents,
or trademarks, whether by implication, estoppel or otherwise.

### Questions
Email questions to: sqlserversamples@microsoft.com

