## Lab1 - Creating High-performing Cacheable Service

In this lab, we'll develop 5 microservices into the cloud-native appliation architecture. These cloud-native applications
will have transactions with multiple datasources such as `PostgreSQL` and `MongoDB`. Especially, we will learn how to configure datasources easily using
`Quarkus Extensions`. In the end, we will optimize `data transaction performance` of the shopping cart service thru integrating with a `Cache(Data Grid) server`
to increase end users'(customers) satification. And there's more fun facts how easy it is to deploy applications on OpenShift 4 via `oc` command line tool.

#### Goals of this lab

---

The goal is to develop advanced cloud-native applications on `Red Hat Runtimes` and deploy them on `OpenShift 4` including
`single sign-on access management` and `distributed cache manageemnt`. After this lab, you should end up with something like:

![goal]({% image_path lab1-goal.png %})

####1. Deploying Inventory Service

---

`Inventory Service` serves inventory and availability data for retail products. Lets's go through quickly how the inventory service works and built on
`Quarkus` Java runtimes. Go to `Project Explorer` in `CodeReady Workspaces` Web IDE and expand `inventory-service` directory.

![inventory_service]({% image_path codeready-workspace-inventory-project.png %}){:width="500px"}

While the code is surprisingly simple, under the hood this is using:

 * `RESTEasy` to expose the REST endpoints
 * `Hibernate ORM` with Panache to perform the CRUD operations on the database
 * `Maven` Java project structure

`Hibernate ORM` is the de facto JPA implementation and offers you the full breadth of an Object Relational Mapper.
It makes complex mappings possible, but it does not make simple and common mappings trivial. Hibernate ORM with
Panache focuses on making your entities trivial and fun to write in Quarkus.

When you open `Inventory.java` in `src/main/java/com/redhat/cloudnative/` as below, you will understand how easy to create a domain model
using Quarkus extension([Hibernate ORM with Panache](https://quarkus.io/guides/hibernate-orm-panache-guide)).

~~~java
@Entity
@Cacheable
public class Inventory extends PanacheEntity {

    public String itemId;
    public String location;
    public int quantity;
    public String link;

    public Inventory() {

    }

}
~~~

 * By extending `PanacheEntity` in your entities, you will get an ID field that is auto-generated. If you require a custom ID strategy, you can extend `PanacheEntityBase` instead and handle the ID yourself.

 * By using Use public fields, there is no need for functionless getters and setters (those that simply get or set the field). You simply refer to fields like Inventory.location without the need to write a Inventory.geLocation() implementation. Panache will
auto-generate any getters and setters you do not write, or you can develop your own getters/setters that do more than get/set, which will be called when the field is accessed directly.

The `PanacheEntity` superclass comes with lots of super useful static methods and you can add your own in your derived entity class, and much like traditional object-oriented programming it's natural and recommended to place custom queries as close to the entity as possible, ideally within the entity definition itself.
Users can just start using your entity Inventory by typing Inventory, and getting completion for all the operations in a single place.

When an entity is annotated with `@Cacheable`, all its field values are cached except for collections and relations to other entities.
This means the entity can be loaded without querying the database, but be careful as it implies the loaded entity might not reflect recent changes in the database.

Next, let's find out how `inventory service` exposes `RESTful APIs` on Quarkus. Open `InventoryResource.java` in `src/main/java/com/redhat/cloudnative/` and
you will see the following code sniffet.

The REST services defines two endpoints:

* `/api/inventory` that is accessible via `HTTP GET` which will return all known product Inventory entities as JSON
* `/api/inventory/<itemId>` that is accessible via `HTTP GET` at for example `/inventory/329199` with the last path parameter being the location which
we want to check its inventory status.

![inventory_service]({% image_path inventoryResource.png %})

`In Development`, we will configure to use local `in-memory H2 database` for local testing, as defined in `src/main/resources/application.properties`:

~~~java
quarkus.datasource.url=jdbc:h2:file://projects/database.db
quarkus.datasource.driver=org.h2.Driver
quarkus.datasource.username=inventory
quarkus.datasource.password=mysecretpassword
quarkus.datasource.max-size=8
quarkus.datasource.min-size=2
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.log.sql=false
~~~

Let's run the inventory application locally using `maven plugin command` via CodeReady Workspaces `Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/inventory-service/`

`mvn compile quarkus:dev`

You should see a bunch of log output that ends with:

![inventory_service]({% image_path inventory_mvn_compile.png %})

Open a `new` CodeReady Workspaces `Terminal` and invoke the RESTful endpoint using the following CURL commands. The output looks like here:

`curl http://localhost:8080/api/inventory ; echo`

`curl http://localhost:8080/api/inventory/329199 ; echo`

![inventory_service]({% image_path inventory_local_test.png %})

> `NOTE`: Make sure to stop Quarkus development mode via `Close` terminal.

`In production`, the inventory service will connect to `PostgeSQL` on `OpenShift` cluster.

We will use `Quarkus extension` to add `PostgreSQL JDBC Driver`. Go back to CodeReady Workspaces `Terminal` and run the following maven plugin:

`mvn quarkus:add-extension -Dextensions="jdbc-postgresql"`

Package the applicaiton via running the following maven plugin in CodeReady Workspaces`Terminal`:

`mvn clean package -DskipTests -Dquarkus.profile=prod`

> `NOTE`: You should `SKIP` the Unit test because you don't have PostgreSQL database in local environment.

Let's create a cloud-native applications projects in OpenShift cluster and deploy the inventory service as a Linux container.
To deploy applicaitons to OpenShift via `oc` tool, we need to copy login command and Login OpenShift cluster:

![codeready-workspace-copy-login-cmd]({% image_path codeready-workspace-oc-login-copy.png %}){:width="700px"}

Then you will redirect to OpenShift Login page again.

![openshift_login]({% image_path openshift_login.png %})

When you login with your credential, you will see `Display Token` link in the redirected page.

![openshift_login]({% image_path display_token_link.png %})

Click on the link and copy the `oc login` command:

![openshift_login]({% image_path your_token.png %})

Paste it on CodeReady Workspaces `Terminal` window.

Our production inventory microservice will use an external database (PostgreSQL) to house inventory data.
First, deploy a new instance of PostgreSQL by executing the following commands via CodeReady Workspaces `Terminal`:

 * Create a new project in OpenShift Cluster. You need to replace `userXX` with your username:

`oc project userXX-cloudnativeapps`

 * Deploy PostgreSQL to the project:

~~~shell
oc new-app -e POSTGRESQL_USER=inventory \
  -e POSTGRESQL_PASSWORD=mysecretpassword \
  -e POSTGRESQL_DATABASE=inventory openshift/postgresql:10 \
  --name=inventory-database
~~~

 * Build the image using on OpenShift:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=inventory -l app=inventory`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

  * Start and watch the build, which will take about minutes to complete:

`oc start-build inventory --from-file target/*-runner.jar --follow`

![inventory]({% image_path inventory-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done and override the Postgres URL to specify our production Postgres credentials:

`oc new-app inventory -e QUARKUS_DATASOURCE_URL=jdbc:postgresql://inventory-database:5432/inventory`

 * Create the route

`oc expose svc/inventory`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory`

Wait for that command to report replication controller `inventory-1` successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

* Get the route URL

`export URL="http://$(oc get route | grep inventory | awk '{print $2}')"`

`curl $URL/api/inventory ; echo`

You will see the following result:

~~~shell
[{"id":1,"itemId":"329299","link":"http://maps.google.com/?q=Raleigh","location":"Raleigh","quantity":736},{"id":2,"itemId":"329199","link":"http://maps.google.com/?q=Bost
on","location":"Boston","quantity":512},{"id":3,"itemId":"165613","link":"http://maps.google.com/?q=Seoul","location":"Seoul","quantity":256},{"id":4,"itemId":"165614","li
nk":"http://maps.google.com/?q=Singapore","location":"Singapore","quantity":54},{"id":5,"itemId":"165954","link":"http://maps.google.com/?q=London","location":"London","qu
antity":87},{"id":6,"itemId":"444434","link":"http://maps.google.com/?q=NewYork","location":"NewYork","quantity":443},{"id":7,"itemId":"444435","link":"http://maps.google.
com/?q=Paris","location":"Paris","quantity":600},{"id":8,"itemId":"444437","link":"http://maps.google.com/?q=Tokyo","location":"Tokyo","quantity":230}]
~~~

![openshift_login]({% image_path inventory_curl_result.png %})

So now `Inventory` service is deployed to OpenShift. You can also see it in the Project Status in the OpenShift Console
with its single replica running in 1 pod, along with the Postgres database pod.

####2. Deploying Catalog Service

---

`Catalog Service` serves products and prices for retail products. Lets's go through quickly how the catalog service works and built on
`Spring Boot` Java runtimes.  Go to `Project Explorer` in `CodeReady Workspaces` Web IDE and expand `catalog-service` directory.

![catalog]({% image_path codeready-workspace-catalog-project.png %}){:width="500px"}

First of all, we won't implement the catalog application to retrieve data because of all funtions are already built when we imported this project from Git server.
There're a few interesting things what we need to take a look at this Spring Boot application before we will deploy it to OpenShift cluster.

This catalog service is not using the default BOM (Bill of material) that Spring Boot projects typically use. Instead, we are using
a BOM provided by Red Hat as part of the [Snowdrop](http://snowdrop.me/) project.

~~~xml
<dependencyManagement>
<dependencies>
  <dependency>
    <groupId>me.snowdrop</groupId>
    <artifactId>spring-boot-bom</artifactId>
    <version>${spring-boot.bom.version}</version>
    <type>pom</type>
    <scope>import</scope>
  </dependency>
</dependencies>
</dependencyManagement>
~~~

![catalog]({% image_path catalog-pom.png %})

Also, catalog service calls the inventory service that we deployed earlier using REST to retrieve the inventory status and include that in the response.
Open `CatalogService.java` in `src/main/java/com/redhat/coolstore/service` directory via Project Explorer and how `read()` and `readAll()` method work:

![catalog]({% image_path catalog-service-codes.png %})

Build and deploy the project using the following command, which will use the maven plugin to deploy via CodeReady Workspaces `Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/catalog-service/`

`mvn clean package spring-boot:repackage -DskipTests`

The build and deploy may take a minute or two. Wait for it to complete. You should see a `BUILD SUCCESS` at the
end of the build output.

Our production catalog microservice will use an external database (PostgreSQL) to house inventory data.
First, deploy a new instance of PostgreSQL by executing via CodeReady Workspaces `Terminal`:

Make sure if the current project is `userXX-cloudnativeapps`.

 * Deploy PostgreSQL to the project:

~~~shell
oc new-app -e POSTGRESQL_USER=catalog \
    -e POSTGRESQL_PASSWORD=mysecretpassword \
    -e POSTGRESQL_DATABASE=catalog \
    openshift/postgresql:10 \
    --name=catalog-database
~~~

 * Build the image using on OpenShift:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=catalog -l app=catalog`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build catalog --from-file=target/catalog-1.0.0-SNAPSHOT.jar --follow`

![catalog]({% image_path catalog-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done and override the Postgres URL to specify our production Postgres credentials:

`oc new-app catalog`

 * Create the route

`oc expose service catalog`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/catalog`

Wait for that command to report replication controller `catalog-1` successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find a certain inventory:

* Get the route URL

`export URL="http://$(oc get route | grep catalog | awk '{print $2}')"`

`curl $URL/api/product/329299 ; echo`

You will see the following result:

`{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99,"quantity":736}`

![openshift_login]({% image_path catalog_curl_result.png %})

So now `Catalog` service is deployed to OpenShift. You can also see it in the Project Status in the OpenShift Console
with running 4 pods such as catalog, catalog-database, inventory, and inventory-database.

![catalog]({% image_path catalog-project-status.png %})

####3. Developing and Deploying Shopping Cart Service

---

By now, you have deployed some of the essential elements for the Coolstore application. However, an online shop without a cart means no checkout experience. In this section, we are going to implement the Shopping Cart; in our Microservice world, we are going to call it the `cart service` and our java artifact/repo is called the `cart-service`.

##### Let's get started!

In a nutshell, the Cart service is RESTful and built with Quarkus using the Red Hat's Distributed `Data Grid` technology.

##### What are the building blocks of the Shopping cart a.k.a cart-service?

It uses a Red Hat's Distributed `Data Grid` technology to store all shopping carts and assigns a unique id to them. It uses the `Quarkus Infinispan client` to do this.
The Shopping cart makes a call via the Quarkus Rest client to fetch all items in the Catalog. In the end, Shopping cart also throws out a `Kafka` message to the topic Orders, when checking out. For that, we use the `Quarkus Kafka client` in the next lab. Last and perhaps worth mentioning the `REST+Swagger UI` also part of the REST API support in `Quarkus`.

What is a `Shopping Cart` in our context? A Shopping cart has a list of Shopping Items. Quantity of a product in the Items list `Discount` and promotional details. We will see these in more details when we look at our model.

For this lab, we are using the code ready workspaces, make sure you have the following project open in your workspace. Lets's go through quickly how the cart service works and built on `Quarkus` Java runtimes.  Go to `Project Explorer` in `CodeReady Workspaces` Web IDE and expand `cart-service` directory.

![cart]({% image_path codeready-workspace-cart-project.png %}){:width="500px"}

##### Adding a distributed cache to our cart-service

We are going to use the Red hat Distributed `Data Grid` for caching all the users' carts.

`Red Hat® Distributed Data Grid` is an in-memory, distributed, NoSQL datastore solution. Your applications can access, process, and analyze data at in-memory speed to deliver a superior user experience with features and benefits as below:

* `Act faster` - Quickly access your data through fast, low-latency data processing using memory (RAM) and distributed parallel execution.

 * `Scale quickly` - Achieve linear scalability with data partitioning and distribution across cluster nodes.

 * `Always available`- Gain high availability through data replication across cluster nodes.

 * `Fault tolerance` - Attain fault tolerance and recover from disaster through cross-data center geo-replication and clustering.

 * `More productivity` - Gain development flexibly and higher productivity with a highly versatile, functionally rich NoSQL data store.

 * `Protect data` - Obtain comprehensive data security with encryption and role-based access.

Lets create a simple version of the cache service in our cluster. Open the Terminal in your CodeReady workspace and run the following command:

`oc new-app jboss/infinispan-server:10.0.0.Beta3 --name=datagrid-service`

>`NOTE`: This will create a single instance of infinispan server the community version of the DataGrid. At the time of writing this guide, the infinspan client for Quarkus does not work with DataGrid, and Quarkus itself is also a community project.

Once deployed you should see the newly created `datagrid-service` in your project dashboard as follows:

![cart]({% image_path cart-cache-pod.png %})

Now that our cache service a.k.a datagrid-service is deployed. We want to ensure that everything in our cart is persisted in this blazing fast cache. It will help us when we have a few million users per second on a black Friday.

Following is what we need to do:

* Model our data

* Choose how we store the data

* Create a marshaller for our data

* Inject our cache connection into the service

We have made this choice easier for you. The default serialization is done using a library based on `protobuf`. We need to define the protobuf schema and a marshaller for each user type(s).

Let's take a look at our `cart.proto` file in `META-INF`:

~~~java
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
~~~

* So our ShoppingCart has ShoppingCartItem

* ShoppingCartItem has Product

But we havent defined the `Product` yet. Lets go ahead and do that. Add this code to the `//TODO ADD Product` marker:

~~~java
message Product {
  required string itemId = 1;
  required string name = 2;
  required string desc = 3;
  required double price = 4;
}
~~~

`Great!`, now we have the Product defined in our proto model.
We should also ensure that this model also exists as `POJO`(Plain Old Java Object), that way our `REST Endpoint`, or `Cache` will be able to directly serialize and desrialize the data.

Lets open up our `Product.java` in package model:

~~~java
    private String itemId;
    private String name;
    private String desc;
    private double price;
~~~

Notice that the entities match our proto file. The rest or Getters and Setters, so we can read and write data into them.

Lets go ahead and create a `Marshaller `for our Product class which will do exactly what we intend, read and write to our cache.

Create a new Java class called `ProductMarshaller.java` in `com.redhat.cloudnative.model`

~~~java
package com.redhat.cloudnative.model;

import com.redhat.cloudnative.model.Product;
import org.infinispan.protostream.MessageMarshaller;

import java.io.IOException;

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
~~~

So now we have the capability to read from a `ProtoStream` and `Write` to it. And this will be done directly into our cache. We have already created the other model classes and mashallers, feel free to look around.

Now its time to configure our `RemoteCache`, since its not embedded into our service. Open the file `com.redhat.cloudnative.Producers`.

We use the producer to ensure our RemoteCache gets instantiated. We create methods called getCache and getConfigBuilder

* getConfigBuilder: sets up the basic cache config

* getCache, sets up our marshallers and proto files

* other config properties are injected at runtime

Add this code below the `// TODO Add getCache` and `// TODO add getConfigBuilder` marker:

~~~java
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
~~~

`Perfect`, now we have all the building blocks ready to use the cache. Lets start using our cache.

Next we need to make sure we will inject our cache in our service. Open `com.redhat.cloudnative.service.ShoppingCartServiceImpl` and add this at the `// TODO Inject RemoteCache` marker:

~~~java
    @Inject
    @Remote("default")
    RemoteCache<String, ShoppingCart> carts;
~~~

##### Building `cart-service` REST API with Quarkus

The cart is quite simple; All the information from the browser i.e., via our `Angular App` is via `JSON` at the `/api/cart` endpoint:

* `GET` request `/{cartId}` gets the items in the cart, or creates a new unique ID if one is not present
* `POST` to `/{cartId}/{itemId}/{quantity}` will add items to the cart
* `DELETE` to `/{cartId}/{itemId}/{quantity}` will remove items from the cart. * And finally a `POST` to `/checkout/{cartId}` will remove the items and invoke the checkout procedure

Let's take a look at how we do this with Quarkus. In our `cart-service` project and in our main package i.e., `com.redhat.cloudnative` is the `CartResource`. Let's take a look at the getCart method.

At the `// TODO ADD getCart method` marker, add this method:

~~~java
    public ShoppingCart getCart(@PathParam("cartId") String cartId) {
        return shoppingCartService.getShoppingCart(cartId);
    }
~~~

The code above is using the `ShoppingCartService`, which is injected into the `CartResource` via the Dependency Injection. The `ShoppingCartService` take a `cartId` as a parameter and returns the associated ShoppingCart. So that's perfect, however, for our Endpoint i.e., `CartResource` to respond, we need to define a couple of things:

 * The type of HTTPRequest

 * The type of data it can receive

 * The path it resolves too

 Add the following code on top of the `getCart` method

~~~java
    @GET
    @Produces(MediaType.TEXT_PLAIN)
    @Path("/{cartId}")
    @Operation(summary = "get the contents of cart by cartId")
~~~

We have now successfully stated that the method adheres to a GET request and accepts data in `plain text`. The path would be `/api/cart/{cartId}`
finally, we add the `@Operation` annotation for some documentation, which is important for other developers using our service.

Take this opportunity to look at some of the other methods. You will find `@POST` and `@DELETE` and also the paths they adhere too. This is how we can construct a simple Endpoint for our application.


##### Package and Deploy the cart-service

Package the cart application via clicking on `Package for OpenShift` in `Commands Palette`:

![cart]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following maven plugin in CodeReady Workspaces`Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/cart-service/`

`mvn clean package -DskipTests`

Build the image using on OpenShift:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=cart -l app=cart`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

  * Start and watch the build, which will take about minutes to complete:

`oc start-build cart --from-file target/*-runner.jar --follow`

![cart]({% image_path cart-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done:

`oc new-app cart`

 * Create the route

`oc expose svc/cart`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/cart`

Wait for that command to report replication controller `cart-1` successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don`t have any liveness check configured.

With the app deployed, we can check out the API page that Quarkus generates.

Run this command in the CodeReady Terminal to discover the URL to the app:

`echo http://$(oc get route cart -o=go-template --template='{{ .spec.host }}')/swagger-ui`

Open this URL in your browser!

![cart]({% image_path cart-swagger-ui.png %})

Notice that the documentation after the methods, this is an excellent way for other service developers to know what you intend to do with each service method. You can try to invoke the methods and see the output from the service. Hence an excellent way to test quickly as well.

####4. Developing and Deploying Order Service

---

`Order Service` manages all orders when customers checkout items in the shopping cart. Lets's go through quickly how the order service get
`REST` services to use the `MongoDB` database with `Quarkus` Java runtimes. Go to `Project Explorer` in `CodeReady Workspaces` Web IDE and
expand `order-service` directory.

![catalog]({% image_path codeready-workspace-order-project.png %}){:width="500px"}

The application built in `Quarkus` is quite simple: the user can add elements in a list using `RESTful APIs` and the list is updated.
All the information between the client and the server are formatted as `JSON`. The elements are stored in `MongoDB`.

##### Adding Maven Dependencies using Quarkus Extensions

Execute the following command via CodeReady Workspaces `Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/order-service/`

`mvn quarkus:add-extension -Dextensions="resteasy-jsonb,mongodb-client"`

This command generates a Maven structure importing the RESTEasy/JAX-RS, JSON-B and MongoDB Client extensions. After this,
the quarkus-mongodb-client extension has been added to your `pom.xml`.

![catalog]({% image_path order-pom-dependency.png %})

##### Creating Order Service using JSON REST service

First, let’s have a look at the `Order` bean in `src/main/java/com/redhat/cloudnative/`as follows:

![openshift_login]({% image_path order_bean.png %}){:width="700px"}

Nothing fancy. One important thing to note is that having a default constructor is required by the `JSON serialization layer`.

Now, open a `com.redhat.cloudnative.OrderService` that will be the business layer of our application and `store/load` the orders from the mongoDB database.
Add the following Java codes at each market.

 * `// TODO: Inject MongoClient here` marker:

~~~java
@Inject MongoClient mongoClient;
~~~

 * `// TODO: Add a while loop to make an order lists using MongoCursor here` marker in `list()` method:

~~~java
MongoCursor<Document> cursor = getCollection().find().iterator();

try {
    while (cursor.hasNext()) {
        Document document = cursor.next();
        Order order = new Order();
        order.setId(document.getString("id"));
        order.setName(document.getString("name"));
        order.setTotal(document.getString("total"));
        order.setCcNumber(document.getString("ccNumber"));
        order.setCcExp(document.getString("ccExp"));
        order.setBillingAddress(document.getString("billingAddress"));
        order.setStatus(document.getString("status"));
        list.add(order);
    }
} finally {
    cursor.close();
}
~~~

 * `// TODO: Add to create a Document based order here` marker in `add(Order order)` method:

~~~java
Document document = new Document()
        .append("id", order.getId())
        .append("name", order.getName())
        .append("total", order.getTotal())
        .append("ccNumber", order.getCcNumber())
        .append("ccExp", order.getCcExp())
        .append("billingAddress", order.getBillingAddress())
        .append("status", order.getStatus());
getCollection().insertOne(document);
~~~

Now, edit the `com.redhat.cloudnative.OrderResource` class as follows in each marker:

 * `// TODO: Add JAX-RS annotations here` marker:

~~~java
@Path("/api/orders")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
~~~

 * `// TODO: Inject OrderService here` marker:

~~~java
@Inject OrderService orderService;
~~~

 * `// TODO: Add list(), add(), updateStatus() methods here` marker:

~~~java
@GET
public List<Order> list() {
    return orderService.list();
}

@POST
public List<Order> add(Order order) {
    orderService.add(order);
    return list();
}

@GET
@Path("/{orderId}/{status}")
public List<Order> updateStatus(@PathParam("orderId") String orderId, @PathParam("status") String status) {
    orderService.updateStatus(orderId, status);
    return list();
}
~~~

The implementation is pretty straightforward and you just need to define your endpoints using the `JAX-RS annotations` and
use the `OrderService` to list/add new orders.

##### Configuring the MongoDB database

The main property to configure is the URL to access to `MongoDB,` almost all configuration can be included in the connection URI
so we advise you to do so, you can find more information in the [MongoDB documentation](https://docs.mongodb.com/manual/reference/connection-string/)

Open `application.properties` in `src/main/resources/` and add the following configuration:

`quarkus.mongodb.connection-string = mongodb://order-database:27017`

![order]({% image_path order_application_properties.png %})

##### Simplifying MongoDB Client usage using BSON codec

By using a Bson `Codec`, the MongoDB Client will take care of the transformation of your domain object to/from a MongoDB `Document` automatically.

First you need to create a Bson `Codec` that will tell Bson how to transform your entity to/from a MongoDB `Document`.
Here we use a `CollectibleCodec` as our object is retrievable from the database (it has a MongoDB identifier), if not we would have used a `Codec` instead.
More information in the [codec documentation](https://mongodb.github.io/mongo-java-driver/3.10/bson/codecs).

Edit the `com.redhat.cloudnative.codec.OrderCodec` class as follows:

 * `// TODO: Add Encode & Decode contexts here` marker:

~~~java
@Override
public void encode(BsonWriter writer, Order Order, EncoderContext encoderContext) {
    Document doc = new Document();
    doc.put("id", Order.getId());
    doc.put("name", Order.getName());
    doc.put("total", Order.getTotal());
    doc.put("ccNumber", Order.getCcNumber());
    doc.put("ccExp", Order.getCcExp());
    doc.put("billingAddress", Order.getBillingAddress());
    doc.put("status", Order.getStatus());
    documentCodec.encode(writer, doc, encoderContext);
}

@Override
public Class<Order> getEncoderClass() {
    return Order.class;
}

@Override
public Order generateIdIfAbsentFromDocument(Order document) {
    if (!documentHasId(document)) {
        document.setId(UUID.randomUUID().toString());
    }
    return document;
}

@Override
public boolean documentHasId(Order document) {
    return document.getId() != null;
}

@Override
public BsonValue getDocumentId(Order document) {
    return new BsonString(document.getId());
}

@Override
public Order decode(BsonReader reader, DecoderContext decoderContext) {
    Document document = documentCodec.decode(reader, decoderContext);
    Order order = new Order();
    if (document.getString("id") != null) {
        order.setId(document.getString("id"));
    }
    order.setName(document.getString("name"));
    order.setTotal(document.getString("total"));
    order.setCcNumber(document.getString("ccNumber"));
    order.setCcExp(document.getString("ccExp"));
    order.setBillingAddress(document.getString("billingAddress"));
    order.setStatus(document.getString("status"));
    return order;
}
~~~

Then you need to create a `CodecProvider` to link this `Codec` to the Order class.

Edit the `com.redhat.cloudnative.codec.OrderCodecProvider` class as follows:

 * `// TODO: Add Codec get method here` marker:

~~~java
@Override
public <T> Codec<T> get(Class<T> clazz, CodecRegistry registry) {
    if (clazz == Order.class) {
        return (Codec<T>) new OrderCodec();
    }
    return null;
}
~~~

`Quarkus` will register the `CodecProvider` for you.

Finally, when getting the `MongoCollection` from the database you can use directly the `Order` class instead of the `Document` one,
the codec will automatically map the `Document` to/from your `Order` class.

Edit the `com.redhat.cloudnative.CodecOrderService` class as follows:

 * `// TODO: Add MongoCollection method here` marker:

~~~java
private MongoCollection<Order> getCollection(){
    return mongoClient.getDatabase("order").getCollection("order", Order.class);
}
~~~

##### Building and Deploying Application to OpenShift

Package the cart application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following maven plugin in CodeReady Workspaces`Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/order-service/`

`mvn clean package -DskipTests`

![order]({% image_path order-mvn-package.png %})

##### Deploying Order service with MongoDB to OpenShift

Run the following `oc` command to deploy a `MongoDB` to OpenShift via CodeReady Workspaces `Terminal`:

`oc new-app --docker-image mongo:4.0 --name=order-database`

Once the MongoDB is deployed successfully, it will be showd in `Project Status`.

![order]({% image_path order-monogo-status.png %})

Build the image using on OpenShift:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=order -l app=order`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

 * Start and watch the build, which will take about minutes to complete:

`oc start-build order --from-file target/*-runner.jar --follow`

![order]({% image_path order-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done:

`oc new-app order`

 * Create the route

`oc expose svc/order`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/order`

Wait for that command to report replication controller `order-1` successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

* Get the route URL

`export URL="http://$(oc get route | grep order | awk '{print $2}')"`

`curl $URL/api/orders ; echo`

You will see empty result because you didn't add any shopping items yet:

`[]`

####5. Deploying WEB-UI Service

---

`WEB-UI Service` serves a frontend based on [AngularJS](https://angularjs.org/) and [PatternFly](http://patternfly.org/) running in a
[Node.js](https://access.redhat.com/documentation/en/openshift-container-platform/3.3/paged/using-images/chapter-2-source-to-image-s2i) container.
[Red Hat OpenShift Application Runtimes](https://developers.redhat.com/products/rhoar) includes `Node.js` support in enterprise prouction environment.

Lets's go through quickly how the frontend service works and built on `Node.js` runtimes. Go to `Project Explorer` in `CodeReady Workspaces` Web IDE
and expand `coolstore-ui` directory.

![coolstore-ui]({% image_path codeready-workspace-coolstore-ui.png %}){:width="500px"}

You will see javascripts for specific cloud-native services such as cart, catatlog, and order service as above.

Now, we will deploy a presentation layer to OpenShift cluster using `Nodeshift` command line tool, a programmable API that you can use to deploy Node.js projects to `OpenShift`.

 * Install the `Nodeshift` tool via CodeReady Workspaces `Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/coolstore-ui/`

`npm install --save-dev nodeshift`

 * Deploy the web-ui service using `Nodeshift` via CodeReady Workspaces `Terminal` and it will take a couple of minutes to complete the web-ui application deployment:

`npm run nodeshift`

![coolstore-ui]({% image_path coolstore-ui-deploy.png %})

 * Create the route

`oc expose svc/coolstore-ui`

Go to `Networking > Routes` in OpenShift web console and click on the route URL of `coolstore-ui`:

![coolstore-ui]({% image_path web-ui-route.png %})

You will see the prouct page of `Red Hat Cool Store` as below:

![coolstore-ui]({% image_path web-ui-landing.png %})

#### Summary

In this scenario we developed five microservices with `REST API` exposure to communicate with the other microservices. We also used a variety of application
runtimes such as `Quarkus`, `Spring Boot`, and `NodeJS` to compile, package, and containerize applications which is a major capability of the advanced cloud-native architecture.

To deploy the cloud-native applications with multiple datasources on `OpenShift` cluster, `Quarkus` provides an easy way to connect multiple datasources and
obtain a reference to those datasources such as `PostgreSQL` and `MongoDB` in code.

In the end, we optimized `data transaction performance` of the shopping cart service thru integrating with a `JBoss Data Grid`
to increase end users'(customers) satification. `Congratulations!`
