#### 3. Developing and Deploying Shopping Cart Service

By now, you have deployed some of the essential elements for the Coolstore application. However, an online shop without a cart means no checkout experience. In this section, we are going to implement the Shopping Cart; in our Microservice world, we are going to call it the "cart service," and our java artifact/repo is called the "cart-service."

## Let's get started !

In a nutshell, the Cart service is RESTful and built with Quarkus. 

#### What are the building blocks of the shopping cart a.k.a cart-service? 
It uses a Red Hat's Distributed Data Grid technology to store all shopping carts and assigns a unique id to them. It uses the Quarkus Infinispan client to do this.
The Shopping cart makes a call via the Quarkus Rest client to fetch all items in the Catalog. 
The Shopping cart also throws out a Kafka message to the topic Orders, when checking out. For that, we use the Quarkus Kafka client.
Last and perhaps worth mentioning the REST+Swagger UI also part of the REST API support in Quarkus.

What is a Shopping Cart in our context?
- A Shopping cart has a list of Shopping Items.
- Quantity of a product in the Items list
- Discount and promotional details. 
- We will see these in more details when we look at our model.

For this lab, we are using the code ready workspaces, make sure you have the following project open in your workspace

```
cloud-native-workshop-v3m4-labs 
```

#### Building REST API with Quarkus
The cart is quite simple; All the information from the browser i.e., via our Angular App is via JSON. 
What is the endpoint '/api/cart'?:
- GET request '/{cartId}' gets the items in the cart, or creates a new unique ID if one is not present
- POST to '/{cartId}/{itemId}/{quantity}' will add items to the cart
- DELETE '/{cartId}/{itemId}/{quantity}' will remove items from the cart
- And finally '/checkout/{cartId}' will remove the items and invoke the checkout procedure

Let's take a look at how we do this with Quarkus. 
In our project and in our main package i.e., com.redhat.cloudnative is the CartResource. Let's take a look at the getCart method.

```
// TODO
 public ShoppingCart getCart(@PathParam("cartId") String cartId) {	 
 return shoppingCartService.getShoppingCart(cartId);	 
 }
```

The code above is using the ShoppingCartService, which is injected into the CartResource via the Dependency Injection. The ShoppingCartService take a cartId as a parameter and returns the associated ShoppingCart. So that's perfect, however, for our Endpoint i.e., CartResource to respond, we need to define a couple of things. 
The type of HTTPRequest
The type of data it can receive
The path it resolves too. 

 Add the following code on top of the getCart method

```
 @GET     
 @Produces(MediaType.TEXT_PLAIN)     
 @Path("/{cartId}")     
 @Operation(summary = "get the contents of cart by cartId")     
```

We have now successfully stated that the method adheres to a GET request and accepts data in 'plain text' 
The path would be '/api/cart/{cartId}'
finally, we add the @Operation for some documentation, which is important for other developers using our service.

Take this opportunity to look at some of the other methods. You will find @POST and @DELETE and also the paths they adhere too. This is how we can construct a simple Endpoint for our application.

```
// Run the following command
mvn compile quarkus:dev:
```
And hit the preview URL and add /swagger-ui to the end.
You should see the following output in your browser.

![cart]({% image_path cart-swagger-ui.png %})


Notice that the documentation after the methods, this is an excellent way for other service developers to know what you intend to do with each service method. 
You can try to invoke the methods and see the output from the service. Hence an excellent way to test quickly as well.

#### Adding a distributed cache to our cart-service
We are going to use the Red hat Distributed Data Grid for caching all the users' carts. 

Red Hat® Distributed Data Grid is an in-memory, distributed, NoSQL datastore solution. Your applications can access, process, and analyze data at in-memory speed to deliver a superior user experience with features and benefits as below:
Act faster - Quickly access your data through fast, low-latency data processing using memory (RAM) and distributed parallel execution.
Scale quickly - Achieve linear scalability with data partitioning and distribution across cluster nodes.
Always available - Gain high availability through data replication across cluster nodes.
Fault tolerance - Attain fault tolerance and recover from disaster through cross-data center geo-replication and clustering.
More productivity - Gain development flexibly and higher productivity with a highly versatile, functionally rich NoSQL data store.
Protect data - Obtain comprehensive data security with encryption and role-based access.

Lets create a simple version of the cache service in our cluster. 
Open the Terminal in your CodeReady workspace and run the following command

```
oc new-app jboss/infinispan-server:10.0.0.Beta3 --name=datagrid-service
```

>NOTE: This will create a single instance of infinispan server the community version of the DataGrid. At the time of writing this guide, the infinspan client for Quarkus does not work with DataGrid, and Quarkus itself is also a community project.

Once deployed you should see the newly created 'datagrid-service' in your project dashboard as follows:

![cart]({% image_path cart-cache-pod.png %})

Now that our cache service a.k.a datagrid-service is deployed. We want to ensure that everything in our cart is persisted in this blazing fast cache. It will help us when we have a few million users per second on a black Friday.

Following is what we need to do
- Model our data
- Choose how we store the data
- Create a marshaller for our data
- Inject our cache connection into the service

We have made this choice easier for you. The default serialization is done using a library based on `protobuf`. We need to define the protobuf schema and a marshaller for each user type(s).

Let's take a look at our cart.proto file in META-INF

```
package coolstore;

message ShoppingCart {
  required double cartItemTotal = 1;
  required double cartItemPromoSavings = 2;
  required double shippingTotal = 3;
  required double shippingPromoSavings = 4;
  required double cartTotal = 5;
  required string cartId = 6;

  repeated ShoppingCartItem shoppingCartItemList = 7;
}

message ShoppingCartItem {
  required double price = 1;
  required int32 quantity = 2;
  required double promoSavings = 3;
  required Product product = 4;
}


// TODO ADD Product

message Promotion {
  required string itemId = 1;
  required double percentOff = 2;
}

```

- So our ShoppingCart has ShoppingCartItem
- ShoppingCartItem has Product

But we havent defined the Product yet. Lets go ahead and do that.


```
message Product {
  required string itemId = 1;
  required string name = 2;
  required string desc = 3;
  required double price = 4;
}
```

Great, now we have the Product defined in our proto model. 
We should also ensure that this model also exists as POJO (Plain Old Java Object), that way our REST Endpoint , or Cache will be able to directly serialize and desrialize the data. 

Lets open up our Product.java in package model

```
    private String itemId;
    private String name;
    private String desc;
    private double price;
```

Notice that the entities match our proto file. The rest or Getters and Setters, so we can read and write data into them.

Lets go ahead and create a Marshaller for our Product class which will do exactly what we intend, read and write to our cache.

Create a new Java class called ProductMarshaller.java in com.redhat.cloudnative.model


```
public class ProductMarshaller implements MessageMarshaller<Product> {

    /**
     * Proto file specimen
     * message Product {
     * required string itemId = 1;
     * required string name = 2;
     * required string desc = 3;
     * required double price = 4;
     * }
     */

    @Override
    public Product readFrom(ProtoStreamReader reader) throws IOException {
        String itemId = reader.readString("itemId");
        String name = reader.readString("name");
        String desc = reader.readString("desc");
        double price = reader.readDouble("price");

        return new Product(itemId, name, desc, price);
    }

    @Override
    public void writeTo(ProtoStreamWriter writer, Product product) throws IOException {
        writer.writeString("itemId", product.getItemId());
        writer.writeString("name", product.getName());
        writer.writeString("desc", product.getDesc());
        writer.writeDouble("price", product.getPrice());
    }

    @Override
    public Class<? extends Product> getJavaClass() {
        return Product.class;
    }

    @Override
    public String getTypeName() {
        return "coolstore.Product";
    }


}

```

So now we have the capability to read from a ProtoStream and Write to it. And this will be done directly into our cache. We have already created the other model classes and mashallers, feel free to look around.

Now its time to configure our RemoteCache, since its not embedded into our service. 

Open the file com.redhat.cloudnative.Producers

We use the producer to ensure our RemoteCache gets instantiated.
We create methods called getCache and getConfigBuilder
- getConfigBuilder: sets up the basic cache config
- getCache, sets up our marshallers and proto files
- other config properties are injected at runtime

```
    @Produces
    RemoteCache<String, ShoppingCart> getCache() throws IOException {

        RemoteCacheManager manager = new RemoteCacheManager(getConfigBuilder().build());

        SerializationContext serCtx = ProtoStreamMarshaller.getSerializationContext(manager);
        FileDescriptorSource fds = new FileDescriptorSource();
        fds.addProtoFiles("META-INF/cart.proto");
        serCtx.registerProtoFiles(fds);
        serCtx.registerMarshaller(new ShoppingCartMarshaller());
        serCtx.registerMarshaller(new ShoppingCartItemMarshaller());
        serCtx.registerMarshaller(new ProductMarshaller());
        serCtx.registerMarshaller(new PromotionMarhsaller());
        return manager.getCache();
    }



    protected ConfigurationBuilder getConfigBuilder() {
        ConfigurationBuilder cfg = null;
        cfg = new ConfigurationBuilder().addServer()
                .host(dgHost)
                .port(dgPort)
                .marshaller(new ProtoStreamMarshaller())
                .clientIntelligence(ClientIntelligence.BASIC);

        return cfg;

    }

```

Perfect, now we have all the building blocks ready to use the cache. 
Lets start using our cache. 

First we need to make sure we will inject our cache in our service like this in 
com.redhat.cloudnative.service.ShoppingCartServiceImpl
```
    @Inject
    @Remote("default")
    RemoteCache<String, ShoppingCart> carts;
```

And then we need to make sure we add it to the application.properties file
```
quarkus.infinispan-client.server-list=datagrid-service:11222
```

#### Adding Kafka to our midst
By now we have added our REST API, Cache for our Cart. Quite often, other services or functions would need the data we are working with. And same in this case, once a user checks out, there are other services like the Order Service and the Payment Service that will need this information, and would most likely want to process further. 
So we need to make sure we can send a Kafka message to topic 'orders'.

To do that add the following methods in the CartResource

The init method as it denotes creates the Kafka configuration, we have externalized this configuration and injected the variables as properties on the class.

```
    public void init(@Observes StartupEvent ev) {
        Properties props = new Properties();

        props.put("bootstrap.servers", bootstrapServers);
        props.put("value.serializer", ordersTopicValueSerializer);
        props.put("key.serializer", ordersTopicKeySerializer);
        producer = new KafkaProducer<String, String>(props);
    }
```

The Send Order method is quite simple, it takes the Order POJO as a param and serializes that into JSON to send over the KafkaTopic. 

```
    private void sendOrder(Order order, String cartId) {
        order.setKey(cartId);
        order.setTotal(shoppingCartService.getShoppingCart(cartId).getCartTotal() + "");
        producer.send(new ProducerRecord<String, String>(ordersTopic, Json.encode(order)));
        log.info("Sent message: " + Json.encode(order));
    }
```

Now that we have those methods, lets call the sendOrder and we should do it in our checkout method like following:
```
    @POST
    @Path("/checkout/{cartId}")
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    @Operation(summary = "checkout")
    public ShoppingCart checkout(@PathParam("cartId") String cartId, Order order) {
        sendOrder(order, cartId);
        return shoppingCartService.checkout(cartId);
    }

```

Almost there; Next lets add the configuration to our application.properties file

```
mp.messaging.outgoing.orders.bootstrap.servers=my-cluster-kafka-bootstrap:9092
mp.messaging.outgoing.orders.connector=smallrye-kafka
mp.messaging.outgoing.orders.topic=orders
mp.messaging.outgoing.orders.value.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.orders.key.serializer=org.apache.kafka.common.serialization.StringSerializer
```

#### Package and Deploy the cart-service
Package the cart application via clicking on `Package for OpenShift` in `Commands Palette`:

![cart]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following maven plugin in CodeReady Workspaces`Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/cart-service/`

`mvn clean package -DskipTests`

Build the image using on OpenShift:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=cart -l app=cart`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

 * Create a temp directory to store only previously-built application with necessary lib directory:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build cart --from-dir=target/binary --follow`

![cart]({% image_path cart-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done:

`oc new-app cart`

 * Create the route

`oc expose svc/cart`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/cart`

Wait for that command to report replication controller `cart-1` successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for

Like we had done before, you can test the serice by appending /swagger-ui to the URL

