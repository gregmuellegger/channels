Getting Started with Channels
=============================

(If you haven't yet, make sure you :doc:`install Channels <installation>`)

Now, let's get to writing some consumers. If you've not read it already,
you should read :doc:`concepts`, as it covers the basic description of what
channels and groups are, and lays out some of the important implementation
patterns and caveats.

First Consumers
---------------

When you run Django out of the box, it will be set up in the default layout -
where all HTTP requests (on the ``http.request`` channel) are routed to the
Django view layer - nothing will be different to how things worked in the past
with a WSGI-based Django, and your views and static file serving (from
``runserver`` will work as normal)

As a very basic introduction, let's write a consumer that overrides the built-in
handling and handles every HTTP request directly. This isn't something you'd
usually do in a project, but it's a good illustration of how Channels
underlies even core Django - it's less of an addition and more adding a whole
new layer under the existing view layer.

Make a new project, a new app, and put this in a ``consumers.py`` file in the app::

    from django.http import HttpResponse
    from channels.handler import AsgiHandler

    def http_consumer(message):
        # Make standard HTTP response - access ASGI path attribute directly
        response = HttpResponse("Hello world! You asked for %s" % message.content['path'])
        # Encode that response into message format (ASGI)
        for chunk in AsgiHandler.encode_response(response):
            message.reply_channel.send(chunk)

The most important thing to note here is that, because things we send in
messages must be JSON serializable, the request and response messages
are in a key-value format. You can read more about that format in the
:doc:`ASGI specification <asgi>`, but you don't need to worry about it too much;
just know that there's an ``AsgiRequest`` class that translates from ASGI into
Django request objects, and the ``AsgiHandler`` class handles translation of
``HttpResponse`` into ASGI messages, which you see used above. Usually,
Django's built-in code will do all this for you when you're using normal views.

Now we need to do one more thing, and that's tell Django that this consumer
should be tied to the ``http.request`` channel rather than the default Django
view system. This is done in the settings file - in particular, we need to
define our ``default`` channel layer and what its routing is set to.

Channel routing is a bit like URL routing, and so it's structured similarly -
you point the setting at a dict mapping channels to consumer callables.
Here's what that looks like::

    # In settings.py
    CHANNEL_LAYERS = {
        "default": {
            "BACKEND": "asgiref.inmemory.ChannelLayer",
            "ROUTING": "myproject.routing.channel_routing",
        },
    }

    # In routing.py
    from channels.routing import route
    channel_routing = [
        route("http.request", "myapp.consumers.http_consumer"),
    ]

.. warning::
   This example, and most of the examples here, use the "in memory" channel
   layer. This is the easiest to get started with but provides absolutely no
   cross-process channel transportation, and so can only be used with
   ``runserver``. You'll want to choose another backend (discussed later)
   to run things in production.

As you can see, this is a little like Django's ``DATABASES`` setting; there are
named channel layers, with a default one called ``default``. Each layer
needs a channel layer class, some options (if the channel layer needs them),
and a routing scheme, which points to a list containing the routing settings.
It's recommended you call this ``routing.py`` and put it alongside ``urls.py``
in your project, but you can put it wherever you like, as long as the path is
correct.

If you start up ``python manage.py runserver`` and go to
``http://localhost:8000``, you'll see that, rather than a default Django page,
you get the Hello World response, so things are working. If you don't see
a response, check you :doc:`installed Channels correctly <installation>`.

Now, that's not very exciting - raw HTTP responses are something Django has
been able to do for a long time. Let's try some WebSockets, and make a basic
chat server!

We'll start with a simple server that just echoes every message it gets sent
back to the same client - no cross-client communication. It's not terribly
useful, but it's a good way to start out writing Channels consumers.

Delete that previous consumer and its routing - we'll want the normal Django view layer to
serve HTTP requests from now on, which happens if you don't specify a consumer
for ``http.request`` - and make this WebSocket consumer instead::

    # In consumers.py
    from channels import Group

    def ws_message(message):
        # ASGI WebSocket packet-received and send-packet message types
        # both have a "text" key for their textual data. 
        message.reply_channel.send({
            "text": message.content['text'],
        })

Hook it up to the ``websocket.receive`` channel like this::

    # In routing.py
    from channels.routing import route
    from myapp.consumers import ws_message

    channel_routing = [
        route("websocket.receive", ws_message),
    ]

Now, let's look at what this is doing. It's tied to the
``websocket.receive`` channel, which means that it'll get a message
whenever a WebSocket packet is sent to us by a client.

When it gets that message, it takes the ``reply_channel`` attribute from it, which
is the unique response channel for that client, and sends the same content
back to the client using its ``send()`` method.

Let's test it! Run ``runserver``, open a browser and put the following into the
JavaScript console to open a WebSocket and send some data down it (you might
need to change the socket address if you're using a development VM or similar)::

    // Note that the path doesn't matter for routing; any WebSocket
    // connection gets bumped over to WebSocket consumers
    socket = new WebSocket("ws://127.0.0.1:8000/chat/");
    socket.onmessage = function(e) {
        alert(e.data);
    }
    socket.onopen = function() {
        socket.send("hello world");
    }

You should see an alert come back immediately saying "hello world" - your
message has round-tripped through the server and come back to trigger the alert.

Groups
------

Now, let's make our echo server into an actual chat server, so people can talk
to each other. To do this, we'll use Groups, one of the :doc:`core concepts <concepts>`
of Channels, and our fundamental way of doing multi-cast messaging.

To do this, we'll hook up the ``websocket.connect`` and ``websocket.disconnect``
channels to add and remove our clients from the Group as they connect and
disconnect, like this::

    # In consumers.py
    from channels import Group

    # Connected to websocket.connect
    def ws_add(message):
        Group("chat").add(message.reply_channel)

    # Connected to websocket.disconnect
    def ws_disconnect(message):
        Group("chat").discard(message.reply_channel)

Of course, if you've read through :doc:`concepts`, you'll know that channels
added to groups expire out if their messages expire (every channel layer has
a message expiry time, usually between 30 seconds and a few minutes, and it's
often configurable) - but the ``disconnect`` handler will get called nearly all
of the time anyway.

.. note::
    Channels' design is predicated on expecting and working around failure;
    it assumes that some small percentage of messages will never get delivered,
    and so all the core functionality is designed to *expect failure* so that
    when a message doesn't get delivered, it doesn't ruin the whole system.

    We suggest you design your applications the same way - rather than relying
    on 100% guaranteed delivery, which Channels won't give you, look at each
    failure case and program something to expect and handle it - be that retry
    logic, partial content handling, or just having something not work that one
    time. HTTP requests are just as fallible, and most people's reponse to that
    is a generic error page!

.. _websocket-example:

Now, that's taken care of adding and removing WebSocket send channels for the
``chat`` group; all we need to do now is take care of message sending. Instead
of echoing the message back to the client like we did above, we'll instead send
it to the whole ``Group``, which means any client who's been added to it will
get the message. Here's all the code::

    # In consumers.py
    from channels import Group

    # Connected to websocket.connect
    def ws_add(message):
        Group("chat").add(message.reply_channel)

    # Connected to websocket.receive
    def ws_message(message):
        Group("chat").send({
            "text": "[user] %s" % message.content['text'],
        })

    # Connected to websocket.disconnect
    def ws_disconnect(message):
        Group("chat").discard(message.reply_channel)

And what our routing should look like in ``routing.py``::

    from channels.routing import route
    from myapp.consumers import ws_add, ws_message, ws_disconnect

    channel_routing = [
        route("websocket.connect", ws_add),
        route("websocket.receive", ws_message),
        route("websocket.disconnect", ws_disconnect),
    ]

With all that code, you now have a working set of a logic for a chat server.
Test time! Run ``runserver``, open a browser and use that same JavaScript
code in the developer console as before::

    // Note that the path doesn't matter right now; any WebSocket
    // connection gets bumped over to WebSocket consumers
    socket = new WebSocket("ws://127.0.0.1:8000/chat/");
    socket.onmessage = function(e) {
        alert(e.data);
    }
    socket.onopen = function() {
        socket.send("hello world");
    }

You should see an alert come back immediately saying "hello world" - but this
time, you can open another tab and do the same there, and both tabs will
receive the message and show an alert. Any incoming message is sent to the
``chat`` group by the ``ws_message`` consumer, and both your tabs will have
been put into the ``chat`` group when they connected.

Feel free to put some calls to ``print`` in your handler functions too, if you
like, so you can understand when they're called. You can also use ``pdb`` and
other similar methods you'd use to debug normal Django projects.


Running with Channels
---------------------

Because Channels takes Django into a multi-process model, you no longer run
everything in one process along with a WSGI server (of course, you're still
free to do that if you don't want to use Channels). Instead, you run one or
more *interface servers*, and one or more *worker servers*, connected by
that *channel layer* you configured earlier.

There are multiple kinds of "interface servers", and each one will service a
different type of request - one might do both WebSocket and HTTP requests, while
another might act as an SMS message gateway, for example.

These are separate from the "worker servers" where Django will run actual logic,
though, and so the *channel layer* transports the content of channels across
the network. In a production scenario, you'd usually run *worker servers*
as a separate cluster from the *interface servers*, though of course you
can run both as separate processes on one machine too.

By default, Django doesn't have a channel layer configured - it doesn't need one to run
normal WSGI requests, after all. As soon as you try to add some consumers,
though, you'll need to configure one.

In the example above we used the in-memory channel layer implementation
as our default channel layer. This just stores all the channel data in a dict
in memory, and so isn't actually cross-process; it only works inside
``runserver``, as that runs the interface and worker servers in different threads
inside the same process. When you deploy to production, you'll need to
use a channel layer like the Redis backend ``asgi_redis`` that works cross-process;
see :doc:`backends` for more.

The second thing, once we have a networked channel backend set up, is to make
sure we're running an interface server that's capable of serving WebSockets.
Luckily, installing Channels will also install ``daphne``, an interface server
that can handle both HTTP and WebSockets at the same time, and then ties this
in to run when you run ``runserver`` - you shouldn't notice any difference
from the normal Django ``runserver``, though some of the options may be a little
different.

*(Under the hood, runserver is now running Daphne in one thread and a worker
with autoreload in another - it's basically a miniature version of a deployment,
but all in one process)*

Let's try out the Redis backend - Redis runs on pretty much every machine, and
has a very small overhead, which makes it perfect for this kind of thing. Install
the ``asgi_redis`` package using ``pip``, and set up your channel layer like this::

    # In settings.py
    CHANNEL_LAYERS = {
        "default": {
            "BACKEND": "asgi_redis.RedisChannelLayer",
            "CONFIG": {
                "hosts": [("localhost", 6379)],
            },
            "ROUTING": "myproject.routing.channel_routing",
        },
    }

Fire up ``runserver``, and it'll work as before - unexciting, like good
infrastructure should be. You can also try out the cross-process nature; run
these two commands in two terminals:

* ``manage.py runserver --noworker``
* ``manage.py runworker``

As you can probably guess, this disables the worker threads in ``runserver``
and handles them in a separate process. You can pass ``-v 2`` to ``runworker``
if you want to see logging as it runs the consumers.


Persisting Data
---------------

Echoing messages is a nice simple example, but it's ignoring the real
need for a system like this - persistent state for connections.
Let's consider a basic chat site where a user requests a chat room upon initial
connection, as part of the query string (e.g. ``wss://host/websocket?room=abc``).

The ``reply_channel`` attribute you've seen before is our unique pointer to the
open WebSocket - because it varies between different clients, it's how we can
keep track of "who" a message is from. Remember, Channels is network-transparent
and can run on multiple workers, so you can't just store things locally in
global variables or similar.

Instead, the solution is to persist information keyed by the ``reply_channel`` in
some other data store - sound familiar? This is what Django's session framework
does for HTTP requests, using a cookie as the key. Wouldn't it be useful if
we could get a session using the ``reply_channel`` as a key?

Channels provides a ``channel_session`` decorator for this purpose - it
provides you with an attribute called ``message.channel_session`` that acts
just like a normal Django session.

Let's use it now to build a chat server that expects you to pass a chatroom
name in the path of your WebSocket request (we'll ignore auth for now - that's next)::

    # In consumers.py
    from channels import Group
    from channels.sessions import channel_session

    # Connected to websocket.connect
    @channel_session
    def ws_connect(message):
        # Work out room name from path (ignore slashes)
        room = message.content['path'].strip("/")
        # Save room in session and add us to the group
        message.channel_session['room'] = room
        Group("chat-%s" % room).add(message.reply_channel)

    # Connected to websocket.receive
    @channel_session
    def ws_message(message):
        Group("chat-%s" % message.channel_session['room']).send({
            "text": message['text'],
        })

    # Connected to websocket.disconnect
    @channel_session
    def ws_disconnect(message):
        Group("chat-%s" % message.channel_session['room']).discard(message.reply_channel)

Update ``routing.py`` as well::

    # in routing.py
    from channels.routing import route
    from myapp.consumers import ws_connect, ws_message, ws_disconnect

    channel_routing = [
        route("websocket.connect", ws_connect),
        route("websocket.receive", ws_message),
        route("websocket.disconnect", ws_disconnect),
    ]

If you play around with it from the console (or start building a simple
JavaScript chat client that appends received messages to a div), you'll see
that you can set a chat room with the initial request.


Authentication
--------------

Now, of course, a WebSocket solution is somewhat limited in scope without the
ability to live with the rest of your website - in particular, we want to make
sure we know what user we're talking to, in case we have things like private
chat channels (we don't want a solution where clients just ask for the right
channels, as anyone could change the code and just put in private channel names)

It can also save you having to manually make clients ask for what they want to
see; if I see you open a WebSocket to my "updates" endpoint, and I know which
user you are, I can just auto-add that channel to all the relevant groups (mentions
of that user, for example).

Handily, as WebSockets start off using the HTTP protocol, they have a lot of
familiar features, including a path, GET parameters, and cookies. We'd like to
use these to hook into the familiar Django session and authentication systems;
after all, WebSockets are no good unless we can identify who they belong to
and do things securely.

In addition, we don't want the interface servers storing data or trying to run
authentication; they're meant to be simple, lean, fast processes without much
state, and so we'll need to do our authentication inside our consumer functions.

Fortunately, because Channels has an underlying spec for WebSockets and other
messages (:doc:`ASGI <asgi>`), it ships with decorators that help you with
both authentication and getting the underlying Django session (which is what
Django authentication relies on).

Channels can use Django sessions either from cookies (if you're running your
websocket server on the same port as your main site, using something like Daphne),
or from a ``session_key`` GET parameter, which is works if you want to keep
running your HTTP requests through a WSGI server and offload WebSockets to a
second server process on another port.

You get access to a user's normal Django session using the ``http_session``
decorator - that gives you a ``message.http_session`` attribute that behaves
just like ``request.session``. You can go one further and use ``http_session_user``
which will provide a ``message.user`` attribute as well as the session attribute.

Now, one thing to note is that you only get the detailed HTTP information
during the ``connect`` message of a WebSocket connection (you can read more
about that in the :doc:`ASGI spec <asgi>`) - this means we're not
wasting bandwidth sending the same information over the wire needlessly.

This also means we'll have to grab the user in the connection handler and then
store it in the session; thankfully, Channels ships with both a ``channel_session_user``
decorator that works like the ``http_session_user`` decorator we mentioned above but
loads the user from the *channel* session rather than the *HTTP* session,
and a function called ``transfer_user`` which replicates a user from one session
to another. Even better, it combines all of these into a ``channel_session_user_from_http``
decorator.

Bringing that all together, let's make a chat server where users can only
chat to people with the same first letter of their username::

    # In consumers.py
    from channels import Channel, Group
    from channels.sessions import channel_session
    from channels.auth import http_session_user, channel_session_user, transfer_user

    # Connected to websocket.connect
    @channel_session_user_from_http
    def ws_add(message):
        # Add them to the right group
        Group("chat-%s" % message.user.username[0]).add(message.reply_channel)

    # Connected to websocket.receive
    @channel_session_user
    def ws_message(message):
        Group("chat-%s" % message.user.username[0]).send({
            "text": message['text'],
        })

    # Connected to websocket.disconnect
    @channel_session_user
    def ws_disconnect(message):
        Group("chat-%s" % message.user.username[0]).discard(message.reply_channel)

If you're just using ``runserver`` (and so Daphne), you can just connect
and your cookies should transfer your auth over. If you were running WebSockets
on a separate port, you'd have to remember to provide the
Django session ID as part of the URL, like this::

    socket = new WebSocket("ws://127.0.0.1:9000/?session_key=abcdefg");

You can get the current session key in a template with ``{{ request.session.session_key }}``.
Note that Channels can't work with signed cookie sessions - since only HTTP
responses can set cookies, it needs a backend it can write to to separately
store state.


Routing
-------

Channels' ``routing.py`` acts very much like Django's ``urls.py``, including the
ability to route things to different consumers based on ``path``, or any other
message attribute that's a string (for example, ``http.request`` messages have
a ``method`` key you could route based on).

Much like urls, you route using regular expressions; the main difference is that
because the ``path`` is not special-cased - Channels doesn't know that it's a URL -
you have to start patterns with the root ``/``, and end includes without a ``/``
so that when the patterns combine, they work correctly.

Finally, because you're matching against message contents using keyword arguments,
you can only use named groups in your regular expressions! Here's an example of
routing our chat from above::

    http_routing = [
        route("http.request", poll_consumer, path=r"^/poll/$", method=r"^POST$"),
    ]

    chat_routing = [
        route("websocket.connect", chat_connect, path=r"^/(?P<room>[a-zA-Z0-9_]+)/$),
        route("websocket.disconnect", chat_disconnect),
    ]

    routing = [
        # You can use a string import path as the first argument as well.
        include(chat_routing, path=r"^/chat"),
        include(http_routing),
    ]

Channels will resolve the routing in order, short-circuiting around the
includes if one or more of their matches fails. You don't have to start with
the ``^`` symbol - we use Python's ``re.match`` function, which starts at the
start of a line anyway - but it's considered good practice.

When an include matches part of a message value, it chops off the bit of the
value it matched before passing it down to its routes or sub-includes, so you
can put the same routing under multiple includes with different prefixes if
you like.

Because these matches come through as keyword arguments, we could modify our
consumer above to use a room based on URL rather than username::

    # Connected to websocket.connect
    @channel_session_user_from_http
    def ws_add(message, room):
        # Add them to the right group
        Group("chat-%s" % room).add(message.reply_channel)

In the next section, we'll change to sending the ``room`` as a part of the
WebSocket message - which you might do if you had a multiplexing client -
but you could use routing there as well.


Models
------

So far, we've just been taking incoming messages and rebroadcasting them to
other clients connected to the same group, but this isn't that great; really,
we want to persist messages to a datastore, and we'd probably like to be
able to inject messages into chatrooms from things other than WebSocket client
connections (perhaps a built-in bot, or server status messages).

Thankfully, we can just use Django's ORM to handle persistence of messages and
easily integrate the send into the save flow of the model, rather than the
message receive - that way, any new message saved will be broadcast to all
the appropriate clients, no matter where it's saved from.

We'll even take some performance considerations into account: We'll make our
own custom channel for new chat messages and move the model save and the chat
broadcast into that, meaning the sending process/consumer can move on
immediately and not spend time waiting for the database save and the
(slow on some backends) ``Group.send()`` call.

Let's see what that looks like, assuming we
have a ChatMessage model with ``message`` and ``room`` fields::

    # In consumers.py
    from channels import Channel
    from channels.sessions import channel_session
    from .models import ChatMessage

    # Connected to chat-messages
    def msg_consumer(message):
        # Save to model
        ChatMessage.objects.create(
            room=message.content['room'],
            message=message.content['message'],
        )
        # Broadcast to listening sockets
        Group("chat-%s" % room).send({
            "content": message.content['message'],
        })

    # Connected to websocket.connect
    @channel_session
    def ws_connect(message):
        # Work out room name from path (ignore slashes)
        room = message.content['path'].strip("/")
        # Save room in session and add us to the group
        message.channel_session['room'] = room
        Group("chat-%s" % room).add(message.reply_channel)

    # Connected to websocket.receive
    @channel_session
    def ws_message(message):
        # Stick the message onto the processing queue
        Channel("chat-messages").send({
            "room": channel_session['room'],
            "message": content,
        })

    # Connected to websocket.disconnect
    @channel_session
    def ws_disconnect(message):
        Group("chat-%s" % message.channel_session['room']).discard(message.reply_channel)

Note that we could add messages onto the ``chat-messages`` channel from anywhere;
inside a View, inside another model's ``post_save`` signal, inside a management
command run via ``cron``. If we wanted to write a bot, too, we could put its
listening logic inside the ``chat-messages`` consumer, as every message would
pass through it.


.. _enforcing-ordering:

Enforcing Ordering
------------------

There's one final concept we want to introduce you to before you go on to build
sites with Channels - consumer ordering.

Because Channels is a distributed system that can have many workers, by default
it just processes messages in the order the workers get them off the queue.
It's entirely feasible for a WebSocket interface server to send out a ``connect``
and a ``receive`` message close enough together that a second worker will pick
up and start processing the ``receive`` message before the first worker has
finished processing the ``connect`` worker.

This is particularly annoying if you're storing things in the session in the
``connect`` consumer and trying to get them in the ``receive`` consumer - because
the ``connect`` consumer hasn't exited, its session hasn't saved. You'd get the
same effect if someone tried to request a view before the login view had finished
processing, but there you're not expecting that page to run after the login,
whereas you'd naturally expect ``receive`` to run after ``connect``.

Channels has a solution - the ``enforce_ordering`` decorator. All WebSocket 
messages contain an ``order`` key, and this decorator uses that to make sure that
messages are consumed in the right order, in one of two modes:

* Slight ordering: Message 0 (``websocket.connect``) is done first, all others
  are unordered

* Strict ordering: All messages are consumed strictly in sequence

The decorator uses ``channel_session`` to keep track of what numbered messages
have been processed, and if a worker tries to run a consumer on an out-of-order
message, it raises the ``ConsumeLater`` exception, which puts the message
back on the channel it came from and tells the worker to work on another message.

There's a cost to using ``enforce_ordering``, which is why it's an optional
decorator, and the cost is much greater in *strict* mode than it is in
*slight* mode. Generally you'll want to use *slight* mode for most session-based WebSocket
and other "continuous protocol" things. Here's an example, improving our
first-letter-of-username chat from earlier::

    # In consumers.py
    from channels import Channel, Group
    from channels.sessions import channel_session, enforce_ordering
    from channels.auth import http_session_user, channel_session_user, channel_session_user_from_http

    # Connected to websocket.connect
    @enforce_ordering(slight=True)
    @channel_session_user_from_http
    def ws_add(message):
        # Add them to the right group
        Group("chat-%s" % message.user.username[0]).add(message.reply_channel)

    # Connected to websocket.receive
    @enforce_ordering(slight=True)
    @channel_session_user
    def ws_message(message):
        Group("chat-%s" % message.user.username[0]).send({
            "text": message['text'],
        })

    # Connected to websocket.disconnect
    @enforce_ordering(slight=True)
    @channel_session_user
    def ws_disconnect(message):
        Group("chat-%s" % message.user.username[0]).discard(message.reply_channel)

Slight ordering does mean that it's possible for a ``disconnect`` message to
get processed before a ``receive`` message, but that's fine in this case;
the client is disconnecting anyway, they don't care about those pending messages.

Strict ordering is the default as it's the most safe; to use it, just call
the decorator without arguments::

    @enforce_ordering
    def ws_message(message):
        ...

Generally, the performance (and safety) of your ordering is tied to your
session backend's performance. Make sure you choose session backend wisely
if you're going to rely heavily on ``enforce_ordering``.


Next Steps
----------

That covers the basics of using Channels; you've seen not only how to use basic
channels, but also seen how they integrate with WebSockets, how to use groups
to manage logical sets of channels, and how Django's session and authentication
systems easily integrate with WebSockets.

We recommend you read through the rest of the reference documentation to see
all of what Channels has to offer; in particular, you may want to look at
our :doc:`deploying` and :doc:`scaling` resources to get an idea of how to
design and run apps in production environments.
