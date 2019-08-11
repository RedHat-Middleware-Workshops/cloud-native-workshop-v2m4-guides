## Lab1 - Creating High-performing Cache Services

In this lab, we'll develop 5 microservices into the cloud-native appliation architecture. These cloud-native applications
will be allowed to communicate with athentication by `Single Sign-On` server. To do that, we will configure how to `authenticate REST API requests` 
across services. In the end, we will optimize `data transaction performance` of the shopping cart service thru integrating with a `Cache(Data Grid) server` 
to increase end users'(customers) satification. And there's more fun facts how easy it is to deploy applications on OpenShift 4 via `ODO` command line tool.

#### Goals of this lab

---


The goal is to develop advanced cloud-native applications on `Red Hat Application Runtimes` and deploy them on `OpenShift 4` including 
`single sign-on access management` and `distributed cache manageemnt`. After this lab, you should end up with something like:

![goal]({% image_path module4-goal.png %})

####1. Developing and Deploying Inventory Service

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

`quarkus.datasource.url=jdbc:h2:file://projects/database.db`
`quarkus.datasource.driver=org.h2.Driver`

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

`quarkus.datasource.url=jdbc:postgresql:inventory`
`quarkus.datasource.driver=org.postgresql.Driver`

Now we will `push and commit` all changes that we implemented applications, production configuration to the remote `Git server(Gogs)` which is 
running on OpenShift cluster. As usual, we will use the following `git commands` in CodeReady Workspaces Terminal:
 them 

`git add .`

`git commit -m 'add posgresql datasource'`

`git push`

You might need to input your crededntial of Gogs server duriing `git push` as below:

 * Username: `userXX`
 * Password: `r3dh4t1!`

![inventory_service]({% image_path inventory_git_push.png %})

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

`oc project userXX-cloud-native-apps`

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

 * Start and watch the build, which will take about a minute to complete:

`oc start-build inventory-service --from-dir=target/binary --follow`

 * Deploy it as an OpenShift application after the build is done:

`oc new-app inventory-service`

# Create the route

`oc expose svc/inventory-service`

Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory-quarkus`

Wait for that command to report replication controller "inventory-quarkus-1" successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

# Get the route URL

`export URL="http://$(oc get route | grep inventory-service | awk '{print $2}')"`

`curl $URL/services/inventory : echo`

So now `Inventory` service is deployed to OpenShift. You can also see it in the Project Status in the OpenShift Console 
with its single replica running in 1 pod, along with the Postgres database pod.