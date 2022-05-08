# Using gRPC

`gRPC` was released by Google. It is a high performance Remote Procedure Call (RPC) platform. gRPC uses protobuf (protocol buffers) to the payload. Due to the functionalities, it is common to compare with HTTP API /w JSON.

| Items | gRPC | HTTP API /w JSON |
| -- | -- | -- |
| Contract | Necessary (.proto) | Optional (OpenAPI) |
| Protocol | HTTP/2 | HTTP |
| Payload | Protocol Buffers (Binary) | JSON (Human Readable) |
| Prescriptiveness | Constraint | Loose |
| Streaming | Client-Server, Bidirectional | Client-Server |
| Security | TLS | TLS |

Basically gRPC is good at performance, code generation, strict specification (for parsing), streaming (especially for bidirection), and timeout/deadline than the HTTP API system. However, gRPC's disadvantages are that are hard for human reading, and limited browser support.

In the repository, I will show you how the gRPC works. If you are not familiar with the protocol buffers, please have a look at [using-protobuf](https://github.com/jiankaiwang/using-protobuf).

## How to start

### Create the rpc service on the protobuf

The first thing is to add the rpc definition to the protobuf. The following is a simple example.

```text
/* ... */

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
  int32 value = 2;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}

/* ... */
```

After that, you can get the auto-generated script or source code by using the `protoc` (protobuf compiler). Here I use Python as the example.

```sh
protoc --python_out=. helloworld.proto
```

In this example, two scripts are generated, `helloworld_pb2.py` and `helloworld_pb2_grpc.py`. 
**Notice you don't need to manipulate these scripts at all.**

Next you can start diving to developing both the server-side and the client-side of gRPC.
In the server side, you have to implement the function taking the inputs from the client and returning the outputs.
In addition, you have to define the access port too.

```py
# ...

class Greeter(helloworld_pb2_grpc.GreeterServicer):
  def SayHello(self, request, context):
    return helloworld_pb2.HelloReply(
      message='Hello, {}! Your value is {}.'.format(request.name, request.value))


def serve():
  server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
  helloworld_pb2_grpc.add_GreeterServicer_to_server(Greeter(), server)
  server.add_insecure_port('[::]:50051')
  server.start()
  server.wait_for_termination()

# ...
```

To the client side, you have to implement the way of sending data to the server.

```py
# ...

def run():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = helloworld_pb2_grpc.GreeterStub(channel)
        response = stub.SayHello(helloworld_pb2.HelloRequest(name='you', value=100))
    print("Greeter client received: " + response.message)

# ...
```

### Let's put them together

In the first you have to create the runtime.

```sh
python3 -m virtualenv -p python3 env
source ./env/bin/activate
pip3 install --no-cache-dir -r ./requirements.txt
```

You have to install the protobuf compiler. If you are not familiar with this step, please have a look at [using-protobuf](https://github.com/jiankaiwang/using-protobuf). You can also check the protoc state by typing `protoc --version`.

Now let's build the protobuf.

```sh
protoc --python_out=. helloworld.proto
```

Please make sure two necessary files are generated. (`helloworld_pb2.py` and `helloworld_pb2_grpc.py`)

Next in one terminal, type the command to start the server.

```sh
python3 greeter_server.py
```

In another terminal, type the command to establish the connection between the server and the client.

```sh
# you should see
#
# Greeter client received: Hello, you! Your value is 100.
#
python3 greeter_client.py
```

