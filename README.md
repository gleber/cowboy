Cowboy
======

Cowboy is a small, fast and modular HTTP server written in Erlang.

Goals
-----

Cowboy aims to provide the following advantages:

* **Small** code base.
* Damn **fast**.
* **Modular**: transport and protocol handlers are replaceable.
* **Binary HTTP** for greater speed and lower memory usage.
* Easy to **embed** inside another application.
* Selectively **dispatch** requests to handlers, allowing you to send some
  requests to your embedded code and others to a FastCGI application in
  PHP or Ruby.
* No parameterized module. No process dictionary. **Clean** Erlang code.

The server is currently in early development stage. Comments, suggestions are
more than welcome. To contribute, either open bug reports, or fork the project
and send us pull requests with new or improved functionality. You should
discuss your plans with us before doing any serious work, though, to avoid
duplicating efforts.

Quick start
-----------

* Add Cowboy as a rebar or agner dependency to your application.
* Start Cowboy and add one or more listeners.
* Write handlers for your application.

Getting Started
---------------

At heart, Cowboy is nothing more than an TCP acceptor pool. All it does is
accept connections received on a given port and using a given transport,
like TCP or SSL, and forward them to a request handler for the given
protocol. Acceptors and request handlers are of course supervised
automatically.

It just so happens that Cowboy also includes an HTTP protocol handler.
But Cowboy does nothing by default. You need to explicitly ask Cowboy
to listen on a port with your chosen transport and protocol handlers.
To do so, you must start a listener.

A listener is a special kind of supervisor that manages both the
acceptor pool and the request processes. It is named and can thus be
started and stopped at will.

An acceptor pool is a pool of processes whose only role is to accept
new connections. It's good practice to have many of these processes
as they are very cheap and allow much quicker response when you get
many connections. Of course, as with everything else, you should
**benchmark** before you decide what's best for you.

Cowboy includes a TCP transport handler for HTTP and an SSL transport
handler for HTTPS. The transport handlers can of course be reused for
other protocols like FTP or IRC.

The HTTP protocol requires one last thing to continue: dispatching rules.
Don't worry about it right now though and continue reading, it'll all
be explained.

You can start and stop listeners by calling `cowboy:start_listener/6` and
`cowboy:stop_listener/1` respectively.

The following example demonstrates the startup of a very simple listener.

``` erlang
application:start(cowboy),
Dispatch = [
    %% {Host, list({Path, Handler, Opts})}
    {'_', [{'_', my_handler, []}]}
],
%% Name, NbAcceptors, Transport, TransOpts, Protocol, ProtoOpts
cowboy:start_listener(http, 100,
    cowboy_tcp_transport, [{port, 8080}],
    cowboy_http_protocol, [{dispatch, Dispatch}]
).
```

This is not enough though, you must also write the my_handler module
to process the incoming HTTP requests. Of course Cowboy comes with
predefined handlers for specific tasks but most of the time you'll
want to write your own handlers for your application.

Following is an example of a "Hello World!" HTTP handler.

``` erlang
-module(my_handler).
-behaviour(cowboy_http_handler).
-export([init/3, handle/2, terminate/2]).

init({tcp, http}, Req, Opts) ->
    {ok, Req, undefined_state}.

handle(Req, State) ->
    {ok, Req2} = cowboy_http_req:reply(200, [], <<"Hello World!">>, Req),
    {ok, Req2, State}.

terminate(Req, State) ->
    ok.
```

Continue reading to learn how to dispatch rules and handle requests.

Dispatch rules
--------------

Cowboy allows you to dispatch HTTP requests directly to a specific handler
based on the hostname and path information from the request. It also lets
you define static options for the handler directly in the rules.

To match the hostname and path, Cowboy requires a list of tokens. For
example, to match the "dev-extend.eu" domain name, you must specify
`[<<"dev-extend">>, <<"eu">>]`. Or, to match the "/path/to/my/resource"
you must use `[<<"path">>, <<"to">>, <<"my">>, <<"resource">>]`. All the
tokens must be given as binary.

You can use the special token `'_'` (the atom underscore) to indicate that
you accept anything in that position. For example if you have both
"dev-extend.eu" and "dev-extend.fr" domains, you can use the match spec
`[<<"dev-extend">>, '_']` to match any top level extension.

Any other atom used as a token will bind the value to this atom when
matching. To follow on our hostnames example, `[<<"dev-extend">>, ext]`
would bind the values `<<"eu">>` and `<<"fr">>` to the ext atom, that you
can later retrieve in your handler by calling `cowboy_http_req:binding/{2,3}`.

You can also accept any match spec by using the atom `'_'` directly instead of
a list of tokens. Our hello world example above uses this to forward all
requests to a single handler.

There is currently no way to match multiple tokens at once.

Requests handling
-----------------

Requests are passed around in the Request variable. Although they are
defined as a record, it is recommended to access them only through the
cowboy_http_req module API.

You can retrieve the HTTP method, HTTP version, peer address and port,
host tokens, raw host, used port, path tokens, raw path, query string
values, bound values from the dispatch step, header values from the
request. You can also read the request body, if any, optionally parsing
it as a query string. Finally, the request allows you to send a response
to the client.

See the cowboy_http_req module for more information.

Websockets
----------

The Websocket protocol is built upon the HTTP protocol. It first sends
an HTTP request for an handshake, performs it and then switches
to Websocket. Therefore you need to write a standard HTTP handler to
confirm the handshake should be completed and then the Websocket-specific
callbacks.

A simple handler doing nothing but sending a repetitive message using
Websocket would look like this:

``` erlang
-module(my_ws_handler).
-behaviour(cowboy_http_handler).
-behaviour(cowboy_http_websocket_handler).
-export([init/3, handle/2, terminate/2]).
-export([websocket_init/3, websocket_handle/3, websocket_terminate/3]).

init({tcp, http}, Req, Opts) ->
    {upgrade, protocol, cowboy_http_websocket}.

handle(Req, State) ->
    error(foo). %% Will never be called.

terminate(Req, State) ->
    error(foo). %% Same for that one.

websocket_init(TransportName, Req, _Opts) ->
    erlang:start_timer(1000, self(), <<"Hello!">>),
    {ok, Req, undefined_state}.

websocket_handle({timeout, _Ref, Msg}, Req, State) ->
    erlang:start_timer(1000, self(), <<"How' you doin'?">>),
    {reply, Msg, Req, State};
websocket_handle({websocket, Msg}, Req, State) ->
    {reply, << "That's what she said! ", Msg/binary >>, Req, State}.

websocket_terminate(_Reason, _Req, _State) ->
    ok.
```

Of course you can have an HTTP handler doing both HTTP and Websocket
handling, but for the sake of this example we're ignoring the HTTP
part entirely.

Using Cowboy with other protocols
---------------------------------

One of the strength of Cowboy is of course that you can use it with any
protocol you want. The only downside is that if it's not HTTP, you'll
probably have to write the protocol handler yourself.

The only exported function a protocol handler needs is the start_link/3
function, with arguments Socket, Transport and Opts. Socket is of course
the client socket; Transport is the module name of the chosen transport
handler and Opts is protocol options defined when starting the listener.
Anything you do past this point is up to you!

You should definitely look at the cowboy_http_protocol module for a great
example of fast requests handling if you need to. Otherwise it's probably
safe to use `{active, once}` mode and handle everything as it comes.

Note that while you technically can run a protocol handler directly as a
gen_server or a gen_fsm, it's probably not a good idea, as the only call
you'll ever receive from Cowboy is the start_link/3 call. On the other
hand, feel free to write a very basic protocol handler which then forwards
requests to a gen_server or gen_fsm. By doing so however you must take
care to supervise their processes as Cowboy only know about the protocol
handler itself.
