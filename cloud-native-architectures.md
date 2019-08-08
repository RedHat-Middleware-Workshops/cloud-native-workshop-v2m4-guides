## Cloud-Native Application Architectures

In this module, we will learn what cloud-native architecture we need to design for running containerized applications on DevOps/Cloud-Native platform in scale and speed.
Then we will also develop cloud-natvie applications align with multiple architecture patterns such as high-performing cache, event-driven/reactive, and serverless using 
[Red Hat Application Runtimes](https://www.redhat.com/en/technologies/cloud-computing/openshift/application-runtimes), [Red Hat CodeReady Workspaces](https://developers.redhat.com/products/codeready-workspaces/overview) and 
[Red Hat OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift).


#### What is Cloud-Native Application Architectures?

---

Cloud-native is a new paradigm to develop application services and manage them on cloud computing infrastructure 
which has elasticity, scalability, resiliency, and high-performance. Cloud-native application architectures enable 
developers/architects to implement various application patterns from imperative to reactive, serverless in continuous delivery workflows.

The benefits of cloud-native application architectures are to augment development productivity in terms of speed, 
flexibility, quality, and reliability. More importantly, it allows developers to integrate the applications with the 
latest open source technologies without a steep learning curve. For example, with the architectures, you can deal with a certain 
microservices as serverless workloads for business seasonality. More characteristics that cloud-native application architectures should have here:

 * `Programming languages` - More likely to be written in the container first or/and Kubernetes native language, which means Java, Node.js, Go, Python, and Ruby.

 * `Efficiency` - Focused on tiny memory footprint, small artifact size, and fast booting time to make portable applications across hybrid/multi-cloud platforms. Especially, it will leverage an expected spice in production via fast scaling with consuming minimal computing resources.

 * `Interoperability` - Easy to integrate cloud-native apps with the latest open source technologies such as Infinispan, MicroProfile, Hibernate, Apache Kafka, Jaeger, Prometheus, and more for building standard runtimes architecture.

 * `DevOps` - Designed for continuous deployment to production in line with minimum viable product (MVP) development, automating testing, and collaborating ops teams in DevOps practices.


#### How Red Hat support the Cloud-Native App Architectures?

---

`Red Hat Applicaton Runtime​s​` is a recommended set of products, tools and components to develop and maintaincloud-native applications. It provides lightweight 
runtimes and frameworks for highly-distributed cloud environments such asmicroservices, with in-memory caching for fast data access, and messaging for quick data 
transfer supporting existingapplications. Features and benifits are here:

 * `Runtimes and frameworks` - A collection of runtimes, frameworks, and languages so developers and architects can choose the right tool for the right task.

 * `Distributed, in-memory caching` - An in-memory distributed data management system designed for scalability and fast access to large volumes of data.
 
 * `Single sign-on` - Enable developers to provide web single sign-on capabilities based on industry standards for enterprise security.
 
 * `Messaging` - A message broker that offers specialized queueing behaviors, message persistence, and manageability.
 
 * `Launcher service` - Build and deploy a new application in minutes. This service creates application scaffolding so you can focus on writing business logic and delivering value.
 
 * `OpenJDK` - The Red Hat build of OpenJDK is an open source implementation of the Java™ Platform, Standard Edition (Java SE) supported and maintained by the OpenJDK community.

Red Hat Applicaton Runtime​s​ products and components included in this portfolio offering are:

![rhar]({% image_path rhar.png %})

 * `Red Hat® JBoss® Enterprise Application Platform 7​​ (JBoss EAP)` is the market-leading open source platform formodern Java applications deployed in any environment. 
    JBoss EAP’s architecture is lightweight, modular, and cloud ready.Based on the open source WildFly app server project, the platform offers powerful management and 
    automation for greaterdeveloper productivity.

 * `A set of cloud-native runtimes` are Spring Boot with Tomcat, Reactive Vert.X, Javascript Node.Js, MicroProfile Throntail.(`Quarkus is coming soon!!`) 

 * `Red Hat OpenJDK​`​ is an open source implementation of the Java Platform SE (Standard Edition) supported andmaintained 
    by the OpenJDK community. OpenJDK is the default Java development and runtime in Red Hat EnterpriseLinux.

 * `Red Hat Data Grid​`​ is an open source ​in-memory distributed data management system designed for scalability and fastaccess to large volumes of data. 
    More than just a distributed caching solution, it also offers additional functionality such asmap/reduce, querying, processing for streaming data, and 
    transaction capabilities​.

 * `Red Hat AMQ​​ (Broker)` is a pure-Java multiprotocol message broker that offers specialized queueing behaviors, messagepersistence, and manageability.
 
 * `Red Hat Application Migration Toolkit​`​ provides a set of utilities for easing the process of taking customers’ proprietaryor outdated middleware platforms to 
    state-of-the-art lightweight, modular and cloud-ready middleware applicationinfrastructure making teams more productive and ready for the future.

 * `Missions and Boosters`​​ are a combination of runtime implementations and working applications that accelerateapplication development. ​Missions are working 
    applications that showcase different fundamental pieces of building cloudnative applications and services. A booster is the implementation of a mission in a specific runtime.

 * `Red Hat Single Sign-On​​` based on the Keycloak project, Red Hat sso enables customers to secure web applications byproviding Web single sign-on) capabilities 
    based on popular standards such as SAML 2.0, OpenID Connect and OAuth 2.0.The RH-sso server can act as a SAML or OpenID Connect-based identity provider, 
    mediating your enterprise user directoryor 3rd-party SSO provider for identity information with your applications via standards-based tokens.

Red Hat Applications Runtimes​​ also provide integrated and optimized products and components to deliver modernapplications, whether the goal is to keep 
existing applications or create new ones. Applications Runtimes enable developers to containerize applications with a microservices architecture, improve data access 
speedvia in-memory data caching, enhance application performance with messaging, or adapt cloud-native application development using moderndevelopment patterns and technologies.

#### What is Quarkus? 

---

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


#### Your Connection is not secure?

---

When you access OpenShift web console or the other route URL via `HTTPS` protocol, you will see `Your Connection is not secure` warning message.
Because, OpenShift uses self-certification to create TLS termication route as default. For example, if you're using `Firefox`, you will see the following screen.

Click on `Advanced > Add Exception...`.

![warning]({% image_path browser_warning.png %})

Then, you can access the `HTTPS` page when you click on `Confirm Security Exception`

![warning]({% image_path browser_warning_confirmation.png %})