# Getting Started #
## What you need to download ##
You need to download:
  * **[Eneter.ProtoBuf.Serializer 1.0.0](http://eneter.net/Downloads/Extensions/EneterProtoBufSerializer/EneterProtoBufSerializer100.zip)** - protocol buffer serializer for Eneter, and in addition protocol buffer libraries and utility applications for 'proto' files.
  * **[Eneter.Messaging.Framework](http://www.eneter.net/ProductDownload.htm)** - communication framework that can be  downloaded for free for non-commercial use.

## What you need to reference from your project ##
**.NET project:**
  * _protobuf-net.dll_ - protocol buffer serializer for .NET, Windows Phone, Silverlight and Compact Framework developed by Marc Gravell. https://code.google.com/p/protobuf-net/
  * _Eneter.ProtoBuf.Serializer.dll_ - implements serializer for Eneter Messaging Framework using protobuf-net.dll
  * _Eneter.Messaging.Framework.dll_ - lightweight cross-platform framework for interprocess communication. http://www.eneter.net/

**Java project:**
  * _protobuf.jar_ - protocol buffer serializer for Java implemented by Google. https://code.google.com/p/protobuf/
  * _eneter-protobuf.jar_ - implements serializer for Eneter Messaging Framework using protobuf.jar
  * _eneter-messaging.jar_ - lightweight cross-platform framework for interprocess communication. http://www.eneter.net/

Further, you may need a utility application to convert 'proto' files to the source code:
  * _protogen.exe_ - to convert 'proto' file to C#.
  * _protoc_ - to convert 'proto' files to Java.

## Quick example ##
The following example demonstrates simple request-response communication between Android and .NET application.
The Android application is a simple client using the .NET application as a service to calculate length of the text message.

The whole example can be downloaded from:
http://eneter.net/Downloads/Extensions/EneterProtoBufSerializer/Examples/AndroidNetCommunicationProtoBuf.zip

#### 1. Adding Related References to the Android Project ####
Add following references into your Android project:
  * eneter-messaging-android-5.1.0.jar
  * eneter-protobuf-serializer-android-1.0.0.jar
  * protobuf-android-2.5.0.jar

Please notice, if you use Eclipse it is not enough to add the library simply via 'Properties' -> 'Java Build Path' -> 'Libraries'. If you do it you can build your Android application but when you upload it to the Android device it will not work and will be forced to close.

Here is the recommended procedure how to add libraries into the Android project (for Eclipse):
  * Create a new folder 'libs' in your project.
  * Right click on 'libs' and choose 'Import...' -> 'General/File System' -> 'Next'.
  * Then click 'Browser' button for 'From directory' and navigate to directory with eneter-messaging-android' library.
  * Select check box for libraries you want to add.
  * Press 'Finish'.

#### 2. Adding Related References to the .NET project ####
Add following references into your .NET project:
  * Eneter.Messaging.Framework.dll
  * Eneter.ProtoBuf.Serializer.dll
  * protobuf-net.dll

#### 3. Declaring Messages using Proto File ####
Details about syntax of 'proto language' can be found at:
https://developers.google.com/protocol-buffers/docs/proto.

In our example the proto file contains:
```
package message.declarations;

// Request Message
message MyRequest
{
    required string Text = 1;
}

// Response Message
message MyResponse
{
    required int32 Length = 1;
}
```

3.1 Compile the proto file to C#
```
protogen.exe -i:MessageDeclarations.proto -o:MessageDeclarations.cs
```

3.2 Compile the proto file to Java
```
protoc.exe -I=./ --java_out=./ ./MessageDeclarations.proto
```

3.3 Include generated source files in Android and .NET projects.

#### 4. Using Eneter.ProtoBuf.Serializer in the Android client ####
The Android client is a very simple application allowing user to put some text message and send the request to the service to get back the length of the text.

The example bellow uses Eneter ProtoBuf Serializer to serialize/deserialize messages.
```
public class AndroidNetCommunicationClientActivity extends Activity
{
    // UI controls
    private Handler myRefresh = new Handler();
    private EditText myMessageTextEditText;
    private EditText myResponseEditText;
    private Button mySendRequestBtn;
    
    
    // Sender sending MyRequest and as a response receiving MyResponse.
    private IDuplexTypedMessageSender<MyResponse, MyRequest> mySender;
    
    /** Called when the activity is first created. */
    @Override
    public void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        
        // Get UI widgets.
        myMessageTextEditText = (EditText) findViewById(R.id.messageTextEditText);
        myResponseEditText = (EditText) findViewById(R.id.messageLengthEditText);
        mySendRequestBtn = (Button) findViewById(R.id.sendRequestBtn);
        
        // Subscribe to handle the button click.
        mySendRequestBtn.setOnClickListener(myOnSendRequestClickHandler);
        
        // Open the connection in another thread.
        // Note: From Android 3.1 (Honeycomb) or higher
        //       it is not possible to open TCP connection
        //       from the main thread.
        Thread anOpenConnectionThread = new Thread(new Runnable()
            {
                @Override
                public void run()
                {
                    try
                    {
                        openConnection();
                    }
                    catch (Exception err)
                    {
                        EneterTrace.error("Open connection failed.", err);
                    }
                }
            });
        anOpenConnectionThread.start();
    }
    
    @Override
    public void onDestroy()
    {
        // Stop listening to response messages.
        mySender.detachDuplexOutputChannel();
        
        super.onDestroy();
    } 
    
    private void openConnection() throws Exception
    {
        // Instantiate Protocol Buffer based serializer.
        ISerializer aSerializer = new ProtoBufSerializer();
        
        // Create sender sending MyRequest and as a response receiving MyResponse
        // The sender will use Protocol Buffers to serialize/deserialize messages. 
        IDuplexTypedMessagesFactory aSenderFactory = new DuplexTypedMessagesFactory(aSerializer);
        mySender = aSenderFactory.createDuplexTypedMessageSender(MyResponse.class, MyRequest.class);
        
        // Subscribe to receive response messages.
        mySender.responseReceived().subscribe(myOnResponseHandler);
        
        // Create TCP messaging for the communication.
        // Note: 10.0.2.2 is a special alias to the loopback (127.0.0.1)
        //       on the development machine.
        IMessagingSystemFactory aMessaging = new TcpMessagingSystemFactory();
        
        IDuplexOutputChannel anOutputChannel
            = aMessaging.createDuplexOutputChannel("tcp://10.0.2.2:8060/");
            //= aMessaging.createDuplexOutputChannel("tcp://192.168.1.102:8060/");
        
        // Attach the output channel to the sender and be able to send
        // messages and receive responses.
        mySender.attachDuplexOutputChannel(anOutputChannel);
    }
    
    private void onSendRequest(View v)
    {
        // Create the request message using ProtoBuf builder pattern.
        final MyRequest aRequestMsg = MyRequest.newBuilder()
                .setText(myMessageTextEditText.getText().toString())
                .build();
        
        // Send the request message.
        try
        {
            mySender.sendRequestMessage(aRequestMsg);
        }
        catch (Exception err)
        {
            EneterTrace.error("Sending the message failed.", err);
        }
        
    }
    
    private void onResponseReceived(Object sender,
                                    final TypedResponseReceivedEventArgs<MyResponse> e)
    {
        // Display the result - returned number of characters.
        // Note: Marshal displaying to the correct UI thread.
        myRefresh.post(new Runnable()
            {
                @Override
                public void run()
                {
                    myResponseEditText.setText(Integer.toString(e.getResponseMessage().getLength()));
                }
            });
    }
    
    private EventHandler<TypedResponseReceivedEventArgs<MyResponse>> myOnResponseHandler
            = new EventHandler<TypedResponseReceivedEventArgs<MyResponse>>()
    {
        @Override
        public void onEvent(Object sender,
                            TypedResponseReceivedEventArgs<MyResponse> e)
        {
            onResponseReceived(sender, e);
        }
    };
    
    private OnClickListener myOnSendRequestClickHandler = new OnClickListener()
    {
        @Override
        public void onClick(View v)
        {
            onSendRequest(v);
        }
    };
}
```


#### 5. Using Eneter.ProtoBuf.Serializer in the .NET service ####
The .NET service is a simple console application listening to TCP and receiving requests to calculate the length of a given text.

The example bellow uses Eneter ProtoBuf Serializer to serialize/deserialize messages.
```
class Program
    {
        private static IDuplexTypedMessageReceiver<MyResponse, MyRequest> myReceiver;

        static void Main(string[] args)
        {
            // Instantiate Protocol Buffer based serializer.
            ISerializer aSerializer = new ProtoBufSerializer();

            // Create message receiver receiving 'MyRequest' and receiving 'MyResponse'.
            // The receiver will use Protocol Buffers to serialize/deserialize messages. 
            IDuplexTypedMessagesFactory aReceiverFactory = new DuplexTypedMessagesFactory(aSerializer);
            myReceiver = aReceiverFactory.CreateDuplexTypedMessageReceiver<MyResponse, MyRequest>();

            // Subscribe to handle messages.
            myReceiver.MessageReceived += OnMessageReceived;

            // Create TCP messaging.
            IMessagingSystemFactory aMessaging = new TcpMessagingSystemFactory();
            
            IDuplexInputChannel anInputChannel
                = aMessaging.CreateDuplexInputChannel("tcp://127.0.0.1:8060/");

            // Attach the input channel and start to listen to messages.
            myReceiver.AttachDuplexInputChannel(anInputChannel);

            Console.WriteLine("The service is running. To stop press enter.");
            Console.ReadLine();

            // Detach the input channel and stop listening.
            // It releases the thread listening to messages.
            myReceiver.DetachDuplexInputChannel();
        }

        // It is called when a message is received.
        private static void OnMessageReceived(object sender, TypedRequestReceivedEventArgs<MyRequest> e)
        {
            Console.WriteLine("Received: " + e.RequestMessage.Text);

            // Create the response message.
            MyResponse aResponse = new MyResponse();
            aResponse.Length = e.RequestMessage.Text.Length;

            // Send the response message back to the client.
            myReceiver.SendResponseMessage(e.ResponseReceiverId, aResponse);
        }
    }
```