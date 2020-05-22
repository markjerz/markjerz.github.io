---
title: Implementing circuit relay in PeerTalk
---

# Introduction

Decentralising communication is very much in vogue at the moment. A move away from centralised (and corporate) control of data and privacy is being explored and built by various organisations. Even Sir Tim Berners-Lee is pushing ahead with a project, [Solid](https://solid.mit.edu/), attempting to "radically change the way Web Applications work".

One of the bigger, and more ambitious, attempts at building a decentralised internet is [IPFS](https://ipfs.io/), the Interplanetary File System. It's a "peer to peer hypermedia protocol, designed to make the web faster, safer and more open." It's basically, and this might be dumbing it down, a more open version of bittorrent. It stores files across computers and uses a distributed hash table to find content across the network. While IPFS is the distributed file system, it is built on top of [libp2p](https://libp2p.io/) which is "a modular network stack" that provides the peer to peer networking and protocols that underpin IPFS.

There are various applications built on top of IPFS and there are additionally multiple implementations of the specifications. At the time of writing, the supported implementations are in go, javascript and rust. Having spent the majority of my time working in .Net I obviously went hunting for a .Net port and found [IPFS Engine](https://github.com/richardschneider/net-ipfs-engine), written by Richard Schneider and targing .Net Standard 2.0.

Richard has done a great job of putting this together. It says that [PeerTalk](https://github.com/richardschneider/peer-talk) is "a collection of peer-to-peer protocols in the spirit of libp2p" but seeing as it appears to interoperate flawlessly I'm not sure why it's "in the spirity of". 

# Implementing Circuit Relay

Having attempted to use IPFS Engine to start building a new application I noticed that I couldn't connect some of my machines together. It turned out that the connections were unable to traverse the NAT that my router provides. libp2p mitigates against this with an experimental [circuit relay](https://docs.libp2p.io/concepts/circuit-relay/) capability that operates a little like [TURN](https://tools.ietf.org/html/rfc5766), by proxying connections through other computers. Essentially, if I know that my computer is behind an impenetrable NAT, I open a connection to a known relay and tell people that they can connect to me through that relay.

In order to implement circuit the nodes must implement the relay protocol, which is a simple message sending contract to indicate that traffic should be forwarded on. It acts both as a protocol and as a transport layer as you then use the existing connections to make a "virtual" connection.

In PeerTalk, this was seemingly quite easy to achieve, as everything had been built on top of .Net Streams so the connections were unaware of the nature of the Streams over which they were communicating anyway.

Unfortunately, some of the existing stream functions in PeerTalk had been designed with a constant and known message format which meant that they did not obey the `Read` and `ReadAsync` contracts around the `count` parameter. This says "The maximum number of bytes to read". Instead, the implementations in PeerTalk had been written to always return that number of bytes, which makes sense for a bunch of strict protocols that you control but does not work at all when you're proxying another encrypted connection through your stream and therefore have no idea how big the messages are.

It's been a while since I've had to properly understand a codebase that I've not written at all so, needless to say, it took me a while to figure this out. It was also being done in two different places. Here's a before and after of the ReadAsync implementation in the Secio1Stream class (which implements encryption over the connection):

## Before

```
public override async Task<int> ReadAsync(byte[] buffer, int offset, int count, CancellationToken cancellationToken)
{
    int total = 0;
    while (count > 0)
    {
        // Does the current packet have some unread data?
        if (inBlock != null && inBlockOffset < inBlock.Length)
        {
            var n = Math.Min(inBlock.Length - inBlockOffset, count);
            Array.Copy(inBlock, inBlockOffset, buffer, offset, n);
            total += n;
            count -= n;
            offset += n;
            inBlockOffset += n;
        }
        // Otherwise, wait for a new block of data.
        else
        {
            inBlock = await ReadPacketAsync(cancellationToken);
            inBlockOffset = 0;
        }
    }

    return total;
}
```

## After

```
public override async Task<int> ReadAsync(byte[] buffer, int offset, int count, CancellationToken cancellationToken)
{
    // reads whole packets (of encrypted data) at a time
    // buffers them and returns the correct number of bytes
    if (inBlock == null) {
        // we don't have an existing packet
        inBlock = await ReadPacketAsync(cancellationToken);
        inBlockOffset = 0;
    }

    var bytesRead = Math.Min(inBlock.Length - inBlockOffset, count);
    Array.Copy(inBlock, inBlockOffset, buffer, offset, bytesRead);
    inBlockOffset += bytesRead;
    if (inBlockOffset >= inBlock.Length)             {
        inBlock = null; // fetch a new packet on next invocation
    }

    return bytesRead;
}
```

The interesting thing here is that due to the nature of the encryption you must read entire packets from the underlying stream at once but then you can't provide that entire packet back to the caller if they've asked for less than the size.

What the previous code did though, was also, only provide a response when it had enough bytes to do so. So, if there was no more data, on the proxied connection, you would get stuck waiting for another block to arrive.

As a side note, it turns out that testing networking protocols is quite difficult when they've been encrypted twice and are all running on the same test suite concurrently. The following library [Nerdbank.Streams](https://github.com/AArnott/Nerdbank.Streams) provides some excellent specialised .Net stream classes to help. In particular, there is a [FullDuplexStream](https://github.com/AArnott/Nerdbank.Streams/blob/master/doc/FullDuplexStream.md) class which provides a pair of bidirectional streams for in-proc two-way communication. These were very useful for mocking out NetworkStreams.

# The relay itself

The actual coding of the relay protocol was fairly simple due it's inherent simplicity. There's a bunch of sending messages back and forth using protocol buffers in order to perform the relay handshake.

Once that's done we then pipe the two streams together, which at this point are multiplexed substreams over the top of the underlying network streams. This looks broadly like:

```
if (stopResponse.IsSuccess())
{
    log.Trace($"Received success from STOP request");
    await SendRelayMessageAsync(CircuitRelayMessage.NewStatusResponse(Status.SUCCESS), srcStream, cancel);
    log.Trace($"Piping the streams together");
    var srcToDst = PipeAsync(srcStream, dstStream);
    var dstToSrc = PipeAsync(dstStream, srcStream);
    await srcToDst;
    await dstToSrc;
}

async Task PipeAsync(Stream inStream, Stream outStream)
{
    while (!cancel.IsCancellationRequested)
    {
        var buffer = new byte[1024];
        var bytesRead = await inStream.ReadAsync(buffer, 0, 1024, cancellationToken: cancel);
        log.Trace($"Received message from peer");
        if (bytesRead > 0)
        {
            await outStream.WriteAsync(buffer, 0, bytesRead, cancel);
            await outStream.FlushAsync(cancel);
        }
    }
}
```

The key thing that tripped me up here, was that I now had multiplexed streams on top of multiplexed streams and both of them attempting to read messages on the same stream. 

I updated the connection handler to recognise when a connection was being instantiated over an existing multiplexed connection so that it could inform the connection to stop processing requests on that substream, as the content was being handled by the `PipeAsync` instead. This, incidentally, is why the streams only returning exactly `count` bytes was problematic: I was buffering up to 1024 bytes at a time across the connections.