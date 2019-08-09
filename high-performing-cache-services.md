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


#### Setup CodeReady Workspaces for Lab Environment

---

`SKIP this setup guide if you already completed earlier in the other Modules`


Follow these instructions to setup the development environment on CodeReady Workspaces. 

You might be familiar with the Eclipse IDE which is one of the most popular IDEs for Java and other
programming languages. [CodeReady Workspaces](https://access.redhat.com/documentation/en-us/red_hat_codeready_workspaces) is the next-generation Eclipse IDE which is web-based and gives you a full-featured IDE running in the cloud. You have an CodeReady Workspaces instance deployed on your OpenShift cluster
which you will use during these labs.

Go to the [CodeReady Workspaces URL]({{ ECLIPSE_CHE_URL }}) in order to configure your development workspace.

First, you need to register as a user. Register and choose the same username and password as 
your OpenShift credentials.

![codeready-workspace-register]({% image_path codeready-workspace-register.png %})

Log into CodeReady Workspaces with your user. You can now create your workspace based on a stack. A 
stack is a template of workspace configuration. For example, it includes the programming language and tools needed
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and needed configuration
on-demand. 

For this lab, click on the `Cloud Native Roadshow` stack and then on the `Create` button. 

![codeready-workspace-create-workspace]({% image_path codeready-workspace-create-workspace.png %})

Click on `Open` to open the workspace and then on the `Start` button to start the workspace for use, if it hasn't started automatically.

![codeready-workspace-start-workspace]({% image_path codeready-workspace-start-workspace.png %})

You can click on the left arrow icon to switch to the wide view:

![codeready-workspace-wide]({% image_path codeready-workspace-wide.png %})

It takes a little while for the workspace to be ready. When it's ready, you will see a fully functional 
CodeReady Workspaces IDE running in your browser.

![codeready-workspace-workspace]({% image_path codeready-workspace.png %})

Now you can import the project skeletons into your workspace.

In the project explorer pane, click on the `Import Projects...` and enter the following:

> You can find `GIT URL` when you log in {{GIT_URL}} with your credential(i.e. user1 / openshift).

  * Version Control System: `GIT`
  * URL: `{{GIT_URL}}/userXX/cloud-native-workshop-v2m3-labs.git`
  * Check `Import recursively (for multi-module projects)`
  * Name: `cloud-native-workshop-v2m3-labs`

![codeready-workspace-import]({% image_path codeready-workspace-import.png %}){:width="700px"}

The projects are imported now into your workspace and is visible in the project explorer.

CodeReady Workspaces is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to a Maven projects. In the project explorer, right-click on each project and then click on `Convert to Project` continuously.

![codeready-workspace-convert]({% image_path codeready-workspace-convert.png %}){:width="500px"}

Choose `Maven` from the project configurations and then click on `Save`.

![codeready-workspace-maven]({% image_path codeready-workspace-maven.png %}){:width="700px"}

Repeat the above for inventory and catalog projects.

> `NOTE`: the `Terminal` window in CodeReady Workspaces. For the rest of these labs, anytime you need to run 
a command in a terminal, you can use the CodeReady Workspaces `Terminal` window.

![codeready-workspace-terminal]({% image_path codeready-workspace-terminal.png %})

####1. Developing and Deploying Inventory Service

---

Let's take a look at `inventory service project` via `CodeReady Workspaces` Web IDE. In the project explorer, click on `inventory-service` directory 
in `cloud-native-workshop-v2m4-labs` folder. You will see s familiar structure of the maven project.

![inventory_service]({% image_path codeready-workspace-inventory-project.png %}){:width="500px"}

The `inventory service project` is already built on `Quarkus` and `Hibernate ORM with Panache` to expose RESTful APIs that retreive data.

> Click on the `inventory` folder in the project explorer and navigate below folders and files.

While the code is surprisingly simple, under the hood this is using:

 * RESTEasy to expose the REST endpoints
 * Hibernate ORM with Panache to perform the CRUD operations on the database
 * A PostgreSQL database; see below to run one via Linux Container

`Hibernate ORM` is the de facto JPA implementation and offers you the full breadth of an Object Relational Mapper. 
It makes complex mappings possible, but it does not make simple and common mappings trivial. Hibernate ORM with 
Panache focuses on making your entities trivial and fun to write in Quarkus.

Open `Inventory.java` in `src/main/java/com/redhat/cloudnative/` and check how easy you create a domain model using `PanacheEntity` class.

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

By extending `PanacheEntity` in your entities, you will get an ID field that is auto-generated. If you require a custom ID strategy, 
you can extend `PanacheEntityBase` instead and handle the ID yourself.

By using Use public fields, there is no need for functionless getters and setters (those that simply get or set the field). 
You simply refer to fields like Inventory.location without the need to write a Inventory.geLocation() implementation. Panache will 
auto-generate any getters and setters you do not write, or you can develop your own getters/setters that do more than get/set, which will be called when the field is accessed directly.

The `PanacheEntity` superclass comes with lots of super useful static methods and you can add your own in your derived entity class, and much like traditional object-oriented programming it's natural and recommended to place custom queries as close to the entity as possible, ideally within the entity definition itself. 
Users can just start using your entity Inventory by typing Inventory, and getting completion for all the operations in a single place.

When an entity is annotated with `@Cacheable`, all its field values are cached except for collections and relations to other entities.
This means the entity can be loaded without querying the database, but be careful as it implies the loaded entity might not reflect recent changes in the database.

Next, let's find out how `inventory service` exposes `RESTful API` on Quarkus. Open `InventoryResource.java` in `src/main/java/com/redhat/cloudnative/` and 
you will see the following code sniffet.

~~~java
@Path("/services/inventory")
@ApplicationScoped
@Produces("application/json")
@Consumes("application/json")
public class InventoryResource {

    @GET
    public List<Inventory> getAll() {
        return Inventory.listAll();
    }

    @GET
    @Path("{itemId}")
    public List<Inventory> getAvailability(@PathParam String itemId) {
        return Inventory.<Inventory>streamAll()
        .filter(p -> p.itemId.equals(itemId))
        .collect(Collectors.toList());
    }
}
~~~

The above REST services defines two endpoints:

* `/inventory` that is accessible via `HTTP GET` which will return all known product Inventory entities as JSON
* `/inventory/<location>` that is accessible via `HTTP GET` at
for example `/inventory/Boston` with
the last path parameter being the location which we want to check its inventory status.

This project also contains `Dockerfile` to build a container that runs the Quarkus application in JVM mode as well as native (no JVM) mode.
Open `Dockerfile.jvm` in `src/main/docker/` and you will see similar contents as here: 

~~~java
FROM fabric8/java-alpine-openjdk8-jre
ENV JAVA_OPTIONS="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
ENV AB_ENABLED=jmx_exporter
COPY target/lib/* /deployments/lib/
COPY target/*-runner.jar /deployments/app.jar
ENTRYPOINT [ "/deployments/run-java.sh" ]
~~~

In Development, we will configure to use local `in-memory H2 database` for local testing, as defined in `src/main/resources/application.properties`:

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

`cd /projects/cloud-native-workshop-v2m1-labs/inventory`

`mvn compile quarkus:dev`

You should see a bunch of log output that ends with:

~~~java
12:56:43,106 INFO  [io.quarkus] Quarkus 0.15.0 started in 9.429s. Listening on: http://[::]:8080
12:56:43,106 INFO  [io.quarkus] Installed features: [agroal, cdi, hibernate-orm, jdbc-h2, narayana-jta, resteasy, resteasy-jsonb]
~~~

Open a `new` CodeReady Workspaces `Terminal` and invoke the RESTful endpoint using the following CURL commands. The output looks like here:

`curl http://localhost:8080/services/inventory ; echo`

~~~java
[{"id":1,"itemId":"329299","link":"http://maps.google.com/?q=Raleigh","location":"Raleigh","quantity":736},{"id":2,"itemId":"329199","link":"http://maps.google.com
/?q=Boston","location":"Boston","quantity":512},{"id":3,"itemId":"165613","link":"http://maps.google.com/?q=Seoul","location":"Seoul","quantity":256},{"id":4,"item
Id":"165614","link":"http://maps.google.com/?q=Singapore","location":"Singapore","quantity":54},{"id":5,"itemId":"165954","link":"http://maps.google.com/?q=London"
,"location":"London","quantity":87},{"id":6,"itemId":"444434","link":"http://maps.google.com/?q=NewYork","location":"NewYork","quantity":443},{"id":7,"itemId":"444
435","link":"http://maps.google.com/?q=Paris","location":"Paris","quantity":600},{"id":8,"itemId":"444437","link":"http://maps.google.com/?q=Tokyo","location":"Tok
yo","quantity":230}]
~~~

`curl http://localhost:8080/services/inventory/329199 ; echo`

~~~java
[{"id":2,"itemId":"329199","link":"http://maps.google.com/?q=Boston","location":"Boston","quantity":512}]
~~~

> `NOTE`: Make sure to stop Quarkus development mode via `Close` terminal.


In production on `OpenShift` cluster, the inventory service will connect to `PostgeSQL database` in production on OpenShift cluster.

Add a `quarkus-jdbc-postgresql` extendsion via CodeReady Workspaces `Terminal`:

`cd /projects/cloud-native-workshop-v2m1-labs/inventory`

`mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-jdbc-postgresql"`

Update the `quarkus.datasource.url, quarkus.datasource.drive` variables in `src/main/resources/application.properties` as below:

~~~shell
quarkus.datasource.url=jdbc:postgresql:inventory
quarkus.datasource.driver=org.postgresql.Driver
~~~

Package the inventory application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Or you can run a maven plugin command directly in `Terminal`:

`mvn clean package -DskipTests`

> `NOTE`: You should `SKIP` the Unit test because you don't have PostgreSQL database in local environment.

Open a web browser to access `OpenShift Web Console` then click on `Create Project` in `Project Status`.

![create_new]({% image_path cloud-native-apps-project-create.png %})

Input the following data in the popup window and click on `Create`.

 * Name: `userXX-cloud-native-apps`
 * Display Name: `USER0 Cloud-Native Apps Architecture`
* Description: _leave this field empty_

![create_dialog]({% image_path cloud-native-apps-project-input.png %}){:width="700"}

This will take you to the project overview. There's nothing there yet, but that's about to change.

> Let's deploy our new inventory microservice to OpenShift!

We'll use the CLI to deploy the components for our monolith. To deploy the monolith template using the CLI, execute the following commands via CodeReady Workspaces `Terminal` window:

Copy login command in `OpenShift Web Console`:

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

`oc project userXX-cloud-native-apps`

~~~shell
oc new-app -e POSTGRESQL_USER=inventory \
  -e POSTGRESQL_PASSWORD=mysecretpassword \
  -e POSTGRESQL_DATABASE=inventory openshift/postgresql:10 \
  --name=inventory-database
~~~

> `NOTE:` If you change the username and password you also need to update `src/main/resources/application.properties` which contains
the credentials used when deploying to OpenShift.

This will deploy the database to our new project. 

![inventory_db_deployments]({% image_path inventory-database-deployment.png %})

Red Hat OpenShift Application Runtimes includes a powerful maven plugin that can take an
existing Quarkus application and generate the necessary Kubernetes configuration.

Build and deploy the project using the following command, which will use the maven plugin to deploy via CodeReady Workspaces `Terminal`:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=inventory-service -l app=inventory-service`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

Next, create a temp directory to store only previously-built application with necessary lib directory via CodeReady Workspaces `Terminal`:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

> `NOTE`: You can also use a true source-based S2I build, but we're using binaries here to save time.

And then start and watch the build, which will take about a minute to complete:

`oc start-build inventory-service --from-dir=target/binary --follow`

Once the build is done, we'll deploy it as an OpenShift application and override the Postgres URL to specify our production Postgres credentials:

`oc new-app inventory-service -e QUARKUS_DATASOURCE_URL=jdbc:postgresql://inventory-database:5432/inventory`

and expose your service to the world:

`oc expose service inventory-service`

Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory-quarkus`

Wait for that command to report replication controller "inventory-quarkus-1" successfully rolled out before continuing.

>`NOTE:` Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

`oc get routes`

Replace your own route URL in the above command output: 

`curl http://YOUR_INVENTORY_ROUTE_URL/services/inventory ; echo`

So now `Inventory` service is deployed to OpenShift. You can also see it in the Project Status in the OpenShift Console 
with its single replica running in 1 pod, along with the Postgres database pod.