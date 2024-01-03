# APUE 学习笔记（参考李慧芹老师的Linux系统编程）

# 1.标准IO

功能说明：复制src_file内容到dest_file中
``` C
#include <stdio.h>
#include <stdlib.h>

// 命令行传参
int main(int argc, char **argv) {

    FILE *fps, *fpd;
    int ch; // 存储读入的字符

    if(argc < 3) {
        fprintf(stderr, "Usage:%s <src_file> <dest_file>\n", argv[0]);
        exit(1);
    }

    fps = fopen(argv[1], "r");
    if(fps == NULL) {
        perror("fopen()");
        exit(1);
    }

    fpd = fopen(argv[2], "w");
    if(fpd == NULL) {
        fclose(fps);
        perror("fopen()");
        exit(1);
    }

    while(1) {
        ch = fgetc(fps);
        if(ch == EOF) { // 读到文件末尾结束循环
            break;
        }
        fputc(ch, fpd);
    }
    
	// 释放内存，后开的先关
    fclose(fpd);
    fclose(fps);

    exit(0);
}
```

## 1.1 getline
### 1.1.1函数原型  
``` C
#define _GNU_SOURCE // 通常将这种宏写在makefile中，现在的编译器没有了该宏，直接使用即可
#include <stdio.h>
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```
### 1.1.2使用示例： 
``` C
//mycpy_getline.c
int main(int argc, char **argv) {
    FILE *fp;
    // 一定要初始化，否则指针会指向内存中的随机位置
    char *linebuf = NULL;
    size_t linesize = 0;
    if(argc < 2) {
        fprintf(stderr, "Usage...\n");
    }
    fp = fopen(argv[1], "r");
    if(fp == NULL) {
        perror("fopen()");
        exit(1);
    }
    while(1) {
        // 当返回-1时则读完
    	if(getline(&linebuf, &linesize, fp) < 0)
            break;
       	printf("%d\n", strlen(linebuf));
    }
    fclose(fp);
    exit(0);
}

```
### 1.1.3内部实现原理
一个简单的getline()的实现示例  
说明：由于在函数内部可能会改变指针所指向的内存地址，所以需要使用指向指针的指针，以便在函数内部能够修改指针的值并在函数外部保持更改
如果传入的是一级指针，虽然在函数内部我们可以通过修改一级指针的内容来更改分配的内存块，但这里的 lineptr 实际上是传递的指针的副本。这意味着在函数内部改变 lineptr 的值不会影响函数外部传递的实际指针值。  

``` C
#include <stdio.h>
#include <stdlib.h>

ssize_t getline(char **lineptr, size_t *n, FILE *stream) {
    if (*lineptr == NULL || *n == 0) {
        *n = 128; // 初始分配大小
        *lineptr = (char *)malloc(*n * sizeof(char)); // 分配内存
        if (*lineptr == NULL) {
            return -1; // 内存分配失败
        }
    }

    int c;
    size_t i = 0;
    while ((c = fgetc(stream)) != EOF && c != '\n') {
        (*lineptr)[i++] = (char)c;

        if (i >= *n - 1) { // 如果超出当前分配大小，则重新分配内存
            *n *= 2;
            char *temp = (char *)realloc(*lineptr, *n * sizeof(char));
            if (temp == NULL) {
                free(*lineptr);
                return -1; // 内存重新分配失败
            }
            *lineptr = temp;
        }
    }

    (*lineptr)[i] = '\0'; // 在字符串末尾添加 null 终止符
    return i; // 返回读取的字符数（不包括 null 终止符）
}

```

补充：变参函数（va_func）
典型实现：printf、scanf
示例： 
``` C
#include <stdio.h>
#include <stdarg.h>

double sum(int num, ...) {
    va_list args;
    double total = 0.0;

    // 初始化va_list，使其指向参数列表的开始位置
    va_start(args, num);

    // 访问参数列表中的每个参数
    for (int i = 0; i < num; ++i) {
        total += va_arg(args, double);
    }

    // 清理va_list
    va_end(args);

    return total;
}

int main() {
    double result = sum(4, 1.1, 2.2, 3.3, 4.4);
    printf("Sum is: %f\n", result);

    return 0;
}

```