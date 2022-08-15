# ForYou | Laghrour Holdings
 
 Please consult the global documentation before looking at the specifics:
 https://github.com/laghrourholdings/ForYou
 
 You'll find:
  - Base architecture
  - Separation of concerns
  - Interservices communications
      - Via the EventBus
      - Via HTTP
  - Logging Standards
  - Other essential informations
 

# ForYou | Laghrour Holdings

ForYou is a new mobile services exchange application. It's summer and you just don't feel like mowing the lawn today, don't you just wish you could be able to  briefly hire a worry-free neighbour to do that chore for you in one click? ForYou is there for you (pun more or less intended).

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
| IUser | Auth Service |
| IEnterpriseUser | Auth Service|
| IRegularUser | Auth Service |
| IEmployeeUser | Auth Service |
| IInstitution | Auth Service |
| IBillingForm | Billing Service|
| IVerificationForm | Billing Service|
| IObject | Internal Service |
| IMedia | Internal Service |
| ILogHandle | Log Service|
| ILogMessage | Log Service |
| IPaymentProcessor | Payment Service |
## Back to basics
