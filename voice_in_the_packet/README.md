```
Writeup — voice-in-the-packet

Challenge:

I intercepted a suspicious phone call over the network. something tells me there's more to this conversation than meets the ear...

1. Recon

Challenge given file .pcap, so the first we need open it with wireshark to analysics

We take a look the file and see: 

the first packet contain fake flag
and the last packet also contain fake flag
In the middle we have many UDP packet, it seem similar each other:

This is very suspicious beacause the name's challenge is:

voice-in-the-packet

=> I think real data will in audio stream:

2. Decide protocol 

We choose one packet in the middle

Right click -> Decode As -> RTP

Wireshark realize this is:

RTP payload type 0

it means:

PCMU / G.711 μ-law audio

And one more thing is: 

source port: 10000
destination port: 20000

=> This is one VoIP call RTP stream.

3. Check stream audio

Use:

Telephony -> RTP -> Stream Analysis

or export audio to .wav.

When we listen the audio, nothing special.

We can deduce

This challenge don't about listen content:
Data can be hide by steganography in audio payload
4. Analysics payload RTP

Write script read RTP payload:
```
import struct

pcap = "call (1).pcap"
data = open(pcap, "rb").read()

o = 24
rtp_audio = []

while o + 16 <= len(data):
    ts_sec, ts_usec, inc_len, orig_len = struct.unpack("<IIII", data[o:o+16])
    o += 16

    pkt = data[o:o+inc_len]
    o += inc_len

    if pkt[0] >> 4 != 4:
        continue

    ihl = (pkt[0] & 0xf) * 4
    proto = pkt[9]

    if proto != 17:
        continue

    src_port, dst_port, udp_len, udp_sum = struct.unpack(
        "!HHHH", pkt[ihl:ihl+8]
    )

    udp_payload = pkt[ihl+8:]

    if src_port == 10000 and dst_port == 20000:
        audio = udp_payload[12:]   # bỏ RTP header
        rtp_audio.append(audio)

print(len(rtp_audio))
```
Output:

1000 RTP packets
5. Find pattern suspicious.

Compare packet cyclical:

for i in range(5):
    a = rtp_audio[i]
    b = rtp_audio[i + 5]

    diffs = []

    for j, (x, y) in enumerate(zip(a, b)):
        if x != y:
            diffs.append((j, x, y, x ^ y))

    print(diffs[:10])

Output:

(120,121, xor=1)
(12,13, xor=1)
(182,183, xor=1)
...

Extremely important point:

Every xor each other will equals 1:

It means:

Only the lowest bit (LSB) is changed:
This is a very typical sign of LSB steganography
6. Observe cycle of packet

Payload RTP not random.

It repeat cyclical:

5 packet sample
5 packet sample
5 packet sample
...

But the lowest bits of certain bytes are flipped.

=> The attacker hid the data by modifying the LSB of the audio samples.

7. Trích xuất bit ẩn

After check location of byte, we realize:

We just need take even bytes:
0,2,4,6,...
Take LSB of our:

Script:

bits = []

payload = b"".join(rtp_audio[:5])

for i in range(0, len(payload), 2):
    bits.append(str(payload[i] & 1))

bitstream = ''.join(bits)

out = []

for i in range(0, len(bitstream), 8):
    byte = bitstream[i:i+8]

    if len(byte) < 8:
        break

    out.append(chr(int(byte, 2)))

print(''.join(out))

Output:

dGpjdGZ7aDN5X3YwaXBfczczZ19pc180XzdoaW5nfQ==

This is Base64.

8. Decode Base64
echo 'dGpjdGZ7aDN5X3YwaXBfczczZ19pc180XzdoaW5nfQ==' | base64 -d

Output:

tjctf{h3y_v0ip_s73g_is_4_7hing}

```
