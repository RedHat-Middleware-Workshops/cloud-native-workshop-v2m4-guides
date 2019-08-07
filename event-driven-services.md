## Lab3 - Creating Event-Driven/Reactive Services

####1. 


Lab 2 Reliable Async with AMQ - replace rest calls for order to inventory, shopping cart and payment service and introduce compensating transactions (Saga pattern)
Developer creates Queues and changes the Order service to use XXX to send an order event.
User orders and pays with a valid CC (4444333322221111)
Verifies that the payments succeeds, inventory has been updated and an order exists in the order history of the user
User orders and pays with an invalid CC (1111222233334444)
User verifies that the Payment failed, inventory has been rollback and no order exists 

Lab 3: Replace imperative catalog service with a reactive catalog service
