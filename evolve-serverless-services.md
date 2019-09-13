## Lab3 - Evolving Serverless Services

In this cloud-native application architecture, We now have multiple microservices to implement traditional application workloads and reactive system. 
However, it's not necessary that whole applications/services have to run all the time(24/7) for serving funtions because a certain service can be 
responsed `on demand`. The reason why `serverless` application came up in the advanced cloud-native architecture. Before getting started with developing 
serverless applcation, we will walk you through quickly what `serverless` means as below:

 * The term [serverless](https://www.redhat.com/en/topics/cloud-native-apps/what-is-serverless) has been coming up in more conversations recently. 
Let’s clarify the concept, and those related to it, such as serverless computing and serverless platform.

 * `Serverless` is often used interchangeably with the term FaaS (Functions-as-a-Service). But serverless doesn’t mean that there is no server. 
In fact, there are many servers—serverful—because a public cloud provider provides the servers that deploy, run, and manage your application.

 * `Serverless computing` is an emerging category that represents a shift in the way developers build and deliver software systems. 
Abstracting application infrastructure away from the code can greatly simplify the development process while introducing new cost and efficiency benefits. 
I believe serverless computing and FaaS will play an important role in helping to define the next era of enterprise IT, along with cloud-native services 
and the [hybrid cloud](https://enterprisersproject.com/hybrid-cloud).

 * `Serverless platforms` provide `APIs` that allow users to run code functions (also called actions) and return the results of each function. 
Serverless platforms also provide HTTPS endpoints to allow the developer to retrieve function results. These endpoints can be used as inputs for other 
functions, thereby providing a sequence (or chaining) of related functions.

The severless application enables `DevOps teams` to enjoy `benefits` as here:

 * Optimizing computing resources(i.e CPU, Memory)

 * Autoscaling

 * Simplifying CI/CD pipeline

#### Goals of this lab

---

The goal is to develop serverless applications on `Red Hat Runtimes` and deploy `Knative service` on `OpenShift 4` with 
a cloud-native, continuous integration and delivery (CI/CD) Pipelines using `Tekton`. After this lab, you should end up with something like:

![goal]({% image_path lab3-goal.png %})

In this lab, we'll deploy the `Payment Service` as a serverless application using `Knative Serving`, `Istio`, `Tekton Pipelines` and `Quarkus native application` 
because the payment service doesn't need to be served for the end users as the other services like the shopping cart and catalog services.

The `Apache Kafka Event` source enables `Knative Eventing` integration with Apache Kafka. When a message is produced to Apache Kafka, 
the Apache Kafka Event Source will consume the produced message and post that message to the corresponding event sink.

##### What is Knative?

![Logo]({% image_path knative-audience.png %}){:width="700px"}

[Knative](https://www.openshift.com/learn/topics/knative) extends [Kubernetes](https://www.redhat.com/en/topics/containers/what-is-kubernetes) 
to provide components for building, deploying, and managing [serverless](https://developers.redhat.com/topics/serverless-architecture/) applications. 
Build serverless applications that run wherever you need them—on-premise or on any cloud—with Knative and OpenShift.

Knative components leverage best practices from real-world Kubernetes deployments:

 * [Build](https://github.com/knative/build) provides a pluggable model for building containers for serverless applications from source code.

 * [Serving](https://github.com/knative/serving)uses Kubernetes and [Istio](https://www.redhat.com/en/topics/microservices/what-is-a-service-mesh) to 
 rapidly deploy, network, and automatically scale serverless workloads.
 
 * [Eventing](https://github.com/knative/eventing) is common infrastructure for consuming and producing events to stimulate applications.

[Service Mesh](https://www.openshift.com/learn/topics/service-mesh) provides `traffic monitoring`, `access control`, `discovery`, `security`, `resiliency`, 
and other useful things to a group of services. `Istio` does all that, but it doesn't require any changes to the code of any of those services. 
To make the magic happen, Istio deploys a `proxy (called a sidecar)` next to each service. All of the traffic meant for a service goes to the proxy, 
which uses policies to decide how, when, or if that traffic should go on to the service. `Istio` also enables sophisticated DevOps techniques such as 
`canary deployments`, `circuit breakers`, `fault injection`, and more. 

`Istio Service Mesh` is a sidecar container implementation of the features and functions needed when creating and managing microservices. 
Monitoring, tracing, circuit breakers, routing, load balancing, fault injection, retries, timeouts, mirroring, access control, rate limiting, and more, 
are all a part of this. While all those features and functions are now available by using a myriad of libraries in your code, what sets Istio apart is that 
you get these benefits with no changes to your source code.

By using the `sidecar model`, Istio runs in a Linux container in your Kubernetes pods (much like a sidecar rides along side a motorcycle) and injects and 
extracts functionality and information based on your configuration. Again (for emphasis), this is your configuration that lives outside of your code. 
This immediately lessens code complexity and heft.

It also (and this is important), moves operational aspects away from code development and into the domain of operations. Why should a developer be 
burdened with circuit breakers and fault injections and should they respond to them? Yes, but for handling and/or creating them? Take that out of 
your code and let your code focus on the underlying business domain. Make the code smaller and less complex.

####1. Building a Native Executable

---

Let’s now produce a native executable for our application. It improves the startup time of the application, and produces a minimal disk and 
memory footprint. The executable would have everything to run the application including the `JVM`(shrunk to be just enough to run the application), 
and the application. This is accomplished using [GraalVM](https://graalvm.org/).

`GraalVM` is a universal virtual machine for compiling and running applications written in JavaScript, Python, Ruby, R, JVM-based languages like 
Java, Scala, Groovy, Kotlin, Clojure, and LLVM-based languages such as C and C++. It includes ahead-of-time compilation, aggressive dead code elimination, 
and optimal packaging as native binaries that moves a lot of startup logic to build-time, thereby reducing startup time and memory resource requirements significantly.

![serverless]({% image_path native-image-process.png %}

`GraalVM` is already installed for you. Inspect the value of the GRAALVM_HOME variable in the CodeReady Workspaces `Terminal` with:

`echo $GRAALVM_HOME`

In this step, we will learn how to compile the application to a native executable and run the native image on local machine.

##### Build the image

Within the `pom.xml` is the declaration for the Quarkus Maven plugin which contains a profile for `native-image`:

~~~xml
<profile>
    <id>native</id>
    <build>
        <plugins>
        <plugin>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-maven-plugin</artifactId>
            <version>${quarkus.version}</version>
            <executions>
            <execution>
                <goals>
                <goal>native-image</goal>
                </goals>
                <configuration>
                <enableHttpUrlHandler>true</enableHttpUrlHandler>
                </configuration>
            </execution>
            </executions>
        </plugin>
        </plugins>
    </build>
</profile>
~~~

We use a profile because, you will see very soon, packaging the native image takes a few seconds. However, this compilation time is only incurred once, 
as opposed to every time the application starts, which is the case with other approaches for building and executing JARs.

##### Build Native Image and Run it Locally

Let's find out why `Qaurkus` is named with `SuperSonic Subatomic Subatomic Java`. In order to run the payment service locally, we need to a Kafka server locally so we will run a `native image` without `Kafka relevant codes and configuration`.

 * `Comment` all properties in `src/main/resources/application.properties` 

~~~java
# Outgoing stream
#mp.messaging.outgoing.payments.bootstrap.servers=my-cluster-kafka-bootstrap:9092
#mp.messaging.outgoing.payments.connector=smallrye-kafka
#mp.messaging.outgoing.payments.topic=payments
#mp.messaging.outgoing.payments.value.serializer=org.apache.kafka.common.serialization.StringSerializer
#mp.messaging.outgoing.payments.key.serializer=org.apache.kafka.common.serialization.StringSerializer
~~~

 * `Comment` the following Kafka codes in `src/main/resources/PaymentResources.java`  

~~~java
//    @ConfigProperty(name = "mp.messaging.outgoing.payments.bootstrap.servers")
//    public String bootstrapServers;
//
//    @ConfigProperty(name = "mp.messaging.outgoing.payments.topic")
//    public String paymentsTopic;
//
//    @ConfigProperty(name = "mp.messaging.outgoing.payments.value.serializer")
//    public String paymentsTopicValueSerializer;
//
//    @ConfigProperty(name = "mp.messaging.outgoing.payments.key.serializer")
//    public String paymentsTopicKeySerializer;
//
//    private Producer<String, String> producer;
~~~

 * private void pass(String orderId, String paymentId, String remarks)
~~~java
//        producer.send(new ProducerRecord<String, String>(paymentsTopic, payload.toString()));
~~~

 * private void fail(String orderId, String paymentId, String remarks)
~~~java
//        producer.send(new ProducerRecord<String, String>(paymentsTopic, payload.toString()));
~~~

 * public void init(@Observes StartupEvent ev)
~~~java
//        props.put("key.serializer", paymentsTopicKeySerializer);
//        producer = new KafkaProducer<String, String>(props);
~~~

Create a `native executable` via CodeReady Workspaces `Terminal`:

`mvn package -Pnative`

![serverless]({% image_path payment-native-image-build.png %})

Since our environment here is Linux, you can just run it. In the CodeReady Workspaces `Terminal`, run:

`target/payment-1.0-SNAPSHOT-runner`

Notice the amazingly fast startup time:

~~~shell
2019-09-11 13:32:16,260 INFO  [io.sma.rea.mes.ext.MediatorManager] (main) Deployment done... start processing
2019-09-11 13:32:16,261 INFO  [io.sma.rea.mes.imp.ConfiguredChannelFactory] (main) Found incoming connectors: [smallrye-kafka]
2019-09-11 13:32:16,261 INFO  [io.sma.rea.mes.imp.ConfiguredChannelFactory] (main) Found outgoing connectors: [smallrye-kafka]
2019-09-11 13:32:16,261 INFO  [io.sma.rea.mes.imp.ConfiguredChannelFactory] (main) Stream manager initializing...
2019-09-11 13:32:16,261 INFO  [io.sma.rea.mes.ext.MediatorManager] (main) Initializing mediators
2019-09-11 13:32:16,261 INFO  [io.sma.rea.mes.ext.MediatorManager] (main) Connecting mediators
2019-09-11 13:32:16,261 INFO  [io.quarkus] (main) Quarkus 0.21.2 started in 0.016s. Listening on: http://[::]:8080
2019-09-11 13:32:16,261 INFO  [io.quarkus] (main) Installed features: [cdi, resteasy, smallrye-context-propagation, smallrye-reactive-messaging, smallrye-reactive-messagin
g-kafka, smallrye-reactive-streams-operators, vertx]
~~~

That’s `16 milliseconds` to start up.

![serverless]({% image_path payment-native-runn.png %})

And extremely low memory usage as reported by the Linux `ps` utility. While the app is running, open another Terminal 
(click the `+` button on the terminal tabs line) and run:

`ps -o pid,rss,command -p $(pgrep -f runner)`

You should see something like:

~~~shell
  PID   RSS COMMAND
 17509 58304 target/payment-1.0-SNAPSHOT-runner
~~~

![serverless]({% image_path payment-native-pss.png %})

This shows that our process is taking around `50 MB` of memory ([Resident Set Size](https://en.wikipedia.org/wiki/Resident_set_size), or RSS). Pretty compact!

> NOTE: The RSS and memory usage of any app, including Quarkus, will vary depending your specific environment, and will rise as the application experiences load.

Make sure the app is still working as expected (we’ll use `curl` this time to access it directly). In a new CdeReady Workspaces Terminal run:

`curl -s -w '%{http_code}' -o /dev/null http://localhost:8080 ; echo`

You should see the return code:

`200`

`Congratuations!` You’ve now built a Java application as a native executable JAR and a Linux native binary. We’ll explore the benefits of native 
binaries later in when we start deploying to Kubernetes.

Before moving to the next step, go to the first Terminal tab and press `CTRL+C` to stop our native app (or close the Terminal window).

##### Re-Build Native Image and Run it on OpenShift

`Uncomment` all above codes and properties and create a native executable by `once again` opening the command palette. Choose `Build Native Quarkus App`. This will execute `mvn clean package -Pnative` behind the scenes. The `-Pnative` argument selects the native maven profile which invokes the Graal compiler.

![serverless]({% image_path native-image-build-palette.png %})

`This will take about 3-4 minutes to finish. Wait for it!`

![serverless]({% image_path native-image-build.png %})

> NOTE: You can safely ignore any warnings like `Warning: RecomputeFieldValue.FieldOffset automatic substitution failed`. 
These are harmless and will be removed in future releases of Quarkus. 

> NOTE: Since we are on Linux in this environment, and the OS that will eventually run our application is also Linux, we can use our local OS to 
build the native Quarkus app. If you need to build native Linux binaries when on other OS’s like Windows or Mac OS X, you’ll need to have Docker 
installed and then use `mvn clean package -Pnative -Dnative-image.docker-build=true -DskipTests=true`.

In addition to the regular files, the build will produce `target/payment-1.0-SNAPSHOT-runner`. This is a native Linux binary. Not a shell script, 
or a JAR file, but a native binary.

Before moving to the next step, go to the first Terminal tab and press `CTRL+C` to stop our native app (or close the Terminal window).

####2. Deploying the serverless application in Knative Serving

---

`Knative Serving` builds on Kubernetes and `Istio` to support deploying and serving of serverless applications and functions. `Serving` is easy to 
get started with and scales to support advanced scenarios.

The Knative Serving project provides `middleware primitives` that enable:

 * Rapid deployment of serverless containers

 * Automatic scaling up and down to zero

 * Routing and network programming for Istio components

 * Point-in-time snapshots of deployed code and configurations

In the lab, `Knative Serving` is already `installed` on OpenShift cluster but if you want to install Knative Serving on your own OpenShift cluster, 
you can play with [Installing the Knative Serving Operator](https://knative.dev/docs/install/knative-with-openshift/) as below:

![serverless]({% image_path knative_serving_tile_highlighted.png %})

Before deploying the payment service using a native image, let's delete existing payment Kubernetes object such as Pod, DeploymentConfig, Service,

First, we need to delete existing `BuildConfig` based an excutable Jar that we deployed it in lab 2.

`oc delete bc/payment`

Create a new binary build within OpenShift

`oc new-build quay.io/quarkus/ubi-quarkus-native-binary-s2i:19.2.0 --binary --name=payment -l app=payment`

You should get a `--> Success message` at the end.

> `NOTE`: This build uses the new [Red Hat Universal Base Image](Red Hat Universal Base Image), providing foundational software needed to 
run most applications, while staying at a reasonable size.

And then start and watch the build, which will take about a minute to complete:

`oc start-build payment --from-file target/*-runner --follow`

This step will combine the native binary with a base OS image, create a new container image, and push it to an internal image registry.

Once that’s done, go to `Builds > Image Streams` on the left menu then input `payment` to show the payment imagestream. Click on `payment`:

![serverless]({% image_path payment-is.png %})

In `Overview` tab, copy the `IMAGE REPOSITORY` to update it in the `Knative Serving Resouce`.

![serverless]({% image_path payment-is-oveview.png %})

Open `knative/knative-serving-service.yaml` to specify the Knative Serving custom resource align with the above payment service.

 * Replace `IMAGE REPOSITORY` URL with `YOUR_IMAGE_SERVICE_URL` in the following YAML:

~~~yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: payment
spec:
  template:
    metadata:
      name: payment
      annotations:
        # disable istio-proxy injection
        sidecar.istio.io/inject: "false"
    spec:
      containers:
        # Replace Project name userXX-cloudnativeapps with project in which payment is deployed
      - image: YOUR_IMAGE_SERVICE_URL:latest
~~~

The service can be deployed using the following command via CodeReady Workspaces `Terminal`:

`oc apply -f /projects/cloud-native-workshop-v2m4-labs/payment/knative/knative-serving-service.yaml`

After successful creation of the service we should see a Kubernetes Deployment named similar to `payment-service-v1-deployment` available.
Go to  `Home > Status` on the left menu and click on `payment-service-v1-deployment`. You will confirm `1` pod is `available` now.

![serverless]({% image_path payment-serving-deployment.png %})

Let's `leave the payment service` without any request via WEB UI or RESTful API more than `30 seconds`. You're probably wondering why we need to wait more than `30 seconds`? Don't worry about it because we will walk you through the reason from now on.

In the lab environment, `Knative Serving` and `Knative Eventing` components are already installed and you can only view permission of `Knative Serving` namespace/project.
Go to `Workloads > Config Maps` in `knative-serving` project via OpenShift web console and click on `config-autoscaler`.

![serverless]({% image_path knative-serving-config.png %})

Once you click on `config-autoscaler`, you will see the details on how Knative autoscaling feature is specified when you click on `YAML` tab.

As default, Knative will scale down to zero `automatically` when the service(i.e. payment) has no request in 30 seconds. This capability is how `Knative` handles your cloud-native applications for `Serverless workloads`.

 * `scale-to-zero-grace-period: 30s`

![serverless]({% image_path scale-to-zero-grace-period.png %})

In the meantime, it probably took at least 30 seconds so go back to `Home > Status` on the left menu and click on `payment-service-v1-deployment`. You will confirm `0` pod is `available` now. 

![serverless]({% image_path payment-serving-down-to-zero.png %})

`Congratulations!` We completed to deploy the payment service using `Quarkus native image` and `Knative Serving` quicker than traditional Java applications. This is not the end about Knative capabilites sowWe will see how the payment service will scale up `magically` in the following exercises.

####3. Enable Knative Eventing integration with Apache Kafka Event

---

`Apache Kafka Event source` enables `Knative Eventing` integration with Apache Kafka. When a message is produced to Apache Kafka, the `Apache Kafka Event Source` will consume the produced message and post that message to the corresponding event sink.

`Knative Eventing` is a system that is designed to address a common need for cloud native development and provides composable primitives to enable `late-binding` event sources and event consumers with below goals:

 * Services are loosely coupled during development and deployed independently.

 * Producer can generate events before a consumer is listening, and a consumer can express an interest in an event or class of events that is not yet being produced.

 * Services can be connected to create new applications without modifying producer or consumer, and with the ability to select a specific subset of events from a particular producer.

In this lab, `Knative Eventing` is already `installed` but if you want to install it in your own `OpenShift` cluster then you can install it via `Knative Eventing Operator` in OpenShift web console.

Create the `KafkaSource` custom objects, by configuring the required `consumerGroup`, `bootstrapServers` and `topics` values on the CR file of your source.

Open `knative/kafka-event-source.yaml` to define `KafkaSource` to integrate with the `Knative Eventing`. Copy the following `YAML` to `knative/kafka-event-source.yaml`.

~~~yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: KafkaSource
metadata:
  name: kafka-source
spec:
  consumerGroup: payment-consumer-group
  bootstrapServers: my-cluster-kafka-bootstrap:9092
  topics: orders
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: payment
~~~

The source can be deployed using the following command via CodeReady Workspaces `Terminal`:

`oc apply -f /projects/cloud-native-workshop-v2m4-labs/payment/knative/kafka-event-source.yaml`

![serverless]({% image_path kafka-event-source.png %})

`Great job!!` We completed to deploy a payment service as `serverless application` using `knative serving` then the `Apache Kafka Event source` enables `Knative Eventing` integration with `Apache Kafka`. Let's make sure if whole functions work properly for the end-user perspective.

####4. End to End Functional Testing

---

Let's go shopping some cool items via the frontend service(`Coolstore WEB UI`) then run the following shopping scenarios:

 * 1) Add a `Red Hat Fedora` to `Cart` via click on `Add to Cart` then you will see the `Success! Added!` message under the top munu.

![serverless]({% image_path add-to-cart.png %})

 * 2) Click on `Checkout` button in `Your Shopping Cart` and input the credit card information. The `Card Info` should be `16 `digits.

![serverless]({% image_path checkout.png %})

 * 3) `Input` your `Credit Card information` to pay the item you chose

 ![serverless]({% image_path input-cc-info.png %})

 * 4) Confirm `Payment Status` of the your shopping items in the `All Orders` tab. It should be `Processing`.

 ![serverless]({% image_path payment-processing.png %})

 * 5) Reload `All Orders` page to confirm after at least `5s` if the Payment Status changed to `COMPLETED` or `FAILED`.

 >`Note`: If the status is still `Processing`, the order service is processing incoming Kafka messages and store thme in MongoDB. Please reload the page a few times more. 

 ![serverless]({% image_path payment-completedorfailed.png %})

####5. Creating Cloud-Native CI/CD Pipelines using Tekton 

---

##### What is the Cloud-Native CI/CD Pipelines?

There're lots of open source CI/CD tools to build, test, deploy, and manage cloud-natvie applications/microservices from on-premise to private, public, 
and hybrid cloud. Each tool provides different features to integrate with existing platforms/systems such as `plugins`, `RESTful APIs`. This sometimes makes complex 
for DevOps teams to create the CI/CD pipelines and maintain them on `Kubernetes clusters`. The `cloud-natvie CI/CD pipeline` should be defined and executed in 
the Kubernetes native way. For example, the pipeline can be specified as Kubernetes resources using YAML format. 
in the hybrid cloud. What

`OpenShift Pipelines` provides a `cloud-native`, `continuous integration and delivery (CI/CD)` solution for building pipelines using [Tekton](https://tekton.dev/). 
`Tekton` is a flexible, Kubernetes-native, open-source CI/CD framework that enables automating deployments across multiple platforms (Kubernetes, serverless, VMs, etc) 
by abstracting away the underlying details.

OpenShift Pipelines features:

 * Standard CI/CD pipeline definition based on Tekton

 * Build images with Kubernetes tools such as S2I, Buildah, Buildpacks, Kaniko, etc

 * Deploy applications to multiple platforms such as Kubernetes, serverless and VMs

 * Easy to extend and integrate with existing tools

 * Scale pipelines on-demand

 * Portable across any Kubernetes platform

 * Designed for microservices and decentralized teams

 * Integrated with the OpenShift Developer Console

In the lab, `OpenShift Pipelines` is already `installed` on OpenShift cluster but if you want to install OpenShift Pipelines on `your own OpenShift cluster`, 
OpenShift Pipelines is provided as an add-on on top of OpenShift that can be installed via an `operator` available in the `OpenShift OperatorHub`. 
Follow [these instructions](https://github.com/openshift/pipelines-tutorial/blob/master/install-operator.md) in order to install OpenShift Pipelines 
on OpenShift via the `OperatorHub` as below:

![serverless]({% image_path operatorhub-pipeline.png %})

##### What is Tekton?

`Tekton` defines a number of [Kubernetes custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) as 
building blocks in order to standardize pipeline concepts and provide a terminology that is consistent across `CI/CD solutions`. These custom resources 
are an extension of the `Kubernetes API` that let users create and interact with these objects using the `OpenShift CLI (oc)`, `kubectl`, and other Kubernetes tools.

The `custom resources` needed to define a pipeline are listed below:

 * `Task`: a reusable, loosely coupled number of steps that perform a specific task (e.g. building a container image)

 * `Pipeline`: the definition of the pipeline and the tasks that it should perform

 * `PipelineResource`: inputs (e.g. git repository) and outputs (e.g. image registry) to and out of a pipeline or task

 * `TaskRun`: the execution and result (i.e. success or failure) of running an instance of task

 * `PipelineRun`: the execution and result (i.e. success or failure) of running a pipeline

![severless]({% image_path tekton-arch.png %})

For further details on pipeline concepts, refer to the [Tekton documentation](https://github.com/tektoncd/pipeline/tree/master/docs#learn-more) that 
provides an excellent guide for understanding various parameters and attributes available for defining pipelines.

In this lab, we will walk you through pipeline concepts and how to create and run a CI/CD pipeline for building and deploying `serverless applications` 
on `Knative` on OpenShift.

##### Deploying Nexus

To make maven builds faster, we will deploy `Sonatype Nexus`. You need to replace `userXX` with your credential:

`oc new-app sonatype/nexus`

Once you complete to deploy the Nexus server on OpenShift cluster, you can confirm it in `Project Status` page of OpenShift web console:

![severless]({% image_path nexus-deployment.png %})

##### Creating Tekton Tasks

`Tasks` consist of a number of steps that are executed sequentially. Each `task` is executed in a separate container within the same pod. 
They can also have inputs and outputs in order to interact with other tasks in the pipeline.

Here is an example of a Maven task for building a Maven-based Java application:

~~~yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: maven-build
spec:
  inputs:
    resources:
    - name: workspace-git
      targetPath: /
      type: git
  steps:
  - name: build
    image: maven:3.6.0-jdk-8-slim
    command:
    - /usr/bin/mvn
    args:
    - install
~~~

When a `task` starts running, it starts a pod and runs each `step` sequentially in a separate container on the same pod. This task happens to have a 
single step, but tasks can have multiple steps, and, since they run within the same pod, they have access to the same volumes in order to cache files, 
access configmaps, secrets, etc. `Tasks` can also receive inputs (e.g., a git repository) and outputs (e.g., an image in a registry) in order to interact 
with each other.

Note that only the requirement for a git repository is declared on the task and not a specific git repository to be used. That allows `tasks` to be 
reusable for multiple pipelines and purposes. You can find more examples of reusable `tasks` in the [Tekton Catalog](https://github.com/tektoncd/catalog) 
and [OpenShift Catalog](https://github.com/openshift/pipelines-catalog) repositories.

Install the `openshift-client` and `s2i-java` tasks from the catalog repository using `oc` or `kubectl`, which you will need for 
creating a pipeline in the next section:

Create the following Tekton tasks which will be used in the `Pipelines`

`oc create -f knative/openshift-client-task.yaml`

`oc create -f knative/s2i-java-8/s2i-java-8-task.yaml`

~~~shell
oc create -f https://raw.githubusercontent.com/tektoncd/catalog/master/openshift-client/openshift-client-task.yaml \
  -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/quarkus/s2i-quarkus-task.yaml \
  -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/quarkus/quarkus-native-task.yaml \
  -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/quarkus/quarkus-jvm-task.yaml \
  -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/knative-client/kn-service-create-task.yaml
~~~

You can take a look at the list of install tasks using the [Tekton CLI](https://github.com/tektoncd/cli/releases) that already installed 
in CodeReady Workspaces `Terminal`:

`tkn task ls`

openshift-client   58 seconds ago
s2i-java-8         1 minute ago

~~~shell
NAME                AGE
kn-create-service   12 seconds ago
openshift-client    13 seconds ago
quarkus-jvm         12 seconds ago
quarkus-native      12 seconds ago
s2i-quarkus         13 seconds ago
~~~

tkn pipeline start ccn-deploy-pipeline \
        -r app-git=quarkus-todo-git \
        -r app-image=quarkus-todo-image \
        -s pipeline

### Summary

In this scenario we developed five microservices with `REST API` exposure to communicate with the other microservices. We also used a variety of application 
runtimes such as `Quarkus`, `Spring Boot`, and `NodeJS` to compile, package, and containerize applications which is a major capability of the advanced cloud-native architecture.

To deploy the cloud-native applications with multiple datasources on `OpenShift` cluster, `Quarkus` provides an easy way to connect multiple datasources and 
obtain a reference to those datasources such as `PostgreSQL` and `MongoDB` in code.

In the end, we optimized `data transaction performance` of the shopping cart service thru integrating with a `JBoss Data Grid` 
to increase end users'(customers) satification. `Congratulations!`

##### Additional Resources:

 * [Knative on OpenShift](https://www.openshift.com/learn/topics/knative)

 * [Knative Install on OpenShift](https://knative.dev/docs/install/knative-with-openshift/)

 * [Knative Tutorial](https://redhat-developer-demos.github.io/knative-tutorial)

 * [Knative, Serverless Kubernetes Blogs](https://developers.redhat.com/topics/knative/)

 * [7 open source platforms to get started with serverless computing](https://opensource.com/article/18/11/open-source-serverless-platforms)

 * [How to develop functions-as-a-service with Apache OpenWhisk](https://opensource.com/article/18/11/developing-functions-service-apache-openwhisk)

 * [How to enable serverless computing in Kubernetes](https://opensource.com/article/19/4/enabling-serverless-kubernetes)
