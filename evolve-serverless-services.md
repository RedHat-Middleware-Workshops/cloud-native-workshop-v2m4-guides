## Lab4 - Evoling Serverless Services

In this cloud-native application architecture, We now have multiple microservices to implement traditional application workloads and reactive system. 
However, it's not necessary that whole applications/services have to run all the time(24/7) for serving funtions because a certain service can be 
responsed `on demain`. The reason why `serverless` application came up in the advanced cloud-native architecture. Before getting started with developing 
serverless applcation, we will walk you through quickly what `serverless` means as below:

 * The term [serverless](https://enterprisersproject.com/article/2018/9/what-serverless) has been coming up in more conversations recently. 
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

The severless application enables `DevOps` teams to enjoy benefits as here:

 * Optimizing computing resources(i.e CPU, Memory, Networking, and Storage)
 * Autoscaling on demand
 * Simplifying CI/CD pipeline

In this lab, we'll deploy the `Payment Service` as a serverless application using `Knative Serving`, `Istio`, `Tekton Pipelines` and `Quarkus native application` 
because the payment service doesn't need to be served for the end users as the other services like the shopping cart and catalog services.

[Knative](https://cloud.google.com/knative/) was born for developers to create serverless experiences natively without depending on extra serverless or 
FaaS frameworks and many custom tools. Knative has three primary components—[Build](https://github.com/knative/build), 
[Serving](https://github.com/knative/serving), and [Eventing](https://github.com/knative/eventing)—for addressing common patterns 
and best practices for developing serverless applications on Kubernetes platforms.

[Istio](istio.io) is an open, platform-independent service mesh designed to manage communications between microservices and
applications in a transparent way.It provides behavioral insights and operational control over the service mesh
as a whole. It provides a number of key capabilities uniformly across a network of services such as `Traffic Management`, `Observability`, 
`Policy Enforcement`, and `Service Identity and Security`.

####1. Developing Native Application on Quarkus

---

aa

####2. Deploying the application in Knative

---

aa

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

