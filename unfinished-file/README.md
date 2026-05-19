# Unfinished File — Writeup
```
Challenge: forensics/unfinished-file
Author: jonathan

Đề bài

Ta được cung cấp một file:

secret_archive.zip.crdownload

Đây là phần mở rộng thường xuất hiện khi Chrome tải file chưa hoàn tất.

Mô tả bài:

my stupid friend tried downloading this file before i shut my laptop down, what was he trying to do?

Ý tưởng bài cho thấy ta cần khôi phục hoặc phân tích file tải dở này để tìm flag.

1. Kiểm tra file ban đầu

Dùng file:

file secret_archive.zip.crdownload

Kết quả:

secret_archive.zip.crdownload: data

File không còn nhận dạng được là ZIP hoàn chỉnh.

Tiếp tục dùng exiftool:

exiftool secret_archive.zip.crdownload

Kết quả:

Error : Unknown file type

Điều này xác nhận file đã bị hỏng hoặc chưa tải xong.

2. Dùng strings để tìm dấu vết

Trong forensics, khi gặp file lạ hoặc file corrupt, strings luôn là công cụ rất hữu ích.

strings secret_archive.zip.crdownload

Output:

CRDL
https://example.com/secret_archive.zip
xmB6(!6$9,q4q0
r6*'0
2qr2.'
6r7!*
!r/276'0?
readme.txt
This file is incomplete. Keep looking...
hidden/.flagdata
...
PK
3. Phân tích output

Ta thấy nhiều chi tiết đáng chú ý:

CRDL

Đây là magic thường thấy trong file .crdownload.

URL gốc
https://example.com/secret_archive.zip

Xác nhận ban đầu file thật sự là ZIP.

Tên file bên trong ZIP
readme.txt
hidden/.flagdata

Rõ ràng file ZIP từng chứa dữ liệu bí mật.

Đoạn dữ liệu lạ
6(!6$9,q4q0
r6*'0
2qr2.'
6r7!*
!r/276'0?

Đây là phần quan trọng nhất.

Chuỗi này:

gồm ký tự printable,
nhìn không ngẫu nhiên,
có vẻ bị mã hóa nhẹ.

Trong các challenge forensics dễ/trung bình, dạng này thường là:

XOR 1 byte,
Caesar,
ROT,
hoặc substitution đơn giản.
4. Thử brute-force XOR 1 byte

Ta brute force toàn bộ 256 key XOR và tìm chuỗi giống flag.

Script:

python3 - << 'EOF'
data = open("secret_archive.zip.crdownload", "rb").read()

for key in range(256):
    decoded = bytes(b ^ key for b in data)

    if b"tjctf{" in decoded.lower():
        print("KEY =", hex(key))

        start = decoded.lower().find(b"tjctf{")
        end = decoded.find(b"}", start)

        print(decoded[start:end+1].decode())
EOF

Kết quả:

KEY = 0x42
tjctf{n3v3r_0ther_p30ple_t0uch_c0mputer}
5. Vì sao XOR hoạt động?

Ta có thể kiểm tra thủ công.

Ký tự đầu tiên của chuỗi lạ là:

6

Nếu XOR với 0x42:

ord('6') ^ 0x42 = ord('t')

Ký tự tiếp theo:

ord('(') ^ 0x42 = ord('j')

Tiếp tục sẽ tạo thành:

tjctf{...}

Nghĩa là dữ liệu trong file đã bị XOR bằng key 0x42.

6. Kết luận

File .crdownload thực chất chứa:

metadata của file tải dở,
một phần nội dung ZIP,
và dữ liệu flag đã bị XOR 1 byte.

Sau khi brute-force XOR, ta thu được flag.

Flag
tjctf{n3v3r_0ther_p30ple_t0uch_c0mputer}
```
