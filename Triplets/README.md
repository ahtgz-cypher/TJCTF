# Writeup Triplets
```
Ý tưởng bài này là:

Ảnh RGB gốc đã bị “trải phẳng” (flatten) thành grayscale bằng cách lấy từng bộ 3 byte màu (R,G,B) biến thành 3 pixel xám liên tiếp.

Tên challenge là triplets → hint rằng dữ liệu phải xử lý theo nhóm 3 byte.

1. Quan sát ban đầu

Ảnh chall.png nhìn như nhiễu ngang:

nhiều sọc ngang,
có pattern lặp,
không giống ảnh bị encrypt hoàn toàn.

Điều này thường gợi ý:

dữ liệu ảnh vẫn còn,
nhưng layout pixel bị sai.
2. Dùng binwalk

Bạn chạy:

binwalk -e chall.png

và thấy:

FC
FC.zlib

Điều này khiến ta nghĩ:

có stream zlib bị nhúng trong PNG.

Nhưng sau đó:

zlib.decompress(data)

bị lỗi:

invalid literal/lengths set

=> nghĩa là FC.zlib không phải stream hoàn chỉnh.

3. Phân tích PNG đúng cách

PNG không chứa 1 stream zlib riêng lẻ.

PNG có nhiều chunk:

IHDR
IDAT
IDAT
IDAT
...
IEND

Toàn bộ dữ liệu ảnh thật nằm trong các chunk IDAT.

Ta viết script:

import struct
import zlib

png = open("chall.png","rb").read()

pos = 8
idat = b""

while pos < len(png):
    length = struct.unpack(">I", png[pos:pos+4])[0]
    typ = png[pos+4:pos+8]

    if typ == b"IDAT":
        idat += png[pos+8:pos+8+length]

    pos += 12 + length

raw = zlib.decompress(idat)
4. Kích thước raw cực kỳ quan trọng

Ta thu được:

raw size: 3566432

Tính:

1888 * (1888 + 1)

ra đúng:

3566432

Điều đó chứng minh:

ảnh là grayscale,
mỗi dòng PNG có:
1 byte filter + 1888 byte pixel
5. Hint quyết định: metadata

Bạn chạy:

strings chall.png

và thấy:

2000x594.y

Đây gần như chắc chắn là:

width = 2000
height = 594

tức là kích thước ảnh gốc.

6. Ý tưởng thật sự của bài

Ảnh grayscale hiện tại thực chất là:

R G B R G B R G B ...

mỗi byte trở thành 1 pixel xám.

Muốn phục hồi ảnh gốc:

lấy 3 pixel xám liên tiếp,
ghép thành 1 pixel RGB.
7. Recover ảnh

Code cuối cùng:

from PIL import Image
import numpy as np

img = Image.open("chall.png").convert("L")
data = np.array(img).flatten()

W = 2000
H = 594

need = W * H * 3

chunk = data[:need]

rgb = chunk.reshape(H, W, 3)

Image.fromarray(rgb).save("flag.png")
8. Tại sao phải thử offset/permutation

Vì có thể:

dữ liệu lệch 1–2 byte,
hoặc channel order là:
RGB
BGR
GRB
...

nên ta brute-force:

for off in range(20):

và:

for p in permutations([0,1,2]):
9. Bản chất forensics của bài

Bài này là:

Data reinterpretation

Không phải:

mã hóa,
stego LSB,
hay corruption.

Mà là:

Dữ liệu ảnh bị diễn giải sai format.

Ảnh RGB:

[R,G,B][R,G,B][R,G,B]

bị xem nhầm thành:

gray gray gray gray ...

nên tạo ra các sọc ngang kỳ lạ.

10. Dấu hiệu để nhận ra kiểu bài này

Khi gặp ảnh:

nhiều horizontal lines,
pattern lặp,
gradient vẫn “có nghĩa”,
không random hoàn toàn,

hãy nghĩ đến:

pixel packing sai,
width sai,
RGB/BGR swap,
channel interleave,
triplets/stride mismatch,
raw image reconstruction.
```
<img width="2000" height="594" alt="image" src="https://github.com/user-attachments/assets/8b12187d-edcb-4013-af3f-6309a99d3921" />
