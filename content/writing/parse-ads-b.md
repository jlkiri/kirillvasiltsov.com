---
title: "Planespotting with Rust: using nom to parse ADS-B messages"
date: "2023-09-02"
---

Using *software defined radio* (SDR) to listen to [ADS-B](https://mode-s.org/decode/content/ads-b/1-basics.html) transmissions has been on my mind for a while. This article is inspired by the [cool experiment by Charlie Gerard](https://charliegerard.dev/blog/aircraft-radar-system-rtl-sdr-web-usb/), who used JavaScript to parse and display ADS-B messages as they come through the WebUSB interface right in the browser. I wanted to do something similar, but use my go-to parsing library [nom](https://github.com/rust-bakery/nom) and the terminal.

## ADS-B

ADS-B is a protocol used by aircrafts to broadcast their position, altitude, speed, and other information. Nowadays, the majority of aircrafts broadcast ADS-B messages constantly. Anyone with the right equipment can listen to these messages. You can buy a relatively cheap USB dongle with an antenna on Amazon and install drivers for it on Linux. In my case I used [usbipd-win](https://github.com/dorssel/usbipd-win) to mount the USB device inside Ubuntu running in WSL2. Then I installed the Linux drivers and [dump1090](https://github.com/flightaware/dump1090), a program that makes use of these drivers and then outputs ADS-B messages in a format that is easy to parse. While you can use `dump1090` to display a neat table full of information about aircrafts, I wanted to use its raw output capabilities to parse ADS-B messages myself. It starts a simple TCP server that outputs raw ADS-B messages wrapped in Mode-S Beast *frames*. I'm not sure what `Beast` means, but I found something that looks like its spec [here](https://github.com/firestuff/adsb-tools/blob/master/protocols/beast.md).

The [ADS-B specification](https://mode-s.org/decode/content/ads-b/1-basics.html) is pretty easy to read. Basically every message is 112 bits long and has the following structure:

```
+----------+----------+-------------+------------------------+-----------+
|  DF (5)  |  CA (3)  |  ICAO (24)  |         ME (56)        |  PI (24)  |
+----------+----------+-------------+------------------------+-----------+
```

The DF (downlink format) field can be used to figure out whether this is an ADS-B message. This time I only care about DF=17 or DF=18 messages, the only messages in ADS-B format. CA (capability) is kind of boring: capabilities of the hardware that broadcasts the message. ICAO (International Civil Aviation Organization) 24-bit address or (informally) Mode-S "hex code" is a unique identifier of the aircraft that normally never changes. ME (message extension) is the actual message. The contents of this field depend on its first 5 bits, the [typecode](https://mode-s.org/decode/content/ads-b/1-basics.html#ads-b-message-types). For example, aircrafts identify themselves and send their altitude information in messages with different typecodes. This time I want to at least recognize aircrafts' callsigns (TC 1-4) and their altitude (TC 9-18, 20-22).

## nom

Just in case you are not familiar with [nom](https://github.com/rust-bakery/nom), it is a `parser combinator` written in Rust. The most basic thing you can do with it is import one of its parsing functions, give it some byte or string input and then get a `Result` as output with the parsed value and the rest of the input or an error if the parser failed. [`tag`](https://docs.rs/nom/latest/nom/bytes/complete/fn.tag.html) for example is used to recognize literal character/byte sequences.

```rust
use nom::bytes::complete::tag;

fn parser(s: &str) -> IResult<&str, &str> {
  tag("Hello")(s)
}

assert_eq!(parser("Hello, World!"), Ok((", World!", "Hello")));
```

`nom` also has a bunch of [`combinators`](https://docs.rs/nom/latest/nom/combinator/index.html), helpers that allow us to apply many parsers in a sequence, or apply parsers conditionally etc.

This time I am going to parse byte input (`&[u8]`), because this is what we get from a raw TCP server started by `dump1090`. Parsing bytes is as easy as parsing strings:

```rust
use nom::bytes::complete::tag;

fn parser(s: &[u8]) -> IResult<&[u8], &[u8]> {
  tag([0x01, 0x02, 0x03])(s)
}
```

## Parsing ADS-B messages

Parsing ADS-B messages is basically a matter of recognizing some bytes, then taking some more bytes and interpreting them as either a number or a string. The only tricky part is that some parts of the message are 5 or 3 bits long. Normally you would take a byte and shift some bits but fortunately `nom` has nice helpers for bits as well. First, I need to parse the Mode-S Beast frame header I mention above. I keep it very simple because I just want to discard it and read the information I'm interested in.

```rust
fn mode_s_beast_header(input: &[u8]) -> IResult<&[u8], &[u8], ()> {
    let header = tuple((tag([0x1a]), tag([0x33]), take(6u8), take(1u8)));
    recognize(header)(input)
}
```

All frames start with `0x1a`. The next byte, `0x33` represents Mode S messages (apparently there are more modes) and since I'm only interested in Mode S messages I expect `0x33` and just error if the message is not Mode S. Next is the MLAT timestamp. Honestly, I don't know what it's for but I know that it's always 6 bytes long so I just take the next 6 bytes. Finally there is a single byte RSSI (received signal strength indicator). I'm not interested in this information either so I just take another byte. Then the `recognize` combinator is used to return the successfully parsed input as is.

The following bytes are the actual ADS-B message. First I parse DF and CA fields. DF is 5 bits long and CA is 3 bits long. I use `bits` module of `nom` to parse the two and return them as a tuple.

```rust
fn df_ca(input: (&[u8], usize)) -> IResult<(&[u8], usize), (u8, u8), ()> {
    use nom::bits::complete::take;

    let ((input, offset), df) = take(5u8)(input)?;
    let ((input, offset), ca) = take(3u8)((input, offset))?;
    assert!(offset == 0);
    Ok(((input, offset), (df, ca)))
}
```