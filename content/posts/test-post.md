---
title: "Adding exception detail while moving message to DLQ and limiting parallel message processing in Azure Service Bus"
date: 2020-12-04
draft: false
tags : ["Azure Service Bus"]
categories : ["Azure"]
---

### Background
We are considering we already completed getting started part from [here](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues) and we can handle/ process messages from queue and topic. Now we would like to move one step further. If we want to process message, then there is always a chance of exception. Once exception happened message is moved to dead letter queue (DLQ) automatically. From DLQ we can explore the message and check the message detail but unfortunately there is no exception detail. Then it is difficult to identify why this message failed. Here we will try to add exception detail with the message so that we can diagnosis the cause easily.

Let’s move to concurrent message handling/ processing. We want to process message one by one instead of multiple. We are focusing one by one because this is not default and our processing unit has limited resource.  So we can limit concurrent message handling by prefetch count.

What we will cover through the post
- Adding exception detail with the message while moving to DLQ
- How we can limit multiple message handling or execute message one by one using prefetch.


### Adding exception detail with the message
From the message handler we have Message and it has an attribute called “” we can put exception there and then we can abandon it. Something like following
``` cs
public void Listen<T>(Func<T, Task> action) 
{
  var messageHandlerOptions = new MessageHandlerOptions(ExceptionReceivedHandler) 
  {
    AutoComplete = true,
    MaxConcurrentCalls = 1
  };
  _queueClient.RegisterMessageHandler((message, token) =>
  {
    try 
    {
      var typedMessage = JsonConvert.DeserializeObject<T>(Encoding.UTF8.GetString(message.Body));
      return action(typedMessage);
    }
    catch(Exception ex) {
      message.UserProperties.Add("Exception_detail", ex.ToString());
      throw;
    }
  },
  messageHandlerOptions);
}
```


 
 

If it is from azure function and we are using service bus trigger then we can inject MessageReceiver as to function and from message receiver we can invoke.

 

### Executing message one by one by setting prefetch count 1

If we set prefetch count 1 then always message receiver will receive message one by one. So, you don’t need to worry about concurrency.

We can consider following example.

 

If it is azure function then we can set  prefetch count from host.json like following.

 

Full source is available here.

