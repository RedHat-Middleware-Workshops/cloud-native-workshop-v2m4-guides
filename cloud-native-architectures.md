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


#### What application environment should be considered for the architecture?

---

`Red Hat Applicaton Runtime​s​` is a recommended set of products, tools and components to develop and maintaincloud-native applications. It provides lightweight 
runtimes and frameworks for highly-distributed cloud environments such asmicroservices, with in-memory caching for fast data access, and messaging for quick data 
transfer supporting existingapplications. Red Hat Middleware products and components included in this portfolio offering are:

 * `Red Hat® JBoss® Enterprise Application Platform 7​​ (JBoss EAP)` - is the market-leading open source platform formodern Java applications deployed in any environment. 
    JBoss EAP’s architecture is lightweight, modular, and cloud ready.Based on the open source WildFly app server project, the platform offers powerful management and 
    automation for greaterdeveloper productivity.

 * `Red Hat OpenJDK​`​ - is an open source implementation of the Java Platform SE (Standard Edition) supported andmaintained 
    by the OpenJDK community. OpenJDK is the default Java development and runtime in Red Hat EnterpriseLinux.

 * `Red Hat Data Grid​`​ - is an open source ​in-memory distributed data management system designed for scalability and fastaccess to large volumes of data. 
    More than just a distributed caching solution, it also offers additional functionality such asmap/reduce, querying, processing for streaming data, and 
    transaction capabilities​.

 * `Red Hat AMQ​​ (Broker)` - is a pure-Java multiprotocol message broker that offers specialized queueing behaviors, messagepersistence, and manageability.
 * `Red Hat Application Migration Toolkit​`​ - provides a set of utilities for easing the process of taking customers’ proprietaryor outdated middleware platforms to 
    state-of-the-art lightweight, modular and cloud-ready middleware applicationinfrastructure making teams more productive and ready for the future.

 * `Missions and Boosters`​​ - are a combination of runtime implementations and working applications that accelerateapplication development. ​Missions are working 
    applications that showcase different fundamental pieces of building cloudnative applications and services. A booster is the implementation of a mission in a specific runtime.

 * `Red Hat Single Sign-On​​` - Based on the Keycloak project, Red Hat sso enables customers to secure web applications byproviding Web single sign-on) capabilities 
    based on popular standards such as SAML 2.0, OpenID Connect and OAuth 2.0.The RH-sso server can act as a SAML or OpenID Connect-based identity provider, 
    mediating your enterprise user directoryor 3rd-party SSO provider for identity information with your applications via standards-based tokens.

![rhar]({% image_path rhar.png %})

Red Hat Applications Runtimes​​ also provide integrated and optimized products and components to deliver modernapplications, whether the goal is to keep 
existing applications or create new ones. Applications Runtimes enable developers to containerize applications with a microservices architecture, improve data access 
speedvia in-memory data caching, enhance application performance with messaging, or adapt cloud-native application development using moderndevelopment patterns and technologies.


#### Your Connection is not secure?

---

When you access OpenShift web console or the other route URL via `HTTPS` protocol, you will see `Your Connection is not secure` warning message.
Because, OpenShift uses self-certification to create TLS termication route as default. For example, if you're using `Firefox`, you will see the following screen.

Click on `Advanced > Add Exception...`.

![warning]({% image_path browser_warning.png %})

Then, you can access the `HTTPS` page when you click on `Confirm Security Exception`

![warning]({% image_path browser_warning_confirmation.png %})