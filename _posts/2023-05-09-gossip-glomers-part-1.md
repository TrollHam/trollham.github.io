When I saw that Fly.io released a [series of distributed system challenges](https://fly.io/dist-sys/), it was only a matter of time before I was going to be sinking my teeth in. The purpose of each challenge is to implement solutions to increasingly complicated problems in a way that is fault-tolerant.

The challenges are:

1. [Echo](./) (You're here!)
2. Unique ID Generation
3. Broadcast
4. Grow-only Counter
5. Kafka-style Log
6. Totally-available Transactions

Over the course of this series, I'm going to be implementing the Maelstrom protocol and the solutions in Rust. Before we can dive into the system, I want to spend a little time talking about the Maelstrom protocol.

# Maelstrom

Maelstrom is a workbench for testing distributed system written by Kyle Kingsbury (you should check out [his blog](https://aphyr.com/)!). It handles the busy work of distributed systems (networking and process management) so you (okay, really me) can focus on the details of the algorithm. You provide Maelstrom a path to an executable to test, and Maelstrom will run the executable runs as a "node" in the Maelstrom network. Each node is expected to conform to the Maelstrom protocol, which is fairly simple: a node consumes incoming JSON-serialized messages from STDIN, and outputs JSON-serialized messages on STDOUT.

Easy! Let's get started with implmenting the basics of the protocol.

## Messages

Taking a look at the [documentation](https://github.com/jepsen-io/maelstrom/blob/main/doc/protocol.md), messages are formatted in an easy to understand way:

```
{
  "src":  A string identifying the node this message came from
  "dest": A string identifying the node this message is to
  "body": An object: the payload of the message
}
```

The fields specified in the body's payload above are the fields Maelstrom recognizes: it can contain any other information as well. This is up to the node's implementation to handle.

```
{
  "type":        (mandatory) A string identifying the type of message this is
  "msg_id":      (optional)  A unique integer identifier
  "in_reply_to": (optional)  For req/response, the msg_id of the request
}
```

Starting with a fresh project, I'm adding `serde` and `serde_json`, since we're going to have to be [de]serializing to and from JSON in our nodes.

```shell
$ cargo add serde serde_json --features serde/derive
    Updating crates.io index
      Adding serde v1.0.162 to dependencies.
             Features:
             + derive
             + serde_derive
             + std
             - alloc
             - rc
             - unstable
      Adding serde_json v1.0.96 to dependencies.
             Features:
             + std
             - alloc
             - arbitrary_precision
             - float_roundtrip
             - indexmap
             - preserve_order
             - raw_value
             - unbounded_depth

```

Let's replicate the Message data structure in Rust.

```rust
use serde::{Deserialize, Serialize};
use serde_json::Value;

#[derive(Debug, Deserialize, Serialize)]
pub struct Message {
    src: String,
    dest: String,
    body: Body,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct Body {
    r#type: String,
    msg_id: Option<u64>,
    in_reply_to: Option<u64>,
    #[serde(flatten)] 
    payload: Option<Value>,
}
```

See, simple! We brought in serde's `derive` feature in order to allow us to derive the implementation of Deserialize and Serialize using procedural macros. Since the body of a message can contain additional fields in the payload, I'm leaving them in JSON as a `serde_json::Value`.

This was so easy, we don't even need a test. But, just to be sure things work _exactly_ how I expect, I'll add two.

```rust
#[test]
fn serialize_test() {
    let msg = Message {
        src: "foo".to_string(),
        dest: "bar".to_string(),
        body: Body {
            r#type: "hello_world".to_string(),
            msg_id: None,
            in_reply_to: None,
            payload: Some(json!({"hello": "world!"})),
        },
    };

    let expected = json!(
    {
        "src": "foo",
        "dest": "bar",
        "body": {
            "type": "hello_world",
            "hello": "world!"
        }
    }
    );

    assert_eq!(serde_json::to_value(&msg).unwrap(), expected)
}

#[test]
fn deserialize_test() {
    let raw_json = json!({
        "src": "foo",
        "dest": "bar",
        "body": {
            "type": "hello_world",
            "hello": "world!",
        }
    });

    let expected = Message {
        src: "foo".to_string(),
        dest: "bar".to_string(),
        body: Body {
            r#type: "hello_world".to_string(),
            msg_id: None,
            in_reply_to: None,
            payload: Some(json!({
                "hello": "world!"
            })),
        },
    };

    let val = serde_json::from_value::<Message>(raw_json).unwrap();

    assert_eq!(val, expected);
}
```

Drum roll, please!

```shell
$ cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.01s
     Running unittests src/lib.rs (target/debug/deps/gossip_glomers-4840b83f04cb7a78)

running 2 tests
test proto::tests::deserialize_test ... ok
test proto::tests::serialize_test ... FAILED

failures:

---- proto::tests::serialize_test stdout ----
thread 'proto::tests::serialize_test' panicked at 'assertion failed: `(left == right)`
  left: `"{\"src\":\"foo\",\"dest\":\"bar\",\"body\":{\"type\":\"hello_world\",\"msg_id\":null,\"in_reply_to\":null,\"hello\":\"world!\"}}"`,
 right: `"{\"body\":{\"hello\":\"world!\",\"type\":\"hello_world\"},\"dest\":\"bar\",\"src\":\"foo\"}"`', src/proto.rs:51:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    proto::tests::serialize_test

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
```

Hmm. Well, deserialization works fine, but it looks like something failed with serialization. On closer inspection, we're serializing the `None` values as `null`.

```console?comments=true
... \"msg_id\":null,\"in_reply_to\":null"`, ...
               👆                   👆
```

An easy fix, we just need to annotate our types to skip the nullable fields.

```rust
#[derive(Debug, Deserialize, Serialize, PartialEq)]
pub struct Body {
    r#type: String,
    #[serde(skip_serializing_if = "Option::is_none")] // 👈
    msg_id: Option<u64>,
    #[serde(skip_serializing_if = "Option::is_none")] // 👈
    in_reply_to: Option<u64>,
    #[serde(flatten)]
    #[serde(skip_serializing_if = "Option::is_none")] // 👈
    payload: Option<Value>,
}
```

And lo, the tests are happy!

```shell
$ cargo test
   Compiling gossip-glomers v0.1.0 
    Finished test [unoptimized + debuginfo] target(s) in 0.23s
     Running unittests src/lib.rs (target/debug/deps/gossip_glomers-4840b83f04cb7a78)

running 2 tests
test proto::tests::serialize_test ... ok
test proto::tests::deserialize_test ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests gossip-glomers

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

This seems like a good enough foundation to get started, let's dive into the Echo challenge.

# Part 1: [Echo](https://fly.io/dist-sys/1/)

First, let's get our project's test framework set up. Since Maelstrom wants to run each node's implementation as a subprocess, I'm going to create a binary for each test. Cargo has a built in way of doing that: the `bin/` directory.

```shell
mkdir src/bin
touch src/bin/echo.rs
```

```rust
use gossip_glomers::proto::Message;

fn main() {
    // TODO: draw the rest of the owl
}
```

Maelstrom is a `stdin/stdout`-based protocol. Let's initialize our input and output channels.

```rust
use std::io;
use gossip_glomers::proto::Message;

fn main() {
    let input = io::stdin().lock();
    let output = io::stdout().lock();
}
```

> Tip: we **could** use `println!` here instead of initializing stdout directly. However, that comes with a cost. Each call to `println!` is going to acquire the lock on stdout, print, and release the lock. This is time consuming, and we already know ahead of time that we're the only writers. It's more efficient to grab the lock now.

Our incoming messages are JSON, so we it would be nice if we could read directly into our Message type, rather than into an intermediate buffer. Luckily, `serde_json` provides a way to deserialize from any source that implements the `Read` trait using [`Deserializer::from_reader`](https://docs.rs/serde_json/latest/serde_json/struct.Deserializer.html#method.from_reader) and [`Deserializer::into_iter`](https://docs.rs/serde_json/latest/serde_json/struct.Deserializer.html#method.into_iter). I'll use those to pull messages coming from stdin, and implement the logic of the echo node in order to finish up the challenge.

```rust
use std::io::{self, Write};

use gossip_glomers::proto::Message;

fn main() {
    let input = io::stdin().lock();
    let mut output = io::stdout().lock();

    let mut input = serde_json::Deserializer::from_reader(input).into_iter::<Message>();

    while let Some(msg) = input.next() {
        match msg {
            Ok(mut msg) => {
                if msg.body.r#type == "echo" {
                    msg.body.r#type = "echo_ok".to_string();

                    msg.body.in_reply_to = msg.body.msg_id;
                    msg.body.msg_id = msg.body.msg_id.map(|i| i + 1);

                    std::mem::swap(&mut msg.src, &mut msg.dest);

                    let serialized_resp = serde_json::to_string(&msg).unwrap();
                    let resp_buf = serialized_resp.as_bytes();
                    output
                        .write_all(resp_buf)
                        .expect("failed to write to stdout");
                }
            }
            Err(e) => eprintln!("unrecognized message: {e:?}"),
        }
    }
}
```

In order to test, we first need to build our binary.

```console?comments=true
$ cargo build --bin echo
   Compiling gossip-glomers v0.1.0 
    Finished dev [unoptimized + debuginfo] target(s) in 0.40s
```

Then, we execute Maelstrom, feeding it the path to the `echo` excutable.

```console?comments=true
$ ~/maelstrom/maelstrom  test -w echo --bin target/debug/echo --node-count 1 --time-limit 10
...
ERROR [2023-05-09 18:10:17,508] main - jepsen.cli Oh jeez, I'm sorry, Jepsen broke. Here's why:
clojure.lang.ExceptionInfo: Expected node n0 to respond to an init message, but node did not respond.
```

> Maelstrom is incredibly verbose, and the `...` here hides a LOT of output. For my sanity and your time, I'm just going to leave it out.

Apparently there ought to have been an init message our node should have responded to. Scanning through the docs, aaaaand... Found it! It was just a little further down in the [protocol documentation](https://github.com/jepsen-io/maelstrom/blob/main/doc/protocol.md#initialization) 😅.

The payload we should be looking for is:

```json
{
  "type":     "init",
  "msg_id":   1,
  "node_id":  "n3",
  "node_ids": ["n1", "n2", "n3"]
}
```

The expected response is simply:

```json
{
  "type": "init_ok",
  "in_reply_to": 1
}
```

Let's handle that.

```rust
...
match msg {
  Ok(mut msg) => {
      let serialized_resp = match msg.body.r#type.as_str() {
          "echo" => {
              msg.body.r#type = "echo_ok".to_string();

              msg.body.in_reply_to = msg.body.msg_id;
              msg.body.msg_id = msg.body.msg_id.map(|i| i + 1);

              std::mem::swap(&mut msg.src, &mut msg.dest);
              serde_json::to_string(&msg).unwrap()
          }
          "init" => {
              msg.body.r#type = "init_ok".to_string();

              msg.body.in_reply_to = msg.body.msg_id;
              msg.body.msg_id = msg.body.msg_id.map(|i| i + 1);

              std::mem::swap(&mut msg.src, &mut msg.dest);
              serde_json::to_string(&msg).unwrap()
          }
          _ => continue,
      };
      let resp_buf = serialized_resp.as_bytes();
      output
          .write_all(resp_buf)
          .expect("failed to write to stdout");
      
...
```

At the moment we're dropping the `node_id` and `node_ids` in the init message on the floor, but for now, let's see if Maelstrom likes our `init_ok` response.

```console?comments=true
$ cargo build --bin echo
   Compiling gossip-glomers v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
$ ~/maelstrom/maelstrom test -w echo --bin target/debug/echo --node-count 1 --time-limit 10  
...
ERROR [2023-05-09 18:23:22,243] main - jepsen.cli Oh jeez, I'm sorry, Jepsen broke. Here's why:
clojure.lang.ExceptionInfo: Expected node n0 to respond to an init message, but node did not respond.
```

Hmm... I figured that was all we needed. Am I missing somethi-
> Both STDIN and STDOUT messages are JSON objects, separated by newlines (\n).

Ah!

```rust
output
    .write_all(resp_buf)
    .expect("failed to write to stdout");
output
    .write(b"\n")
    .expect("Come on, how can you fail here? It's just a newline!");
```

And run it through once more...

```console?comments=true
$ ~/maelstrom/maelstrom test -w echo --bin target/debug/echo --node-count 1 --time-limit 10
...
Everything looks good! ヽ(‘ー`)ノ
```

We did it! If we were curious about the run, we could use `maelstrom serve` to open an interface where we could see analytics for our node, but this blog post is long enough already. Next time, we'll do some cleanup of the code and dig into the protocol a little bit more. No more throwing away node IDs, they're going to be important!

If you want to browse the code, you can find it [here](https://github.com/trollham/gossip-glomers).
