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



- 只有内置类型的全局变量才会被默认初始化为0，定义于**函数中、块中、类中**的变量被默认初始化时，它们的值是未定义的。



- lambda表达式

  ```cpp
  //[捕获列表](参数列表)->返回类型{函数体}
  //参数列表和返回类型可以省略
  auto f = []{return 42;};
  //捕获列表可以捕获lambda所在函数的局部变量（全局变量可以直接调用）
  //值捕获：创建时拷贝
  //引用捕获：如果引用的对象已经不存在，会出现问题
  //除非显示定义返回引用，否则默认返回右值
  int b = 0;
  auto fun = [&b]{return b;};
  fun() = 10; //error: lvalue required as left operand of assignment
  
  int b = 0;
  auto fun = [&b]()->int&{return b;};
  fun() = 10;// OK
  //隐式捕获：[=]:值捕获，[&]:引用捕获
  //如果要改变值捕获的内容，需要加上mutable,值捕获默认不改变其值
  auto f = [v1]()mutable{return ++v1;};
  ```

- emplace和insert的区别

  - emplace接收构造函数的参数，内部用其构造函数创建一个对象

  - insert接收一个对象，将对象拷贝到容器中，可以使用**初始值列表**

    ```cpp
    map<int,int> hash;
    hash.emplace(1,2);
    hash.insert({3,4});
    ```

    