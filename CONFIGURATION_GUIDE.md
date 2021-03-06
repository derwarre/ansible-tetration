
# CONFIGURATION_GUIDE.md

## Network Policy Publisher 
The Tetration Network Policy Publisher uses a Kafka message broker for publishing network policy, which are seralized using Google ProtoBuffers.

### Kafka Python

The Kafka Python package was selected rather than the Confluent Kafka package. Confluent does not provide a means to disable certification checking. The Tetration cluster installed in the Advanced Technology Center (ATC) uses a self-signed certificate. In order to authenticate and bypass the certificate checking, the program creates an SSL context with CERT_NONE, no certificates from the server are required (or looked at if provided)) and provide the context to Kafka Python.

The Kafka Python [documentation](https://media.readthedocs.org/pdf/kafka-python/master/kafka-python.pdf) is used as reference.

The KafaConsumer method requires Snappy for decompression. Python-snappy requires the Snappy C library. Installing `libsnappy-dev`, `kafka-python`, and `python-snappy` is required to execute the program.

### Installing Required Packages

The development environment uses apt as the package manager, the following 'apt' and 'pip' packages and SDKs will need be installed to used this software.

```YAML
- name: Installs packages for my development environment
  hosts: localhost

  vars:
    packages:
      apt:
          - python-pip
          - python-dev
          - git
          - unzip                           # unzip utility
          - libsnappy-dev                   # install Snappy C library for python-snappy
      pip:
          - tetpyclient                    # Tetration 1.0.7 
          - ipaddress                      # manipulates IP addresses
          - pyopenssl                      # ACI modules when using certificates for authentication 18.0.0
          - kafka-python                   # Kafka for Tetration 1.4.3
          - python-snappy                  # KafkaConsumer Snappy decompression (requires libsnappy-dev)  0.5.3
          - pydevd                         # Remote debugger for PyCharm 1.4.0

 
```   

### Protocol Buffers
The Tetration Kafka broker transports the Network Policy as a Google Protocol Buffer. To use Protocol Buffers in Python, you need three things:

* Tetration .proto definition file
* Python Protocol Buffer library
* Protocol buffer compiler

The .proto definition file `tetration_network_policy.proto` is used by the compiler to generate a Python file `tetration_network_policy_pb2.py` and imported by the program.

#### Definition file
Source code for the TetrationNetworkPolicyProto definition file is available from the Tetration appliance GUI, or can be downloaded from the tetration-exchange repo:

https://github.com/tetration-exchange/pol-client-java/blob/master/proto/network_enforcement/tetration_network_policy.proto

This definition file is compiled by the protocol buffer compiler. The output generated is comprised of Python classes and methods to decode the policy generated by Tetration.

The compiled protocol buffer is provided in `./library`.

#### Python Library

Refer to the Google Python tutorial, at  [https://developers.google.com/protocol-buffers/docs/pythontutorial](https://developers.google.com/protocol-buffers/docs/pythontutorial]), download the Python library.
```
/usr/share$ sudo mkdir protobufs
$ cd protobufs
$ sudo wget https://github.com/protocolbuffers/protobuf/releases/download/v3.6.1/protobuf-python-3.6.1.tar.gz
$ sudo gunzip protobuf-python-3.6.1.tar.gz
$ sudo tar xvf protobuf-python-3.6.1.tar
$ sudo rm protobuf-python-3.6.1.tar
```

#### Compiler Install instructions for Protocobuf3

Install the precompiled binary version of the protocol buffer compiler (protoc).

These [instructions](https://gist.github.com/rvegas/e312cb81bbb0b22285bc6238216b709b) are a useful reference.

Download and unzip the [file](https://github.com/protocolbuffers/protobuf/releases/download/v3.6.1/protoc-3.6.1-linux-x86_64.zip). Given a work directory at `~/protobufs` . 

```
$ cd /usr/share/protobufs
$ sudo wget https://github.com/protocolbuffers/protobuf/releases/download/v3.6.1/protoc-3.6.1-linux-x86_64.zip
$ sudo unzip protoc-3.6.1-linux-x86_64.zip -d protoc3
```
###### Move protoc to /usr/local/bin/
```
sudo mv protoc3/bin/* /usr/local/bin/
```
###### Move protoc3/include to /usr/local/include/
```
sudo mv protoc3/include/* /usr/local/include/
```
The only file remaining in `~/protobufs/protoc3` is the readme.txt. The work directory can be removed if desired.

###### Optional: change owner
```
sudo chown $USER /usr/local/bin/protoc
sudo chown -R $USER /usr/local/include/google
```
###### Verify the version
```
~/protobufs$ which protoc
/usr/local/bin/protoc
administrator@flint:~/protobufs$ protoc --version
libprotoc 3.6.1
```

#### Optional Compile the .proto file
The Google tutorial syntax for compilation of the .proto file is:
```
protoc -I=$SRC_DIR --python_out=$DST_DIR $SRC_DIR/addressbook.proto
```
Assuming the .proto file is `/files/tetration_network_policy.proto`, write the output to the same directory as the source code `/library/`, invoke the compiler:
```
protoc -I=/ansible-tetration/files/ --python_out=/ansible-tetration/library/ /ansible-tetration/files/tetration_network_policy.proto
```
View, but do not edit, the output file.

#### Ansible Configuration file

The Ansible configuration file, `ansible.cfg` should specify a location to look for the `tetration_network_policy.py` module and the protocol buffer module, `tetration_network_policy_pb2.py`.

Modify the `ansible.cfg` file to include:
```
library        = /usr/share/ansible/
module_utils   = /usr/share/ansible/module_utils/
```
and copy (or move) `tetration_network_policy.py` to the directory specified as the `library` and  `tetration_network_policy_pb2.py` to the directory specified by the `module_utils`.

#### Imports

Ansible module:

```
tetration_network_policy.py
    import ansible.module_utils.network.tetration.tetration_network_policy_pb2     
        tetration_network_policy_pb2.py
            from google.protobuf import ...   

```

| **Diretory**                     | **Source**   | **Target Description**       |
|----------------------------------|--------------|-------------------------------|
| /home/administrator/protobufs/   | https://github.com/protocolbuffers/protobuf/releases/download/v3.6.1/protobuf-python-3.6.1.tar.gz | unzip / tar |
| /home/administrator/protobufs/protobuf-3.6.1/python/google | n/a | target of symlink |
| /usr/lib/python2.7/dist-packages | symlink      |  google -> /home/administrator/protobufs/protobuf-3.6.1/python/google       |           
| /usr/lib/python2.7/dist-packages/ansible/module_utils/network  | symlink      |  tetration -> /usr/share/ansible/module_utils/network/tetration  |
| /usr/share/ansible               | ansible.cfg  | library        = /usr/share/ansible/ |
| /usr/share/ansible/tetration_network_policy.py | cp | tetration_network_policy.py from {{playbook_dir}}/library |
| /usr/share/ansible/module_utils  | ansible.cfg  | module_utils = /usr/share/ansible/module_utils/ |
| /usr/share/ansible/module_utils/network/tetration | cp |  tetration_network_policy_pb2.py from {{playbook_dir}}/library |

  

### Authenticating with the Kafka Broker

THIS SECTION INTENTIONALLY LEFT BLANK

### References, Tips and examples

This section includes tips for using the protocol buffer methods. There are very few tutorials exist on using Python and protocol buffers.

Review the `.proto` file, for example:

```
// CatchAll policy action.
message CatchAllPolicy {
  enum Action {
    INVALID = 0;
    // Allow the corresponding flows.
    ALLOW = 1;
    // DROP the corresponding flows.
    DROP = 2;
  };
  Action action = 1;
}
```
Derive a meaningful name to the value of catch_all.action with:

```python
import tetration_network_policy_pb2 

    ### assume the buffer is stored in policy.buffer 
    tnp = policy.buffer.tenant_network_policy
    for item in tnp.network_policy:
        print ('catch_all', tetration_network_policy_pb2.CatchAllPolicy.Action.Name(item.catch_all.action))

('catch_all', 'ALLOW')        
```
IPAddressFamily example:
```
enum IPAddressFamily {
  INVALID = 0;
  IPv4 = 1;
  IPv6 = 2;
};
```
Use the Name method to decode the values provided.
```
>>> import tetration_network_policy_pb2 as tnp
>>> tnp.IPAddressFamily.Name(0)
'INVALID'
>>> tnp.IPAddressFamily.Name(1)
'IPv4'
>>> tnp.IPAddressFamily.Name(2)
'IPv6'
>>> try:
...     tnp.IPAddressFamily.Name(3)
... except ValueError as e:
...     print e
...
Enum IPAddressFamily has no name defined for value 3
>>>
```
#### Print help

You can use the interactive Python interpreter to print the help from the compiled .proto file:

```python
import tetration_network_policy_pb2

import pydoc
help = pydoc.render_doc(tetration_network_policy_pb2, "Help on %s")
f = open("/tmp/tetration_network_policy_pb2.txt", 'w+')
print >>f, help
f.close()
quit()
```
Review the output file.
```bash
cat /tmp/tetration_network_policy_pb2.txt
```

#### Examples

Example code in the `tetration-exchange` repo:

* [Policy consumer client implementation reference written in Go](https://github.com/tetration-exchange/pol-client-go)
* [Tetration Network Policy Enforcement Client](https://github.com/tetration-exchange/pol-client-java)

The development environment is running Ubuntu 16.04.05
```
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.5 LTS
Release:        16.04
Codename:       xenial
```
## Author
joel.king@wwt.com GitHub / GitLab @joelwking 