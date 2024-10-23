
As part of the OpenFlow integration of this project, we decided that the best course of action was to use the [Ryu Controller](https://github.com/faucetsdn/ryu) as a reference OpenFlow controller. There isn't much sense trying to explain why we pick Ryu, as in the end, we ended up modifying it quite a bit (if modifying can also mean "cutting it down and chopping it to pieces" that is!).

This will document the necessary background required to know about how Ryu operates and how we changed it to fit with our implementations.

# Background

Ryu is a component-based OpenFlow controller written in Python. It is perhaps one of the oldest implementations of SDN controller to find use in main stream, and as such, it is much less about fancy functionality than it is about providing a neat little wrapper around [[OpenFlow]].

Before we dive in to how we used it, some necessary background is required.

## Thread Implementation and Synchronization

Ryu is at least to some extent, concerned with performance with a lot of switches in the topology, since it is a monolithic controller, which means that it will inevitably become inefficient when the network grows big, but at least we might as well try.

The main concern here is how we handle asynchronous switch events (i.e. handshakes, ACKs, etc.), as these require handlers and listeners in Ryu that will need some degree of provision. When the number of switches increases, these events scale up and as such we need to find some way of mitigating the sheer number of events that the controller will receive.

Ryu tackles this by using a very lightweight user space thread implementation similar to using non-blocking IO operations (like `kqueue` and `epoll`). The current implementation uses [[Green Threads]], which are very lightweight and optimal for asynchronous IO operations.

To provide a more robust abstraction around the usage of these threads, Ryu builds a "***hub***" layer that spawns and manages all of these threads, and implements the specific event loops that many programmers sometimes have a hard time getting right.

All of this code is under `ryu/lib/hub.py` and it implements:
- Thread spawning/joining/killing/sleeping routines
- Queues and Semaphores 
- Generic condition variables (i.e. `Event`)
- Basic server and client codes

The basic implementation backend is Python's [`eventlet`](https://eventlet.net/).

It might be useful to discuss the server (`StreamServer`) and client (`StreamClient`) codes a bit. The server code for our intents and purposes basically just blocks on a Green Socket and then spawns a thread running a specific handler, and the client just connects to a server and periodically runs an event loop. Without too much details, the bird's eye view of the code is:
```python
"""
The method 'spawn' is a wrapper around `eventlet.spawn` which handles error logging
and exceptions as well as spawning a Green Thread.
The method 'sleep' is also just a wrapper around `eventlet.sleep`, which is just the sleep method for Green Threads.
"""

class StreamServer:
	def __init__(self, listen_info, handler=None, **kwargs):
		self.server = eventlet.listen(listen_info)
		self.handle = handle
		
	def serve_forever(self):
		while True:
			sock, addr = self.server.accept()
			spawn(self.handle, sock, addr)

class StreamClient:
	def __init__(self, addr):
		self.addr = addr
		self._is_active = True

	def connect(self):
		try:
			client = socket.create_connection(self.addr)
		except:
			return None
		return client

	def connect_loop(self, handle, interval):
		while self._is_active:
			sock = self.connect()
			if sock:
				handle(sock, self.addr)
			sleep(interval)

	def stop(self):
		self._is_active = False
```
These are the main low level building blocks of Ryu.

# Event Routing

Ryu is a component-based controller, meaning that it's main function is to serve a set of events to registered components in the controller. These events are of course, OpenFlow events, generated through Ryu's own implementation of specific OpenFlow protocol versions. We'll discuss the applications later, but for now, we discuss this layer of event routing used to serve different components in the controller. Basically all of the code that we'll discuss is in `ryu/controller`.

## Event Definition

First, we need to define an event. At this point, we might as well show you the code directly:
```python
class EventBase:
    def __init__(self):
        super(EventBase, self).__init__()

class EventRequestBase(EventBase):
    def __init__(self):
        super(EventRequestBase, self).__init__()
        self.dst = None           # Application name for event producer
        self.src = None           # Application name for event consumer
        self.sync = False
        self.reply_q = None

class EventReplyBase(EventBase):
    def __init__(self, dst):
        super(EventReplyBase, self).__init__()
        self.dst = dst
```
As you can see, we define events with a base class that all other events will then inherit directly. An event is issued from some application (i.e. `ofproto`) and is then consumed by some other application. This is reflected in `RequestBase`. There is also two additional fields:
- `sync` marks the event as synchronous, meaning that the event source will wait for the response of the event.
- `reply_q` is the specific Green Thread queue that the source will block on while waiting for a reply. Of course, this is only needed when the event is marked as synchronous. 

## Event Handling

Once an event is specified, we need to show how and who is supposed to handle it internally, so we can implement the request and response classes with them. This is where Ryu defines "***dispatchers***". 

In the end, the main source of event generation in the controller is the switches, or more specifically, the abstraction of the switches in the controller which we refer to as **Datapaths**. A Datapath can be in 4 distinct states as deduced from the specific semantics of the OpenFlow protocol:

- **Handshake State:** Sending and waiting for OpenFlow hello message. This is referred to as the `HANDSHAKE_DISPATCHER`.
- **Configuration State:** Controller and switch have negotiated an OpenFlow version and the controller has sent a "feature request" message. This is `CONFIGURATION_DISPATCHER`.
- **Main State:** Switch feature has been received and the controller is allowed to send "set-config" and "get-config" messages. This is `MAIN_DISPATCHER`.
- **Dead State:** Disconnected from the peer or in the process of disconnecting. This is `DEAD_DISPATCHER`.

To handle an event, beyond just who generated that event (i.e. the event source), we also need to know which dispatcher is supposed to be associated with this event. The pair of event source and it's dispatcher is the **Event's Caller** and is thus defined simply as:
```python
class _Caller(object):
    def __init__(self, dispatchers, ev_source):
        self.dispatchers = dispatchers
        self.ev_source = ev_source
```
The dispatcher can be a list of possible values, while the source can only be the name of a single module or application.
Ryu wants to keep things simple, we want for an event handler to just basically just be a normal function that accepts an event and does something with it. In order to do that though, the event callers must be specified for each handler. 

The reason for that, is that there may not be just one consumer for a specific set of events, there can be multiple, and as such when one instance of the event happens, we need to notify ***all*** of the applications listening for it, we need to keep a list somewhere that tells us who these listeners are.

We do this by using a specific `callers` attribute for each method designated as an event handler. Python's decorator pattern makes quick work of making this readable and easy:
```python
def set_ev_cls(ev_cls: List, dispatchers: List = None):
    def _set_ev_cls_dec(handler):
        if 'callers' not in dir(handler):
            handler.callers = {}
        for e in ev_cls:
            handler.callers[e] = _Caller(dispatchers, e.__module__)
        return handler
    return _set_ev_cls_dec

def set_ev_handler(ev_cls: List, dispatchers: List = None):
    def _set_ev_cls_dec(handler):
        if 'callers' not in dir(handler):
            handler.callers = {}
        for e in ev_cls:
            handler.callers[e] = _Caller(dispatchers, None)
        return handler
    return _set_ev_cls_dec
```
So if you were trying to make the method `ev_handler` an event handler for class `RandomEvent` during `CONFIG_DISPATCHER` and `MAIN_DISPATCHER`, you can do:
```python
@set_ev_cls(RandomEvent, [CONFIG_DISPATCHER, MAIN_DISPATCHER])
def ev_handler(*args):
	# handle event ...
```
There is some other details as well, but they are not important for us.

## Applications

We now come to the main massage router between applications, the Ryu App Manager. To further relieve the burden of tracking event handlers in an application, Ryu offers to turn entire classes into what it calls **Ryu Applications** (in the above we showed you how *methods* are turned to event handlers, this is just a natural but welcome extension of that procedure).

A Ryu Application may freely use all event class handlers, and Ryu will then automatically register all of it's designated methods (i.e. those with the decorators) as event handlers. This means that the programmer need only specify which methods are event handlers and what they are supposed to do. They do not need to be worried about OpenFlow semantics or even other applications.

This abstraction adds some complexity to Ryu's switch abstraction (i.e. Datapaths) which is exactly why we prefer to first discuss this module, and then move on to Datapaths (which is probably the main meat and potato of this writing anyway).

The code that we will now discuss is in `ryu/base/app_manager.py`.

### Ryu Applications

We first define what a Ryu App actually is, and that's done with a `RyuApp` class. Keeping in track with our previous discussion, we need some list of the things that we want to listen for (i.e. event classes), that would be the private class variable `_EVENTS`, but we also need something else.

Since we are separating event handlers from the main core of the controller, we don't have immediate access to Datapaths and other crucial data structures. The problem is almost the same as the interaction between user space and kernel space in an operating system, where the distinction provides a nice abstraction, but also requires stronger interfaces between the two to be even remotely useful.

Indeed, we are writing an operating system for Datapaths (albeit a very *VERY* simple operating system), but we don't have to be worried about generic interfaces and worrying whether or not the user space is going murder us!
So Ryu just shares the data structures directly! Giving applications a direct reference to any object or class that they need in order to process an event (which may at the very least, include the all important Datapath instance).

As ill-defined as the term tends to be, we call this the **Context** of the application and we keep it also in a private class variable as `_CONTEXTS`.

With this, the instance variables of the base application class can now be set as follows:
```python
class RyuApp:
    _CONTEXTS = {}
    _EVENTS = []
    def __init__(self, *_args, **_kwargs):
        super(RyuApp, self).__init__()
        self.name = self.__class__.__name__
        self.event_handlers = {}       # ev_cls -> handlers:list
        self.observers = {}            # ev_cls -> observer-name -> states:set
        self.threads = []
        self.main_thread = None
        self.events = hub.Queue(128)
        self._events_sem = hub.BoundedSemaphore(self.events.maxsize)
        
        # prevent accidental creation of instances of this class outside RyuApp
        class _EventThreadStop(event.EventBase):
            pass
        self._event_stop = _EventThreadStop()
        self.is_active = True
```
We'll use the `name` variable to designate this application when we route events, and the `event_handlers` will list all handler methods that we want to listen for. The `observers` variable is probably the most confusing part of Ryu.

>[!FAQ] A Terminology Clarification
>The term `observer` refers to a set of event handlers that **are not part of the controller core**. These are specifically event handlers defined by users who create applications for the controller. The word `handler` is mostly meant for OpenFlow event handlers that maintain the state of the switches inside the controller.
>
>When we are discussing applications, an observer refers to the Ryu Application as a whole, while handlers refer to the specific methods that each event will have.
>
>The distinction will become clear once we discuss [[#OpenFlow Handler]].

Of course, since we are routing multiple events, some sort of synchronization is needed to both keep things in line and keep them consistent. Ryu opts for a simple producer-consumer model, meaning that events are queued up and protected with a bounded semaphore. The size is set to be `128`, an admittedly arbitrary value as far as Ryu's own developers think.

The `is_ative` flag is self explanatory, and to signal it, we use a private inner class `_EventThreadStop` whose instances we will treat as events just like others (so when the core of the controller receives  a shutdown signal, it can wait for these higher applications to first release and then close off completely by just setting up this event).

So what does a Ryu Application actually do?
Well, each class has the `start` method as follows:
```python
def start(self):
	self.threads.append(hub.spawn(self._event_loop))
```
So, we create a Green Thread and pass `_event_loop` to it. What is the loop?
```python
def _event_loop(self):
	while self.is_active or not self.events.empty():
		ev, state = self.events.get()
		self._events_sem.release()
		if ev == self._event_stop:
			continue
		handlers = self.get_handlers(ev, state)
		for handler in handlers:
			try:
				handler(ev)
			except:
				# Log event and scream ...
```
The loop runs as long as we have not been specifically shut down or we have a non-empty queue on our hands. We then pick an item from the queue (we'll block here if the queue is empty) and tick down the semaphore.

We then find the "handlers" for the event (bare with me, we'll look at that soon), and then we just call each one of those handlers on the event. So we are basically multiplexing the event between our registered handlers.
Hopefully nothing explodes and we just move on to the next handler.

Take note that <u>the handler's return value is ignored</u>. Indeed, the core shouldn't be concerned about what we do with the event (we can generate other events to communicate though) and the application itself should have it's own internal routing to actually check what the handler did and what it returned.

So what's with that `get_handlers` method?
```python
def get_handlers(self, ev, state=None):
	ev_cls = ev.__class__
	handlers = self.event_handlers.get(ev_cls, [])
	if state is None:
		return handlers

	def test(h):
		""" Wildcard Starts """
		if not hasattr(h, 'callers') or ev_cls not in h.callers:
			# dynamically registered handlers do not have
			# h.callers element for the event.
			return True
		""" Wildcard Ends """
			
		states = h.callers[ev_cls].dispatchers
		if not states:
			# empty states means all states
			return True
		return state in states
		
	 return filter(test, handlers)
```
Here, `state` is yet another name for Dispatchers, a better name you might say, but there are those who argue consistency is more important than semantic beauty! But I digress. 

What we want to do, is to return the handlers of a specific event, given a specific state. Here, we gingerly mention that an empty state (i.e. just `{}`) in Ryu, is perhaps understandably considered to mean ***ALL*** state, not none of them.
The reason is, why bother having an event that has no state? So instead use it for something meaningful, like making it mean all states so people don't have to write ugly lists to convey that!

The process is quite easy then, get the appropriate handlers for the event in the application, if no `state` is given, return all of them; if there is `state` to consider, check the handler's dispatchers by using the `callers` attribute (see [[#Event Handling]]) and if the state is found in the list of dispatchers, return that handler.

>[!NOTE]- Wildcard Code
>As you might have noticed, there is a little block of code that we have encased in a "Wildcard" comment, which curiously makes the filter accept methods that don't have the `callers` attribute!
>The reason for this, is that some internal event handlers in Ryu are not registered with a Decorator, but rather hard coded to be that way, and we can't just ignore them!
>This code makes sure that we multiplex the event for those methods as well.
>
>What are those methods? Not particularly important, they are boring.

Who sets up these handlers though?
Well, each application will have a `register_handlers` method that adds it to it's `event_handlers` dictionary:
```python
def register_handler(self, ev_cls, handler):
	self.event_handlers.setdefault(ev_cls, [])
	self.event_handlers[ev_cls].append(handler)
```
Who class this? Well, probably a higher level *Application Manager*, we'll get to that soon.

### Signaling

How do apps communicate?

Well, we keep track of them as a whole (we can do that, we are dealing with a monolithic controller after all). Ryu keeps all of it's application instances accessible via their name (class name to be precise, i.e. `self.__class__.__name__`) in a neat dictionary that maps application names to instances.

Perhaps one would call it `APPLICATIONS`, or `SERVING_APPLICATIONS` if you were pretentious, you wouldn't call it anything else would you? God forbid you would call it a weird name like `SERVICE_BRICKS` right? RIGHT?!!
Anyone who did that deserves a brick to the face! What is it with these names???

Yes, it is indeed called `SERVICE_BRICKS` and all `RyuApp` subclass instances are also called "Bricks", that's something that we just have to accept about the world, in the same vein one accepts Entropic Decay and Global Warming, but I digress.

So if you wanted to tell another application what to do, you can send an event to it, by putting it in it's `events` queue. 
```python
# This method is actually called '_send_event', but I'm calling it `_add_event`,
# because that's what it's actually doing!!
def _add_event(self, ev, state):
	self._events_sem.acquire()
	self.events.put((ev, state))

def send_event(self, name, ev, state=None):
	if name in SERVICE_BRICKS:
		if isinstance(ev, EventRequestBase):
			ev.src = self.name
		SERVICE_BRICKS[name]._add_event(ev, state)
	else:
		# complain, or scream, or whatever ...
```
This serves us well for asynchronous requests, where we just send an event and go back to what we were doing, but what if we need to wait for a response (i.e. synchronous events)?

We've seen the request/reply abstraction of events. As you might remember, an `EventRequest` must instantiate `EventRequestBase` that has a queue as an instance variable. So to make a request and wait for it's response is actually very easy:
```python
def send_request(self, req):
	assert isinstance(req, EventRequestBase)
	
	req.sync = True
	req.reply_q = hub.Queue()
	self.send_event(req.dst, req)
	
	return req.reply_q.get()

def reply_to_request(self, req, rep):
	assert isinstance(req, EventRequestBase)
	assert isinstance(rep, EventReplyBase)
	
	rep.dst = req.src
	if req.sync:
		req.reply_q.put(rep)
	else:
		self.send_event(rep.dst, rep)
```
Now we come to a bit of a semantic mine field. Ryu treats the words `observer` and `handler` a bit clumsily. The distinction becomes apparent later in this writing, but for now, observers are just a wrapper around event handlers, so just the names of callable objects. 

We can finally tell you what that `observers` attribute is. It is in simple terms, a nested dictionary. It maps the name of an event class to a dictionary that maps the name of it's application to their set of dispatchers. 

Given an event and the state of it's associated Datapath, we can query the observers of the event. Most applications don't usually specify their dispatchers directly, they usually do it with handler methods that are decorated. So by default, we let the set of dispatchers be empty, which means that the application is active for all dispatchers.

This is useful, as it means that if you do want to observe say, Echo Request messages which are usually just meant for the controller core, you can do that!

To get an observer for an event and a given state:
```python
def get_observers(self, ev, state):
	observers = []
	for k, v in self.observers.get(ev.__class__, {}).items():
		if not state or not v or state in v:
			observers.append(k)

	return observers
```
And to send events to observers:
```python
def send_event_to_observers(self, ev, state=None):
	for observer in self.get_observers(ev, state):
		self.send_event(observer, ev, state)
```

### Application Manager

The application manager is a singleton instance of the `AppManager` class that sets up all necessary applications as requested by the user. We won't be using it, but it's important to know what it's actually doing to proceed.

Since `AppManager` is a singleton class, then there is always only one instance of it, and we can always get it anywhere, anytime, with a static method. Here it is the aptly named `get_instance` method.

The App Manager needs to keep the running applications, contexts, and their classes.
```python
class AppManager:
	def __init__(self):
		self.applications_cls = {}
		self.applications = {}
		self.contexts_cls = {}
		self.contexts = {}
		self.close_sem = hub.Semaphore()
```
Here:
- `applications_cls` is the list of Ryu Application classes that we want to run
- `applications` are the instances of each class in `applications_cls`
- `contexts_cls` is the class of each object specified in the applications `_CONTEXTS` attribute (which is set by the application provider)
- `contexts` is the instances of each context class
- `close_sem` is just used for the manager to wait for each application to exit when we want to shutdown

The main job of the manager is to first *load applications*. The manager is given a list of applications to run in the beginning, and then it will import the module for each application by just searching the local dictionaries and then inspects them to find all classes that extend the `RyuApp` class.
```python
def load_app(self, name):
        mod = utils.import_module(name)
        clses = inspect.getmembers(mod, # Filter anything that extends RyuApp .. )
        if clses:
            return clses[0][1]
        return None
```
So to do this with a list of applications (we simplified this a bit!):
```python
def load_apps(self, app_lists):
	while len(app_lists) > 0:
		app_cls_name = app_lists.pop(0)
		cls = self.load_app(app_cls_name)
		self.applications_cls[app_cls_name] = cls

		for key, context_cls in cls._CONTEXTS.items():
			self.contexts_cls.setdefault(key, context_cls)
```
Once we have the application classes, we need to instantiate them, their contexts and then register them. So how do we do that?

Well, step by step, for each class in `applications_cls`:
1. Make an instance of that class, call it `app`
2. Add it to `SERVICE_BRICKS` so that anyone can get a hold of you. You should map the name of the application (i.e. `app.name`) to `app`.
3. Create an instance of the applications context classes and add them to `contexts` by again mapping their name to their instance.
4. We should now add the event handlers of this application instance. To do this, we iterate on every method inside the instance `app` in filter out anyone that has the `callers` attribute (this effectively filters the methods that are decorated).
   We now call that `register_handler` method that we mentioned before on `app` for this instance.
4. Add `app` to the `applications` dictionary by mapping `app.name` to `app`.
5. We now have the name of handlers, we need to specify the observers of the events so that the events go to the right place.
   We iterate on all of our application instances (i.e. anything in `SERVICE_BRICKS`) and we inspect each of their handlers one by one. Each handler will have a list of `_CALLER` objects which specify the dispatcher and event class that they process. We then add them to the application's list of observers and we are done!

Finally, to start the application proper, for each instance `app`, call it's `start` method and set it as the main thread of that application. We are done here!

To recap, we now know:
1. How events are defined
2. How applications are started
3. How applications declare their event handlers
4. How applications listen for events
5. How events are routed

All that is left, is to actually *generate the events*. 
We finally reach the main focus of this writing, which is Datapaths.

# Datapaths

If there is any really really component inside the controller, it's the modules that contain and handle Datapaths. The module in question is the generically named `ryu/controller/controller.py`.

Since Datapaths do have some specific interactions with the NIB in our implementation, it is necessary to see what Ryu expects from them. The Datapath mostly serves as a wrapper around polymorphic methods that serve OpenFlow messages from different OpenFlow parsers (there is multiple OpenFlow versions, which one we'll need for the switch is negotiated during handshake, Datapaths need to abstract that away).
We are not that much interested in those methods, but mostly the instance variables of the Datapaths which need to be persisted in the NIB. 

A Datapath must describe an OpenFlow switch that is currently connected to the controller. So by definition, it needs a live socket instance, as well as a queue for sending/receiving messages. 
As per Ryu's implementation, this would be the best place to put the state of the switch (i.e. it's Dispatcher) as well as the all-important unique Datapath identifier.

Now, Datapath Identifier is actually part of the OpenFlow semantics, and it is indeed negotiated with the switch during handshake. Most switches usually opt to use the Mac Address as the Datapath identifier and that is absolutely fine. The identifier as such, effectively acts as the switch's name, and as such *MUST* be shared with the NIB.

The Datapath may also hold the ports on the switch (though, OpenFlow will not give all we need to us directly, LLDP is needed for that as well), so we can create topologies, as well as send/receive routing instructions consistent with the state of the switch.

A Datapath will serve OpenFlow events, which are a distinct set of events that we have not yet discussed directly, so before we move on, we should discuss what they are.

### Datapath Attributes and Methods

Datapaths are defined as follows:
```python
"""
The base class `ProtocolDesc` is just a wrapper around OpenFlow protocol version. It adds pretty much nothing.
"""
class Datapath(ofproto_protocol.ProtocolDesc):
	def __init__(self, socket, address):
        super(Datapath, self).__init__()

        self.socket = socket
        self.socket.setsockopt(IPPROTO_TCP, TCP_NODELAY, 1)
        self.socket.settimeout(CONF.socket_timeout)
        self.address = address
        self.is_active = True
		
		self.send_q = hub.Queue(16)
        self._send_q_sem = hub.BoundedSemaphore(self.send_q.maxsize)
        
        self.xid = random.randint(0, self.ofproto.MAX_XID)
        self.id = None 
        self._ports = None
        self.state = None 
```
Again, note that this describes active switches, meaning that we are effectively waiting for switches to connect to us at the very least, and as such, with a TCP connection to some source, we know the address of the switches when we receive a hello from them.

Most of this is self-explanatory, except perhaps:
- `xid` is a transaction identifier used for the semantics of OpenFlow. Where we start it after a reconnect really isn't important, but it's best to choose the start randomly. The only consistency guarantee needed on this is that request/reply messages (even multipart messages) *MUST* have the same `xid`.
- `_ports` is a dictionary mapping port numbers to port instances, we'll see it in the controller, later.
- `state` is the dispatcher phase. We start with `HANDSHAKE_DISPATCHER`. You might notice that we have not set the state yet, we'll do that once we discuss the OpenFlow event messages next section.

>[!FAQ] When Do We Create Datapaths?
>There is both an active and a passive approach to creating these Datapath objects. 
>
>In an active approach, the controller is given the address of a switch, and it will then immediately create the Datapath and send a hello message to the address.
>
>In the passive approach, the controller is waiting for a hello message on a socket, and once it arrives, the Datapath is created and the controller immediately sends a hello message to negotiate an OpenFlow version with the switch.
>
>We speak almost always of the passive approach to creating Datapaths, so please keep that in mind.

So what do you do with a Datapath object? Well in order:

1. Spawn a thread that let's you send messages to the actual switch.
2. Immediately send an OpenFlow Hello message to the switch.
3. Spawn a thread that will periodically send echo requests to the switch.
4. Spawn a thread (or just yield the current thread) to receiving messages from the switch.
5. If something goes wrong or the switch ends the session, kill the send and echo threads and join with them. Once you return, set the Datapath `is_active` flag to False and exit.

So, in code, that'll be:
```python
def serve(self):
	send_thr = hub.spawn(self._send_loop)
	# This parse is actually set during negotiation, as we have multiple OpenFlow
	# versions. You may assume that we only work with one version and as such, this
	# parser is just hardocded into the Datapath.
	
	hello = self.ofproto_parser.OFPHello(self)
	self.send_msg(hello)

	echo_thr = hub.spawn(self._echo_request_loop)

	try:
		self._recv_loop()
	finally:
		hub.kill(send_thr)
		hub.kill(echo_thr)
		hub.joinall([send_thr, echo_thr])
		self.is_active = False
```
So, we need only show what the loops actually do.
Let's see how messages are sent first, these messages are actually serialized OpenFlow packets, generated using the parser, as you can see with the hello message above. A generic `send` method can be created like:
```python
def send(self, buf, close_socket=False):
	msg_enqueued = False
	self._send_q_sem.acquire()
	if self.send_q:
		self.send_q.put((buf, close_socket))
		msg_enqueued = True
	else:
		self._send_q_sem.release()
	if not msg_enqueued:
		# Datapath is dead! Just terminate ...
	return msg_enqueued
```
Here, `buf` is a raw byte array, so we aren't doing any fancy serialization here, that's the parser's job. The pattern here is essentially the builder pattern in Java, you create an base object (an abstract message for an OpenFlow packet), and then you are actually allowed to change some of it (like say, the `xid`, which depends on the context and when the message is being sent).
Once you made that modification, you build that actual object that you want (the raw bytes for that packet).

The builder method for all of these messages is just a generically named `serialize` method. So once the message is defined, you call `serialize` and pick up your actual packet in the `buf` attribute. With that, you'll have the `send_msg` method like this:
```python
def send_msg(self, msg, close_socket=False):
	if msg.xid is None:
		self.set_xid(msg)
	msg.serialize()
	return self.send(msg.buf, close_socket=close_socket)
```
The `close_socket` parameter will mark socket as no-write after this packet is sent. It's useful when the controller wants to end the session.

Now that we know how to send messages, that echo request loop is quite easy!
```python
def _echo_request_loop(self):
	if not self.max_unreplied_echo_requests:
		return
	while (self.send_q and
		   (len(self.unreplied_echo_requests) <= self.max_unreplied_echo_requests)):
		echo_req = self.ofproto_parser.OFPEchoRequest(self)
		self.unreplied_echo_requests.append(self.set_xid(echo_req))
		self.send_msg(echo_req)
		hub.sleep(self.echo_request_interval)
	self.close()
```
The Datapath attribute `max_unreplied_echo_requests` and `echo_request_interval` are just generic configurations set by the App Manager.  The Datapath is automatically disconnected when it fails to answer to a certain amount of queued up requests as given in the first attribute, and interval will define how long we wait for each request before sending another one. The default values are 0 (meaning we won't tolerate even a single one) and 15 seconds.

We keep all un-replied echo request `xid` (that's the only thing we need) in a list for this matter (i.e. `unreplied_echo_requests`). Actually catching this message requires the OpenFlow Handler which we are yet to discuss, but eventually it just pops the `xid` of the reply from the list.
```python
def acknowledge_echo_reply(self, xid):
	try:
		self.unreplied_echo_requests.remove(xid)
	except ValueError:
		pass
```
All of the above is then put on a send queue, which is handled by the eponymous `send_loop` thread:
```python
def _send_loop(self):
	try:
		while self.state != DEAD_DISPATCHER:
			buf, close_socket = self.send_q.get()
			self._send_q_sem.release()
			self.socket.sendall(buf)
			if close_socket:
				break
	except:
		# scream
	finally:
		q = self.send_q
		# First, clear self.send_q to prevent new references.
		self.send_q = None
		# Now, drain the send_q, releasing the associated semaphore for each entry.
		# This should release all threads waiting to acquire the semaphore.
		try:
			while q.get(block=False):
				self._send_q_sem.release()
		except hub.QueueEmpty:
			pass
		# Finally, disallow further sends. (just set the socket to read-only)
		self._close_write()
```
Now, all that remains is the receiver loop. This is actually the most important part, since it is the main source of generating events in the whole controller! 

If you ever coded a low level socket receiver, then this should seem familiar to you. We block on receiving for each of our sockets, and we then receive some minimum length of bytes (which since we are using OpenFlow, is should be at least the OpenFlow header size).

We then immediately ask the OpenFlow parser to tell us what the header says about the message. It is either nonsense, in which case we kill the connection, or it computes and produces the following:

- The OpenFlow version being used
- The type of the message
- The length of the message
- The transaction identifier (i.e. `xid`)

We now receive on the socket until we have at least the length of the packet worth of data. We then pass it to the parser, and *generate an event for it*. 
```python
# This decorator closes the connection if we get some unhandled exception
@_deactivate
def _recv_loop(self):
	buf = bytearray()
	min_read_len = remaining_read_len = ofproto_common.OFP_HEADER_SIZE

	while self.state != DEAD_DISPATCHER:
		read_len = min_read_len
		if remaining_read_len > min_read_len:
			read_len = remaining_read_len
		ret = self.socket.recv(read_len)
		
		if not ret:
			break

		buf += ret
		buf_len = len(buf)
		while buf_len >= min_read_len:
			(version, msg_type, msg_len, xid) = ofproto_parser.header(buf)
			if buf_len < msg_len:
				remaining_read_len = (msg_len - buf_len)
				break
				
			# If we are here, then we have the full message ...
			msg = ofproto_parser.msg(
				self, version, msg_type, msg_len, xid, buf[:msg_len])
			if msg:
				"""****************************************
				| Generate the event and call the handler |
				|   	(we'll come back to this)         |
				****************************************"""

			buf = buf[msg_len:]
			buf_len = len(buf)
			remaining_read_len = min_read_len
```
>[!WARNING]- About Green Thread Scheduling
>This isn't important, but it might be helpful to point out the cost of the simplistic approach that Ryu is taking for handling these things.
>
>You see, at the end of the day, a Green Thread, is just a thread, and as such some scheduler is supposed to switch between them. As such, we are not immune to starvation, meaning that some really active threads can effectively block other threads from moving forward.
>
>Green Threads handle this with *Cooperative Yielding*, in which they politely put themselves at the end of the run queue, and let others proceed. But rather vexingly, for threads that are under a lot of work (for example collecting many many messages from a single switch), the scheduler will *NOT* intervene, and it is the job of the programmer to yield the thread when things are getting out of hand.
>
>Now the act of yielding is easy, it is just a single call `hub.sleep(0)`, the question is *when* is the time to do it? When can we decide that a thread is hugging too much resource?
>
>Ryu is using a very simple method, all receiver threads yield after they processed 2048 messages (no matter how large or small they are). We have removed that code from the snippet above, but it's important to know that it is there. We won't comment too much on this, just know that it is definitely not a very ideal solution.

The rest of the code for Datapaths is just some utilities for sending OpenFlow messages like barriers or flow modifications. We still however have a few things to discuss:

- Who actually creates the Datapath?
- How do we manage ports?
- How do we manage the Datapath state?
- How do we generate the events in the receiver loop?

For the 3 final questions, we need to move on to the OpenFlow handler, so let's discuss the first one.

### OpenFlow Controller

The OpenFlow controller, despite it's grand name, is really simple! In fact, laughably so!
```python
class OpenFlowController:
    def __init__(self):
        self.ofp_tcp_listen_port = CONF.ofp_tcp_listen_port
        
    def server_loop(self, ofp_tcp_listen_port):
	    # The host is most likely just the default 127.0.0.1
        server = StreamServer((CONF.ofp_listen_host,
							   ofp_tcp_listen_port),
							  datapath_connection_factory)
							  
        server.serve_forever()
```
The TCP listen port is defined by IANA to be 6653 or 6633 (6633 is the older one, so 6653 is the true standard now). Recall that `StreamServer` was just a server blocking on a single socket. The handler now as you can see is `datapath_connection_factory`, which is just:
```python
def datapath_connection_factory(socket, address):
    with contextlib.closing(Datapath(socket, address)) as datapath:
        try:
            datapath.serve()
        except:
	        # Blah Blah ...
```
FYI the `close` method called when we exit the `with` clause above basically just marks the Datapath socket as read-only and changes it's state to `DEAD_DISPATCHER`. So the controller creates the Datapaths, *who creates the controller though*?

That'll be the OpenFlow handler application which we'll discuss the following sections.

### OpenFlow Events

This code is in `ryu/controller/ofp_event.py`. The module is named `ofp_event` and as such, as a Ryu Application …, sorry, *Service Brick* (spoken with contempt), it can be tracked by that name exactly. 
These events however, are not handled just by Ryu applications, we definitely can't delegate the maintenance of Datapaths to Ryu Applications of course, we have to do it right inside the controller core. So Ryu also defines a specific OpenFlow Handler that the controller immediately instantiates as another Service Brick.

Let's discuss the events first, then we move on to the handler.
The base event class is the following:
```python
class EventOFPMsgBase(event.EventBase):
    def __init__(self, msg):
        self.timestamp = time.time()
        super(EventOFPMsgBase, self).__init__()
        self.msg = msg
```
As you can see, it adds a timestamp, as well a `msg` attribute to the normal event base class. Timestamp is self-explanatory, but the `msg` is rather arbitrary.
The `msg` is quite a simply an amorphous blob of information, an instance of an arbitrary class that can describe all sorts of different OpenFlow events. It's just a generic wrapper around OpenFlow information.

There is one attribute of `msg` that is always needed, and it is (you guessed it!) the Datapath instance (not just the identifier!), simply because that since switches are not applications, they don't have a well defined source like other applications in Ryu, in fact, they all have the same source! (i.e. `ofp_event` as we shall see), we need this to distinguish events based on the source switches.

We also have two generic OpenFlow events that are used internally to maintain the state of the Datapath instances:
```python
class EventOFPStateChange(event.EventBase):
    def __init__(self, dp):
        super(EventOFPStateChange, self).__init__()
        self.datapath = dp

class EventOFPPortStateChange(event.EventBase):
    def __init__(self, dp, reason, port_no):
        super(EventOFPPortStateChange, self).__init__()
        self.datapath = dp
        self.reason = reason
        self.port_no = port_no
```
The first one is generated when the state of a Datapath is changed (i.e. for example when we completed handshake and moved from `HANDSHAKE_DISPATCHER` to `CONFIG_DISPATCHER`). 

The second one is used to signal a change in OpenFlow ports on the switch. In OpenFlow semantics, ports are identified with a number and any change in their states can be expressed by a signal of 3 kinds:

- `OFPP_ADD` for port addition
- `OFPP_DELETE` for port removal
- `OFPP_MODIFY` for any change on an active port

We keep the port number in `port_no` and the signal in `reason`. There are a few routines inside the file associated with this code, but they are not important, we should move on to the handler.

What about other events? Where is the event for Hello messages, Feature Requests and whatnot?
Well, since every named class of messages requires an event, why not just read the parser and create an event for each one of them?
Indeed, this is what Ryu does.

These few lines do it all:
```python
for ofp_mods in ofproto.get_ofp_modules().values():
    ofp_parser = ofp_mods[1]
    _create_ofp_msg_ev_from_module(ofp_parser)
```
Here, `get_ofp_modules` is a dictionary that maps OpenFlow versions to their parser instances. As such, this code effectively grabs the parsers and passes them to `_create_ofp_msg_ev_from_module`.
```python
def _create_ofp_msg_ev_from_module(ofp_parser):
    # print mod
    for _k, cls in inspect.getmembers(ofp_parser, inspect.isclass):
        if not hasattr(cls, 'cls_msg_type'):
            continue
        _create_ofp_msg_ev_class(cls)
```
Here, `cls_msg_type` is an attribute that is set for each message class in the parser (so for example, an echo request for OpenFlow version 1.3 will have a `cls_msg_type` value of `ofproto_v1.3.EchoRequest`). This function iterates over all of the classes in the parser, picks any that have this attribute and creates an event class for them with `_create_ofp_msg_ev_class`.

It's useful to keep these dynamically loaded classes somewhere, so Ryu keeps them in a dictionary called `_OFP_MSG_EVENTS` in the `ofp_event` file. We basically map the event name to it's created class. If you used dynamic class loading for Python, this should look familiar.
```python
def _create_ofp_msg_ev_class(msg_cls):
    name = _ofp_msg_name_to_ev_name(msg_cls.__name__)
    if name in _OFP_MSG_EVENTS:
        return

    cls = type(name, (EventOFPMsgBase,),
               dict(__init__=lambda self, msg:
                    super(self.__class__, self).__init__(msg)))
    globals()[name] = cls
    _OFP_MSG_EVENTS[name] = cls
```
What are we doing?
- Create a name for the event class. The method `_ofp_msg_name_to_ev_name` literarily just adds the word `Event` to the beginning of the class name and returns it, so `OFPEchoRequest` messages will have the event class `EventOFPEchoRequest`.
- Create a class for it that extends `EventOFPMsgBase` and initialize it.
- Add it to the list global variables as well as the OpenFlow message dictionary that we mentioned.

Now, we finally have **all the OpenFlow events that we'll ever need**. Time to write a handler for them. We can now finally answer the question of how the events are generated. 
Remember that box that we left empty in the receiver loop? Here is what it does:
```python
if msg:
	ev = ofp_event.ofp_msg_to_ev(msg)
	self.ofp_brick.send_event_to_observers(ev, self.state)
	
	def dispatchers(x):
		return x.callers[ev.__class__].dispatchers
	
	handlers = [handler for handler in
				self.ofp_brick.get_handlers(ev) if
				self.state in dispatchers(handler)]
	for handler in handlers:
		handler(ev)
```
We have the message object, to generate it's event, we need only check it's class name, look it up in `_OFP_MSG_EVENTS` to see what the event class associated with the message is, and then make an instance of that event.
**This is actually where we also pass the message to the event!**
```python
def ofp_msg_to_ev(msg):
	name = _ofp_msg_name_to_ev_name(msg.__class__.__name__)
	return _OFP_MSG_EVENTS[name](msg)
```
We now call the handlers of the event. 
As you might remember, the terms `handler` and `observer` are treated differently. Observers are specifically event handlers in Ryu Applications outside of the controller core, whereas here, handlers are specifically OpenFlow Handlers as we will see soon.

The `ofp_brick` is just the OpenFlow Handler, which is implemented as a Ryu Application as you might have guessed.

### OpenFlow Handler

The handler is in `ryu/controller/ofp_handler`. The main purpose of the handler is to negotiate the `HANDSHAKE` and `CONFIG` dispatchers and move the Datapath into the `MAIN_DISPATCHER`, and then it will serve all generated events to the applications to do whatever they want with it. It also implements some other parts of the OpenFlow semantics, like keep-alive messages and whatnot.

It's important to note that the handler is indeed, just another Ryu Application, but it is of a higher privilege compared to other applications, since it is the main core of the controller. 
```python
class OFPHandler(ryu.base.app_manager.RyuApp):
    def __init__(self, *args, **kwargs):
        super(OFPHandler, self).__init__(*args, **kwargs)
        self.name = ofp_event.NAME     # this is just the string `ofp_event`
        self.controller = None
```
As you can see, it extends `RyuApp` class. It also contains a `controller` instance which is an instance of the `OpenFlowController`, so we now know who makes the controller in the first place!
```python
def start(self):
	super(OFPHandler, self).start()
	self.controller = OpenFlowController()
	return hub.spawn(self.controller)
```
You might notice that the controller instance is treated as callable, indeed, there is a call method for the controller, which just calls `server_loop`!
```python
def __call__(self):
	self.server_loop(self.ofp_tcp_listen_port)
```
So the nitty-gritties of setting up sockets and actually talking to switches is in that controller instance, this handler isn't concerned with that, instead, it is concerned with setting up event handlers for specific OpenFlow messages.

#### Echo Reply Message Handler

By this point, `ofp_event` has loaded an event class for Echo Reply messages called `EventOFPEchoReply`, which you can see that we passed as the first argument of the decorator.
```python
@set_ev_handler(ofp_event.EventOFPEchoReply,
                    [HANDSHAKE_DISPATCHER, CONFIG_DISPATCHER, MAIN_DISPATCHER])
def echo_reply_handler(self, ev):
	msg = ev.msg
	datapath = msg.datapath
	datapath.acknowledge_echo_reply(msg.xid)
```
As you see, it internally consumes the message and pops the request from the list of un-replied requests by calling the acknowledge function on the Datapath instance.

Now, we can start to tie up the loose ends that we left during our discussion on Datapaths. 
How do we manage Datapath state changes? 

#### State Management

During the handling of certain events, when the handler decides that it needs to change the state, it does so by calling a `set_state` method on the Datapath.

Here is an excerpt from the Hello message event handler:
```python
@set_ev_handler(ofp_event.EventOFPHello, HANDSHAKE_DISPATCHER)
def hello_handler(self, ev):
	msg = ev.msg
	datapath = msg.datapath

	"""
	Here, there is some removed code that negotiates the OpenFlow
	version. It's long and boring, so let's just pretend we have a
	list of usable verions for the switch.
	"""
	datapath.set_version(max(usable_versions))

	# Move on to config state
	self.logger.debug('move onto config mode')
	datapath.set_state(CONFIG_DISPATCHER)

	# Finally, send feature request
	features_request = datapath.ofproto_parser.OFPFeaturesRequest(datapath)
	datapath.send_msg(features_request)
```
As you see, in the end when moving to `CONFIG` state, we call the `set_state` method. It does pretty much exactly what you might expect:
```python
def set_state(self, state):
	if self.state == state:
		return
	self.state = state
	ev = ofp_event.EventOFPStateChange(self)
	ev.state = state
	if self.ofp_brick is not None:
		self.ofp_brick.send_event_to_observers(ev, state)
```
#### Port Status Handler

Similar to state changes, reporting a change in the state of a port is also easy:
```python
@set_ev_handler(ofp_event.EventOFPPortStatus, MAIN_DISPATCHER)
def port_status_handler(self, ev):
	msg = ev.msg
	datapath = msg.datapath
	ofproto = datapath.ofproto

	if msg.reason in [ofproto.OFPPR_ADD, ofproto.OFPPR_MODIFY]:
		datapath.ports[msg.desc.port_no] = msg.desc
	elif msg.reason == ofproto.OFPPR_DELETE:
		datapath.ports.pop(msg.desc.port_no, None)
	else:
		return

	self.send_event_to_observers(
		ofp_event.EventOFPPortStateChange(
			datapath, msg.reason, msg.desc.port_no),
		datapath.state)
```
>[!NOTE] 
>Since OpenFlow 1.3 removed pretty much all reports on port statistics, we have to use LLDP to actually get useful info out of this. In fact, the actual port attribute in a Datapath is pretty much useless. Ryu uses an augmentation of the Datapath as it's actual switch.

# Topologies

To actually be useful, the data about just switches alone is kind of useless. We need to know the network as a whole to program routing instructions, but since OpenFlow 1.3, that has become a bit more difficult.

Ryu has a specific topology management library for this that implements it's own sets of events on top of what other applications are doing:
- The events are defined in `ryu/topology/events.py`
- The event handlers are defined in `ryu/topology/switches.py`

>[!TODO]
>Future me should probably finish this part. We don't need it for our implementation though ...

# OpenFlow Protocol Implementation

So far, we have maintained quite a bit of distance from the underlying protocol used to communicate with the switches and instead just referred to collecting "messages" from Datapaths and then passing it to the handlers.

We should however, address *how* these messages are actually created, since it is crucial to our implementation, and as such, I am devoting this section to describing how these are done. I will not go into detail, because honestly, what can I say that the [OpenFlow Spec](http://www.cs.yale.edu/homes/yu-minlan/teach/csci599-fall12/papers/openflow-spec-v1.3.0.pdf) has not said?

All that we discuss is in `ryu/ofproto/ofproto_v1_3.py` and `ryu/ofproto/ofproto_v1_3_parser.py`. We are not concerned with the simultaneous support of multiple OpenFlow versions (which Ryu very much is concerned), and as such, we only discuss those modules, and **cut out everything that isn't about that**. So expect things to look much simpler than they actually are, but not simpler than our own implementation!

## The Protocol

All things start with the actual protocol implementation, by that we mean:
- Protocol constants and enumerations
- Message formats and packets
- Headers
- Error codes

And all else. You are encouraged to take a little look at the OpenFlow spec that we linked above, but not too much. You will find these definitions in the `ofproto_vx_y.py` files. Perhaps your main interest should be the definition of the OpenFlow *header*, for that's really the main thing that you need for parsing messages.

Taking a look at it, we see this:
```python
from struct import calcsize

# struct ofp_header
OFP_HEADER_PACK_STR = '!BBHI'
OFP_HEADER_SIZE = 8
assert calcsize(OFP_HEADER_PACK_STR) == OFP_HEADER_SIZE
```
>[!Note]- If You Don't Understand The Code Above
>If you have not used the `struct` module of python, it does exactly as its name says, it creates tightly packed byte representations of a series of primitive data types (i.e. just the good old C `struct`). 
>
>Indeed, what better representation for packets than a struct there is? You might however be weirded out by that format string `!BBHI`. See [this](https://docs.python.org/3/library/struct.html#format-characters) for details but in brief:
>
>- The `!` at the beginning signals the endian-ness of the byte stream. On a network, you should send the header first, so that the receiver gets it first and sees how long your packet is, so you put the MSB on the smallest address of your array, hence you use Big-Endian. The `!` character is just a shortcut for that.
>- The rest are just the length of the primitive data types in your struct. The OpenFlow spec says that OpenFlow header contains in order the **protocol version**, **message type**, **packet length** and **transaction ID (XID)**, all of which are unsigned integers of lengths 1, 1, 2 and 4 bytes.
>  That is what those characters are. The `B` is Unsigned Char, the `H` is Unsigned Short and `I` is the normal Unsigned 32 bit Integer. 
>
>The `calcsize` is just a sanity check for making sure the struct has the right length,

Once you have received what you think is an OpenFlow packet on a socket and put it in a buffer (i.e. a `bytearray`) like `buf`, you can get the protocol header by:
```python
def header(buf):
    assert len(buf) >= ofproto_common.OFP_HEADER_SIZE
    return struct.unpack_from(ofproto_common.OFP_HEADER_PACK_STR, buf)
```
(The above is defined under `ryu/ofproto/ofproto_parser.py`)

What you get from this would be the unpacked, primitive type representation of the struct (i.e. just the protocol version, message type, packet length and XID).

So, if you knew how to make sense of the message body, to actually parse a series of OpenFlow messages, you just:

1. Receive data on a socket until you have at lease a headers worth of bytes collected. 
2. Parse the header, determine the message type and its length as `msg_len`
3. Keep receiving on the socket until you have `msg_len` bytes (including the header)
4. Cut the initial `msg_len` bytes from the array, pass it on to the OpenFlow parser to see what the body contains, and return to step '1'.

What would the OpenFlow parser do? Well, it parses the header again to see what the message type is and then parses the packet body (i.e. anything after the  initial `OFP_HEADER_SIZE` bytes) by using the appropriate struct format.

>[!Example] Feature Reply Messages
>After the hello is sent and the  OpenFlow version is negotiated. The controller will most likely immediately send an OpenFlow Feature Request message. 
>The switch responds with its features (some codes that say what it can do and what type of links it has), but most importantly, it responds with its **Datapath Identifier**, a very important piece of information for our implementation in particular.
>
>If you take a look at the switch feature struct, you see its defined like this:
>```python
> # ofp_switch_features
> OFP_SWITCH_FEATURES_PACK_STR = '!QIBB2xII'
> OFP_SWITCH_FEATURES_SIZE = 32
> assert (calcsize(OFP_SWITCH_FEATURES_PACK_STR) + OFP_HEADER_SIZE ==  OFP_SWITCH_FEATURES_SIZE)
>```
>Here, `Q` means Unsigned Long Long, and it is the 8 byte Datapath ID that we need. The OpenFlow Feature Reply message has a type of ***6***, and so, parsing the header should yield the same value for the message type field.
>Once you get that value, you know what type of message this is and you can recover the numerical value of the DP ID,

As such, it is pretty easy to make a class out of each type of OpenFlow message and then attach a `parser` class method to each one that receives the buffer (and maybe the result of parsing the header so it wouldn't have to do that again) and then parse it and return an instance of its class. 
Indeed, that is *exactly* what Ryu does.

To go from a class representation to the struct is also quite easy as well, we just unpack the class attributes into a packet and then return is as a buffer. This is what the `serialize` method and the `buf` instance variable is for.

And that's it. That is all you need to know about how Ryu works!
