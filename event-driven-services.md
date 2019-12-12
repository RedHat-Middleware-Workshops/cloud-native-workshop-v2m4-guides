## Lab2 - Creating Event-Driven/Reactive Services

Traditional microservice architecture is typically composed of many individual services with different functions. Each application service has many clients that need to communicate with the service for fetching data. It will become more complex to handle data streams because everything can be a stream of data such as end-user clicks, RESTful APIs, IoT devices generating data. And complexity rises when these services are running on hybrid or multi-cloud infrastructure.

In an event-driven architecture, we can treat data streams as _events_ using reactive programming and distributed messaging. _Reactive programming_ is an asynchronous programming paradigm concerned with data streams and the propagation of change. In the previous lab, we developed Inventory, Catalog, Shopping Cart and Order services with obvious interactions.

In this lab, we'll change our shopping cart and order implementation and add a payment service as an Event-Driven/Reactive application in our cloud-native appliation architecture. These cloud-native applications will use AMQ Streams (based on Apache Kafka) as a messaging/streaming backbome. AMQ Streams makes it easy to run Apache Kafka on OpenShift with key features:

 * Designed for horizontal scalability
 * Message ordering guarantee at the partition level
 * Message rewind/replay - _Long term_ storage allows the reconstruction of an application state by replaying the messages

#### Goals of this lab

The goal is to develop advanced cloud-native applications on **Red Hat Runtimes** and deploy them on **OpenShift 4** including
**AMQ Streams** for distributed messaing capabilities. After this lab, you should end up with something like:

![goal]({% image_path lab2-goal.png %})

Scalability is one of the flagship features of Apache Kafka. It is achieved by partitioning the data
and distributing them across multiple brokers. Such data sharding also has a big impact on how clients connect and use the broker. This is especially visible
when Kafka is running within a platform like Kubernetes but is accessed from outside of that platform.

[Strimzi](https://strimzi.io/) is an open source project that provides container images and operators for running
[Apache Kafka](https://developers.redhat.com/videos/youtube/CZhOJ_ysIiI/).

In this lab, we will use productized and supported versions of the Strimzi and Apache Kafka projects through [Red Hat AMQ](https://www.redhat.com/en/technologies/jboss-middleware/amq?extIdCarryOver=true&sc_cid=701f2000001OH7TAAW){:target="_blank"}.

#### 1. Create a Kafka Cluster and Topics

---

AMQ Streams is already installed using the following _Operators_ so you don't need to install it in this lab:

 * `Kafka Operator` - Responsible for deploying and managing Apache Kafka clusters within an OpenShift cluster.

 * `Topic Operator` - Responsible for managing Kafka topics within a Kafka cluster running within an OpenShift cluster.

 * `User Operator` - Responsible for managing Kafka users within a Kafka cluster running within an OpenShift cluster.

The basic architecture of operators in AMQ is seen below:

![amqstreams]({% image_path kafka-operators-arch.png %}){:width="900px"}

 * Creating a `Kafka cluster` in `userXX-cloudnativeapp` project

Navigate to _Catalog > Developer Catalog_ in the left menu. In the search box, type in 'kafka' and Click on the `Kafka` box.

![kafka]({% image_path kafka-catalog.png %})

Click on **Create** to represent a Kafka cluster using AMQ Streams Operator.

> **WARNING**
>
> Be sure you are in your `userXX-cloudnativeapp` project in the drop-down menu at the top. If you are in any other project, and try to create things, it will fail with permission denied!

![kafka]({% image_path kafka-create.png %})

You will enter YAML editor that defines a `Kafka` Cluster. Keep the all values as-is then click on **Create** on the bottom.

![kafka]({% image_path kafka-create-detail.png %})

Next, we will create Kafka _Topic_. Return to _Catalog > Developer Catalog_ and type in `kafka` but this time click on the **Kafka Topic** box.

![kafka]({% image_path kafka-topic-catalog.png %})

Click on **Create** to create a topic inside the Kafka cluster.

![kafka]({% image_path kafka-topic-create.png %})

You will enter YAML editor that defines a `KafkaTopic` object. Change the name to `orders` as shown then click on **Create** on the bottom.

![kafka]({% image_path kafka-topic-orders-create.png %})

Create another topic using the same process as above, but called `payments`

![kafka]({% image_path kafka-another-topic-create.png %})

Change the name to `payments` then click on **Create** on the bottom.

![kafka]({% image_path kafka-topic-payments-create.png %})

**Well done!** You now have a running Kafka cluster with two Kafka Topics called `payments` and `orders`.

![kafka]({% image_path kafka-topics-created.png %})

#### 2. Develop and Deploy Payment Service

---

Our **Payment Service** will offer online services for accepting electronic payments by a variety of payment methods including credit card or
bank-based payments when orders are checked out in shopping cart. It doesn't really do anything but will represent a payment microservice that will "process" online shopping orders as they are posted to our services.

In CodeReady Workspaces, navigate to the `payment-service` directory.

![catalog]({% image_path codeready-workspace-payment-project.png %}){:width="500px"}

In this step, we will learn how our Quarkus-based payment service can use Kafka to receive order events and _react_ with payment events.

##### Adding Maven Dependencies using Quarkus Extensions

Execute the following command via CodeReady Workspaces _Terminal_:

`cd /projects/cloud-native-workshop-v2m4-labs/payment-service/`

`mvn quarkus:add-extension -Dextensions="kafka"`

This command imports the Kafka extensions for Quarkus applications
and provides all the necessary capabilities to integrate with Kafka clusters. Confirm your `pom.xml` looks as below, with the new dependencies:

![payment]({% image_path payment-pom-dependency.png %})

##### Writing the application

Let’s start by adding fields to access configuration using `@ConfigProperty` and a `Producer` field which will be used to send messages. We'll also add a `log` field so we can see debug messages later on.

 * Add this code to the `PaymentResource.java` file (in the `src/main/java/com/redhat/cloudnative` directory) at the `// TODO: Add Messaging ConfigProperty here` marker:

~~~java
    @ConfigProperty(name = "mp.messaging.outgoing.payments.bootstrap.servers")
    public String bootstrapServers;

    @ConfigProperty(name = "mp.messaging.outgoing.payments.topic")
    public String paymentsTopic;

    @ConfigProperty(name = "mp.messaging.outgoing.payments.value.serializer")
    public String paymentsTopicValueSerializer;

    @ConfigProperty(name = "mp.messaging.outgoing.payments.key.serializer")
    public String paymentsTopicKeySerializer;

    private Producer<String, String> producer;

    public static final Logger log = LoggerFactory.getLogger(PaymentResource.class);

~~~

Next, we need a method to handle incoming events, which in this lab will be coming directly from Kafka, but later will come through as HTTP POST events.

 * Add this code at the `// TODO: Add handleCloudEvent method here` marker:

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
            orderId = event.getString("orderId");
            String total = event.getString("total");
            JsonObject ccDetails = event.getJsonObject("creditCard");
            String name = event.getString("name");

            // fake processing time
            Thread.sleep(5000);
            if (!ccDetails.getString("number").startsWith("4")) {
                 fail(orderId, paymentId, "Invalid Credit Card: " + ccDetails.getString("number"));
            }
             pass(orderId, paymentId, "Payment of " + total + " succeeded for " + name + " CC details: " + ccDetails.toString());
        } catch (Exception ex) {
             fail(orderId, paymentId, "Unknown error: " + ex.getMessage() + " for payment: " + cloudEventJson);
        }
    }
~~~

> Note that the `Thread.sleep(5000);` will cause credit card "processing" to take 5 seconds, to simulate a real world processing time.

Now we need to implement the `pass()` and `fail()` methods referenced above. These methods will send messages to Kafka using our `producer` field.

 * Add the following code to the `// TODO: Add pass method here` marker:

~~~java
    private void pass(String orderId, String paymentId, String remarks) {

        JsonObject payload = new JsonObject();
        payload.put("orderId", orderId);
        payload.put("paymentId", paymentId);
        payload.put("remarks", remarks);
        payload.put("status", "COMPLETED");
        log.info("Sending payment success: " + payload.toString());
        producer.send(new ProducerRecord<String, String>(paymentsTopic, payload.toString()));
    }
~~~

 * Add this code to the `// TODO: Add fail method here` marker:

~~~java
    private void fail(String orderId, String paymentId, String remarks) {
        JsonObject payload = new JsonObject();
        payload.put("orderId", orderId);
        payload.put("paymentId", paymentId);
        payload.put("remarks", remarks);
        payload.put("status", "FAILED");
        log.info("Sending payment failure: " + payload.toString());
        producer.send(new ProducerRecord<String, String>(paymentsTopic, payload.toString()));
    }
~~~

Next, add a method that will receive events from Kafka. We will use the MicroProfile reactive messaging API `@Incoming` annotation to do this.

* Add this code to the `// TODO: Add consumer method here` marker:

~~~java
    @Incoming("orders")
    public CompletionStage<Void> onMessage(KafkaMessage<String, String> message)
            throws IOException {

        log.info("Kafka message with value = {} arrived", message.getPayload());
        handleCloudEvent(message.getPayload());
        return message.ack();
    }
~~~

And finally, we need a method to initialize the Kafka producer (the consumer will be initialized automatically via Quarkus Kafka extension). We will use the Quarkus `StartupEvent` Lifecycle listener API, with the `@Observes` annotation to mark this method as one that should run when the app starts:

 * Add this code to the `// TODO: Add init method here` marker:

~~~java
    public void init(@Observes StartupEvent ev) {
        Properties props = new Properties();

        props.put("bootstrap.servers", bootstrapServers);
        props.put("value.serializer", paymentsTopicValueSerializer);
        props.put("key.serializer", paymentsTopicKeySerializer);
        producer = new KafkaProducer<String, String>(props);
    }
~~~

This method will consume Kafka streams from the `orders` topic and call our `handleCloudEvent()` method. Later on we'll delete this method and use Knative Events to handle the incoming stream. But for now we'll use this method to listen to the topic.

##### Configuring the application

Quarkus and its extensions are configured by an `application.properties` file. Open this file (it is in the `src/main/resources` directory).

 * Add these values to the file:

~~~java
# Outgoing stream
mp.messaging.outgoing.payments.bootstrap.servers=my-cluster-kafka-bootstrap:9092
mp.messaging.outgoing.payments.connector=smallrye-kafka
mp.messaging.outgoing.payments.topic=payments
mp.messaging.outgoing.payments.value.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.payments.key.serializer=org.apache.kafka.common.serialization.StringSerializer

# Incoming stream (unneeded when using Knative events)
mp.messaging.incoming.orders.connector=smallrye-kafka
mp.messaging.incoming.orders.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.orders.key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.orders.bootstrap.servers=my-cluster-kafka-bootstrap:9092
mp.messaging.incoming.orders.group.id=payment-order-service
mp.messaging.incoming.orders.auto.offset.reset=earliest
mp.messaging.incoming.orders.enable.auto.commit=true
mp.messaging.incoming.orders.request.timeout.ms=30000
~~~

##### Deploying Payment service to OpenShift

Package the payment application by clicking on **Package for OpenShift** in the Commands Palette`:

![payment]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following command in a CodeReady Workspaces _Terminal_:

`mvn clean package -DskipTests`

This will build an executable JAR file in the `target/` directory.

* To deploy this to OpenShift, define a new build in our project:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=payment -l app=payment`

> This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index){:target="_blank"}, providing foundational software needed to run Java applications, while staying at a reasonable size.

 * Force update the OpenJDK image tags just in case they haven't been imported yet:

`oc import-image openjdk18-openshift --all`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build payment --from-file target/*-runner.jar --follow`

![payment]({% image_path payment-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done:

`oc new-app payment`

 * Create the route

`oc expose svc/payment`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/payment`

Wait for that command to report `replication controller payment-1 successfully rolled out` before continuing.

> **NOTE:**
> Even if the rollout command reports success the application may not be ready yet and the reason for
> that is that we currently don't have any liveness check configured.

 * Testing the Application

Go to _Workloads > Pods_ on the left menu then search `cluster-kafka` pods. Click on the `my-cluster-kafka-0` pod:

![payment]({% image_path my-cluster-kafka-0.png %})

We will watch the Kafka topic via a CLI to confirm the messages are being sent/received in Kafka. Click on the _Terminal_ tab in OpenShift (not in CodeReady!) then execute the following command:

`bin/kafka-console-consumer.sh --topic payments --bootstrap-server localhost:9092`

![payment]({% image_path kafka-console-consumer.png %})

Keep this tab open to act as a debugger for Kafka messages.

Let's produce a new topic message using `curl` command in CodeReady Workspaces _Terminal_:

First, fetch the URL of our new payment service and store it in an environment variable:

`export URL="http://$(oc get route | grep payment | awk '{print $2}')"`

Then execute this to HTTP POST a message to our payment service with an example order:

~~~shell
curl -i -H 'Content-Type: application/json' -X POST -d'{"orderId": "12321","total": "232.23", "creditCard": {"number": "4232454678667866","expiration": "04/22","nameOnCard": "Jane G Doe"}, "billingAddress": "123 Anystreet, Pueblo, CO 32213", "name": "Jane Doe"}' $URL
~~~

The payment service will recieve this _order_ and produce a _payment_ result on the Kafka _payment_ topic. You will see the following result in `Pod Terminal`:

~~~shell
{"orderId":"12321","paymentId":"25658","remarks":"Payment of 232.23 succeeded for Jane Doe CC details: {\"number\":\"4232454678667866\",\"expiration\":\"04/22\",\"nameOnCard\":\"Jane G Doe\"}","status":"COMPLETED"}
~~~

![payment]({% image_path payment_curl_result.png %})

Before moving to the next step, stop the Kafka consumer console via `CTRL + C` in Terminal:

![payment]({% image_path kafka-console-consumer-stop.png %})

#### 3. Adding Kafka Client to Cart Service

---

By now we have added several microservices to operate on our retail shopping data. Quite often, other services or functions would need the data we are working with. e.g.  once a user checks out, there are other services like an _Order Service_ and our _Payment Service_ that will need this information, and would most likely want to process further. So we will integrate our Cart service with Kafka so that it can send an order message when a shopper checks out.

To do that open the `cart-service/src/main/java/com/redhat/cloudnative/CartResource.java` file in CodeReady.

##### Adding Maven Dependencies using Quarkus Extensions

Execute the following command via CodeReady Workspaces _Terminal_:

`cd /projects/cloud-native-workshop-v2m4-labs/cart-service/`

`mvn quarkus:add-extension -Dextensions="kafka"`

This will add the Kafka extension and APIs to our Cart service app.

* Like our Payment service, add this code to the `// TODO: Add annotation of orders messaging configuration here` marker inside the `CartResource` class inside the `com.redhat.cloudnative` package:

~~~java
    @ConfigProperty(name = "mp.messaging.outgoing.orders.bootstrap.servers")
    public String bootstrapServers;

    @ConfigProperty(name = "mp.messaging.outgoing.orders.topic")
    public String ordersTopic;

    @ConfigProperty(name = "mp.messaging.outgoing.orders.value.serializer")
    public String ordersTopicValueSerializer;

    @ConfigProperty(name = "mp.messaging.outgoing.orders.key.serializer")
    public String ordersTopicKeySerializer;

    private Producer<String, String> producer;
~~~

Next, un-comment (or add if they are missing) the following `import` statements:

~~~java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;
~~~

The init method as it denotes creates the Kafka configuration, we have externalized this configuration and injected the variables as properties on the class.

* Replace the empty `init()` method with this code:

~~~java
    public void init(@Observes StartupEvent ev) {
        Properties props = new Properties();

        props.put("bootstrap.servers", bootstrapServers);
        props.put("value.serializer", ordersTopicValueSerializer);
        props.put("key.serializer", ordersTopicKeySerializer);
        producer = new KafkaProducer<String, String>(props);
    }
~~~

The `sendOrder()` method is quite simple, it takes the Order POJO as a param and serializes that into JSON to send over the Kafka topic.

* Replace the empty `sendOrder()` method with this code:

~~~java
    private void sendOrder(Order order, String cartId) {
        order.setTotal(shoppingCartService.getShoppingCart(cartId).getCartTotal() + "");
        producer.send(new ProducerRecord<String, String>(ordersTopic, Json.encode(order)));
        log.info("Sent message: " + Json.encode(order));
    }
~~~

Now that we have those methods, lets add a call to our `sendOrder()` method when we are checking out. Replace the code for `checkout()` with this code:

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

Almost there! Next let's add the configuration to our `application.properties` file (in the `src/main/resources` of the `cart-service` project):

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

Or run the following maven plugin in CodeReady Workspaces _Terminal_:

`mvn clean package -DskipTests`

Rebuild a container image based the cart artifact that we just packaged, which will take about minutes to complete:

`oc start-build cart --from-file target/*-runner.jar --follow`

![cart]({% image_path cart-build-logs.png %})

The cart service will be redeployed automatically via [OpenShift Deployment triggers](https://docs.openshift.com/container-platform/4.1/applications/deployments/managing-deployment-processes.html#deployments-triggers_deployment-operations){:target="_blank"} after it completes to build.

#### 4. Adding Kafka Client to Order Service

Like the `payments` service, our `order` service will listen for orders being placed, but will not process payments - instead the order service will merely record the orders and their states for eventual display in the UI. Let's add this capability to the order service.

---

##### Adding Maven Dependencies using Quarkus Extensions

Execute the following command via CodeReady Workspaces Terminal:

`cd /projects/cloud-native-workshop-v2m4-labs/order-service/`

`mvn quarkus:add-extension -Dextensions="kafka"`

This command generates a Maven project, importing the Kafka extensions for Quarkus applications
and provides all the necessary capabilities to integrate with the Kafka clusters and subscribe `payments` topic and `orders` topic. Let's confirm your `pom.xml` as below:

![order]({% image_path order-kafka-pom-dependency.png %})

##### Creating Orders and Payments Consumer in Order Service

In the `order-service` project, Create a new Java class, `KafkaOrders.java` in `src/main/java/com/redhat/cloudnative` to consume messages from the Kafka `orders` and `payments` topic. Copy the following entire code into `KafkaOrders.java`.

~~~java
package com.redhat.cloudnative;

import io.smallrye.reactive.messaging.kafka.KafkaMessage;
import org.eclipse.microprofile.reactive.messaging.Incoming;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.enterprise.context.ApplicationScoped;

import java.io.IOException;
import java.util.concurrent.CompletionStage;

import javax.inject.Inject;
import io.vertx.core.json.JsonObject;

@ApplicationScoped
public class KafkaOrders {

    private static final Logger LOG = LoggerFactory.getLogger(KafkaOrders.class);

    @Inject
    OrderService orderService;

    @Incoming("orders")
    public CompletionStage<Void> onMessage(KafkaMessage<String, String> message)
            throws IOException {

        LOG.info("Kafka order message with value = {} arrived", message.getPayload());

        JsonObject orders = new JsonObject(message.getPayload());
        Order order = new Order();
        order.setOrderId(orders.getString("orderId"));
        order.setName(orders.getString("name"));
        order.setTotal(orders.getString("total"));
        order.setCcNumber(orders.getJsonObject("creditCard").getString("number"));
        order.setCcExp(orders.getJsonObject("creditCard").getString("expiration"));
        order.setBillingAddress(orders.getString("billingAddress"));
        order.setStatus("PROCESSING");
        orderService.add(order);

        return message.ack();
    }

    @Incoming("payments")
    public CompletionStage<Void> onMessagePayments(KafkaMessage<String, String> message)
            throws IOException {

        LOG.info("Kafka payment message with value = {} arrived", message.getPayload());

        JsonObject payments = new JsonObject(message.getPayload());
        orderService.updateStatus(payments.getString("orderId"), payments.getString("status"));

        return message.ack();
    }

}
~~~

Almost there; Next lets add the configuration to our `src/main/resources/application.properties` file in the `order-service` project:

~~~java
# Incoming payment topic messages
mp.messaging.incoming.payments.connector=smallrye-kafka
mp.messaging.incoming.payments.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.payments.key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.payments.bootstrap.servers=my-cluster-kafka-bootstrap:9092
mp.messaging.incoming.payments.group.id=order-service
mp.messaging.incoming.payments.auto.offset.reset=earliest
mp.messaging.incoming.payments.enable.auto.commit=true
mp.messaging.incoming.payments.request.timeout.ms=30000

# Enable CORS requests from browsers
quarkus.http.cors=true

# Incoming order topic messages
mp.messaging.incoming.orders.connector=smallrye-kafka
mp.messaging.incoming.orders.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.orders.key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.orders.bootstrap.servers=my-cluster-kafka-bootstrap:9092
mp.messaging.incoming.orders.group.id=order-service
mp.messaging.incoming.orders.auto.offset.reset=earliest
mp.messaging.incoming.orders.enable.auto.commit=true
mp.messaging.incoming.orders.request.timeout.ms=30000
~~~

##### Re-Deploying Order service to OpenShift

Package the order application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following maven plugin in CodeReady Workspaces _Terminal_:

`mvn clean package -DskipTests`

![order]({% image_path order-mvn-package.png %})

Rebuild a container image based the cart artifact that we just packaged, which will take about minutes to complete:

`oc start-build order --from-file target/*-runner.jar --follow`

The order service will be redeployed automatically via [OpenShift Deployment triggers](https://docs.openshift.com/container-platform/4.1/applications/deployments/managing-deployment-processes.html#deployments-triggers_deployment-operations){:target="_blank"} after it completes to build.

Let's confirm if the all services works correctly using `Kafka` messaging via coolstore GUI test.

####5. End to End Functional Testing

---

Let's go shopping! Open the Web UI in your browser. To get the URL to the Web UI, run this command in CodeReady _Terminal_:

`oc get route | grep coolstore-ui | awk '{print $2}'`

Add some cool items to your shopping cart in the following shopping scenarios:

 * 1) Add a _Red Hat Fedora_ to your cart by click on **Add to Cart**. You will see the `Success! Added!` message under the top munu.

![serverless]({% image_path add-to-cart.png %})

 * 2) Go to the **Your Shopping Cart** tab and click on the **Checkout** button . Input the credit card information. The Card Info should be 16 digits and begin with the digit `4`. For example `4123987754646678`.

![serverless]({% image_path checkout.png %})

 * 3) Input your Credit Card information to pay for the items:

 ![serverless]({% image_path input-cc-info.png %})

 * 4) Confirm the _Payment Status_ of the your shopping items in the **All Orders** tab. It should be `Processing`.

 ![serverless]({% image_path payment-processing.png %})

 * 5) After a few moments, reload the **All Orders** page to confirm that the Payment Status changed to `COMPLETED` or `FAILED`.

 >`Note`: If the status is still `Processing`, the order service is processing incoming Kafka messages and storing them in MongoDB. Please reload the page a few times more.

 ![serverless]({% image_path payment-completedorfailed.png %})

### Summary

In this scenario we developed an _Event-Driven/Reactive_ cloud-native appliction to deal with data streams from the shopping cart service to the order service and payment service using _Apache Kafka_).

We also used Quarkus and its _Kafka extension_ to integrate the app with Kafka. `AMQ Streams`, a fully supported Kafka solution from Red Hat, enables you to create Apache Kafka clusters very easily via OpenShift developer catalog.

In the end, we now have message-driven microservices for implementing reactive systems, where all the components interact using asynchronous messages passing. Most importantly, **Quarkus** is perfectly suited to implement event-driven microservices and reactive systems. Congratulations!
