# JIT是如何工作的

## 定义

JIT就是Just In Time的缩写，但在编程中，JIT是这样的一个玩意：当一个程序运行的同时，生产一些程序本体之外可执行的代码，并执行它们，这就是JIT。

## JIT:生产机器码，然后运行它

JIT一般包含两个步骤：

* 程序运行时编译和生产机器码
* 程序运行时执行机器码

## 执行动态生成的代码

	这里只阐述**Unix**下JIT的实现。

第一步：分配可执行代码内存

``` cpp
// 分配一段可读写可执行的内存
void* alloc_executable_memory(size_t size) {
  void* ptr = mmap(0, size,
                   PROT_READ | PROT_WRITE | PROT_EXEC,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (ptr == (void*)-1) {
    perror("mmap");
    return NULL;
  }
  return ptr;
}
// 给分配的内存设置上可执行的标志位，内存必须在同一片内存页
int make_memory_executable(void* m, size_t size) {
  if (mprotect(m, size, PROT_READ | PROT_EXEC) == -1) {
    perror("mprotect");
    return -1;
  }
  return 0;
}
```

第二步：生成机器码，将代码拷贝至**`可执行`**内存

```cpp
void emit_code_into_memory(unsigned char* m) {
  unsigned char code[] = {
    0x48, 0x89, 0xf8,                   // mov %rdi, %rax
    0x48, 0x83, 0xc0, 0x04,             // add $4, %rax
    0xc3                                // ret
  };
  memcpy(m, code, sizeof(code));
}
```

第三步：执行该段代码

```cpp
const size_t SIZE = 1024;
typedef long (*JittedFunc)(long);
//执行机器码
void run_from_rwx() {
  void* m = alloc_executable_memory(SIZE);
  emit_code_into_memory(m);
  make_memory_executable(m, SIZE);

  JittedFunc func = m;
  int result = func(2);
  printf("result = %d\n", result);
}
```

看到这里，原理很简单吧 ^_^ 。

> 原文：[How to JIT - an introduction](http://eli.thegreenplace.net/2013/11/05/how-to-jit-an-introduction)