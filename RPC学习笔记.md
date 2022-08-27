# 基于zeromq的简易RPC学习笔记



## 整体框架

![rpc](.\linux_fig\rpc.jpg)



## buttonrpc c++11

项目包含两个hpp，分别是`buttonrpc.hpp`和`Serializer.hpp`，后者负责序列化部分，前者通过一个类实现了server和client部分。

### Server部分

#### 调用方式

```cpp
buttonrpc server;
server.as_server(5555);
//绑定函数
server.bind("foo_1", foo_1);
//绑定成员函数
ClassMem s;
server.bind("foo_6", &ClassMem::bar, &s);
//绑定函数指针
// ...
server.run();
```

#### 原理

##### 1. as_server

在调用`server.as_server(port)`后，将注册socket，并绑定端口。

```cpp
void buttonrpc::as_server( int port )
{
	m_role = RPC_SERVER;
	m_socket = new zmq::socket_t(m_context, ZMQ_REP);
	ostringstream os;
	os << "tcp://*:" << port;
	m_socket->bind (os.str());
}
```

##### 2. bind

用于绑定相应的函数，`class buttonrpc`维护一个`std::map<std::string, std::function<void(Serializer*, const char*, int)>>`，用于保存索引对应函数。

bind的接口实现如下

```cpp
//function & function pointer
template<typename F>
void buttonrpc::bind( std::string name, F func )
{
	m_handlers[name] = std::bind(&buttonrpc::callproxy<F>, this, func, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3);
}
//class member function
template<typename F, typename S>
inline void buttonrpc::bind(std::string name, F func, S* s)
{
	m_handlers[name] = std::bind(&buttonrpc::callproxy<F, S>, this, func, s, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3);
}
```

由`buttonrpc::callproxy`统一调用func，由于**函数参数个数未知**，该框架通过模板判断，并实现了最多5个参数的函数调用。

```cpp
template<typename F>
void buttonrpc::callproxy( F fun, Serializer* pr, const char* data, int len )
{
	callproxy_(fun, pr, data, len);
}
template<typename R>
void callproxy_(std::function<R()>, Serializer* pr, const char* data, int len);

template<typename R, typename P1>
void callproxy_(std::function<R(P1)>, Serializer* pr, const char* data, int len);

template<typename R, typename P1, typename P2>
void callproxy_(std::function<R(P1, P2)>, Serializer* pr, const char* data, int len);

template<typename R>
void buttonrpc::callproxy_(std::function<R()> func, Serializer* pr, const char* data, int len)
{
	typename type_xx<R>::type r = call_helper<R>(std::bind(func));

	value_t<R> val;
	val.set_code(RPC_ERR_SUCCESS);
	val.set_val(r);
	(*pr) << val;
}
```

最终在`call_helper`中调用func并判断是否返回参数。

##### 3.server.run

在绑定好所有函数之后，进入run死循环。

```cpp
void buttonrpc::run()
{
	// only server can call
	if (m_role != RPC_SERVER) {
		return;
	}
	while (1){
		zmq::message_t data;
		recv(data);
		StreamBuffer iodev((char*)data.data(), data.size());
		Serializer ds(iodev);

		std::string funname;
		ds >> funname;
		Serializer* r = call_(funname, ds.current(), ds.size()- funname.size());

		zmq::message_t retmsg (r->size());
		memcpy (retmsg.data (), r->data(), r->size());
		send(retmsg);
		delete r;
	}
}
```

接受到client的调用请求之后，获取函数名，并调用`call_`执行请求。

```cpp
Serializer* buttonrpc::call_(std::string name, const char* data, int len)
{
	Serializer* ds = new Serializer();
	if (m_handlers.find(name) == m_handlers.end()) {
		(*ds) << value_t<int>::code_type(RPC_ERR_FUNCTIION_NOT_BIND);
		(*ds) << value_t<int>::msg_type("function not bind: " + name);
		return ds;
	}
	auto fun = m_handlers[name];
    // 由于bind期间已经对fun进行了封装，此时可以直接调用。
    // 函数返回值保存在ds中。
    // data和len为函数传参的信息
	fun(ds, data, len);
	ds->reset();
	return ds;
}
```

### client部分

#### 调用方式

```cpp
buttonrpc client;
client.as_client("127.0.0.1", 5555);
client.set_timeout(2000);
client.call<void>("foo_1");
int foo6r = client.call<int>("foo_6", 10, "buttonrpc", 100).val();
```

#### 原理

##### 1.as_client

创建socket，并连接到server

```cpp
void buttonrpc::as_client( std::string ip, int port )
{
	m_role = RPC_CLIENT;
	m_socket = new zmq::socket_t(m_context, ZMQ_REQ);
	ostringstream os;
	os << "tcp://" << ip << ":" << port;
	m_socket->connect (os.str());
}
```

##### 2.call

`call`方法接受函数名及相应参数，将这些内容序列化之后，调用`net_call`。`net_call`中将序列化的消息发送，并接收`reply`，获取返回值。

```cpp
template<typename R, typename P1>
inline buttonrpc::value_t<R> buttonrpc::call(std::string name, P1 p1)
{
	Serializer ds;
	ds << name << p1;
	return net_call<R>(ds);
}

template<typename R>
inline buttonrpc::value_t<R> buttonrpc::net_call(Serializer& ds)
{
	zmq::message_t request(ds.size() + 1);
	memcpy(request.data(), ds.data(), ds.size());
	if (m_error_code != RPC_ERR_RECV_TIMEOUT) {
		send(request);
	}
	zmq::message_t reply;
	recv(reply);
	value_t<R> val;
	if (reply.size() == 0) {
		// timeout
		m_error_code = RPC_ERR_RECV_TIMEOUT;
		val.set_code(RPC_ERR_RECV_TIMEOUT);
		val.set_msg("recv timeout");
		return val;
	}
	m_error_code = RPC_ERR_SUCCESS;
	ds.clear();
	ds.write_raw_data((char*)reply.data(), reply.size());
	ds.reset();

	ds >> val;
	return val;
}
```



### 序列化

`class Serializer`采用流式io，即

```cpp
Serializer ds;
ds << 1 << "param2" << classA();
int a;
ds >> a;
string b;
ds >> b;
classA c;
ds >> c;
```

序列化方式分两类（考虑大小端存储方式）：

1. string

   string.size() + string

2. other type

   sizeof(T) + char[sizeof(T)]

## buttonrpc c++14

解决了不同函数输入参数的重载接口问题，减少了代码量。