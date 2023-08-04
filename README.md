# Mate Finance

Mata Finance Metrics is a financial advisor app built to demonstrate the [Microservice Architecture Pattern](http://martinfowler.com/microservices/) using Spring Boot, Spring Cloud and Docker.

<br>

![](https://cloud.githubusercontent.com/assets/6069066/13864234/442d6faa-ecb9-11e5-9929-34a9539acde0.png)
![Piggy Metrics](https://cloud.githubusercontent.com/assets/6069066/13830155/572e7552-ebe4-11e5-918f-637a49dff9a2.gif)

## Functional services

The Project is decomposed into three core microservices. All of them are independently deployable applications organized around certain business domains.

<img width="880" alt="Functional services" src="https://cloud.githubusercontent.com/assets/6069066/13900465/730f2922-ee20-11e5-8df0-e7b51c668847.png">

#### Notes
- Each microservice has its own database, so there is no way to bypass API and access persistence data directly.
- MongoDB is used as a primary database for each of the services.
- All services are talking to each other via the Rest API

## Infrastructure
[Spring cloud](https://spring.io/projects/spring-cloud) provides powerful tools for developers to quickly implement common distributed systems patterns -
<img width="880" alt="Infrastructure services" src="https://cloud.githubusercontent.com/assets/6069066/13906840/365c0d94-eefa-11e5-90ad-9d74804ca412.png">
### Config service
[Spring Cloud Config](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html) is horizontally scalable centralized configuration service for the distributed systems. It uses a pluggable repository layer that currently supports local storage, Git, and Subversion.

In this project, we are going to use `native profile`, which simply loads config files from the local classpath. You can see `shared` directory in [Config service resources](https://github.com/sqshq/PiggyMetrics/tree/master/config/src/main/resources). Now, when Notification-service requests its configuration, Config service responses with `shared/notification-service.yml` and `shared/application.yml` (which is shared between all client applications).

##### Client side usage
Just build Spring Boot application with `spring-cloud-starter-config` dependency, autoconfiguration will do the rest.

Now you don't need any embedded properties in your application. Just provide `bootstrap.yml` with application name and Config service url:
```yml
spring:
  application:
    name: notification-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
```

### Auth service
Authorization responsibilities are extracted to a separate server, which grants [OAuth2 tokens](https://tools.ietf.org/html/rfc6749) for the backend resource services. Auth Server is used for user authorization as well as for secure machine-to-machine communication inside the perimeter.

In this project, I use [`Password credentials`](https://tools.ietf.org/html/rfc6749#section-4.3) grant type for users authorization (since it's used only by the UI) and [`Client Credentials`](https://tools.ietf.org/html/rfc6749#section-4.4) grant for service-to-service communciation.

``` java
@PreAuthorize("#oauth2.hasScope('server')")
@RequestMapping(value = "accounts/{name}", method = RequestMethod.GET)
public List<DataPoint> getStatisticsByAccountName(@PathVariable String name) {
	return statisticsService.findByAccountName(name);
}
```

### API Gateway
API Gateway is a single entry point into the system, used to handle requests and routing them to the appropriate backend service or by [aggregating results from a scatter-gather call](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html). Also, it can be used for authentication, insights, stress and canary testing, service migration, static response handling and active traffic management.

### Service Discovery

Service Discovery allows automatic detection of the network locations for all registered services. These locations might have dynamically assigned addresses due to auto-scaling, failures or upgrades.

### Load balancer, Circuit breaker and Http client

#### Ribbon
Ribbon is a client side load balancer which gives you a lot of control over the behaviour of HTTP and TCP clients. Compared to a traditional load balancer, there is no need in additional network hop - you can contact desired service directly.

#### Hystrix
Hystrix is the implementation of [Circuit Breaker Pattern](http://martinfowler.com/bliki/CircuitBreaker.html), which gives us a control over latency and network failures while communicating with other services. The main idea is to stop cascading failures in the distributed environment - that helps to fail fast and recover as soon as possible - important aspects of a fault-tolerant system that can self-heal.

#### Feign
Feign is a declarative Http client which seamlessly integrates with Ribbon and Hystrix. Actually, a single `spring-cloud-starter-feign` dependency and `@EnableFeignClients` annotation gives us a full set of tools, including Load balancer, Circuit Breaker and Http client with reasonable default configuration.
### Monitor dashboard

In this project configuration, each microservice with Hystrix on board pushes metrics to Turbine via Spring Cloud Bus (with AMQP broker). The Monitoring project is just a small Spring boot application with the [Turbine](https://github.com/Netflix/Turbine) and [Hystrix Dashboard](https://github.com/Netflix-Skunkworks/hystrix-dashboard).

Let's see observe the behavior of our system under load: Statistics Service imitates a delay during the request processing. The response timeout is set to 1 second:

<img width="880" src="https://cloud.githubusercontent.com/assets/6069066/14194375/d9a2dd80-f7be-11e5-8bcc-9a2fce753cfe.png">

<img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127349/21e90026-f628-11e5-83f1-60108cb33490.gif">	| <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127348/21e6ed40-f628-11e5-9fa4-ed527bf35129.gif"> | <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127346/21b9aaa6-f628-11e5-9bba-aaccab60fd69.gif"> | <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127350/21eafe1c-f628-11e5-8ccd-a6b6873c046a.gif">
--- |--- |--- |--- |
| `0 ms delay` | `500 ms delay` | `800 ms delay` | `1100 ms delay`
| Well behaving system. Throughput is about 22 rps. Small number of active threads in the Statistics service. Median service time is about 50 ms. | The number of active threads is growing. We can see purple number of thread-pool rejections and therefore about 40% of errors, but the circuit is still closed. | Half-open state: the ratio of failed commands is higher than 50%, so the circuit breaker kicks in. After sleep window amount of time, the next request goes through. | 100 percent of the requests fail. The circuit is now permanently open. Retry after sleep time won't close the circuit again because a single request is too slow.
### Log analysis

Centralized logging can be very useful while attempting to identify problems in a distributed environment. Elasticsearch, Logstash and Kibana stack lets you search and analyze your logs, utilization and network activity data with ease.

### Distributed tracing

Analyzing problems in distributed systems can be difficult, especially trying to trace requests that propagate from one microservice to another.
## Infrastructure automation

Deploying microservices, with their interdependence, is much more complex process than deploying a monolithic application. It is really important to have a fully automated infrastructure. We can achieve following benefits with Continuous Delivery approach:

Here is a simple Continuous Delivery workflow, implemented in this project:

<img width="880" src="https://cloud.githubusercontent.com/assets/6069066/14159789/0dd7a7ce-f6e9-11e5-9fbb-a7fe0f4431e3.png">

