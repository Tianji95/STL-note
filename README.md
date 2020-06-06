# STL-note
note of book &lt;effective STL> and &lt;the Annotated STL Sources>



## STL源码剖析

##### allocator

allocator是一个可以将内存分配和对象构造分离开来的东西，因为new是先分配内存，然后一定会调用类的构造函数，两者绑定在一起会不是很方便，效率也比较低。所以allocator允许我们先分配一个内存，不调用对象的构造函数。使用的时候必须显式的调用对象的构造函数。举个例子：

```
std::allocator<std::string> alloc; // 可以分配string的allocator对象
int n{ 5 };
auto const p = alloc.allocate(n); // 分配n个未初始化的string

auto q = p; // q指向最后构造的元素之后的位置
alloc.construct(q++); // *q为空字符串
alloc.construct(q++, 10, 'c'); // *q为cccccccccc
alloc.construct(q++, "hi"); // *q为hi

std::cout << *p << std::endl; // 正确：使用string的输出运算符
//std::cout << *q << std::endl; // 灾难：q指向未构造的内存
std::cout << p[0] << std::endl;
```

allocator的必要接口：

```C++
allocator::value_type
allocator::pointer
allocator::const_pointer
allocator::reference
allocator::const_reference
allocator::size_type
allocator::difference_type
```

allocator 的源码如下，实际上只是operator new和operator delete的包装。没做什么事情，实际上我们使用的是名为alloc的特定版本，拥有allocator和construct、deallocator和deconstruct：

```
template<class T>
inline T allocate(ptrdiff_t size, T*){
    set_new_handler(0);
    T* tmp = (T*)(::operator new((size_t)(size * sizeof(T))));
    if(tmp == 0){
       cerr << "out of memory" << endl;
    }
    return tmp;
}
template<class T>
inline void deallocate(T* buffer){
    ::operator delete(buffer);
}
template<class T>
class allocator{
public:
    typedef T value_type;
    typedef T* pointer;
    typedef const T* const_pointer;
    typedef T& reference;
    typedef const T& const_reference;
    typedef size_t size_type
    typedef ptrdiff_t difference_type;
    pointer allocate(size_type n){
        return ::allocate((difference_type)n, (pointer)0);
    }
    void deallocate(pointer p) { ::deallocate(p); }
    pointer address(reference x) { return (pointer)&x; }
    const_pointer const_address(const_reference x){
        return (const_pointer)&x;
    }
    size_type init_page_size(){
        returnmax(size_type(1), size_type(4096/sizeof(T)));
    }
    size_type max_size() const {
        return max(size_type(1), size_type(UINT_MAX/ sizeof(T)));
    }
};
//特化版本
class allocator<void>{
public:
    typedef void* pointer;
}
```

allocator的destroy有两个版本，一个版本是直接把对象析构掉，另一个版本是把[first, last)个对象析构掉，如果这些对象都是trivial destructor（无关痛痒的析构函数），那么就不管他，如果不是的话，就一个一个析构掉。所以为了实现第二个版本，STL会增加“对象是否是trivial”的判断，使用value_type()来判断

**空间的配置和释放**

allocator会考虑内存破碎问题，所以在底层设计的时候有一个双层配置器，当配置区块超过128Bytes的时候，认为足够大，调用第一级配置器，当申请区块小于128Bytes的时候，被认为足够小，使用memory pool内存整理方式。具体如下所示

![allocator_memory](.\images\allocator_memory.png)

能够看出的是，allocator使用的是malloc和free来配置内存，而不是用new和delete来配置内存，这一点归结于历史原因。

第一级配置器在内存不够的时候尝试调用oom_realloc和oom_malloc，如果oom_realloc和oom_malloc再出现错误的话，会直接抛出异常，这时候就需要客户端来处理内存不足错误了。

第二级配置器在内存不够的时候会尝试调用第一级配置器，如果小于128Bytes，

