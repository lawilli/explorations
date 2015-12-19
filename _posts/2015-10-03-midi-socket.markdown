---
layout: post
title:  "MIDI Sockets"
date:   2015-10-03 23:42:45
tags: midi socket websockets
---

[midi-socket](http://midi-socket.azurewebsites.net) is an application that allows for a source to play MIDI for listeners.

### MIDI

For the MIDI portion, I utilized this [demo](http://www.keithmcmillen.com/blog/making-music-in-the-browser-web-midi-api/).

The application can't access the necessary resources from the filesystem without a server or some other means. From the root directory of the project, use `python` as a server, connect your MIDI controller and try it out.

{% highlight sh %}
$ python -m SimpleHTTPServer 8080
{% endhighlight %}

### socket.io

This application utilizes **websockets** via **socket.io**. Websockets are useful for real-time communication between clients. Socket.io is a wrapper for websockets.

> Socket.IO enables real-time bidirectional event-based communication.
It works on every platform, browser or device, focusing equally on reliability and speed.

Wikipedia says:

> It has two parts: a client-side library that runs in the browser, and a server-side library for node.js. Both components have a nearly identical API. Like node.js, it is event-driven.

I didn't find the documentation or examples provided on the site to be that helpful for quick usage. I did some googling to find [this example](http://www.programwitherik.com/socket-io-tutorial-with-node-js-and-express/), which utilizes [Express](http://expressjs.com/) and [Node.js](https://nodejs.org/en/) to make the bare bones of a chat application.

When a user (player) is connected to `index.html` with a MIDI device connected to their computer, the user can play a note. The MIDI information is sent to the server via a `play` event. The data from the `play` event is then broadcast to listeners on `listener.html` via a `broad` event.

Here is the abstraction:

* server >> app.js
* player >> index.html
* listener >> listener.html

Some notable code is provided below. All actual MIDI aspects are excluded, since the noteworthy portion of this application and exploration is the real-time aspect.

{% highlight js %}
// app.js
var express = require('express');
var app = express();
var server = require('http').createServer(app);
var io = require('socket.io')(server);

// the path to static resources
app.use(express.static(__dirname + '/public'));

// route definition
app.get('/', function (req, res, next) {
    res.sendFile(__dirname + '/index.html');
});

// route definition
app.get('/listener', function (req, res, next) {
    res.sendFile(__dirname + '/listener.html');
});

io.on('connection', function (client) {

    // notify server that client is connected
    console.log('Client connected...');

    // '/' route emits 'play' event
    // '/listener' route listens for 'broad' event
    client.on('play', function (data) {
        // optional logging on server
        console.log(data);
        console.log('note played');

        // emit event to everyone except sender
        client.broadcast.emit('broad', data);
    });
});

server.listen(4200);
{% endhighlight %}

{% highlight javascript %}
// listener side
(function(){

    // ...

    // handler for MIDI events
    function onMIDIMessage(data){ /* ... */ }

    // ...

    // connect to server
    var socket = io.connect('localhost:4200');

    //
    socket.on('connect', function (data) {
        socket.emit('join', 'new listener');
    });

    // assign onMIDIMessage as event handler for broad events
    //  i.e. for when the player emits a 'play' event
    socket.on('broad', onMIDIMessage);

})();
{% endhighlight %}

{% highlight javascript %}
// player side
(function(){

  // ...

  function onMIDIMessage(event){
    console.log('%c Oh my heavens! You\'ve played a note!', 'background: #222; color: #bada55');
    data = event.data,
    socket.emit('play', data);

    // and then handle the data to actually play the note in the client
    // ...
  }

  // more audio functions
  // ...

  //
  var socket = io.connect('localhost:4200');
  socket.on('connect', function (data) {
    socket.emit('join', 'Hello World from client');
  });

})();
{% endhighlight %}

Client side debugging is possible by entering this in your browser console.
{% highlight javascript %}
localStorage.debug = '*';
{% endhighlight %}

To test listener functionality, via the `/listener` route, I used another device to navigate to the IP address in which the `node` server was running with the port that resources were being served to. The page loaded as expected, however `socket.io` didn't work correctly. This may be due to a network firewall. To effectively ensure that this application worked, I needed to deploy this to a production server. For this, I utilized the [Microsoft Azure documention](https://azure.microsoft.com/en-us/documentation/articles/cloud-services-nodejs-develop-deploy-app/) for deploying a `node` application.

After pushing to Azure, some additional server-side configuration may be necessary.

### TODO

1. Change interface and instrument to [something more melodic](http://www.google.com/doodles/robert-moogs-78th-birthday)
2. Enhance synchronization between successive notes for the listener.
