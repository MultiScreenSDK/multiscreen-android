# MultiScreen SDK Android API

This API is the Android component of the MultiScreen SDK.
For more information, see the [MultiScreen SDK](http://multiscreen.samsung.com) and [Samsung Developer Forum](http://www.samsungdforum.com).

## Include in your Android project
Copy `/dist/android-msf-api-2.x.x.jar` to your libs folder and link as normal.
Supports Eclipse or Android Studio projects.

**Note:**

The `/dist/android-msf-api-lite-2.x.x.jar` jar contains the MultiScreen SDK API without the necessary dependencies.

## Javadocs
API reference docs are [online](http://multiscreensdk.github.io/android-msf-api/apidocs/index.html) and included in `/dist/docs/`.

# API Usage

The API supports three main features:

- **Discover**: Discover a compatible Samsung SmartTV on your network
- **Connect**: Connect and launch applications on the SmartTV
- **Communicate**: Communicate with a SmartTV application and other mobile devices.

## Discover
You can discover a compatible Samsung SmartTV on your network using the [Search](http://multiscreensdk.github.io/multiscreen-android/apidocs/com/samsung/multiscreen/Search.html) class. The workflow is:

1. Start the Discovery Process
1. Listen for events indicating services added/removed.
1. Stop the Discovery Process

Generally, while the discovery process is running, you should display a list of discovered services to your user, and allow them to select one. You can get a list of the discovered services at any time, even after the discovery process is stopped.

Example:

```java

    // Get an instance of Search
    Search search = Service.search(getContext());
    
    // Add a listener for the service found event
    search.setOnServiceFoundListener(
        new OnServiceFoundListener() {
                
            @Override
            public void onFound(Service service) {
                Log.d(LOGTAG, "Search.onFound() service: " + service.toString());

                // Add service to a displayed list where your user can select one.
                // For display, we recommend that you show: service.getName()
            }
        }
    );
    
    // Add a listener for the service lost event
    search.setOnServiceLostListener(
        new OnServiceLostListener() {
                
            @Override
            public void onLost(Service service) {
                Log.d(LOGTAG, "Search.onLost() service: " + service.toString());

                // Remove this service from the displayed list.
            }
        }
    );
    
    // Start the discovery process
    search.start();

    // Do something while we wait for services to be discovered.
	...
    
	// Stop the discovery process after some amount of time, preferably once the user 
	// has selected a service to work with.
    search.stop();
```


## Connect
Once a device or "service" has been discovered, you can interact with the service to get additional information about the device, or start, stop, install, and retrieve information about applications. Both installed applications and cloud applications can be launched.

Example:

```java

    // Example uri
    Uri url = Uri.parse("http://dev-multiscreen-examples.s3-website-us-west-1.amazonaws.com/examples/helloworld/tv/"); 

    // Example channel id
    // Note: We recommend that you use a reverse domain style id for your 
	// channel to prevent collisions.
    String channelId = "com.samsung.multiscreen.helloworld";

    // Get an instance of Application.
    Application application = service.createApplication(url, channelId);

    // Listen for the connect event
    application.setOnConnectListener(new OnConnectListener() {

        @Override
        public void onConnect(Client client) {
            Log.d(LOGTAG, "Application.onConnect() client: " + client.toString());                
        }
    });

	// Connect and launch the application.
	// When you connect to a service, the specified application will 
	// be launched automatically.
    application.connect(new Result<Client>() {

        @Override
        public void onSuccess(Client client) {
            Log.d(LOGTAG, "Application connect onSuccess() client: " + client.toString());
            
            // The application is launched, and is ready to accept 
			// messages.
        }

        @Override
        public void onError(Error error) {
            Log.d(LOGTAG, "Application connect onError() error: " + error.toString());
            
            // Uh oh. Handle the error.
        }
    );
```

## Communicate
After connecting and launching the application on the TV, otherwise known as the host, the client can now publish both text and binary messages to any and/or all connected clients.

### Publishing Messages
To send or publish a message, the client must supply:

1. **Message Event**: Application defined event describing the message
2. **Message Data**: A string or object representing the message data

Example:

```java

	// Note: The TV application is designated as the "HOST"
	String event = "say";
	String messageData = "Hello World!";
	
	// Send a message to the TV application only, by default.
	application.publish(event, messageData);

	// Send a message to the TV application, explicitly.
	application.publish(event, messageData, Message.TARGET_HOST);

	// Send a "broadcast" message to all connected clients EXCEPT yourself. 
	application.publish(event, messageData, Message.TARGET_BROADCAST);

	// Send a message to all clients INCLUDING yourself 
	application.publish(event, messageData, Message.TARGET_ALL);

	// Send a message to a specific client
	String clientId = "123467"; // Assuming that this is a valid id of a connected client
    Client client = channel.getClients().get(clientId);
    application.publish(event, messageData, client);

	// Send a message to a list of clients
	List<Client> clients = ... // You can create any combination list of clients
	application.publish(event, messageData, clients);
	
	// Send a binary message to the TV application only, by default.
	byte[] payload = {0x00, 0x01, 0x02, 0x03};
	application.publish(event, messageData, payload);
```

### Receiving Messages
To receive a message on the connected channel, the client must simply add a message listener.

Example:
		
```java

	// The event that this client is interested in receiving published messages.
	String event = "say";

    // Listen for a message by event
    application.addOnMessageListener(event, new OnMessageListener() {
        
        @Override
        public void onMessage(Message message) {
            Log.d(LOGTAG, "Application.onMessage() message: " + message.toString());
			
			// Got a message with some data or a binary payload.
        }
    });
```

## Advanced Options
Advanced options are not required to be called by default, but may allow the developer flexibility in the design and implementation of the application.

### disconnect
By default, the specified application will stop automatically when this is the last client to disconnect. To disable this behavior, call `disconnect(false, ...)`.

Example:

```java

	// Disconnect from the application. The application will continue to run on the TV, even 
	// if this was the last connected client.
    application.disconnect(false, new Result<Client>() {

        @Override
        public void onSuccess(Client client) {
            Log.d(LOGTAG, "Application disconnect onSuccess() client: " + client.toString());
			
			// The application will continue to run on the TV.
        }

        @Override
        public void onError(Error error) {
            Log.d(LOGTAG, "Application disconnect onError() error: " + error.toString());
        }
    });
```

### install
To install an application, you must create an application with the specified application id issued by Samsung when the application is submitted. See the [App Submission](http://www.samsungdforum.com/Support/Appsubmisson) pages at the Samsung Developer Forum for more information on the app submission process.

Example:

```java

	// Example app id
    String appId = "111299000796";

    // Example channel id
    String channelId = "com.samsung.multiscreen.helloworld";

    // Get an instance of Application. Then connect to the service
    Application application = service.createApplication(appId, channelId);

    // Install the application on the TV. Note: This will only bring up the installation page on the TV. 
    // The user will still have to acknowledge by selecting "install" using the TV remote.
    application.install(new Result<Boolean>() {

        @Override
        public void onSuccess(Boolean result) {
            Log.d(LOGTAG, "Application install onSuccess() result: " + result.toString());

			// Application installed
        }

        @Override
        public void onError(Error error) {
            Log.d(LOGTAG, "Application install onError() error: " + error.toString());
			
            // Uh oh. Handle the error.
        }
    });
```