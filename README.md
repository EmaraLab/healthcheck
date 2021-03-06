# healthcheck-fwk
A simple healthcheck framework for checking the health of an application and its dependent services asynchronously through messaging. 
It is divided into two modules
  * healthcheck-server
  * healthcheck-client

### healthcheck-server
A web application for pinging the application(s) and receving HealthCheckEvents. 
#### High Level Architecture

![High Level Architecture](https://github.com/EmaraLab/healthcheck/blob/master/health_check_main.png)

###### Structure of the HealthCheckEvent

```javascript
{"id":"760a99d0-f154-4395-a623-305e8e3ca40c", //Unique ID
"application":"VC", // Application Name
"service":"vision_app_database", // A Service of the application
"status":"OK", // status [OK|NOT_OK|TIMED_OUT]
"error":null, // error in case status was NOT_OK
"timestamp":1472705410450, // time at which the event was created
"latency":1, // time taken to ping the service
"healthCheckPing":null // the request against which this event is sent. It is Optional.
}
```


It can be configured using the following properties
```
  #Queue for pushing the health check request. It will be appended by the application ID (VC/IQC etc.)
  hc.request.queue.suffix=HC.PING.QUEUE

  #For listening for the replies of the health check received from the application
  hc.reply.queue=HC.EVENT.REPLY.QUEUE

  #For relaying the replies on to the web socket queue
  hc.websocket.queue=/topic/hc-web-queue

  #Time limit for the client for a particular service
  hc.client.timeout.secs=2
```
To run the server, from the command line, simply go to its folder ```healthcheck-server``` and type ```mvn spring-boot:run``` and the service will be availble at (http://localhost:8080/index.html). In case of any build issues, go the root folder of the project and type ``` mvn clean install -DskipTests``` 

### healthcheck-client
The client side library that any application will use that is monitored by the framework. For clients to use this, they have to do the following

#### Implement the HealthCheck interface
```
public class MockHealthCheck implements HealthCheck {

  private String service;

  public MockHealthCheck(String service) {
    this.service = service;
  }

  @Override
  public HealthCheckEvent perform(HealthCheckContext context) {

    HealthCheckEventBuilder builder = new HealthCheckEventBuilder(context);

    return builder
        .service(service)
        .timestamp(new Date()).build();

  }

}
````
#### Define the HealthCheck Executor
```
  @Bean
  public HealthCheckExecutor healthCheckExecutor(HealthCheckEventPublisher eventPublisher){

    HealthCheck[] healthChecks = new HealthCheck[]{new MockHealthCheck("vision_app")
        , new MockHealthCheck("vision_est")};

    HealthCheckExecutor executor = new HealthCheckExecutor(healthChecks, eventPublisher);

    return executor;

  }
```  

#### Implement the Message Listener

```
@Component
public class MockHealthCheckPingListener implements MessageListener {


  @Autowired
  HealthCheckEventPublisher eventPublisher;

  @Autowired
  Serializer<HealthCheckPing> serializer;

  @Autowired
  HealthCheckExecutor executor;

  @Override
  public void onMessage(Message messageIn) {

    try {
      String message = ((TextMessage) messageIn).getText();
      System.out.println("Received <" + message + ">");

      HealthCheckPing ping = serializer.deserialize(message, HealthCheckPing.class);

      executor.performAndSubmit(ping, 1L);

    } catch (JMSException e) {
      e.printStackTrace();
    }

  }
}
```

For more details visit the ```ae.emaratech.devops.tools.healthcheck.client.mock``` package in the source code.

