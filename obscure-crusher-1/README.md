# obscure-crusher-1 Writeup
```
Challenge: obscure-crusher-1
Category: Forensics
Points: 244

Description

What if I make it so you need 3 keys to unlock the flag...

Ta được cung cấp một file duy nhất:

chall.bin
1. Initial Recon

Đầu tiên kiểm tra loại file:

file chall.bin

Output:

chall.bin: Mac OS X icon, 256 bytes, "icns" type

File được nhận diện là file icon của macOS (icns), tuy nhiên dung lượng chỉ khoảng 169 bytes nên khá đáng nghi.

Tiếp tục kiểm tra bằng strings:

strings chall.bin

Output:

icns
icns
name
lzmaKLZMA_DATA:
@s+E

Ở đây xuất hiện nhiều chuỗi đáng chú ý:

icns
name
lzma
KLZMA_DATA:

Đề bài nói cần “3 keys”, nên rất có thể các chuỗi này liên quan đến key.

2. Hex Analysis

Tiếp tục xem raw bytes bằng xxd:

xxd chall.bin

Output quan trọng:

00000040: 0000 0000 0000 006e 616d
00000050: 6500 0874 7466 0278 7900
...
00000070: 8000 6c7a 6d61 4b4c 5a4d 415f 4441 5441
00000080: 3a1d 090d 0767 ...

Chuyển sang ASCII:

6e 61 6d 65 = name
74 74 66    = ttf
78 79       = xy
6c 7a 6d 61 = lzma
4b           = K

Ta nhận thấy file chứa nhiều chuỗi ngắn có vẻ giống các “key fragments”:

icns
ttf
xy
lzma
K

Ngoài ra còn có 2 byte đặc biệt:

01
02

nằm xen giữa các chuỗi.

3. False Lead: LZMA

Do xuất hiện chuỗi:

LZMA_DATA:

nên ban đầu mình nghĩ file chứa compressed stream LZMA.

Thử extract và decompress:

xz --format=lzma -dc data.lzma

nhưng nhận được:

Compressed data is corrupt

Sau nhiều lần sửa header LZMA vẫn không thể giải nén được.

Điều này cho thấy chữ lzma chỉ là mồi nhử (red herring), không phải compressed data thật.

4. Identifying the Cipher

Sau marker:

LZMA_DATA:

là một đoạn bytes ngắn:

1d 09 0d 07 67 ...

Độ dài ciphertext rất nhỏ, phù hợp với kiểu mã hóa XOR lặp key.

Ta thử XOR bằng một vài chuỗi trong file như icns, name, lzma:

pt = bytes(c ^ key[i % len(key)] for i, c in enumerate(ct))

Kết quả với key icns:

tjct ... ush3 ... zm3}

Đã bắt đầu lộ format flag.

5. Recovering the Full Key

Quan sát cấu trúc file, ta nhận ra key thực chất được ghép từ nhiều thành phần trong file:

icns
01
ttf
02
xy
lzma
K

Tạo key:

key = b"icns\x01ttf\x02xylzmaK"
6. Final Solve Script
data = open("chall.bin","rb").read()

marker = b"LZMA_DATA:"
ct = data.split(marker)[1]

key = b"icns\x01ttf\x02xylzmaK"

pt = bytes(c ^ key[i % len(key)] for i, c in enumerate(ct))

print(pt.decode())

Output:

tjctf{0bscur3_crush3r_1cns_ttf_lzm3}
Flag
tjctf{0bscur3_crush3r_1cns_ttf_lzm3}
```
