

#leanbeef - Small, fast, modular HTTP server.#


Copyright (c) 2011-2012 Loïc Hoguin

__Version:__ HEAD

# leanbeaf

leanbeef is a [cowboy](http://github.com/extend/cowboy.git) fork that
aims to put back the simplicity in Cowboy and HTTP handling.

Cowboy is a small, fast and modular HTTP server written in Erlang.

## Goals

Cowboy aims to provide a **complete** HTTP stack in a **small** code
base.  It is optimized for **low latency** and **low memory usage**, in
parts because it uses **binary strings**.

Cowboy provides **routing** capabilities, selectively dispatching
requests to handlers written in Erlang.

Because it uses Ranch for managing connections, Cowboy can easily be
**embedded** in any other application.

No parameterized module. No process dictionary. **Clean** Erlang code.

## Cowboy does nothing by default

Cowboy uses Ranch for handling connections, and provides convenience
functions to start and stop Ranch listeners. The Ranch application must
always be started before Cowboy. The crypto application must also be
started.

The `cowboy:start_http/4` function will handle HTTP connections using
the TCP transport. Similarly, `cowboy:start_https/4` will handle HTTP
connections using the SSL transport.

You can start as many listeners as you need to. To allow this, you are
required to give a name to your listeners. It is the first argument to
the start functions. The name can be of any type.

You can stop listeners using `cowboy:stop_listener/1`, giving it the
name of the listener to be stopped.

The following example demonstrates the startup of a very simple HTTP
listener. It redirects all requests to the `my_handler` module.<pre>application:start(crypto),
application:start(ranch),
application:start(cowboy),
Dispatch = [
    %% {URIHost, list({URIPath, Handler, Opts})}
    {'_', [{'_', my_handler, []}]}
],
%% Name, NbAcceptors, TransOpts, ProtoOpts
cowboy:start_http(my_http_listener, 100, [{port, 8080}],
    [{dispatch, Dispatch}]
).</pre>

This is not enough though, you must also write the `my_handler` module
to process the incoming HTTP requests. Of course Cowboy comes with
predefined handlers for specific tasks but most of the time you will
need to write the handlers appropriate for your application.

Following is an example of a "Hello World!" HTTP handler.<pre>-module(my_handler).
-export([init/3, handle/2, terminate/2]).

init({tcp, http}, Req, Opts) ->
    {ok, Req, undefined_state}.

handle(Req, State) ->
    {ok, Req2} = cowboy_req:reply(200, [], <<"Hello World!">>, Req),
    {ok, Req2, State}.

terminate(Req, State) ->
    ok.</pre>

You can also write handlers that do not reply directly. Instead, such
handlers will wait for an Erlang message from another process and only
reply when receiving such message, or timeout if it did nott arrive in
time.

This is especially useful for long-polling functionality, as Cowboy will
handle process hibernation and timeouts properly, preventing mistakes if
you were to write the code yourself.

It is of course possible to combine both type of handlers together as long as
you return the proper tuple from `init/3`.

Ex:<pre>-module().
-export([init/3, info/3, terminate/2]).

-define(TIMEOUT, 60000).

init({tcp, http}, Req, Opts) ->
	{loop, Req, undefined_state, ?TIMEOUT, hibernate}.

info({reply, Body}, Req, State) ->
	{ok, Req2} = cowboy_req:reply(200, [], Body, Req),
	{ok, Req2, State};
info(Message, Req, State) ->
	{loop, Req, State, hibernate}.

terminate(Req, State) ->
	ok.</pre>

Continue reading to learn how to dispatch rules and handle requests.

Dispatch rules
--------------

Cowboy allows you to dispatch HTTP requests directly to a specific
handler based on the hostname and path information from the request. It
also lets you define static options for the handler directly in the
rules.

To match the hostname and path, Cowboy requires a list of tokens. For
example, to match the "ninenines.eu" domain name, you must specify
`[<<"ninenines">>, <<"eu">>]`. Or, to match the "/path/to/my/resource"
you must use `[<<"path">>, <<"to">>, <<"my">>, <<"resource">>]`. All the
tokens must be given as binary.

You can use the special token `'_'` (the atom underscore) to indicate
that you accept anything in that position. For example if you have both
"ninenines.eu" and "ninenines.fr" domains, you can use the match spec
`[<<"ninenines">>, '_']` to match any top level extension.

Finally, you can also match multiple leading segments of the domain name
and multiple trailing segments of the request path using the atom
`'...'` (the atom ellipsis) respectively as the first host token or the
last path token. For example, host rule `['...', <<"ninenines">>, <<"eu">>]` 
can match both "cowboy.bugs.ninenines.eu" and "ninenines.eu"
and path rule `[<<"projects">>, '...']` can match both "/projects" and
"/projects/cowboy/issues/42". The host leading segments and the path
trailing segments can later be retrieved through
`cowboy_req:host_info/1` and `cowboy_req:path_info/1`.

Any other atom used as a token will bind the value to this atom when
matching. To follow on our hostnames example, `[<<"ninenines">>, ext]`
would bind the values `<<"eu">>` and `<<"fr">>` to the ext atom, that
you can later retrieve in your handler by calling
`cowboy_req:binding/{2,3}`.

You can also accept any match spec by using the atom `'_`' directly
instead of a list of tokens. Our hello world example above uses this to
forward all requests to a single handler.

There is currently no way to match multiple tokens at once.

Requests handling
-----------------

Requests are passed around in the Request variable. Although they are
defined as a record, it is recommended to access them only through the
cowboy_req module API.

You can retrieve the HTTP method, HTTP version, peer address and port,
host tokens, raw host, used port, path tokens, raw path, query string
values, bound values from the dispatch step, header values from the
request. You can also read the request body, if any, optionally parsing
it as a query string. Finally, the request allows you to send a response
to the client.

See the cowboy_req module for more information.

Websockets
----------

The Websocket protocol is built upon the HTTP protocol. It first sends
an HTTP request for an handshake, performs it and then switches
to Websocket. Therefore you need to write a standard HTTP handler to
confirm the handshake should be completed and then the Websocket-specific
callbacks.

A simple handler doing nothing but sending a repetitive message using
Websocket would look like this:<pre>-module(my_ws_handler).
-export([init/3]).
-export([websocket_init/3, websocket_handle/3,
    websocket_info/3, websocket_terminate/3]).

init({tcp, http}, Req, Opts) ->
    {upgrade, protocol, cowboy_websocket}.

websocket_init(TransportName, Req, _Opts) ->
   :start_timer(1000, self(), <<"Hello!">>),
    {ok, Req, undefined_state}.

websocket_handle({text, Msg}, Req, State) ->
    {reply, {text, << "Thats what she said! ", Msg/binary >>}, Req, State};
websocket_handle(_Data, Req, State) ->
    {ok, Req, State}.

websocket_info({timeout, _Ref, Msg}, Req, State) ->
   :start_timer(1000, self(), <<"How you doin'?">>),
    {reply, {text, Msg}, Req, State};
websocket_info(_Info, Req, State) ->
    {ok, Req, State}.

websocket_terminate(_Reason, _Req, _State) ->
    ok.</pre>

Of course you can have an HTTP handler doing both HTTP and Websocket
handling, but for the sake of this example were ignoring the HTTP
part entirely.

As the Websocket protocol is still a draft the API is subject to change
regularly when support to the most recent drafts gets added. Features may
be added, changed or removed before the protocol gets finalized. Cowboy
tries to implement all drafts transparently and give a single interface to
handle them all, however.

## Contribute

For issues, comments or feedback please [create an
issue](http://github.com/refuge/leanbeef/issues).

### Notes for developers

If you want to contribute patches or improve the doc, you will need to
build leanbeef using the `rebar_dev.config`  file. It can also be built
using the **Makefile**:<pre>$ make dev ; # compile & get deps
$ make devclean ; # clean all files</pre>

##Modules##


<table width="100%" border="0" summary="list of modules">
<tr><td><a href="cowboy.md" class="module">cowboy</a></td></tr>
<tr><td><a href="cowboy_bstr.md" class="module">cowboy_bstr</a></td></tr>
<tr><td><a href="cowboy_clock.md" class="module">cowboy_clock</a></td></tr>
<tr><td><a href="cowboy_cookies.md" class="module">cowboy_cookies</a></td></tr>
<tr><td><a href="cowboy_dispatcher.md" class="module">cowboy_dispatcher</a></td></tr>
<tr><td><a href="cowboy_http.md" class="module">cowboy_http</a></td></tr>
<tr><td><a href="cowboy_http_handler.md" class="module">cowboy_http_handler</a></td></tr>
<tr><td><a href="cowboy_loop_handler.md" class="module">cowboy_loop_handler</a></td></tr>
<tr><td><a href="cowboy_multipart.md" class="module">cowboy_multipart</a></td></tr>
<tr><td><a href="cowboy_protocol.md" class="module">cowboy_protocol</a></td></tr>
<tr><td><a href="cowboy_req.md" class="module">cowboy_req</a></td></tr>
<tr><td><a href="cowboy_rest.md" class="module">cowboy_rest</a></td></tr>
<tr><td><a href="cowboy_static.md" class="module">cowboy_static</a></td></tr>
<tr><td><a href="cowboy_websocket.md" class="module">cowboy_websocket</a></td></tr>
<tr><td><a href="cowboy_websocket_handler.md" class="module">cowboy_websocket_handler</a></td></tr></table>
