# Writeup forensics/loud-packets
```
Bài cho một file PCAP và mô tả:

“I was transferring a file with very sensitive info over bluetooth…”

Ý tưởng chính của bài là:

trong PCAP có dữ liệu được truyền qua các packet UDP,
các packet chứa từng mảnh của một file âm thanh,
file âm thanh này nghe như tiếng nhiễu,
nhưng khi xem spectrogram sẽ hiện ra flag.
1. Khảo sát file PCAP

Đầu tiên kiểm tra loại file:

file chall.pcap

Mở bằng Wireshark:

wireshark chall.pcap

Quan sát sẽ thấy rất nhiều packet UDP.

2. Tìm payload đáng ngờ

Trong Wireshark:

chọn một packet UDP,
xem phần Data.

Ta thấy payload bắt đầu bằng:

BTAV

Ví dụ dạng:

42 54 41 56 ...
 B  T  A  V

Đây là dấu hiệu cho thấy dữ liệu được đóng gói theo format riêng.

3. Phân tích cấu trúc packet

Sau khi inspect vài packet, ta nhận ra:

BTAV | sequence_number | data

Cụ thể:

Phần	Kích thước
magic BTAV	4 byte
sequence number	4 byte
chunk data	còn lại

Điều này cho thấy:

file lớn đã bị chia nhỏ thành nhiều packet,
cần ghép lại theo sequence_number.
4. Viết script khôi phục dữ liệu

Tạo script:

import struct

pcap = open("chall.pcap", "rb").read()

off = 24
chunks = []

while off + 16 <= len(pcap):

    # đọc packet header của PCAP
    ts, usec, inc, orig = struct.unpack(
        ">IIII",
        pcap[off:off+16]
    )

    off += 16

    pkt = pcap[off:off+inc]
    off += inc

    # bỏ packet quá ngắn
    if len(pkt) < 42:
        continue

    # chỉ lấy IPv4
    if pkt[12:14] != b"\x08\x00":
        continue

    # chỉ lấy UDP
    if pkt[23] != 17:
        continue

    # tính IP header length
    ihl = (pkt[14] & 0x0f) * 4

    udp_off = 14 + ihl

    # parse UDP header
    sport, dport, ulen, checksum = struct.unpack(
        "!HHHH",
        pkt[udp_off:udp_off+8]
    )

    data = pkt[udp_off+8:udp_off+ulen]

    # packet hợp lệ
    if data.startswith(b"BTAV"):

        seq = struct.unpack(">I", data[4:8])[0]

        chunks.append((seq, data[8:]))

# sắp xếp theo sequence
chunks.sort()

# ghép toàn bộ data
blob = b"".join(x for _, x in chunks)

# tìm file WAV
idx = blob.find(b"RIFF")

wav = blob[idx:]

open("recovered.wav", "wb").write(wav)

print("saved recovered.wav")

Chạy:

python3 solve.py

Kết quả:

saved recovered.wav
5. Kiểm tra file WAV

Kiểm tra:

file recovered.wav

Ta thu được file âm thanh hợp lệ.

Nghe thử sẽ chỉ thấy tiếng nhiễu/âm thanh kỳ lạ.

Đây là dấu hiệu rất quen thuộc của bài steganography âm thanh.

6. Tạo spectrogram

Dùng sox:

sox recovered.wav -n spectrogram -o spec.png

Hoặc dùng Audacity:

mở file WAV,
đổi sang chế độ Spectrogram.
7. Đọc flag

Mở spec.png:

sẽ thấy flag hiện ra dưới dạng hình ảnh trong phổ tần số.

Đây là kỹ thuật:

giấu dữ liệu bằng tần số âm thanh,
mắt thường không nghe được,
nhưng spectrogram sẽ hiển thị.
Kiến thức rút ra

Bài này kết hợp:

phân tích PCAP,
reconstruct stream/file carving,
hiểu packet structure,
audio steganography,
spectrogram analysis.

Các dấu hiệu cần nhớ cho future CTF:

Dấu hiệu	Ý nghĩa
UDP packet có magic bytes	dữ liệu custom
sequence number	cần reorder packet
file WAV nghe như nhiễu	có thể chứa spectrogram flag
spectrogram	kỹ thuật giấu hình trong âm thanh

```
<img width="944" height="628" alt="image" src="https://github.com/user-attachments/assets/79a3f80a-8175-4b6d-83df-5962ac871285" />
