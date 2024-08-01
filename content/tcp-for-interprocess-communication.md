+++

title = "Tcp for Interprocess Communication"
description = "Sample article showcasing syntax highlighting and formatting for Code Blocks with a custom theme."
date = 2024-08-01
draft = false
+++

## Features of TCP 
1. connection-oriented protocol (must establish connection before data transmission)
2. end-to-end communication
3. reliable
4. full duplex
5. data is transmitted as bytes stream
6. the transmitted data has no message boundary (consistent with 4)

## Connection-Oriented 
todo: how to accept tcp connection, how to handle stream 

## End-to-End 
todo: can't boardcast 

## Reliable 
todo: the reason network buffer may become full 

## Data has No Memory Boundary
todo: application layer protocol


## Read/Write Data Packet via TCP Stream
TCP是连续的字节流，分割成单独的请求是应用层协议的事情。所以，直接读取TCP Stream的内容，可能刚好是一个完整的请求，可能只有一个请求的前一部分，也可能有一个半的请求等等的情况。

一般在应用层定义一个固定大小的Request Header，其中包含一个field指明Data的长度。

当Sender发送Request到Receiver时，Sender显然可以一次性把Request中的header和data写入Stream中。

当Receiver接收来自Sender的Request时，Receiver先从Stream中读取固定大小的RequestHeader.

从header中的data_len field得知data的长度，再继续从Stream中读取已知大小的data.

## Blocking/Non-Blocking


## Code Snippet
- start a TCP listener to accept exactly one TCP connection 
- a TCP Server as data receiver and a TCP Client as data sender
- define application layer protocol
  - define packet header, packet data format
  - implement header encode manually
  - implement header decode via `From<&[u8]>` trait 
  - implement data encode via `serde::Serialize`
  - implement data decode via `serde::Deserialize`
- [TODO] show TCP `write_all` operation will be stuck in blocking mode if network buffer was full
```rust 
use core::time;
use std::{
    io::{Read, Write},
    net::{Shutdown, TcpListener, TcpStream},
    thread,
};
use anyhow::{Ok, Result};
use serde;
use serde_json;

// I do not use serde to serialize and deserialize RequestHeader, because json format is not fixed sized.
// I manually encode and decode it.
// use bincode crate may be also a good choice.
#[derive(Debug)]
struct RequestHeader {
    data_len: u64,
}

// RequestData can be complex, use serde to make it simple
#[derive(Debug, serde::Serialize, serde::Deserialize)]
struct RequestData {
    data: Vec<u8>,
}

impl RequestData {
    // len in bytes
    pub fn len(&self) -> u64 {
        self.data.len() as u64
    }
}
// This struct may never be used directly.
// Because I encode and send header first,
// and then serialize and deserialize data.
#[allow(dead_code)]
struct Request {
    header: RequestHeader,
    data: RequestData,
}

// impl From trait so that I can build RequestHeader from &[u8]
impl From<&[u8]> for RequestHeader {
    fn from(slice: &[u8]) -> Self {
        let (int_bytes, _rest) = slice.split_at(std::mem::size_of::<u64>());
        Self {
            data_len: u64::from_be_bytes(int_bytes.try_into().unwrap()),
        }
    }
}

impl RequestHeader {
    // impl this so that I can extract &[u8] from RequestHeader,
    // slice is a mutable input here, it can't fit From trait's fn signature
    pub fn to_u8_slice(&self, slice: &mut [u8]) {
        slice.copy_from_slice(&self.data_len.to_be_bytes())
    }

    // getter of data_len
    pub fn data_len(&self) -> u64 {
        self.data_len
    }
}

const LOCAL_ADDRESS: &str = "127.0.0.1:12344";

// server as Receiver
fn tcp_server() -> Result<()> {
    let tcp_listener = TcpListener::bind(LOCAL_ADDRESS)?;
    println!("bind to {:?}", LOCAL_ADDRESS);

    let tcp_stream_join_handle = thread::spawn(move || {
        let (mut tcp_stream, socket_address) = tcp_listener.accept()?;
        println!("accept socket address: {:?}", socket_address);
        loop {
            // pre-allocate memory space for header
            let mut header_buffer: Vec<u8> = vec![0; std::mem::size_of::<RequestHeader>()];
            // read header data from stream
            tcp_stream.read_exact(header_buffer.as_mut())?;
            // build header from &[u8]
            let header = RequestHeader::from(header_buffer.as_ref());
            println!("read header: {:?}", header);

            // pre-allocate memory space for data
            let mut data_buffer: Vec<u8> = vec![0; header.data_len() as usize];
            // read data from stream
            tcp_stream.read_exact(data_buffer.as_mut()).unwrap();
            // deserialize Vec<u8> to data structure via serde
            let data: RequestData = serde_json::from_slice(data_buffer.as_ref()).unwrap();
            println!("read data: {:?}", data);
        }
        #[allow(unreachable_code)]
        Ok(())
    });

    //let result = tcp_stream_join_handle.join().unwrap();

    Ok(())
}

// client as sender
fn tcp_client() -> Result<()> {
    let mut tcp_stream = TcpStream::connect(LOCAL_ADDRESS)?;
    println!("connect to {:?}", LOCAL_ADDRESS);
    let tcp_stream_join_handle = thread::spawn(move || {
        for i in 0..10 {
            let data: RequestData = RequestData {
                data: vec![i + 1, i + 2, i + 3, i + 4],
            };
            // serialize data to vec u8 via serde
            let data_buffer = serde_json::to_vec(&data)?;

            // header store data_len in bytes
            let header = RequestHeader {
                data_len: data_buffer.len() as u64,
            };
            // declare buffer of header
            let mut header_buffer: Vec<u8> = vec![0; std::mem::size_of::<RequestHeader>()];
            // encode header into buffer
            header.to_u8_slice(header_buffer.as_mut());
            tcp_stream.write_all(&header_buffer)?;
            println!("send header buffer: {:?}", header_buffer);
            tcp_stream.write_all(&data_buffer)?;
            println!("send data buffer: {:?}", data_buffer);

            tcp_stream.flush()?;
        }
        tcp_stream.shutdown(Shutdown::Both)?;
        Ok(())
    });

    Ok(())
}

fn main() -> Result<()> {
    tcp_server()?;
    tcp_client()?;
    // NOTE: leave some time to let threads run
    thread::sleep(time::Duration::from_secs(10));
    Ok(())
}

```
```
```
