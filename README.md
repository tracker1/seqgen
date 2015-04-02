
```
#######################################################################
WARNING  WARNING  |  WORK IN PROGRESS - DO NOT USE  |  WARNING  WARNING
#######################################################################
```

# seqgen

High availability sequential number generator (Server, Proxy and Client) written in JavaScript meant to run under Node.js or io.js.

The interface provided is a ZeroMQ REQ/RES interface.  Messages are encoded as JSON (UTF-8 without BOM).  Other client libraries will need to adjust to match this style of interface.

* It's best to limit your server instances to 2 or 3 at most.
  * Use proxy instances to better distribute load/connections 
  * proxies do not forward administrative requests, or jumps in last-value
    * this provides a slightly better security/isolation model 
* **SERVERS TO NOT TALK TO EACHOTHER**
  * You should not change the number of instances after deployment.
  * If you have many client nodes, you may wish to setup a number of proxies to act as intermediaries for requests.
* Numbers are not guaranteed to be issued in sequential order between servers.
  * When running multiple instances for high availability each instance will only deliver the offset of instances as values.  For two nodes, one will return ODD numbers, the other will return EVEN.  Variance will happen.
* All numbers are not guaranteed to be issued
  * When the drift between servers occurs, numbers will be skipped to "catch up" to keep numbers close to eachother.


### Reasoning

When running newer clustered databases you will generally rely on larger unique values for record ids.  While this allows for wide clusters and high availability, the down side is that the display and usage of these identifiers are in some cases cumbersome to deal with.

Generally speaking, you want smaller, easier to remember say or telephone numbers to use.  You usually don't care if some numbers are skipped, or that the numbers aren't quite issued in order, only that they are relatively close together, and that they aren't repeated. You also want to be able to continue operating in case of a service failure.

With this in mind, the `seqgen` servers are meant to run 2-3 instances (though you can run more) each issuing only its' own Nth number on request.  With a drift parameter (default 100) and each request sending the last known/received value, this allows for servers to skip in order to catch up once that drift limit is exceeded.  With two servers in use (`seqgen --instance 1 --instances 2` and `--instance 2 --instances 2`) you can have one server dealing ODD numbers, while the other deals EVEN numbers.  This allows you to keep relatively short document numbers in use where needed.

This does not mean you should give up on using UUIDs, it only means that in those cases where you absolutely want to keep numbers in use, you can do so easily.


### Install

Requires ZeroMQ Headers (for [zmq](https://www.npmjs.com/package/zmq))

```
sudo apt-get install zeromq-devel
```

Global install for Proxy or Server instances

```
npm install -g seqgen
```


## Client

This module exposes a node client interface.

```
// module exports a method, call it with an array of proxies or servers to connect to
var seqgen = require('seqgen')(['tcp://10.0.0.11:6411','tcp://10.0.0.12:6411']);

seqgen.next('my_sequence',function(err,value){
  if (err) {
    // do something with error
  }

  //value is a bignum
});
```

Each process will have a connection to each server for request/response processing.  If you want a more complex pipeline, you can run one or more proxy instances to distribute/relay client connections.


## Server / Proxy

You will likely want to run 2-3 server instances for redundancy, more is likely overkill and will create more unused numbers/drift.


### Listen options

* --bind - the zmq uri to bind to 
  * defaults to tcp://0.0.0.0:6410 (any ip)
* --admin - the address:port to bind to for the admin/status http interface
  * defaults to http://0.0.0.0:6411 (any ip)

### Proxy options

In order to run as a proxy, you need to specify one or more `--host` parameters to connect to.

```
seqgen --host tcp://10.0.0.11:6410 --host tcp://10.0.0.12:6410
``` 

### Server Options

A server instance can set the following command line arguments.

* --instance - the number for the current instance
  * positive integer value
  * defaults to 1
* --instances - the number of instances that will be running
  * positive integer value
  * defaults to 1
* --drift - the offset ids allowed before skipping to catch up
  * defaults to 100  
* --data - the directory to use to store data
  * defaults to `/var/lib/seqgen/`  

Start instance #2, when you expect to have 3 instances running (each will return every 3rd number).

```
seqgen --instance 2 --instances 3
```


## Protocol

The examples below are do demonstrate the message contents, and the process.  They are not representative of node workflows.  For node.js or io.js, you should use the client interface.

Messages are buffers/binary containing a UTF-8 encoded JSON string.

### Request

Request is an array, the first item is a case-sensitive string representing the name of the sequence.  The second item is a string representation of the last number received(base10). 

```
sequence_name = "some sequence"
last_id_received = "0" //64-bit integer as base-10 string

request = new Buffer(JSON.stringify([sequence_name, last_id_received]));
```

### Response

The response is an array.  The first item is `null` when no error and an error code when an error occurred.  The second item is a string.  In the case of an error it is the error message.  In the case of a successful response it is a base-10 based number.

```
var [error_code, value] = JSON.parse(response.toString());
if (error_code !== null) {
  var err = new Error(error_code);
  err.code = error_code;
  throw err;
}
return bignum(value,10); //value is a 64-bit integer as string
```
