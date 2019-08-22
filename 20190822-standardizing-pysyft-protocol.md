# Standardizing PySyft Protocol


| Status        | Proposed        |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Justin Patriquin (justin@dropoutlabs.com) |
| **Sponsor**   | Jason Mancuso  (jason@dropoutlabs.com)                  |
| **Updated**   | 2019-08-22                                           |

## Objective

- Standardize the PySyft Protocol and the serialization framework using gRPC and Protobufs
- Improve the protocol as needed to help support the standardization process
- Make something that is easily extendable as new features are added

## Motivation

Viewing PySyft as a generic platform for encrypted, privacy-preserving deep learning, the protocol to this platform could be implemented in any programming language. To ease this process it would be great if the PySyft Protocol was standardized and documented so that users/developers can easily look it up and understand it.

Some other benefits:

- gRPC is a highly tuned HTTP/2 protocol so there should be performance benefits to implementing the protocol in it.
- Not having to hand write the protocol in every language will reduce code duplication between different languages.

There has been some interest in standardizing the protocol related to Jason’s work on generalizing the hooking mechanisms in the PySyft, see [here](https://github.com/OpenMined/rfcs/pull/3#issuecomment-519863747).

## User Benefit

Developers creating new clients to PySyft will benefit from the protocol being standardized. These developers will not need to dig into the PySyft code base to figure out the protocol, they will just have to look at the Protobuf definition. They will also benefit from the code generation provided by gRPC/protobufs. This will help reduce code complexity and reduce the number of bugs.

## Design Proposal

Here we propose that we should replace PySyft’s current messaging protocol and framework with something well-defined in gRPC and Protobufs.

This will be a fairly big change to the PySyft codebase but it shouldn’t have any user impact as these details should be hidden. Developers of projects like Grid and syft.js will be impacted as they rely on the current protocol. The developers will need to update the projects to use the new protocol. It is out of scope of this document, but to ease this process it would help to come up with a migration strategy for dependent projects.

There are three major changes that will need to take place:

1. All of the messages defined by [syft/codes.py](https://github.com/OpenMined/PySyft/blob/dev/syft/codes.py#L1) will need an analogous RPC call in gRPC. The structure of each message will need to be strictly defined for each RPC call.

2. The recent [messaging abstraction](https://github.com/OpenMined/PySyft/blob/dev/syft/messaging/message.py) will be replaced by the Protobuf definition. We should be able to use this abstraction to help define what the messages being sent by the RPC calls actually look like.

3. One of the biggest changes will be around how the Worker abstraction works. Currently if you’re creating new Workers you only need to override `_send_msg` and `_recv_msg`. With the introduction of gRPC/Protobufs the sending/receiving is hidden and taken care of automatically with gRPC so its not immediately clear if it’s worth having these two bottleneck functions at the PySyft level (it might not even be possible). With gRPC we’ll need to create a series of send and receive functions that must be overridden by subclasses. Each RPC defined by the Protobuf file would require its own send and receive function that would then need to be overridden by the subclass.

One thing to consider is that although we’re leaning heavily on gRPC it is still possible to remove it as a dependency. For example, if you wanted to use a different transport mechanism (e.g. Websockets, hand-written UDP protocol) you can just serialize the Protobuf messages directly and send those using whatever mechanism you’ve chosen.

### Performance Benefits

With this proposal we should see performance benefits, decreased maintenance costs and decreased costs when trying to implement PySyft across platforms.

Performance benefits comes down to using the types correctly to define the exact objects that are being passed across the wire. Currently, objects are being serialized using msgpack which has the flexibility to turn any object into a binary blob. To begin implementation we might be able to continue to use msgpack but if we can anticipate what the objects we’re passing around are it’d be ideal to give these precise types. This will help us take advantage of all the performance tuning that gRPC/Protobufs has done. gRPC also has optional compression builtin so we can take advantage of that as needed.

### Non-goals

While this proposal’s intention is to get all of the frameworks using a well defined RPC inteface, it is out of scope for gRPC to define how these messages actually get used. One example is how PySyft sends garbage collection messages when a worker is stopped, it is an implementation detail that cannot be defined by gRPC. Ideally this is a first step towards creating an implementation standard or designing the message flow when certain objectives are trying to be achieved.

### Rejected Alternative Designs

#### Hand-Written Binary Protocol with Documentation

Currently PySyft is using a hand-written binary protocol where the data is serialized with msgpack. A valid design for standardizing this could be to just write a standardization document detailing the precise protocol. This idea was rejected because this document would be hard to maintain and the protocol would need to be hand-written in every language increasing complexity, the chance of bugs (security bugs and other) and would need a lot of work to tune for performance.

#### Capn Proto

[Capn Proto](https://capnproto.org/) is quite similar to gRPC and protobufs. It provides strongly typed data interchange format and servers that can exchange this data. It seems like a very strong contender to gRPC and protobufs but there are a couple downsides mainly related to how gRPC has much more widespread use.

First, while it might have some security scrutiny it does not have as much scrutiny as gRPC/protobufs would have. Secondly, both PyTorch and TensorFlow have relations with gRPC/Protobufs so it might be helpful to continue to use gRPC just in case this becomes important in the future.

## Detailed Design

Here I’m going to give some examples of how the messages might actually look when implemented for the `BaseWorker` and `VirtualWorker` as well as should what a .proto file could start to look like for PySyft.

### Testing with Virtual Worker and Protobufs

One of PySyft’s most powerful features is the ability to use the Virtual Worker to iterate on features/bug fixes without having the overhead of setting up network connections. With the introduction of gRPC/Protobuf we can continue to do this by ignoring the gRPC side for the most part.

Here’s an example `BaseWorker` creating common interface which can then be used by the `VirtualWorker` and other workers who want to actually use a gRPC network connection. The below might not be how this actually get implemented but should be a good guiding example.

```
class BaseWorker(abc.ABC):

  # this is in the form that will be expected by gRPC
  @abc.abstractmethod
  def ExampleSyftMessage(self, request, context):
     raise NotImplementedError

  # id and owner are args to ExampleSyftMessage
  # location is the worker to send to
  def send_example_syft_message(id, owner, location):
    raise NotImplementedError
```

Then the `VirtualWorker` can implement the above interface:

```
class VirtualWorker(BaseWorker):

  # this is in the form that will be expected by gRPC
  def ExampleSyftMessage(self, request, context):
     # handle message here
     handle_example(request.id, request.owner)

     res = ExampleResponse(ok=True)

     return res

  # id and owner are args to ExampleSyftMessage
  # location is the worker to send to
  def send_example_syft_message(id, owner, location):
    request = ExampleRequest(id=id, owner=owner)

    # just call handler directly, set context to None
    location.ExampleSyftMessage(request, None)
```

### Example PySyft gRPC Service

```protobuf
syntax = "proto3";

service Syft {
    rpc SendCommand(Command) returns (CommandResponse) {}

    // ID is the ID of the tensor
    rpc GetTensorShape(ID) returns (TensorShape) {}

   // … rest of the commands go here
   // … object request, object delete, set object
}

message Command {
    // sender id
    string id = 1;

    // the actual message ideally this can be defined better
    // to take advantage of serialization performance benefits
    string message = 2;
}

message CommandResponse {
    Status status = 1;
}

message ID {
  string id = 1;
}

message TensorShape {
  repeated int shape = 1;
}

message Status {
    bool ok = 1;
    string error_message = 2;
}

```

## Questions and Discussion Topics

- Can we give all the objects passed over the network strong types?
- Are there any improvements that can be made to the current messages? Can we remove or add any messages to make the interface better while we have the chance?
- What other abstractions could be affected by these changes? Is there something we’re missing related to [Plans](https://github.com/OpenMined/PySyft/blob/dev/syft/messaging/plan.py)?

## Appendix

### Generating Code with gRPC

gRPC/Protobuf generates a lot of code so that making RPC calls over the network is very easy. To do this in python the user first needs to install `grpcio` and the related tools package `grpcio-tools`. This can be done with pip:

```bash
pip install grpcio grpcio-tools
```

If you create a .proto file such as syft.proto and put its in a `protos/syft` directory in the root of the project you can then run the following command to generate all of the required code:

```bash
python -m grpc_tools.protoc -Iprotos --python_out=. \
            --grpc_python_out=. protos/syft/syft.proto
```

This will generate the code and put it in the `syft` directory. This step can be performed automatically as a pre-commit hook whenever the protobuf has changed (which should be minimal once the protocol has stabilized).

### Starting a Server/Connecting with a Client

Example of how to start a gRPC server:

```
# create the server with 10 workers, each request get run in a separate thread
server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))

# this tells grpc about the Servicer who will handle messages and the server which handles
# forwards the incoming messages to the Servicer
syft_pb2_grpc.add_SyftServicer_to_server(SyftServicer(), server)

# adds an insecure port to IPv6 localhost
server.add_insecure_port(f'[::]:{port}')

server.start()
```

Example of how to create a channel:

```
# creates a channel using the given address and port
channel = grpc.insecure_channel(f"{address}:{port}")

# creates the stub which is what is used to send messages
stub = syft_pb2_grpc.SyftStub(channel)

# example using stub to send a message
stub.ExampleSyftMessage(id=..., owner=...)
```
