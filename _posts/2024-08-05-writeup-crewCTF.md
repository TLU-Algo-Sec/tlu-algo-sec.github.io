---
layout: post
title:  "Writeup crewCTF 2024"
tags: [writeup,forensics,pcap,memory]
authors: [4]
image: /assets/challenge files/crewCTF/crewCTF-logo.png
---

Khá lâu mình không thêm writeup lên trang của câu lạc bộ. Không để phí công sức gây dựng của anh Thảo nên mình đã viết lại bài này. Giải này mình chơi 1 mình và chỉ làm được 2 bài Forensics. Còn giải chính thì mình viết bên blog của mình mất rồi hehe.Cảm ơn mọi người đã đọc. 

## 1. Recursion
- Challenge: <br>
    ![pic](/assets/challenge%20files/crewCTF/Recursion/chall.png)

Thử thách cung cấp cho ta 1 file pcap. Đúng như tên của nó, mở lên ta sẽ thấy các luồng dữ liệu USB :
    ![pic](/assets/challenge%20files/crewCTF/Recursion/usbdata.png)

Sau một hồi lướt qua, mình để ý thấy 1 số luồng có dữ liệu như này: 
    ![pic](/assets/challenge%20files/crewCTF/Recursion/layer4.png)

Để trích xuất dữ liệu thô ra, mình dùng `tshark`:
    ![pic](/assets/challenge%20files/crewCTF/Recursion/tshark.png)

Có khá nhiều dòng dữ liệu nhưng ta chỉ cần decode dòng khác biệt lớn nhất,bắt đầu bằng `1f8b08083b19116602...`, ta có thể thấy khi dùng Cyberchef:
    ![pic](/assets/challenge%20files/crewCTF/Recursion/cyberchef.png)

Save file và unzip nó ra, mình nhận được 1 file pcap nữa,file layer4.pcap. Làm tương tự để extract dữ liệu từ file pcap, ta sẽ nhận lần lượt các file 7z,tar,zip và các file layer3,2,1. Khi trích xuất dữ liệu từ file layer1 ra, mình khá bối rối vì nó không thuộc loại nào cả. Sau một lúc suy nghĩ thì mình thử grep và nhận được cờ: 
    ![pic](/assets/challenge%20files/crewCTF/Recursion/flag.png)

FLAG: *crew{l00ks_l1ke_y0u_mad3_1t!}*

## 2. Crymem
Đúng như cái tên, làm bài này mà mình chắc khóc mất. Thử thách cho 1 file memdump.raw và 1 file aes_sample.c. 
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/mman.h>
#include <sys/random.h>
#include <openssl/evp.h>


size_t file_read_with_delete(char* filename, unsigned int maxdatasize, uint8_t* readdata) {
    int fd;
    struct stat stbuf;
    size_t file_size;
    long page_size, map_size;
    char* map;
    char c = 0;

    if (!filename) {
        perror("filename is invalid\n");
        return -1;
    }

    // open file
    fd = open(filename, O_RDONLY);
    if (fd < 0) {
        perror("Error opening file for read\n");
        return -1;
    }

    // obtain file size
    if (fstat(fd, &stbuf) == -1) {
        perror("fstat error\n");
        return -1;
    }

    file_size = (size_t)(stbuf.st_size);

    if ((unsigned int)file_size > maxdatasize) {
        perror("filesize is too large\n");
        return -1;
    }

    // mmap file
    page_size = getpagesize();
    map_size = (file_size / page_size + 1) * page_size;

    if ((map = (char*)mmap(NULL, map_size, PROT_READ, MAP_SHARED, fd, 0)) == MAP_FAILED) {
        perror("mmap error\n");
        return -1;
    }

    // read file to writedata
    memcpy(readdata, map, file_size);

    close(fd);
    munmap(map, map_size);

    // open file
    fd = open(filename, O_RDWR);
    if (fd < 0) {
        perror("Error opening file for write\n");
        return -1;
    }

    // mmap file
    if ((map = (char*)mmap(NULL, map_size, PROT_WRITE, MAP_SHARED, fd, 0)) == MAP_FAILED) {
        perror("mmap error\n");
        return -1;
    }

    // null overwrite
    memset(map, 0, file_size);
    msync(map, map_size, 0);

    close(fd);
    munmap(map, map_size);

    return file_size;
}


int dummy_file_clear() {
    #define DUMMYINSIZE (2097152)

    int fd;
    struct stat stbuf;
    size_t file_size;
    long page_size, map_size;
    char* map;
    char dummy_in[DUMMYINSIZE];

    // open file
    fd = open("/lib/x86_64-linux-gnu/libc.a", O_RDONLY);
    if (fd < 0) {
        perror("Error opening file for read\n");
        return -1;
    }

    // obtain file size
    if (fstat(fd, &stbuf) == -1) {
        perror("fstat error\n");
        return -1;
    }

    file_size = (size_t)(stbuf.st_size);

    // mmap file
    page_size = getpagesize();
    map_size = (file_size / page_size + 1) * page_size;

    if ((map = (char*)mmap(NULL, map_size, PROT_READ, MAP_SHARED, fd, 0)) == MAP_FAILED) {
        perror("mmap error\n");
        return -1;
    }

    // read file to writedata
    memcpy(dummy_in, map, DUMMYINSIZE);

    close(fd);
    munmap(map, map_size);

    return 0;
}


int encrypt(uint8_t* out, const uint8_t* in, const uint8_t* key, const uint8_t* iv, int* out_len, int in_len) {
    EVP_CIPHER_CTX *ctx;

    if (!(ctx = EVP_CIPHER_CTX_new())) {
        perror("Error creating EVP_CIPHER_CTX\n");
        return -1;
    }

    if (1 != EVP_EncryptInit_ex(ctx, EVP_aes_128_cbc(), NULL, key, iv)) {
        perror("Error initializing encryption\n");
        return -1;
    }

    if (1 != EVP_EncryptUpdate(ctx, out, out_len, in, in_len)) {
        perror("Error during encryption\n");
        return -1;
    }

    //EVP_CIPHER_CTX_free(ctx);

    return 0;
}


int main() {
    #define KEYSIZE (16)
    #define IVSIZE (16)
    #define MAXPLAINTEXTSIZE (64)
    #define MAXCIPHERTEXTSIZE (64)

    uint8_t key[KEYSIZE] = {0};
    uint8_t iv[IVSIZE] = {0};
    uint8_t in[MAXPLAINTEXTSIZE] = {0};
    uint8_t out[MAXCIPHERTEXTSIZE] = {0};
    int plaintextsize, ciphertextsize;
    int i;

    uint8_t dummy_out[MAXCIPHERTEXTSIZE] = {0};

    if (getrandom(iv, IVSIZE, 0) != IVSIZE) {
        perror("getrandom error\n");
        return -1;
    }

    if (file_read_with_delete("tmp/key.txt", KEYSIZE, key) != KEYSIZE) {
        perror("cannot read key.txt\n");
        return -1;
    }

    if ((plaintextsize = (int)file_read_with_delete("tmp/plaintext.txt", MAXPLAINTEXTSIZE-1, in)) == -1) {
        perror("cannot read plaintext.txt\n");
        return -1;
    }
    in[plaintextsize] = '\0';

    if (encrypt(out, in, key, iv, &ciphertextsize, plaintextsize) != 0) {
        perror("Error at encrypt\n");
        return -1;
    }

    // clear contexts
    memset(in, 0, MAXPLAINTEXTSIZE);
    dummy_file_clear();

    printf("IVVALUE:");
    for (i = 0; i < IVSIZE; i++) {
        printf("%02x", iv[i]);
    }
    printf("\n");

    printf("ENCFLAG:");
    for (i = 0; i < ciphertextsize; i++) {
        printf("%02x", out[i]);
    }
    printf("\n");

    return 0;
}
```

 Mình nghĩ như mọi lần dùng volatility3 thì dễ dàng quá, thế nó cho file C làm gì. Đọc qua file C mình thấy nó là mã hóa AES. Code sẽ in ra 2 giá trị là IVVALUE và ENCFLAG. Thầm cười vì nghĩ dễ ăn nhưng đời không như mơ, vol3 không dùng được cho file raw này. Mình thử tìm các bài viết khác về memfor, thử mount, thử extract bằng binwalk và dùng isoinnfo, thử strings xong ngồi lọc lọc lọc... nhưng vẫn chỉ có thể nhận được 2 giá trị là ENCFLAG và IVVALUE, còn KEY thì không tìm ra. 
    ![pic](/assets/challenge%20files/crewCTF/crymem/IVandENC.png)

 Tình cờ ngồi xem lại link mấy cái tool thì mình thấy thằng [AESkeyfind](https://www.kali.org/tools/aeskeyfind/) này. Mình thử down và chạy thì ra key luôn. 
 ```
 ➜  dist $ aeskeyfind -v memdump.raw
FOUND POSSIBLE 128-BIT KEY AT BYTE fddf900

KEY: 11f9f5aafad8e57c0d14b2e1b52d83d6
```

Cộng với IVVALUE và ENCFLAG mà ta strings được, ta có thể giải mã nó bằng Cyberchef : 
    ![pic](/assets/challenge%20files/crewCTF/crymem/flag.png)

FLAG: *crew{M3m0ry_f0r3N_is_mysterious_@_crypt0_Challs}*

# Lời nói cuối 
Mình chỉ mới tập chơi Forensics và còn nhiều thiếu xót. Mong được mọi người góp ý, bổ sung để mình phát triển hơn. Mình xin cảm ơn.
