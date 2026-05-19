# Writeup – Invisible Ink
```
Challenge cho một file PDF:

chall.pdf

Mô tả:

I wonder if there's anything hidden inside!

=> hướng đầu tiên là kiểm tra xem PDF có nhúng file ẩn hay không.

1. Phân tích PDF

Dùng binwalk:

binwalk chall.pdf

Kết quả:

0             PDF document
31464         Zip archive data, encrypted
name: original_distorted.png

PDF có nhúng một file ZIP mã hóa chứa:

original_distorted.png
2. Trích xuất ZIP ẩn

Extract toàn bộ:

binwalk -e chall.pdf

hoặc:

dd if=chall.pdf of=hidden.zip bs=1 skip=31464

Kiểm tra:

unzip hidden.zip

ZIP yêu cầu password.

3. Tìm password trong PDF

Ban đầu thử:

zip2john hidden.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

nhưng không crack được.

Vì challenge nói về PDF nên khả năng cao password nằm ngay trong file PDF.

Tôi thử bôi đen file và đầu tiên nhận được dòng chữ: thing here lol why are you looking

Và sau đó tôi thử qua trang tiếp theo và bôi đen. Thật may, tôi nhận được chuỗi có vẻ giống password

Ok fine here’s the password: DBf8nEBgwRhZ

Đó chính là password ZIP.

4. Giải nén ảnh
unzip -P 'DBf8nEBgwRhZ' hidden.zip

Thu được:

original_distorted.png
5. Phân tích ảnh

Ảnh nhìn giống một đống nét đỏ bị xoáy mạnh.

Dùng:

file original_distorted.png
exiftool original_distorted.png

không thấy metadata đặc biệt.

Kiểm tra stego:

zsteg original_distorted.png
strings original_distorted.png

không ra flag.

Quan sát bằng mắt thấy ảnh giống bị áp dụng hiệu ứng:

swirl / whirl distortion
6. Khôi phục ảnh bằng unswirl

Mở bằng GIMP:

gimp original_distorted.png

Vào:

Filters → Distorts → Whirl and Pinch

Thử các góc xoay âm:

-90
-180
-270
-360

và điều chỉnh tới khi chữ hiện lại bình thường.

Có thể dùng Python:

from PIL import Image
import matplotlib.pyplot as plt
from skimage.transform import swirl
import numpy as np

img = np.array(Image.open("original_distorted.png"))

out = swirl(
    img,
    strength=-8,
    radius=700,
    rotation=0
)

plt.imshow(out)
plt.axis("off")
plt.show()

Sau khi unswirl sẽ đọc được flag.

Ý tưởng challenge

Challenge gồm 2 lớp:

PDF chứa ZIP ẩn
Ảnh bị swirl distortion để che flag

Tên bài:

Invisible Ink

ám chỉ thông tin bị “ẩn trong tài liệu”, không phải steganography phức tạp.

Kỹ năng sử dụng
binwalk
strings
zip2john
phân tích PDF container
image distortion recovery
GIMP Whirl & Pinch filter
```
