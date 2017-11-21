# Input Validation

These are the requirements for input validation as defined by Intel's 1.0
release process:

1. Ensure that all input meets specifications.

2. Validate all input to ensure that it is in the expected form, does not
   contain unwanted garbage or executable characters to ensure that an overflow,
   improper access or other unexpected behavior cannot occur as a result of
   consuming the input.

3. Proper handling of invalid input: Use exception handling or other
   programmatic methods to ensure overflow or unexpected behavior is handled
   properly.

4. Verify that unit tests exercise positive and negative input validation,
   document if coming up short

The first step to meeting these requirements is to define what is meant by
"input". We will define "input" to be network traffic received by the validator
after a validator has been deployed. We will specifically exclude CLI arguments,
configuration files, and environment variables from consideration as well as
input in components other than the validator.

## Validation Layers

Because input to the validator is over a network endpoint, validation must occur
in at several layers. These layers of validation are:

1. TCP/IP Protocol
2. ZMQ Protocol
3. Protocol Buffer Message Specification
4. Business Logic

### TCP/IP and ZMQ

The TCP/IP and ZMQ protocols provide an initial layer of input validation.
TCP/IP ensure that clients are formatting packets correctly and ZMQ ensures
input conforms to the message data structures defined by ZMQ. Validation
requirements 1-4 are handled automatically at these layers and these layers do
not require any additional effort by the project team.

### Protocol Buffers

The Protocol Buffer Message Specification (Protobuf) layer defines the structure
of data sent over the first two layers. Using Protobuf, Sawtooth defines a
2-layer message structure for all messages sent to and from the validator. The
first layer is a simple `Message` structure containing a message type, a
correlation id, and the serialized second layer structure.

    Message {
        MessageType message_type = 1;
        string correlation_id = 2;
        bytes content = 3;
    }

The specification for the second layer structure is determined by reading the
message type and looking up the specification from a map between message types
and the second layer message specification.

Some basic input validation is provided "for free" when deserializing this
2-layer message structure, since Protobuf messages are strongly typed and
improperly formatted or typed data will cause a parsing error. *Parsing errors
should be caught by the validator and converted to a response to the sender.*
Failure to catch parsing errors may cause the validator to break or senders to
hang waiting for a response.

### Business Logic

The final layer of input validation is to *enforce application-defined rules on
the fields defined in each Protobuf message*. This includes rules such as:

- Batch and Transaction IDs must be 128 character hex strings
- Block IDs must be 128 character hex strings or the string "0000000000000000"
- Global State Addresses must be 70 character hex strings
- State Roots must be 64 character hex strings
- Public keys must 66 character hex strings
- ...

In addition to validating Protobuf message fields defined, it is important to
*validate that messages received on a given endpoint are allowed on that
endpoint*. The validator creates at least two separate endpoints, one for
network messages and one for local "component" messages. For example, a client
message received on a network endpoint would be invalid input.

## Status of Input Validation

Currently, the existence and quality of input validation in layers 3 and 4 is
mixed.
