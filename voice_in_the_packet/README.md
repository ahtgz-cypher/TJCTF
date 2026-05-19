```
Writeup — voice-in-the-packet

Challenge:

I intercepted a suspicious phone call over the network. something tells me there's more to this conversation than meets the ear...

1. Recon

File được cho là một .pcap, nên bước đầu tiên là mở bằng Wireshark.

Quan sát nhanh thấy:

packet đầu chứa một fake flag
packet cuối cũng chứa fake flag
ở giữa là hàng loạt UDP packet trông gần như giống hệt nhau

Điều này khá đáng nghi vì challenge tên là:

voice-in-the-packet

=> nhiều khả năng dữ liệu thật nằm trong audio stream.

2. Xác định giao thức

Chọn một packet ở giữa:

Right click -> Decode As -> RTP

Wireshark nhận ra đây là:

RTP payload type 0

tức:

PCMU / G.711 μ-law audio

Ngoài ra:

source port: 10000
destination port: 20000

=> đây là một VoIP call RTP stream.

3. Kiểm tra stream audio

Dùng:

Telephony -> RTP -> Stream Analysis

hoặc export audio ra .wav.

Khi nghe audio thì không có gì đặc biệt.

Điều này gợi ý rằng:

challenge không yêu cầu nghe nội dung
dữ liệu có thể được giấu bằng steganography trong audio payload
4. Phân tích payload RTP

Viết script đọc RTP payload:

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

Kết quả:

1000 RTP packets
5. Tìm pattern bất thường

So sánh packet theo chu kỳ:

for i in range(5):
    a = rtp_audio[i]
    b = rtp_audio[i + 5]

    diffs = []

    for j, (x, y) in enumerate(zip(a, b)):
        if x != y:
            diffs.append((j, x, y, x ^ y))

    print(diffs[:10])

Kết quả:

(120,121, xor=1)
(12,13, xor=1)
(182,183, xor=1)
...

Điểm cực kỳ quan trọng:

mọi khác biệt đều XOR = 1

Nghĩa là:

chỉ có bit thấp nhất (LSB) bị thay đổi
đây là dấu hiệu rất điển hình của LSB steganography
6. Quan sát chu kỳ packet

Payload RTP không random.

Nó lặp lại theo chu kỳ:

5 packet mẫu
5 packet mẫu
5 packet mẫu
...

Nhưng các bit thấp nhất của những byte nhất định bị flip.

=> attacker đã giấu dữ liệu bằng cách sửa LSB của audio samples.

7. Trích xuất bit ẩn

Sau khi kiểm tra nhiều vị trí byte, nhận ra:

chỉ cần lấy các byte chẵn:
0,2,4,6,...
lấy LSB của chúng

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

Kết quả:

dGpjdGZ7aDN5X3YwaXBfczczZ19pc180XzdoaW5nfQ==

Đây là Base64.

8. Decode Base64
echo 'dGpjdGZ7aDN5X3YwaXBfczczZ19pc180XzdoaW5nfQ==' | base64 -d

Kết quả:

tjctf{h3y_v0ip_s73g_is_4_7hing}
Flag
tjctf{h3y_v0ip_s73g_is_4_7hing}
```
