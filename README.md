# Introduction
[Twitter Iago][iago] looks to be a very capable load testing tool.
However it is rather lacking in examples and explanation. Yes, the [full code][iago-git] is available, but if this is your first introduction to Scala (as it is mine) it becomes overwhelming fast.
The goal here is to create a few basic, but non-trivial, as a launching point to creating "real" load tests.

# Complexities of Iago
* Designed to be very async/concurrent
    * different pieces are started up by using templates and then running the resulting `sh` scripts
* several layers of dynamic compilation
    * eg the config file is dynamically loaded and compiled, and then the values of `Strings` contained in the configuration object are compiled

# Problems Encountered
I suspect/hope that the following are due to my inexperience with Scala. If you know how to do the following, please, please teach me!

## Maven
It's great, but a bit of a pain to package, then unzip things before running

## Running From an IDE
I'm new to Scala and also IntelliJ IDEA; I couldn't figure out how to launch a test run from within the IDE.

I tried setting up a 'tool' to run my `run-iagomaven.sh` script from a custom debug build configuration,  but I couldn't get IntelliJ to pass in project macro values.
 It looked like I would have to hardcode a 'tool' for each project; not very appealing.

## Debugging
I learn a lot by just setting a breakpoint and looking at the state of an application.

I created a custom debug configuration in IntelliJ and set IntelliJ to wait for remote debug connections.  I also set the Debug environment variable as per IntelliJ's help before running Iago.

Sometimes I would see a very brief connection to IntelliJ, but never had it successfully stop at my breakpoint.

I've had limited success reversing the connection strategy and having IntelliJ connect to the application under test. However this requires running the test first, waiting for the prompt, and then hitting enter and starting the IntelliJ debugger as fast as possible.
 It's relatively easy to miss the window of opportunity.

Hitting a breakpoint also seemed to cause death of the application. I've yet to successfully continue execution afterwards. Cleanup (killing java processes) required.

# Want To Do List
* trace through where `customLogSource` in the config file ends up and understand what it is for
    * maybe misnamed; looks like it's executed as part of the configuration of the `Ostrich` run-time environment
* create a very simple, single request, blocking, version of Iago where everything is statically compiled- just to make it easier to debug new load tests.
* add OS detection and enhance the script templating to support other startup scripts - eg windows
* create a patch that adds dynamic types to the `ParrotServerConfig` class so that test specific configuration can be included.
    * right now you can embed custom code in the `loadTest` string but you don't get any help from the IDE (it's a string! ie it's the second layer down in the chain of dynamic compilation.)
* Make a version of `ParrotService`/`ParrotRequest`/`RequestConsumer` etc that call back to user provided call-backs- so Iago could be used to drive non-TCP/IP load tests - trivial use case: console output example that is rate limited.

# Utilities Created
## `run-iagomaven.sh` Script
I have yet to discover how to run Iago tests directly from an IDE (eg IntelliJ Idea) and got tired of repeatedly going through the `mvn package`, `unzip`, and run shuffle.

I've created a bash script that takes care of compiling, packaging, unzipping, setting up debugging environment variables, and running it.

Example usage:

`./scripts/run-iagomaven.sh $PWD examples nodebug 'examples-1.0.jar -f config/LinePrintingExample-config.scala'`

# Examples

## Pre-Requisites
Should be taken care of by the `pom.xml`/Maven.

However [iago] is not (yet?) published to a Maven repository.

So to be able to use these examples:
1. checkout [iago from git][iago-git]
2. `mvn install -DskipTests`

This will build [iago] and publish it to your local Maven cache (typically `~/.m2`.)

## Line Printing Example
The `LinePrintingExample` demonstrates a basic configuration for [Iago] which simply writes out the lines generated by the feeder to the load driver on the console.

It demonstrates:
* how to set up a http based test
* how to pass the configuration to the `loadtest` implementation
* `start`, `shutdown`
* collecting custom statistics

Note: the output rate is uncontrolled- the rate limiting is done by the `[RequestConsumer][https://github.com/twitter/iago/blob/master/src/main/scala/com/twitter/parrot/server/RequestConsumer.scala]` class, which we don't end up using in this example.

One day this might become a reasonable template/starting point for 'real' tests.

From the root directory compile and run via:

`./scripts/run-iagomaven.sh $PWD examples nodebug 'examples-1.0.jar -f config/LinePrintingExample-config.scala'`

## HTTP Requests

### Installing a Test HTTP Service
My colleague, @wilsohn, found this excellent tool for trying HTTP requests: [httpbin].

Install [httpbin] locally so there is no chance of accidental DOS attacks.

Assuming you have python installed then a quick hack to get [httpbin] running locally is:

```
git clone https://github.com/kennethreitz/httpbin.git
cd httpbin/
pip install -r requirements.txt
python manage.py  runserver
```

It should output on the console which port it is running on.

### Example HTTP REST Load Test
The `HttpExample` class implements a simple load test against [httpbin], featuring:
- parsing a simple CSV input file
- making http requests, including setting headers
    - To do: setting query strings
- checking the response, both status code, returned headers, and returned content
- config file that sets a custom logger (Not working yet!), a slow-start-up request rate, and a few other example settings tweaks.

To run:
`./scripts/run-iagomaven.sh $PWD examples nodebug 'examples-1.0.jar -f config/HttpExample-config.scala'`

*Note*
There seems to be a bug in Iago that if there are any connection timeouts (and other errors?) then it never exits.
[httpbin] seems to fail requests at pretty low request rates, so I've found `pgrep -u <user> java` and then `pkill -u <user> java` (after checking that I'm not going to close down other instances of java) to be pretty useful.0



[iago]:http://twitter.github.io/iago/
[iago-git]:https://github.com/twitter/iago/
[ostrich-runtime]:https://github.com/twitter/ostrich#RuntimeEnvironment
[httpbin]:http://httpbin.org/