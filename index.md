---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
title: IncludeOS - Run your application with zero overhead
---

IncludeOS allows you to run your application in the cloud without an operating system. IncludeOS adds operating system functionality to your application allowing you to create performant, secure and resource efficient virtual machines.

IncludeOS applications boot in tens of milliseconds and require only a few megabytes of disk and memory.


[[View on Github]](https://github.com/includeos/IncludeOS)
[[Chat on Slack]](https://join.slack.com/t/includeos/shared_invite/zt-5z7ts29z-_AX0kZNiUNE7eIMUP60GmQ)
[[Tell me more]](technology.html)


To run a service with IncludeOS on Linux or macOS you do not need to install IncludeOS, however you need to install a few dependencies depending on the service you will run. You can start by trying out our simplest hello_world service. For this service you will need the following dependencies.

 * Conan package manager
 * cmake, make, nasm
 * clang, or alternatively gcc on linux. Prebuilt packages are available for clang 6.0 and gcc 7.3
 * qemu
 * python3 packages: psutil, jsonschema

With the above dependencies you should be able to build an application within minutes.

```sh
$ conan config install https://github.com/includeos/conan_config.git
$ git clone https://github.com/includeos/hello_world.git
$ mkdir build_hello_world
$ cd build_hello_world
$ conan install ../hello_world -pr 
$ source ./activate.sh
$ cmake ../hello_world
$ cmake --build .
$ boot hello
```        

The hello world booted service should look like this:

```
================================================================================
 IncludeOS 0.14.1-1093 (x86_64 / 64-bit)
 +--> Running [ Hello world - OS included ]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Hello world
    [ main ] returned with status 0
```    

For detailed instructions see the [GitHub README](https://github.com/includeos/IncludeOS/blob/master/README.md). Once installed we suggest looking at and booting a few of the demo-examples to familarize yourself with the system.

We strive to make it easy to create fast and useful services. The below code will set up a simple TCP echo service and happily talk to anyone connecting.

```cpp
#include <os>
#include <iostream>
#include <net/interfaces>

void Service::start()
{
  // Get the IP stack thats already been automatically configured
  auto& inet = net::Interfaces::get(0);
  // Setup a TCP echo server on port 7 (echo port)
  auto& server = inet.tcp().listen(7);

  server.on_connect([] (auto conn) {
    // Log incomming connections on the console:
    std::cout << "Connection " << conn->to_string() << " established\n";
    // When data is received, echo back
    conn->on_read(1024, [conn] (auto buf) {
      conn->write(buf);
    });
  });
}
```

The network configuration of the virtual machine can reside in a JSON file, named config.json, placed in the same folder. It should look something like this, depending on your need:
```json
{
  "net" : [
    {
      "iface": 0,
      "config": "dhcp-with-fallback",
      "address": "10.0.0.42",
      "netmask": "255.255.255.0",
      "gateway": "10.0.0.1"
    }
  ]
}
```

## Security

For security related inquires please send email to security@includeos.org. You can use our [PGP key](https://pgp.mit.edu/pks/lookup?search=security@includeos.org&op=index) to encrypt the email.

This project is maintained by Alfred Bratterud.
