## Lab1 - Creating High-performing Cache Services

In this lab, we'll develop 5 microservices into the cloud-native appliation architecture. These cloud-native applications
will be allowed to communicate with athentication by `Single Sign-On` server. To do that, we will configure how to `authenticate REST API requests` 
across services. In the end, we will optimize `data transaction performance` of the shopping cart service thru integrating with a `Cache(Data Grid) server` 
to increase end users'(customers) satification. And there's more fun facts how easy it is to deploy applications on OpenShift 4 via `oc` command line tool.

#### Goals of this lab

---


The goal is to develop advanced cloud-native applications on `Red Hat Application Runtimes` and deploy them on `OpenShift 4` including 
`single sign-on access management` and `distributed cache manageemnt`. After this lab, you should end up with something like:

![goal]({% image_path module4-goal.png %})

####1. Deploying Inventory Service

---

`Inventory Service` serves inventory and availability data for retail products. Lets's go through quickly how inventory service works and built on 
`Quarkus` Java runtimes.  Go to `Project Explorer` in `CodeReady Workspaces` Web IDE and expand `inventory-service` directory.

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

* `/inventory` that is accessible via `HTTP GET` which will return all known product Inventory entities as JSON
* `/inventory/<itemId>` that is accessible via `HTTP GET` at for example `/inventory/329199` with the last path parameter being the location which 
we want to check its inventory status.

![inventory_service]({% image_path inventoryResource.png %})

`In Development`, we will configure to use local `in-memory H2 database` for local testing, as defined in `src/main/resources/application.properties`:

~~~java
quarkus.datasource.url=jdbc:h2:file://projects/database.db
quarkus.datasource.driver=org.h2.Driver
~~~

Let's run the inventory application locally using `maven plugin command` via CodeReady Workspaces `Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/inventory-service/`

`mvn compile quarkus:dev`

You should see a bunch of log output that ends with:

![inventory_service]({% image_path inventory_mvn_compile.png %})

Open a `new` CodeReady Workspaces `Terminal` and invoke the RESTful endpoint using the following CURL commands. The output looks like here:

`curl http://localhost:8080/services/inventory ; echo`
`curl http://localhost:8080/services/inventory/329199 ; echo`

![inventory_service]({% image_path inventory_local_test.png %})

> `NOTE`: Make sure to stop Quarkus development mode via `Close` terminal.

`In production`, the inventory service will connect to `PostgeSQL` on `OpenShift` cluster.

We will use `Quarkus extension` to add `PostgreSQL JDBC Driver`. Go back to CodeReady Workspaces `Terminal` and run the following maven plugin:

`mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-jdbc-postgresql"`

Then, modify `quarkus.datasource.url, quarkus.datasource.drive` variables in `src/main/resources/application.properties` as below:

~~~java
quarkus.datasource.url=jdbc:postgresql:inventory
quarkus.datasource.driver=org.postgresql.Driver
~~~

![inventory_service]({% image_path inventory_update_properties.png %})

Package the applicaiton via running the following maven plugin in CodeReady Workspaces`Terminal`:

`mvn clean package -DskipTests`

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

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=inventory-service -l app=inventory-service`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

 * Create a temp directory to store only previously-built application with necessary lib directory:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build inventory-service --from-dir=target/binary --follow`

![inventory]({% image_path inventory-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done and override the Postgres URL to specify our production Postgres credentials:

`oc new-app inventory-service -e QUARKUS_DATASOURCE_URL=jdbc:postgresql://inventory-database:5432/inventory`

 * Create the route

`oc expose svc/inventory-service`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory-service`

Wait for that command to report replication controller "inventory-service-1" successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

* Get the route URL

`export URL="http://$(oc get route | grep inventory-service | awk '{print $2}')"`

`curl $URL/services/inventory ; echo`

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

`Catalog Service` serves products and prices for retail products. Lets's go through quickly how catalog service works and built on 
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

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=catalog-service -l app=catalog-service`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build catalog-service --from-file=target/catalog-1.0.0-SNAPSHOT.jar --follow`

![catalog]({% image_path catalog-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done and override the Postgres URL to specify our production Postgres credentials:

`oc new-app catalog-service`

 * Create the route

`oc expose service catalog-service`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/catalog-service`

Wait for that command to report replication controller "catalog-service-1" successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find a certain inventory:

* Get the route URL

`export URL="http://$(oc get route | grep catalog-service | awk '{print $2}')"`

`curl $URL/services/product/329299 ; echo`

You will see the following result:

`{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99,"quantity":736}`

![openshift_login]({% image_path catalog_curl_result.png %})

So now `Catalog` service is deployed to OpenShift. You can also see it in the Project Status in the OpenShift Console 
with running 4 pods such as catalog-service, catalog-database, inventory-service, and inventory-database.

![catalog]({% image_path catalog-project-status.png %})

####3. Developing and Deploying Shopping Cart Service

---

`Shopping Cart Service` manages shopping cart for each customer. Lets's go through quickly how cart service works and built on 
`Quarkus` Java runtimes.  Go to `Project Explorer` in `CodeReady Workspaces` Web IDE and expand `cart-service` directory.

![catalog]({% image_path codeready-workspace-cart-project.png %}){:width="500px"}

Let's figure out the `basic structure of Quarkus project` in terms of how `models, services, and RESTful APIs` are already implemented to serve 
functions of the shopping cart service. 

Open `CartResource.java` in `src/main/java/com/redhat/cloudnative/` to understand how this application exposes `REST` endpoints on Quarkus in simple codes.
And you will see that `POST`, `GET` methods exist to serve the shopping cart service in terms of creating a new shopping cart, get details of a certain cart, and delete an existing cart.

 * `@Path`, `@GET` and `@PathParam` are the standard `JAX-RS annotations` used to define how to access the service.
 * `@Inject` is Standard CDI annotation to use `ShoppingCartService` for creating, updating, deleting `ShoppingCart`.

![catalog]({% image_path cart-resource.png %})

You can also have a look at `PromotionService`, `ShippingService` in `src/main/java/com/redhat/cloudnative/service` to understand how to 
define `@ApplicationScoped` in each funtion service.

![catalog]({% image_path cart-services-code.png %})

Let's run the cart-service application locally to test if fuctions work correctly. Run this Quarkus application as 
`Development Mode` using `quarkus-maven-plugin` or click on `Build and Run Locally` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-paletter.png %})

`cd /projects/cloud-native-workshop-v2m4-labs/cart-service`

`mvn compile quarkus:dev`

Then, access the following endpoints using `curl` and the result looks like:

`curl -X POST http://localhost:8080/cart/1111/329199/50`

`curl http://localhost:8080/cart/1111`

~~~java
bla bla bla
~~~

##### Developing Quarkus Infinispan Client

We confirmed that the cart service serves designed CRUD funtions as we expected. Now, we will transform into `high-performing cache service` by 
addin a `In Memory Data Grid` server that allows running in a server outside of application processes. More importantly, we will learn how 
`Quarkus Infinispan Client` extension provides functionality to allow the client that can connect to the data grid server when running in Quarkus.

Once you have the cart service built on Quarkus project configured you can add the `infinispan-client extension` to the project 
by running the following from the command line in your project base directory via CodeReady Workspaces `Terminal`:

`mvn quarkus:add-extension -Dextensions="infinispan-client"`

This will add the following to your `pom.xml`.

![codeready-workspace-maven]({% image_path catalog-pom-infinispan-client.png %})

Now, let's create Cache Services using `cache-service` of `Red Hat JBoss Data Grid` to quickly set up clusters that give you optimal performance 
and ease of use with minimal configuration.

`Cache Service`(cache-service) provides an easy-to-use implementation of `Data Grid` for `OpenShift` that is designed to increase application 
response time through high-performance caching. With cache-service you can create new caches only as copies of the default cache definition.

The Infinispan client is configurable in the `application.properties` file that can be provided in the `src/main/resources` directory. 
These are the properties that can be configured in this file:

`quarkus.infinispan-client.server-list=cache-service:11222`

It is also possible to configure a `hotrod-client.properties` as described in the Infinispan user guide. Note that the `hotrod-client.properties` values 
overwrite any matching property from the other configuration values(eg. near cache).

By default the client will support keys and values of the following types: byte[], primitive wrappers (eg. Integer, Long, Double etc.), String, Date and Instant. 
User types require some additional steps that are detailed here. Let’s say we have the following user classes in cart service:

 * `Product`

 * `Promotion`

 * `ShoppingCart`

 * `ShoppingCartItem`

The default serialization is done using a library based on `protobuf`. We need to define the proto buf schema and a marshaller for each user type(s).

> `NOTE`: Annotation based proto stream marshalling is not yet supported in the Quarkus Infinispan client. This will be added soon, allowing you to only annotate your classes, skipping the following steps.

We already have`cart.proto` in the `META-INF directory` of the cart project. These files will automatically be picked up at initialization time.

![catalog]({% image_path catalog-cart-proto.png %})

The next thing to do is to provide a `org.infinispan.protostream.MessageMarshaller` implementation for each user class defined in the proto schema. 
This class is then provided via `@Produces` in a similar fashion to the code based proto schema definition above.

Here is the Marshaller class for our `Product`, `Promotion`, `ShoppingCart`, and `ShoppingCartItem` classes.

 * Create `ProductMarshaller.java` in `src/main/java/com/redhat/cloudnative/model` and copy the following codes to the Java file.

~~~java
package com.redhat.cloudnative.model;

import com.redhat.cloudnative.model.Product;
import org.infinispan.protostream.MessageMarshaller;

import java.io.IOException;

public class ProductMarshaller implements MessageMarshaller<Product> {

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

 * Create `PromotionMarhsaller.java` in `src/main/java/com/redhat/cloudnative/model` and copy the following codes to the Java file.

~~~java
package com.redhat.cloudnative.model;

import com.redhat.cloudnative.model.Promotion;
import org.infinispan.protostream.MessageMarshaller;

import java.io.IOException;

public class PromotionMarhsaller implements MessageMarshaller<Promotion> {

    @Override
    public Promotion readFrom(ProtoStreamReader reader) throws IOException {
        String itemId = reader.readString("itemId");
        double percentOff = reader.readDouble("percentOff");
        return new Promotion(itemId, percentOff);
    }

    @Override
    public void writeTo(ProtoStreamWriter writer, Promotion promotion) throws IOException {
        writer.writeString("itemId", promotion.getItemId());
        writer.writeDouble("percentOff", promotion.getPercentOff());
    }

    @Override
    public Class<? extends Promotion> getJavaClass() {
        return Promotion.class;
    }

    @Override
    public String getTypeName() {
        return "coolstore.Promotion";
    }
}
~~~

 * Create `ShoppingCartItemMarshaller.java` in `src/main/java/com/redhat/cloudnative/model` and copy the following codes to the Java file.

~~~java
package com.redhat.cloudnative.model;

import com.redhat.cloudnative.model.Product;
import com.redhat.cloudnative.model.ShoppingCartItem;
import org.infinispan.protostream.MessageMarshaller;

import java.io.IOException;

public class ShoppingCartItemMarshaller implements MessageMarshaller<ShoppingCartItem> {

    @Override
    public ShoppingCartItem readFrom(ProtoStreamReader reader) throws IOException {
        double price = reader.readDouble("price");
        int quantity = reader.readInt("quantity");
        double promoSavings = reader.readDouble("promoSavings");
        Product product = reader.readObject("product", Product.class);

        return new ShoppingCartItem(price, quantity, promoSavings, product);
    }

    @Override
    public void writeTo(ProtoStreamWriter writer, ShoppingCartItem shoppingCartItem) throws IOException {
        writer.writeDouble("price", shoppingCartItem.getPrice());
        writer.writeInt("quantity", shoppingCartItem.getQuantity());
        writer.writeDouble("promoSavings", shoppingCartItem.getPromoSavings());
        writer.writeObject("product", shoppingCartItem.getProduct(), Product.class);
    }

    @Override
    public Class<? extends ShoppingCartItem> getJavaClass() {
        return ShoppingCartItem.class;
    }

    @Override
    public String getTypeName() {
        return "coolstore.ShoppingCartItem";
    }
}
~~~

 * Create `ShoppingCartMarshaller.java` in `src/main/java/com/redhat/cloudnative/model` and copy the following codes to the Java file.

~~~java
package com.redhat.cloudnative.model;

import com.redhat.cloudnative.model.ShoppingCart;
import com.redhat.cloudnative.model.ShoppingCartItem;
import org.infinispan.protostream.MessageMarshaller;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class ShoppingCartMarshaller implements MessageMarshaller<ShoppingCart> {

    @Override
    public ShoppingCart readFrom(ProtoStreamReader reader) throws IOException {
        double cartItemTotal = reader.readDouble("cartItemTotal");
        double cartItemPromoSavings = reader.readDouble("cartItemPromoSavings");
        double shippingTotal = reader.readDouble("shippingTotal");
        double shippingPromoSavings = reader.readDouble("shippingPromoSavings");
        double cartTotal = reader.readDouble("cartTotal");
        String cartId = reader.readString("cartId");
        List<ShoppingCartItem> shoppingCartItemList = new ArrayList<>();
        shoppingCartItemList = reader.readCollection("shoppingCartItemList", shoppingCartItemList, ShoppingCartItem.class);

        return new ShoppingCart(cartItemTotal, cartItemPromoSavings, shippingTotal, shippingPromoSavings, cartTotal, cartId, shoppingCartItemList);
    }

    @Override
    public void writeTo(ProtoStreamWriter writer, ShoppingCart shoppingCart) throws IOException {
        writer.writeDouble("cartItemTotal", shoppingCart.getCartItemTotal());
        writer.writeDouble("cartItemPromoSavings", shoppingCart.getCartItemPromoSavings());
        writer.writeDouble("shippingTotal", shoppingCart.getShippingTotal());
        writer.writeDouble("shippingPromoSavings", shoppingCart.getShippingPromoSavings());
        writer.writeDouble("cartTotal", shoppingCart.getCartTotal());
        writer.writeString("cartId", shoppingCart.getCartId());
        writer.writeCollection("shoppingCartItemList", shoppingCart.getShoppingCartItemList(), ShoppingCartItem.class);
    }

    @Override
    public Class<? extends ShoppingCart> getJavaClass() {
        return ShoppingCart.class;
    }

    @Override
    public String getTypeName() {
        return "coolstore.ShoppingCart";
    }

}
~~~

 * Create `MarshallerConfig.java` in `src/main/java/com/redhat/cloudnative/model` and copy the following codes to the Java file.

~~~java
package com.redhat.cloudnative.model;

import org.infinispan.protostream.MessageMarshaller;

import javax.enterprise.context.ApplicationScoped;
import javax.enterprise.inject.Produces;

@ApplicationScoped
public class MarshallerConfig {

    @Produces
    MessageMarshaller promotionMarshaller(){
        return new PromotionMarhsaller();
    }

    @Produces
    MessageMarshaller productMarshaller(){
        return new ProductMarshaller();
    }

    @Produces
    MessageMarshaller shoppingCartItemMarhsaller(){
        return new ShoppingCartItemMarshaller();
    }

    @Produces
    MessageMarshaller shoppingCartMarshaller(){
        return new ShoppingCartMarshaller();
    }

}
~~~

Add `Dependency Injection` to `ShoppingCartServiceImpl` class as below:

~~~java
bla bla or cp pre-built codes from somewhere in the repo
~~~

Package the cart application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Or runthe following maven plugin in CodeReady Workspaces`Terminal`:

`mvn clean package -DskipTests`

##### Deploying Cart service with JBoss Data Grid to OpenShift

Run the following `oc` command to create a `cache-service` in OpenShift via CodeReady Workspaces `Terminal`:

`oc new-app cache-service -p APPLICATION_USER=developer -p APPLICATION_PASSWORD=developer -p NUMBER_OF_INSTANCES=3 -p REPLICATION_FACTOR=2`

 * `NUMBER_OF_INSTANCES` sets the number of nodes in the Data Grid for OpenShift cluster. The default is 1.

 * `APPLICATION_USER` creates a user to securely access the cache. There is no default value. You must always create a user.
 
 * `APPLICATION_PASSWORD` specifies a password for the user. If you do not set a password, the service template randomly generates one and stores it as a secret.
 
 * `REPLICATION_FACTOR` specifies the number of copies for each cache entry. The default is 1.

[Red Hat® Data Grid](https://access.redhat.com/documentation/en-us/red_hat_data_grid/7.3/html-single/red_hat_data_grid_for_openshift/index) is an 
in-memory, distributed, NoSQL datastore solution. Your applications can access, process, and analyze data at in-memory speed to deliver a superior 
user experience with features and benefits as below:

 * `Act faster` - Quickly access your data through fast, low-latency data processing using memory (RAM) and distributed parallel execution.

 * `Scale quickly` - Achieve linear scalability with data partitioning and distribution across cluster nodes.

 * `Always available` - Gain high availability through data replication across cluster nodes.

 * `Fault tolerance` - Attain fault tolerance and recover from disaster through cross-datacenter georeplication and clustering.

 * `More productivity` - Gain development flexibly and greater productivity with a highly versatile, functionally rich NoSQL data store.

 * `Protect data` - Obtain comprehensive data security with encryption and role-based access.

You can also create a cache service when you go to `Catalog > Developer Catalog` in OpenShift web console and select `data grid service`.

![catalog]({% image_path catalog-cache-service.png %})

Once the cache-service is deployed successfully, it will be showd in `Project Status`.

![catalog]({% image_path catalog-cache-service-status.png %})

Build the image using on OpenShift:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=cart-service -l app=cart-service`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

 * Create a temp directory to store only previously-built application with necessary lib directory:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build cart-service --from-dir=target/binary --follow`

![inventory]({% image_path cart-build-logs.png %})

 * Deploy it as an OpenShift application after the build is done:

`oc new-app cart-service`

 * Create the route

`oc expose svc/cart-service`

 * Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/cart-service`

Wait for that command to report replication controller "cart-service-1" successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

* Get the route URL

`export URL="http://$(oc get route | grep cart-service | awk '{print $2}')"`

`curl -X POST $URL/cart/1111/329199/50 ; echo`

`curl h$URL/cart/1111 ; echo`

You will see the following result:

~~~shell
bla bla
~~~

![openshift_login]({% image_path inventory_curl_result.png %})

####4. Developing and Deploying Order Service

---

####5. Developing and Deploying Payment Service

---