# Simple Transaction: Microservices sample architecture for .Net Core Application
### .Net Core 2.2 Sample With C#.Net, EF and SQL Server
## Introduction
This is a .Net Core sample application and an example of how to build and implement a microservices based back-end system for a simple automated banking feature like Balance, Deposit, Withdraw in ASP.NET Core Web API with C#.Net, Entity Framework and SQL Server. 

## Application Architecture

The sample application is build based on the microservices architecture. There are serveral advantages in building a application using Microservices architecture like Services can be developed, deployed and scaled independently.The below diagram shows the high level design of Back-end architecture.

- **Identity Microservice** - Authenticates user based on username, password and issues a JWT Bearer token which contains Claims-based identity information in it.
- **Transaction Microservice** - Handles account transactions like Get balance, deposit, withdraw
- **API Gateway** - Acts as a center point of entry to the back-end application, Provides data aggregation and communication path to microservices.



## Security : JWT Token based Authentication
JWT Token based authentication is implementated to secure the WebApi services. **Identity Microservice** acts as a Auth server and issues a valid token after validating the user credentitals. The API Gateway sends the token to the client. The client app uses the token for the subsequent request.


## Development Environment

- [.Net Core 2.2 SDK](https://dotnet.microsoft.com/download)
- [Visual Studio .Net 2017](https://visualstudio.microsoft.com/downloads/)
- [SQL Server Management Studio 17.9.1](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-2017)

## Technologies
- C#.NET
- ASP.NET WEB API Core
- SQL Server

## Opensource Tools Used
- Automapper (For object-to-object mapping)
- Entity Framework Core (For Data Access)
- Swashbucke (For API Documentation)
- XUnit (For Unit test case)
- Ocelot (For API Gateway Aggregation)



## WebApi Endpoints
The application has four API endpoints configured in the API Gateway to demonstrate the features with token based security options enabled. These routes are exposed to the client app to cosume the back-end services. 

### End-points configured and accessible through API Gateway

1. Route: **"/user/authenticate"** [HttpPost] - To authenticate user and issue a token
2. Route: **"/account/balance"** [HttpGet] - To retrieve account balance.
3. Route: **"/account/deposit"** [HttpPost] - To deposit amount in an account.
4. Route: **"/account/withdraw"** [HttpPost] - To withdraw amount from an account.

### End-points implemented at the Microservice level

1. Route: **"/api/user/authenticate"** [HttpPost]- To authenticate user and issue a token
2. Route: **"/api/account/balance"** [HttpGet]- To retrieve account balance.
3. Route: **"/api/account/deposit"** [HttpPost]- To deposit amount in an account.
4. Route: **"/api/account/withdraw"** [HttpPost]- To withdraw amount from an account.

## Solution Structure

- **Identity.WebApi**
    - Handles the authentication part using username, password as input parameter and issues a JWT Bearer token with Claims-Identity info in it.
- **Transaction.WebApi** 
    - Supports three http methods 'Balance', 'Deposit' and 'Withdraw'. Receives http request for these methods.
    - Handles exception through a middleware
    - Reads Identity information from the Authorization Header which contains the Bearer token
    - Calls the appropriate function in the 'Transaction' framework
    - Returns the transaction response result back to the client
- **Transaction.Framework** 
    - Defines the interface for the repository (data) layer and service (business) layer
    - Defines the domain model (Business Objects) and Entity Model (Data Model)
    - Defines the business exceptions and domain model validation
    - Defines the required data types for the framework 'Struct', 'Enum', 'Consants'
    - Implements the business logic to perform the required account transactions
    - Implements the data logic to read and update the data from and to the SQL database
    - Performs the task of mapping the domain model to entity model and vice versa 
    - Handles the db update concurrency conflict
    - Registers the Interfaces and its Implementation in to Service Collection through dependency injection
- **Gateway.WebApi** 
    - Validates the incoming Http request by checking for authorized JWT token in it.
    - Reroute the Http request to a downstream service.
- **SimpleBanking.ConsoleApp**
    - A console client app that connects to Api Gateway, can be used to login with username, password and perform transactions like 'Balance', 'Deposit' and 'Withdraw' against a account.

## Exception Handling
A Middleware is written to handle the exceptions and it is registered in the startup to run as part of http request. Every http request, passes through this exception handling middleware and then executes the Web API controller action method. 

* If the action method is successfull then the success response is send back to the client. 
* If any exception is thrown by the action method, then the exception is caught and handled by the Middleware and appropriate response is sent back to the client.

public async Task InvokeAsync(HttpContext context, RequestDelegate next)
{
    try
    {
        await next(context);
    }
    catch (Exception ex)
    {
        var message = CreateMessage(context, ex);
        _logger.LogError(message, ex);

        await HandleExceptionAsync(context, ex);
    }
}

## Db Concurrency Handling

Db concurrency is related to a conflict when multiple transactions trying to update the same data in the database at the same time. In the below diagram, if you see that Transaction 1 and Transaction 2 are against the same account, one trying to deposit amount into account and the other system tring to Withdraw amount from the account at the same time. The framework contains two logical layers, one handles the Business Logic and the other handles the Data logic. 

When a data is read from the DB and when business logic is applied to the data, at this context, there will be three different states for the values relating to the same record.

- **Database values** are the values currently stored in the database.
- **Original values** are the values that were originally retrieved from the database
- **Current values** are the new values that application attempting to write to the database.

The state of the values in each of the transaction produces a conflict when the system attempts to save the changes and identifies using the concurrency token that the values being updated to the database are not the Original values that was read from the database and it throws DbUpdateConcurrencyException.

The approach to handle the concurrency conflict is:

1. Catch **DbUpdateConcurrencyException** during SaveChanges
2. Use **DbUpdateConcurrencyException.Entries** to prepare a new set of changes for the affected entities.
3. **Refresh the original values** of the concurrency token to reflect the current values in the database.
4. **Retry the process** until no conflicts occur.

```
while (!isSaved)
{
    try
    {
        await _dbContext.SaveChangesAsync();
        isSaved = true;
    }
    catch (DbUpdateConcurrencyException ex)
    {
        foreach (var entry in ex.Entries)
        {
            if (entry.Entity is AccountSummaryEntity)
            {
                var databaseValues = entry.GetDatabaseValues();

                if (databaseValues != null)
                {
                    entry.OriginalValues.SetValues(databaseValues);
                    CalculateNewBalance();

                    void CalculateNewBalance()
                    {
                        var balance = (decimal)entry.OriginalValues["Balance"];
                        var amount = accountTransactionEntity.Amount;

                        if (accountTransactionEntity.TransactionType == TransactionType.Deposit.ToString())
                        {
                            accountSummaryEntity.Balance =
                            balance += amount;
                        }
                        else if (accountTransactionEntity.TransactionType == TransactionType.Withdrawal.ToString())
                        {
                            if(amount > balance)
                                throw new InsufficientBalanceException();

                            accountSummaryEntity.Balance =
                            balance -= amount;
                        }
                    }
                }
                else
                {
                    throw new NotSupportedException();
                }
            }
        }
    }
}  
```


## Swagger: API Documentation

Swashbuckle Nuget package added to the "Transaction Microservice" and Swagger Middleware configured in the startup.cs for API documentation. when running the WebApi service, the swagger UI can be accessed through the swagger endpoint "/swagger".

```
public void ConfigureServices(IServiceCollection services)
{            
     services.AddSwaggerGen(c => {
        c.SwaggerDoc("v1", new Info { Title = "Simple Transaction Processing", Version = "v1" });
     });
}
```

```
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory log)
{           
     app.UseSwagger();
     app.UseSwaggerUI(c => {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Simple Transaction Processing v1");
     });           
}

