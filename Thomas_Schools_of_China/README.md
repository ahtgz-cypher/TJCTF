# Writeup – Thomas Schools of China
```
Challenge cho file lạ:

chall.tsc

Các tool cơ bản đều không nhận diện được:

file chall.tsc
exiftool chall.tsc
binwalk chall.tsc

Kết quả chỉ thấy magic bytes:

TSCF
1. Phân tích hex của file

Dump vài byte đầu:

xxd -g 1 -l 256 chall.tsc

Kết quả:

54 53 43 46 01 00 00 00 3c 00 00 00 3d 00 00 05
39 db e9 cc 00 db e9 cc 00 db e9 cc ...

Ta thấy:

54 53 43 46 = TSCF
Sau header là pattern 4 byte lặp đi lặp lại:
00 db e9 cc

Entropy của file cũng rất thấp → file không bị mã hóa.

2. Thử render dữ liệu thành ảnh

Do dữ liệu lặp theo cụm 4 byte nên mình nghi đây là raw RGBA.

Tạo script:

from PIL import Image

data = open("chall.tsc", "rb").read()

body = data[16:]

w = 60
h = len(body) // (4 * w)

body = body[:w * h * 4]

img = Image.frombytes("RGBA", (w, h), body)

img.save("out.png")

Chạy:

python3 solve.py

Ta nhận được một ảnh con vịt màu cyan.

Ban đầu tưởng flag nằm trong stego hoặc alpha channel, nhưng kiểm tra:

zsteg out.png
binwalk out.png
strings out.png

không cho kết quả hữu ích.

3. Nhận ra điều bất thường

Quan sát kỹ dump hex thấy xuất hiện các cụm giống ASCII:

0074 6a63
0074 667b
0063 306e
0067 7234

Chuyển sang ASCII:

tjc
tf{
c0n
gr4

Ghép lại:

tjctf{c0ngr4...

=> Flag đang bị giấu trực tiếp trong dữ liệu pixel.

4. Trích xuất ASCII từ các pixel

Mỗi pixel gồm 4 byte:

00 XX YY ZZ

Byte đầu thường là 00, còn 3 byte sau chứa ký tự ASCII.

Script extract:

import re

data = open("chall.tsc","rb").read()
body = data[16:]

chunks = []

for i in range(0, len(body)-3, 4):
    s = bytes(body[i+1:i+4]).decode(errors="ignore")

    # bỏ AAA, BBB, kkk...
    if len(s) == 3 and s[0] == s[1] == s[2]:
        continue

    if re.fullmatch(r"[a-z0-9_{}!]+", s):
        chunks.append(s)

text = ''.join(chunks)

start = text.find("tjctf{")
end = text.find("}", start)

print(text[start:end+1])

Chạy:

python3 extract.py

Kết quả:

tjctf{c0ngr4ts_u_s0lv3d_my_f1st_CTF_chall!_btw_1_l1ke_b1rds}
Flag
tjctf{c0ngr4ts_u_s0lv3d_my_f1st_CTF_chall!_btw_1_l1ke_b1rds}
```
