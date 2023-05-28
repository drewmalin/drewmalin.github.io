---
layout: post
title: "Java Client/Server"
date: 2014-03-02 22:01:00
categories: [java]
permalink: :year-:month-:day-:title
---

Weekend projects are great not only as a change of pace, a stress reliever, or even a way of learning new things, but as a way of remembering those little skills that have begun to slip away. The targeted skills for this weekend? The general, *multithreaded networking*, and the specific, *ant scripts*.

It begins.

Getting started with a simple ant script:

```java
<project name="ClientServer" default="build" basedir=".">
    <property name="src"    location="src"/>
    <property name="build"  location="bin"/>

    <target name="build" depends="clean, init, compile"/>

    <target name="clean">
        <delete dir="${build}"/>
    </target>

    <target name="init">
        <mkdir dir="${build}"/>
    </target>

    <target name="compile">
        <javac srcdir="${src}"
               destdir="${build}" 
               includeantruntime="false" 
               debug="true"
               debuglevel="lines,vars,source">
        </javac>
    </target>

    <target name="run">
        <java fork="true"
              classpath="${build}"
              classname="com.clientserver.Server">
        </java>
    </target>
</project>
```

A couple of notes about the various build options above. The *includeantruntime* option, if true,  will include all libraries specific and available to Ant in the build's classpath. This is unnecessary for this project. Setting *fork* to true will allow the project to run in a fresh VM, rather than being invoked from within the VM in which ant resides.

Additionally, this particular ant setup expects a file structure where the ant script sits at the same level as two independent directories, bin and src. Compilation of the \*.java files within the src directory will be outputted into the \*.bin directory.

Writing the server.


First, stubbing out some methods:

```java
public class Server extends Thread {
    public Server() {
    
    }
    
    /**
     * The run() method will server as the entry point for clients. The thread will
     * await any connections, verify them, and then save the information necessary to
     * maintain this connection.
     */
    @Override
    public void run() {

    }

    /**
     * Transmit a packet of information to a target client. If the target client is
     * null, then treat the transmission as a broadcast, sending the packet to all
     * clients.
     */
    public void transmitPacket(final Packet packet, final ClientThread ctxClient) {

    }

    /**
     * Remove a client upon its disconnection from the server.
     */
    public void disconnectClient(final int clientId) {

    }

    /**
     * Generate a unique ID to identify each client.
     */
    private int nextUniqueId() {

    }

    public static void main(String[] args) {
        final Server server = new Server();
    }
}
```

The above is not quite complete without something to handle each individual connection to a client. This is where ClientThread comes into play:

```java
public class ClientThread extends Thread {
    public ClientThread(final Socket socket, final int id, final Server serverHandle) {

    }

    /**
     * The run() method will serve as the connection to a unique client. Any incoming
     * information will be processed here.
     */
    @Override
    public void run() {

    }

    public void sendPacket(final Packet p) {

    }

    /**
     * Verify connection with the client.
     */
    public boolean handshake() {

    }

    /**
     * Close connection.
     */
    public void close() {

    }

    public int getClientId() {

    }
}
```

And now for some implementation! First, the Server:

```java
    private static final int SERVER_PORT = 1337;
    private ServerSocket serverSocket;
    private final Map<Integer, ClientThread> clientThreads;

    public Server() {
        this.clientThreads = new HashMap<Integer, ClientThread>();
        try {
            this.serverSocket = new ServerSocket(Server.SERVER_PORT);
            this.start();
        } catch (final IOException e) {
            System.out.println("ERROR - Failed to create server.");
            e.printStackTrace();
            System.exit(1);
        }
    }
```

The constructor will handle the initialization of the data structure housing the client threads, the server socket through which clients will connect, and some basic error handling. A Map is used here so that some control can be had over the ID which uniquely identifies each client (allowing the server to pinpoint client activity). An array would work, but rearranging clients after they disconnect goes beyond the point of this project. The server port here was chosen fairly arbitrarily.
The error handling is extremely basic, but if there is an issue with the server socket, there isn't much point continuing the program execution.

Now, we have the server up and running! What happens when a client attempts a connection?

```java
    @Override
    public void run() {
        while (true) {
            if (this.clientThreads.size() < 16) { // Arbitrary client limit
                System.out.println("SERVER - Awaiting connections.");
                try {
                    final Socket clientSocket = this.serverSocket.accept();
                    final ClientThread tmpClient = 
                        new ClientThread(clientSocket, nextUniqueId(), this);
                    if (tmpClient.handshake()) {
                        tmpClient.start();
                        this.clientThreads.put(tmpClient.getClientId(), tmpClient);
                    }
                } catch (final IOException e) {
                    System.out.println("ERROR - Failed to connect client.");
                    e.printStackTrace();
                }
            }
        }
    }
```

The combination of the infinite loop and the line *this.serverSocket.accept()* means our server will churn away, continuously accepting and validating client connections. The thread will actually block at the *accept()* call (hence the implementation of Thread in the first place) moving forward only if a connection is attempted. Once the client socket is discovered, the handshaking routine will need to pass before the server will save the client's information. However, if all goes
horribly wrong and an error occurs, there's no reason to kill the server. In this case, we just allow the server to ignore the client who failed to connect.

We've got a server up, and now it's listening for connections. Let's quickly wrap up the server code:

```java
    // If the context client is null, broadcast the packet to all clients.
    // Make sure not to transmit a packet back to the client which originated it.
    public void transmitPacket(final Packet packet, final ClientThread ctxClient) {
        for (final ClientThread client : this.clientThreads.values()) {
            if (ctxClient == null || client != ctxClient) {
                client.sendPacket(packet); 
            }
        }
    }

    public void disconnectClient(final int id) {
        if (this.clientThreads.containsKey(id)) {
            System.out.println("SERVER - Removing client: " + id);
            this.clientThreads.get(id).close();
            this.clientThreads.remove(id);
        }
    }

    private int nextUniqueId() {
        int tmpId = Server.idGenerator.nextInt(); // static Generator object
        while (this.clientThreads.containsKey(tmpId)) {
            tmpId = Server.idGenerator.nextInt();
        }
    }
```

Our server can now happily churn away, connecting, disconnecting, and messaging clients. But how does the outside world actually interact with the server?

Via ClientThread objects. We begin again with the constructor:

```java
    private final Socket socket;
    private final Server serverHandle; // a hook back to the server
    private final int id;

    private ObjectOutputStream oos;
    private ObjectInputStream ois;
    private Packet pHandle; // a reusable reference to a bundle of data

    public ClientThread(final Socket socket, final int id, final Server serverHandle) {
        this.socket = socket;
        this.serverHandle = serverHandle;
        this.id = id;
        try {
            this.ois = new ObjectInputStream(this.socket.getInputStream());
            this.oos = new ObjectOutputStream(this.socket.getOutputStream());
        } catch (final IOException e) {
            System.out.println("ERROR - Failed to create client IO.");
            e.printStackTrace();
        }
    }
```

This constructor is reminiscent of that in the Server class, with the exception of the input/output streams which are our literal bridges across the network to the remote clients.

```java
    @Override
    public void run() {
        while (true) {
            try {
                this.pHandle = (Packet) ois.readObject();
                this.serverHandle.transmitPacket(this.pHandle, this);
            } catch (final Exception e) {
                System.out.println("ERROR - Client " + this.id + " disconnected.");
                this.serverHandle.disconnectClient(this.id);
                break;
            }
        }
    }
```

More familiar code! This while loop will allow for this thread to continuously wait for and act upon client input. the *readObject()* call will halt the thread until data is received, at which point the raw object is cast to a Packet (which will be outlined soon) and sent up to the server for transmission to the other clients. Failure here means the connection to the client has been severed in some way. If this happens, we first notify the server that this client should be
removed, then we break from the infinite loop, allowing this thread to die.

What about when this client thread needs to send information to its remote client? Never fear: 

```java
    public void sendPacket(final Packet p) {
        try {
            this.oos.writeObject(p);
            this.oos.flush();
        } catch (final IOException e) {
            System.out.println("ERROR - Failed to transmit packet.");
            e.printStackTrace();
        }
    }
```

All of this will be for naught, however, if the sever is not able to identify this client as being valid for connection. Our handshaking routine will be a simple one: a mere exchange of "Hello's," but this could be improved to be a bit more robust at a later time:

```java
    // Returns true if handshaking succeeds, false otherwise.
    public boolean handshake() {
        System.out.println("Attempted connection: " + this.socket.getInetAddress());
        System.out.println("Handshaking...");
        try {
            final Packet inPacket = (Packet) this.ois.readObject();
            if (!inPacket.getHeader().equals("HELLO")) {
                System.out.println("Attempted connection refused");
                return false;
            }
            sendPacket(new Packet("HELLO"));
        } catch (final Exception e) {
            System.out.println("ERROR: Handshaking failed.");
            e.printStackTrace();
            return false;
        }
        System.out.println("Connection accepted.");
        return true;
    }
```

We can wrap this up by supplying the *close()* and *getClientId()* methods used by the server:

```java
    public void close() {
        try {
            this.socket.close();
            this.ois.close();
            this.oos.close();
        } catch (final IOException e) {
            System.out.println("ERROR - Failed to close client " + this.id + ".");
            e.printStackTrace();
        }
    }

    public int getClientId() {
        return this.id;
    }
```

Server: written. The bulk is complete. But before moving onto the client code, first we should establish what exactly this Packet object is all about.

```java
public class Packet implements Serializable {

    // The serialVersionUID is needed by the Serializable interface-- this ensures
    // the packet sent from the remote client is the same object as the packet
    // recieved by the local server.
    private static final long serialVersionUID = 1L;
    private String header;
    private final String data;
    
    public Packet(final String header, final String data) {
        this.header = header;
        this.data = data;
    }

    // Secondary constructor allows us to quickly create packets for use when
    // non-unique data is needed (such as during handshaking)
    public Packet(final String header) {
        this.header = header;
        this.data = "";
    }

    public void setHeader(final String header) {
        this.header = header;
    }

    public String getHeader() {
        return this.header;
    }

    public String getData() {
        return this.data;
    }
}
```

Of note: the Packet class will be used by both the Client and Server objects. Due to the use of Serializable objects being sent between object streams, the *exact* same class needs to be present on both systems. For us, all this means is one copy of Packet.java will reside with the server in its own directory, and one with the client.

This marks a perfect time to segue into the need for a slightly updated directory structure! Something like this will do:

```
|- .
|- build.xml
|- client/
   |- src/
   |- bin/
|- server/
   |- src/
   |- bin/
```

This will emulate separate server and client systems. In addition to this, we might as well update the ant script (build.xml) to accommodate this change:

```xml
<project name="ClientServer" default="build-all" basedir=".">
	<property name="server_src" 	location="server/src"/>
	<property name="server_build" 	location="server/bin"/>
	<property name="client_src"		location="client/src"/>
	<property name="client_build"	location="client/bin"/>

	<target name="build-all" depends="build-client, build-server"/>
	<target name="build-client" depends="clean-client, init-client, compile-client"/>
	<target name="build-server" depends="clean-server, init-server, compile-server"/>

	<target name="clean-client">
		<delete dir="${client_build}"/>
	</target>

	<target name="clean-server">
		<delete dir="${server_build}"/>
	</target>

	<target name="init-client">
		<mkdir dir="${client_build}"/>
	</target>

	<target name="init-server">
		<mkdir dir="${server_build}"/>
	</target>

	<target name="compile-client">
		<javac srcdir="${client_src}"
               destdir="${client_build}" 
               includeantruntime="false" 
               debug="true" 
               debuglevel="lines,vars,source">
		</javac>
	</target>

	<target name="compile-server">
		<javac srcdir="${server_src}" 
               destdir="${server_build}" 
               includeantruntime="false" 
               debug="true" 
               debuglevel="lines,vars,source">
		</javac>
	</target>

    <target name="client" description="run the client!">
        <java fork="true" 
              classpath="${client_build}" 
              classname="com.clientserver.Client">
        </java>
    </target>

    <target name="server" description="run the server!">
        <java fork="true" 
              classpath="${server_build}" 
              classname="com.clientserver.Server">
        </java>
    </target>

</project>
```

Nothing too fancy here, just a splitting of the build process between client and server. Note that 'build-all' is now the default command, meaning by simply running 'ant' within the terminal, both the client and the server will be built. Running 'ant server' or 'ant client' will start up the server and client respectively (perform these commands within their own terminal windows).

And now, the client!

```java
public class Client extends Thread {
    public Client() {

    }

    /**
     * Perform socket setup, object stream initialization, and server handshaking.
     */
    public void connect(final String host, final int port) {

    }

    /**
     * Read in all input from all other clients connected to the same server.
     */
    @Override
    public void run() {

    }

    /**
     * Handle and submit terminal input from the user.
     */
    public void handleUserInput() {

    }

    /**
     * A convenience method to tell other clients at what time a message was sent.
     */
    public String getTimestamp() {

    }

    public static void main(String[] args) {

    }
}
```

Again, some familiar-looking code. The gist here is that, once the handshaking routine is complete, all that the client needs to do is behave as a conduit for i/o from the server and other clients.  Speaking of i/o...

```java
    private Socket socket;
    private ObjectOutputStream oos;
    private ObjectInputStream ois;
    privatePacket pHandle;

    // ...

    public void connect(final String host, final int port) {
        System.out.println("Attempting to connect to " + host + ":" + port);
        try {
            this.socket = new Socket(host, port);
            this.oos = new ObjectOutputStream(this.socket.getOutputStream());
            this.ois = new ObjectInputStream(this.socket.getInputStream());
            System.out.println("Connection successful: " + 
                this.socket.getInetAddress() + "\nHandshaking...");
            this.oos.writeObject(new Packet("HELLO");
            this.oos.flush();
            this.pHandle = (Packet) this.ois.readObject();
            if (this.pHandle.getHeader().equals("HELLO")) {
                System.out.println("Success!");
                this.start();
                handleUserInput();
            }
            else {
                System.out.println("ERROR - Handshaking failed.");
                System.exit(1);
            }
        } catch (final Exception e) {
            System.out.println("ERROR - Failed to connect to server.");
            e.printStackTrace();
            System.exit(1);
        }
    }
```

Handshaking complete! Lots of error catching that results in the client coming down. This is done mainly because, well, there's not much to do if the server is unable to be reached. If this model were to be inserted into a greater project, this could be replaced with a 'retry connection' method.

And now a couple infinite loops. The first will handle reading in input from the server (and therefore from the other clients), the second will handle reading in user input, transmitting it to the server so the other clients will be able to read it.

```java
    @Override
    public void run() {
        while (true) {
            try {
                this.pHandle = (Packet) this.ois.readObject();
                System.out.println(this.pHandle.getHeader() + 
                    ": " + this.pHandle.getData());
            } catch (final Exception e) {
                System.out.println("ERROR - Connection to server failed.");
                System.exit(1);
            }
        }
    }

    public void handleUserInput() {
        final BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        while (true) {
            try {
                this.oos.writeObject(new Packet(getTimestamp(), br.readLine()));
                this.oos.flush();
            } catch (final IOException e) {
                System.out.println("ERROR - Failed to retrieve console input.");
                System.exit(1);
            }
        }
    }
```

Pretty self-explanatory given the server code already written. *run()* will continuously print out information from the server, while *handleUserInput()* will continuously transmit back to the server any console input from the user. Text provied to standard in is a bit contrived, but one can imagine this being expanded (by simply enhancing the *Packet* class) for other more interesting information.

Finally, the main routine to actually run the client (and a little helper go grab the timestamp of the packet creation)!

```java
    public String getTimestamp() {
        return new java.text.SimpleDateFormat("MM/dd/yyyy h:mm:ss a").format(new Date());
    }

    public static void main(String[] args) {
        final Client client = new Client();
        client.connect("localhost", 1337);
    }
```

When all is said and done, we should be able to see the following in three separate terminals (representing three separate systems):

```
$ ant server
Buildfile: /Users/drewmalin/Desktop/ClientServerJava/build.xml

server:
     [java] SERVER - Awaiting connections. 0 clients connected.
     [java] Attempted connection: /127.0.0.1
     [java] Handshaking...
     [java] Connection accepted from: /127.0.0.1
     [java] SERVER - Awaiting connections. 1 clients connected.
     [java] Attempted connection: /127.0.0.1
     [java] Handshaking...
     [java] Connection accepted from: /127.0.0.1
     [java] SERVER - Awaiting connections. 2 clients connected.
```

```
$ ant client
Buildfile: /Users/drewmalin/Desktop/ClientServerJava/build.xml

client:
     [java] Attempting to connect to localhost:1337
     [java] Connection successful: localhost/127.0.0.1
     [java] Handshaking...
     [java] Success!
Hello, world!
```

```
ant client
Buildfile: /Users/drewmalin/Desktop/ClientServerJava/build.xml

client:
     [java] Attempting to connect to localhost:1337
     [java] Connection successful: localhost/127.0.0.1
     [java] Handshaking...
     [java] Success!
     [java] MSG (2124925754) [03/03/2014 9:30:21 PM]: Hello, world!
```

World domination is well within reach.

[Full project](https://github.com/drewmalin/simple-java-client-server)

Action item: write shorter posts.