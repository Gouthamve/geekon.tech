+++
date = "2016-02-22T22:44:58+05:30"
draft = false
title = "Learning networking by reading Go net package"

slug = "learning-networking-golang-net-package"

socialsharing = true

author = "Goutham"
authortwitter = "https://twitter.com/putadent"

categories = ["networking", "golang"]
+++
<script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/gist-embed/2.4/gist-embed.min.js"></script>

# Prologue

After attending [GopherConIndia](http://www.gophercon.in) and talking to [Brad](https://github.com/bradfitz), I wanted to contribute to the GoLang project (More on that later). And I chose [this](https://github.com/golang/go/issues/5757) as my first issue to get used to the contribution process.

As I was diving through the `net` package source trying to document all the methods and functions exposed, I realized that I was slowly going one level deeper to the internal workings. I didn't have a networking course until now, and I decided to understand networks by reading the source.

This is a completely new approach to learning stuff, read the source and Google a little to learn networks and I was pleasantly surprised. I could understand to fair degree how sockets worked and I am now confident that I can implement atleast the first four assignments in next sems Networking course :P

# DialTCP

Yep. I am going to cover one function `DialTCP` and show how it uses a socket to compose the whole connection. The `http` package implements the HTTP protocol on top of the net package and **huge** shoutout to [@bradfitz](https://twitter.com/bradfitz) for his work. Below is a simple GET request to illustrate this.

I will be posting code via github gists through out this post and they are fully runnable samples. You can view the full file by clicking on the name of the file on the bottom left corner. We have an example function to send a GET request to http://google.com/ using DialTCP. The full [gist](https://gist.github.com/Gouthamve/7c507e49c083d3438ed1#file-get-go) also contains a slightly modified example using `net.Dial`. While `Dial` seems to be a lower level implementation, it just uses a [switch case](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/dial.go#L368-L383) internally and invokes `net.DialTCP`

<code data-gist-id="7c507e49c083d3438ed1" data-gist-line="15-38"></code>

## Some Basics

So, lets see how opening a connection works. As I said, I am no expert, but Google led me to <a href="https://beej.us/guide/bgnet/output/html/multipage/theory.html" target="_blank">this</a> and he does it exceptionally well!

Now with that in mind, things get very simple. When we call `net.DialTCP`, these are the function calls that are made:


  * [DialTCP](https://github.com/golang/go/blob/24a83d35453e29087c4d63954bb8c78a982fc207/src/net/tcpsock_posix.go#L158-L168) calls [dialTCP](https://github.com/golang/go/blob/24a83d35453e29087c4d63954bb8c78a982fc207/src/net/tcpsock_posix.go#L170-L208)
  * dialTCP calls [internetSocket](https://github.com/golang/go/blob/24a83d35453e29087c4d63954bb8c78a982fc207/src/net/tcpsock_posix.go#L201) which returns a socket
  * [newTCPConn](https://github.com/golang/go/blob/24a83d35453e29087c4d63954bb8c78a982fc207/src/net/tcpsock_posix.go#L207) binds the socket to give a net.TCPConn


## internetSocket

We now know that we need a network file-descriptor of type `SOCK_STREAM` to get a TCP connection. And thats what [internetSocket](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/ipsock_posix.go#L159-L162) does. It internally [spawns](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/sock_posix.go#L37) a `SOCK_STREAM`. Now we will get a [socket](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/sock_posix.go#L38) which is mapped to a [struct](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/fd_unix.go#L19-L34) [netFd](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/sock_posix.go#L46) for manipulation. Next we [intialise](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/sock_posix.go#L89-L92) the socket by [dialing](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/sock_posix.go#L120-L153), i.e, [set](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/sock_posix.go#L148) [the](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/sock_posix.go#L101) [TCP addresses](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/tcpsock_posix.go#L16-L24) of the socket from its socketAddr.

## newTCPConn

Now that we have a file-descriptor, we need to perform the Read and Write operations on it. We use a interface [`Conn`](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/net.go#L112-L157) which is a superset of the `io.Reader` and `io.Writer`. [conn](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/net.go#L159-L177) implements a the `Conn` interface. If you look into [`Read`](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/net.go#L168-L177) and [`Write`](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/net.go#L180-L189) methods, you will see that we are calling [`fd.Read`](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/fd_unix.go#L246) which performs Read/Write [syscalls](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/fd_unix.go#L246) to read and write. Go uses a networkPoller detailed <a href="http://morsmachine.dk/netpoller" target="_blank">here</a>, which adds some complexity to the `syscall` but it can be safely ignored :)

Now while we have a generic conn, the [`TCPConn`](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/tcpsock_posix.go#L43-L47) embeds `conn` and implements additional functions like [`SetKeepAlive`](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/tcpsock_posix.go#L118-L128). [`newTCPConn`](https://github.com/golang/go/blob/ebf1f0fcbe7127fc6a96b57ac41d886ae36aaa66/src/net/tcpsock_posix.go#L49-L53) just takes the netFD and returns a new `TCPConn`.

Boom. Thats it. The length of the post seems to be small, but all of the content is inside the links which point to the source-code.

# Epilogue

People said GoLang is easy to understand. Hell yes it is! And reading source code and blogging about it is a great way to learn. Expect more from me on this front.

Learnings:

  1. TCP connection needs a TCP socket which is gotten by using `SOCK_STREAM`
  2. Go uses `read` and `write` syscalls to read and write from the socket

TODO:

  1. Implement a simple TCP Client.
  2. Try to do it using the `send` and `recv` syscalls as mentioned in the above networking post.

<script type='text/javascript'>$(document).ready(function(){$("a[href^='http://']").each(function(){-1==this.href.indexOf(location.hostname)&&$(this).attr("target","_blank")}),$("a[href^='https://']").each(function(){-1==this.href.indexOf(location.hostname)&&$(this).attr("target","_blank")})});</script>
