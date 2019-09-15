## Cloud Native Application Architectures

Developing applications that are reactive, imperative, event driven, and polyglot are all requirements of modern application architecture. While cloud infrastructure and container Kubernetes solutions, such as Red Hat OpenStack Platform and Red Hat OpenShift, provide a robust infrastructure foundation for distributed environments, similar seamless application services require building applications that take full advantage of such an infrastructure.

Moreover, applications require hybrid and multi cloud architecture to thrive in the digital economy. A unified cloud native application environment has become essential to developers because it enables higher productivity and innovation with a full, cohesive development platform. An application environment is equally critical to operations because of the rapid change and high demands for scalability, agility, and reliability.

In this module, we will learn what cloud-native architecture we need to design for running containerized applications on DevOps/Cloud Native platform in scale and speed.
Then we will also develop cloud native applications based on architecture patterns such as high-performing cache, event-driven/reactive, and serverless using
[Red Hat Runtimes](https://www.redhat.com/en/technologies/cloud-computing/openshift/application-runtimes), [Red Hat CodeReady Workspaces](https://developers.redhat.com/products/codeready-workspaces/overview) and 
[Red Hat OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift).


#### Capabilities of a Cloud Native Application Architectures?
---
The benefits of cloud native application architectures enable speed of development and deployment, flexibility, quality, and reliability. More importantly, it allows developers to integrate the applications with the latest open source technologies without a steep learning curve. While there are many ways to build and architect cloud native applications following are some great ingredients for consideration:

 * *Runtimes* - More likely to be written in the container first or/and Kubernetes native language, which means runtimes such as Java, Node.js, Go, Python, and Ruby, etc.

 * *Security* - Deploying and maintaining applications in a multi cloud, hybrid cloud application environment, security becomes of utmost importance and should be part of the environment.

 * *Observability* - The ability to observer applications and their behavior in the cloud. Tools that can give realtime metrics, and more information about the use e.g., Prometheus, Grafana, Kiali, etc.

 * *Efficiency* - Focused on tiny memory footprint, small artifact size, and fast booting time to make portable applications across hybrid/multi-cloud platforms. Primarily, it will leverage an expected spice in production via rapid scaling with consuming minimal computing resources.

 * *Interoperability* - Easy to integrate cloud native apps with the latest open source technologies such as Infinispan, MicroProfile, Hibernate, Apache Kafka, Jaeger, Prometheus, and more for building standard runtimes architecture.

 * *DevOps/DevSecOps* - Designed for continuous deployment to production in line with the minimum viable product (MVP), with security as part of the tooling together with development, automating testing, and collaboration.


#### How to build Cloud Native applications and architecture with Red Hat?

---

`Red Hat Runtime​s​` is a recommended set of products, tools, and components to develop and maintain cloud-native applications. It provides lightweight runtimes and frameworks for highly-distributed cloud environments such as microservices, with in-memory caching for fast data access, and messaging for quick data transfer supporting existing applications. 

Red Hat Runtime​s​ products and components:

![rhar]({% image_path rhar.png %})

 * *Red Hat® JBoss® Enterprise Application Platform 7​​ (JBoss EAP)* is the market-leading opensource platform for modern Java applications deployed in any environment. 
    JBoss EAP’s architecture is lightweight, modular, and cloud ready. Based on the opensource WildFly app server project, the platform offers powerful management and automation for higher developer productivity.

 * *A set of cloud-native runtimes* are Spring Boot with Tomcat, Reactive Vert.x, Javascript Node.Js, MicroProfile Throntail.(Quarkus is coming soon!) 

 * *Red Hat OpenJDK​* is an opensource implementation of the Java Platform SE (Standard Edition) supported and maintained by the OpenJDK community. OpenJDK is the default Java development and runtime in Red Hat Enterprise Linux.

 * *Red Hat Data Grid​*​ is an opensource ​in-memory distributed data management system designed for scalability and fast access to large volumes of data. 
    More than just a distributed caching solution, it also offers additional functionality such as map/reduce, querying, processing for streaming data, and 
    transaction capabilities​.

 * *Red Hat AMQ​​ (Broker)* is a pure-Java multiprotocol message broker that offers specialized queueing behaviors, message persistence, and manageability.
 
 * *Red Hat Application Migration Toolkit​*​ provides a set of utilities for easing the process of taking customers’ proprietary or outdated middleware platforms to 
    state-of-the-art lightweight, modular, and cloud-ready middleware application infrastructure is making teams more productive and ready for the future.

 * *Missions and Boosters*​​ are a combination of runtime implementations and working applications that accelerate application development. ​Missions are working applications that showcase different fundamental pieces of building cloudnative applications and services. A booster is the implementation of a mission in a specific runtime.

 * *Red Hat Single Sign-On​​* based on the Keycloak project, Red Hat sso enables customers to secure web applications byproviding Web single sign-on) capabilities 
    based on popular standards such as SAML 2.0, OpenID Connect and OAuth 2.0.The RH-sso server can act as a SAML or OpenID Connect-based identity provider, 
    mediating your enterprise user directoryor 3rd-party SSO provider for identity information with your applications via standards-based tokens.

Red Hat Runtimes​​ also provide integrated and optimized products and components to deliver modern applications, whether the goal is to keep existing applications or create new ones. Applications Runtimes enable developers to containerize applications with a microservices architecture, improve data access speed via in-memory data caching, enhance application performance with messaging, or adapt cloud-native application development using modern development patterns and technologies.


Additionally we have also chosed to use Quarkus for most of the applications in the labs. Read on to learn more about Quarkus. 

> NOTE: At the time of writing this guide, Quarkus is still a community project and is not part of any of the Red Hat Middleware products.


##### What is Quarkus? 

![quarkus-logo]({% image_path quarkus-logo.png %})

For years, the client-server architecture has been the de-facto standard to build applications. 
But a major shift happened. The one model rules them all age is over. A new range of applications 
and architecture styles has emerged and impacts how code is written and how applications are deployed and executed. 
HTTP microservices, reactive applications, message-driven microservices and serverless are now central players in modern systems.

[Qurakus](https://Quarkus.io/) offers 4 major benefits to build cloud-native, microservices, and serverless Java applicaitons:

* `Developer Joy` - Cohesive platform for optimized developer joy through unified configuration, Zero config with live reload in the blink of an eye,
   streamlined code for the 80% common usages with flexible for the 20%, and no hassle native executable generation.

* `Unifies Imperative and Reactive` - Inject the EventBus or the Vertx context for both Reactive and imperative development in the same application.

* `Functions as a Service and Serverless` - Superfast startup and low memory utilization. With Quarkus, you can embrace this new world without having 
  to change your programming language.

* `Best of Breed Frameworks & Standards` - CodeReady Workspaces Vert.x, Hibernate, RESTEasy, Apache Camel, CodeReady Workspaces MicroProfile, Netty, Kubernetes, OpenShift,
  Jaeger, Prometheus, Apacke Kafka, Infinispan, and more.


### Getting Ready for the labs

#### Connnecting to Openshift
When you access OpenShift web console or the other route URL via `HTTPS` protocol, you will see `Your Connection is not secure` warning message.
Because, OpenShift uses self-certification to create TLS termication route as default. For example, if you're using `Firefox`, you will see the following screen.

Click on `Advanced > Add Exception...` then, you can access the `HTTPS` page when you click on `Confirm Security Exception`!!!

![warning]({% image_path browser_warning.png %})

#### Setup CodeReady Workspaces for Lab Environment

---

> SKIP this setup guide if you have already completed this earlier in the previous Modules


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
  * URL: `{{GIT_URL}}/userXX/cloud-native-workshop-v2m4-labs.git`
  * Check `Import recursively (for multi-module projects)`
  * Name: `cloud-native-workshop-v2m4-labs`

![codeready-workspace-import]({% image_path codeready-workspace-import.png %}){:width="700px"}

The projects are imported now into your workspace and is visible in the project explorer.

CodeReady Workspaces is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to a Maven projects. In the project explorer, right-click on each project and 
then click on `Convert to Project` continuously.

![codeready-workspace-convert]({% image_path codeready-workspace-convert.png %}){:width="500px"}

Choose `Maven` from the project configurations and then click on `Save`.

![codeready-workspace-maven]({% image_path codeready-workspace-maven.png %}){:width="700px"}

Repeat the above for inventory and catalog projects.

> `NOTE`: the Terminal window in CodeReady Workspaces. For the rest of these labs, anytime you need to run 
a command in a terminal, you can use the CodeReady Workspaces `Terminal` window.

![codeready-workspace-terminal]({% image_path codeready-workspace-terminal.png %})
