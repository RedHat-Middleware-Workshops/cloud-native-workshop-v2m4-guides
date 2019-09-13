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

 * Create a temp directory to store only previously-built application with necessary lib directory:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build inventory --from-dir=target/binary --follow`

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

`Shopping Cart Service` manages shopping cart for each customer. Lets's go through quickly how the cart service works and built on 
`Quarkus` Java runtimes.  Go to `Project Explorer` in `CodeReady Workspaces` Web IDE and expand `cart-service` directory.

![cart]({% image_path codeready-workspace-cart-project.png %}){:width="500px"}

Let's figure out the `basic structure of Quarkus project` in terms of how `models, services, and RESTful APIs` are already implemented to serve 
functions of the shopping cart service. 

Open `CartResource.java` in `src/main/java/com/redhat/cloudnative/` to understand how this application exposes `REST` endpoints on Quarkus in simple codes.
And you will see that `POST`, `GET` methods exist to serve the shopping cart service in terms of creating a new shopping cart, get details of a certain cart, and delete an existing cart.

 * `@Path`, `@GET` and `@PathParam` are the standard `JAX-RS annotations` used to define how to access the service.
 * `@Inject` is Standard CDI annotation to use `ShoppingCartService` for creating, updating, deleting `ShoppingCart`.

![catalog]({% image_path cart-resource.png %})

You can also have a look at `PromotionService`, `ShippingService` in `src/main/java/com/redhat/cloudnative/service` to understand how to 
define `@ApplicationScoped` in each funtion service.

![cart]({% image_path cart-services-code.png %})

##### Developing Quarkus Infinispan Client

We confirmed that the cart service serves designed CRUD funtions as we expected. Now, we will transform into `high-performing cache service` by 
addin a `In Memory Data Grid` server that allows running in a server outside of application processes. More importantly, we will learn how 
`Quarkus Infinispan Client` extension provides functionality to allow the client that can connect to the data grid server when running in Quarkus.

Once you have the cart service built on Quarkus project configured you can add the `infinispan-client extension` to the project 
by running the following from the command line in your project base directory via CodeReady Workspaces `Terminal`:

`mvn quarkus:add-extension -Dextensions="infinispan-client"`

This will add the following to your `pom.xml`.

![cart]({% image_path catalog-pom-infinispan-client.png %})

Now, let's create Cache Services using `cart-cache` of `Red Hat JBoss Data Grid` to quickly set up clusters that give you optimal performance 
and ease of use with minimal configuration.

`Cache Service`(cart-cache) provides an easy-to-use implementation of `Data Grid` for `OpenShift` that is designed to increase application 
response time through high-performance caching. With cart-cache you can create new caches only as copies of the default cache definition.

The Infinispan client is configurable in the `application.properties` file that can be provided in the `src/main/resources` directory. 
These are the properties that can be configured in this file:

`quarkus.infinispan-client.server-list=cart-cache:11222`

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

![cart]({% image_path catalog-cart-proto.png %})

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

![cart]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following maven plugin in CodeReady Workspaces`Terminal`:

`cd /projects/cloud-native-workshop-v2m4-labs/cart-service/`

`mvn clean package -DskipTests`

##### Deploying Cart service with JBoss Data Grid to OpenShift

Run the following `oc` command to create a `cart-cache` in OpenShift via CodeReady Workspaces `Terminal`:

`oc new-app jboss/infinispan-server:10.0.0.Beta3 --name=datagrid-service`

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

Once the cart-cache is deployed successfully, it will be showd in `Workloads > Pods`.

![cart]({% image_path datagrid-pod.png %})

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
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

* Get the route URL

`export URL="http://$(oc get route | grep cart | awk '{print $2}')"`

`curl -X POST $URL/cart/1111/329199/50 ; echo`

`curl h$URL/cart/1111 ; echo`

You will see the following result:

~~~shell
bla bla
~~~

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

`Quarkus` will  register the `CodecProvider` for you.

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

ackage the cart application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Or run the following maven plugin in CodeReady Workspaces`Terminal`:

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

 * Create a temp directory to store only previously-built application with necessary lib directory:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

 * Start and watch the build, which will take about minutes to complete:

`oc start-build order --from-dir=target/binary --follow`

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