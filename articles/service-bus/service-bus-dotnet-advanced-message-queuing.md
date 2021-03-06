<properties 
	pageTitle="How to use AMQP 1.0 with the .NET Service Bus API | Microsoft Azure" 
	description="Learn how to use Advanced Message Queuing Protodol (AMQP) 1.0 with the Azure .NET Service Bus API." 
	services="service-bus" 
	documentationCenter=".net" 
	authors="sethmanheim" 
	manager="timlt" 
	editor=""/>

<tags 
	ms.service="service-bus" 
	ms.workload="na" 
	ms.tgt_pltfrm="na" 
	ms.devlang="dotnet" 
	ms.topic="article" 
	ms.date="05/10/2016" 
	ms.author="sethm"/>

# How to use AMQP 1.0 with the Service Bus .NET API

The Advanced Message Queuing Protocol (AMQP) 1.0 is an efficient, reliable, wire-level messaging protocol that you can use to build robust, cross-platform, messaging applications.

Support for AMQP 1.0 in Service Bus means that you can use the queuing and publish/subscribe brokered messaging features from a range of platforms using an efficient binary protocol. Furthermore, you can build applications comprised of components built using a mix of languages, frameworks, and operating systems.

This article explains how to use the Service Bus brokered messaging features (queues and publish/subscribe topics) from .NET applications using the Service Bus .NET API. There is a [companion article](service-bus-java-how-to-use-jms-api-amqp.md) that explains how to do the same using the standard Java Message Service (JMS) API. You can use these two guides together to learn about cross-platform messaging using AMQP 1.0.

## Get started with Service Bus

This article assumes that you already have a Service Bus namespace containing a queue named "queue1." If you do not, then you can create the namespace and queue using the [Azure classic portal](http://manage.windowsazure.com). For more information about how to create Service Bus namespaces and queues, see [How to use Service Bus queues](service-bus-dotnet-get-started-with-queues.md).

## Download the Service Bus SDK

AMQP 1.0 support is available in Service Bus SDK version 2.1 or later. You can download the latest SDK from NuGet at [http://nuget.org/packages/WindowsAzure.ServiceBus/](http://nuget.org/packages/WindowsAzure.ServiceBus/).

## Code .NET applications

By default, the Service Bus .NET client library communicates with the Service Bus service using a dedicated SOAP-based protocol. To use AMQP 1.0 instead of the default protocol requires explicit configuration on the Service Bus connection string as described in the next section. Other than this change, application code remains basically unchanged when using AMQP 1.0.

In the current release, there are a few API features that are not supported when using AMQP. These unsupported features are listed later in the section [Unsupported features and restrictions](#unsupported-features-and-restrictions). Some of the advanced configuration settings also have a different meaning when using AMQP. None of these settings are used in this article, but more details are available in the [Service Bus AMQP overview](service-bus-amqp-dotnet.md#unsupported-features-restrictions-and-behavioral-differences).

### Configure via App.config

It is a recommended practice that applications use the App.config configuration file to store settings. For Service Bus applications, you can use App.config to store the Service Bus **ConnectionString**. This sample application also uses App.config to store the name of the Service Bus messaging entity that it uses.

A sample App.config file is shown here:

```
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  	<appSettings>
	    <add key="Microsoft.ServiceBus.ConnectionString"
       	     value="Endpoint=sb://[namespace].servicebus.windows.net;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=[SAS key];TransportType=Amqp" />
	    	<add key="EntityName" value="queue1" />
	</appSettings>
</configuration>
```

### Configure the Service Bus connection string

The value of the **Microsoft.ServiceBus.ConnectionString** setting is the Service Bus connection string that is used to configure the connection to Service Bus. The format is as follows:

```
Endpoint=sb://[namespace].servicebus.windows.net;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=[SAS key];TransportType=Amqp
```

Where `[namespace]` and `[SAS key]` are obtained from the [Azure classic portal][]. For more information, see [How to use Service Bus queues][].

When using AMQP, the connection string is appended with `;TransportType=Amqp`, which tells the client library to make its connection to Service Bus using AMQP 1.0.

### Configure the entity name

This sample application uses the `EntityName` setting in the **appSettings** section of the App.config file to configure the name of the queue with which the application exchanges messages.

### A simple .NET application using a Service Bus queue

The following example sends and receives messages to and from a Service Bus queue.

```
// SimpleSenderReceiver.cs
	
using System;
using System.Configuration;
using System.Threading;
using Microsoft.ServiceBus.Messaging;
	
namespace SimpleSenderReceiver
{
    class SimpleSenderReceiver
    {
        private static string connectionString;
        private static string entityName;
        private static Boolean runReceiver = true;
        private MessagingFactory factory;
        private MessageSender sender;
        private MessageReceiver receiver;
        private MessageListener messageListener;
        private Thread listenerThread;
	
        static void Main(string[] args)
        {
            try
            {
                if ((args.Length > 0) && args[0].ToLower().Equals("sendonly"))
                    runReceiver = false;
	                
                string ConnectionStringKey = "Microsoft.ServiceBus.ConnectionString";
                string entityNameKey = "EntityName";
                entityName = ConfigurationManager.AppSettings[entityNameKey];
                connectionString = ConfigurationManager.AppSettings[ConnectionStringKey];
                SimpleSenderReceiver simpleSenderReceiver = new SimpleSenderReceiver();
	
                Console.WriteLine("Press [enter] to send a message. " +
                    "Type 'exit' + [enter] to quit.");
	
                while (true)
                {
                    string s = Console.ReadLine();
                    if (s.ToLower().Equals("exit"))
                    {
                        simpleSenderReceiver.Close();
                        System.Environment.Exit(0);
                    }
                    else
                        simpleSenderReceiver.SendMessage();
                }
            }
            catch (Exception e)
            {
                Console.WriteLine("Caught exception: " + e.Message);
            }
        }
	
        public SimpleSenderReceiver()
        {
            factory = MessagingFactory.CreateFromConnectionString(connectionString);
            sender = factory.CreateMessageSender(entityName);
	
            if (runReceiver)
            {
                receiver = factory.CreateMessageReceiver(entityName);
                messageListener = new MessageListener(receiver);
                listenerThread = new Thread(messageListener.Listen);
                listenerThread.Start();
            }
        }
	
        public void Close()
        {
            messageListener.RequestStop();
            listenerThread.Join();
            factory.Close();
        }
	
        private void SendMessage()
        {
            BrokeredMessage message = new BrokeredMessage("Test AMQP message from .NET");
            sender.Send(message);
            Console.WriteLine("Sent message with MessageID = " 
                + message.MessageId);
        }

    }
	
    public class MessageListener
    {
        private MessageReceiver messageReceiver;
        public MessageListener(MessageReceiver receiver)
        {
            messageReceiver = receiver;
        }
	
        public void Listen()
        {
            while (!_shouldStop)
            {
                try
                {
                    BrokeredMessage message = 
                        messageReceiver.Receive(new TimeSpan(0, 0, 10));
                    if (message != null)
                    {
                        Console.WriteLine("Received message with MessageID = " + 
                            message.MessageId);
                        message.Complete();
                    }
                }
                catch (Exception e)
                {
                    Console.WriteLine("Caught exception: " + e.Message);
                    break;
                }
            }
        }
	
        public void RequestStop()
        {
            _shouldStop = true;
        }
	
        private volatile bool _shouldStop;
    }
}
```

### Run the application

Running the application produces output of the form:

```
> SimpleSenderReceiver.exe
Press [enter] to send a message. Type 'exit' + [enter] to quit.
	
Sent message with MessageID = fb7f5d3733704e4ba4bd55f759d9d7cf
Received message with MessageID = fb7f5d3733704e4ba4bd55f759d9d7cf
	
Sent message with MessageID = d00e2e088f06416da7956b58310f7a06
Received message with MessageID = d00e2e088f06416da7956b58310f7a06
	
Received message with MessageID = f27f79ec124548c196fd0db8544bca49
Sent message with MessageID = f27f79ec124548c196fd0db8544bca49
exit
```

## Cross-platform messaging between JMS and .NET

This topic showed how to send messages to Service Bus using .NET and also how to receive those messages using .NET. However, one of the key benefits of AMQP 1.0 is that it enables applications to be built from components written in different languages, with messages exchanged reliably and at full fidelity.

Using the sample .NET application described above and a similar Java application taken from a companion guide, [How to use the Java Message Service (JMS) API with Service Bus & AMQP 1.0](service-bus-java-how-to-use-jms-api-amqp.md), it's possible to exchange messages between .NET and Java. 

For more information about the details of cross-platform messaging using Service Bus and AMQP 1.0, see the [Service Bus AMQP 1.0 overview](service-bus-amqp-overview.md).

### JMS to .NET

To demonstrate JMS to .NET messaging:

* Start the .NET sample application without any command-line arguments.
* Start the Java sample application with the "sendonly" command-line argument. In this mode, the application will not receive messages from the queue, it will only send.
* Press **Enter** a few times in the Java application console, which will cause messages to be sent.
* These messages are received by the .NET application.

### Output from JMS application

```
> java SimpleSenderReceiver sendonly
Press [enter] to send a message. Type 'exit' + [enter] to quit.
Sent message with JMSMessageID = ID:4364096528752411591
Sent message with JMSMessageID = ID:459252991689389983
Sent message with JMSMessageID = ID:1565011046230456854
exit
```

### Output from .NET application

```
> SimpleSenderReceiver.exe	
Press [enter] to send a message. Type 'exit' + [enter] to quit.
Received message with MessageID = 4364096528752411591
Received message with MessageID = 459252991689389983
Received message with MessageID = 1565011046230456854
exit
```

## .NET to JMS

To demonstrate .NET to JMS messaging:

* Start the .NET sample application with the "sendonly" command line argument. In this mode, the application will not receive messages from the queue, it will only send.
* Start the Java sample application without any command line arguments.
* Press **Enter** a few times in the .NET application console, which will cause messages to be sent.
* These messages are received by the Java application.

#### Output from .NET application

```
> SimpleSenderReceiver.exe sendonly
Press [enter] to send a message. Type 'exit' + [enter] to quit.
Sent message with MessageID = d64e681a310a48a1ae0ce7b017bf1cf3	
Sent message with MessageID = 98a39664995b4f74b32e2a0ecccc46bb
Sent message with MessageID = acbca67f03c346de9b7893026f97ddeb
exit
```

#### Output from JMS application

```
> java SimpleSenderReceiver	
Press [enter] to send a message. Type 'exit' + [enter] to quit.
Received message with JMSMessageID = ID:d64e681a310a48a1ae0ce7b017bf1cf3
Received message with JMSMessageID = ID:98a39664995b4f74b32e2a0ecccc46bb
Received message with JMSMessageID = ID:acbca67f03c346de9b7893026f97ddeb
exit
```

## Unsupported features and restrictions

The following features of the .NET Service Bus API are not currently supported when using AMQP:

* Transactions
* Send via transfer destination
* Receive by message sequence number
* Message and session browse
* Session state
* Batch-based APIs
* Scaled-out receive
* Runtime manipulation of subscription rules
* Session lock renewal
* Some minor differences in behavior

For more information, see the [Service Bus AMQP overview](service-bus-amqp-dotnet.md). This article includes a detailed list of unsupported APIs.

## Summary

This article showed how to access the Service Bus brokered messaging features (queues and publish/subscribe topics) from .NET using AMQP 1.0 and the Service Bus .NET API.

You can also use Service Bus AMQP 1.0 from other languages including Java, C, Python, and PHP. Components built using these languages can exchange messages reliably and at full fidelity using AMQP 1.0 in Service Bus. For more information, see the [Service Bus AMQP overview](service-bus-amqp-dotnet.md).

## Next steps

Now that you've read an overview of Service Bus and AMQP with .NET, see the following links for more information.

* [AMQP 1.0 support in Azure Service Bus](service-bus-amqp-overview.md)
* [How to use the Java Message Service (JMS) API with Service Bus & AMQP 1.0](service-bus-java-how-to-use-jms-api-amqp.md)
* [How to use Service Bus queues](service-bus-dotnet-get-started-with-queues.md)
 
[Azure classic portal]: https://manage.windowsazure.com
