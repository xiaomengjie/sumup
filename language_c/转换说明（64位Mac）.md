##### 有符号整数

| int（-2^31 - 2^31 - 1） | short（-32768 - 32767） | long（-2^63 - 2^63 - 1） | long long（-2^63 - 2^63 - 1） |
| ----------------------- | ----------------------- | ------------------------ | ----------------------------- |
| %d %o %x                | %hd %ho %hx             | %ld %lo %lx              | %lld %llo %llx                |

##### 无符号整数

| unsigned int（0 - 2^32 - 1） | unsigned short（0 - 65536） | unsigned long（0 - 2^64 - 1） | unsigned long long（0 - 2^64 - 1） |
| ---------------------------- | --------------------------- | ----------------------------- | ---------------------------------- |
| %u %o %x                     | %hu %ho %hx                 | %lu %lo %lx                   | %llu %llo %llx                     |

##### 浮点数

| float                           | double                          |
| ------------------------------- | ------------------------------- |
| %f（10进制计数） %e（指数计数） | %f（10进制计数） %e（指数计数） |

```c
#include <stdio.h>
#include <limits.h>

int main(){
    printf("十进制==================================\n");
    printf("int max = %d\n", INT_MAX);
    printf("int min = %d\n", INT_MIN);
    printf("short max = %hd\n", SHRT_MAX);
    printf("short min = %hd\n", SHRT_MIN);
    printf("long max = %ld\n", LONG_MAX);
    printf("long min = %ld\n", LONG_MIN);
    printf("long long max = %lld\n", LLONG_MAX);
    printf("long long min = %lld\n", LLONG_MIN);
    printf("unsigned int max = %u\n", UINT_MAX);
    printf("unsigned short max = %u\n", USHRT_MAX);
    printf("unsigned long max = %lu\n", ULONG_MAX);
    printf("unsigned long long max = %llu\n", ULLONG_MAX);
  
    printf("十六进制==================================\n");
    printf("int max = %#x\n", INT_MAX);
    printf("int min = %#x\n", INT_MIN);
    printf("short max = %#hx\n", SHRT_MAX);
    printf("short min = %#hx\n", SHRT_MIN);
    printf("long max = %#lx\n", LONG_MAX);
    printf("long min = %#lx\n", LONG_MIN);
    printf("long long max = %#llx\n", LLONG_MAX);
    printf("long long min = %#llx\n", LLONG_MIN);
    printf("unsigned int max = %#x\n", UINT_MAX);
    printf("unsigned short max = %#hx\n", USHRT_MAX);
    printf("unsigned long max = %#lx\n", ULONG_MAX);
    printf("unsigned long long max = %#llx\n", ULLONG_MAX);

    return 0;
}

/**
 * 基本数据类型
 * 整数：
 *  int：32位 或 16位，转换说明：%d
 *      0x前缀表示16进制，0前缀表示8进制
 *      %d：十进制显示
 *      %o：八进制显示，%#o：显示前缀0
 *      %x：十六进制显示，%#x：显示前缀0x
 *  short：16位，  %hd，%ho，%hx
 *
 *  long：32位，   %ld，%lo，%lx
 *
 *  long long：64位，  %lld
 *
 *  unsigned int：无符号int，    %u
 *  unsigned long：无符号long，  %lu
 *  unsigned long long：     %llu
 */
```

##### 转义序列

| 转义序列     | 含义          |
| ------------ | ------------- |
| \a           | 警报（ANSIC） |
| \b           | 退格          |
| \f           | 换页          |
| \n           | 换行          |
| \r           | 回车          |
| \t           | 水平制表符    |
| \v           | 垂直制表符    |
| \\\          | 反斜杠        |
| \\'          | 单引号        |
| \\"          | 双引号        |
| \?           | 问号          |
| \0oo（\07）  | 八进制值      |
| \xhh（\xff） | 十六进制值    |
