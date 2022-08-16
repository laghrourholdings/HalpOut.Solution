# ForYou 

ForYou is a new mobile services exchange application. It's summer and you just don't feel like mowing the lawn today, don't you just wish you could be able to  briefly hire a worry-free neighbour to do that chore for you in one click?

Informations/features/misc :
- Quickly hire other members at your proximity in just one click.
- Share your job offer to your favorite social media apps (Sign-in with OAuth2).
- Free to use, no fees on deposits and no fees on withdrawals: a trade comission of 10% is applied to every completed transaction.
- The Consumer always sees the net job price offerring, while the Offerer sees the cut Laghrour Holdings is taking.
- Members have incentive to refer other users with the following benefits:
	- Referer receives 25% off the comission Laghrour Holdings makes on offers, reducing the trade comissison to 7.5%
	- Referee receives 25% off the comission Laghrour Holdings makes on offers, reducing the trade comissison to 7.5%
	- Referer can have no more than 2 referees 	 
- Members have the opportunity to upgrade to a 9.99$/month Partner Membership:
	- Eliminates the trade commission
	- Grants web access for comprehensive reports and dashboards
	- Special access to pre-release and app customization features
	- Partners can refer friends and receive 50% of the comission Laghror Holdings makes of the referee as long as they remain Partners
	- Partners can have no more for 4 referees.
- Local business can also send offers and receive efficient and ephemeral workforce.
- Larger businesses and enteprises have the option to register SSO logins.
- Every member must present at least one primary identity that will be checked by a soft-credit check as to not affect credit scores.
- Members below 18 years old must present parental approval before using any of our service.

## The Stack
ForYou is a modern mobile application that uses modern bleeding-edge technologies to bring its members blazing fast performance on Web, IOS and Android. As very high availability is ForYou's central focus, ForYou uses future-proof microservices architecture with asychronous messaging using IBM RabbitMq and Azure Service Bus to prevent solution-wide outages in case of an internal server failure. Server and client services run on the latest long-term support of the Microsoft .Net Core framework; .Net 6, along with C# 10, bring a stable and reliable working foundation to build upon WebApis and Clients. Clients will run on top of the latest UI/UX CLR-Compatible framework with .NET MAUI, bringing modern development patterns such as Dependency Injection and IoC (inversion of control) into mobile applications. 

Using the same technologies throughout the whole stack allows ForYou to kill code duplication by providing the CommonLibrary, CommonLibrary.AspNetCore and CommonLibrary.ClientServices to each of the elligible services, thus standarizing the contracts and the data transfer objects (DTOs). Blazor WASM will be used as ForYou's next generation web framework, killing the need to use JavaScript and its derivatives. It will allow .Net interfaces to be enforced throughout the processes, allowing code reusability and zeroing potential for legacy code. Longer initial web loading times will be compensated by the performance gains when running natively.

For its deployment into Microsoft Azure, ForYou uses the latest container and CI/CD pipelines technologies allowing for high-scalability and shorter development cycles. Kubernetes, along with extensive use of Docker and Azure CI/CD allows time wasted with debugging low-level errors to be used for bringing new features for the members. Lastly, they considerably reduce the risk for deployment failures and runtime production issues.

## Base Architecture
### Figure 1 : Global services architecture
![Image](https://user-images.githubusercontent.com/35415121/184698805-7c478a95-a17d-4f59-96c4-af9b37282117.PNG)
### Figure 2 : Internal objects layout
![Image](https://user-images.githubusercontent.com/35415121/184696934-ab1af990-054c-4315-82e2-b60692fd910d.PNG)
### Figure 3 : Objects ownership
 
| Business object interface  | Discretionnary |
| ------ | ------ |
| ILogHandle | Log Service|
| ILogMessage | Log Service |
| IObject | Internal Service |
| IMedia | Internal Service |
| IUser | Auth Service |
| IEnterpriseUser | Auth Service|
| IRegularUser | Auth Service |
| IEmployeeUser | Auth Service |
| IInstitution | Auth Service |
| IBillingForm | Billing Service|
| IVerificationForm | Billing Service|
| IPaymentProcessor | Payment Service |

## Back to Basics
IObject is the central business object interface (BOI) holding the essential information required by all subsequent BOIs for their normal operations. IObject provides first inheritors access to an object which has property such as CreationDate, Deleted?, Suspended?, LogHandleId, Descriptor. The purpose of IObject is to have a concrete BOI that is not dependant on any service and that can also allow ForYou to maintain business even in the event of an outage (through its Descriptor property). Although managed by the Internal Service, IObject is declared in the CommonLibrary framework and is ready to be used by any referencer and by any service. LogHandleId is the reference to all the log data related to the object, which means that even if a derivative of an IObject had to transform, it can do so without losing access to logs and other core properties:

├── IObject
  -> LogHandleId: 0xDEADBEEF
  
│   ├── IUser

Can easily become:

├── IObject
  -> LogHandleId: 0xDEADBEEF
  
│   ├── IMedia

Of course this example will not be seen in practice, but it shows that the architecture gives the developer great flexiblity on the domain model.
The CommonLibrary also provisions the microservices with a one line configurator:

```csharp
//In startup.cs
builder.Services.AddCommonLibrary(
			builder.Configuration,
			builder.Logging,
			logger,
			myAllowSpecificOrigins);

		AddCommonLibrary(
			this IServiceCollection services,
        		IConfiguration configuration,
			ILoggingBuilder logging,
			LoggerConfiguration loggerConfiguration,
			string originName)
```

## Backend interservices communication
### Using the EventBus (asynchronous messaging)
Every backend service is provided with the `IServiceBus` interfaces package which include `IServiceMessage` and `IServiceRequest<TSubject>`. The `IServiceBus` allows for inter-services communications allowing for flexible request types. The protocol, at its bear minimum, requires you to supply a `Descriptor` and a `Contract` along every request made to another service via `IServiceBusMessage`. The `Descriptor` is a `string` description of the current request and its goals, it is mainly requiered for logging purposes. The `Contract` is the request template, which include the type of request and the type of payload, reflected on a `string` format. The CommonLibrary also provides MassTransit.RabbitMq which allows developers to concretise requests:

#### Example: Creating an IObject
In this example, we will go over the full process which will cover most cases scenarios. These sample codes will show how the Gateway Service can call the Internal Service, and get as a response an IObject to which a LogHandleId will be given by the Log Service.

##### CommonLibrary.AspNetCore CreateObjectContract.cs
The call chain to create an IObject are made in this exact order:
```csharp
//Gateway to Internal
public record CreateObject(IServiceBusMessage Payload);

//Internal to Log
public record LogCreateObject(ServiceBusLogContext<ServiceBusRequest<IIObject>> Payload);

//Log to Internal
public record LogCreateObjectResponse(ServiceBusMessageReponse<IIObject> Payload);

//Internal to Gateway
public record CreateObjectResponse(IServiceBusRequest<IIObject> Payload);

/*
Note: 	Using IServiceBusMessageResponse<IIObject> instead of IServiceBusRequest<IIObject>
	in CreateObjectResponse would've been more appropriate if we follow exactly the protocol.
	For this specific case, it was not necessary for the Gateway Service 
	to have access to the initial requests. 
*/
```

##### GatewayService.Repositories ObjectRepository.cs 

```csharp
/*
Gateway to Internal Service Contract:
public record CreateObject(IServiceBusMessage Payload);
*/

//CreateObject()
var request = new ServiceBusMessage
{
	Descriptor = ServiceSettings.GetMessage($"Requesting object creation"),
	Contract = nameof(CreateObject),
};
_logger.Information("The following request was made: {@Request}", request);
await _publishEndpoint.Publish(new CreateObject(request));
```

##### InternalService.Slots CreateObjectConsumer.cs
On the other end, the service who accepts the contracts, or the consumer, will be responsible for dealing with the request by notifying the Log Service and by providing a response to the caller:

```csharp
public class CreateObjectConsumer : IConsumer<CreateObject>
{
    ...
	public CreateObjectConsumer(...) {...}
	
	//Context contains the payload declared in the contract
    public async Task Consume(ConsumeContext<CreateObject> context)
    {
        var requestedGuid = Guid.NewGuid();
        IIObject obj = new IIObject
        {
            Id = requestedGuid,
            CreationDate = DateTimeOffset.Now,
            IsDeleted = false,
            DeletedDate = default,
            IsSuspended = false,
            SuspendedDate = default,
            LogHandleId = default,
            Descriptor = null
        };
		 _logger.Information("Received request: {@Context}",context);

		// [1/2] If the response object requires a LogHandleId, it must first ask the LogService
		// for a LogHandleId before returing to the original caller
        var request = new ServiceBusRequest<IIObject>
        {
            Subject = obj,
            Descriptor = $"Requesting LogHandleId for object {requestedGuid}"
        };
		
		// [2/2] _logger.<LogLevel>ToBusLog() is in charge of calling the Log Service with the provided 
		// contract. The LogContext is required by the Log Service for awareness purposes.
        var logContext = request.GetLogContext(_configuration, LogLevel.Information);
        _logger.GeneralToBusLog(
            logContext,
            $"Object created: {requestedGuid}",
            _publishEndpoint, new LogCreateObject(logContext));
			
        await _objectRepository.UpdateOrCreateAsync(obj);
    }
}
```
##### LogService.Slots LogCreateObjectConsumer.cs
The Log Service now receives the task to create an `ILogHandle` to manage logs related to that BOI and to return a response containing the BOI but with a non-null `LogHandleId`:
```csharp
public class LogObjectCreateConsumer : IConsumer<LogCreateObject>
{
    ...
    public LogObjectCreateConsumer(...) {...}
    
    public async Task Consume(ConsumeContext<LogCreateObject> context)
    {
		//Retreive the payload
        var logContext = context.Message.Payload;
		
		//Create a message that will be added to the newly created IlogHandle 
        var messages = new List<LogMessage>();
        messages.Add(new LogMessage
        {
            Id = 0,
            Message = ServiceSettings.GetMessage($"LogCreateObject: {logContext.Message.Subject}"),
            CreationDate = DateTimeOffset.Now,
            Severity = logContext.Severity
        });
		
		//Create the IlogHandle and attach the initial message
        CommonLibrary.Logging.LogHandle logHandle = new CommonLibrary.Logging.LogHandle
        {
            Id = Guid.NewGuid(),
            ObjectId = logContext.Message.Subject.Id,
            Messages = messages
        };
		
		//Bind the LogHandleId to the subject passed on with the payload
        var obj = logContext.Message.Subject;
        obj.LogHandleId = logHandle.Id;
        obj.LogHandle = logHandle;
		
		//Callback the Internal Service with its object linked with an ILogHandle
        var response = new ServiceBusMessageReponse<IIObject>
        {
            Contract = nameof(LogCreateObjectResponse),
            InitialRequest = logContext.Message,
            Subject = obj,
            Descriptor = ServiceSettings.GetMessage($"LogHandleID: {{logHandle.Id}} assigned to : ${{logContext.Message.Subject}}")
        };
        _logger.Information("{ResponeDescriptor}", response.Descriptor);
        await _handleRepository.CreateAsync(logHandle);
        await context.RespondAsync(new LogCreateObjectResponse(response));
    }
}
```

##### InternalService.Slots LogCreateObjectResponseConsumer.cs
The Internal Service is now ready to callback the Gateway Service with a new, ready to use, IObject.
```csharp
public class LogCreateObjectResponseConsumer : IConsumer<LogCreateObjectResponse>
{
	...
    public LogCreateObjectResponseConsumer(...) {...}

    public async Task Consume(ConsumeContext<LogCreateObjectResponse> context)
    {
        var logContext = context.Message.Payload;
        var obj = logContext.Subject;
        var response = new IiObjectServiceBusMessageResponse
        {
            Subject = obj,
            Descriptor = $"Creation for object {obj.Id} completed with success.",
            InitialRequest = logContext.InitialRequest,
            Contract = nameof(CreateObjectResponse),
            StatusCode = HttpStatusCode.OK
        };
        _logger.Information("{@Descriptor} | Object with ID: {@ObjectID} assigned LogHandleId: {@LogHandleID}",response.Descriptor, obj.Id,obj.LogHandleId); 
        await context.RespondAsync(new CreateObjectResponse(response));
    }
}
```
##### GatewayService.Slots CreateObjectResponseConsumer.cs
The Gateway Service is now ready to use its new IObject:
```csharp
public class CreateObjectResponseConsumer : IConsumer<CreateObjectResponse>
{
    ...
    public CreateObjectResponseConsumer(...) {...}
    
    public async Task Consume(ConsumeContext<CreateObjectResponse> context)
    {
        var message = context.Message.Payload;
        _logger.Information("Creation for: {@message.Subject} COMPLETED",message.Subject);
        await Task.CompletedTask;
    }
}
```
### Using the HttpClient (synchronous messaging)
The CommonLibrary also provides the Flurl library and a retry policy in case of a request failure by server error for simple and reliable requests:
##### GatewayService.Repositories ObjectRepository.cs
```csharp
public async Task<IEnumerable<IIObject>> GetAllAsync()
    {
	//Equivalent of calling: https://localhost:4042/api/v1/Objects
        return await ServicesSettings.InternalServiceDevURL
            .AppendPathSegment("objects")
            .GetJsonAsync<IEnumerable<IIObject>>();
    }
```
