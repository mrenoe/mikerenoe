+++
title = "Learning Go by writing a TCP Server"
date = 2019-04-30
draft = false
summary = "Takeout Central had a problem. We were running a TCP Server and a UDP Server. Both were written in Perl. Neither was taking advantage of concurrency. At the same time, our team of Delivery Heroes…"
tags = ["golang", "iot", "on-demand", "servers", "system-language"]
canonicalURL = ""
+++

![](/images/blog/learning-go-by-writing-a-tcp-server/img-1.jpeg)
*Photo by [Matthew Henry](https://burst.shopify.com/@matthew_henry?utm_campaign=photo_credit&utm_content=Free+Server+Ethernet+Photo+%E2%80%94+High+Res+Pictures&utm_medium=referral&utm_source=credit) from [Burst](https://burst.shopify.com/communication?utm_campaign=photo_credit&utm_content=Free+Server+Ethernet+Photo+%E2%80%94+High+Res+Pictures&utm_medium=referral&utm_source=credit)*

[Takeout Central](https://www.takeoutcentral.com) had a problem. We were running a TCP Server and a UDP Server. Both were written in Perl. Neither was taking advantage of concurrency. At the same time, our team of Delivery Heroes became frustrated while beta testing an IoT device in the field. Information sent to the ten beta test devices took an hour or more to update, which we found to be an unacceptable amount of time.

Enter Go. I’d used Go in the past. It was my language of choice for [Advent Of Code](https://adventofcode.com/) 2017. As I kept up with the five days of challenges, I found that writing in Go was a breeze. After that, I kept an eye out for a problem that called for a robust systems language. Here was the opening.

Go uses concurrency rather than parallelism. While the differences between concurrency and parallelism are slight to most, co-designer of the Go programming language Rob Pike explains it like this: “Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.”[\*](https://blog.golang.org/concurrency-is-not-parallelism)

First, I wanted to know how concurrency functioned in Go. It was so straightforward that even beginner [tutorials](https://tour.golang.org/concurrency/1) discussed it relatively early. Next, I needed to know how to implement concurrency in a server environment. After a few more minutes of googling, I found the scaffold for the TCP server. I learned how Go handles socket communications on this [blog](https://coderwall.com/p/wohavg/creating-a-simple-tcp-server-in-go), which also explains how these can all be leveraged using goroutines. The go keyword before handleRequest does all of the work for concurrency, because each function called with a preceding go starts a new concurrent process. Getting the server off the ground was easy, because all it entailed was prefacing a function call with “go”.

```go
//Taken from https://coderwall.com/p/wohavg/creating-a-simple-tcp-server-in-go
package main

import (
    "fmt"
    "net"
    "os"
)

const (
    CONN_HOST = "localhost"
    CONN_PORT = "3333"
    CONN_TYPE = "tcp"
)

func main() {
    // Listen for incoming connections.
    l, err := net.Listen(CONN_TYPE, CONN_HOST+":"+CONN_PORT)
    if err != nil {
        fmt.Println("Error listening:", err.Error())
        os.Exit(1)
    }
    // Close the listener when the application closes.
    defer l.Close()
    fmt.Println("Listening on " + CONN_HOST + ":" + CONN_PORT)
    for {
        // Listen for an incoming connection.
        conn, err := l.Accept()
        if err != nil {
            fmt.Println("Error accepting: ", err.Error())
            os.Exit(1)
        }
        // Handle connections in a new goroutine.
        go handleRequest(conn)
    }
}

// Handles incoming requests.
func handleRequest(conn net.Conn) {
  // Make a buffer to hold incoming data.
  buf := make([]byte, 1024)
  // Read the incoming connection into the buffer.
  reqLen, err := conn.Read(buf)
  if err != nil {
    fmt.Println("Error reading:", err.Error())
  }
  // Send a response back to person contacting us.
  conn.Write([]byte("Message received."))
  // Close the connection when you're done with it.
  conn.Close()
}
```

I’d become comfortable with concurrency and goroutines, so I turned my attention back to the problem at Takeout Central. To replace the old system that was slowing us down without losing any functionality, we needed the new TCP server to:

1.  Concurrently handle each Device-to-Server TCP connection using goroutines
2.  Respond to *valid* requests with a JSON packet
3.  Log requests, failures, updated states
4.  Update a MySQL database accordingly

### JSON

Because our devices sent requests and expected responses in JSON, it was my next topic of interest. The [json](https://golang.org/pkg/encoding/json/) package [Marshal](https://golang.org/pkg/encoding/json/#Marshal) function does the work producing the JSON string and it’s documentation lays out the possible inputs.

I chose to use an exported struct. It offered clearly-defined messages, meaning the struct was also self-documenting. I prefer to use explicit key names whenever possible, which is why I also utilized the ‘json’ tag in cases where a device’s expected key did not conform to Go’s style.

```go
type errorMessage struct {
		Error string `json:"error"`
	}
```

This struct is populated using the explicit assignment seen below where the variable message is a string containing a user defined error message. The resulting struct is Marshaled into a byte slice to be returned by our connection handler.

```go
ourErrorMessage := errorMessage{Error: message}
errorMessageAsJSON, err := json.Marshal(newMsg)
```

On the flip side, requests need to be validated by the [Unmarshal](https://golang.org/pkg/encoding/json/#Unmarshal) function. Given the errorMessage struct above we can put together the pieces to get an error message from the client using the code below. If invalid JSON is passed, the Unmarshal function will pass an error for our server to log. This validates that messages are at least using valid JSON.

The standard Unmarshal function omits fields that don’t exist in your struct definition by default. Here at Takeout Central on this project specifically we’re working with standard messages with very little variance between them. If that’s not the case for your application, you might want to look into handling unexpected messages using a different interface than a struct.

```go
//given a JSON String
var errorRequest errorMessage 
request := "{\"error\": \"Unknown Error Occurred\"}"
err := json.Unmarshal([]byte(request), &errorRequest)
if err != nil{
  //handle error
}
errorMessage = errorRequest.Error
```

### Logging

For logging, Go provides the [log](https://golang.org/pkg/log/) package. We start here by initializing a pointer to [Logger](https://golang.org/pkg/log/#Logger) in main using [Log.new](https://golang.org/pkg/log/#New). Using this pointer and dependency injection we pass this along to the handleConnection function and furthermore into our subsequent log helper functions.

```go
f, err := os.OpenFile("/your/log/file", os.O_RDWR|os.O_CREATE|os.O_APPEND, 0666)
if err != nil {
  log.Fatalf("error opening file: %v", err)
}
defer f.Close()
logger := log.New(f, "", 0)
```

For the formatting I prefer something along the lines of 2019–04–25 21:26:17 \[error\], and so our log functions look like this:

```go
func logError(l *log.Logger, msg string) {
	l.SetPrefix(time.Now().Format("2006-01-02 15:04:05") + " [error] ")
	l.Print(msg)
}
```

It’s important to note here that when using dependency injection on concurrent functions the objects you’re passing need to be thread-safe. This is necessary to avoid race conditions as well as ensuring one of the many concurrent processes don’t fail because it’s attempting to modify our log file while it’s in use. Luckily for us, the documentation for [Logger](https://golang.org/pkg/log/#Logger) spells that out explicitly.

If one thing has come through thus far, I hope it’s how straightforward and ‘batteries-included’ Go is for these tasks in particular. Up until now and in fact, excluding the MySQL drivers I’m about to delve into, everything in our server is powered by the Go packages from the core team.

### MySQL

Go doesn’t implement each database in their own core libraries, it’s been left as a challenge to the users or the [reader](https://en.wiktionary.org/wiki/exercise_for_the_reader). They do provide the [sql](https://golang.org/pkg/database/sql/) package as an interface and a comprehensive list of [drivers](https://github.com/golang/go/wiki/SQLDrivers) for your ease. I went with [go-sql-driver](https://github.com/go-sql-driver/mysql/) as it best fit my projects needs.

go-sql-driver and Go sql follow standard patterns if you’ve worked with MySQL before. Starting with a pointer to the DB(database) object we use it like Logger before as this is also designed to be [threadsafe](https://golang.org/pkg/database/sql/#DB) as you might expect.

For our use-case, we’re monitoring and updating devices one at a time. This means there’s very few cases where we have to fetch more than one row from the database, meaning our requests and data grabs tend to look like the code below. Like log our database pointer is initialized in main and passed to each handleConnection goroutine.

The variables you see that aren’t initialized in view: dbh, our database pointer; log, pointer to our log object; errors which is a [package](https://golang.org/pkg/errors/) Go provides to handle passing errors between functions.

```go
var foo, baz int
err := dbh.QueryRow("SELECT foo, baz FROM devices WHERE id=? LIMIT 1", ID).
  Scan(&foo, &baz)
if err != nil {
  logError(log, "No devices with ID: "+ID)
  return nil, errors.New("No devices with ID:" + ICCID)
}
```

### Wrapping it all together

This was one of the most enjoyable projects I’ve worked on recently and I owe it largely to Go’s internal packages and tools. In particular, goroutines as a thread abstraction. Having worked with threads manually in the past I fully appreciate just how much heavy lifting they do. The simple ‘go’ keyword provided a smooth abstraction that is less prone to error and automatically efficiently schedules threads. If you’re implementing this yourself, please keep in mind other aspects of a production server like malicious attackers and prepare accordingly. With that in mind though, this is a great TCP server app for a few IoT devices connecting to a central server; I hope it serves you well.

* * *

[Learning Go by writing a TCP Server](https://blog.takeoutcentral.com/learning-go-by-writing-a-tcp-server-54ed2c28152e) was originally published in [Takeout Central](https://blog.takeoutcentral.com) on Medium, where people are continuing the conversation by highlighting and responding to this story.
