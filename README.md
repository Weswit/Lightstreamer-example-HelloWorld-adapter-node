# Lightstreamer - "Hello World" Tutorial - Node.js Adapter #
<!-- START DESCRIPTION lightstreamer-example-helloworld-adapter-node -->

The *"Hello World" Tutorial* is a very basic example, based on [Lightstreamer](http://www.lightstreamer.com/), where we push the alternated strings "Hello" and "World", followed by the current timestamp, from the server to the browser.

This project is a [Node.js](http://nodejs.org/) port of the [Lightstreamer - "Hello World" Tutorial - Java Adapter](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-java), and contains the source code and all the resources needed to install the Node.js Remote Adapters for the *"Hello World" Tutorial*.

As an example of [Clients Using This Adapter](#clients-using-this-adapter), you may refer to the [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript) and view the corresponding [Live Demo](http://demos.lightstreamer.com/HelloWorld/).

## Detail

First, please take a look at the previous installment [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript), which provides some background and the general description of the application. Notice that the front-end will be exactly the same. We created a very simple HTML page that subscribes to the "greetings" item, using the "HELLOWORLD" Adapter. Now, we will replace the "HELLOWORLD" Adapter implementation based on Java with a JavaScript equivalent. On the client side, nothing will change, as server-side Adapters can be transparently switched and changed, as long as they respect the same interfaces. Thanks to this decoupling, provided by Lightstreamer Server, we could even do something different. For example, we could keep the Java Adapter on the server side and use node.js, instead of HTML, on the client side. Or, we could use the Node.js Adapter on the server side and use Java, instead of HMTL or node.js, on the client side. Basically, all the combinations of languages and technologies on the client side and on the server side are supported.

This project shows the use of DataProvider and MetadataProvider classes provided in the [Lightstreamer SDK for Node Adapters](https://github.com/Lightstreamer/Lightstreamer-lib-node-adapter).

### Node.js Interfaces

Lightstreamer Server exposes native Java Adapter interfaces. The Node.js interfaces are added through the **Lightstreamer Adapter Remoting Infrastructure** (ARI). Let's have a look at it.

![General architecture](ls-ari.png)

ARI is simply made up of two Proxy Adapters and a **Network Protocol**. The two Proxy Adapters implement the Java interfaces and are meant to be plugged into Lightstreamer Kernel, exactly as we did for our original "HELLOWORLD" Java Adapter. There are two Proxy Adapters because one implements the Data Adapter interface and the other implements the Metadata Adapter interface. Our "Hello World" example uses a default Metadata Adapter, so we only need the **Proxy Data Adapter**.

What does the Proxy Data Adapter do? Basically, it exposes the Data Adapter interface through TCP sockets. In other words, it offers a Network Protocol, which any remote counterpart can implement to behave as a Lightstreamer Data Adapter. This means you can write a remote Data Adapter in C, in PHP, or in COBOL (?!?), provided that you have access to plain TCP sockets.

But - here is some magic - if your remote Data Adapter is based on Node.js, you can forget about direct socket programming, and leverage a ready-made library that exposes a higher level **JavaScript interface**. So, you will simply have to implement this JavaScript interface.

Okay, let's recap... The Proxy Data Adapter converts from a Java interface to TCP sockets. The Node.js library converts from TCP sockets to a JavaScript interface. Clear enough?

You may find more details about ARI in [Adapter Remoting Infrastructure Network Protocol Specification](https://lightstreamer.com/api/ls-generic-adapter/latest/ARI%20Protocol.pdf).
You may find more details about how to write Data Adapters and Metadata Adapters for Lightstreamer Server in a Node.js environment in [Lightstreamer SDK for Node Adapters](https://github.com/Lightstreamer/Lightstreamer-lib-node-adapter).
The full Node.js Adapter API Reference covered in this tutorial are available at [Lightstreamer Node.js Adapter SDK API](https://www.lightstreamer.com/api/ls-nodejs-adapter/latest/).

<!-- END DESCRIPTION lightstreamer-example-helloworld-adapter-node -->

### Dig the Code

#### The JavaScript Data Adapter

The `helloworld.cjs` file containing all the required JavaScript code.

First, we include the modules we need and setup some configuration variables.

```js
var DataProvider = require('lightstreamer-adapter').DataProvider;
var net = require('net');

var HOST = 'localhost';
var PORT = 6663;
```

Then, we create a stream that will be used by our DataProvider to communicate with the Proxy Data Adapter. We use the standard net module to do so:
```js
var stream = net.createConnection(PORT, HOST);
```

Finally, we create our DataProvider instance.
```js
var dataProvider = new DataProvider(stream);
```

At this point, the application is kind of ready-ish. It doesn't do anything but "it works".

Now we want to handle the DataProvider events to listen for subscribe/unsubscribe calls and start generating real-time events.

```js
var greetingsThread;

dataProvider.on('subscribe', function(itemName, response) {
  if (itemName === "greetings") {
    greetingsThread = setTimeout(generateGreetings,0);
    response.success();    
  }
});

dataProvider.on('unsubscribe', function(itemName, response) {
  console.log("Unsubscribed item: " + itemName);
  if (itemName === "greetings") {
    clearTimeout(greetingsThread);
    response.success();
  } 
});

var c = false;
function generateGreetings() {
  c = !c;
  dataProvider.update("greetings", false, {
    'timestamp': new Date().toLocaleString(),
    'message': c ? "Hello" : "World"
  });
  greetingsThread = setTimeout(generateGreetings,1000+Math.round(Math.random()*2000));
}
```

When the "greetings" item is subscribed to by the first user, the Adapter receives the **subscribe** event and starts generating the real-time data. 
If more users subscribe to the "greetings" item, the subscribe event is not fired anymore. 
When the last user unsubscribes from this item, the Adapter is notified through the **unsubscribe** event. In this case, we stop the generating events for that item. 
If a new user re-subscribes to "greetings", the subscribe event is fired again. 
As already mentioned in the previous installment, this approach avoids consuming processing power for items no one is currently interested in.

The **greetingsThread** function is responsible for generating the real-time events. 
Its code is very simple. We simply call the dataProvider.update method passing in the item name we want to update and a JSON object representing our update containing a message (alternating "Hello" and "World") and the current timestamp. 
The function then calls itself using setTimeout to wait for a random time between 1 and 3 seconds and generate the next event.

##### ES Module

The `lightstreamer-adapter` package is compatible with ES modules. For an example, refer to the `helloworld.mjs` file.

##### Typescript

The `helloworld.ts` file demonstrates how to use the `lightstreamer-adapter` package in a Typescript module. 

To compile the Typescript code to Javascript, follow these steps:

1. Install the dependencies listed in `package.json`
  
  ```sh
  $ npm install
  ```

2. Invoke the Typescript compiler

  ```sh
  $ npx tsc helloworld.ts
  ```

This command will result in a new file named `helloworld.js` that can be run using Node.js. 

#### The Adapter Set Configuration

This Adapter Set is configured and will be referenced by the clients as `NODE_HELLOWORLD`.
For this demo, we configure just the Data Adapter as a *Proxy Data Adapter*, while instead, as Metadata Adapter, we use the [LiteralBasedProvider](https://github.com/Lightstreamer/Lightstreamer-lib-adapter-java-inprocess#literalbasedprovider-metadata-adapter), a simple full implementation of a Metadata Adapter, already provided by Lightstreamer server.
As *Proxy Data Adapter*, you may configure also the robust versions. The *Robust Proxy Data Adapter* has some recovery capabilities and avoid to terminate the Lightstreamer Server process, so it can handle the case in which a Remote Data Adapter is missing or fails, by suspending the data flow and trying to connect to a new Remote Data Adapter instance. Full details on the recovery behavior of the Robust Data Adapter are available as inline comments within the [provided template](https://lightstreamer.com/docs/ls-server/latest/remote_adapter_robust_conf_template/adapters.xml).

The `adapters.xml` file for this demo should look like:
```xml
<?xml version="1.0"?>
 
<adapters_conf id="NODE_HELLOWORLD">
 
  <metadata_provider>
    <adapter_class>com.lightstreamer.adapters.metadata.LiteralBasedProvider</adapter_class>
  </metadata_provider>
 
  <data_provider>
    <adapter_class>PROXY_FOR_REMOTE_ADAPTER</adapter_class>
    <param name="request_reply_port">6663</param>
  </data_provider>
 
</adapters_conf>
```

<i>NOTE: not all configuration options of a Proxy Adapter are exposed by the file suggested above.<br>
You can easily expand your configurations using the generic template
for [basic](https://lightstreamer.com/docs/ls-server/latest/remote_adapter_conf_template/adapters.xml) and [robust](https://lightstreamer.com/docs/ls-server/latest/remote_adapter_robust_conf_template/adapters.xml) Proxy Adapters as a reference.</i>

## Install
If you want to install a version of this demo in your local Lightstreamer Server, follow these steps:
* Download *Lightstreamer Server* (Lightstreamer Server comes with a free non-expiring demo license for 20 connected users) from [Lightstreamer Download page](https://lightstreamer.com/download/), and install it, as explained in the `GETTING_STARTED.TXT` file in the installation home directory.
* Get the `deploy.zip` file of the ["latest" release](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-node/releases) and unzip it, obtaining the `deployment` folder.
* Plug the Proxy Data Adapter into the Server: go to the `deployment/Deployment_LS` folder and copy the `NodeHelloWorld` directory and all of its files to the `adapters` folder of your Lightstreamer Server installation.
* Alternatively, you may plug the *robust* versions of the Proxy Data Adapter: go to the `deployment/Deployment_LS(robust)` folder and copy the `NodeHelloWorld` directory and all of its files into the `adapters` folder.
* Install the lightstreamer-adapter module. 
    * Create a directory where to deploy the Node.js Remote Adapter and let call it `Deployment_Node_Remote_Adapter`.
    * Go to the `Deployment_Node_Remote_Adapter` folder and launch the command:<BR/>
    `> npm install lightstreamer-adapter`<BR/>
    * Download the `helloworld.cjs` file from this project and copy it into the `Deployment_Node_Remote_Adapter` folder.
* Launch Lightstreamer Server. The Server startup will complete only after a successful connection between the Proxy Data Adapter and the Remote Data Adapter.
* Launch the Node.js Remote Adapter: go to the `Deployment_Node_Remote_Adapter` folder and launch:<BR/>
`> node helloworld.cjs`<BR/>
* Test the Adapter, launching the ["Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript)  listed in [Clients Using This Adapter](#clients-using-this-adapter).
    * To make the ["Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript) front-end pages get data from the newly installed Adapter Set, you need to modify the front-end pages and set the required Adapter Set name to NODE_HELLOWORLD, when creating the LightstreamerClient instance. So edit the `index.htm` page of the Hello World front-end, deployed under `Lightstreamer/pages/HelloWorld`, and replace:<BR/>
`var client = new LightstreamerClient(null," HELLOWORLD");`<BR/>
with:<BR/>
`var client = new LightstreamerClient(null,"NODE_HELLOWORLD");;`<BR/>
    * Open a browser window and go to: [http://localhost:8080/HelloWorld/]()

## See Also

*    [Lightstreamer SDK for Node Adapters](https://github.com/Lightstreamer/Lightstreamer-lib-node-adapter "Lightstreamer SDK for Node Adapters")

### Clients Using This Adapter
<!-- START RELATED_ENTRIES -->

* [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-client-javascript)

<!-- END RELATED_ENTRIES -->

### Related Projects

* [Complete list of "Hello World" Adapter implementations with other technologies](https://github.com/Lightstreamer?utf8=%E2%9C%93&q=Lightstreamer-example-HelloWorld-adapter&type=&language=)
* [Lightstreamer - Reusable Metadata Adapters - Java Adapter](https://github.com/Lightstreamer/Lightstreamer-lib-adapter-java-inprocess#literalbasedprovider-metadata-adapter)

## Lightstreamer Compatibility Notes

* Compatible with Lightstreamer SDK for Node.js Adapters version 1.7 or newer and Lightstreamer Server version 7.4 or newer.
- For a version of this example compatible with Lightstreamer Server version since 6.0, please refer to [this tag](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-node/tree/for_Lightstreamer_7.3).
- For a version of this example compatible with Lightstreamer SDK for Node.js Adapters version 1.3 to 1.6, please refer to [this tag](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-node/tree/for_Lightstreamer_7.3).
- For a version of this example compatible with Lightstreamer SDK for Node.js Adapters version 1.0, please refer to [this tag](https://github.com/Lightstreamer/Lightstreamer-example-HelloWorld-adapter-node/tree/for_Lightstreamer_5.1).

## Final Notes

Please [post to our support forums](http://forums.lightstreamer.com) any feedback or question you might have. Thanks!
