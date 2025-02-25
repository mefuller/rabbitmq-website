# RabbitMQ tutorial - "Hello World!" SUPPRESS-RHS

# Introduction

<xi:include href="site/tutorials/tutorials-help.xml.inc"/>
<xi:include href="site/tutorials/tutorials-intro.xml.inc"/>

## "Hello World"
### (using the Objective-C Client)

In this part of the tutorial we'll write a simple iOS app. It will send a
single message, and consume that message and log it using [NSLog][nslog].

In the diagram below, "P" is our producer and "C" is our consumer. The box in
the middle is a queue - a message buffer that RabbitMQ keeps on behalf of the
consumer.

<div class="diagram">
  <img src="../img/tutorials/python-one.png" alt="(P) -> [|||] -> (C)" height="60" />
</div>

> #### The Objective-C client library
> RabbitMQ speaks multiple protocols. This tutorial uses AMQP 0-9-1, which is an open,
> general-purpose protocol for messaging. There are a number of clients
> for RabbitMQ in [many different
> languages][devtools]. We'll
> use the [Objective-C client][client] in this tutorial.

### Creating an Xcode project with the RabbitMQ client dependency

Follow the instructions below to create a new Xcode project.

1. Create a new Xcode project with **File** -> **New** -> **Project…**
1. Choose **iOS Application** -> **Single View Application**
1. Click **Next**
1. Give your project a name, e.g. "RabbitTutorial1".
1. Fill in organization details as you wish.
1. Choose **Objective-C** as the language. You won't need **Unit Tests** for the
   purpose of this tutorial.
1. Click **Next**
1. Choose a place to create the project and click **Create**

Now we must add the Objective-C client as a dependency. This is done partly
from the command-line. For detailed instructions, visit [the client's GitHub
page][client].

Once the client is added as a dependency, build the project with **Product** ->
**Build** to ensure that it is linked correctly.


### Sending

<div class="diagram">
  <img src="../img/tutorials/sending.png" alt="(P) -> [|||]" height="100" />
</div>

To keep things easy for the tutorial, we'll put our send and receive code in
the same view controller. The sending code will connect to RabbitMQ and send a
single message.

Let's edit 
[ViewController.m][controller]
and start adding code.

#### Importing the framework

First, we import the client framework as a module:

<pre class="lang-objectivec">
@import RMQClient;
</pre>

Now we call some send and receive methods from `viewDidLoad`:

<pre class="lang-objectivec">
- (void)viewDidLoad {
    [super viewDidLoad];
    [self send];
    [self receive];
</pre>

The send method begins with a connection to a RabbitMQ node:

<pre class="lang-objectivec">
- (void)send {
    RMQConnection *conn = [[RMQConnection alloc] initWithUri:@"amqp://localhost:5672"
                                                 delegate:[RMQConnectionDelegateLogger new]];
}
</pre>

The connection abstracts the socket connection, and takes care of
protocol version negotiation and authentication and so on for us.

We have to specify its name or IP address using the `initWithUri:delegate:`
convenience initializer:

<pre class="lang-objectivec">
RMQConnection *conn = [[RMQConnection alloc] initWithUri:@"amqp://localhost:5672"
                                             delegate:[RMQConnectionDelegateLogger new]];
</pre>

Next we create a channel, which is where most of the API for getting
things done resides:

<pre class="lang-objectivec">
id&lt;RMQChannel&gt; ch = [conn createChannel];
</pre>

To send, we must declare a queue for us to send to; then we can publish a message
to the queue:

<pre class="lang-objectivec">
RMQQueue *q = [ch queue:@"hello"];
[ch.defaultExchange publish:[@"Hello World!" dataUsingEncoding:NSUTF8StringEncoding] routingKey:q.name];
</pre>

Declaring a queue is idempotent - it will only be created if it doesn't
exist already.

Lastly, we close the connection:

<pre class="lang-objectivec">
[conn close];
</pre>

[Here's the whole controller (including receive)][controller].

> #### Sending doesn't work!
>
> If this is your first time using RabbitMQ and you get errors logged at this
> point then you may be left scratching your head wondering what could
> be wrong. Maybe the broker was started without enough free disk space
> (by default it needs at least 200 MB free) and is therefore refusing to
> accept messages. Check the broker logfile to confirm and reduce the
> limit if necessary. The <a
> href="https://www.rabbitmq.com/configure.html#config-items">configuration
> file documentation</a> will show you how to set <code>disk_free_limit</code>.


### Receiving

That's it for sending. Our `receive` method will spin up a consumer that will
be pushed messages from RabbitMQ, so unlike `send` which publishes a single
message, it will wait for a message, log it and then quit.

<div class="diagram">
  <img src="../img/tutorials/receiving.png" alt="[|||] -> (C)" height="100" />
</div>

Setting up is the same as `send`; we open a connection and a
channel, and declare the queue from which we're going to consume.
Note this matches up with the queue that `send` publishes to.

<pre class="lang-objectivec">
- (void)receive {
    NSLog(@"Attempting to connect to local RabbitMQ broker");
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id&lt;RMQChannel&gt; ch = [conn createChannel];

    RMQQueue *q = [ch queue:@"hello"];
</pre>

Note that we declare the queue here, as well. Because we might start
the receiver before the sender, we want to make sure the queue exists
before we try to consume messages from it.

We're about to tell the server to deliver us the messages from the
queue. Since it will push messages to us asynchronously, we provide a
callback that will be executed when RabbitMQ pushes messages to
our consumer. This is what `RMQQueue subscribe:` does.

<pre class="lang-objectivec">
NSLog(@"Waiting for messages.");
[q subscribe:^(RMQMessage * _Nonnull message) {
    NSLog(@"Received %@", [[NSString alloc] initWithData:message.body encoding:NSUTF8StringEncoding]);
}];
</pre>

[Here's the whole controller again (including send)][controller].

### Running

Now we can run the app. Hit the big play button, or `cmd-R`.

`receive` will log the message it gets from `send` via
RabbitMQ. The receiver will keep running, waiting for messages (Use the Stop
button to stop it), so you could try sending messages to the same queue using
another client.

If you want to check on the queue, try using `rabbitmqctl list_queues`.

Hello World!

Time to move on to [part 2](tutorial-two-objectivec.html) and build a simple _work queue_.

[client]:https://github.com/rabbitmq/rabbitmq-objc-client
[controller]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/objective-c/tutorial1/tutorial1/ViewController.m
[devtools]:http://rabbitmq.com/devtools.html
[nslog]:https://developer.apple.com/library/ios/technotes/tn2347/_index.html
