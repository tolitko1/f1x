Changes in 0.2
==============
 * enum SessionState renamed to SessionStatus. SessionState now describe complex session state.
 * interface SessionID changed to abstract class with final equals and hashCode methods
 + Added MessageLog and FixCommunicator.setMessageLogFactory()
 + Added FixSessionSchedule

andymalakov/f1x
•	
•	
•	
•	
•	
•	
Home
Mr.Rabbit edited this page on Jun 16 • 7 revisions
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
What is F1X
  F1X is open source implementation of FIX Protocol in Java.
Why
Do we need another Java FIX engine when we have QuickFIX/J?
Authors of F1X have been using QuickFIX/J for many years. Unfortunately QuickFIX/J can no longer keep up with modern financial applications - it is too slow and it produces a lot of short-lived objects (garbage). Garbage Collection pauses lead to unpredictable performance.
F1X is:
•	Garbage-free. API and implementation provide alloc-free FIX messaging.
•	Fast (message encoding/decoding is 10x / 6x faster than QuickFIX/J). For example: NewOrderSingle(D) message encoding takes about 500 nanoseconds, decoding takes around 210 nanoseconds.
•	Supports FIX protocol version 4.1 ... 4.4 (5.0 in near term plans).
•	Multiple FIX sessions with different FIX dialects can be hosted in a single JVM.
Documentation
Read Quick Start and FAQ pages.
Advanced topics:
•	Session Schedule
•	Message Logging
•	Message Store
•	Writing a FIX Server
•	Dealing with message gaps
•	Performance
Download
Please download project sources and build them using provided Ant / Maven / Grail scripts.
Third party dependency
F1X uses the following software libraries:
•	Disruptor disruptor buffer is used for asynchronous I/O.
•	GFLogger Garbage Free Logger.
•	QuickFIX/J QuickFIX dictionaries are used to auto-generate standard FIX enumerations and constants.
•	JUnit
FAQ
Mr.Rabbit edited this page on Jun 16 • 1 revision
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
FAQ
Where are FIX Dictionaries?
We do not provide FIX Dictionaries or any way to automatically validate FIX messages. Based on our experience, most FIX brokers do not have strictly defined FIX dialect (some undocumented tags may appear without notice; tags described as required may be sometimes omitted). If you think otherwise, custom validation can be implemented on top of our low-level API.
How to specify text value in non-ASCII encoding?
One way of doing this is to use MessageBuilder.addRaw() method:
mb.add(FixTags.MessageEncoding, MessageEncoding.SHIFT_JIS);
mb.add(FixTags.Text, "Hello");
String hello = new String ("\u3053\u3093\u306b\u3061\u306f");
byte [] helloEncoded = hello.getBytes("Shift_JIS");

mb.add(FixTags.EncodedTextLen, helloEncoded.length);
mb.addRaw(FixTags.EncodedText, helloEncoded, 0, helloEncoded.length);
How to define a repeating group?
You need to add repeating group tags in the order they will appear in the FIX message, starting with tag that specifies number of entries:
mb.add(FixTags.NoMDEntries, 2);

mb.add(FixTags.MDEntryType, MDEntryType.BID);
mb.add(FixTags.MDEntryPx, 12.32);
mb.add(FixTags.MDEntrySize, 100);
mb.add(FixTags.QuoteEntryID, "BID123");

mb.add(FixTags.MDEntryType, MDEntryType.OFFER);
mb.add(FixTags.MDEntryPx, 12.32);
mb.add(FixTags.MDEntrySize, 100);
mb.add(FixTags.QuoteEntryID, "OFFER123");
FixServer
Mr.Rabbit edited this page on Jun 16 • 1 revision
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
FIX Server
This section explains how to program a FIX server(FIX session acceptor).
FIX Acceptor types
Currently FIX server can be two types:
1.	Single Server Acceptor
2.	Multi Server Acceptor
Single Server Acceptor
Supports one acceptor per connection(server socket). First we create session id. FIX session acceptor needs to know bind host and bind port on which it will listen for inbound connections:
SessionIDBean sessionID = new SessionIDBean("Receiver", "Sender");
ServerSocketSessionAcceptor acceptor = 
        new SingleSessionAcceptor("localhost", 9999, FixVersion.FIX44, sessionID, new FixAcceptorSettings());
Multi Server Acceptor
Supports several acceptors per connection(server socket). First we create session manager. Session manager needs to know how much inbound connections simultaneously to handle and which session ids to use:
SessionManager manager = new SimpleSessionManager(10);
SessionIDBean sessionID = new SessionIDBean("Receiver", "Sender");
manager.add(sessionID, new SampleSessionState());
You can add session id on fly as well. Further we create simple factory for creation of session acceptors(communicators):
ObjectFactory<FixSessionAcceptor> acceptorFactory = new ObjectFactory<FixSessionAcceptor>(){
  @Override
  public FixSessionAcceptor create() {
    return new FixSessionAcceptor(FixVersion.FIX44, new FixAcceptorSettings());
  }
};
FIX session acceptor needs to know bind host and bind port on which it will listen for inbound connections as well as logon timeout and logout timeout:
ServerSocketSessionAcceptor acceptor = new MultiSessionAcceptor("localhost", 9999, 1000, 1000, acceptorFactory, manager);
FIX Session instances that we created implement Runnable:
new Thread(acceptor).start();
MessageGaps
Mr.Rabbit edited this page on Jun 16 • 2 revisions
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
Message Gaps
During initialization, or in the middle of a FIX session, message gaps may occur which are detected via the tracking of incoming sequence numbers. This page describes how F1X recover missing messages.
Out of sequence message processing
•	If the incoming message has a sequence number less than expected and the PossDupFlag is not set, it indicates a serious error. F1X terminates FIX connection and logs an error. By default F1X Initiators will attempt to re-establish FIX connectivity after short pause.
•	If the incoming sequence number is greater than expected, it indicates that messages were missed. F1X requests re-transmission of the messages via the Resend Request.
NOTE: By default F1X does not implement strictly ordered message processing. Inbound messages are processed in the order they are received.
This was intentional design decision:
•	In many applications having the latest state is more important than state transitions. For example, in market data feeds the latest snapshot is sufficient (and in fact in some cases gap fill is not even required)
•	It is always possible to organize strictly ordered processing queue outside of core engine.
How to detect out of order messages
Callback for application-level messages has possDup parameter:
protected void processInboundAppMessage(CharSequence msgType, int msgSeqNum, boolean possDup, MessageParser parser) {
   if (posDup) { ... }
}
MessageLogging
Mr.Rabbit edited this page on Jun 16 • 2 revisions
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
Included loggers
F1X provides several out-of-the-box implementations for FIX message loggers:
•	Basic file logger that writes into BufferedOutputStream and can flush buffer asynchronously. There is a variation that starts a new file each midnight. Another variation cycles between N files of fixed size.
•	Fixed-size logger backed by memory mapped file.
•	FIX message logger that stores messages into GFLogger.
Examples
Loggers are configured using their MessageLogFactory.
File logDir = ...
fix.setLogFactory(new DailyFileMessageLogFactory(logDir));
Logging parameter can be customized via log factory:
File logDir = ...
DailyFileMessageLogFactory logFactory = new DailyFileMessageLogFactory(logDir);
logFactory.setFlushPeriod(15000);
logFactory.setLogFormatter(new CustomLogFormatter());
fix.setLogFactory(logFactory);
Most of included implementations support custom formatting via LogFormatter interface. File-based loggers allow to customize output file names via FileNameGenerator interface. Stream-based loggers allow to select appropriate output settings (like buffer size) viaOutputStreamFactory interface.
Custom loggers
Custom implementations should implement interfaces MessageLog andMessageLogFactory.
MessageStore
Mr.Rabbit edited this page on Jun 16 • 2 revisions
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
MessageStore
Message Store is used to support FIX ResendRequest functionality.
By default F1X has no message store. All inbound ResendRequest will get a GapFill response.
Out of the box, F1X provides in-memory implementation of message store backed by fixed circular buffer.
MessageStore messageStore = new InMemoryMessageStore(1 << 22); // 4Mb
initiator.setMessageStore(messageStore);
Custom implementations
You can implement your own FIX Session schedule using interfaceorg.f1x.store.MessageStore.
Performance
Mr.Rabbit edited this page on Jun 16 • 1 revision
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
Performance
This test compares performance of message decoding and message encoding between QuickFIX/J and F1X.
Test Environment
Test platform has the following specifications:
•	Intel(R) Core(TM) i7-5960X (20M L3 Cache, 3.4GHz), Asus X99
•	48 GB RAM DDR4
•	Microsoft Windows 8.1 Pro 64-bit
•	JDK 1.7 64 (build 1.7.0_71-b14)
•	JVM starts with options: -Xms1G -Xms1G –server –verbose:gc
Comparison to Quick FIX
This section compares F1X with QuickFIX/J version 1.5.3.
Test used the following New Order Single (D) message as a sample:
8=FIX.4.4|9=196|35=D|34=78|49=A12345B|50=2DEFGH4|52=20140603-11:53:03.922|56=COMPARO|57=G|142=AU,SY|1=AU,SY|11=4|21=1|38=50|40=2|44=400.5|54=1|55=OC|58=NIGEL|59=0|60=20140603-11:53:03.922|107=AOZ3 C02000|167=OPT|10=116|
Each test has warm up phase (20000 operations) and main phase (1000000 operations). Warm up time is not counted.
Tests were run 10 times. The average result is shown in the table.
Library	Operation	Messages/sec	Average time per message (nanos)	F1X Advantage
QuickFIX/J	encoding	119904	5152	-
	decoding	341763	1446	-
F1X	encoding	1156069	498	10.34x
	decoding	1342282	210	6.85x
Wire-to-wire latency of "Echo Server"
This section measures wire-to-wire latency of FIX processing.
Test setup involved 2 machines:
•	Server (Intel i7-5960 described above)
•	Client (Intel Atom 330 - low-power 'netbook' class server - Supermicro 5015A-H)
In this test client was sending NewOrderSingleFIX(D) message 1000 times per second. Server was running EchoServer sample. This simple FIX Acceptor that reads each new FIX message from socket, decodes it, encodes it back, and sends back to the client over the same socket. Test measured 10 million messages.
Wire-to-wire latency was measured using special utility described here. At a nutshell this utility accurately measures time difference between inbound and outbound TCP packetscontaining FIX messages.
The following printout shows test results (numbers are in microseconds)
MIN: 8
MAX: 2416
MEDIAN: 9
99.000%: 11
99.900%: 15
99.990%: 25
99.999%: 45
99.9999%: 158
99.99999%:2416
During entire time server processed 10 million messages JVM had 0 garbage collections(process was running with "-Xmx1G -Xmx1G -server" parameters).
QuickStart
Andy Malakov edited this page on Jun 16 • 2 revisions
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
3-minute F1X introduction
This page assumes reader is familiar with basics of FIX Protocol.
FIX Client
This section explains how to program a FIX client (FIX session initiator).
Establish FIX connection
First we create FIX session. Sessions are identified by SessionID. FIX session initiator session needs to know FIX acceptor's host and port:
SessionID sessionID = new SessionIDBean("SENDER-COMP-ID", "TARGET-COMP-ID");
FixSession session = new FixSessionInitiator(host, port, FixVersion.FIX44, sessionID);
FIX Session instance that we created implements Runnable. Standard implementation establishes socket connection and exchange FIX Logon messages with FIX counter party. Once FIX-protocol level connectivity is established, FIX Session instance runs a message loop that processes inbound messages.
new Thread(session).start();
Send a FIX message
Let's send a FIX message as soon as both sides exchanged LOGON requests and FIX connection is established.
session.setEventListener(new SessionEventListener() {
  @Override
  public void onStateChanged(SessionID sessionID, SessionState oldState, SessionState newState) {
    if (newState == SessionState.ApplicationConnected)
      sendSampleMessage(session);
  }
F1X uses Builder pattern to construct F1X messages. Our approach is very similar tojava.lang.StringBuilder or java.lang.Appendable.
private static void sendSampleMessage(FixSession client) throws IOException {
  MessageBuilder mb = client.createMessageBuilder(); // sample only
  mb.setMessageType(MsgType.ORDER_SINGLE);
  mb.add(FixTags.ClOrdID, 123);
  mb.add(FixTags.HandlInst, HandlInst.AUTOMATED_EXECUTION_ORDER_PRIVATE);
  mb.add(FixTags.OrderQty, 1);
  mb.add(FixTags.OrdType, OrdType.LIMIT);
  mb.add(FixTags.Price, 1.43);
  mb.add(FixTags.Side, Side.BUY);
  mb.add(FixTags.Symbol, "EUR/USD");
  mb.add(FixTags.SecurityType, SecurityType.FOREIGN_EXCHANGE_CONTRACT);
  mb.add(FixTags.TimeInForce, TimeInForce.DAY);
  mb.add(FixTags.ExDestination, "HOTSPOT");
  mb.addUTCTimestamp(FixTags.TransactTime, System.currentTimeMillis());
  client.send(mb);
}
Process inbound FIX messages
In order to process inbound messages override processInboundAppMessage() method:
FixSession session = new FixSessionInitiator("localhost", 9999, FixVersion.FIX44, sessionID) {
  @Override
  protected void processInboundAppMessage(CharSequence msgType, MessageParser parser) throws IOException {
    if(Tools.equals(MsgType.MARKET_DATA_REQUEST, msgType))
      processMarketDataRequest(parser);
  }
};
F1X uses Iterator pattern to process inbound FIX messages.
General idea is:
1.	Call MessageParser.next() to advance parser to the next tag in the current FIX message.
2.	Lookup current tag using MessageParser.getTagNum().
3.	Extract value of the current tag according to its data type.
4.	Move to the next tag.
F1X provides a number of convenience methods to extract tag values without creating new instances. For example:
private static final ByteEnumLookup<SubscriptionRequestType> subscrTypeLookup =
              new ByteEnumLookup<>(SubscriptionRequestType.class);
private final ByteArrayReference symbol = new ByteArrayReference();

private void processMarketDataRequest(MessageParser parser) throws IOException {
  SubscriptionRequestType subscrType = null;
  int depth = -1;
  while (parser.next()) {
    switch (parser.getTagNum()) {
      case FixTags.Symbol:
        parser.getByteSequence(symbol);
        break;
      case FixTags.MarketDepth:
        depth = parser.getIntValue();
        break;
      case FixTags.SubscriptionRequestType:
        subscrType = subscrTypeLookup.get(parser.getByteValue());
      ...
Parser supports all commonly used data types in FIX protocol. Enumerated FIX types can be mapped to Java enums.
Next steps
•	Session Schedule
•	Message Logging
•	Message Store
•	Writing a FIX Server
•	Dealing with message gaps
•	Back to Home page
SessionSchedule
Mr.Rabbit edited this page on Jun 16 • 2 revisions
 Pages 9
•	Home
•	FAQ
•	FixServer
•	MessageGaps
•	MessageLogging
•	MessageStore
•	Performance
•	QuickStart
•	SessionSchedule
Clone this wiki locally
 
 Clone in Desktop
You can call FixSession.close() to initiate FIX Logout and close the socket.
Standard implementation
F1X provides session schedule similar QuickFIX.
SessionSchedule schedule = 
   new SimpleSessionSchedule(
      Calendar.SUNDAY, Calendar.FRIDAY, 
      "17:30:00", "17:00:00", true,
     TimeZone.getTimeZone("America/New_York"));
initiator.setSchedule(schedule);
An additional option allows to have daily vs. weekly session (e.g. sessions that span more than 24h, say Sunday-Friday).
Custom implementations
You can implement your own FIX Session schedule using interfaceorg.f1x.v1.schedule.SessionSchedule.

