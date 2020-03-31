# 标准IO

## 库的编写

缓存问题与程序员影响

### 静态库的编写

假设库文件包含a.h、a1.cpp、a2.cpp（示例3.1）

```c
//a.h
void f();
void g();
```

```c
//a1.cpp
#include<iostream>
#include"a.h"

using namespace std;

void f(){
    cout << "f()" << endl;
}
```

```c
//a2.cpp
#include<iostream>
#include"a.h"

using namespace std;

void g(){
    cout << "g()" << endl;
}
```

```c
#include"a.h"
#include<iostream>
using namespace std;
int main(){
    f();
    g();
    return 0;
}
```

创建库

```bash
$g++ -c a1.cpp a2.cpp
$ar -rc libtest.a a1.o a2.o
```

使用库

```bash
$g++ -o statictest statictest.cpp –L. -ltest
```



### 动态库的编写

动态库的编写

```bash
$g++ -fpic –shared –o libtest.so a1.cpp a2.cpp#生成libtest.so
```

#### 动态库使用

打开动态链接库

```c
#include<dlfcn.h>
void *dlopen(const char *file, int mode);
```

参数

- file：动态链接库的文件名，包括路径信息
- mode：动态链接库的使用方式，例如RTLD_LAZY：动态地加入动态链接库中的函数
- 返回值：引用动态链接库的句柄；出错返回NULL

映射动态链接库中的函数

```c
#include<dlfcn.h>
void *dlsym(void *handle, const char *FuncName);
```

参数

- handle：dlopen的返回值
- FuncName：动态链接库中的函数名
- 返回值：FuncName函数被加载后，在进程地址空间中的地址；出错返回NULL

查看出错原因

```c
#include<dlfcn.h>
char *dlerror();
```

返回值

当dlopen、dlsym等函数出错时，dlerror返回字符串说明这些函数出错的原因

卸载动态链接库

```c
#include<dlfcn.h>
int dlclose(void *handle);
```

参数

handle：dlopen的返回值

动态库使用者的编译

\#g++ -o test test.cpp –ldl

\#test    出错？

```c
//test.cpp
#include"a.h"
#include<dlfcn.h>
#include<iostream>

using namespace std;

int main()
{
    void *handle = dlopen("./libtest.so", RTLD_LAZY);
    if(0 == handle)
    {
	cout << "dlopen error" << endl;
	return 0;
    }
    
    typedef void (*Fun)();

    Fun f1 = (Fun)dlsym(handle, "f");//f1函数原型

    if(0 == f1)
    {
	cout << "f1 error" << endl;
    	char *str = dlerror();
	cout << str << endl;
	return 0;
    }
	
    (*f1)(); //用函数指针调用函数

    dlclose(handle);

    return 0;
}

```

```bash
$./test
f1 error
./libtest.so:undefined symbol: f
$nm libtest.so #查看文件符号
#这返回结果中没有看到f函数
#看可以到_Z1fv和_Z1gv
```

动态库导出函数的变形

查看动态库导出的函数

	#nm libtest.so

f函数实际上在动态库中的名字是：

	_Z1fv

#### 库编编写注意事项

##### 导出函数的名称

在动态库的实现文件中函数的名称，与动态库导出的函数名称可能不同

使用extern “C”使的导出的函数名称和实际名称一致

```c
//（示例3.3）a.h
extern "C" void f();
extern "C" void g();
```

extern “C”：告诉编译器按C语言的方式设定函数的导出名

<u>不同的编译器、不同的语言，对函数名的修改都有可能不同</u>

##### 函数调用约定

C语言调用约定

```c
void __cdecl f(int a, int b);  //VC环境
void f(int a, int b) __attribute__((cdecl)) //g++环境
```

f被表示成_f

从右至左，将参数压入堆栈

函数调用者负责压入参数和堆栈平衡

```c
void __stdcall f(int a, int b);  //VC环境
```

f被表示成_f@8；8表示参数的字节数

从右至左，将参数压入堆栈

函数内负责堆栈平衡

```c
void __fastcall f(int a, int b);  //VC环境
```

由寄存器传送参数，用ecx和edx传送参数列表中前两个双字或更小的参数，剩下的参数仍然从右至左压入堆栈

函数内负责堆栈平衡

C++类成员函数的调用约定：thiscall

this指针存放于ecx寄存器中

参数从右至左压入堆栈

##### 结构体大小:star:

```c
struct A{
	int i;
};//4字节

```



##### 结构体对齐

##### 谁分配谁释放

### 接口的注意事项:star:

