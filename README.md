# Microservice runtime draft

## Overview
Most microservices implementations rely on REST endpoints for inter-process communications. I'm working through a concept in which REST endpoint are not used for inter-process communication, this is a most to do when propagating writes operations across microservices. A write operation that spans multiple microservices is a distributed transaction that it should not be implemented using REST endpoint. 

Let’s said, you have an orchestration layer or a microservice that orchestrate operations across microservices. Let's say, that to create an employee the architecture uses an orchestration microservice that invoke the On-Boarding microservice, upon valid respond it invoke the PayRoll microservice. In this scenario, we have a distributed transaction, there are solutions to deal with failure, like using compensation patterns to revert changes done. 

The architecture gets complicated with handling network partition errors to remains consistent; it resembles the n-tier architecture rather than microservice architecture that focus on domain functionalities. Microservices usually are not restricted like the n-tier architecture in which a tier layer can only talk to a certain tier(s) (strict or relaxed), etc., microservice are cohesive and solve a domain problem, they emit events, and they can be invoked by any other microservices. Microservice are protected by creating network controls to limit the access to their interfaces.

Read operations: When a microservice need data from another service to compute a request it's typical to use a REST endpoint to retrieve the data to compute a response. 

There are alternatives to this pattern:
* Collocate data on each microservice given the cost of collocating data is acceptable. Why? collocating data on each microservice increase their autonomy, availability, and performance. This can be done in those services where the availability and performance are a must in your microservices implementations rather than a hammer for every read operation. 
* Create a group of service that are read-only services, these services subscribe to the domain emitted events to create documents that match the read operation required by the client applications. These group of services will reduce the amount of data duplication; one cons, is that they have an old snapshot of the data. The data they provide is behind according to the latency between emitted events and their computation.

What I’m working on?

The goal of this concept is to have a practical implementation in .Net core that include an out of the box implementation for raising and consuming events with security controls to protect confidential data in transit. It would be expensive to build the code that handle the requirements of saving, sending, and receiving events on each microservice. Implementing a runtime service that implement these capabilities is what this concept is intended to provide you.

This architecture use a local database transaction to save the entities and events associated with the entities change. Each microservice database have a table for outgoing events added by the runtime entity framework context. It breaks the distributed transaction to save data in a database and dispatch events to the service bus hub. 

## Architecture

### High level requirements
* Each microservice owns the write operation of a graph data (it could be a flat document or a document with nested documents). 
* Each microservice can reach to other microservice for read only data. These microservices do not use REST endpoint to update other microservices about a change. These microservices emit events that are bound to a transaction that modified the graph data the microservices owns.  
* Events are double encrypted with AES symmetric algorithm and RSA asymmetric algorithm to protect the message and AES keys in transit. 
* Events are configured to be send to a queue or topic and configured to be read from a queue or topic. The event itself contain the queue or topic to which it was sent. The receiver microservice would valid that the message queue name or topic is valid by checking local configuration with the message queue names. Why? This provide guarantee that the message was not moved across different queue or topic in the hub, it enable to enforce an architecture in which message are routed as configured in the microservices. Its like an TCP/IP handshake, it guarantees that the sender and receiver agreement in which queue and topic they would be sending and receiving message is enforce by the microservices.
* Each microservice have his own Azure Key Vault instance with a certificate used to sign the event message sent.
* The microservice runtime (is a custom implementation) have an Azure Key Vault instance with a certificate used to encrypt and decrypt the AES Keys used to encrypt the messages. Each microservice will invoke a local host endpoint to decrypt the AES Keys.
* AES encryption: A message is flatted into a key value pair. The key is the property of the object, and the value is the property value. The key and the property value are double encrypted using random AES keys. A random AES key is generated per property name and property value to encrypt. 30 AES key are generated by the runtime at startup to double encrypt the property name and property value encrypted. The collections of keys are encrypted by the runtime using the certificate in the azure key vault of the runtime.
* Each microservice would use a unique service principal from Azure AD to access azure native service. The runtime uses a unique service principal from Azure AD to access the runtime services. These allows to segregate the permission required by the microservice vs the runtime.

### Implementation details
* AesCcm algorithm is used for symmetric encryption.
* RSA asymmetric algorithm is used to protected AES keys.
* An Azure Key Vault is assigned to each microservice.
* A service principal account is created per microservice.
* A service principal account is created for the runtime.
* Service principal password are saved in Azure Key Vault as secret.
* Azure Key Vault secrets are read through Azure App Configuration. 
* The runtime creates a generic host with a hosted service to setup Azure services and verify their setup.
* The runtime loads the microservice IHost application to registered additional services dependencies like ServiceContext.
* The runtime registers the hosted service responsible for sending and receiving events into the microservice generic host.
* The runtime creates the environment variables with the App Configuration key.
* The runtime supports multiple microservices assemblies on development environment. It makes easy to run multiple microservices locally.

### Concept 
* Microservices are bound to a subdomain transaction.
* Each microservice use a persistence object to save entities and events.
* A runtime generic host read events and dispatch them into Azure Service Bus Queue.
* Write service to service communication is done via Azure Service Bus Queue.


![alt_text][concept]

[concept]: https://learningruntimestor.blob.core.windows.net/runtimedocumentation/MicroserviceConceptAndDb120by120.png "Microservice concept"


## Project structure

### Assemblies, namespaces and classes

![alt_text][assemblies]

[assemblies]: https://learningruntimestor.blob.core.windows.net/runtimedocumentation/ProjectStructure.png "Project assemblies"




### Configuration
* Azure App Configuration will be used for all development life cycle environments.

#### Azure Service Bus
* Each microservice load at startup the configuration of queue to send messages.
* Service bus instance are created and stored in a concurrent collection during startup.

#### Queue
* Each sender queue have an acknowledge queue, if acknowledgement is required from receiver. The ackqueue is part of the configuration to validate that the reply a message is send to a queue agrees during setup. This control enable to secure that message are routed through queues as intended during the architecture cycle. 
* Each message have a DecryptScope that is required to decrypt message using the service runtime local crypto endpoint (it would be explained later)
* Each queue is setup in a service endpoint. Receiver microserivce must validate the message intransit data (defined later) like the endpoint match the endpoint from where the message was received. This would provide gurantee that message are send to the queue and received from the queue that was intended too.   

```
"Sender": [
          {
            "ConfigType": "Sender",
            "ConnStr": "Endpoint=sb://leraningyusbel.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SharedAccessKeyValueGoesHere",
            "Identifier": "ServiceBusClientEmployeeMessages",
            "MessageInTransitOptions": [
              {
                "MsgQueueName": "EmployeeAdded",
                "AckQueueName" :  "AckEmployeeAdded",
                "MsgDecryptScope": "EmployeeAdded.Decrypt",
                "MsgQueueEndpoint": "leraningyusbel.servicebus.windows.net"
              },
              {
                "MsgQueueName": "EmployeeUpdated",
                "AckQueueName" : "",
                "MsgDecryptScope": "EmployeeUpdated.Decrypt",
                "MsgQueueEndpoint": "leraningyusbel.servicebus.windows.net"
              },
              {
                "MsgQueueName": "EmployeeDeleted",
                "AckQueueName" :  "",
                "MsgDecryptScope": "EmployeeDeleted.Decrypt",
                "MsgQueueEndpoint": "leraningyusbel.servicebus.windows.net"
              }
            ]
          }
        ]
```
#### Azure service bus receiver
* Each microservice load the receiver configuration and add a service bus processor into a concurrent collection.

```
"Receiver": [
          {
            "ConfigType": "Receiver",
            "ConnStr": "Endpoint=sb://leraningyusbel.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SharedAccessKeyGoesHere",
            "Identifier": "ServiceBusClientPayRollMessages",
            "MessageInTransitOptions": [
              {
                "MsgQueueName": "PayRollAdded",
                "AckQueueName":  "",
                "MsgDecryptScope": "PayRollAdded.Decrypt",
                "MsgQueueEndpoint": "leraningyusbel.servicebus.windows.net"
              },
              {
                "MsgQueueName": "PayRollUpdated",
                "AckQueueName":  "",
                "MsgDecryptScope": "PayRollUpdated.Decrypt",
                "MsgQueueEndpoint": "leraningyusbel.servicebus.windows.net"
              },
              {
                "MsgQueueName": "PayRollDeleted",
                "AckQueueName": "",
                "MsgDecryptScope": "PayRollDeleted.Decrypt",
                "MsgQueueEndpoint": "leraningyusbel.servicebus.windows.net"
              }
            ]
          }
        ]
```

#### Save entity and event
* Save entity and event in a transaction. 
```
public async Task<bool> SaveWithEvent(OutgoingEventEntity eventEntity, CancellationToken token)
        {
            _dbContext.Add(eventEntity);
            var strategy = _dbContext.Database.CreateExecutionStrategy();

            if (token.IsCancellationRequested)
                return false;

            await strategy.ExecuteInTransactionAsync(
                operation: async () =>
                    {
                        await _dbContext.SaveChangesAsync(acceptAllChangesOnSuccess: false).ConfigureAwait(false);
                    },
                verifySucceeded: async () =>
                    {
                        return await _dbContext.Set<OutgoingEventEntity>().AsNoTracking().AnyAsync(ee => ee.Id == eventEntity.Id).ConfigureAwait(false);
                    }).ConfigureAwait(false);

            _dbContext.ChangeTracker.AcceptAllChanges();
            if (OnSave != null)
            {
                OnSave(this, new ExternalMessageEventArgs()
                {
                    ExternalMessage = eventEntity!.ConvertToExternalMessage()
                });
            }
            return true;
        }
```

# Runtime
A microservice assembly is loaded and invoked by a service host runtime. The service host runtime do:
* Set environment variables. 
* Setup service dependencies in Azure, like: registering the service in active directory, manage secret, key and policies in azure key vault, load configuration values from Azure App Configuration.
* Verify service dependecies status.

![alt_text][runtime]

[runtime]: https://learningruntimestor.blob.core.windows.net/runtimedocumentation/RuntimeSample.png "Runtime activity diagram"

### Load microservice assembly
* Microservice assembly location are stored on Azure App Configuration or can be passed as a command argument.
* Runtime is nothing more that a generic host application.
* Runtime excecute a hosted service to setup Azure service and verify their configuration status on a schedule.

### Sample code that load the microservice assembly and execute the run async
```
private static bool LaunchCoreHostApp(string[] args, 
            ServiceHostBuilderOption service, 
            ServiceRegistration serviceReg,
            CancellationToken runtimeToken) 
        {
            var logger = GetLogger();
            if (!File.Exists(service.Service))
                return false;
            var hostService = AppDomain.CurrentDomain.CreateInstanceFromAndUnwrap(service.Service, service.HostBuilder);

            if (hostService is ICoreHostBuilder serviceHost)
            {
                var serviceHostBuilder = serviceHost.GetHostBuilder(args);
                try
                {
                    if (serviceReg == null)
                        throw new InvalidOperationException();
                    _ = Task.Run(async () => 
                    {
                        await HostService.Run(serviceHostBuilder, serviceReg, args, runtimeToken)
                                        .ConfigureAwait(false);
                    }, runtimeToken);
                    return true;
                }
                catch (Exception e)
                {
                    e.LogException(logger.LogCritical);
                }
            }
            return false;
        }
```


## Create employee activity diagram
* Once an employee is create a new event EmployeeAdded is raised.
* PayRoll microservice is subscribed to EmployeeAdded queue. 
* PayRoll create a payroll entry for the new created employee and send an acknowledge message to employee service.

### Activity diagram

![alt_text][createemployee]

[createemployee]: https://learningruntimestor.blob.core.windows.net/runtimedocumentation/CreateEmployeeActivityDiagram.png "Employee Added"
