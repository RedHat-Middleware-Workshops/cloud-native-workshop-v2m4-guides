## Lab3 - Evoling Serverless Services

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

Create a native executable by once again opening the command palette and choose `Build Native Quarkus App`. This will execute `mvn clean package -Pnative` 
behind the scenes. The `-Pnative` argument selects the native maven profile which invokes the Graal compiler.

`This will take about 3-4 minutes to finish. Wait for it!`

![serverless]({% image_path native-image-build.png %})

> NOTE: You can safely ignore any warnings like `Warning: RecomputeFieldValue.FieldOffset automatic substitution failed`. 
These are harmless and will be removed in future releases of Quarkus. 

> NOTE: Since we are on Linux in this environment, and the OS that will eventually run our application is also Linux, we can use our local OS to 
build the native Quarkus app. If you need to build native Linux binaries when on other OS’s like Windows or Mac OS X, you’ll need to have Docker 
installed and then use `mvn clean package -Pnative -Dnative-image.docker-build=true -DskipTests=true`.

In addition to the regular files, the build will produce `target/payment-1.0-SNAPSHOT-runner`. This is a native Linux binary. Not a shell script, 
or a JAR file, but a native binary.


##### Run native image

Since our environment here is Linux, you can just run it. In the CodeReady Workspaces Terminal, run:

`target/payment-1.0-SNAPSHOT-runner`

Notice the amazingly fast startup time:

~~~shell
2019-08-22 04:04:11,817 INFO  [io.quarkus] (main) Quarkus 0.21.1 started in 0.015s. Listening on: http://[::]:8080
2019-08-22 04:04:11,818 INFO  [io.quarkus] (main) Installed features: [agroal, cdi, jdbc-h2, narayana-jta, resteasy]
~~~

That’s `15 milliseconds` to start up.

And extremely low memory usage as reported by the Linux `ps` utility. While the app is running, open another Terminal 
(click the `+` button on the terminal tabs line) and run:

`ps -o pid,rss,command -p $(pgrep -f runner)`

You should see something like:

~~~shell
   PID   RSS COMMAND
 16017 53816 target/payment-1.0-SNAPSHOT-runner
~~~

This shows that our process is taking around `50 MB` of memory ([Resident Set Size](https://en.wikipedia.org/wiki/Resident_set_size), or RSS). Pretty compact!

> NOTE: The RSS and memory usage of any app, including Quarkus, will vary depending your specific environment, and will rise as the application experiences load.

Make sure the app is still working as expected (we’ll use `curl` this time to access it directly). In a new CdeReady Workspaces Terminal run:

`curl http://localhost:8080/api/payment`

You should see:

`hello quarkus from <your-hostname>`

`Congratuations!` You’ve now built a Java application as an executable JAR and a Linux native binary. We’ll explore the benefits of native 
binaries later in when we start deploying to Kubernetes.

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

In the lab, `Knative Serving` is already installed on OpenShift cluster but if you want to install Knative Serving on your own OpenShift cluster, 
you can play with [Installing the Knative Serving Operator](https://knative.dev/docs/install/knative-with-openshift/) as below:

![serverless]({% image_path knative_serving_tile_highlighted.png %})


####3. Accessing your application

---

The application is now exposed as an internal service. If you are using minikube or minishift, you can access it using:

INGRESSGATEWAY=istio-ingressgateway
IP_ADDRESS="$(minikube ip):$(kubectl get svc $INGRESSGATEWAY --namespace istio-system --output 'jsonpath={.spec.ports[?(@.port==80)].nodePort}')" 

curl -v -H 'Host: getting-started-knative.example.com' $IP_ADDRESS/hello/greeting/redhat

you can replace minikube ip with minishift ip if you are using OpenShift

####4. Creating Tekton Pipeline in OpenShift

---

##### Deploying Nexus

To make maven builds faster, we will deploy `Sonatype Nexus`

`oc new-app sonatype/nexus`

##### Creating Tekton Tasks

Create the following Tekton tasks which will be used in the `Pipelines`

~~~shell
oc create -f https://raw.githubusercontent.com/tektoncd/catalog/master/openshift-client/openshift-client-task.yaml \
  -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/quarkus/s2i-quarkus-task.yaml \
  -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/quarkus/quarkus-native-task.yaml \
  -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/quarkus/quarkus-jvm-task.yaml \
  -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/knative-client/kn-service-create-task.yaml
~~~

Check the tasks created:

`tkn task ls`

~~~shell
NAME                AGE
kn-create-service   12 seconds ago
openshift-client    13 seconds ago
quarkus-jvm         12 seconds ago
quarkus-native      12 seconds ago
s2i-quarkus         13 seconds ago
~~~

~~~shell
oc create serviceaccount pipeline && \
oc adm policy add-scc-to-user privileged -z pipeline && \
oc adm policy add-role-to-user edit -z pipeline && \
oc adm policy add-scc-to-user privileged -z default && \
oc adm policy add-scc-to-user anyuid -z default && \
oc create -f https://raw.githubusercontent.com/redhat-developer-demos/pipelines-catalog/master/knative-client/pipeline-sa-roles.yaml -n userXX-cloudnativeapps && \
oc policy add-role-to-user pipeline-roles -z pipeline --role-namespace=userXX-cloudnativeapps
~~~

##### Additional Resources:

 * [Knative on OpenShift](https://www.openshift.com/learn/topics/knative)

 * [Knative Install on OpenShift](https://knative.dev/docs/install/knative-with-openshift/)

 * [Knative Tutorial](https://redhat-developer-demos.github.io/knative-tutorial)

 * [Knative, Serverless Kubernetes Blogs](https://developers.redhat.com/topics/knative/)

 * [7 open source platforms to get started with serverless computing](https://opensource.com/article/18/11/open-source-serverless-platforms)

 * [How to develop functions-as-a-service with Apache OpenWhisk](https://opensource.com/article/18/11/developing-functions-service-apache-openwhisk)

 * [How to enable serverless computing in Kubernetes](https://opensource.com/article/19/4/enabling-serverless-kubernetes)
