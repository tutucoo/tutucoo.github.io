---
title: 算法@CRC32算法逆向与还原
date: 2022-02-11 14:42:37
tags:
---


# CRC32算法逆向与还原

## 算法的识别

经过CRC32加密的密文通常是4个字节长度，可以通过IDA逆向确定它是CRC32算法，方法是找到算法中的常量，通过google搜索会出现CRC32算法相关的搜索结果从而确定它是CRC32算法，下图中的常量0xEDB88320，经过google搜索发现它是CRC32算法 

![Untitled](https://s2.loli.net/2022/02/11/rWyojAlTKO9qLIu.png)

## 算法逆向

逆向目标so，找到CRC32核心算法，CRC32核心算法非常简短，对于这一类算法直接拷贝到clion，进行运行，报错的地方进行修正，直接完成算法还原工具

```c
#include <iostream>
int is_init = 0;
int crc32_table[256] = {0};
void crc_init()
{
    unsigned int i; // r0
    int v1; // r3
    unsigned int v2; // r4
    unsigned int v3; // r5

    for ( i = 0; i != 256; ++i )
    {
        v1 = 8;
        v2 = i;
        while ( v1 )
        {
            v3 = (v2 >> 1) ^ 0xEDB88310;
            if ( !(v2 << 31) )
                v3 = v2 >> 1;
            --v1;
            v2 = v3;
        }
        crc32_table[i] = v2;
    }
    is_init = 1;
}

unsigned int crc32(int a1, unsigned char *a2, unsigned int a3) {
    unsigned char *v5; // r5
    unsigned int v6; // r1
    unsigned int v7; // r1
    unsigned int v8; // r1
    unsigned int v9; // r1
    unsigned int v10; // r1
    unsigned int v11; // r1
    unsigned int v12; // r1
    int v13; // r2
    int v14; // r3
    unsigned int v15; // r1
    int v16; // r2
    int v17; // r3

    if (!a2)
        return 0;
    v5 = a2;
    if (!is_init)

        crc_init();

    v6 = ~a1;
    while (a3 >= 8) {
        a3 -= 8;
        v7 = crc32_table[*v5 ^ (unsigned char)v6] ^(v6 >> 8);
        v8 = crc32_table[(unsigned char)v7 ^ v5[1]] ^(v7 >> 8);
        v9 = crc32_table[(unsigned char)v8 ^ v5[2]] ^(v8 >> 8);
        v10 = crc32_table[(unsigned char)v9 ^ v5[3]] ^(v9 >> 8);
        v11 = crc32_table[(unsigned char)v10 ^ v5[4]] ^(v10 >> 8);
        v12 = crc32_table[(unsigned char)v11 ^ v5[5]] ^ (v11 >> 8);
        v13 = (unsigned char)v12 ^ v5[6];
        v14 = v5[7];
        v5 += 8;
        v15 = crc32_table[v13] ^ (v12 >> 8);
        v6 = crc32_table[(unsigned char )v15 ^ v14] ^ (v15 >> 8);
    }
    if (a3) {
        v16 = 0;
        do {
            v17 = v5[v16++];
            v6 = crc32_table
                 [v17 ^ (unsigned char ) v6] ^ (v6 >> 8);
        } while (a3 != v16);
    }
    return ~v6;
}

int main() {
    std::string str = "kZPpcuZDGquGjAljnCVaq";
    unsigned int ret = crc32(0, (unsigned char *) str.data(), str.size());
    printf("%08x",ret);
    return 0;
}
```

![Untitled](https://s2.loli.net/2022/02/11/oBETJudfrpChAnH.png)