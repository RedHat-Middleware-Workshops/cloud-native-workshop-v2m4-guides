## Lab2 - Creating Event-Driven/Reactive Services

`Traditional microservices architecture` is typically composed of many individual services with different functions. Each application service has probably many clients that need to communicate with the service for fetching data. Also, clients can be application services to other clients. It will become more complex to handle `data streams` because everything can be a stream of data such as `end-user clicks`, `RESTful APIs`, `IoT devices data`, and more when these services are running on `multi-cloud` and `hybrid cloud` infrastructure.

In an `event-driven architecture`, we can treat data streams as `events` using `reactive programming` and `distributed messaging platform`. `Reactive programming` is an asynchronous programming paradigm concerned with data streams and the propagation of change. In the previous lab, we developed Inventory, Catalog, Shopping Cart and Order services with obvious interactions. 

In this lab, we'll change `Shoppiong Cart`, `Order`, and create a new `Payment` as `Event-Driven/Reactive` applications into the cloud-native appliation architecture. These cloud-native applications will read data from and write data to the `Apache Kafka cluster`. `AMQ Streams` is based on `Apache Kafka`, a popular platform for 
streaming data delivery and processing. `AMQ Streams` makes it easy to run `Apache Kafka` on OpenShift with key features:

 * Designed for horizontal scalability

 * Message ordering guarantee at the partition level

 * Message rewind/replay - _Long term_ storage allows the reconstruction of an application state by replaying the messages

#### Goals of this lab

---

The goal is to develop advanced cloud-native applications on `Red Hat Runtimes` and deploy them on `OpenShift 4` including 
`Apache Kafka in AMQ Streams` for distributed messaing capabilities. After this lab, you should end up with something like:

![goal]({% image_path lab2-goal.png %})

####1. Creating a Kafka Cluster and Topics

---

[Strimzi](https://strimzi.io/) is an open source project that provides container images and operators for running 
[Apache Kafka](https://developers.redhat.com/videos/youtube/CZhOJ_ysIiI/) on [Kubernetes](https://developers.redhat.com/topics/kubernetes/) and 
[Red Hat OpenShift](https://developers.redhat.com/openshift/). `Scalability` is one of the flagship features of Apache Kafka. It is achieved by partitioning the data 
and distributing them across multiple brokers. Such data sharding has also a big impact on how `Kafka clients` connect to the brokers. This is especially visible 
when `Kafka` is running within a platform like `Kubernetes` but is accessed from outside of that platform.

In this lab, we will use `productized` and `supported` versions of the Strimzi and Apache Kafka projects are available with `AMQ Streams` as part of the 
[Red Hat AMQ product](https://www.redhat.com/en/technologies/jboss-middleware/amq?extIdCarryOver=true&sc_cid=701f2000001OH7TAAW).

AMQ Streams is `already installed` using the following `Operators` so we don't nee to install it in this lab:

 * `Cluster Operator` - Responsible for deploying and managing Apache Kafka clusters within an OpenShift cluster.

 * `Topic Operator` - Responsible for managing Kafka topics within a Kafka cluster running within an OpenShift cluster.

 * `User Operator` - Responsible for managing Kafka users within a Kafka cluster running within an OpenShift cluster.

Architecture diagram of the `AMQ Strems Cluster` Operator is here.

![amqstreams]({% image_path kafka-operators-arch.png %}){:width="900px"}

 * Creating a `Kafka cluster` in `userXX-cloudnativeapp` project

Click on `Kafka` in `Developer Catalog` left menu.

![kafka]({% image_path kafka-catalog.png %})

Click on `Create` to represent a Kafka cluster using AMQ Streams Operator.

![kafka]({% image_path kafka-create.png %})

You will enter `YAML` editor that defines a `Kafka` object. Keep the all values witout any changes then click on `Create` on the bottom.

![kafka]({% image_path kafka-create-detail.png %})

Next, we will create `Kafka Topics` in `Developer Catalog` left menu. Click on `Kafka Topic`.

![kafka]({% image_path kafka-topic-catalog.png %})

Click on `Create` to represent a `topic inside a Kafka cluster`.

![kafka]({% image_path kafka-topic-create.png %})

You will enter `YAML` editor that defines a `KafkaTopic` object. Change the name with `orders` then click on `Create` on the bottom.

![kafka]({% image_path kafka-topic-orders-create.png %})

Create a new topic in `Installed Operators` lefe menu and click on `Create Kafka Topic`. Be sure to create it under `userXX-cloudnativeapp` project.

![kafka]({% image_path kafka-another-topic-create.png %})

Change the name with `payments` then click on `Create` on the bottom.

![kafka]({% image_path kafka-topic-payments-create.png %})

`Well done!` You completed to create two Kafka Topics as `payments` and `orders`.

![kafka]({% image_path kafka-topics-created.png %})

####2. Developing and Deploying Payment Service

---

`Payment Service` offers shops online services for accepting electronic payments by a variety of payment methods including credit card, 
bank-based payments when orders are checked out in shopping cart. Lets's go through quickly how the payment service get `REST` services to use 
`Kafka Topics` on `Quarkus` Java runtimes. Go to `Project Explorer` in `CodeReady Workspaces` 
Web IDE and expand `payment-service` directory.

![catalog]({% image_path codeready-workspace-payment-project.png %}){:width="500px"}

In this step, we will learn how the payment Quarkus application can use `Kafka Topics` to receive `order event` as well as send `PaymentAction` such as 
`Completed` and `Failed`.


##### Adding Maven Dependencies using Quarkus Extensions

Execute the following command via CodeReady Workspaces `Terminal`:

`mvn quarkus:add-extension -Dextensions="kafka"`

This command generates a Maven project, importing the `Kafka connector` extensions for Quarkus applications 
and provides all the necessary capabilities to integrate with the Kafka clusters and produce `payments topic`. Let's confirm your `pom.xml` as below:

![payment]({% image_path payment-pom-dependency.png %})

##### Writing the application

Let’s start by implementing the `PaymentResource` to handle `Kafka event` from the order service. As you can see from the source code below it is to create `Kafka producer` and payload the `COMPLETED` or `FAILED` result:

 * `// TODO: Add Messaging ConfigProperty here` marker:

~~~java
@ConfigProperty(name = "mp.messaging.outgoing.payments.bootstrap.servers")
public String bootstrapServers;

@ConfigProperty(name = "mp.messaging.outgoing.payments.topic")
public String paymentsTopic;

@ConfigProperty(name = "mp.messaging.outgoing.payments.value.serializer")
public String paymentsTopicValueSerializer;

@ConfigProperty(name = "mp.messaging.outgoing.payments.key.serializer")
public String paymentsTopicKeySerializer;
~~~

 * `// TODO: Add handleCloudEvent method here` marker:

~~~java
@POST
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.TEXT_PLAIN)
public void handleCloudEvent(String cloudEventJson) {
    String orderId = "unknown";
    String paymentId = "" + ((int)(Math.floor(Math.random() * 100000)));

    try {
        log.info("received event: " + cloudEventJson);
        JsonObject event = new JsonObject(cloudEventJson);
        orderId = event.getString("key");
        Double total = event.getDouble("total");
        JsonObject ccDetails = event.getJsonObject("creditCard");
        String name = event.getString("name");

        if (!ccDetails.getString("number").startsWith("4")) {
              fail(orderId, paymentId, "Invalid Credit Card: " + ccDetails.getString("number"));
        }
          pass(orderId, paymentId, "Payment of " + total + " succeeded for " + name + " CC details: " + ccDetails.toString());
    } catch (Exception ex) {
          fail(orderId, paymentId, "Unknown error: " + ex.getMessage() + " for payment: " + cloudEventJson);
    }
}
~~~

 * `// TODO: Add pass method here` marker:

~~~java
private void pass(String orderId, String paymentId, String remarks) {

    JsonObject payload = new JsonObject();
    payload.put("orderId", orderId);
    payload.put("paymentId", paymentId);
    payload.put("remarks", remarks);
    payload.put("status", "COMPLETED");
    producer.send(new ProducerRecord<String, String>(paymentsTopic, payload.toString()));
}
~~~

 * `// TODO: Add fail method here` marker:

~~~java
private void fail(String orderId, String paymentId, String remarks) {
    JsonObject payload = new JsonObject();
    payload.put("orderId", orderId);
    payload.put("paymentId", paymentId);
    payload.put("remarks", remarks);
    payload.put("status", "FAILED");
    producer.send(new ProducerRecord<String, String>(paymentsTopic, payload.toString()));

}
~~~

 * `// TODO: Add init method here` marker:

~~~java
public void init(@Observes StartupEvent ev) {
    Properties props = new Properties();

    props.put("bootstrap.servers", bootstrapServers);
    props.put("value.serializer", paymentsTopicValueSerializer);
    props.put("key.serializer", paymentsTopicKeySerializer);
    producer = new KafkaProducer<String, String>(props);
}
~~~

##### Configuring the application

The Keycloak extension allows you to define the adapter configuration using either the `application.properties` file or using a `keycloak.json`. 
Both files should be located at the src/main/resources directory.

 * Configuring using the `application.properties` file as below:

~~~java
mp.messaging.outgoing.payments.bootstrap.servers=my-cluster-kafka-bootstrap:9092
mp.messaging.outgoing.payments.connector=smallrye-kafka
mp.messaging.outgoing.payments.topic=payments
mp.messaging.outgoing.payments.value.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.payments.key.serializer=org.apache.kafka.common.serialization.StringSerializer
~~~

For more details about this file and all the supported options, please take a look at [Keycloak Adapter Config](https://www.keycloak.org/docs/latest/securing_apps/index.html#_java_adapter_config).

##### Deploying Payment service to OpenShift

Package the payment application via clicking on `Package for OpenShift` in `Commands Palette`:

![payment]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following maven plugin in CodeReady Workspaces`Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/payment-service/`

`mvn clean package -DskipTests`

Build the image using on OpenShift:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=payment -l app=payment`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

 * Create a temp directory to store only previously-built application with necessary lib directory:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build payment --from-dir=target/binary --follow`

![payment]({% image_path payment-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done:

`oc new-app payment`

 * Create the route

`oc expose svc/payment`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/payment`

Wait for that command to report replication controller `payment-1` successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

* Testing the Application

Go to `Workloads > Pods` on the left menu then search `kafka-cluster` pods. Click on `my-cluster-kafka-0` pod:

![payment]({% image_path my-cluster-kafka-0.png %})

Click on `Terminal` tab then execute the following `kafka-console-consumer.sh`:

~~~shell
bin/kafka-console-consumer.sh --topic payments --bootstrap-server localhost:9092
~~~

![payment]({% image_path kafka-console-consumer.png %})

Let's produce a new topic message using `curl` command in CodeReady Workspaces `Terminal`:

~~~shell
export URL="http://$(oc get route | grep payment | awk '{print $2}')"

curl -i -H 'Content-Type: application/json' -X POST \
  -d'{"key": "12321","total": 232.23, "creditCard": {"number": "4232454678667866","expiration": "04/22","nameOnCard": "Jane G Doe"}, "billingAddress": "123 Anystreet, Pueblo, CO 32213", "name": "Jane Doe"}'  \
  $URL
~~~

You will see the following result in `Pod Terminal`:

~~~shell
{"orderId":"12321","paymentId":"40173","remarks":"Payment of 232.23 succeeded for Jane Doe CC details: {\"number\":\"4232454678667866\",\"expiration\":\"04/22\",\"nameOnCard\":\"Jane G Doe\"}","status":"COMPLETED"}
~~~

![payment]({% image_path payment_curl_result.png %})

####3. Adding Kafka Client to Cart Service

---

By now we have added our `REST API`, `Cache` for our `Cart`. Quite often, other services or functions would need the data we are working with. And same in this case, once a user checks out, there are other services like the `Order Service` and the `Payment Service` that will need this information, and would most likely want to process further. So we need to make sure we can send a `Kafka` message to topic `orders`.

To do that add the following methods in the `CartResource`.

The init method as it denotes creates the Kafka configuration, we have externalized this configuration and injected the variables as properties on the class.

~~~java
    public void init(@Observes StartupEvent ev) {
        Properties props = new Properties();

        props.put("bootstrap.servers", bootstrapServers);
        props.put("value.serializer", ordersTopicValueSerializer);
        props.put("key.serializer", ordersTopicKeySerializer);
        producer = new KafkaProducer<String, String>(props);
    }
~~~

The `sendOrder` method is quite simple, it takes the Order POJO as a param and serializes that into JSON to send over the `KafkaTopic`. 

~~~java
    private void sendOrder(Order order, String cartId) {
        order.setKey(cartId);
        order.setTotal(shoppingCartService.getShoppingCart(cartId).getCartTotal() + "");
        producer.send(new ProducerRecord<String, String>(ordersTopic, Json.encode(order)));
        log.info("Sent message: " + Json.encode(order));
    }
~~~

Now that we have those methods, lets call the `sendOrder` and we should do it in our `checkout` method like following:

~~~java
    @POST
    @Path("/checkout/{cartId}")
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    @Operation(summary = "checkout")
    public ShoppingCart checkout(@PathParam("cartId") String cartId, Order order) {
        sendOrder(order, cartId);
        return shoppingCartService.checkout(cartId);
    }

~~~

Almost there; Next lets add the configuration to our `application.properties` file:

~~~java
mp.messaging.outgoing.orders.bootstrap.servers=my-cluster-kafka-bootstrap:9092
mp.messaging.outgoing.orders.connector=smallrye-kafka
mp.messaging.outgoing.orders.topic=orders
mp.messaging.outgoing.orders.value.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.orders.key.serializer=org.apache.kafka.common.serialization.StringSerializer
~~~

##### Re-Deploying Cart service to OpenShift

Package the cart application via clicking on `Package for OpenShift` in `Commands Palette`:

![cart]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following maven plugin in CodeReady Workspaces`Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/cart-service/`

`mvn clean package -DskipTests`

Rebuild a container image based the cart artifact that we just packaged, which will take about minutes to complete:

`oc start-build cart --from-dir=target/binary --follow`

![cart]({% image_path cart-build-logs.png %})

The cart service will be redeployed automatically via [OpenShift Deployment triggers](https://docs.openshift.com/container-platform/4.1/applications/deployments/managing-deployment-processes.html#deployments-triggers_deployment-operations) after it completes to build.

Let's confirm if the cart and order services works correctly via coolstore GUI test.

### Summary

In this scenario we developed `Event-Driven/Reactive` cloud-native applictions to deal with data streams from the shopping cart service to the order service and payment service using `Apache Kafka Topics`. To do that, the shopping cart service produces `order` messages and the order service subscribes the orders topic to add a new order to `MongoDB` as well as receives the `payments topic` to update the payment result in coolstore web-ui. In the meantime, the payment-service receives the `orders topic` to produce payment processing result messages in the `payments topic` for the order service.

In order to implement this cloud-native application architecture, we use `Quarkus Kafka extension` to implement stream processing applications based on `Apache Kafka`. Especially, `AMQ Streams` enables you to create `Apache Kafka cluster` and `Topics` with easy user experiences via OpenShift developer catalog.  

In the end, we have message-driven microservices for implementing reactive systems, where all the components interact using asynchronous messages passing. Most importantly, `Quarkus` is perfectly suited to implement event-driven microservices and reactive systems. `Congratulations!`