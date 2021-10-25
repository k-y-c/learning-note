- 顶层const与底层const
  - 顶层const表示指针是常量，指针不能变，所指内容可以改变
  - 底层const表示指针所指对象是一个常量
  - 使用typedef之后，**不能直接把类型别名替换成原来的样子**

```cpp
int i = 0;
int *const pi = &i;		// pi是顶层const，不能修改pi的值
const int *pi2 = &i;	// pi2是底层const
const int i2 = 1;			// 这是顶层const

typedef char *pstring;
const pstring cstr = 0; //cstr是指向char的常量指针，顶层const
//等同于 char *const cstr = 0;
//如果直接把pstring替换为char*，则会出现歧义
//const char* cstr = 0; 这是一个底层const
```



- decltype用于获取变量类型或函数返回值类型，保留顶层const和引用属性
  - decltype内如果为表达式，那么<u>能左则左</u>，如p为int*类型，decltype(*p)`会返回一个int&，因为*p可以作为左值使用
  - `decltype((var))`的返回类型永远是引用

