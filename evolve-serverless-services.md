## Lab3 - Evolving services with Serverless

In our cloud-native application architecture, We now have multiple microservices in a _reactive_ system.
However, it's not necessary our applications and services be up and running 24 hours  day. They only need to be running _on-demand_, when something needs to use the service. This is one of the reasons why _serverless_ architectures have gained popularity.

 * **Serverless** is often used interchangeably with the term _FaaS_ (Functions-as-a-Service). But serverless doesn’t mean that there is no server.
In fact, there _are_ servers - a public cloud provider provides the servers that deploy, run, and manage your application.

 * **Serverless computing** is an emerging category that represents a shift in the way developers build and deliver software systems.
Abstracting application infrastructure away from the code can greatly simplify the development process while introducing new cost and efficiency benefits.
Serverless computing and FaaS will play an important role in helping to define the next era of enterprise IT, along with cloud-native services
and the [hybrid cloud](https://enterprisersproject.com/hybrid-cloud){:target="_blank"}.

 * **Serverless platforms** provide APIs that allow users to run code snippets (functions, also called _actions_) and return the results of each function.
Serverless platforms also provide endpoints to allow the developer to retrieve function results. These endpoints can be used as inputs for other
functions, thereby providing a sequence (or chain) of related functions.

The severless application enables DevOps teams to enjoy benefits like:

 * Optimizing computing resources(i.e CPU, Memory)
 * Autoscaling
 * Simplifying CI/CD pipeline

#### Goals of this lab

---

The goal is to develop serverless applications on **Red Hat Runtimes** and deploy them on **OpenShift 4** using **OpenShift Serverless** (Based on the [Knative](https://www.openshift.com/learn/topics/knative){:target="_blank"} project) with
a cloud-native, continuous integration and delivery (CI/CD) Pipelines using `Tekton`. After this lab, you should end up with something like:

![goal]({% image_path lab3-goal.png %})

In this lab, we'll deploy the Payment Service as a Quarkus-based serverless application using Knative Serving, Istio, and Tekton Pipelines.

The Knative Kafka Event _source_ enables _Knative Eventing_ integration with Apache Kafka. When a message is produced in Apache Kafka,
the Apache Kafka Event Source will consume the produced message and post that message to the corresponding event _sink_.

##### What is Knative?

![Logo]({% image_path knative-audience.png %}){:width="700px"}

[Knative](https://www.openshift.com/learn/topics/knative) extends [Kubernetes](https://www.redhat.com/en/topics/containers/what-is-kubernetes){:target="_blank"}
to provide components for building, deploying, and managing [serverless](https://developers.redhat.com/topics/serverless-architecture/){:target="_blank"} applications.
Build serverless applications that run wherever you need them—on-premise or on any cloud—with Knative and OpenShift.

Knative components leverage best practices from real-world Kubernetes deployments:

 * [Serving](https://github.com/knative/serving){:target="_blank"}uses Kubernetes and [Istio](https://www.redhat.com/en/topics/microservices/what-is-a-service-mesh){:target="_blank"} to
 rapidly deploy, network, and automatically scale serverless workloads.

 * [Eventing](https://github.com/knative/eventing){:target="_blank"} is common infrastructure for consuming and producing events to stimulate applications.

##### What is a Service Mesh?

[Service Mesh](https://www.openshift.com/learn/topics/service-mesh) provides traffic monitoring, access control, discovery, security, resiliency,
and other useful things to a group of services. Istio does all that, but it doesn't require any changes to the code of any of those services.
To make the magic happen, Istio deploys a proxy (called a _sidecar_) next to each service. All of the traffic meant for a service goes to the proxy,
which uses policies to decide how, when, or if that traffic should go on to the service. Istio also enables sophisticated DevOps techniques such as
canary deployments, circuit breakers, fault injection, and more.

Istio also moves operational aspects away from code development and into the domain of operations. Why should a developer be
burdened with circuit breakers and fault injections and should they respond to them? Yes, but for handling and/or creating them? Take that out of
your code and let your code focus on the underlying business domain.

#### 1. Building a Native Executable

---

Let’s now produce a native executable for an example Quarkus application. It improves the startup time of the application, and produces a minimal disk and
memory footprint. The executable would have everything to run the application including the `JVM`(shrunk to be just enough to run the application),
and the application. This is accomplished using [GraalVM](https://graalvm.org/){:target="_blank"}.

`GraalVM` is a universal virtual machine for compiling and running applications written in JavaScript, Python, Ruby, R, JVM-based languages like
Java, Scala, Groovy, Kotlin, Clojure, and LLVM-based languages such as C and C++. It includes ahead-of-time compilation, aggressive dead code elimination,
and optimal packaging as native binaries that moves a lot of startup logic to build-time, thereby reducing startup time and memory resource requirements significantly.

![serverless]({% image_path native-image-process.png %})

`GraalVM` is already installed for you. Inspect the value of the `GRAALVM_HOME` variable in the CodeReady Workspaces _Terminal_ with:

`echo $GRAALVM_HOME`

In this step, we will learn how to compile the application to a native executable and run the native image on local machine.

Compiling a native image takes longer than a regular JAR file (bytecode) compilation. However, this compilation time is only incurred once,
as opposed to every time the application starts, which is the case with other approaches for building and executing JARs.

##### Build Native Image and Run it Locally

Let's find out why Quarkus calls itself _SuperSonic Subatomic Subatomic Java_. Let's build a sample app. In CodeReady Terminal, run this command:

~~~sh
mkdir /tmp/hello && cd /tmp/hello && \
mvn io.quarkus:quarkus-maven-plugin:0.21.2:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=getting-started \
    -DclassName="org.acme.quickstart.GreetingResource" \
    -Dpath="/hello"
~~~

This will create a simple Quarkus app in the `/tmp/hello` directory.

Next, create a `native executable` with this command:

`mvn -f /tmp/hello clean package -Pnative -DskipTests`

> This may take a minute or two to run. One of the benefits of Quarkus is amazingly fast startup time, at the expense of a longer build time to optimize and remove dead code, process annotations, etc. This is only incurred once, at build time rather than *every* startup!

> NOTE: You can safely ignore any warnings like `Warning: RecomputeFieldValue.FieldOffset automatic substitution failed`.
These are harmless and will be removed in future releases of Quarkus.

> NOTE: Since we are on Linux in this environment, and the OS that will eventually run our application is also Linux, we can use our local OS to
build the native Quarkus app. If you need to build native Linux binaries when on other OS’s like Windows or Mac OS X, you’ll need to have Docker
installed and then use `mvn clean package -Pnative -Dnative-image.docker-build=true -DskipTests=true`.


![serverless]({% image_path payment-native-image-build.png %})

Since our environment here is Linux, you can just run it. In the CodeReady Workspaces Terminal, run:

`/tmp/hello/target/*-runner`

Notice the amazingly fast startup time:

~~~shell
2019-09-16 08:02:29,096 INFO  [io.quarkus] (main) Quarkus 0.21.2 started in 0.014s. Listening on: http://[::]:8080
2019-09-16 08:02:29,096 INFO  [io.quarkus] (main) Installed features: [cdi, resteasy]
~~~

That’s _14 milliseconds_ to start up.

![serverless]({% image_path payment-native-runn.png %})

And extremely low memory usage as reported by the Linux `ps` utility. While the app is running, open another Terminal
(click the `+` button on the terminal tabs line) and run:

`ps -o pid,rss,command -p $(pgrep -f runner)`

You should see something like:

~~~shell
   PID   RSS COMMAND
 74810 50388 /tmp/hello/target/getting-started-1.0-SNAPSHOT-runner
~~~

![serverless]({% image_path payment-native-pss.png %})

This shows that our process is taking around `50 MB` of memory ([Resident Set Size](https://en.wikipedia.org/wiki/Resident_set_size){:target="_blank"}, or RSS). Pretty compact!

> NOTE: The RSS and memory usage of any app, including Quarkus, will vary depending your specific environment, and will rise as the application experiences load.

Make sure the app works. In a new CodeReady Workspaces Terminal run:

`curl -i http://localhost:8080/hello; echo`

You should see the return:

~~~console
HTTP/1.1 200 OK
Connection: keep-alive
Content-Type: text/plain;charset=UTF-8
Content-Length: 5
Date: Mon, 16 Sep 2019 03:35:40 GMT

hello
~~~

`Congratuations!` You’ve now built a Java application as a native executable JAR and a Linux native binary. We’ll explore the benefits of native
binaries later in when we start deploying to Kubernetes.

Before moving to the next step, go to the first Terminal tab and press `CTRL+C` to stop our native app (or close the Terminal window).

#### 2. Delete old payment service

---

_Knative Serving_ builds on Kubernetes and Istio to support deploying and serving of serverless applications and functions. _Serving_ is easy to
get started with and scales to support advanced scenarios.

The Knative Serving project provides middleware primitives that enable:

 * Rapid deployment of serverless containers
 * Automatic scaling up and down to zero
 * Routing and network programming for Istio components
 * Point-in-time snapshots of deployed code and configurations

In the lab, _Knative Serving_ is already installed on your OpenShift cluster but if you want to install Knative Serving on your own OpenShift cluster,
you can play with [Installing the Knative Serving Operator](https://knative.dev/docs/install/knative-with-openshift/){:target="_blank"} as below:

![serverless]({% image_path knative_serving_tile_highlighted.png %})

First, we need to delete existing `BuildConfig` as it is based an excutable Jar that we deployed it in lab 2.

`oc delete bc/payment`

We also will delete our existing payment _deployment_ and _route_ since Knative will handle deploying the payment service and routing traffic to its managed pod when needed. Delete the existing payment deployment and its associated route and service with:

`oc delete dc/payment route/payment svc/payment`

#### 3. Enable Knative Eventing integration with Apache Kafka Event

---

Knative Eventing is a system that is designed to address a common need for cloud native development and provides composable primitives to enable `late-binding` event sources and event consumers with below goals:

 * Services are loosely coupled during development and deployed independently.

 * Producer can generate events before a consumer is listening, and a consumer can express an interest in an event or class of events that is not yet being produced.

 * Services can be connected to create new applications without modifying producer or consumer, and with the ability to select a specific subset of events from a particular producer.

The _Apache Kafka Event source_ enables Knative Eventing integration with Apache Kafka. When a message is produced to Apache Kafka, the Event Source will consume the produced message and post that message to the corresponding event sink.

##### Remove direct Knative integration code

Currently our Payment service directly binds to Kafka to listen for events. Now that we have Knative eventing integration, we no longer need this code. Open the `PaymentResource.java` file (in `payment-service/src/main/java/com/redhat/cloudnative` directory).

Delete (or comment out) the `onMessage()` method:

~~~java
//    @Incoming("orders")
//    public CompletionStage<Void> onMessage(KafkaMessage<String, String> message)
//            throws IOException {
//
//        log.info("Kafka message with value = {} arrived", message.getPayload());
//        handleCloudEvent(message.getPayload());
//        return message.ack();
//    }
~~~

And delete the configuration for the incoming stream. In `application.properties`, delete (or comment out) the following lines for the _Incoming_ stream:

~~~java
# Incoming stream (unneeded when using Knative events)
# mp.messaging.incoming.orders.connector=smallrye-kafka
# mp.messaging.incoming.orders.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
# mp.messaging.incoming.orders.key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
# mp.messaging.incoming.orders.bootstrap.servers=my-cluster-kafka-bootstrap:9092
# mp.messaging.incoming.orders.group.id=payment-order-service
# mp.messaging.incoming.orders.auto.offset.reset=earliest
# mp.messaging.incoming.orders.enable.auto.commit=true
# mp.messaging.incoming.orders.request.timeout.ms=30000
~~~

##### Rebuild and re-deploy new Payment service

Open the `payment-service/pom.xml` in the editor, then in the CodeReady command palette, Choose `Build Native Quarkus App`. This will re-build our native executable in the `target/` directory.

Or you can run the commands directly:

`cd /projects/cloud-native-workshop-v2m4-labs/payment-service/`

`mvn clean package -Pnative -DskipTests`

This will execute `mvn clean package -Pnative` behind the scenes. The `-Pnative` argument selects the native maven profile which invokes the Graal compiler.

> The native compilation could take several minutes, so please be patient!

We've deleted our old build configuration that took a JAR file. We need a new build configuration that can take our new native compiled Quarkus app. Create a new build config with this command:

`oc new-build quay.io/quarkus/ubi-quarkus-native-binary-s2i:19.2.0 --binary --name=payment -l app=payment`

You should get a `--> Success message` at the end.

 * Next, start and watch the build, which will take about 3-4 minutes to complete:

`oc start-build payment --from-file target/*-runner --follow`

This step will combine the native binary with a base OS image, create a new container image, and push it to an internal image registry.

Once that’s done, go to _Builds > Image Streams_ on the left menu then input `payment` to show the payment imagestream. Click on `payment` imagestream:

![serverless]({% image_path payment-is.png %})

In the _Overview_ tab, copy the `IMAGE REPOSITORY` value shown and then open the `payment-service/knative/knative-serving-service.yaml` file and update the `image:` line with this value.

![serverless]({% image_path payment-is-oveview.png %})

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

The service can then be deployed using the following command via CodeReady Workspaces Terminal:

`oc apply -f /projects/cloud-native-workshop-v2m4-labs/payment-service/knative/knative-serving-service.yaml`

After successful creation of the service we should see a Kubernetes Deployment named similar to `payment-v1-deployment` available.

Go to _Home > Status_ on the left menu and click on **payment-v1-deployment**. You will confirm 1 pod is _available_.

![serverless]({% image_path payment-serving-deployment.png %})

In the lab environment, _Knative Serving_ and _Knative Eventing_ components are already installed. Select the `knative-serving` project in the project drop-down selector, then go to `Workloads > Config Maps` in the left menu.

Click on **config-autoscaler**.

![serverless]({% image_path knative-serving-config.png %})

Once you click on **config-autoscaler**, click on the **YAML** tab to show the source code to the config map. Here you will see the details on how Knative autoscaling feature is specified.

As default, Knative will automatically scale services down to zero instances when the service(i.e. payment) has no request after 30 seconds:

 * `scale-to-zero-grace-period: 30s`

![serverless]({% image_path scale-to-zero-grace-period.png %})

In the meantime, it probably took at least 30 seconds so select your `userXX-cloudnativeapps` project using the drop-down at the top and then go back to _Home > Status_ on the left menu and click on **payment-v1-deployment**. You will see 0 pods are available:

![serverless]({% image_path payment-serving-down-to-zero.png %})

You can't access the serverless service using traditional routing (e.g. `oc get route`). Knative maintains its own routing table for managed services. You can list the routes that knative knows of with:

`oc get rt`

~~~console

NAME      URL                                                 READY   REASON
payment   http://payment.userXX-cloudnativeapps.[subdomain]   True
~~~

If you send traffic to this endpoint it will trigger the autoscaler to scale the app up. Trigger the app:

`export SVC_URL=$(oc get rt payment -o template={% raw %}'{{ .status.url }}'{% endraw %})`

`curl -i -H 'Content-Type: application/json' -d '{"foo": "bar"}' $SVC_URL`

This will send some dummy data to the `payment` service,  but more importantly it triggered knative
to spin up the pod again automatically, and will shut it down 30 seconds later.

![serverless]({% image_path payment-serving-magic.png %})

`Congratulations!` You've now deployed the payment service as a Quarkus native image, served with _Knative Serving_, quicker than traditional Java applications. This is not the end of Knative capabilites so we will now see how the payment service will scale up _magically_ in the following exercises.


##### Create KafkaSource to enable Knative Eventing

In this lab, Knative Eventing is already installed but if you want to install it in your own OpenShift cluster then you can install it via the _Knative Eventing Operator_ in [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}.

Open `knative/kafka-event-source.yaml` (in the _payment-service_ project) to define a _KafkaSource_ to integrate with the Knative Eventing. Copy the following YAML code to this file:

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

The object can then be deployed using the following command via CodeReady Workspaces Terminal:

`oc apply -f /projects/cloud-native-workshop-v2m4-labs/payment-service/knative/kafka-event-source.yaml`

![serverless]({% image_path kafka-event-source.png %})

You can also see a new pod spun up which will manage the connection between Kafka and our **payments** service:

`oc get pods -l knative-eventing-source-name=kafka-source`

~~~console
NAME                                 READY   STATUS    RESTARTS   AGE
kafka-source-jktp9-b6b4c6797-4rspk   2/2     Running   1          21s
~~~

Great job!

Let's make sure if the payment service works properly with Knative features via Coolstore Web UI.

#### 4. End to End Functional Testing

---

Before getting started, we need to make sure if _payment service_ is scaled down to _zero_ again in _Project Status_:

![serverless]({% image_path payment-down-again.png %})

Let's go shopping! Open the Web UI in your browser. To get the URL to the Web UI, run this command in CodeReady _Terminal_:

`oc get route | grep coolstore-ui | awk '{print $2}'`

Add some cool items to your shopping cart in the following shopping scenarios:

 * 1) Add a _Froge Laptop Sticker_ to your cart by click on **Add to Cart**. You will see the `Success! Added!` message under the top munu.

![serverless]({% image_path add-to-cart-serverless.png %})

 * 2) Go to the **Your Shopping Cart** tab and click on the **Checkout** button . Input the credit card information. The Card Info should be 16 digits and begin with the digit `4`. For example `4123987754646678`.

![serverless]({% image_path checkout-serverless.png %})

 * 3) Input your Credit Card information to pay for the items:

 ![serverless]({% image_path input-cc-info-serverless.png %})

 * 4) Let's find out how _Kafka Event_ enables _Knative Eventing_. Go back to _Project Status_ in [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} then confirm if _payment service_ is up automatically. It's `MAGIC!!`

  ![serverless]({% image_path payment-up-again.png %})

 * 5) Confirm the _Payment Status_ of the your shopping items in the **All Orders** tab. It should be `Processing`.

 ![serverless]({% image_path payment-processing-serverless.png %})

 * 5) After a few moments, reload the **All Orders** page to confirm that the Payment Status changed to `COMPLETED` or `FAILED`.

 >`Note`: If the status is still `Processing`, the order service is processing incoming Kafka messages and store thme in MongoDB. Please reload the page a few times more.

 ![serverless]({% image_path payment-completedorfailed-serverless.png %})

This is the same result as before, but using Knative eventing to make a more powerful event-driven system that can scale with demand.

#### 5. Creating Cloud-Native CI/CD Pipelines using Tekton

---

##### What is the Cloud-Native CI/CD Pipelines?

There're lots of open source CI/CD tools to build, test, deploy, and manage cloud-native applications/microservices: from on-premise to private, public,
and hybrid cloud. Each tool provides different features to integrate with existing platforms/systems. This sometimes makes it more complex
for DevOps teams to be able to create the CI/CD pipelines and maintain them on Kubernetes clusters. The _cloud-native CI/CD pipeline_ should be defined and executed in
the Kubernetes native way. For example, the pipeline can be specified as Kubernetes resources using YAML format.

_OpenShift Pipelines_ provides a cloud-native, continuous integration and delivery (CI/CD) solution for building pipelines using [Tekton](https://tekton.dev/){:target="_blank"}.

Tekton is a flexible, Kubernetes-native, open-source CI/CD framework that enables automating deployments across multiple platforms (Kubernetes, serverless, VMs, etc)
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

> In the lab, OpenShift Pipelines is already installed on OpenShift cluster but if you want to install OpenShift Pipelines on your own OpenShift cluster, OpenShift Pipelines is provided as an add-on on top of OpenShift that can be installed via an operator available in the OpenShift OperatorHub.

##### What is Tekton?

Tekton defines a number of [Kubernetes custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/){:target="_blank"} as
building blocks in order to standardize pipeline concepts and provide a terminology that is consistent across CI/CD solutions. These custom resources
are an extension of the Kubernetes API that let users create and interact with these objects using the OpenShift CLI (`oc`), `kubectl`, and other Kubernetes tools.

The custom resources needed to define a pipeline are listed below:

 * `Task`: a reusable, loosely coupled number of steps that perform a specific task (e.g. building a container image)

 * `Pipeline`: the definition of the pipeline and the tasks that it should perform

 * `PipelineResource`: inputs (e.g. git repository) and outputs (e.g. image registry) to and out of a pipeline or task

 * `TaskRun`: the execution and result (i.e. success or failure) of running an instance of task

 * `PipelineRun`: the execution and result (i.e. success or failure) of running a pipeline

![severless]({% image_path tekton-arch.png %})

For further details on pipeline concepts, refer to the [Tekton documentation](https://github.com/tektoncd/pipeline/tree/master/docs#learn-more){:target="_blank"} that
provides an excellent guide for understanding various parameters and attributes available for defining pipelines.

In this lab, we will walk you through pipeline concepts and how to create and run a CI/CD pipeline for building and deploying serverless applications on `Knative` on OpenShift.

##### Deploy Sample Application

Change to your developer project for the sample application that you will be using in this lab using this command: (replace `userXX` with your username):

`oc project userXX-cloudnative-pipeline`

You will use the [Spring PetClinic](https://github.com/spring-projects/spring-petclinic){:target="_blank"} sample application during this tutorial, which is a simple Spring Boot application.

Create the Kubernetes objects for deploying the PetClinic app on OpenShift. The deployment will not complete since there are no container images built for the PetClinic application yet. That you will do in the following sections through a CI/CD pipeline.

Replace `userXX-cloudnative-pipeline` with your username in **knative/pipeline/petclinic.yaml**:

![serverless]({% image_path petclinic-namespace.png %})

Then create the object in Kubernetes:

`oc create -f knative/pipeline/petclinic.yaml`

You should be able to see the deployment in the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}.

![serverless]({% image_path petclinic-deployed-1.png %})

##### Install Tasks

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
reusable for multiple pipelines and purposes. You can find more examples of reusable `tasks` in the [Tekton Catalog](https://github.com/tektoncd/catalog){:target="_blank"}
and [OpenShift Catalog](https://github.com/openshift/pipelines-catalog){:target="_blank"} repositories.

Install the `openshift-client` and `s2i-java` tasks from the catalog repository using `oc` or `kubectl`, which you will need for
creating a pipeline in the next section:

Create the following Tekton tasks which will be used in the `Pipelines`:

`oc create -f knative/pipeline/openshift-client-task.yaml`

`oc create -f knative/pipeline/s2i-java-8-task.yaml`

Let's confirm if the **tasks** are installed properly using [Tekton CLI](https://github.com/tektoncd/cli/releases){:target="_blank"} that already installed in CodeReady Workspaces.

`tkn task list`

~~~shell
openshift-client   7 seconds ago
s2i-java-8         3 seconds ago
~~~

##### Create Pipeline

A pipeline defines a number of tasks that should be executed and how they interact with each other via their inputs and outputs.

In this lab, we will create a pipeline that takes the source code of PetClinic application from GitHub and then builds and deploys it on OpenShift using [Source-to-Image (S2I)](https://docs.openshift.com/container-platform/4.1/builds/understanding-image-builds.html#build-strategy-s2i_understanding-image-builds){:target="_blank"}.

![serverless]({% image_path pipeline-diagram.png %})

Here is the YAML file that represents the above pipeline:

~~~yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: petclinic-deploy-pipeline
spec:
  resources:
  - name: app-git
    type: git
  - name: app-image
    type: image
  tasks:
  - name: build
    taskRef:
      name: s2i-java-8
    params:
      - name: TLSVERIFY
        value: "false"
    resources:
      inputs:
      - name: source
        resource: app-git
      outputs:
      - name: image
        resource: app-image
  - name: deploy
    taskRef:
      name: openshift-client
    runAfter:
      - build
    params:
    - name: ARGS
      value:
        - rollout
        - latest
        - spring-petclinic
~~~

This pipeline performs the following:

 * Clones the source code of the application from a Git repository (`app-git` resource)

 * Builds the container image using the `s2i-java-8` task that generates a `Dockerfile` for the application and uses [Buildah](https://buildah.io/){:target="_blank"} to build the image

 * The application image is pushed to an image registry (`app-image` resource)

 * The new application image is deployed on OpenShift using the `openshift-cli`

You might have noticed that there are no references to the PetClinic Git repository and its image in the registry. That's because `Pipelines` in Tekton are designed to be generic and re-usable across environments and stages through the application's lifecycle. `Pipelines` abstract away the specifics of the Git source repository and image to be produced as `resources`. When triggering a pipeline, you can provide different Git repositories and image registries to be used during pipeline execution. Be patient! You will do that in a little bit in the next section.

The execution order of `tasks` is determined by dependencies that are defined between the tasks via `inputs` and `outputs` as well as explicit orders that are defined via `runAfter`.

In the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}, you can click on _Add > Import YAML_ at the top right of the screen while you are in the `userXX-cloudnative-pipeline` project.

![serverless]({% image_path console-import-yaml-1.png %})

Paste the YAML into the textfield, and click on `Create`.

![serverless]({% image_path console-import-yaml-2.png %})

Check the list of pipelines you have created in CodeReady Workspaces Terminal:

`tkn pipeline ls`

~~~shell
NAME                       AGE              LAST RUN   STARTED   DURATION   STATUS
petclinic-deploy-pipeline  8 seconds ago   ---        ---       ---        ---
~~~

##### Trigger Pipeline

Now that the pipeline is created, you can trigger it to execute the tasks specified in the pipeline. Triggering pipelines is an area that is under development and in the next release it will be possible to be done via the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} and Tekton CLI. In this tutorial, you will trigger the pipeline through creating the Kubernetes objects (the hard way!) in order to learn the mechanics of triggering.

First, you should create a number of `PipelineResources` that contain the specifics of the Git repository and image registry to be used in the pipeline during execution. Expectedly, these are also reusable across multiple pipelines.

The following `PipelineResource` defines the Git repository and reference for the PetClinic application. Create the following pipeline resources via the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} via `Add → Import YAML`:

~~~yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-git
spec:
  type: git
  params:
  - name: url
    value: https://github.com/spring-projects/spring-petclinic
~~~

And the following defines the OpenShift internal registry for the PetClinic image to be pushed to. Create the following pipeline resources via the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} via `Add → Import YAML`. Replace your username with `userXX`:

~~~yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-image
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/userXX-cloudnative-pipeline/spring-petclinic
~~~

Create the above pipeline resources via the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} via `Add → Import YAML`.

You can see the list of resources created in CodeReady Workspaces Terminal:

`tkn resource ls`

~~~shell
NAME              TYPE    DETAILS
petclinic-git     git     url: https://github.com/spring-projects/spring-petclinic
petclinic-image   image   url: image-registry.openshift-image-registry.svc:5000/userXX-cloudnative-pipeline/spring-petclinic
~~~

A `PipelineRun` is how you can start a pipeline and tie it to the Git and image resources that should be used for this specific invocation. You can start the pipeline in CodeReady Workspaces Terminal:

~~~shell
tkn pipeline start petclinic-deploy-pipeline \
      -r app-git=petclinic-git \
      -r app-image=petclinic-image \
      -s pipeline
~~~

The result looks like:

`Pipelinerun started: petclinic-deploy-pipeline-run-97kdv`

The `-r` flag specifies the PipelineResources that should be provided to the pipeline and the `-s` flag specifies the service account to be used for running the pipeline.

As soon as you started the `petclinic-deploy-pipeline pipeline`, a pipelinerun is instantiated and pods are created to execute the tasks that are defined in the pipeline.

`tkn pipeline list`

~~~shell
NAME                        AGE              LAST RUN                              STARTED          DURATION   STATUS
petclinic-deploy-pipeline   21 seconds ago   petclinic-deploy-pipeline-run-97kdv   11 seconds ago   ---        Running
~~~

Check out the logs of the pipeline as it runs using the `tkn pipeline logs` command which interactively allows you to pick the pipelinerun of your interest and inspect the logs:

`tkn pipeline logs -f`

~~~shell
? Select pipeline : petclinic-deploy-pipeline
? Select pipelinerun : petclinic-deploy-pipeline-run-97kdv started 39 seconds ago

...
[build : push] Copying config sha256:6c2be43b49deee05b0dee97bd23dab0dcfd9b1b6352fd085f833f62e7d106ae8
[build : push] Writing manifest to image destination
[build : push] Copying config sha256:6c2be43b49deee05b0dee97bd23dab0dcfd9b1b6352fd085f833f62e7d106ae8
[build : push] Writing manifest to image destination
...
[build : image-digest-exporter-bj6dr] 2019/09/17 05:06:09 Image digest exporter output: []
[deploy : oc] deploymentconfig.apps.openshift.io/spring-petclinic rolled out

~~~

>`Note`: The build log(_ImageResource petclinic-image doesn't have an index.json file_) doesn't mean an error but it's vailation check. Even if you're failed, **Pipeline Build** will continue.

After a few minutes, the pipeline should finish successfully.

`tkn pipeline list `

~~~shell
NAME                        AGE             LAST RUN                              STARTED         DURATION    STATUS
petclinic-deploy-pipeline   7 minutes ago   petclinic-deploy-pipeline-run-97kdv   5 minutes ago   4 minutes   Succeeded
~~~

Looking back at the project, you should see that the PetClinic image is successfully built and deployed.

![serverless]({% image_path petclinic-deployed-2.png %})

### Summary

In this module, we learned how to develop cloud-native applications using multiple Java runtimes (Quarkus and Spring Boot), Javascript (Node.js) and different datasources (i.e. PostgreSQL, MongoDB) to handle a variety of business use cases which implement real-time _request/response_ communication using REST APIs, high performing cacheable services using **JBoss Data Grid**, event-driven/reactive shopping cart service using Apache Kafka in **Red Hat AMQ Streams**, and in the end, we treated the payment service as a `Serverless` application using `Knative` with Serving, Eventing, and Pipeline(Tekton).

**Red Hat Runtimes** enables enterprise developers to design the advanced cloud-native architecture and develop, build, deploy the cloud-native application on hybrid cloud on the **Red Hat OpenShift Container Platform**. Congratulations!

##### Additional Resources:

 * [Knative on OpenShift](https://www.openshift.com/learn/topics/knative){:target="_blank"}

 * [Knative Install on OpenShift](https://knative.dev/docs/install/knative-with-openshift/){:target="_blank"}

 * [Knative Tutorial](https://redhat-developer-demos.github.io/knative-tutorial){:target="_blank"}

 * [Knative, Serverless Kubernetes Blogs](https://developers.redhat.com/topics/knative/){:target="_blank"}

 * [7 open source platforms to get started with serverless computing](https://opensource.com/article/18/11/open-source-serverless-platforms){:target="_blank"}

 * [How to develop functions-as-a-service with Apache OpenWhisk](https://opensource.com/article/18/11/developing-functions-service-apache-openwhisk){:target="_blank"}

 * [How to enable serverless computing in Kubernetes](https://opensource.com/article/19/4/enabling-serverless-kubernetes){:target="_blank"}
