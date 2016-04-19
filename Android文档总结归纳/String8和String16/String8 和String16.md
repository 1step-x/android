#String8 和String16
>String8 和String16是android底层库单字节字符和双字符字符的实现。

```cpp
system/core/include/utils/String8.h
system/core/libutils/String8.cpp
system/core/include/utils/String16.h
system/core/libutils/String16.cpp
```
###1、实现
```cpp
[1]
String8::String8(const String8& o)
    : mString(o.mString)
{
    SharedBuffer::bufferFromData(mString)->acquire();
}

[2]
String8::String8(const char* o, size_t len)
    : mString(allocFromUTF8(o, len))
{
    if (mString == NULL) {
        mString = getEmptyString();
    }
}
```
对于第一种构造函数，很明显是将`o.mString`的值直接赋给`mString`，而`mString`是类型为`char*`的指针，即将`mString`指向`o.mString`指向的字符串，这里并不发生内存的拷贝。
函数体中的`SharedBuffer::bufferFromData(mString)`实现如下，即将`mString`的指针强制转换成`SharedBuffer*`之后再-1。
```cpp
SharedBuffer* SharedBuffer::bufferFromData(void* data) {
    return data ? static_cast<SharedBuffer *>(data)-1 : 0;
}
```
上面是如何通过`mString`如何得到一个`SharedBuffer*`指针的呢？我们来看看`SharedBuffer`是什么？
```cpp
private:
        inline SharedBuffer() { }
        inline ~SharedBuffer() { }
        SharedBuffer(const SharedBuffer&);
        SharedBuffer& operator = (const SharedBuffer&);
 
        // 16 bytes. must be sized to preserve correct alignment.
        mutable int32_t        mRefs;
                size_t         mSize;
                uint32_t       mReserved[2];
```
`SharedBuffer`的构造函数都是`private`属性，因此我们没有办法直接创建一个SharedBuffer对象，只能通过静态方法`alloc`实现:
```
SharedBuffer* SharedBuffer::alloc(size_t size)
{
    // Don't overflow if the combined size of the buffer / header is larger than
    // size_max.
    LOG_ALWAYS_FATAL_IF((size >= (SIZE_MAX - sizeof(SharedBuffer))),
                        "Invalid buffer size %zu", size);
	//注意下面的分配的内存大小是 SharedBuffer的大小 + 要求的size
    SharedBuffer* sb = static_cast<SharedBuffer *>(malloc(sizeof(SharedBuffer) + size)); 
    if (sb) {
        sb->mRefs = 1;
        sb->mSize = size;
    }
    return sb;
}
```
这里`alloc`分配的内存大小为`sizeof(SharedBuffer)`加上`需要的size`，此处`需要的size`即是我们字符串数据的大小。并且将`mRefs`引用个数置为1。
再通过下面这个函数将`mString`指向`SharedBuffer`结构体的尾部。
```cpp
void* SharedBuffer::data() {
    return this + 1;
}
```
所以，当字符串不为空的时候，`SharedBuffer`和`mString`是成对出现的，`mString`紧接在`SharedBuffer`的后面，`SharedBuffer`中保存着`mString`的引用次数，如果引用次数变为`1`(调用release函数之前)，则通过alloc分配的内存都会被回收。

String8 提供了10个构造函数，主要由上面两种实现。
 * String8::String8()
 * String8::String8(StaticLinkage)
 * String8::String8(const String8& o)
 * String8::String8(const char* o)
 * String8::String8(const char* o, size_t len)
 * String8::String8(const String16& o)
 * String8::String8(const char16_t* o)
 * String8::String8(const char16_t* o, size_t len)
 * String8::String8(const char32_t* o)
 * String8::String8(const char32_t* o, size_t len)

###2、其他函数
**赋值函数**
```
[1]
void String8::setTo(const String8& other)
{
    SharedBuffer::bufferFromData(other.mString)->acquire();
    SharedBuffer::bufferFromData(mString)->release();
    mString = other.mString;
}

[2]
status_t String8::setTo(const char* other)
{
    const char *newString = allocFromUTF8(other, strlen(other));
    SharedBuffer::bufferFromData(mString)->release();
    mString = newString;
    if (mString) return NO_ERROR;

    mString = getEmptyString();
    return NO_MEMORY;
}
```

**添加函数**
```
[1]
status_t String8::append(const String8& other)
{
    const size_t otherLen = other.bytes();
    if (bytes() == 0) {
        setTo(other);
        return NO_ERROR;
    } else if (otherLen == 0) {
        return NO_ERROR;
    }

    return real_append(other.string(), otherLen);
}

[2]
status_t String8::append(const char* other, size_t otherLen)
{
    if (bytes() == 0) {
        return setTo(other, otherLen);
    } else if (otherLen == 0) {
        return NO_ERROR;
    }

    return real_append(other, otherLen);
}
```
还有添加路径，得到路径等。
###String16和String8的实现类似，只不过String16被定义为2个字节的字符串。


