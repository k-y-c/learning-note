# NanoLog学习

## 1. 使用方法

- 调用`namespace nanolog`中的initialize函数

```cpp
nanolog::initialize(nanolog::NonGuaranteedLogger(10), "/tmp/", "nanolog", 1);
//参数_1设置log模式，NonGuaranteedLogger(X)效率更高，但不保证一定记录，存在日志丢失的情况，(X)表示内部缓存大小(MB)，若为GuaranteedLogger(),则不存在日志丢失，但延迟会更大。
//_2为文件路径，_3为文件名称，_4为rollfile大小（MB)
```

- 记录日志

```cpp
LOG_INFO << "Logging " << benchmark << i << 0 << 'K' << -42.42;
```

---

## 2. 源码分析

### 1. `nanolog::initialize()`

在`namespace nanolog`中有2个全局变量，调用initialize()是会将这些变量初始化。

```cpp
	std::unique_ptr<NanoLogger> nanologger;
	std::atomic<NanoLogger *> atomic_nanologger;

	void initialize(NonGuaranteedLogger ngl, std::string const &log_directory, std::string const &log_file_name, uint32_t log_file_roll_size_mb)
	{
		nanologger.reset(new NanoLogger(ngl, log_directory, log_file_name, log_file_roll_size_mb));
		atomic_nanologger.store(nanologger.get(), std::memory_order_seq_cst);
	}
	//还有一个重载版本，仅参数__1不同
```

该函数会在**堆上**开辟一块内存，存储NanoLogger，在`class NanoLogger`的构造函数中，会生成一条消费者线程，用于将日志写入file。

```cpp
public:
		NanoLogger(NonGuaranteedLogger ngl, std::string const &log_directory, std::string const &log_file_name, uint32_t log_file_roll_size_mb)//还有一个版本的构造函数，同样的，仅参数__1不同
			: m_state(State::INIT), 
        m_buffer_base(new RingBuffer(std::max(1u, ngl.ring_buffer_size_mb) * 1024 * 4)), 
        m_file_writer(log_directory, log_file_name, std::max(1u, log_file_roll_size_mb)), 
        m_thread(&NanoLogger::pop, this)	//新建一个写日志的线程
		{
			m_state.store(State::READY, std::memory_order_release);
		}

private:
		std::thread m_thread;
```

在构造函数的初始值列表中，`m_thread(&NanoLogger::pop, this)`用于创建一条新线程，线程函数为类成员函数，故需传入this指针。NanoLogger::pop()的作用是从buffer中取出日志并写入file，留作后续分析。

---

### 2. LOG宏

```cpp
LOG_INFO << "Logging " << benchmark << i << 0 << 'K' << -42.42;
```

记录日志的代码`LOG_INFO,LOG_WARN,LOG_CRIT`实际上是宏定义。

```cpp
#define NANO_LOG(LEVEL) nanolog::NanoLog() == nanolog::NanoLogLine(LEVEL, __FILE__, __func__, __LINE__)
#define LOG_INFO nanolog::is_logged(nanolog::LogLevel::INFO) && NANO_LOG(nanolog::LogLevel::INFO)
#define LOG_WARN nanolog::is_logged(nanolog::LogLevel::WARN) && NANO_LOG(nanolog::LogLevel::WARN)
#define LOG_CRIT nanolog::is_logged(nanolog::LogLevel::CRIT) && NANO_LOG(nanolog::LogLevel::CRIT)
```

`nanolog::is_logged(nanolog::LogLevel::XXX)`用于判断该日志级别的日志是否需要被记录。

```cpp
enum class LogLevel : uint8_t { INFO, WARN, CRIT };
```

```cpp
	bool is_logged(LogLevel level)
	{
		return static_cast<unsigned int>(level) >= loglevel.load(std::memory_order_relaxed);
	}
```

`NANO_LOG`对应第一行的宏，总体说，LOG_INFO可以替换为`is_logged && NanoLog() == NanoLogLine(…)`,先判断`is_logged`，然后进行'=='操作符运算，注意，此处的'=='已被重载。

`nanolog::NanoLogLine(LEVEL, __FILE__, __func__, __LINE__)`会生成一个匿名对象，该类重载了'<<'，故可在LOG_INFO后像stream一样使用'<<'。最终NanoLogLine对象会记录该条日志信息。

---

`class NanoLog`是个空类，内部只重载了operator ==，换句话说，该类的目的就是使用operator ==的重载版本。

```cpp
	bool NanoLog::operator==(NanoLogLine &logline)
	{
		atomic_nanologger.load(std::memory_order_acquire)->add(std::move(logline));
		return true;
	}
```

---

### 3. NanoLogLine的实现

#### 1. 数据成员

```cpp
private:
	size_t m_bytes_used;			// 已记录的bytes数
	size_t m_buffer_size;			// buffer容量
	std::unique_ptr < char [] > m_heap_buffer;		//当栈上buffer不够时，启用堆上buffer
	char m_stack_buffer[256 - 2 * sizeof(size_t) - sizeof(decltype(m_heap_buffer)) - 8 /* Reserved */];
```

NanoLogLine的底层就是一个char*类型的buffer，该类对这个buffer进行封装，并且在传入数据时进行了“编码”

---

#### 2. 日志行记录方式

在`class NanoLogLine`构造函数中，我们传入了日志等级、文件名、函数名、行号这些信息，在函数内部，分别调用相应的encode函数对其进行“编码”。

```cpp
	NanoLogLine::NanoLogLine(LogLevel level, char const *file, char const *function, uint32_t line)
		: m_bytes_used(0), m_buffer_size(sizeof(m_stack_buffer))
	{
		encode<uint64_t>(timestamp_now());		//记录时间戳
		encode<std::thread::id>(this_thread_id());		//线程id
		encode<string_literal_t>(string_literal_t(file));		//文件名
		encode<string_literal_t>(string_literal_t(function));		//函数名
		encode<uint32_t>(line);		//行号
		encode<LogLevel>(level);		//日志等级
	}
```

```cpp
	template <typename Arg>
	void NanoLogLine::encode(Arg arg)
	{
		*reinterpret_cast<Arg *>(buffer()) = arg;		// buffer()返回char*，此处将其强转为对应的类型指针，再进行赋值
    // 相当于在一块地址空间中任意存储我们想要的数据，但必须人为保证这是安全的！
		m_bytes_used += sizeof(Arg);		// 将buffer头部位置更新
	}

	char *NanoLogLine::buffer()
	{
		return !m_heap_buffer ? &m_stack_buffer[m_bytes_used] : &(m_heap_buffer.get())[m_bytes_used];// 每次都是返回更新后的指针
	}
```

---

#### 3. operator<<重载

除了NanoLogLine构造函数中记录的常规信息外，我们还需要在日志中记录个性化信息，这是用类似stream的方式实现，NanoLog支持的数据成员有`char, uint32_t, uint64_t, int32_t, int64_t, double, NanoLogLine::string_literal_t, char *`。

其中，对于char、uint32_t、uint64_t、 int32_t、int64_t、double这几个数据，调用的接口几乎相同，只需修改模板即可。

```cpp
	NanoLogLine &NanoLogLine::operator<<(double arg)
	{
		encode<double>(arg, TupleIndex<double, SupportedTypes>::value); // value对应每个类型的id号，在编译期即确定
		return *this;
	}

	NanoLogLine &NanoLogLine::operator<<(char arg)
	{
		encode<char>(arg, TupleIndex<char, SupportedTypes>::value); 
		return *this;
	}
//	还有几个重载版本，此处略去……
```

```cpp
	template <typename Arg>
	void NanoLogLine::encode(Arg arg, uint8_t type_id)
	{
		resize_buffer_if_needed(sizeof(Arg) + sizeof(uint8_t));	// 个性化的日志内容可能会撑爆栈上buffer，故在此处判断
		encode<uint8_t>(type_id);	
		encode<Arg>(arg);	//	对于个性化日志的存储方式：type_id + arg
	}
```

针对char * 数据类型，作者定义了许多种接口，包括char *、const char *、const std::string、string_literal_t。对于不同的字符串类型将进入不同的接口函数。

**对于`LOG_INFO << "Logging "`这种情况，只有两种接口函数能够适配。**

```cpp
NanoLogLine& operator<<(const char * const &arg)	//此处去掉&也能识别，但是这样编译器会优先进入string的接口

template < size_t N >
NanoLogLine& operator<<(const char (&arg)[N])
```

若为定义这两种接口函数，编译器会将"Logging"转换为std::string类型，进入string的接口函数。

编译器会将"Logging"的类型识别为const char[9]，const char [] 、const char *、char *均不能与之匹配。

> char * 为字符串指针
>
> const char * 和 char const * 等价，用于定义一个指向字符串常量的指针，ptr指向一个char*类型的常量，即不可用ptr修改所指向的内容。`ptr[2] = 'a'`将报错。
>
> char * const用于定义一个常量指针，ptr的值不可变，但可以改变指向的内容。`ptr = ptr2`将报错。

作者为兼容string literal，为"Logging"这样的类型单独定义了接口，与char、double等类型共用encode函数。

对于其他类型，将在重载操作符函数中将调用encode_c_string()

```cpp
	NanoLogLine &NanoLogLine::operator<<(std::string const &arg)
	{
		encode_c_string(arg.c_str(), arg.length());	//	std::string("abc");
		return *this;
	}
	void NanoLogLine::encode_c_string(char const *arg, size_t length)
	{
		if (length == 0)
			return;

		resize_buffer_if_needed(1 + length + 1);
		char *b = buffer();
		auto type_id = TupleIndex<char *, SupportedTypes>::value;	//	获取char *的id
		*reinterpret_cast<uint8_t *>(b++) = static_cast<uint8_t>(type_id);	//	放入id
		memcpy(b, arg, length + 1);	//	放入字符串
		m_bytes_used += 1 + length + 1;
	}
```

---

#### 4.resize_buffer_if_needed

考虑到个性化的日志内容会撑爆栈上的buffer，故定义该函数。在每次调用encode时首先进行resize_buffer_if_needed。

```cpp
	void NanoLogLine::resize_buffer_if_needed(size_t additional_bytes)
	{
		size_t const required_size = m_bytes_used + additional_bytes;

		if (required_size <= m_buffer_size)	//	无需扩容
			return;

		if (!m_heap_buffer)	//	暂未开辟堆上空间
		{
			m_buffer_size = std::max(static_cast<size_t>(512), required_size);
			m_heap_buffer.reset(new char[m_buffer_size]);
			memcpy(m_heap_buffer.get(), m_stack_buffer, m_bytes_used);
			return;
		}
		else	//	已开辟堆上空间，但仍不够用
		{
			m_buffer_size = std::max(static_cast<size_t>(2 * m_buffer_size), required_size);
			std::unique_ptr<char[]> new_heap_buffer(new char[m_buffer_size]);
			memcpy(new_heap_buffer.get(), m_heap_buffer.get(), m_bytes_used);
			m_heap_buffer.swap(new_heap_buffer);
		}
	}
```

---

### 4. 将日志行导入buffer

在调用`LOG_INFO << …… << …… << …… `时，首先将<<……这些内容全部输入到NanoLogLine中，随后将nanologline加入buffer。

```cpp
std::unique_ptr<NanoLogger> nanologger;
std::atomic<NanoLogger *> atomic_nanologger;	//	全局变量，在调用initialize(……)时已经初始化
bool NanoLog::operator==(NanoLogLine &logline) //	这里是非常量引用
	{
		atomic_nanologger.load(std::memory_order_acquire)->add(std::move(logline));	// 将logline转为右值
		return true;
	}
void NanoLogger::add(NanoLogLine &&logline)	//右值引用
		{
			m_buffer_base->push(std::move(logline)); // 同样的，将左值logline转为右值
		}
```

**重要：`#define NANO_LOG(LEVEL) nanolog::NanoLog() == nanolog::NanoLogLine(LEVEL, __FILE__, __func__, __LINE__)`这里创建的是一个NanoLogLine匿名对象，而operator==的参数中确实非常量引用，按理说这是不合法的，但是编译通过了，原因在于我们使用LOG时，都会用上'<<'，operator<<会返回*this的引用，将匿名对象转换为左值，也就是说，如果单独使用LOG_INFO，会报错。**

---

`m_buffer_base->push(std::move(logline));`会将logline导入buffer中，这里作者使用了面向对象编程的思想，m_buffer_base是一个基类指针，具体指向在构造函数中确定。

```cpp
private:
	std::unique_ptr<BufferBase> m_buffer_base;
```

```cpp
	struct BufferBase	//这是一个抽象基类
	{
		virtual ~BufferBase() = default;
		virtual void push(NanoLogLine &&logline) = 0;
		virtual bool try_pop(NanoLogLine &logline) = 0;
	};
```

作者定义了两种buffer方式，具体在下文介绍。

---

### 5. QueueBuffer

这种buffer模式保证log不会丢失，但效率会更低（延迟变大）。

```cpp
class QueueBuffer : public BufferBase
{
  //	接口函数
private:
		std::queue<std::unique_ptr<Buffer>> m_buffers;	//	buffer队列
		std::atomic<Buffer *> m_current_write_buffer;		//	当前使用buffer，可能会有多个线程同时往buffer中写数据，故使用原子操作
		Buffer *m_current_read_buffer;	//	供写日志的线程调用（这是一个线程，故不需要原子操作）
		std::atomic<unsigned int> m_write_index;	//	buffer下标，依旧是原子操作
		std::atomic_flag m_flag;
		unsigned int m_read_index;	//	buffer下标，供写日志的线程调用
}
```

作者采用原子操作往buffer中写数据，从而实现无锁，提高效率。

#### 1. 往buffer中写入数据

```cpp
		void setup_next_write_buffer()	// 在当前写入的buffer满或buffer初始化时，调用该函数
		{
			std::unique_ptr<Buffer> next_write_buffer(new Buffer());	// 新建一个buffer
			m_current_write_buffer.store(next_write_buffer.get(), std::memory_order_release);	// 更新m_current_write_buffer
			SpinLock spinlock(m_flag);	// 自旋锁，锁住后面两句代码，当被锁时，进行忙等
			m_buffers.push(std::move(next_write_buffer));	// buffer队列更新
			m_write_index.store(0, std::memory_order_relaxed);	// 初始化buffer下标
		}

		Buffer *get_next_read_buffer()	// 供写日志线程调用
		{
			SpinLock spinlock(m_flag);
			return m_buffers.empty() ? nullptr : m_buffers.front().get();
		}
```

```cpp
		QueueBuffer() : m_current_read_buffer{nullptr}, m_write_index(0), m_flag{ATOMIC_FLAG_INIT}, m_read_index(0)
		{
			setup_next_write_buffer();	// 初始化一个buffer
		}

		void push(NanoLogLine &&logline) override
		{
			unsigned int write_index = m_write_index.fetch_add(1, std::memory_order_relaxed); //每写入一条日志，下标加一
			if (write_index < Buffer::size)
			{
				if (m_current_write_buffer.load(std::memory_order_acquire)->push(std::move(logline), write_index))
				{
					setup_next_write_buffer();	// 如果当前buffer写满了，申请下一个buffer
				}
			}
			else
			{
				while (m_write_index.load(std::memory_order_acquire) >= Buffer::size)	// 等待其他线程开辟新buffer
					;
				push(std::move(logline));	//该函数可能被多个线程调用，故存在其他线程调用后导致buffer已满，而本线程还没有申请新空间的情况，当其他线程申请了新空间之后，再次调用本函数。
			}
		}
```

---

关于`class Buffer`的内部实现

```cpp
	class Buffer
	{
	public:
		struct Item
		{
			Item(NanoLogLine &&nanologline) : logline(std::move(nanologline)) {}
			char padding[256 - sizeof(NanoLogLine)];
			NanoLogLine logline;	// 虽然nanologline会开辟堆上空间，但是sizeof NanoLogLine是不变的
		};

		static constexpr const size_t size = 32768; // 8MB. 该buffer会存储size条logline

		Buffer() : m_buffer(static_cast<Item *>(std::malloc(size * sizeof(Item))))	// 构造函数申请空间
		{
			for (size_t i = 0; i <= size; ++i)
			{
				m_write_state[i].store(0, std::memory_order_relaxed);
			}
			static_assert(sizeof(Item) == 256, "Unexpected size != 256");
		}

		~Buffer()
		{
			unsigned int write_count = m_write_state[size].load();
			for (size_t i = 0; i < write_count; ++i)
			{
				m_buffer[i].~Item();
			}
			std::free(m_buffer);	// 释放内存
		}

		// Returns true if we need to switch to next buffer
		bool push(NanoLogLine &&logline, unsigned int const write_index)
		{
			new (&m_buffer[write_index]) Item(std::move(logline));
      // 此处的new不是开辟新内存，是placement new，在&m_buffer[write_index]这个地址上安放Item(std::move(logline))
			m_write_state[write_index].store(1, std::memory_order_release); // 该位置标记1，代表此处内存已写入日志
			return m_write_state[size].fetch_add(1, std::memory_order_acquire) + 1 == size; // 若m_write_state的最后一个位置也被置1，则代表需要开辟新buffer
		}

		Buffer(Buffer const &) = delete;
		Buffer &operator=(Buffer const &) = delete;	// 禁止拷贝

	private:
		Item *m_buffer;
		std::atomic<unsigned int> m_write_state[size + 1];
	};
```

#### 2. 取出buffer数据

前文提到，initialize(……)会新建一个写日志的线程，这个线程会在buffer中取出logline，并写进文件，该线程函数如下。

```cpp
		void NanoLogger::pop()
		{
			// Wait for constructor to complete and pull all stores done there to this thread / core.
			while (m_state.load(std::memory_order_acquire) == State::INIT)
				std::this_thread::sleep_for(std::chrono::microseconds(50));

			NanoLogLine logline(LogLevel::INFO, nullptr, nullptr, 0);

			while (m_state.load() == State::READY)	// 线程主循环
			{
				if (m_buffer_base->try_pop(logline))	// 从buffer中提取logline，这里会直接修改logline的值
					m_file_writer.write(logline);			// 往文件写入logline
				else
					std::this_thread::sleep_for(std::chrono::microseconds(50));
			}

			// Pop and log all remaining entries
			while (m_buffer_base->try_pop(logline))
			{
				m_file_writer.write(logline);
			}
		}
```

```cpp
		bool QueueBuffer::try_pop(NanoLogLine &logline) override
		{
			if (m_current_read_buffer == nullptr)	// 读buffer为空时，从buffer队列中取一个
				m_current_read_buffer = get_next_read_buffer();

			Buffer *read_buffer = m_current_read_buffer;

			if (read_buffer == nullptr)	// buffer为空，没有logline可写
				return false;

			if (bool success = read_buffer->try_pop(logline, m_read_index))
			{
				m_read_index++;
				if (m_read_index == Buffer::size)	// 如果该buffer读完了，直接弃掉
				{
					m_read_index = 0;
					m_current_read_buffer = nullptr;
					SpinLock spinlock(m_flag);	// 自旋锁，只用于锁住buffer队列操作
					m_buffers.pop();	// std::queue<std::unique_ptr<Buffer>> m_buffers;
          									// Buffer会自动销毁
				}
				return true;
			}

			return false;
		}
```

**注意：pop操作不会反复利用buffer空间，该函数会按顺序读取buffer内容（没有回退），一旦读完直接将buffer弃掉。**

```cpp
		bool Buffer::try_pop(NanoLogLine &logline, unsigned int const read_index)
		{
			if (m_write_state[read_index].load(std::memory_order_acquire)) //判断该位置是否已写入logline
			{
				Item &item = m_buffer[read_index];
				logline = std::move(item.logline);	// logline是引用传参，可修改其内容
				return true;
			}
			return false;
		}
```

---

### 6. RingBuffer

这个buffer是一个环形buffer，当buffer满时，后面的日志会覆盖最初的日志。

```cpp
	private:
		size_t const m_size;
		Item *m_ring;		//	环形缓冲区
		std::atomic<unsigned int> m_write_index;
		char pad[64];
		unsigned int m_read_index;	// 这里必须是无符号整数
```

```cpp
		void push(NanoLogLine &&logline) override
		{
			unsigned int write_index = m_write_index.fetch_add(1, std::memory_order_relaxed) % m_size;	// 这里是取余，以实现环形缓存
			Item &item = m_ring[write_index];
			SpinLock spinlock(item.flag);
			item.logline = std::move(logline);
			item.written = 1;		// item和QueueBuffer类似，但多了一个写入标记
		}
```

```cpp
		bool try_pop(NanoLogLine &logline) override
		{
			Item &item = m_ring[m_read_index % m_size];	// 同样取余
			SpinLock spinlock(item.flag);
			if (item.written == 1)		// 判断写入标记
			{
				logline = std::move(item.logline);
				item.written = 0;
				++m_read_index;
				return true;
			}
			return false;
		}
```

### 7. FileWriter（往文件中写日志）

写日志线程在取出logline之后，会调用`m_file_writer.write(logline);`往文件中写入日志

```cpp
	class FileWriter
	{
	public:
		FileWriter(std::string const &log_directory, std::string const &log_file_name, uint32_t log_file_roll_size_mb)
			: m_log_file_roll_size_bytes(log_file_roll_size_mb * 1024 * 1024), m_name(log_directory + log_file_name)
		{
			roll_file();
		}

		void write(NanoLogLine &logline)
		{
			auto pos = m_os->tellp();	// 返回文件尾部
			logline.stringify(*m_os);	// 写文件函数
			m_bytes_written += m_os->tellp() - pos;
			if (m_bytes_written > m_log_file_roll_size_bytes)	// 文件滚动
			{
				roll_file();
			}
		}

	private:
		void roll_file()	//用于文件滚动，当文件size达到阈值后，另建新文件
		{
			if (m_os)
			{
				m_os->flush();	// 立即刷新缓冲区
				m_os->close();
			}

			m_bytes_written = 0;
			m_os.reset(new std::ofstream());
			// TODO Optimize this part. Does it even matter ?
			std::string log_file_name = m_name;
			log_file_name.append(".");
			log_file_name.append(std::to_string(++m_file_number));
			log_file_name.append(".txt");
			m_os->open(log_file_name, std::ofstream::out | std::ofstream::trunc);
		}

	private:
		uint32_t m_file_number = 0;
		std::streamoff m_bytes_written = 0;
		uint32_t const m_log_file_roll_size_bytes;
		std::string const m_name;
		std::unique_ptr<std::ofstream> m_os;
	};
```

此处作者将写文件的具体操作函数放在了`class NanoLogLine`中，***合理？***合理的原因可能是logline中含有数据，直接调用其成员函数可以减小拷贝。

```cpp
	void NanoLogLine::stringify(std::ostream &os)
	{
		char *b = !m_heap_buffer ? m_stack_buffer : m_heap_buffer.get(); // 这块内存中装满了各种杂数据，取时需要一一对应
		char const *const end = b + m_bytes_used;
		uint64_t timestamp = *reinterpret_cast<uint64_t *>(b);
		b += sizeof(uint64_t);
		std::thread::id threadid = *reinterpret_cast<std::thread::id *>(b);
		b += sizeof(std::thread::id);
		string_literal_t file = *reinterpret_cast<string_literal_t *>(b);
		b += sizeof(string_literal_t);
		string_literal_t function = *reinterpret_cast<string_literal_t *>(b);
		b += sizeof(string_literal_t);
		uint32_t line = *reinterpret_cast<uint32_t *>(b);
		b += sizeof(uint32_t);
		LogLevel loglevel = *reinterpret_cast<LogLevel *>(b);
		b += sizeof(LogLevel);	// 写入常规信息

		format_timestamp(os, timestamp);

		os << '[' << to_string(loglevel) << ']'
		   << '[' << threadid << ']'
		   << '[' << file.m_s << ':' << function.m_s << ':' << line << "] ";

		stringify(os, b, end);	// 用于写入个性化日志信息

		os << std::endl;

		if (loglevel >= LogLevel::CRIT)
			os.flush();	// 如果是critical级别的日志，立即刷新缓冲区
	}
```

```cpp
	void NanoLogLine::stringify(std::ostream &os, char *start, char const *const end)
	{
		if (start == end)
			return;

		int type_id = static_cast<int>(*start);
		start++;

		switch (type_id)	//后续为递归调用
		{
		case 0:
			stringify(os, decode(os, start, static_cast<std::tuple_element<0, SupportedTypes>::type *>(nullptr)), end);
			return;
		case 1:
			stringify(os, decode(os, start, static_cast<std::tuple_element<1, SupportedTypes>::type *>(nullptr)), end);
			return;
		case 2:
			stringify(os, decode(os, start, static_cast<std::tuple_element<2, SupportedTypes>::type *>(nullptr)), end);
			return;
		case 3:
			stringify(os, decode(os, start, static_cast<std::tuple_element<3, SupportedTypes>::type *>(nullptr)), end);
			return;
		case 4:
			stringify(os, decode(os, start, static_cast<std::tuple_element<4, SupportedTypes>::type *>(nullptr)), end);
			return;
		case 5:
			stringify(os, decode(os, start, static_cast<std::tuple_element<5, SupportedTypes>::type *>(nullptr)), end);
			return;
		case 6:
			stringify(os, decode(os, start, static_cast<std::tuple_element<6, SupportedTypes>::type *>(nullptr)), end);
			return;
		case 7:
			stringify(os, decode(os, start, static_cast<std::tuple_element<7, SupportedTypes>::type *>(nullptr)), end);
			return;
		}
	}
```

这里的decode函数，作者定义了三个重载，分别对应string_literal、char *和其他类型。

```cpp
	template <typename Arg>
	char *decode(std::ostream &os, char *b, Arg *dummy)	// 普通类型可以直接导入
	{
		Arg arg = *reinterpret_cast<Arg *>(b);
		os << arg;
		return b + sizeof(Arg);
	}

	template <>
	char *decode(std::ostream &os, char *b, NanoLogLine::string_literal_t *dummy)	// string_literal
	{
		NanoLogLine::string_literal_t s = *reinterpret_cast<NanoLogLine::string_literal_t *>(b);
		os << s.m_s;
		return b + sizeof(NanoLogLine::string_literal_t);
	}

	template <>
	char *decode(std::ostream &os, char *b, char **dummy)	// char*类型需要逐字符导入
	{
		while (*b != '\0')
		{
			os << *b;
			++b;
		}
		return ++b;
	}
```

