# cloud-native-workshop-v2m4-guides
This is the module 4 guides of the Cloud Native workshop v2.

# Lab Instructions on OpenShift

To deploy the workshop, it is recommended you first create a fresh project into which to deploy it. The same project will then be used by the workshop as it steps you through the different ways workshops can be deployed.

```
oc new-project labs
```

In the active project you want to use, run:

```oc new-app https://raw.githubusercontent.com/openshift-labs/workshop-dashboard/3.5.0/templates/production.json \
  --param TERMINAL_IMAGE="quay.io/quaa/cloud-native-workshop-v2m4-guides:latest" \
  --param APPLICATION_NAME=cn-workshop-v2m4 \
  --param AUTH_USERNAME=workshop \
  --param AUTH_PASSWORD=workshop
```

Access to the workshop environment will be password protected and you will see a browser popup for entering user credentials. The username to enter is `workshop`. The password will also be `workshop`.

You can use a different password by changing the value passed to the `AUTH_PASSWORD` template parameter.

The hostname for accessing the workshop environment in your browser can be found by running:

```
oc get route cn-workshop-v2m4
```
When you are finished you can delete the project you created..

If you didn't follow the recommendation of using a new project, and instead used an existing project, run:

```
oc delete all,serviceaccount,rolebinding,configmap -l app=cn-workshop-v2m4
```

Note that this will not delete anything which may have been deployed when you went through the workshop. If you used an existing project, ensure that you go right through the workshop and execute any steps described in it for deleting any deployments it had you make.
