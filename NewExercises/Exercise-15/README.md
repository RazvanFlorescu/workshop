# Exercise 15: Generic outbox

In the previous exercise we implemented the Outbox pattern inline in the handler of the `AddItem` message. While it did solve our problem, it is not ideal from the code reuse perspective. Who would like to have the same code copied over and over again? In this exercise we are going to extract that piece of code and make it more generic. In order to achieve this we will use the *behavior* extension system of NServiceBus; the same one that we previously used to simulate various failure conditions.

Before we start coding, let's open the completed exercise 14 and the solution for the exercise 15 side-by-side. Go to the `BookPaymentHandler` and `AddItemHandler`. Take a look at the code. It looks almost exactly the same, right? That's not coincidence. Both handlers execute different business logic but have the same structure. That is a sign of an opportunity for code sharing. Taking advantage of that opportunity is the point of this exercise.

Now let's get our hands dirty.

- Open the `OutboxBehavior` class. You should be by now familiar with behaviors. They are a way of expessing cross-cutting concerns in NServiceBus. This behavior does not do anything right now but in the course of this exercise we are going to fill it with code.
- In the `OutboxBehavior`, before passing the invocation further in line 27, load the order from the repository and add it to the context.
  - Load the order based on the `OrderId` property of the `orderMessage`
  - Put it in the handling context: `context.Extensions.Set(order);`
- Persist the changes done to the order via `orderRepository.Store` after the call to `next()` (this line invokes the message handler)
- Replace usages of repository in the `AddItemHandler` with usages of the `Order` instance managed by the `OutboxBehavior`
  - Replace the repository usage with retrieving the `Order` from the context: `var order = context.Extensions.Get<Order>();`
  - Remove the second first to `orderRepository.Store` as it is handled by the behavior
- Run the code

With this change we moved most of the code responsible for loading and storing the entity to the `OutboxBehavior` and the handler can focus on the actual business logic. But there is some stuff left so let's continue.

- Move the code responsible for pushing out generated messages to the `OutboxBehavior`
  - Remove (cut) the last section of code from the `AddItemHandler` where messages are published and cleared from the collection
  - Add (paste) that code just below the `orderRepository.Store` in the `Invoke` method of the `OutboxBehavior`

Looks better, doesn't it? Event better, it compiles and works. Let's now take care of the last bit of the handler that is related to the deduplication -- the check if a message has been processed.

Remove the `!order.ProcessedMessages.Contains(context.MessageId)` from the handler and move it to the `OutboxBehavior` to guard the calls to `next()` and `orderRepository.Store`. Voila!

## It is alive!

The solution works perfectly. You can be proud of yourselves. Let's take a moment to apprieciate that. What we have implemented is in fact the cutting-edge deduplication approach used by multiple commercial and open-source frameworks. That has been a long and tough journey but we made it! From now one we are going to be entering a much less known territory and the algorithms we are going to talk about are not yet available from production-ready tools. But that's good, right? 