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


Additionally, we have also chosen to use Quarkus for most of the applications in the labs. Read on to learn more about Quarkus. 

> NOTE: At the time of writing this guide, Quarkus is still a community project and is not part of any of the Red Hat Middleware products.


##### What is Quarkus? 

![quarkus-logo]({% image_path quarkus-logo.png %})

For years, the client-server architecture has been the de-facto standard to build applications. 
But a major shift happened. The one model rules them all age is over. A new range of applications 
and architecture styles has emerged and impacts how code is written and how applications are deployed and executed. 
HTTP microservices, reactive applications, message-driven microservices and serverless are now central players in modern systems.

[Qurakus](https://Quarkus.io/) offers 4 major benefits to build cloud-native, microservices, and serverless Java applicaitons:

* *Developer Joy* - Cohesive platform for optimized developer joy through unified configuration, Zero config with live reload in the blink of an eye,
   streamlined code for the 80% common usages with flexible for the 20%, and no hassle native executable generation.

* *Unifies Imperative and Reactive* - Inject the EventBus or the Vertx context for both Reactive and imperative development in the same application.

* *Functions as a Service and Serverless* - Superfast startup and low memory utilization. With Quarkus, you can embrace this new world without having 
  to change your programming language.

* *Best of Breed Frameworks & Standards* - CodeReady Workspaces Vert.x, Hibernate, RESTEasy, Apache Camel, CodeReady Workspaces MicroProfile, Netty, Kubernetes, OpenShift,
  Jaeger, Prometheus, Apacke Kafka, Infinispan, and more.


#### Getting Ready for the labs

---

##### Access Your Development Environment

You will be using Red Hat CodeReady Workspaces, an online IDE based on [Eclipe Che](https://www.eclipse.org/che/){:target="_blank"}. **Changes to files are auto-saved every few seconds**, so you don't need to explicitly save changes.

To get started, [access the Che instance]({{ ECLIPSE_CHE_URL }}) and log in using the username and password you've been assigned (e.g. `{{ CHE_USER_NAME }}/{{ CHE_USER_PASSWORD }}`):

![cdw]({% image_path che-login.png %})

Once you log in, you'll be placed on your personal dashboard. We've pre-created workspaces for you to use. Click on the name of the pre-created workspace on the left, as shown below (the name will be different depending on your assigned number). You can also click on the name of the workspace in the center, and then click on the green button that says "OPEN" on the top right hand side of the screen:

![cdw]({% image_path che-precreated.png %})

After a minute or two, you'll be placed in the workspace:

![cdw]({% image_path che-workspace.png %})

You might see **Workspace agent is not running** in the popup message, click on `Restart`:

![cdw]({% image_path che-workspace-restart.png %})

To gain extra screen space, click on the yellow arrow to hide the left menu (you won't need it):

![cdw]({% image_path che-realestate.png %})

Users of Eclipse, IntelliJ IDEA or Visual Studio Code will see a familiar layout: a project/file browser on the left, a code editor on the right, and a terminal at the bottom. You'll use all of these during the course of this workshop, so keep this browser tab open throughout. **If things get weird, you can simply reload the browser tab to refresh the view.**

In the project explorer pane, click on the `Import Projects...` and enter the following:

  * Version Control System: `GIT`
  * URL: `{{GIT_URL}}/userXX/cloud-native-workshop-v2m4-labs.git`(IMPORTANT: replace userXX with your lab user)
  * Check `Import recursively (for multi-module projects)`
  * Name: `cloud-native-workshop-v2m4-labs`

`Tip`: You can find GIT URL when you click on {{GIT_URL}} then login with your credentials. 

![codeready-workspace-import]({% image_path codeready-workspace-import.png %}){:width="700px"}

The projects are imported now into your workspace and is visible in the project explorer.

CodeReady Workspaces is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to a Maven projects. In the project explorer, right-click on each project and 
then click on `Convert to Project` continuously.

![codeready-workspace-convert]({% image_path codeready-workspace-convert.png %}){:width="500px"}

Choose `Maven` from the project configurations and then click on `Save`.

![codeready-workspace-maven]({% image_path codeready-workspace-maven.png %}){:width="700px"}

Repeat the above for inventory and catalog projects.

> `NOTE`: the Terminal window in CodeReady Workspaces. For the rest of these labs, anytime you need to run a command in a terminal, you can use the CodeReady Workspaces `Terminal` window.

![codeready-workspace-terminal]({% image_path codeready-workspace-terminal.png %})

##### Connnecting to Openshift

When you access [OpenShift web console]({{ CONSOLE_URL}}) or the other route URL via HTTPS protocol, you will see `Your Connection is not secure` warning message.
Because, OpenShift uses self-certification to create TLS termication route as default. For example, if you're using *Chrome Browser*, you will see the following screen.

Click on `Advanced` then, you can access the HTTPS page when you click on `Proceed to...`!!!

![warning]({% image_path browser_warning.png %})
