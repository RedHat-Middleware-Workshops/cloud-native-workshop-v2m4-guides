## Lab2 - Creating Event-Driven/Reactive Services

In this lab, we'll develop `Event-Driven/Reactive` applications into the cloud-native appliation architecture. These cloud-native applications...

#### Goals of this lab

---

The goal is to develop advanced cloud-native applications on `Red Hat Runtimes` and deploy them on `OpenShift 4` including 
`Kafka in Red Hat Integration` for distributed messaing capabilities. After this lab, you should end up with something like:

![goal]({% image_path lab2-goal.png %})

####1. Creating Event-Driven/Reactive Services

---


Lab 2 Reliable Async with AMQ - replace rest calls for order to inventory, shopping cart and payment service and introduce compensating transactions (Saga pattern)
Developer creates Queues and changes the Order service to use XXX to send an order event.
User orders and pays with a valid CC (4444333322221111)
Verifies that the payments succeeds, inventory has been updated and an order exists in the order history of the user
User orders and pays with an invalid CC (1111222233334444)
User verifies that the Payment failed, inventory has been rollback and no order exists 

Lab 3: Replace imperative catalog service with a reactive catalog service
