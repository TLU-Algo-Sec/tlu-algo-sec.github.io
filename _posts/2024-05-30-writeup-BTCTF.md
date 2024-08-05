---
layout: post
title:  "Writeup BTCTF"
tags: [writeup,forensics,python,crypto,web]
authors: [4]
image: https://ctftime.org/media/team/bt_code_club_pfp_2.png
---

## 1. Scrambled 

- [Source](/assets/challenge%20files/BTCTF/Forensics/scrambled/scrambled.png)

- Tools : ```hexedit,bless``` 
- Đề bài cho 1 file có đuôi là .png nhưng mình mở lên thì không được. Mình dùng lệnh file để check thử thì kết quả trả về là data. Từ đề bài và kiểm tra ban đầu, dễ thấy đây là dạng bài ```corrupted file```. Dạng bài này sẽ yêu cầu sửa các file bị hỏng để nhận cờ. 
- Bắt tay vào làm, mình dùng hexedit để kiểm tra header của file : 
![](/assets/challenge%20files/BTCTF/Forensics/scrambled/hex-bit.png)
- Và mình nhận ra là cái header này mình chưa thấy bao giờ luôn. Gì chưa biết thì ta tìm. Mang theo suy nghĩ đó thì mình tìm kiếm file signature và kết quả là không tìm thấy loại file đó. Đến đây mình định bỏ ngang thì tự nhiên 4 bit mình tìm kiếm (mình thử đảo lại ) ```4E 44``` lại nhảy vào IEND của file PNG . Để hiểu rõ hơn về PNG cũng như IEND, mình có để lại [bài viết này](https://en.wikipedia.org/wiki/PNG) và [bài viết này](https://stackoverflow.com/questions/30550346/understanding-image-its-hex-values). Mình xem lại header của file ban đầu và nhận ra nó hoàn toàn trùng khớp với IEND, chỉ là bị đảo lại. Mình chắc chắn hơn khi mình lướt đến cuối của file :
![](/assets/challenge%20files/BTCTF/Forensics/scrambled/tail-bit.png)
- Để giải quyết chỉ cần đảo lại thứ tự bit là xong : 

```python
def reverse_hex_file(input_file, output_file):
    # Mở file đầu vào dưới dạng nhị phân
    with open(input_file, 'rb') as f:
        data = f.read()

    # Chuyển đổi nội dung file thành danh sách các byte
    data_bytes = list(data)

    # Đảo ngược thứ tự các byte
    reversed_data_bytes = data_bytes[::-1]

    # Ghi lại các byte đã đảo ngược vào file mới
    with open(output_file, 'wb') as f:
        f.write(bytes(reversed_data_bytes))

# Sử dụng hàm với file cụ thể
input_file = 'scrambled.png'
output_file = 'out.png'
reverse_hex_file(input_file, output_file)
```

- Sau đó ta nhận được cờ : 

![Smile](/assets/challenge%20files/BTCTF/Forensics/scrambled/out.png)

Flag : 
*btctf{How_DID_u_unsCraMBl3_tHI5_EGg}*

## 2. No end in sight

- [Source](/assets/challenge%20files/BTCTF/Forensics/No%20end%20in%20sight/layer1.zip) 

- File thử thách cho là 1 file zip. Well, có vẻ dễ ăn. Mình unzip nó ra,lại 1 file zip nữa. Thử thêm lần nữa, vẫn là 1 file zip. Có vẻ bài cho 1 số lượng file zip cực lớn, nên mình dùng code cho nhanh : 

```python 
import zipfile
import os

def unzip_nested_zip(zip_file_path, output_dir):
    with zipfile.ZipFile(zip_file_path, 'r') as zip_ref:
        zip_ref.extractall(output_dir)
        extracted_files = zip_ref.namelist()
    
    for file_name in extracted_files:
        file_path = os.path.join(output_dir, file_name)
        if zipfile.is_zipfile(file_path):
            unzip_nested_zip(file_path, output_dir)
            os.remove(file_path)

# Đường dẫn đến file zip ban đầu
initial_zip_path = 'layer1.zip'
# Thư mục để giải nén các file
output_directory = 'extr'

# Giải nén file zip ban đầu
unzip_nested_zip(initial_zip_path, output_directory)
```

- Hoặc 1 cách nhanh hơn mà mình biết sau khi giải kết thúc : ```strings layer1.zip | grep "btctf"``` :v

Flag : *btctf{i_hope_you_didnt_do_this_by_hand}*