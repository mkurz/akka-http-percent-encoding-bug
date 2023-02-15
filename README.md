akka-http's experimental http2 support does not correctly handle URL parsing errors.
Specially when passing an invalid percent-encoded character in the path and/or query string (here `%_D`):

This reproducer is exactly the example from the akka docs, but with HTTP/2 enabled (see the first two commits of this repo):

* https://doc.akka.io/docs/akka-http/current/introduction.html#low-level-http-server-apis
* https://doc.akka.io/docs/akka-http/current/server-side/http2.html#enable-http-2-support

```sh
# Start the server
sbt run
```

```sh
# HTTP/1 works:
$ curl --http1.1 -v http://localhost:8080/?param=%_D
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /?param=%_D HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.87.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 400 Bad Request
< Server: akka-http/10.2.10
< Date: Wed, 15 Feb 2023 16:29:18 GMT
< Connection: close
< Content-Type: text/plain; charset=UTF-8
< Content-Length: 78
< 
* Closing connection 0
Illegal request-target: Invalid input '_', expected HEXDIG (line 1, column 10)
```

```sh
# HTTP/2 gives error:
$ curl --http2-prior-knowledge -v http://localhost:8080/?param=%_D
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
* Using HTTP2, server supports multiplexing
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* h2h3 [:method: GET]
* h2h3 [:path: /?param=%_D]
* h2h3 [:scheme: http]
* h2h3 [:authority: localhost:8080]
* h2h3 [user-agent: curl/7.87.0]
* h2h3 [accept: */*]
* Using Stream ID: 1 (easy handle 0xaaabd90f0dc0)
> GET /?param=%_D HTTP/2
> Host: localhost:8080
> user-agent: curl/7.87.0
> accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 256)!
* Closing connection 0
curl: (16) Error in the HTTP2 framing layer
```

At some point, no matter if using HTTP/1 or HTTP/2, the parser ends up here
* https://github.com/akka/akka-http/blob/v10.2.10/akka-http-core/src/main/scala/akka/http/impl/model/parser/UriParser.scala#L244-L246

As you can see it will (also) be looked for `pct-encoded` characters, which expect a `%` sign, followed by two `HEXDIG` signs:
* https://github.com/akka/akka-http/blob/v10.2.10/akka-http-core/src/main/scala/akka/http/impl/model/parser/UriParser.scala#L289-L294

Now if that fails...
* **...when using HTTP/1...**

...then the method [`parseHttpRequestTarget`](https://github.com/akka/akka-http/blob/v10.2.10/akka-http-core/src/main/scala/akka/http/impl/model/parser/UriParser.scala#L309-L315) fails with (=throws) an `IllegalUriException`, which will later be handled so a 400 Bad Request will following body will be send:
```
Illegal request-target: Invalid input '_', expected HEXDIG (line 1, column 10)
```

(There are various places in the akka-http-core module where `IllegalUriException`s and ` ParsingException`s are caught and handled)

* **...when using HTTP/2...**

`PathAndQuery.parse` in following line throws an `ParsingException` which is never handled (this is in the `akka.http.impl.engine.http2` package, so not used by HTTP/1):

* https://github.com/akka/akka-http/blob/v10.2.10/akka-http-core/src/main/scala/akka/http/impl/engine/http2/hpack/HeaderDecompression.scala#L67

A couple of lines later only an `IOException` gets caught:

* https://github.com/akka/akka-http/blob/v10.2.10/akka-http-core/src/main/scala/akka/http/impl/engine/http2/hpack/HeaderDecompression.scala#L81-L90

If you see where the containing method `parseAndEmit` gets called, the `ParsingException` never gets handled.