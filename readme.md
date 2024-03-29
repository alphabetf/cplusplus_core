#### C++核心

在类中写的简短的成员函数实现，默认会自动隐式加上inline关键字，

如果成员函数不改变类中的成员变量，则应该在**函数后面加上const关键字**

**相同class的各个object互为friends(友元)**

```c++
class complex
{
public:
	complex(double r = 0, double i = 0):re(r),im(i)
    { ... }
    
    int func(const complex& param)
    {
    	return param.re + param.im;    
    }
private: // 私有属性不能被外部成员访问
    double re, im;		
}

/* 使用情景 */
complex c1(2,1);
complex c2;

c2.func(c1);  // 非友元函数却可以使用其他同类对象的私有属性
```

原因：类中的成员函数在构建编译时就能清楚的知道类的内存布局，所以编译器也允许成员函数中具有自身类类型的参数，在使用类成员变量时，其实是基于指针偏移的，所以机缘巧合的造成了这种情况，其实并非友元，但可以理解记忆为：**相同class的各个object互为friends(友元)**

**new的理解与实现:**

C++中的new可以理解为是一个写**法比较特殊的函数**，具体实现源码如下：

```c++
Complex* pc = new Complex(1,2);

void* mem = operator new( sizeof(Complex) );		/* 分配内存，内部调用maclloc（n） */
pc = static_cast<Complex*>(mem);					/* 类型转型 */
pc->Complex::Complex(1,2);							/* 调用构造函数 */
//pc->Complex::Complex(this,1,2);	/* 构造函数隐式的隐藏了一个this指针,this指向分配的内存地址*/
```

**delete的理解与实现:**

C++中的delete也可以理解为是一个**写法比较特殊的函数**，具体源码实现如下：
```c++
String* ps = new String("Hello");
...
delete ps;		

//delete内部实现源码如下
String::~String(ps);		/* 先调用类中的析构函数 */
operator delete(ps);		/* 释放分配的内存，内部调用free(ps) */
```

**delete中的内存泄露：**

```c++
/* 内存结构中会记录数组个数,每一个String对象中都会维护一个指针指向实际的字符串内存地址 */
String* p = new String[3]; 
...
delete[] p;	/* 根据内存结构中记录的数组个数,多次调用析构函数,此处会调用三次析构函数, */
delete p;	/* 只会调用一次析构函数 */
```

**虚析构函数：**只要该类将来会**被继承或成为父类**，就应将该类的析构函数**写成虚析构函数**

```C++
class Base
{
public:
    Base(){};
    virtual ~Base(){cout << "父类的析构函数被执行" << endl;};
    void DoSomething(){cout << "父类dosomething" << endl;};
}

class Derived:public Base
{
public:
    Derived(){};
    virtual ~Derived(){cout << "子类的析构函数被执行" << endl;}
    void DoSomething(){cout << "子类dosomething" << endl;};
}

int main()
{
    Base *p = new Derived; /* 此处产生多态 */
    delete p;	/* 如果Base类不是虚析构则之后调用~Base(),而不会调用~Derived()后在调用~Base() */
    return 0;
}

```

**内部原理&机制**：

​		1，所有被virtual关键字修饰的函数都会被编译进虚函数表，析构函数也不例外

​		2，在一个继承体系中按照析构顺序，所有析构函数会被整合成一个统一的析构函数，并按照一定的规则重新			  命名，同于普通函数，析构函数会被整合而不是重新或者覆盖

**设计模式:TemplateMethod**

```c++
/* 这是一个很久以前写的Application framework */
#include <iostream>
using namespace std;

class CDocument
{
public:
    void OnFileOpen()
	{
    	/* `这是一个算法，每一个cout输出代表着一个实际动作 */
    	cout << "dialog..." << endl;
    	cout << "check file status..." << endl;
    	cout << "open file..." << endl;
    	Serialize();		/* 子类重写该函数后，将会以子类的方式去读取文件 */
    	cout << "close file..." << endl;
    	cout << "update all views..." << endl;
	}
    
    virtual void Serialize(){ }; /* 不知道如何读取文件，所以交给未来的代码去处理 */
}

/* 这是一个现在写的代码，需要使用以前的框架 */
class CMyDoc : public CDocument
{
public:
    virtual void Serialize()	/* 以前写的框架将会调用现在写的函数来读取文件 */
    {
        /* 只有应用程序本身才知道该如何读取自己的文件 */
        cout << "CMyDoc::Serialize()" << endl;
    }
}

/* 使用 */
int main()
{
    CMyDoc myDoc;
    myDoc.OnFileOpen();
}
```

**设计模式:Observer**

```c++
/* 这是订阅中心 */
class Sbuject
{
    int m_value;	/* 这是一个公共数据，所有订阅者都会收到这份相同的数据 */
    verctor<Observer*> m_views; /* 订阅者列表 */
public:
    void attach(Observer* obs)	/* 想要订阅数据,应该先向订阅中心进行注册 */
    {
       m_views.push_back(obs);	
    }
    
    void set_val(int value)
    {
        m_value = value;
        notify();	/* 当数据产生变更时,应及时通知所有订阅者 */
    }
    
    void notify() /* 遍历订阅列表,通知所有订阅者 */
    {
        for(int i = 0; i < m_views.size(); ++i)
        {
            m_views[i]->updata(this, m_value);
        }
    }
};

/* 这是所有观察者的基类,所有观察者都应先在订阅中心进行注册 */
class Observer
{
Public:
    /* 子类应重写该函数,可以不同观察者的自定义数据处理或展示 */
    virtual void updata(Sbuject *sub， int value) = 0;
};
```

**设计模式:Composite**

```c++
/* 假设现在要设计一个文件系统,那么一个目录中可能有文件也可能有目录,则我们需要设计一个目录类，一个文件类
   目录类既可以添加文件类又可以添加目录类自身 */
class Component
{
    int value;
public：
    Component(int val) { value = val; }
    virtual void add(Component*){ }		/* 文件中不能添加文件或者目录,所以默认为空 */
};

/* 这是一个文件类,用于标识文件 */
class Primitive:public Componet
{
public:
    Primitive(int val):Component(val){}
};

/* 这是一个目录类,用于标识目录 */
class Composite:public Component
{
    vector<Component*> c;
public:
    Composite(int val):Component(val){}
    
    void add(Component* elem){	/* 目录中既可以添加文件也可以添加目录 */
        c.push_back(elem);
    }
};
```

**设计模式:Prototype**

```c++
/* 使用场景:让现有的继承体系有能力去创建未来才会出现的的子类 */
#include <iostream.h>
enum imageType{
    LAST,SPOT
};
/* 父类 */
class Image
{
public:
    virtual void draw() = 0;	
    static Image* findAndClone(imageType);
protected:
    virtual imageType returnType() = 0;
    virtual Image* clone() = 0;	/* 无法知道未来子类的名称,创建一个新的子类必须由子类自己来完成 */
    static void addPrototype(Image *image)	
    {
        _prototype[_nextSlot++] = image;
    }
private:
    /* 静态变量在此处只是声明 */
    static Image* _prototype[10];	/* 一个容器,存储着向父类注册的子类 */
    static int _nextSlot;
};
/* 此处是静态变量的定义 */
Image* Image::_prototype[];
int Image::_nextSlot;

Image* Image::findAndClone(imageType type)
{
    for(int i = 0; i < _nextSlot; i++){
        if(_prototype[i]->returnType() == type){
            return _protorype[i]->clone(); /* new一个新的子类，只能由子类自己来完成 */
        }
    }
}

/* 子类,未来才会出现 */
class LandSatImage:public Image
{
public:
    imageType returnType(){
        return LSAT;
    }
    void draw()
    {
        cout << "LandSatImage::draw" << _id << endl;
    }
    Image* clone(){
        return new LandSatImage(1);	 /* 调用有参构造函数,避免重复向父类注册 */
    }
protected:
    LandSatImage(int dummy){
        _id = _count++;
    }
private:
    static LandSatImage _landSatImage; /* 默认调用自己的私有的无参构造函数 */
    LandSatImage(){
        addPrototype(this);	/* 将维护的静态类_landSatImage注册到父类中 */
    }
    int _id;
    static int _count;
};
/* 此处才是静态变量的定义 */
LandSatImage LandSatImage::_landSatImage; 
int LandSatImage::_count = 1;

/* 子类,未来才会出现 */
class SpotImage:public Image
{
public:
    imageType returnType{
        return SPOT;
    }
    void draw(){
        cout << "SpotImage::draw" << endl;
    }
    Image* clone(){
        return new SpotImage(1); /* 调用有参构造函数,避免重复向父类注册 */
    }
protected:
    SpotImage(int dummy){
        _id = _count++;
    }
private:
    SpotImage(){	/* 将维护的静态类_SpotImage注册到父类中 */
        addPrototype(this);
    }
    static SpotImage _SpotImage; /* 默认调用自己的私有的无参构造函数 */
    int _id;
    static int _count;
};
/* 此处才是静态变量的定义 */
SpotImage SpotImage::_spotImage;
int SpotImage::_count;

/* 以下是测试程序 */
// Simulated stream of creation requests
const int NUM_IMAGES = 8;
imageType input[NUM_IMAGES] =
{
	LSAT, LSAT, LSAT, SPOT, LSAT, SPOT, SPOT, LSAT
};

int main()
{
    Image *images[NUM_IMAGES];
    // Given an image type, find the right prototype, and return a clone
    for (int i = 0; i < NUM_IMAGES; i++)
        images[i] = Image::findAndClone(input[i]);
    // Demonstrate that correct image objects have been cloned
    for (i = 0; i < NUM_IMAGES; i++)
        images[i]->draw();
    // Free the dynamic memory
    for (i = 0; i < NUM_IMAGES; i++)
        delete images[i];
}
```

**转换函数:conversion function**

```c++
class Fraction
{
public:
    Fraction(int num, int den=1):m_numerator(num),m_denominator(den){ }
    /* 固定写法:转换函数,无需写返回类型,此处返回类型就是double类型(不一定要是基本类型,也		可以是任意类型),将类转换为浮点数 */
    operator double() const{ /* 转换函数一般不修改变量，所以应用const修饰 */
        return (double)(m_numerator/m_denominator);
    }
private:
    int m_numerator;	/* 分子 */
    int m_denominator;	/* 分母 */
};

/* 使用 */
Fraction f(3,5);
double d=4+f; /* 编译器会尝试将f转换为double类型,调用operator double()将f转换为0.6 */
```

**非显示的一个参数：non explicit one argument**：

```c++
class Fraction
{
public:
    Fraction(int num, int den=1):m_numerator(num),m_denominator(den){ }

    Fraction operator+(const Fraction& f){
        return Fraction(...);
    }
private:
    int m_numerator;	/* 分子 */
    int m_denominator;	/* 分母 */
};

/* 使用 */
Fraction f(3,5);
double d2=f+4;/* 编译器会尝试将4转换为Fraction类(调用构造)后在调用operator+()函数 */

```

```c++
class Fraction
{
public:
    Fraction(int num, int den=1):m_numerator(num),m_denominator(den){ }
	/* 固定写法:转换函数,无需写返回类型,此处返回类型就是double类型(不一定要是基本类型,也		可以是任意类型),将类转换为浮点数 */
    operator double() const{ /* 转换函数一般不修改变量，所以应用const修饰 */
        return (double)(m_numerator/m_denominator);
    }
    Fraction operator+(const Fraction& f){
        return Fraction(...);
    }
private:
    int m_numerator;	/* 分子 */
    int m_denominator;	/* 分母 */
};

/* 使用 */
Fraction f(3,5);
/* 编译报错:存在二义性错误,编译器可以将f转化为double类型相加后在转化为Fraction类型，也可以将4转换为	  	 Fraction类型,所以存在二义性 */
Fraction d2=f+4;
```

```c++
class Fraction
{
public:
    /* explict关键字由于显示的告诉编译器,该构造函数只能显示的被调用,不能隐式的调用,如将4隐式的转换为		   Fraction类 */
    explict Fraction(int num, int den=1):m_numerator(num),m_denominator(den){ }
	/* 固定写法:转换函数,无需写返回类型,此处返回类型就是double类型(不一定要是基本类型,也可以是任意类型),		 将类转换为浮点数 */
    operator double() const{ /* 转换函数一般不修改变量，所以应用const修饰 */
        return (double)(m_numerator/m_denominator);
    }
    Fraction operator+(const Fraction& f){
        return Fraction(...);
    }
private:
    int m_numerator;	/* 分子 */
    int m_denominator;	/* 分母 */
};

/* 使用 */
Fraction f(3,5);
/* 编译报错:转换失败,将4转换为Fraction类型不被允许,则编译器将f转换为double类型相加后,在转换为Fraction	 类型,但是double类型也无法转换为Fraction类型 */
Fraction d2=f+4;
```

 **模板特化与泛化：**

```c++
/* 模板的泛化,Key可接受任意类型 */
template <class Key>
    struct hash{}

/* 模板的特化 */
template<>
struct hash<char> {
    size_t operator()(char x) const { return x;	}
};

template<>
struct hash<int> {
    size_t operator()(int x) const { return x; }
};

template<>
struct hash<long> {
    size_t operator()(long x) const { return x; }
}

/* 使用 */
cout << hash<long>()(1000); /* 编译器会优先使用特化模板 */
```

**模板的部分特化:个数上的特化**

```c++
/* 泛化的模板 */
template<typename T, typename Alloc=......>
class  vector
{
    ...
};

/* 模板的部分特化:个数上的特化 */
template<typename Alloc=......> 
class vector<bool, Alloc>	/* 只对第一个进行特化 */
{
    ...
}
```

**模板的部分特化:范围上的特化**

```c++
/* 泛化的模板 */
template <typename T>
class C
{
    ...
}

/* 部分特化:范围上的特化 */
template <typename T>
class C<T*>	/* T是指针类型的就使用这部分特化代码 */
{
    ...
}

/* 使用 */
C<string> obj1;		/* 使用的是泛化的模板 */
C<string*> obj2;	/* 使用的是范围特化的模板 */
```

**模板中的模板参数：**

```c++
template <typename T,
		 template <typename T>	/* 模板中的第二依然是一个模板类 */
			class Container
         >
class XCls
{
private:
    Container<T> c;	/* 第二个模板参数会替换Container */
public:
    ...
}

/* 错误使用 */
XCls<string, list> mylist1;	/* list是一个模板容器,但是有第二模板参数,所以需要声明一下在使用 */

/* 定义类型别名,相当于对容器模板具体类型的明确 */
template <typename T>
using Lst = list<T, allocator<T>>;	/* 相当于typedef List<T, allocator<T>> Lst */

/* 明确类型后的正确使用 */
XCls<string, Lst> mylst2;
```

**不定数量的模板参数：**

```c++
void print()	/* 最后被调用的空的重载版本 */
{
    
}

/* 一个数据+一包数据 */
template <typename T, typename... Types> /* 固定书写格式 */
void print(const T& firstArg, const Types&... args) /* 固定书写格式 */
{
    cout << firstArg << endl;
    printg(args...); /* 固定书写格式 */
    // sizeof...(args);	/* 获得后面一包数据的数量 */
}

/* 使用 */
print(7.5, "hello", bitset<16>(377), 42); /* 依次输出 7.5 hello 0000000101111001 42  */
```

**auto关键字：**

```c++
/* auto让编译器自动推导类型,并定义该类型变量 */
list<string> c;
...
list<string>::iterator ite;	
ite = find(c.begin(), c.end(), target);

list<string> c;
...
auto ite = find(c.begin(), c.end(), target); /* 编译器会自动推导ite类型,并定义 */
```

**ranged-base for新型式的for语句：**

```c++
/* 语法格式 */
for(变量 : 容器)
{
    ...
}

/* 使用例子 */
for(int i : {2,3,4,7,9,13,17,19}){
    cout << i << endl;
}

/* 结合auto关键字,让编译器自动推导变量类型 */
vector<double> vec;
...
for(auto elem : vec){
    cout << elem << endl;
}

for(auto& elem : vec){
    elem *= 3;
}
```

**const：**

```c++
/* c++中多个存储同一字符串的“字符串类”共享一个实际字符串内存,但是读时共享,写时复制 */
class template::basic_string<...>中有如下两个成员函数

/* 类似于此处的const只能写在类的成员函数中,全局函数不能使用如此const修饰 */
/* 返回值不在函数重载的考虑范围内,但const修饰在函数重载的考虑范围内,所以此处不会产生二义性 */
charT operator[](size_type pos) const 
{
	/* 不需要考虑写时拷贝 */
}
reference operator[](size_type pos)
{
    /* 需要考虑写时拷贝 */
}

/* 当const成员函数和非const重载版本“同时存在时“,const对象”只能“调用const版本的成员函数，	 非const对象只能调用非const版本的成员函数 */
/* 当const成员函数和非const重载版本“不同时存在时“,const对象”依然只能“调用const版本的成员函	  数,非const对象既可以调用const版本也可以调用非const版本的成员函数 */
const String str("hello world");	
str.print(); /* print()不是const成员函数,所以此处编译会报错 */
```

**全局重载operator new,operator delete:**

```c++
/* 影响范围较大,不建议全局重载,这些重载函数会由编译器来传入参数并调用 */
inline void* operator new(size_t size){
    cout << "global new()" << endl;
    return malloc(size);
}
inline void* operator new[](size_t size){
    cout << "global new[]()" << endl;
    return malloc(size);
}
inline void operator delete(void* ptr)
{
	cout << "global delete()" << endl;  
    free(ptr);
}
inline void operator delete[](void* ptr)
{
	cout << "global delete[]()" << endl;  
    free(ptr);
}
```

**重载类中的operator new,operator delete:**

```c++
class Foo
{
public:
    void* operator new(size_t);
    void operator delete(void*,size_t); /* size_t为可选参数,可写可不写,由编译器传入 */
    ....
}

Foo* p = new Foo;
/* new操作的编译器内部隐式代码如下: */
try{
    void* mem = operator new(sizeof(Foo));	/* 分配内存 */
    p = static_cast<Foo*>(mem);				/* 强制类型转换 */
    p->Foo::Foo();							/* 调用构造函数进行初始化 */
}

delete p;
/* delete操作的编译器内部隐式代码如下: */
p->~Foo();									/* 先调用析构函数 */
operator delete(p);							/* 释放分配的内存 */
```

**重载类中的operator new[],operator delete[]:**

```c++
class Foo
{
public:
    void* operator new[](size_t);
    void operator delete[](void*,size_t); /* size_t为可选参数,可写可不写,由编译器传入 */
    ....
}

Foo* p = new Foo[N];
/* new操作的编译器内部隐式代码如下: */
try{
    void* mem = operator new(sizeof(Foo)*N+4);	/* 分配内存,多4个字节用于记录数组元素个数 */
    p = static_cast<Foo*>(mem);					/* 强制类型转换 */
    p->Foo::Foo();								/* 调用N次构造函数 */
}

delete [] p;
/* delete操作的编译器内部隐式代码如下: */
p->~Foo();									/* 调N次析构函数 */
operator delete(p);							/* 释放分配的内存 */
```

**重载new,delete的完整示例**

```c++
class Foo
{
public:
    int _id;
    long _data;
    string _str;
public:
    Foo():_id(0){
        cout << "default constructor id=" << id << endl; 
    }
    Foo(int i):_id(i) {
        cout << "constructor id=" << id << endl; 
    }
    virtual ~Foo(){
		cout << "destructor id=" << id << endl;
    }
    static void* operator new(size_t size);
    static void  operator delete(void* pdead, size_t size);
    static void* operator new[](size_t size);
    static void  operator delete[](void* pdead, size_t size);
}

void* Foo::operator new(size_t size){
    Foo* p = (Foo*)malloc(size);
    return p;
}
void Foo::operator delete(void* pdead, size_t size){
    free(pdead);
}
void* Foo::operator new[](size_t size){
    Foo* p = (Foo*)malloc(size);
    return p;
}
void Foo::operator delete[](void* pdead, size_t size){
    free(pdead);
}

/* 如果成员函数没有重载版本就使用全局版本 */
Foo* pf = new Foo;
delete pf;

/* 强制使用全局版本 */
Foo* pf = ::new Foo;	/* void* ::operator new(size_t) */
::delete pf;			/* void ::operator delete(void*) */
```

**重载placement new(),placement delete():**

```c++
/* 第一个参数“必须是size_t类型”,且由编译器自动传入,其对应的placement delete版本并不会被编译器自动调用,只有在对应的placement new版本的“构造函数中发生异常”时,编译器才会调用其对应的placement delete版本重载函数,用于释放所分配发内存 */
class Foo
{
public:
    Foo(){	cout << "Foo::Foo()" << endl; }
    Foo(int){ cout << "Foo::Foo(int)" << endl; throw Bad(); } /* 此处故意抛出异常 */
    /* 一般的operator new()重载 */
    void* operator new(size_t size){
        return malloc(size);
    }
    /* 下面是placement new()的重载 */
    void* operator new(size_t size, void* start){ 
        return start;
    }
    void* operator new(size_t  size, long extra){
        return malloc(size+extra)
    }
    void* operator new(size_t  size, long extra, char init){
        return malloc(size+extra)
    }
    
    /* 这是一般的operator delete()重载 */
    void operator delete(void*, size_t){
        cout << "operator delete(void*, size_t)" << endl;
    }
    /* 对应的placement delete重载版本 */
    void operator delete(void*, void*){
        cout << "operator delete(void* void*)" << endl;
    }
    void operator delete(void*, long){
        cout << "operator delete(void*, long)" << endl;
    }
    void operator delete(void*, long, char){
        cout << "operator delete(void*, long, char)" << endl;
    }
}

/* 使用 */
Foo* p1 = new Foo;	/* 调用void* operator new(size_t size) */
Foo* p2 = new(&start) Foo;	/* 调用operator new(size_t size, void* start) */
Foo* p3 = new(100,a) Foo;	/* 调用operator new(size_t  size, long extra, char init) */ 
```

**placement new在标准库中的使用：**

```c++
/* 用于申请内存时,多申请一部分内存,用于存储字符串 */
template<...>
class basic_string
{
private:
    struct Rep{		/* Rep结构体用于管理和记录字符串状态 */
        ...
        void release(){if(-ref == 0) delete this;}
        inline static void* operator new(size_t, size_t);
        inline static void operator delete(void*);
        inlien static Rep* create(size_t);
        ...
    }
    ...
}

/* 重载的palcement new()函数 */
template <class charT, class traits, class Allocator>
inline void* basic_string<charT, traits, Allocato>::Rep::
operator new(size_t s, size_t extra)
{
    return Allocator::allocate(s+extea*sizeof(charT));
}

/* 重载的operator delete()函数 */
template <class charT, class traits, class Allocator>
inline void* basic_string<charT, traits, Allocato>::Rep::
operator delete(void* ptr)
{
    Allocator::dallocate(ptr, sizeof(Rep)+
                         	  reinterpret_cast<Rep*>(ptr)->res 
                         	  * sizeof(charT));
}

template <class charT, class traits, class Allocator>
inline basic_string<charT, traits, Allocato>::Rep* 
basic_string<charT, traits, Allocato>::Rep::create(size_t extra)
{
    extra = frob_size(extra+1);
    Rep* p = new(extra)Rep;	 /* 除了分配Rep内存还要多分配extra字节内存用于存储字符串 */
    ...
}
```

**STL六大核心组件：**

**容器：**用于存储数据，只需关心数据本身，无需关心数据的内存结构

**分配器：**为容器中的数据提供内存的分配和管理

**算法：**用于处理数据并得到结果

**仿函数：**为算法处理提供一些自定义的特殊处理方式

**迭代器：**容器和算法的连接桥梁,泛化的指针,指向容器中的数据

**适配器：**对数据进行适配，用于迭代器，仿函数，和容器

```c++
/* 这是一个STL列子 */
#include <vector>
#include <algorithm>
#include <functional>
#include <iosteram>
using namespace std;

int main()
{
    int ia[6] = {27,210,12,47,109,83};	
    /* 创建一个vector容器,使用alloctor<int>分配器,将数组的起始和结束地址传入 */
    vector<int,allocator<int>> vi(ia, ia+6);
    /* count_if算法,返回在起始和结束范围内满足特定条件的元素数目 */
    /* less<int>(): 仿函数创建的匿名对象 */
    /* bind2nd:适配器,绑定仿函数的第二参数,始终将40传递给仿函数的第二参数 */
    /* not1:函数适配器,将结果取反,此处条件就是大于等于40 */
    cout << count_if(vi.begin(), vi.end(),
                    not1(bind2nd(less<int>(),40)));
    return 0;
}
```

C++标准库所有容器的迭代器遵循**“前闭后开 [ )”**区间

```c++
*(c.begin()) /* 返回的迭代器指针,指向容器的第一个元素地址 */
*(c.end())	 /* 返回的迭代器指针,指向容器的“最后一个元素的后一个”地址 */
```

**typename关键字:**

​	typename关键字**只能用于模板参数或模板定义之中**

​	typename用于**告诉编译器将该名称当成类型而不是变量**

```c++
template<typename _Iter>	/* 此处的typename用于模板参数 */
 /* typename用于告诉编译器这是一个类型(iterator_traits<_Iter>::iterator_category是一个类型) */
 inline typename iterator_traits<_Iter>::iterator_category 
 __iterator_category(const _Iter&)
 {
     /* 返回这个类型的operator()函数结果 */
     return typename Iterator_traits<_Iter>::iterator_category();
 }
```

**allocator分配器:**

```c++
/* STL中的分配器为容器提供具体的内存分配与管理 */
/* vector容器默认使用的是allocator分配器 */
template<typename _Tp, typename _Alloc = std::allocator<_Tp>>
  class vector: protected _Vector_base<_Tp,_Alloc>
  { ... }
/* list容器默认使用的是allocator分配器 */
template<typename _Tp, typename _Alloc = std::allocator<_Tp>>
  class list: protected _List_base<_Tp,_Alloc>
  { ... }
/* deque容器默认使用的是allocator分配器 */      
template<typename _Tp, typename _Alloc = std::allocator<_Tp>>
  class deque: protected _Deque_base<_Tp,_Alloc>
  { ... }
/* set容器默认使用的是allocator分配器 */      
template<typename Key, typename _Compare = std::less<Key>, 
					   typename _Alloc = std::allocator<_Key>>
  class set { ... }
/* map容器默认使用的是allocator分配器 */    
template<typename _Key, typename _Tp, typename _Compare = std::less<_Key>,
			typename _Alloc = std::allocator<std::pair<const _Key, _Tp>>>
  class map { ... }
/* unordered_set容器默认使用的是allocator分配器 */        
template<class _Value, class _Hash = hash<_Value>
    	 class _Pred = std::equal_to<_Value>
         class _Alloc = std::allocator<_Value>>
  class unordered_set { ... }
/* unordered_map容器默认使用的是allocator分配器 */   
template<class _Key, class _Tp,
		 class _Hash = hash<_Key>,
		 class _Pred = std::equal_to<_Key>
         class _Alloc = std::allocator<std::pair<const _Key, _Tp>>>
  class unordered_map { ... }
```

**GCC2.9 allocator<>实现源码：**

```c++
/* GCC2.9的allocator只是以::operator new和operator delete完成allocate()和deallocate(),没有如何特殊设计 */
template <class T>
  class allocator
  {
  public: 
      typedef T		value_type;
      typedef T*	pointer;
      typedef size_t size_type;    
      typedef ptrdiff_t difference_type
      pointer allocate(size_type n){
          return ::allocate((difference_type)n, (pointer)0); 	/* 分配内存 */
      }
      void deallocate(pointer p){ ::deallocate(p); }	/* 回收内存 */
  }
/* allocate实现 */
template <class T>
inline T* allocate(ptrdiff_t size, T*)
{
    set_new_handler(0);
    T* tmp = (T*)(::operator new((size_t)(size*sizeof(T))));	/* 调用全局new() */
    if(tmp == 0){
        cerr << "out of memory" << nedl;
        exit(1);
    }
    return tmp;
}
/* deallocate实现 */
template<class T>
inline void deallocate(T* buffer)
{
    ::operator delete(buffer);		/* 调用全局delete()函数 */
}
```

**C++标准库中提供了多种分配器,allocator<>只是其中一种**

```c++
/* 直接使用分配器进行内存分配 */
int* p;
allocator<int> alloc1;  					/* 创建分配器 */
p = alloc1.allocate(1);						/* 直接使用分配器分配内存 */
alloc1.deallocate(p,1);						/* 归还分配器分配的内存 */
__gun_cxx::malloc_allocator<int> alloc2;	/* 创建分配器 */
p = alloc2.allocate(1);						/* 直接使用分配器分配内存 */
alloc2.deallocate(p,1);						/* 归还分配器分配的内存 */
__gun_cxx::new_allocator<int> alloc2;		/* 创建分配器 */
p = alloc2.allocate(1);						/* 直接使用分配器分配内存 */
alloc2.deallocate(p,1);						/* 归还分配器分配的内存 */
__gun_cxx::__pool_allocator<int> alloc2;	/* 创建分配器 */
p = alloc2.allocate(1);						/* 直接使用分配器分配内存 */
alloc2.deallocate(p,1);						/* 归还分配器分配的内存 */
__gun_cxx::__mt_allocator<int> alloc2;		/* 创建分配器 */
p = alloc2.allocate(1);						/* 直接使用分配器分配内存 */
alloc2.deallocate(p,1);						/* 归还分配器分配的内存 */
__gun_cxx::bitmap_allocator<int> alloc2;	/* 创建分配器 */
p = alloc2.allocate(1);						/* 直接使用分配器分配内存 */
alloc2.deallocate(p,1);						/* 归还分配器分配的内存 */
```

**实现一个自定义的allocator:**

```c++
/* allocator标准接口,需要满足一下条件：
   需提供一组typedef: allocator::value_type;
   					allocator::pointer;
   					allocator::const_pointer;
   					allocator::reference;
   					allocator::const_reference;
   					allocator::size_type;
   					allcoator::difference_type；
   allocator::rebind:allocator内嵌模板,需要定义other成员
   allocator::allocator():构造函数
   allocator::allocator(const allocator&):拷贝构造函数
   template<typename T> allocator::allocator(const allocator<T>&):泛化的拷贝构造函数
   allocator::~allocator():析构函数
   pointer allocator::address(reference x) const:返回对象地址allocator.address(x)相当于&x
   pointer allocator::allocate(size_type n, const void*=0):分配内存空间
   void allocator::deallocator(pointer p,size_type n):释放分配的空间
   size_type allocator::max_size() const:可分配的最大内存空间
   void allocator::construct(pointer p, const T& x):
   					在指定地址处分配内存,相当于new((const void*)P) T(x);
   void allocator::destroy(pointer p):释放指定地址的内存,相当于p->~T();
*/
template <typename T>
struct MyAllocator{
    typedef T 			value_type;
    typedef T* 			pointer;
    typedef const T*	const_pointer
    typedef T&;			reference;
    typedef const T&	const_reference;
    typedef size_t		size_type;
    typedef int			difference_type;
    template<typename U>	/* 内嵌的rebind模板 */
    struct rebind{
        typedef MyAllocator<U> other;
    }
    /* 构造函数 */
    MyAllocator(){ ... }
    MyAllocator(MyAllocator<T> const&){ ... }
    pointer allocate(size_type n, const void* p = 0){	/* 分配n个T类型的内存 */
        T* buffer = (T*)malloc((size_t)(n*sizeof(T)));
        if(buffer == NULL){ /* 错误处理 */
            ...
        }
        return buffer;
    }
    void deallocate(pointer p,size_type n){
        if(p != NULL){
            free(p);
        }
    }
    /* placement construct函数 */
    void construct(pointer p, const T& value){
        new(p) T(value);	/* 需要重载placement new,在地址处构造T */
    }
    /* placement destroy */
    void destroy(pointer p, size_type n){
        p->~T();	/* 调用T的析构函数 */
    }
    /* 最大可分配内存 */
    size_type max_size() const {
        return size_type(UINT_MAX/sizeof(T)));
    }
    pointer address(reference x){
        return (pointer)&x;
    }
    const_pointer const_address(const_reference x){
        return (const_pointer)&x;
    }
 }
/* 使用 */
int elements[] = {1,2,3,4,5};
const int n = sizeof(elements)/sizeof(int);
vector<int,MyAllocator<int>> myvector(elements,elements+n);
```

**Iterator迭代器：**

```c++
/* list容器的迭代器 */
template<class T, class Ref, class Ptr>
class __list_iterator{
    typedef _list_iterator<T, Ref, Ptr> self;
    /* 迭代器原则:迭代器应为算法提供一下5种类型信息 */
    typedef bidirectioal_iterator_tag iterator_category;	/* 迭代器类型 */
    typedef T 	value_type;									/* 迭代器中的值类型 */
    typedef Ptr pointer;									/* 迭代器指针类型 */
    typedef Ref reference;									/* 迭代器的引用类型 */
    typedef ptrdiff_t difference_type;						/* 存储迭代器元素个数的类型 */
    typedef __list_node<T>* link_type;
    link_type node;											/* list容器中存储数据的节点 */
    
    reference operator*() const { return (*node).data; }
    pointer operator->() const { return &(operator*()); }
    /* self是迭代器,返回的是迭代器实体,返回引用类型,因为c++运行前置++连加,如:(++++i) */
    self& operator++()	{node = (link_type)((*node).next); return *this;} //前置++i
    /* 编译器解释时先遇到=和++,所以此处*this被编译器解释为self(迭代器实体)，所以这里调用的是拷贝构造函		 数和operator++(),返回值类型,因为c++不允许后置++连加,如:(i++++,编译报错)   */
    self operator++(int) { self tmp = *this; ++*this; return tmp };	//后置i++
    ...
    __list_iterator(const iterator& x):node(x.node){ ... }  /* 迭代器的拷贝构造函数 */
};
/* list存储数据的节点 */
template<class T>
struct __list_node
{
	typedef void* void_pointer;
	void_pointer prev;
	void_pointer next;
	T data;				
};
/* list容器 */
template <class T， class Alloc = alloc>
class list
{
protected:
    typedef __list_node<T> list_node;
public:
    typedef list_node* link_type;
    typedef __list_iterator<T,T&,T*> iterator;  /* 容器里面定义一个list迭代器类型 */
protected:
    link_type node;
    ...
};

/* 使用 */
list<Foo>::iterator ite;	/* 根据容器中的迭代器类型创建迭代器 */
*ite 						/* 获取一个Foo对象 */
/* 注意:当你对某一个类型实施operator->(),而这种类型又是非内建类型时,编译器在执行完这次operator->()
   后,在返回值基础上继续重复着上面的operator->()操作,直到遇到内建类型,然后在进行成员值存取 */
ite->method();				/* pointer operator->() const { return &(operator*()); } */
ite->field = 7;
```

**Traits特征提取器：**

```c++
/* 由于迭代器可能是自定义类型也可能是一个指针,指针无法为算法提供类型信息,所以需要加入一个中间层Traits,用于提取转化类型信息,间接的为算法提供所需要的类型信息 */
/* list容器的迭代器 */
template<class T, class Ref, class Ptr>
class __list_iterator{
    typedef _list_iterator<T, Ref, Ptr> self;
    /* 迭代器为算法提供了一下5种类型信息 */
    typedef bidirectioal_iterator_tag iterator_category;	/* 迭代器类型 */
    typedef T 	value_type;									/* 迭代器中的值类型 */
    typedef Ptr pointer;									/* 迭代器指针类型 */
    typedef Ref reference;									/* 迭代器的引用类型 */
    typedef ptrdiff_t difference_type;						/* 存储迭代器元素个数的类型 */
	...
}
/* 算法像一下这种方式从类类型迭代器中获取所需类型信息 */
template<typename I>
inline void algorithm(I first, I last)
{
    ...
    I::iterator_category	/* 获取迭代器中的类型信息 */
    I::value_type			/* 获取迭代器中的值类型 */
    I::pointer				/* 获取迭代器中指针类型 */
    I::reference			/* 获取迭代器中的引用类型 */
    I::difference_type		/* 获取迭代器中存储迭代器元素个数的类型*/
    ...
}

/* iterator_traits中间层,用于分离类类型迭代器和非类类型迭代器 */
template<class I>
struct literator_traits{	/* 获取类类型迭代器信息 */
    typedef typename I::iterator_category iterator_category;
    typedef typename I::value_type value_type;
    typedef typename I::difference_type difference_type;
    typedef typename I::pointer pointer;
    typedef typename I::refernce reference;
}
template<class T>
struct literator_traits<T*>{	/* 获取非类类型迭代器信息 */
    typedef random_access_iterator_tag iterator_category
    typedef T 			value_type;
    typedef ptrdiff_t 	difference_type;
    typedef T* 			pointer;
    typedef T&			reference
}    
template<class T>
struct literator_traits<const T*>{	/* 获取非类类型迭代器信息 */
    typedef random_access_iterator_tag iterator_category
    typedef T 			value_type;		/* 类型是用来声明变量的,所以此处是T,而不是const T */
    typedef ptrdiff_t 	difference_type;
    typedef T* 			pointer;
    typedef T&			reference;
}  
/* 算法可以通过中间层获取类型信息,而无需关注迭代器本身 */
template<typename I, ...>
void algorithm(...){
    typename iterator_traits<I>::value_type v1;
}
```

**容器vector：**

```c++
/* vector容器其本质是一个数组,迭代器是一个指针,GCC2.9Vector部分实现源码如下 */
template<class T， calss Alloc=alloc>
class vector {
public:
    typedef T			 	value_type;
    typedef value_type* 	iterator;		/* vector的迭代器类型是一个指针 */
    typedef value_type& 	reference;
    typedef size_t			size_type;
protected:
    iterator start;			/* 指向数组的起始地址的指针 */
    iterator finish;		/* 指向数组最后一个存储元素的下一个地址的指针 */
    iterator end_of_storage;/* 指向数组末尾的下一个元素的地址 */
public:
    iterator begin() { return start; }
    iterator end() { return finish; }
    size_type size() const { return size_type(end() - begin()); }
    siee_type capacity() const { return size_type(end_of_storage-begin()); }
    bool empty() const { return begin() == end(); }
    reference operator[](size_type n) { return *(begin()+n); }
    reference front() { return *begin(); }
    reference back() { return *(end()-1); }
    void push_back(const T& x) {
        if(finish != end_of_storage){ /* 还有备用存储空间 */
            construct(finish, x);	  /* 将新数据添加到尾部 */
            ++finish;				  /* 更新finish指针 */
        }else{						  /* 没有备用空间,需要扩充vector空间 */
            inser_aux(end(), x);
        }
    }
    void insert_aux(iterator position, const T& x);   /* 辅助函数,还可能被insert()函数调用 */
    ...
};

template<class T， class Alloc>
void vector<T,Alloc>::insert_aux(iterator position, const T& x)
{
    if(finish != end_of_storage){ /* vector还有备用存储空间 */
        construct(finish, *(finish-1));		/* 为insert()函数准备 */
        ++finish;
        T x_copy = x;
        copy_backward(position, finish-2, finish-1);
        *position = x_copy;
    }else{	/* 备用空间已使用完,需要扩充空间 */
        const size_type old_size = size();
        /* 原大小为0,分配1,否则分配为原来的两倍 */
        const size_type len = old_size != 0?2*old_size : 1; 
        
        iterator new_start = data_allocator::allocate(len);	 /* 使用分配器分配新内存 */
        iterator new_finish = new_start;
        try{
            /* 将原vector中的数据拷贝到新的vector内存中 */
            new_finish = uninitialized_copy(start, position,new_start);
            construct(new_finish, x);	/* 写入要插入或push_back进vector的新的元素值 */
            ++new_finish;
            /* 插入还需要拷贝后半段数据进新vector */
            new_finish = uninitialized_copy(position,finish,new_start);
        }catch(...){
            destory(new_start, new_finish);
            data_allocator::deallocate(new_start, len);
            throw;
        }
        destory(begin(), end());/* 析构原来的vector内存 */
        deallocate();
        /* 调整迭代器指针,指向新的内存 */
        start = new_start;
        finish = new_finish;
        end_of_storage = new_start+len;
    }
}

```

**typedef语法细节：**

```c++
int a[100];			//ok
int[100] b;			//fail
typedef int T[100];
T c;				//T类型为int[100]
```

**容器array:**

```c++
template<typename _Tp, std::size_t _Nm>  /* GCC2.9部分源码实现 */
struct array
{
    typedef _Tp 			value_type;
    typedef _Tp*			pointer;
    typedef value_type* 	iterator;	/* 迭代器是指针 */
    
    /* 内部维护一个数组 */
    value_type _M_instance[_Nm ? _Nm : 1];
    
    iterator begin() { return iterator(&_M_instance[0]); }
    iterator end() { return iterator(&_M_instance[_Nm]); }
    ...
}
/* 使用 */
array<int, 10> myarray;
auto ite = myarray.begin();
ite +=3;
cout << *ite;
```

**容器deque:**

```c++
/* GCC2.9源码部分实现 */
/* BufSize是指每一个buffer所能存储的元素个数,其分配规则是,
   如果BufSize有指定大小,则使用指定大小
   如果单个元素大小超过512字节,则BufSize=1,即一个buffer只存储一个元素
   如果单个元素大小小于512字节,则BufSize=512/单个元素大小 */
inline size_t __deque_buf_size(size_t n, size_t sz)
{
    return n != 0 ? n : (sz<512?size_t(512/sz):size_t(1));
}
/* deque内部存储结构:存在一个控制中心map,其实是一个vector数组容器,数据存储在vector中间,方便从两头扩展新的数据,vector中存储的数据是一个个指针,依次指向各个buffer存储块首地址,deque维护着两个迭代器，start和finish这两个迭代器里的node指针分别指向map控制中心vector中的第一个元素的地址和最后一个元素的下一个元素地址,两个迭代器里的first,last和cur指针,分别第一个和最后一个buffer存储块的首地址和最后一个元素的下一个地址，cur则指向第一个和最后一个元素的地址 */
template<class T, class Alloc=alloc, size_t BufSize=0 >
class deque{
public:
    typedef T value_type;
    typedef __deque_iterator<T, T&, T*, BufSize> iterator;
protected:
    typedef pointer* map_pointer;		/* pointer*是T** 类型 */
protected:
    iterator 	start;		/* 一个deque维护着两个迭代器 */
    iterator 	finish;	
    map_pointer map;
    size_type 	map_size;
public:
    iterator begin() { return start; }
    iterator end() { return finish; }
    size_type size() const { return finish-start; }
    reference operator[](size_type n) { return start[difference_type(n)]; }
    reference front() { return *start; }
    reference back() { iterator tmp = finish; --tmp; return *tmp; }
    size_type size() const { return finish - start; }
    bool empty() const { return finish == start; }
    iterator insert(iterator position, const value_type& x){ /* 在position处插入一个元素x */
        if(position.cur == start.cur){	/* 插入点就在deque的最前端 */
            push_front(x);
            return start;
        }else if(position.cur == finish.cur){ /* 插入点在deque的最尾端 */
            push_back(x);
            /* finish中的node指针指向控制中心vector中最后一个元素的下一个元素地址 */
            iterator tmp = finish; 
            --tmp;	/* 指向最后一个元素 */
            return tmp;  /* 返回最后一个元素的迭代器 */
        }else{
            return insert_aux(position,x)	/* 调用辅助函数完成插入动作 */
        }
    }
    ...
};

/* 插入辅助函数 */
template <class T, class Alloc, size_t BufSize>
typename deque<T, Alloc, BufSize>::iterator 
deque<T, Alloc, BufSize>::insert_aux(iterator pos, const value_type& x){
    difference_type index = pos -start;	/* 判断安插点是处在中间偏左还是偏右 */
    value_type x_copy = x;	
    if(index < size()/2){	/* 安插点之前的元素个数较少 */
        push_front(front());
        ...
        copy(front2,pos1,front1); /* 迁移元素 */
    }else{
        push_back(back());
        ...
        copy_backward(pos,back2,back1);
    }
    *pos = x_copy;	/* 插入元素 */
    return pos;
}

/* deque容器的迭代器 */
template<class T， class Ref, class Ptr, size_t BufSize>
struct __deque_iterator{
    typedef random_access_iterator_tag iterator_category; /* 获取迭代器中的类型信息 */
    typedef T	value_type;								  /* 获取迭代器中的值类型 */
    typedef Ptr pointer;								  /* 获取迭代器中的指针类型 */
    typedef Ref reference;								  /* 获取迭代器中的引用类型 */
    typedef ptrdiff_t difference_type;			/* 获取迭代器中存储迭代器元素个数的类型 */
    typedef size_t size_type;	
    typedef T** map_pointer;
    typedef __deque_iterator self;	
    
    T* cur;	 	/* 指向当前要操作的元素 */
    T* first;	/* 指向当前要操作元素所在buffer的起始地址 */
    T* last；   /* 指向当前要操作元素所在buffer的最后一个元素的下一个地址 */
    map_pointer node;	/* 指向当前要操作的buffer的首地址 */
    ...
    reference operator*() const { return *cur; }
    pointer operator->() const { return &(operator*()); }
    /* 两个迭代器之间的距离,是中间相隔完整的buffer的总长度+两头buffer的元素个数 */
    difference_type operator-(const self& x) const {
        return difference_type(buffer_size())*(node-x.node-1)+(cur-first)+(x.last-x.cur);
    }
    void set_node(map_pointer new_node){
        node = new_node; /* 将vector中的下一个元素地址作为新节点 */
        first = *new_node; /* 将first和last指针指向新的buffer区 */
        last = first + difference_type(buffer_size());
    }
    self& operator++(){ 	/* 前置++ */
        ++cur;
        if(cur == last){	 	/* 已经抵达改buffer区的尾端 */
            set_node(node+1);   /* 切换到下一个buffer */
            cur = first;
        }
        return *this;
    }
    self operator++(int){  /* 后置++ */
        self tmp = *this;
        ++*this;
        return tmp;
    }
    self& operator--() {	/* 前置-- */
        if(cur == first){
            set_node(node-1); /* 切换到前一个buffer */
            cur = last;
        }
        --cur;	/* 取得该buffer末尾的最后一个元素 */
        return *this;
    }
    self operator--(int){	/* 后置-- */
        self tmp = *this;
        --*this;
        return tmp;
    }
    self& operator+=(difference_type n){
        difference_type offset = n + ( cur - first);
        if(offset >= 0 && offset < difference_type(buffer_size())){
             cur += n;	/* 在同一个buffer中移动 */
        }else{	/* 在不同的buffer中移动 */
            difference_type node_offset = 
            	 offset > 0 ? offset/difference_type(buffer_size()):
            				 -difference_type((-offset-1)/buffer_size()) -1;
            set_node(node + node_offset);	/* 切换至正确的buffer区 */
            /* 切换至正确的元素 */
            cur = first+(offset-node_offset*difference_type(buffer_size())); 
        }
        return *this;
    }
    self operator+(difference_type n) const {
        self tmp = *this;
        return tmp += n;
    }
    self& operator-=(difference_type n) { return *this += -n; }
    self operator-(difference_type n) const { self tmp = *this; return tmp -= n; }
    reference operator[](difference_type n) const { return *(*this + n); }
};
```

**queue容器:**

```c++
/* 只是对deque容器的一种封装,内部默认维护着一个deque容器 */
template<class T, classn Sequence=deque<T>>  /* queue没有迭代器 */
class queue{
public:
    typedef typename Sequence::value_type value_type;
    typedef typename Sequence::size_type size_type;
    typedef typename Sequence::reference reference;
    typedef typename Sequence::const_reference const_reference;
protected:
    Sequence c;	/* 内部默认维护这一个deque容器 */
public:
    bool empty() const { return c.empty(); }
    size_type size() const {return c.size();}
    ...
}
```

**stack容器：**

```c++
/* 只是对deque容器的一种封装,内部默认维护着一个deque容器 */
template<class T, classn Sequence=deque<T>>  /* stack没有迭代器 */
class stack{
public:
    typedef typename Sequence::value_type value_type;
    typedef typename Sequence::size_type size_type;
    typedef typename Sequence::reference reference;
    typedef typename Sequence::const_reference const_reference;
protected:
    Sequence c;	/* 内部默认维护这一个deque容器 */
public:
    bool empty() const { return c.empty(); }
    size_type size() const {return c.size();}
    ...
}
```

**rb_tree容器：**

```c++
/* 红黑树中存储的Value值是key加data的组合数据包,Value值可以被修改,但这样会破坏红黑树的存储结构
   红黑树提供两种插入元素的方法:insert_unique(),存在的元素不插入,和insert_equal(),存在的元素,可重复    插入 */
template <class key, 		/* 存储的Key值 */
    	  class Value,  	/* key加data的组合 */
          class KeyOfValue,	/* 如何通过一个value得到一个key */
          class Compare,	/* 如何对key进行比较 */
          class Alloc=alloc>
class rb_tree{
protected:
    typedef __rb_tree_node<Value> rb_tree_node;
    ...
public:
    typedef rb_tree_node* link_type;
    ...
protected:
    size_type node_count;	/* 记录着rb_tree的大小 */
    link_type header;		/* 纪律着红黑树的头 */
    Compare key_compare;	/* 比较key的仿函数 */
    ...
}

template<class Arg, class Result>
struct unary_function{
    typedef Arg 	argument_type
    typedef Result 	result_type;
}
template<class T>
struct identity:public unary_function<T,T>{
    const T& operator()(const T&  x) const { return x; }	
}

template<class Arg1, class Arg2, class Result>
struct binary_function{
   typedef Arg1 first_argument_type;
   typedef Arg2 second_argument_type;
   typedef Result result_type;
}
template<class T>
struct less:public binary_function<T,T,bool>{
    bool operator()(const T& x, const T& y) const{ return x<y; }
}
/* 红黑树的使用 */
rb_tree<int, int, identity<int>, less<int>> itree;
cout << itree.empty() << endl;		// 1
cout << itree.size() << endl;		// 0
itree.insert_unique(3);	/* 插入的值是Value,Value是key和data的组合 */
itree.insert_equal(5);
```

**set,multiset容器:**

```c++
/* set和multiset底层存储结构使用的红黑树,通过使用const_iterator来禁止对元素的修改
   set使用的是insert_unique(),来实现元素的独一无二
   multiset使用的是insert_equal(),来实现元素的重复 */
template<class Key,	/* key+data组合成Value */
		 class Compare = lsee<key>,
	     class Alloc = alloc>
class set{
public:
    typedef Key key_type;
    typedef Key value_type;
    typedef Compare key_compare;
    typedef compare value_compare;
private:
    typedef rb_tree<key_type,value_type,identity<value_type>,key_compare,Alloc> rep_type;
    rep_type t;	/* set容器内部维护着一个红黑树 */
public:
    typedef typename rep_type::const_iterator iterator;	/* set使用的是const迭代器 */
    ...
    /* set的所有操作都是通过红黑树来完成的,所有set更像是一个adapter */
};
```

**map，multimap容器:**

```c++
/* map和multimap也是以红黑树为底部存储结构,同过将key设为const属性来禁止修改key,但可以修改data
   map使用的是insert_unique(),来实现元素的独一无二,operator[]是map独有的操作
   multimap使用的是insert_equal(),来实现元素的重复 */
template<class Key,	/* key */
		 class T,	/* data */
		 class Compare = less<key>, /* 如何比较key值 */
         class Alloc=alloc>
class map{
public:
    typedef Key key_type;
    typedef T 	data_type;
    typedef T   mapped_type;
    typedef pair<const Key, T> value_type; /* 通过设置Key为const属性,来禁止修改Key */
    typedef Compare key_compare;
private:
    typedef 
        rb_tree<key_type,value_type,select1st<value_type>,key_compare,Alloc> rep_type;
    rep_type t;		/* 内部维护一个红黑树 */
public:
    typedef typename rep_type::iterator iterator;	/* 使用红黑树的普通迭代器 */
    ...
}
/* 使用 */
map<int, string,less<int>, alloc> imap;
```

**hashtable容器：**

```c++
/* 底层实现原理:假设我们现在有一个M大小的数组容器,还要一个hashcode()生成算法,任何数据经过hashcode()算法,都会生成一个唯一的哈希码H,对H%M即可得到该数据要存储的数组下标,当H足够多变时,则H%M可能得相同的下标,即发生数据碰撞，所以这个数组中应该存储的是一个个链表(一堆篮子),发生碰撞时将数据挂入相同链表中即可(放入同一个篮子),但是依然存储单一链表过长而降低效率的可能,所以规定,当哈希表中存储的元素"总数"大于数组元素个数(篮子总数),则需要扩充数组大小(增加篮子总数),(GCC2.9将数组扩充到原来大小的2倍附近的质数),所以此处的数组应该是一个vector容器,扩充后应该对哈希表内部所有元素进行重新计算和存放 */
template<class Value>
struct __hashtable_node{
    __hashtable_node* next; /* 这里是一个单向链表 */
    Value val;				/* 存储着数据 */
}
template<class Value,class Key,class HashFcn,
		 class ExtractKey,class EqualKey,class Alloc=alloc>
struct __hashtable_iterator{
    ...
    node* cur;		/* 指向当前操作的链表中的结点 */
    hashtable *ht;	/* 指向整个哈希表的vector容器 */
}
template<class Value, 		/* 要存入哈希表的数据 */
		 class Key, 		/* 哈希表是根据key值来进行元素排列的 */
		 class HashFcn，	   /* hashcode生成算法 */
         class ExtractKey,  /* 如何通过Value值得到key值 */
		 class EqualKey,	/* 如何判断连个Key值相等 */
         class Alloc=alloc>
class hashtbale{
public:
    typedef HashFcn hasher;
    typedef EqualKey key_equal;
    typedef size_t size_type;
private:
    hasher hash;
    key_equal equals;
    ExtractKey get_key;
    
    typedef __hashtable_node<Value> node;	/* 链表节点 */
    vector<node*, Alloc>buckets;	/* 内部维护着一个vector容器,容器中存储的是一个个链表 */
    size_type num_elements;			/* 记录着哈希表中的元素总数 */
public:
    size_type bucket_count() const { return buckets.size(); }
    ...
}
/* 使用 */
hashtable<const char*,
    	  const char*,
     	  hash<const char*>,
    	  identity<const char*>,
		  eqstr,alloc> ht(50,hash<const char*>(),eqstr())); /* 构造函数 */
ht.insert_unique("hello");

struct eqstr{
    bool operator()(const char* s1, const char* s2) const { return strcmp(s1,s2)==0; }
}
inline size_t __st1_hash_string(cosnt char* s){ 	/* hashcode生成算法 */
    unsigned long h = 0;
    for(;*s;++s) { h=5*h+*s;}
    return size_t(h);
}
template<class key>			/* 模板的部分特化 */
struct hash<const char*>{
    size_t operator()(const char* s) const { return __st1_hash_string(s); }
}
```

**C++标准库的算法:函数模板**

```c++
template<typename iterator>
Algorithm(iterator itr1, iterator itr2)	 /* c++标准库算法的形式 */
{
    ...
}
```

**iterator的种类：五种iterator category**

```c++
/* 除了输出迭代器,其他迭代器都是相互继承的 */
struct Input_iterator_tag{ ... };
struct output_itertaor_tag{ ... };
/* 单向迭代器,如:单向链表 */
struct forward_iterator_tag:public input_iterator_tag { ... }; 
/* 双向迭代器,如：双向链表 */
struct bidirectional_iterator_tag:public forward_iterator_tag{ ... };
/* 随机访问迭代器 */
struct random_access_iterator_tag:public bidirectional_iterator_tag{ ... }	
/* 使用typeid可获取迭代器种类 */
#include <typeinfo>	//typeid

iterator itr;
cout << typeid(itr).name(); << endl;  /* 取得迭代器的种类 */
```

**迭代器种类对算法效率的影响：**

```c++
/* 非随机访问迭代器的重载版本 */
template<class InputIterator>
inline iterator_traits<InputIterator>::difference_type
__distance(InputIteratorertor first, InputIterator last, Input_iterator_tag){
    ierator_traits<InputIterator>::difference_type n = 0;
    while(first != last){	/* 依次遍历每一个节点 */
        ++first;
        ++n;
    }
    return n;
}
/* 随机访问迭代器版本 */
template<class RandomAccessIterator>
inline iterator_traits<RandomAccessIterator>::difference_type
__distance(RandomAccessIterator first,RandomAccessIterator 					                        last,random_access_iterator_tag){
    return last-first;
}
/* 计算两个迭代器之间的距离 */
template<class InputIterator> /* 除了输出迭代器,其他迭代器都继承自InputIterator */
inline iterator_traits<InputIterator>::difference_type
distance(InputIterator first, InputIterator last){
    typedef typename iterator_traits<InputIterator>::iterator_category category;
    return __distance(first,last,category())  /* 迭代器种类临时对象 */
}
```

```c++
template<class InputIterator, class Distance> 
inline void __advance(InputIterator& i, Distance n, input_iterator_tag){ 
    while(n--){ ++i; } /* 迭代器只能往前走 */
}    
template<class BidirectionalIterator, class Distance>
inline void __advance(BidirectionalIterator& i, Distance n, bidirectional_iterator_tag){
    if(n > 0){	/* 双向迭代器,即可以往前走也可以往后走 */
        while(n--) { ++i; }
    }else{
        while(n++) { --i; }
    }
}
template<class RandomAccessIterator, class Distance>
inline void __advance(RandomAccessIterator& i, Distance n, random_access_iterator_tag){
    i += n;	/* 随机访问迭代器,随意操作 */
}

template <class itertaor>
inline typename iterator_traits<iterator>::iterator_category
iterator_category(const iterator&){ /* 编译器自动推导参数类型 */
    typedef typename iterator_traits<Iterator>::itertaor_category category;
    return category(); /* 返回临时对象 */
}
/* 将迭代器往前偏移n */
template<class InputItertaor, class Distance>
inline void advance(InputItertaor& i, Distance n){
    __advance(i,n,iterator_category(i));
}
```

**常见算法实现:**

```c++
/* 算法accumulate */
template<class InputIterator, class T>
T accumulate(Inputiterator first, Inputiterator last, T init){
    for(;first != last; ++ first){
        init = init + *first;
    }
    return init;
}
/* 算法accumulate,带仿函数版本 */
template<class InputIterator,class T, class BinaryOperation>
T accumulate(InputIterator first, InputIterator last, T init, BinaryOperation binary_op){
    for(;first != last; ++first){
        init = binary_op(init, *first);
    }
    return init;
}
```

```c++
/* 算法for_each */
template<class InputIterator, class Function>
Function for_each(InputIterator first,InputIterator last, Function F){
    for(;first != last; ++first){
        f(*first);
    }
    return f;
}
```

```c++
/* 算法replace,replace_if,replace_copy */
template<class ForwardIterator, class T>
void replace(ForwardIterator first,ForwardIterator last,
             const T& old_value,const T& new_value){
    for(;first!=last; ++first){ /* 将所有的old_value替换为new_value */
        if(*first == old_value){
            *first = new_value;
        }
    }
}
template<class InputIterator,class OutputIterator,class T>
OutputIterator replace_copy(InputIterator first,InputIterator last,OutputIterator result, 							  const T& old_value,const T& new_value){
    for(;first!=last;++first;++result){ 
        *result = *first==old_value?new_value:*first;
    }
    return result;
}
template<class ForwardIterator, class Predicate, class T>
void replace_if(ForwardIterator first,ForwardIterator last,
                Predicate pred,const T& new_value){
    for(;first!=last;++first){
        if(pred(*first)){
            *first = new_value;
        }
    }
}
```

```c++
/* 算法count,count_if */
template<class InputIterator, class T>
typename iterator_traits<InputIterator>::difference_type
count(InputIterator first, InputIterator last, const T& value){
    for(;first!=last; ++first){
        if(*first == value){
			++n;
        }
    }
    return n;
}
template<class InputIterator, class Predicate>
typename iterator_traits<InputIterator>::difference_type
count_if(InputIterator first, InputIterator last, Predicate pred){
    for(;first!=last；++first){
        if(pred(*first)){
            ++n;
        }
    }
    return n;
}
```

```c++
/* 算法find, find_if */
template<class InputIterator, class T>
InputIterator find(InputIterator first, InputIterator last, const T& value){
    while(first!=last && *first != value){
        ++first;
    }
    return first;  
}
template<class InputIterator, class Predicate>
InputIterator find_if(InputIterator first, InputIterator last, Predicate pred){
    while(first!=last && !(pred(*first))){
        ++first;
    }
    return first;
}
```

```c++
/* 算法binary_search,调用该算法的前提是容器内的元素排序是一个递减序列 */
template<class ForwardIterator, class T>
bool binary_search(ForwardIterator first,ForwardIterator last，
                  const T& val){
    first = std::lower_bound(first,last,val);	/* 查找到第一个小于等于val的值 */
    return (first!=last && !(val < *first)); 
}
/* 在一个递减序列中查找第一个小于等于val的元素 */
template<class ForwardIterator, class T>
ForwardIterator lower_bound(ForwardIterator first, ForwardIterator last,
                            const T& val){
    ForwardIterator it;
    iterator_traits<ForwardIterator>::difference_type count, step;
    count = distance(first,last);
    while(count > 0){
        it = first;
        step = count/2;
        advance(it,step);
        if(*it < val){	
            first = ++it;
            count -= step+1;
        }else{
            count = step;
        }
    }
    return first;
}
```

**仿函数functors：**

```c++
/* 仿函数 */
template<class T>
struct identity:public unary_function<T,T>{
    const T& operator()(const T& x) const { return x; }
};
template<class T1， class T2>
struct pair{
    typedef T1 first_type;
    typedef T2 second_type;
    
    T1 first;
    T2 second;
    pair():first(T1()),second(T2()){};
    pair(const T1& a, const T2& b):first(a),second(b){}；
}
template<class Pair>
struct select1st:public unary_function<Pair,typename Pair::first_type>{
    const typename Pair::first_type& operator()(const Pair& x) const{
        return x.first;
    }
}
template<class Pair>
struct select2nd:public unary_function<Pair,typename Pair::second_type>{
    const typename Pair::second_type& operator()(const Pair& x) const{
        return x.second;
    }
}
```

**仿函数的可适配（adaptable）:**

```c++
/* 仿函数如果想被STL的适配器适配,必须继承以下两个类之一,因为STL的适配器需要通过这些继承的类来获取仿函数的参数类型信息,方便定义变量和使用 */
template<class Arg, class Result> /* 一个传入参数 */
struct unary_function{
    typedef Arg 	argument_type;
   	typedef Result  result_type;
}
template<class Arg1, class Arg2, class Result> /* 两个传入参数 */
struct binary_function{
    typedef Arg1 	first_argument_type;
    typedef Arg2 	second_argument_type;
    typedef Result 	result_type;
}
/* 仿函数通过继承的方式,将参数类型信息传递给继承的类,来给STL的适配器使用 */
template<class T>
struct less:public binary_function<T,T,bool>{
    bool operator()(const T& x， const T& y) const {
        return x < y;
    }
};
```

**容器的适配器：**

```c++
template<class T,class Sequence=deque<T>>
class stack{
public:
    typedef typename Sequence::value_type value_type;
    typedef typename Sequence::size_type size_type;
    typedef typename Sequence::reference reference;
    typedef typename Sequence::const_reference const_reference;
protected:
    Sequence c; /* 内部维护着一个其他容器 */
public:
    /* 内部仅仅是对其他容器功能的一个封装 */
    bool empty() const { return c.empty(); }
    size_type size() const { return c.size(); }
    reference top() { return c.back(); }
    const_reference top() const { return c.back(); }
    void push(const value_type& x) { c.push_back(x); }
    void pop() { c.pop_back(); }
}
```

**函数的适配器bind2nd:**

```c++
template<class Operation, class T>
inline binder2nd<Operation>bind2nd(const Operation& op, const T& x){
    typedef typename Operation::second_argument_type arg2_type;
    return binder2nd<Operation>(op,arg2_type(x)); /* 返回一个匿名对象 */
} 
template<class Operation>
class binder2nd:public unary_function<typename Operation::first_argument_type,
									  typename Operation::result_type>{
protected:
    Operation op;	/* 这里相当于创建了一个对象,这里创建对象会调用构造函数 */		
    typename Operation::second_argument_type value;    /* 将传入的两个参数记录起来 */	   
public:
    /* 构造函数 */
    binder2nd(const Operation& x, const typename Operation::second_argument_type& y)
        :op(x),value(y){
            
        };
    typename Operation::result_type
    Operation()(const typename Operation::first_argument_type& x) const{
        return op(x,value);
    };
};
/* count_if算法 */
template<class InputIterator, class Predicate>
typename Iterator_traits<InputIterator>::difference_type 
count_if(InputIterator first, InputIterator last, Predicate pred){
    typename iterator_traits<InputIterator>::difference_type n = 0;
    for(;first !=last){	/* 遍历迭代器 */
        if(pred(*first)){
            ++n;
        }
    }
    return n;
}
/* 使用 */
cout<< count_if(vi.begin(), vi.end(),bind2nd(less<int>(),40));
```

**函数适配器not1:**

```c++
template<class Predicate>
inline unart_negate<Predicate> not1(const Predicate& pred){ /* 模板函数 */
    return unary_negate<Predicate>(pred); /* 返回一个匿名对象 */
}

template<class Prediacte>
class unary_negate:public unary_function<typename Predicate::argument_type,bool>{
protected:
    Predicate pred;	/* 会调用默认的空构造函数 */
public:
    explicit unary_negate(const Predicate& x):pred(x){ /* 拷贝构造函数 */
        
    }
    bool operator()(const typename Predicate::argument_type& x) const {
        return !pred(x); /* 适配器仅仅是起到装饰作用,指示对其他类功能的一种修饰 */
    }
}
/* 使用 */
cout<< count_if(vi.begin(), vi.end(),not1(bind2nd(less<int>(),40));
```

**迭代器适配器:**

```c++
template<class Iterator>
class reverse_iterator{ /* 仅仅只是对正向迭代器功能的一种装饰 */
protected:
    Iterator current;	/* 维护着一个正向迭代器 */
public:
    typedef typename iterator_traits<Iterator>::iterator_category iterator_category;
    typedef typename iterator_traits<Iterator>::value_type value_type;
    typedef Iterator iterator_type;				/* 代表正向迭代器 */
    typedef reverse_iterator<Iterator> self;	/* 代表逆向迭代器 */
public:
    explicit reverse_iterator(iterator_type x):current(x){ }
    reverse_iterator(const self& x):current(x.current){ }
    iterator_type base() const { return current; } 
    /* 对正向迭代器的反向操作 */
    reference operator*() const { Iterator tmp = current; return *--tmp }
    pointer operator->() const { return &(operator*()); }
    self& operator++(){ --current; return *this; }
    self& operator--(){ ++current; return *this; }
    self operator+(difference_type n) const { return self(current -n); }
    self operator-(difference_type n) const { return self(current +n); }
}
/* 使用 */
reverse_iterator rbegin(){ return reverse_iterator(end()); } /* 传入正向迭代器 */
reverse_iterator rbend(){ return reverse_iterator(begin()); } /* 传入正向迭代器 */
```

```c++
template<class InputIterator, class OutputIterator>
OutputIterator copy(InputIterator first,InputIterator last,OutputIterator result){
    while(first!=last){
        *result = *first;	/* result这个类可以重载operator=(),改变=操作行为 */
        ++result; ++first;
    }
    return result;
}
template<class Container>
class insert_iterator{	/* 这个迭代器适配器改变迭代器的赋值行为 */
protected:
    Container* container;
    typename Container::iterator ite;
public:
    typedef output_iterator_tag iterator_category;
    /* 构造函数 */
    insert_iterator(Container& x,typename Container::iterator):container(&x),iter(i){ }
    ...
    insert_iterator<Container>& operator=(const typename Container::value_type& val){
        iter = container->insert(iter,value); /* 将容器的赋值操作改为插入操作 */
        ++iter;
        return *this;
    } 
};
/* 使用 */
list<int>foo,bar;
for(int i = 1; i <= 5; i++){
    foo.push_back(i);
    bar.push_back(i*10);  
}
list<int>::iterator it = foo.begin();
copy(bar.begin(),bar.end(),insert(foo,it)); /* 改变了copy的行为 */
```

```c++
template<class InputIterator, class OutputIterator>
OutputIterator copy(InputIterator first,InputIterator last,OutputIterator result){
    while(first!=last){
        *result = *first;	/* result这个类可以重载operator=(),改变=操作行为 */
        ++result; ++first;
    }
    return result;
}
template<class T, class charT = char, class traits=char_traits<charT>>
class ostream_iterator:public iterator<output_itertaor_tag,void,void,void,void>{
    basic_osteram<charT,traits>* out_stream; /* 使用的输出流 */
    const charT* delim;			/* 每次输出的分隔符 */
public:
    typedef charT char_type;
    typedef traits traits_type;
    typedef basic_ostream<charT,traits>ostream_type;
    /* 构造函数 */
    ostream_iterator(ostream_type& s):out_stream(&s),delim(0){ }
    ostream_iterator(ostream_type& s, const charT* delimiter)
        :out_stream(&s),delim(delimiter){ }
    /* 拷贝构造函数 */
    ostream_iterator(const ostream_iterator<T,charT,traits>& x)
        :out_stream(x.out_stream),delim(x.delim){ }
    ~ostream_iterator(){ }
    /* 将赋值行为适配到输出到out_stream */
    ostream_iterator<T,charT,traits>& operator=(const T& value){ 
        *out_stream << value;
        if(delim != 0){
            *out_stream << delim;
        }
        return *this;
    }
    ostream_iterator<T,charT,traits>& operator*() { return *this; }
    ostream_iterator<T,charT,traits>& operator++() { return *this; } /* 前置++ */
    ostream_iterator<T,charT,traits>& operator++(int) { return *this; } /* 后置++ */
}
/* 使用 */
int main(){
    std::vector<int> myvector;
    for(int i=1;i<10;i++){
        myvector.push_back(i*10);
    }
    std::ostream_iterator<int> out_it(std::cout,",");
    std::copy(myvector.begin(),myvector.end(),out_it); 
    
    return 0;
}
```

```c++
template<class InputIterator, class OutputIterator>
OutputIterator copy(InputIterator first,InputIterator last,OutputIterator result){
    while(first!=last){
        *result = *first;	/* *first操作重载版本operator*() */
        ++result; 			
        ++first;			/* 完成一次读操作 */
    }
    return result;
}
template<class T,class charT=char,
		 class traits=char_traits<charT>,class Distance=ptrdiff_t>
class istream_iterator:public iterator<input_iterator_tag,T,Distance,const T*,const T&>
{
    basic_istream<charT,traits>* in_stream;
    T value;
public:
    typedef charT  char_type;
    typedef traits traits_type;
    typedef basic_istream<charT,traits> istream_type;
    istream_iterator():in_stream(0){}
    istream_iterator(istream_type& s):in_stream(&s){++*this;} /* ++操作已经产生一次读操作 */
    istream_iterator(const isteram_iterator<T,charT,traits,Distance>& x)
        :in_stream(x.in_stream),value(x.value){ }
    ~istream_iterator(){}
    const T& operator*() const { return value }
    const T* operator->() const { return &value }
    istream_iterator<T,charT,traits,Distance>operator++(){ /* 前置++ */
        if(in_stream && !(*in_stream >> value)){ /* 已经完成一次读操作 */
            ins_stream=0；
        }
        return *this;
    }
    istream_iterator<T,charT,traits,Distance> operator++(int){ /* 后置++ */
        istream_iterator<T,charT,traits,Distance> tmp = *this;
        ++*this;
        return tmp;
    }
}
/* 使用 */
istream_iterator<int>iit(cin),eos;
copy(ite,eos,inserter(c,c.begin()));
```

**不定模板参数的使用:**

```c++
/* 利用不定模板参数实现万用的HashCode函数 */
class CustomerHash{
public:
    std::size_t operator()(const Customer& c) const{
        return hash_val(c.fname,c.lname,c.no);
    }
private:
    string fname;
    string lname;
    long no;
}
/* 不定参数模板函数也是可以重载的 */
template<typename... Type>
inline size_t hash_val(const Type&... arg){
    size_t seed = 0;
    hash_val(seed,arg...);
    return seed; /* seed就是最终生成的hashcode */
}
template<typename T，typename... Type>
inline void hash_val(size_t& seed, const T& val, const Type&... arg){
    hash_combine(seed,val); /* 逐一取出模板参数并计算hashcode */
    hash_val(seed,arg...);
}
template<typename T>
inline void hash_val(size_t& seed,const T& val){
    hash_combine(seed,val);
}
template<typename T>
inline void hash_combine(size_t& seed, const T& val){
    seed ^=std::hash<T>()(val)+0x9e3779b9+(seed<<6)+(seed>>2);
}
```

**tuple任意个数类型组合：**

```c++
/* 不定模板参数+自我递归完成可容纳任意类型任意数量组合的类tuple */
template<> class tuple<>{ }; 				/* 空模板参数的特化版本,用于递归结束条件 */
template<typename Head,typename... Tail>
class tuple<Head,Tail...>:private tuple<Tail...>  /* 通过继承完成不定模板参数的分离 */
{
   	typedef tuple<Tail..>inherited;
public:
    tuple(){ }
    tuple(Head v,Tail... vtail):m_head(v),inherited(vtail...){ } /* 不断的自我递归 */
    
    typename Haed::type head(){ return m_head; }
    inherited& tail(){ return *this; }
protected:
    Head m_head; /* 被分离的参数 */
}
/* 使用 */
tuple<int,float,string> t(41,63,"nico");
t.head() /* 取得41 */
t.tail() /* 取得41,63,nico这个数据结构的首 */
t.tail().head() /* 取得6.3 */  
```

**moveable class:**

```c++
class MyString{
public:
    static size_t DCtor;	/* 默认构造函数调用次数 */
    static size_t Ctor;		/* 构造函数调用次数 */
    static size_t CCtor;	/* 拷贝构造函数调用次数 */
    static size_t CAsgn;	/* 拷贝赋值函数调用次数 */
    static size_t MCtor;	/* move构造函数 */
    static size_t MAsgn;	/* move赋值函数 */
    static size_t Dtor;		/* 析构函数 */
private:
    char* _data;
    size_t _len;
    void _init_data(const char* s){
        _data = new char[_len+1];
        memcpy(_data,s,_len);
        _data[_len] = '\0';
public:
     /* 默认构造函数 */
     MyString():_data(NULL),_len(0){
         ++DCtor;
     }
     /* 普通构造函数 */
     MyString(const char* p):_len(strlen(p)){
         ++Ctor;
         _init_data(p);
     }
     /* 拷贝构造函数 */
     MyString(const MyString& str):_len(str._len){
         ++CCtor;
         _init_data(str._data);
     }
     /* 拷贝赋值函数 */
     MyString& operator=(const MyString& str){
         ++CAsgn;
         if(this != &str){
             if(_data) delete _data;
             _len = str._len;
             _init_data(str._data);	/* 深拷贝 */
         }else{
             
         }
         return *this;
     }
     /* move构造函数 */
     MyString(MyString&& str) noexcept:_data(str._data),_len(str._len){
         ++MCtor;	/* 需要确保传入的对象不在会被使用 */
         str._len = 0;
         str._data = NULL;
     }
     /* move赋值函数 */
     MyString& operator=(MyString&& str) noexcept {
         ++MAsgn;
         if(this != &str){ /* 需要确保传入的对象不在会被使用 */
             if(_data) delete _data;
             _len = str._len;
             _data = str._data;
             str.len = 0;
             str._data = null;
		 }
         return *this;
     }
     /* 虚析构函数 */
     virtual ~MyString(){
         if(_data){
             delete _data;
         }
     }
     bool operator<(const MyString& rhs) const{
         return std::string(this->_data) < std::string(rhs._data);
     }
     bool operator==(const MyString& rhs) const {
          return std::string(this->_data) = std::string(rhs._data);
     }
};
size_t MyString::DCtor = 0;
size_t MyString::Ctor = 0;
size_t MyString::CCtor = 0;    
size_t MyString::CAsgn = 0;
size_t MyString::MCtor = 0;
size_t MyString::MAsgn = 0;
size_t MyString::Dtor = 0;
/* 使用 */
std::move(c1)	/* 调用move构造函数 */
```

#### 设计模式:

**面向对象设计原则:**

​		**依赖倒置原则**: 高层**稳定**模块,不应该依赖于低层**易变**的模块,且二者皆应该依赖于抽象层(**稳定**)

​								 抽象层(**稳定**)不应该依赖于具体实现(变化),具体实现应该依赖于抽象(稳定)

​		**开放封闭原则**:对扩展开放,对更改封闭(类模块应该是可扩展(继承),但不可修改)

​		**单一职责原则**:一个类应该隐含着**一个**引起它变化的原因(变化方向应该隐含着类的**责任**)

​		**替换原则:**		子类应该能够替换父类(is a的关系)

​		**接口隔离原则**:**接口应该小而完备**,不需要被使用的接口应该尽量不对外暴露

​		**优先使用对象组合**而不是继承: 继承在某种程度上破坏了封装性,子父类耦合度高

​															对象的组合则只需要被组合对象具有良好的接口定义,耦合度低

​		**封装变化点**: 使用封装来创建对象之间的分界层，让设计者可以在分界层的一侧进行修改，而不会对另一侧产							  生不良的影响，**一侧稳定,一侧变化**,从而实现层次间的松耦合						

​		**针对接口(抽象层)编程**，而不是针对实现编程: 

​							   不将变量类型声明为某个特定的具体类，而是声明为某个接口(抽象层)。

​							   客户程序无需获知对象的具体类型，只需要知道对象所具有的接口(使用抽象层所具有的接口)

​						       减少系统中各部分的依赖关系，从而实现“高内聚、松耦合” 的类型设计方案

##### 设计模式分类:

**组件协作:**通过晚绑定来实现框架与应用程序之间的松耦合

​		**Template Method:**

```c++
/* 应用背景:在某一项任务中,常常有一些固定的操作步骤和流程,但又有部分子步骤的需求常常会发生改变 */
class Library{ /* 应用程序库开发人员,可能很早以前就写好的库 */
public:
    void Run(){  /* 存在较为稳定的执行流程和步骤 */
        Step1();
        if (Step2()) { 	//支持变化 ==> 虚函数的多态调用
            Step3(); 
        }
        for (int i = 0; i < 4; i++){
            Step4(); 	//支持变化 ==> 虚函数的多态调用
        }
        Step5();
    }
	virtual ~Library(){ ... }
protected:
	/* 稳定不变的执行步骤 */
	void Step1() { ... }	
	void Step3() { ... }
	void Step5() { ... }
    /* 易变化的执行步骤,让子类继承,对其进行重写 */
	virtual bool Step2() = 0;
    virtual void Step4() =0; 
};

/* 现在的应用程序开发人员,继承Library并对Library中易变函数进行重写 */
class Application : public Library {
protected:
	virtual bool Step2(){
		//... 子类重写实现
    }
    virtual void Step4() {
		//... 子类重写实现
    }
};

int main(){
	    Library* pLib=new Application(); 
	    lib->Run(); /* 你不要调用我,让我来调用你 */
		delete pLib;
	}
}
```

​		**Strategy:**

```c++
/* 应用背景:在一些任务中,某些对象可能在不同的情况或场景下需要使用不同的算法,如果将这些算法用if.else之类的方式都编码到对象中,将会使得对象本身变得复杂,且相对于不常被调用到的if.else分支,是一种性能负担 */
class TaxStrategy{ /* 计税算法抽象类,易变化的算法应该继承于稳定的抽象类 */
public:
    virtual double Calculate(const Context& context)=0;
    virtual ~TaxStrategy(){}
};
class CNTax : public TaxStrategy{ /* 不同地区的计税算法 */
public:
    virtual double Calculate(const Context& context){ ... }
};
class USTax : public TaxStrategy{ /* 不同地区的计税算法 */
public:
    virtual double Calculate(const Context& context){ ... }
};
class DETax : public TaxStrategy{ /* 不同地区的计税算法 */
public:
    virtual double Calculate(const Context& context){ ... }
};
/* 后期业务变化,新增的计税算法 */
class FRTax : public TaxStrategy{ 
public:
	virtual double Calculate(const Context& context){ ... }
};

class SalesOrder{ /* 稳定的高层模块 */
private:
    TaxStrategy* strategy;	/* 维护着抽象类指针,准备多态的产生 */
public:
    SalesOrder(StrategyFactory* strategyFactory){ /* 根据具体情况创建对应的计税算法类 */
        this->strategy = strategyFactory->NewStrategy();
    }
    ~SalesOrder(){
        delete this->strategy;
    }
    public double CalculateTax(){
        Context context(); 
        double val = strategy->Calculate(context); /* 多态调用 */
        /* ... */
    }
};
```

​		**Observer/Event:**

```c++
/* 应用背景:在某一些任务中,我们需要为对象建立一种"通知依赖关系",即目标对象发生改变时其所对应的依赖对象(观察者对象)将得到通知,且这种依赖关系不能过于紧密 */
class IProgress{ /* 抽象类,稳定的模块应该于抽象层 */
public:
	virtual void DoProgress(float value)=0;
	virtual ~IProgress(){}
};
/* 文件分割功能模块,稳定的高层模块 */
class FileSplitter { /* 订阅中心,数据发送变化时即会通知观察者 */
	string m_filePath;
	int m_fileNumber;
	/* 这里应该是一个抽象类基类,订阅者只需要继承并重写抽象类子类,即可被订阅中心通知调用 */
	List<IProgress*>  m_iprogressList; /* 订阅者列表 */
public:
	FileSplitter(const string& filePath, int fileNumber) : 
		m_filePath(filePath), 
		m_fileNumber(fileNumber){ /* 构造 */
	}
	void split(){ /* 文件分割功能函数 */
		/* 1.读取大文件 */
		/* 2.分批次向小文件中写入 */
		for (int i = 0; i < m_fileNumber; i++){
			/* ... */
			float progressValue = m_fileNumber;
			progressValue = (i + 1) / progressValue;
            /* ... */
			onProgress(progressValue); /* 发送通知,依次调用订阅者功能函数 */
		}
	}
    /* 向订阅列表添加新的订阅者 */
	void addIProgress(IProgress* iprogress){
		m_iprogressList.push_back(iprogress);
	}
    /* 从订阅列表删除订阅者 */
	void removeIProgress(IProgress* iprogress){
		m_iprogressList.remove(iprogress);
	}
protected:
	virtual void onProgress(float value){ /* 发送通知,虚函数是继续方便被子类重写 */		
		List<IProgress*>::iterator itor=m_iprogressList.begin();
		while (itor != m_iprogressList.end()) /* 遍历订阅列表 */
			(*itor)->DoProgress(value);  	  /* 依次调用订阅者功能函数 */
			itor++;
		}
	}
};

class MainForm : public Form, public IProgress
{
	TextBox* txtFilePath;
	TextBox* txtFileNumber;
	ProgressBar* progressBar;
public:
	void Button1_Click(){
		string filePath = txtFilePath->getText();
		int number = atoi(txtFileNumber->getText().c_str());
        FileSplitter splitter(filePath, number);
		ConsoleNotifier cn;
		splitter.addIProgress(this); /* 添加订阅者 */
		splitter.addIProgress(&cn)； /* 添加订阅者 */
		splitter.split();
		splitter.removeIProgress(this);
	}
    /* 订阅者只需要继承抽象类,并重写对应的功能函数,即可被调用中心通知调用 */
	virtual void DoProgress(float value){
		progressBar->setValue(value);
	}
};
/* 订阅者只需要继承抽象类,并重写对应的功能函数,即可被调用中心通知调用 */
class ConsoleNotifier : public IProgress {
public:
	virtual void DoProgress(float value){
		cout << ".";
	}
};
```

**单一职责:**类责任划分不清,盲目使用继承,导致子类数量急剧膨胀,出现大量重复代码

​		**Decorator:**

```c++
/* 应用背景:为了“扩展子类的功能”,过度使用继承,随着扩展子类的增加,各种扩展子类的组合,导致子类数量急剧膨胀 ,盲目继承扩展子类,易违背一个类的单一职责的设计原则 */
class Stream{ 
public：
    virtual char Read(int number)=0;
    virtual void Seek(int position)=0;
    virtual void Write(char data)=0;
    virtual ~Stream(){}
};
/* 业务操作 */
class FileStream: public Stream{ /* 与Stream类职责方向一致 */
public:
    virtual char Read(int number){ ... }	/* 读文件流 */
    virtual void Seek(int position){ ... }  /* 定位文件流 */
    virtual void Write(char data){ ... }    /* 写文件流 */
};
class NetworkStream :public Stream{ /* 与Stream类职责方向一致 */
public:
    virtual char Read(int number){ ... } 	/* 读网络流 */
    virtual void Seek(int position){ ... } 	/* 定位网络流 */
    virtual void Write(char data){ ... } 	/* 写网络流 */  
};
class MemoryStream :public Stream{ /* 与Stream类职责方向一致 */
public:
    virtual char Read(int number){ ... }   /* 读内存流 */
    virtual void Seek(int position){ ... } /* 定位内存流 */
    virtual void Write(char data){ ... }   /* 写内存流 */
};
/* 装饰模式:在现有功能基础上添加一些额外的功能,功能与适配器相似 */
DecoratorStream: public Stream{ /* 继承是为了加入继承体系,这样才能继续被其他类装饰 */
protected:
    Stream* stream; /* 维护一个抽象类指针,在产生多态时才能调用现有基础功能 */
    DecoratorStream(Stream * stm):stream(stm){
    } 
};
/* 扩展的加密功能 */
class CryptoStream: public DecoratorStream { /* 与Stream类职责方向不一致,扩展功能方向 */
public:
    CryptoStream(Stream* stm):DecoratorStream(stm){ ... }
    virtual char Read(int number){
        /* 额外的加密操作... */
        stream->Read(number);  /* 读文件流 */
    }
    virtual void Seek(int position){
        /* 额外的加密操作... */
        stream::Seek(position); /* 定位文件流 */
        /* 额外的加密操作... */
    }
    virtual void Write(byte data){
        /* 额外的加密操作... */
        stream::Write(data);	/* 写文件流 */
        /* 额外的加密操作... */
    }
};
/* 扩展的缓存功能 */
class BufferedStream : public DecoratorStream{ /* 与Stream类职责方向不一致,扩展功能方向 */
public:
    BufferedStream(Stream* stm):DecoratorStream(stm){ ... }    
	/* 同上类似 */
};
void Process(){
    /* 运行时装配,各种扩展功能可相互组合 */
    FileStream* s1=new FileStream(); 			/* 读文件流 */
    CryptoStream* s2=new CryptoStream(s1);  	/* 读文件流+加密 */
    BufferedStream* s3=new BufferedStream(s1);	/* 读文件流+缓存 */
    BufferedStream* s4=new BufferedStream(s2);  /* 读文件流+缓存+加密 */
}
```

​		**Bridge:**

```c++
/* 应用背景:由于某些类型的固有实现逻辑,使得其具有两个甚至多个维度的变化,此时我们需要将其继续抽象分离,使得它们可以相互独立的变化,最终通过抽象类指针进行桥接组合 */
class Messager{ /* 未优化前:原抽象类型 */
public:
    /* 业务实现相关方向 */
    virtual void Login(string username, string password)=0;
    virtual void SendMessage(string message)=0;
    virtual void SendPicture(Image image)=0;
	/* 平台实现相关方向 */
    virtual void PlaySound()=0;
    virtual void DrawShape()=0;
    virtual void WriteText()=0;
    virtual void Connect()=0;
    virtual ~Messager(){}
};
/* 优化后:将存在不同方向职责的抽象类继续分离 */
class Messager{ /* 业务逻辑实现相关抽象类 */
protected:
     MessagerImp* messagerImp; /* 通过该抽象类指针,进行两个变化方向上的桥接 */
public:
    virtual void Login(string username, string password)=0;
    virtual void SendMessage(string message)=0;
    virtual void SendPicture(Image image)=0;
    virtual ~Messager(){}
};
class MessagerImp{ /* 平台实现逻辑相关抽象类 */
public:
    virtual void PlaySound()=0;
    virtual void DrawShape()=0;
    virtual void WriteText()=0;
    virtual void Connect()=0;
    virtual MessagerImp(){}
};
/* 平台实现相关 */
class PCMessagerImp : public MessagerImp{
public:
    virtual void PlaySound(){ ... }
    virtual void DrawShape(){ ... }
    virtual void WriteText(){ ... }
    virtual void Connect(){ ... }
};
class MobileMessagerImp : public MessagerImp{
public:
    virtual void PlaySound(){ ... }
    virtual void DrawShape(){ ... }
    virtual void WriteText(){ ... }
    virtual void Connect(){ ... }
};
/* 业务实现相关 */
class MessagerLite:public Messager { /* 利用平台实现相关抽象类指针进行桥接 */
public:  
    virtual void Login(string username, string password){
        messagerImp->Connect();
        /* ... */
    }
    virtual void SendMessage(string message){  
        messagerImp->WriteText();
        /* ... */
    }
    virtual void SendPicture(Image image){ 
        messagerImp->DrawShape();
        /* ... */
    }
};
class MessagerPerfect:public Messager { /* 利用平台实现相关抽象类指针进行桥接 */
public: 
    virtual void Login(string username, string password){
        messagerImp->PlaySound();
        /* ... */
        messagerImp->Connect();
        /* ... */
    }
    virtual void SendMessage(string message){
        messagerImp->PlaySound();
        /* ... */
        messagerImp->WriteText();
        /* ... */
    }
    virtual void SendPicture(Image image){
        messagerImp->PlaySound();
        /* ... */
        messagerImp->DrawShape();
        /* ... */
    }
};
void Process(){
    //运行时装配
    MessagerImp* mImp=new PCMessagerImp();
    Messager *m =new MessagerPerfect(mImp); /* 注意:上面省略了构造函数 */
}
```

**对象创建:**通过间接的对象创建,来绕开new,以此解开抽象(稳定)与实现(易变)之间的耦合

​		**Factory Mrthod:**

```c++
/* 应用背景:在软件系统中,由于需求的变化,需要创建的对象的具体类型经常需要变化 */
class ISplitter{ /* 文件分割抽象类 */
public:
    virtual void split()=0;
    virtual ~ISplitter(){}
};
/* 具体类 */
class BinarySplitter : public ISplitter{ ... }; /* 二进制文件分割 */
class TxtSplitter: public ISplitter{ ... } 		/* TXT文本分割 */
class PictureSplitter: public ISplitter{ ... } 	/* 图片分割 */
class VideoSplitter: public ISplitter{ ... } 	/* 视频分割 */

class SplitterFactory{ /* 工厂抽象类 */
public:
    virtual ISplitter* CreateSplitter()=0;
    virtual ~SplitterFactory(){}
};
/* 稳定部分 */
class MainForm : public Form
{
    SplitterFactory*  factory;	/* 工厂 */
public:
    MainForm(SplitterFactory*  factory){ /* 传入具体工厂类 */
        this->factory=factory;
    }
	void Button1_Click(){    
        //ISplitter* splitter= new BinarySplitter();/* 不应该依赖于具体实现,应该依赖于抽象 */
		ISplitter* splitter= factory->CreateSplitter(); /*依赖于抽象,利用多态来new具体类 */
        splitter->split();
	}
};
/* 具体工厂 */
class BinarySplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new BinarySplitter(); /* 创建二进制文件分割对象 */
    }
};
class TxtSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){ 
        return new TxtSplitter(); 	/* 创建TXT文本分割对象 */
    }
};
class PictureSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new PictureSplitter();/* 创建图片分割对象 */
    }
};
class VideoSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new VideoSplitter();  /* 创建视频分割对象 */
    }
};
```

​		**Abstract Factory:**

```c++
/* 应用背景:在软件系统中,经常存在多个一系列相互依赖的对象需要被创建 */
/* 抽象工厂模式也可以理解为家族工厂 */

/* 数据库访问有关的基类 */
class IDBConnection{ ... }; /* 一系列具有相互依赖性的抽象基类 */
class IDBCommand{ ... };
class IDataReader{ ... };
/* 支持SQL Server */	
class SqlConnection: public IDBConnection{ ... }; /* 具体实现 */
class SqlCommand: public IDBCommand{ ... };
class SqlDataReader: public IDataReader{ ... };
/* 支持Oracle */
class OracleConnection: public IDBConnection{ ... }; /* 具体实现 */
class OracleCommand: public IDBCommand{ ... };
class OracleDataReader: public IDataReader{ ... };
/* 家族工厂抽象类,将具有相互依赖的一系列抽象类进行一起封装 */
class IDBFactory{ 
public:
    virtual IDBConnection* CreateDBConnection()=0;
    virtual IDBCommand* CreateDBCommand()=0;
    virtual IDataReader* CreateDataReader()=0;
};
/* 抽象工厂具体实现类 */
class SqlDBFactory:public IDBFactory{ 
public:
    virtual IDBConnection* CreateDBConnection(){ ... }; /* new SqlConnection */
    virtual IDBCommand* CreateDBCommand(){ ... }; 		/* new SqlCommand */
    virtual IDataReader* CreateDataReader(){ ... }; 	/* new SqlDataReader */
};
class OracleDBFactory:public IDBFactory{ 
public:
    virtual IDBConnection* CreateDBConnection(){ ... }; /* new OracleConnection */
    virtual IDBCommand* CreateDBCommand(){ ... }; 		/* new OracleCommand */
    virtual IDataReader* CreateDataReader(){ ... }; 	/* new OracleDataReader */
};

class EmployeeDAO{ /* 利用家族工厂抽象类将一系列具有相关性的抽象类进行封装 */
    IDBFactory* dbFactory;
public:
    vector<EmployeeDO> GetEmployees(){
        //IDBConnection* connection = new SqlConnection(); /* 抽象不应该依赖于实现 */
        IDBConnection* connection = dbFactory->CreateDBConnection(); /* 稳定 */
        connection->ConnectionString("...");
        //IDBCommand* command = new SqlCommand(); /* 抽象不应该依赖于实现 */
        IDBCommand* command = dbFactory->CreateDBCommand(); /* 稳定 */
        command->CommandText("...");
        command->SetConnection(connection); 
        IDBDataReader* reader = command->ExecuteReader(); 
        while (reader->Read()){ ... }
    }
};
```

​		**Prototype:    **

```c++
/* 应用背景:由于需求的变更,某些具有稳定一致接口的复杂对象会产生一定变化,存在想保存这种状态,并创建具有该状态的对象的需求 */  
class MainForm : public Form
{
    ISplitter*  prototype; /* 原型对象 */
public:
    MainForm(ISplitter*  prototype){
        this->prototype=prototype; /* 创建对象但不使用对象本身,只使用对象的克隆版本 */
    }
	void Button1_Click(){
		ISplitter * splitter=prototype->clone(); /* 只使用克隆版本,间接保存了原对象的状态 */   
        splitter->split();
	}
};
class ISplitter{ /* 文件分割抽象类 */
public:
    virtual void split()=0;
    virtual ISplitter* clone()=0; /* 通过克隆自己来创建对象 */
    virtual ~ISplitter(){ ... }
};
/* 具体实现类 */
class BinarySplitter : public ISplitter{
public:
    virtual ISplitter* clone(){
        return new BinarySplitter(*this); /* 利用类的构造函数实现对自身的克隆 */
    }
};
class TxtSplitter: public ISplitter{
public:
    virtual ISplitter* clone(){
        return new TxtSplitter(*this); /* 利用类的构造函数实现对自身的克隆 */
    }
};
class PictureSplitter: public ISplitter{
public:
    virtual ISplitter* clone(){
        return new PictureSplitter(*this); /* 利用类的构造函数实现对自身的克隆 */
    }
};
class VideoSplitter: public ISplitter{
public:
    virtual ISplitter* clone(){
        return new VideoSplitter(*this); /* 利用类的构造函数实现对自身的克隆 */
    }
};
```

​		**Builder:**

```c++
/* 应用背景:在软件系统中经常面临着复杂对象的创建,由于需求变化,该复杂对象的各个部分经常面临着剧烈变化,但将其各个部分相互组合在一起时要求相对稳定 */
/* 将复杂对象的创建和表示相互抽离,将创建过程逐步(易变)分解后在组合(稳定) */
class House{ /* 复杂对象本身 */
    /* 复杂对象的表示 */
};
class HouseBuilder { /* 将复杂对象的构建部分进行抽离 */
public:
    House* GetResult(){
        return pHouse;
    }
    virtual ~HouseBuilder(){}
protected:
    House* pHouse;	/* 组合着复杂对象表示部分 */
	virtual void BuildPart1()=0;
    virtual void BuildPart2()=0;
    virtual void BuildPart3()=0;
    virtual void BuildPart4()=0;
    virtual void BuildPart5()=0;
};
class StoneHouse: public House{ ... }; /* 复杂对象表示部分的抽象实现 */
class StoneHouseBuilder: public HouseBuilder{ /* 复杂对象构键部分的抽象实现 */
protected: 
    virtual void BuildPart1(){
        //pHouse->Part1 = ...;
    }
    virtual void BuildPart2(){ ... }
    virtual void BuildPart3(){ ... }
    virtual void BuildPart4(){ ... }
    virtual void BuildPart5(){ ... }
};
class HouseDirector{  /* 对象初始化部分过于复杂,继续进行抽离 */  
public:
    HouseBuilder* pHouseBuilder;  /* 组合着复杂对象的构建部分 */
    HouseDirector(HouseBuilder* pHouseBuilder){
        this->pHouseBuilder=pHouseBuilder;
    }    
    House* Construct(){  /* 复杂对象初始化部分,稳定 */      
        pHouseBuilder->BuildPart1();        
        for (int i = 0; i < 4; i++){
            pHouseBuilder->BuildPart2();
        }
        bool flag=pHouseBuilder->BuildPart3();
        if(flag){
            pHouseBuilder->BuildPart4();
        }
        pHouseBuilder->BuildPart5();      
        return pHouseBuilder->GetResult();
    }
};
/* 使用 */
House* house = new StoneHouse(); /* 将复杂对象的稳定和不稳定部分相互进行分解 */
HouseBuilder* housebuilder = new StoneHouseBuilder(house)
HouseDirector tmp = HouseDirector(housebuilder)
```

**对象性能:**当一个只读对象被创建成千上万个时,其所带来的性能代价将不可忽视

​		**Singleton:**

```c++
/* 应用背景:在软件系统中,经常有一些特殊类,要求其在系统中只存在一个唯一实例,才能保证其逻辑的正确性及良好的效率 */
class Singleton{
private: /* 将构造函数私有化,即可阻止外部对象的创建 */
    Singleton();
    Singleton(const Singleton& other);
public:
    static Singleton* getInstance(); /* 通过静态函数获取唯一的对象实例 */
    static Singleton* m_instance;
};
Singleton* Singleton::m_instance=nullptr;
/* 线程非安全版本 */
Singleton* Singleton::getInstance() {
    if (m_instance == nullptr) { 
        m_instance = new Singleton(); 
    }
    return m_instance;
}
/* 线程安全版本，但锁的代价过高 */
Singleton* Singleton::getInstance() {
    Lock lock; /* 线程锁,伪代码 */ 
    if (m_instance == nullptr) { /* 仅第一次判断是有意义的,之后的判断都是性能消耗 */
        m_instance = new Singleton();
    }
    return m_instance;
}
/* 双检查锁，但由于内存读写reorder不安全 */
Singleton* Singleton::getInstance() {
    if(m_instance==nullptr){ /* 消减了锁的代价 */
        Lock lock; /* 线程锁,伪代码 */
        if (m_instance == nullptr) {
           /* 由于编译器优化的原因,对象创建可能是先返回地址,后初始化,未被初始化的对象,不应该被使用 */
            m_instance = new Singleton(); 
        }
    }
    return m_instance;
}
/* C++ 11版本之后的跨平台实现 (volatile),解决reorder问题 */
std::atomic<Singleton*> Singleton::m_instance;
std::mutex Singleton::m_mutex;
Singleton* Singleton::getInstance() {
    Singleton* tmp = m_instance.load(std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_acquire);/* 获取内存fence */
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            std::atomic_thread_fence(std::memory_order_release);/* 释放内存fence */
            m_instance.store(tmp, std::memory_order_relaxed);
        }
    }
    return tmp;
}
```

​		**Flyweight:**

```c++
/* 应用背景:在软件系统中,经常存在大量类似于完全相同的只读对象,如:字符串,如果采用纯粹的对象创建的方式,将会付出很高的内存代价,采样共享技术,是一个很好的解决方案 */
class Font { /* 在一个系统中,同样的字体对象可能会在多处被使用 */
private:
    //unique object key
    string key;
    //object state
public:
    Font(const string& key){ ... }
};
class FontFactory{
private:
    map<string,Font* > fontPool; /* 共享对象池 */
public:
    Font* GetFont(const string& key){
        map<string,Font*>::iterator item=fontPool.find(key);  
        if(item!=footPool.end()){ /* 从对象池中查找对象,如果存在则返回查找到的对象 */
            return fontPool[key];
        }else{ /* 如果对象池中没有,则需要重新创建并返回 */
            Font* font = new Font(key);
            fontPool[key]= font; /* 将新创建的对象放入对象池 */
            return font;
        }
    }
    void clear(){ ... }
};
```

**接口隔离:**在软件构建过程中,接口之间的直接依赖会带来很多问题,添加一个中间接口层(稳定),是常见的解决方案

​		**Facade:**

```c++
/* 应用背景:需要简化客户程序和系统组件之间的交互接口,为系统组件提供一个稳定一致的对外接口(稳定),将系统组件内对象之间的复杂依赖(变化)限定到系统组件内 */
/* 泛泛的整体概念,无代码实例 */
```
​		**Mediator:**

```c++
/* 应用背景:在系统组件内,经常会存在多个相互关联的对象,且这些对象常常会维持着复杂的引用关系,由于需求的变更,这些关系面临着不断的变化,可以使用一个中介者类来封装这些有复杂关系且变化的类,将这些复杂的引用关系维持在中介中,具体的类在使用时传入中介者即可 */
/* 泛泛的整体概念,无代码实例 */
```

​		**Proxy:**

```c++
/* 应用背景:在面向对象系统中,由于某些原因,直接访问会给使用者或系统带来很多问题,如:对象创建开销大,操作需要安全控制,进程需要对外远程访问等,需要提供一个中间层接口来进行访问控制 */
class ISubject{
public:
    virtual void process();
};
/* 实际类 */
class RealSubject: public ISubject{
public:
    virtual void process(){ ... }
};
/* Proxy的设计 */
class SubjectProxy: public ISubject{
public:
    virtual void process(){
        /* 对RealSubject的一种间接访问 */
        /* ... */
    }
};
class ClientApp{
    ISubject* subject;
public:
    ClientApp(){
        //subject=new RealSubject(); /* 直接使用 */
        subject=new SubjectProxy(); /* 通过中间层间接访问 */
    }
    void DoTask(){
        /* ... */
        subject->process();
        /* ... */
    }
};
```

​		**Adapter:**

```c++
/* 应用背景:由于需求的变化,我们常常需要将一些现有的旧类库放到新环境中使用,然而旧的类库与新环境所要求的接口又有所不同,所以需要针对旧类库的接口进行适配 */
class ITarget{ 	/* 目标接口(新接口) */
public:
    virtual void process()=0;
};
class IAdaptee{ /* 遗留接口(老接口) */
public:
    virtual void foo(int data)=0;
    virtual int bar()=0;
};
class OldClass: public IAdaptee{ ... }; /* 遗留旧的类型 */
/* 对象适配器 */
class Adapter: public ITarget{ /* 继承 */
protected:
    IAdaptee* pAdaptee; /* 通过组合来实现对旧类的适配 */
public:
    Adapter(IAdaptee* pAdaptee){
        this->pAdaptee=pAdaptee;
    }
    virtual void process(){
        int data=pAdaptee->bar();
        pAdaptee->foo(data);   
    }
};
/* 类适配器 */
class Adapter: public ITarget,
               protected OldClass{ /* 多继承 */       
/* 通过继承来实现对类的适配,不建议使用,耦合性太高 */
}
/* 使用 */
int main(){
    IAdaptee* pAdaptee=new OldClass(); 		/* 旧类 */
    ITarget* pTarget=new Adapter(pAdaptee); /* 针对旧类进行新接口的适配 */
    pTarget->process();
}
```

**状态变化:**

​		**Memento:**在组件构建过程中,某些对象的状态经常发生改变,如何管理这些状态的变化,同时又维护高层模块的稳定

```c++
/* 应用背景:在软件构建过程中,由于某种需求,要求程序对象能够回溯到之前某一个时刻的状态,如果让对象对外提供公共的获取状态的接口,则会破坏对象的封装性 */
class Memento /* 在对象之外保存的对象状态 */
{
    string state;
    /* ... */
public:
    Memento(const string & s) : state(s) { }
    string getState() const { return state; }
    void setState(const string & s) { state = s; }
};
class Originator /* 原对象,需要被保存的对象 */
{
    string state; /* 对象状态就是对象的数据 */
    /* ... */
public:
    Originator() {}
    Memento createMomento() { 
        m = new Memento(state); /* 在不破环封装的情况下在对象之外保存对象状态 */
        return m;
    }
    void setMomento(const Memento & m) {
        state = m.getState();
    }
};
int main()
{
    Originator orginator;
    Memento mem = orginator.createMomento();  /* 捕获对象状态，存储到备忘录 */
    /* ... 改变orginator状态 */
    orginator.setMomento(memento);  /* 从备忘录中恢复对象状态 */
}
```

​		**State:**

```c++
/*  应用背景:在软件构建过程中,某些对象的状态发生改变,其行为也会随之发生改变,应将对象的状态和行为封装成一个对象,并保留统一的接口,来实现对象状态的转换和行为变化之间的解耦合 */
/* 使用状态模式解耦合之前 */
enum NetworkState{ /*状态和其对应的行为操作,随需求变化可能有所变化 */
    Network_Open,
    Network_Close,
    Network_Connect,
};
class NetworkProcessor{  
    NetworkState state;
public:
    void Operation1(){ /* 状态的变化会导致行为的变化 */
        if (state == Network_Open){
            /* ... */
            state = Network_Close;
        }else if (state == Network_Close){
			/* ... */
            state = Network_Connect;
        }else if (state == Network_Connect){
            /* ... */
            state = Network_Open;
        }
    }
    public void Operation2(){
        if (state == Network_Open){
            /* ... */
            state = Network_Connect;
        }else if (state == Network_Close){
			/* ... */
            state = Network_Open;
        }else if (state == Network_Connect){
			/* ... */
            state = Network_Close;
        }
    }
    public void Operation3(){ ... }
};
/* 使用state模式将状态变化和行为变化解耦合 */
class NetworkState{ /* 状态和行为的抽象基类 */
public:
    NetworkState* pNext;
    virtual void Operation1()=0;
    virtual void Operation2()=0;
    virtual void Operation3()=0;
    virtual ~NetworkState(){}
};
class OpenState :public NetworkState{ /* 将状态和行为封装成对象 */   
    static NetworkState* m_instance;
public:
    static NetworkState* getInstance(){ /* 状态没有实例对象,各级上下文可共享一个对象 */
        if (m_instance == nullptr) {
            m_instance = new OpenState();
        }
        return m_instance;
    }
    void Operation1(){     
        /* ... */
        pNext = CloseState::getInstance();
    }  
    void Operation2(){       
        /* ... */
        pNext = ConnectState::getInstance();
    }    
    void Operation3(){      
        /* ... */
        pNext = OpenState::getInstance();
    }   
};
class CloseState:public NetworkState{ ... } 
class NetworkProcessor{   
    NetworkState* pState;   /* 使用统一的抽象接口 */
public:   
    NetworkProcessor(NetworkState* pState){
        this->pState = pState;
    }   
    void Operation1(){
        /* ...*/
        pState->Operation1();
        pState = pState->pNext;
        /* ...*/
    }    
    void Operation2(){
        /* ...*/
        pState->Operation2();
        pState = pState->pNext;
        /* ...*/
    }
    void Operation3(){
        /* ...*/
        pState->Operation3();
        pState = pState->pNext;
        /* ...*/
    }
};
```

**数据结构:**常常有一些组件在其内部具有特定的数据结构,直接使用将破坏其复用性,将特定数据结构封装在内部,对外提供统一的接口,来实现与数据结构无关的访问,是一种行之有效的解决方式

​		**Composite:**

```c++
/* 应用背景:复杂对象容器内部结构可能存在多个组成部分,通过让用户程序对单个部分和组合部分的操作具有一致性,来解耦用户程序与对象容器内部数据结构的依赖 */
class Component /* 抽象基类 */
{
public:
    virtual void process() = 0;
    virtual ~Component(){}
};
/* 树节点 */
class Composite : public Component{ /* 组合部分 */
    string name;
    list<Component*> elements;	/* 既可接收单个部分也可以接收组合部分 */
public:
    Composite(const string & s) : name(s) {}
    void add(Component* element) {
        elements.push_back(element);
    }
    void remove(Component* element){
        elements.remove(element);
    }
    void process(){   
        /* 1. process current node */
        /* 2. process leaf nodes */
        for (auto &e : elements)
            e->process(); /* 多态调用 */ 
    }
};
/* 叶子节点 */
class Leaf : public Component{ /* 单个部分 */
    string name;
public:
    Leaf(string s) : name(s) {}   
    void process(){
        /* process current node */
    }
};
/* 使用 */
int main()
{
    /* 创建节点 */
    Composite root("root");
    Composite treeNode1("treeNode1");
    Composite treeNode2("treeNode2");
    Composite treeNode3("treeNode3");
    Composite treeNode4("treeNode4");
    Leaf leat1("left1");
    Leaf leat2("left2");
    /* 添加节点 */
    root.add(&treeNode1);
    treeNode1.add(&treeNode2);
    treeNode2.add(&leaf1);
    root.add(&treeNode3);
    treeNode3.add(&treeNode4);
    treeNode4.add(&leaf2);
    process(root);
    process(leaf2);
    process(treeNode3); 
}
```

​		**Iterator:**

```c++
/* 应用背景:在软件的构建过程中,集合对象的内部结构常常变化各异,对于这种集合对象,我们希望在不暴露其内部结构的同时,让外部用户可以操作其内部元素 */
template<typename T>	/* 迭代器抽象类 */
class Iterator /* 迭代器使用多态会降低性能 */
{
public:
    virtual void first() = 0;
    virtual void next() = 0;
    virtual bool isDone() const = 0;
    virtual T& current() = 0;
};
template<typename T>  /* 集合对象类 */
class MyCollection{ 
public:
    Iterator<T> GetIterator(){ ... }  /* 返回集合的迭代器 */
};
template<typename T>  /* 集合迭代器 */
class CollectionIterator : public Iterator<T>{
    MyCollection<T> mc; /* 维护着一个集合 */
public:
    CollectionIterator(const MyCollection<T> & c): mc(c){ }
    void first() override { ... }
    void next() override { ... }
    bool isDone() const override{ ... }
    T& current() override{ ...}
};
void MyAlgorithm() /* 算法 */
{
    MyCollection<int> mc;   
    Iterator<int> iter= mc.GetIterator();   
    for (iter.first(); !iter.isDone(); iter.next()){
        cout << iter.current() << endl;
    }
}
```

​		**Chain of Resposibility:**

```c++
/* 应用背景:在软件构建过程中,一个请求可能会被多个对象中的一个处理,如果显式指定,必然导致发送者和接收者之间的紧耦合,可以将这些对象连成一条链,并沿着这条链传递请求,直到有一个对象处理它为止 */
enum class RequestType /* 请求的类型 */
{
    REQ_HANDLER1,
    REQ_HANDLER2,
    REQ_HANDLER3
};
class Reqest /* 请求类 */
{
    string description;
    RequestType reqType;
public:
    Reqest(const string & desc, RequestType type) : description(desc), reqType(type) {}
    RequestType getReqType() const { return reqType; }
    const string& getDescription() const { return description; }
};
class ChainHandler{ /* 职责链抽象基类 */
    ChainHandler *nextChain; /* 单向链表,指向下一个 */
    void sendReqestToNextHandler(const Reqest & req){ /* 向下传递请求 */
        if (nextChain != nullptr){
            nextChain->handle(req);
        }   
    }
protected:
    virtual bool canHandleRequest(const Reqest & req) = 0;
    virtual void processRequest(const Reqest & req) = 0;
public:
    ChainHandler() { nextChain = nullptr; } /* 构造 */
    void setNextChain(ChainHandler *next) { nextChain = next; }
    void handle(const Reqest & req){ /* 处理请求 */
        if (canHandleRequest(req)){
            processRequest(req);
        }else{
            sendReqestToNextHandler(req);
        }
    }
};
class Handler1 : public ChainHandler{ 
protected:
    bool canHandleRequest(const Reqest & req) override{ /* 判断是否是自己需要处理的类 */
        return req.getReqType() == RequestType::REQ_HANDLER1;
    }
    void processRequest(const Reqest & req) override{
        cout << "Handler1 is handle reqest: " << req.getDescription() << endl;
    }
};        
class Handler2 : public ChainHandler{
protected:
    bool canHandleRequest(const Reqest & req) override{ /* 判断是否是自己需要处理的类 */
        return req.getReqType() == RequestType::REQ_HANDLER2;
    }
    void processRequest(const Reqest & req) override{
        cout << "Handler2 is handle reqest: " << req.getDescription() << endl;
    }
};
class Handler3 : public ChainHandler{
protected:
    bool canHandleRequest(const Reqest & req) override{ /* 判断是否是自己需要处理的类 */
        return req.getReqType() == RequestType::REQ_HANDLER3;
    }
    void processRequest(const Reqest & req) overrid
    {
        cout << "Handler3 is handle reqest: " << req.getDescription() << endl;
    }
};
/* 使用 */
int main(){
    Handler1 h1;
    Handler2 h2;
    Handler3 h3;
    h1.setNextChain(&h2); /* 构建职责链 */
    h2.setNextChain(&h3);
    Reqest req("process task ... ", RequestType::REQ_HANDLER3);
    h1.handle(req); /* 处理请求,不是自己的类型则继续往下传递 */
    return 0;
}
```

**行为变化:**在软件构建过程中,组件行为变化经常导致组件本身的变化,则需要将组件的行为和组件本身进行解耦合

​		**Command:**

```c++
/* 应用背景:在软件构建过程中,行为的请求者和实现者之间通常呈现紧耦合关系,但是在某些场景下需要对行为进行如:记录,撤销,重做,事务等操作,则需要将"行为对象化",以此来实现请求者和实现者之间的松耦合 */
class Command /* 行为抽象基类 */
{
public:
    virtual void execute() = 0;
};
class ConcreteCommand1 : public Command /* 行为1 */
{
    string arg;
public:
    ConcreteCommand1(const string & a) : arg(a) {}
    void execute() override{
        cout<< "#1 process..."<<arg<<endl;
    }
};
class ConcreteCommand2 : public Command /* 行为2 */
{
    string arg;
public:
    ConcreteCommand2(const string & a) : arg(a) {}
    void execute() override{
        cout<< "#2 process..."<<arg<<endl;
    }
};
class MacroCommand : public Command /* 行为3,组合行为 */
{
    vector<Command*> commands; /* 既可以接收自身,又可以接收行为1,2 */
public:
    void addCommand(Command *c) { commands.push_back(c); }
    void execute() override{
        for (auto &c : commands){
            c->execute();
        }
    }
};
/* 使用 */
int main()
{
    ConcreteCommand1 command1(receiver, "Arg ###"); 
    ConcreteCommand2 command2(receiver, "Arg $$$");
    MacroCommand macro;
    macro.addCommand(&command1);
    macro.addCommand(&command2);
    macro.execute();
}
```

​		**Visitor:**

```c++
/* 应用背景:在软件构建中,存在某种需求,类的整体层次结构稳定不变,但其中的操作需要频繁的变化 */
/* 整体类层次结构稳定不变 */
class Element /* 抽象基类 */
{
public:
    virtual void accept(Visitor& visitor) = 0; /* 第一次多态辨析 */
    virtual ~Element(){}
};
class ElementA : public Element
{
public:
    void accept(Visitor &visitor) override { 
        visitor.visitElementA(*this);  /* 第二次多态辨析 */
    }
};
class ElementB : public Element
{
public:
    void accept(Visitor &visitor) override {
        visitor.visitElementB(*this); /* 第二次多态辨析 */
    }

};
class Visitor{ /* 抽象基类稳定不变 */
public:
    virtual void visitElementA(ElementA& element) = 0;
    virtual void visitElementB(ElementB& element) = 0;
    virtual ~Visitor(){}
};
/* 需要频繁变化的操作行为 */
class Visitor1 : public Visitor{ /* 重写整个类层次结构中的所以行为 */
public:
    void visitElementA(ElementA& element) override{ 
        cout << "Visitor1 is processing ElementA" << endl;
    }    
    void visitElementB(ElementB& element) override{
        cout << "Visitor1 is processing ElementB" << endl;
    }
};
/* 需要频繁变化的操作行为 */
class Visitor2 : public Visitor{ /* 重写整个类层次结构中的所以行为 */
public:
    void visitElementA(ElementA& element) override{
        cout << "Visitor2 is processing ElementA" << endl;
    }
    void visitElementB(ElementB& element) override{
        cout << "Visitor2 is processing ElementB" << endl;
    }
};     
/* 使用 */
int main()
{
    Visitor2 visitor;
    ElementB elementB;
    elementB.accept(visitor); /* 多态的两次分发 */
    ElementA elementA;
    elementA.accept(visitor); /* 多态的两次分发 */
    return 0;
}
```

**领域问题:**在某些特定领域中存在某些频繁变化,但可以通过将其抽象为确定的语法规则,从而提供一般的通用性解决方案的情况

​		**Interpreter:**

```c++
/* 应用背景:在某一特定领域中,存在类似的结构不断出现,且频繁变化的问题,可以将这一变化抽象为特定规则,并构建一个解析器来统一解析 */
class Expression { /* 表达式抽象基类 */
public:
    virtual int interpreter(map<char, int> var)=0;
    virtual ~Expression(){}
};
class VarExpression: public Expression { /* 变量表达式 */
    char key;
public:
    VarExpression(const char& key){
        this->key = key;
    }
    int interpreter(map<char, int> var) override {
        return var[key];
    }
};
class SymbolExpression : public Expression { /* 符号表达式提取基类 */
protected:  /* 运算符左右两个参数 */
    Expression* left;
    Expression* right;
public:
    SymbolExpression( Expression* left,  Expression* right):
        left(left),right(right){ }
};
class AddExpression : public SymbolExpression { /* 加法运算 */
public:
    AddExpression(Expression* left, Expression* right):
        SymbolExpression(left,right){ }
    int interpreter(map<char, int> var) override {
        return left->interpreter(var) + right->interpreter(var);
    }  
};
class SubExpression : public SymbolExpression { /* 减法运算 */
public:
    SubExpression(Expression* left, Expression* right):
        SymbolExpression(left,right){ }
    int interpreter(map<char, int> var) override {
        return left->interpreter(var) - right->interpreter(var);
    }
};
Expression* analyse(string expStr) { /* 规则解析器 */
    stack<Expression*> expStack; 
    Expression* left = nullptr;
    Expression* right = nullptr;
    for(int i=0; i<expStr.size(); i++){
        switch(expStr[i]){
            case '+':  /* 加法运算 */
                left = expStack.top();
                right = new VarExpression(expStr[++i]);
                expStack.push(new AddExpression(left, right));
                break;
            case '-': /* 减法运算 */
                left = expStack.top();
                right = new VarExpression(expStr[++i]);
                expStack.push(new SubExpression(left, right));
                break;
            default: /* 变量表达式 */
                expStack.push(new VarExpression(expStr[i])); /* 将表达式不断压入堆栈 */
        }
    }
    Expression* expression = expStack.top(); /* 栈顶就是最终计算结果的表达式 */
    return expression;
}
void release(Expression* expression){ ... } /* 释放表达式树的节点内存... */
/* 使用 */
int main(int argc, const char * argv[]) {
    string expStr = "a+b-c+d-e"; 	/* 一定语法规则的表达式 */
    map<char, int> var;
    var.insert(make_pair('a',5));
    var.insert(make_pair('b',2));
    var.insert(make_pair('c',1));
    var.insert(make_pair('d',6));
    var.insert(make_pair('e',10));
    Expression* expression= analyse(expStr); /* 解析器解析 */
    int result=expression->interpreter(var);  
    cout<<result<<endl;
    release(expression);  
    return 0;
}
```

### UML

**类与类之间的关系:6种**

**泛化关系:**(is a: cat is a animal) ->继承

​		用实线空心三角箭头指向基类,表示继承关系(虚线用于表示注释)

**实现关系:**(like a: cooker liker a foodmenu) ->继承抽象类并实现

​		用虚线空心三角箭头指向抽象基类或接口

**关联关系:**(has a: programmer has a computer) -> 内部组合,维护着另一个类的指针

​		用实线箭头指向内部所维护的类变量

​		实线所连接的两端,表示存在双向关系,即内部维护两个变量,可能有一个是空值

**聚合关系:** 特殊的关联关系,描述整体和局部之间的关系,且整体和局部的生命周期相互独立

​		实线空心菱形指向整体

**组合关系:**特殊的聚合关系,整体的生命周期决定了部分的生命周期

​		实线实心菱形指向整体

**依赖关系:**类和类中成员函数中的局部变量之间关系

​		虚线箭头指向局部变量

#### C++2.0

​		允许使用nullptr替代0或NULL: nullptr的类型是std::nullptr

```c++
void f(int);
void f(void*);
f(0);		/* 调用f(int) */
f(NULL);	/* 具体调用取决于NULL是否被typedef为0 */
f(nullptr);	/* 调用f(void*) */
```

​		统一的变量或对象初始化

```c++
/* 常用初始化方式有小括号(),大括号{},赋值= ,C++2.0可以支持统一的大括号{},初始化方式 */
/* 编译器看到初始化的大括号{ },会制造一个initializer_list<T>,这是一个array<T,n>,如果存在构造函数接收一整个initializer_list<T>类型,则编译器会直接传递过去,负责会被分解,一个一个的传递过去 */
int values[]{1,2,3};
vector<int> v{2,3,4,5,7,11,13,17};
vector<string> cities{ "aa","bb","cc" }; /* 存在接收整个initializer_list<T>的构造函数 */
complex<double> c{4.0,3.0}

int i;	 	/* 初值未定义 */
int j{}; 	/* 初值为0 */
int* p;		/* 初值未定义 */
int* q{};	/* 初值为nullptr */
int x2{5.0}; /* 错误或警告 */
int x4={5.3};/* 错误或警告 */
```

**initializer_list:**

```c++
/* 大括号初始化initializer_list类,不包含array,但其指向一个array */
template<class _E>
class initializer_list
{
public:
    typedef _E			value_type;
    typedef const _E&	reference;
    typedef const _E&	const_reference;
    typedef size_t		size_type;
    typedef const _E*	iterator;
    typedef const _E*	const_iterator;
private:
    iterator	 	_M_array; /* 维护着迭代器指向背后的数据 */ 
    size_type		_M_len;
    /* 只有编译器可以调用到这里,维护着一个迭代器,指向背后的数组,真正初始化的数据在背后的array中 */
    constexpr initializer_list(const_iterator __a,size_type __1)
        :_M_array(__a),_M_len(__1){}
public:
    constexpr initializer_list() noexcept:_M_array(0),M_len(0)){}
    constexpr size() const constexpr{ return _M_len; }
    constexpr const_iterator begin() const noexcept { return _M_array; }
    constexpr const_iterator end() const noexcept { return begin()-_M_len; }
}
class P{
public:
    P(int a,int b){ /* 构造函数 */
        cout << "P(int,int),a="<<a<<",b="<<b<<end;
    }
    P(initializer_list<int> initlist){
        cout << "P(initialzier_list<int>), values="
        for(auto i: initliast){
            cout << i << ' ';
        }
        cout << endl;
    }
}
/* 使用 */
P p(77,5);		/* 调用P(int a,int b) */
P q{77,5};		/* 调用P(initializer_list<int> initlist) */
```

**range-based for statement:**

```c++
/* 语法规则 */
for(decl:coll){
    statement
}
/* 编译器底层实现 */
for(auto _pos=coll.begin(), __end=coll.end(); __pos!=__end; ++__pos){
    decl = *__pos;
    statement
}
/* 使用 */
vector<double> vec;
...
for(auto elem : vec){
    cout << elem << endl;
}
for(auto& elem : vec){
    elem *=3;
}
```

**explicit关键字:**

```c++
/* explicit几乎只用于构造函数,用于告诉编译器该函数只能被明确调用 */
struct Complex{
    int real,imag;
    /* Complex c2 = c1+5,这里的5可能被编译器隐式调用构造函数构造出一个新的Complex类 */
    //Complex(int re,int im=0):real(re),imag(im){} 
    /* Complex c2 = c1+5,加了explicit关键字这里的5不会被编译器隐式转换 */
   	explicit Complex(int re,int im=0):real(re),imag(im){} 
    Complex operator+(const Complex& x){
        return Complex((real+x.real),(imag+x.imag));
    }
}
/* C++1.0只支持一个参数的隐式调用构造函数的类型转换 */
/* C++2.0支持多个参数的隐式调用构造函数的类型转换 */
```

**=default,=delete关键字:**

```c++
/* C++规定如果一个类提供的构造函数,拷贝构造函数,拷贝赋值函数,析构函数,则编译器不在隐式提供默认的空版本 */
/* =default关键字用于告诉编译器,继续提供默认的空版本函数,=delete关键字用于告诉编译器该函数不会在被使用 */
class Foo{
public: /* 构造函数可以有重载版本,拷贝构造,拷贝赋值只能有一个 */
    Foo(int i):_i(i){ }
    Foo() =default;	/* 要求编译器继续提供空版本的构造函数 */	
    Foo(const Foo& x):_i(x._i){ }
    Foo(const Foo&) =default;	/* 告诉编译器继续提供空的拷贝构造函数,但是由于已经存在,所以报错 */
    Foo(const Foo&) =delete;	/* 告诉编译器不会使用拷贝构造函数,但是由于已经存在,所以报错 */
    Foo& operator=(const Foo& x){ _i=x._i; return *this; } /* 拷贝赋值函数 */
    /* 告诉编译器继续提供空的拷贝赋值函数,但是由于已经存在,所以报错 */
    Foo& operator=(const Foo& x)=default 
    /* 告诉编译器不会使用拷贝赋值函数,但是由于已经存在,所以报错 */
    Foo& operator=(const Foo& x)=delete    
    void fun1()=default; /* 错误,函数不能被default */
    void fun2()=delete;	 /* 告诉编译器,该函数不会在被使用 */
    ~Foo()=delete;	/* 析构对象时会出错 */
    ~Foo()=default;	/* 告诉编译器继续提供默认的空的析构函数 */
private:
    int _i;
}
/* 注意:=delete还可以作用于operator delete/delete[]和operator new/new[],
   这样对象就无法被创建/删除 */
```

**Alias Template:**

```c++
/* 给模板起一个别名,注意:别名不支持特化和偏特化 */
template <typename T>
using Vec = std::vector<T,MyAlloc<T>>;
Vec<int> coll;

/* #define和typedef均无法实现对模板类型起别名 */
#define Vec<T> template<typename T> std::vector<T,MyAlloc<T>>;
Vec<int> coll;	/* 错误:无法使用声明来定义变量 */
/* typedef定义的类型必须要被明确 */
typedef std::vector<int,MyAlloc<int>> Vec;	/* 错误:定义的变量无法传入模板参数 */
```

**Type Alias:**

```c++
/* 给一个类型起一个别名,与typedef功能一致 */
//typedef void(*func)(int,int)
using func = void(*)(int,int);		/* 给函数指针类型起一个别名func */
/* 使用 */
void example(int,int){
    func fn = example;
}
/* 在模板中的使用 */
template<typename T>
struct Container{
    using value_type = T; /* 给模板类型起个别名,等同于typedef T value_type */
}
```

**template template parament:**

```c++
/* 普通的模板参数或编译器推导的模板参数,在类型使用之前"类型必须要是明确的",简言之,无法将一个模板类型作为模板参数传递给模板,需要使用模板的模板参数来解决 */
template<typename T, template<class T> class Container> /* 模板参数中传入模板 */
class XCls
{
private:
    /* 常规模板参数在此处具体类型必须明确 */
    Container<T> c; /* 模板的模板参数具体类型在此处还未明确,但依然需要明确模板类型(一个还是两个模板参数) */
public:
    XCls(){
        for(long i=0;i<SIZE;++i){
            c.insert(c.end(),T());
        }
        output_static _data(T());
        Container<T>c1(c);
        Container<T>c2(std::move(c));
        c1.swap(c2);
    }
}   
template<typename _Tp,typename _Alloc=std::allocator<_Tp>>
class vector:protected _vector_base<_Tp,Alloc>{ ... }
/* 使用 */
XCls<MyString, vector>c1; /* 错误:模板的模板无法识别模板的默认参数 */
/* 给模板类型重新起别名 */
template<typename T>
using Vec=vector<T,allocator<T>>;
template<typename T>
using Lst=list<T,allocator<T>>;
template<typename T>
using Deq=deque<T,allocator<T>>;
XCls<MyString, Vec>c1; /* 正确:别名确定了模板的模板参数具体模板类型 */
```

**noexcept:**

```c++
/* noexcept关键字用于明确的告诉编译器该函数不会抛出异常,且可通过判断条件来控制noexcept关键字是否有效 */
void foo() noexcept; -> void foo() noexcept(true);
/* 根据判断条件确定该函数是否会抛出异常 */
void swap(Type& x,Type& y) noexcept(noexcept(x.swap(y))){ 
    x.swap(y);
}
```

**final:**

```c++
/* final关键字用于告诉编译器该类或者该虚函数是最后一个,不可在被继承或重写 */
struct Base1 final{ };
struct Derived1:Base1{ }; /* 错误:final修饰的子类是最后一个,不可被继承 */

struct Base2{
    virtual void f() final;
};
struct Derived2:Base2{
    void f(); /* 错误:final修饰的虚函数不可在被重写 */
}
```

**override:**

```c++
struct Base{
    virtual void vfun(float){ ... }
}	
struct Derived2:Base{
    virtual void vfun(float) override { ... } /* 用于明确的告诉编译器该函数是重写函数 */
}
```

**decltype:**

```c++
/* decltype关键字用于获取表达式的类型 */
map<string,float> coll;
decltype(coll)::value_type elem; /* C++2.0后可以通过decltype获取类型 */
map<string,float>::value_type elem; /* C++1.0只能通过内部定义获取类型 */

/* 1,用于声明返回值类型 */
template<typename T1,typename T2>
/* decltype(x+y)不能写在前面,因为x,y变量此处还不可见,所有用auto替代 */
auto add(T1 x,T2 y) -> decltype(x+y); /* 返回值类型是x+y */
/* 2,用于获取对象类型 */
typedef typename decltype(obj)::iterator iType; /* typdef typename T::iterator iType */
/* 3,用于获取lambda表达式类型,面对lambda表达式我们只有对象而没有类型 */
auto cmp = [](const Persion& p1,const Persion& p2){ ... };
std::set<Person,decltype(cmp)> coll(cmp); /* decltype获取lambda表达式的具体类型 */
```

**Lambdas:**

```c++
/* C++2.0中引入了lambda表达式,其本质就是一个被写为inline属性的仿函数 */
/* [...](...)mutable(opt) throwspec(opt) -> reType(opt){...} */
/* []中可以是=(可值引用外部所以变量),&(引用的方式引用指定变量),直接写的变量则是值引用 */
auto I = []{ std::cout <<"lambda"<<std::endl;} /* 如果没有参数且没有其他可选修饰则()可以省略 */

int id = 0;
/* mutable用于修饰值引用的变量是否可被修改,如下,如没有mutable则id不能进行++操作,否则编译报错 */
auto f = [id]()mutable{ std::cout<<"id"<<id<<std::endl; ++id;}

/* lambda表达式的内部实现机制 */
int tobefond = 5;
auto lambdal=[tobefond](int val) ->bool{ return val==tobefond;}; 
/* 内部隐式实现机制 */
class UnNameLocalFunction
{  	/* 注意:Lambda表达式没有默认构造函数和赋值函数 */
    int localVar;
public:
    UnNameLocalFunction(int val):loaclVar(var){}
    bool operator()(int var){
        return val==tobefond; 
    }
};
UnNameLocalFunction lambda2(tobefond);
/* 使用 */
bool b1 = lambda1(5);
bool b2 = lambda2(5);
/* 使用 */
template<class Key,class Compare=less<key>,class Alloc=alloc>
class set{
public:
    ...
    typedef Compare key_compare;
    typedef Compare value_compare;
private:
    typedef rb_tree<key_type,value_type,identity<value_type>,key_compare,Alloc> rep_type;
    rep_type t;	/* 内部维护着一个红黑树 */
public:
    set():t(Compare()){ } /* 这里会调用Compare的空构造函数,lambda表达式没有默认空构造函数 */
    explicit set(const Compare& comp):t(comp){} /* 拷贝构造函数 */
}
auto cmp= [](const Person& p1, const Person& p2){ ... };
/* 注意:lambda表达式创建的对象没有提供默认空构造函数, */
std::set<Person,decltype(cmp)> coll(cmp); /* 调用有参构造函数 */
```

**VariadicTempletes:**

```c++
/* 利用可变参数模板,对不定模板参数进行递归拆解 */
void func(){ /* ... */} /* 处理最后一个模板参数 */
template<typename T,typename... Types>
void func(const T& firstArg, const Types&... args){ /* 一个和一包数据 */
    /* firstArg */ /* 拿到第一个数据并处理 */
    func(args...); /* 递归调用,依次取得数据 */
}
```

```c++
/* 利用可变参数模板,对不定模板参数进行递归拆解 */
void printX(){ }
template<typename T,typename... Types>  /* 注意:这是模板的特化版本 */
void printX(const T& firstArg, const Types&... args)
{
    cout << firstArg<<endl; /* 取得第一个参数 */
    printX(args...);	/* 递归处理剩余的一包数据 */
}
template<typename... Types> /* 注意:这才是模板的泛化版本 */
void printX(const Types&... args)
{ /* ... */ }
/* 使用 */
print(7.5,"hello",bitset<16>(337),42);
```

```c++
/* 利用可变参数模板,实现printf函数 */
void printf(const char *s){  /* 剩余部分可能是/r/n */
    while(*s){
        if(*s == '%' && *(++s) != '%'){ /* 拿完所有数依然还存在%,则认为错误 */
            throw std::runtime_error("error")
        }
        std::cout << *s++;
    }
}
template<typename T,typename... Args>
void printf(const char* s,T value, Args... args){
    while(*s){
        if(*s == '%' && *(++s) != '%'){
            std::cout << value;  /* 拿到一个 */
            printf(++s,args...); /* 将剩余的一包继续进行递归处理 */
            return;
        }
        std::cout << *s++;
    }
    throw std::logic_erroe("error");	/* 传入的字符串是空或没有可解析的% */
}
/* 使用 */
int* pi = new int;
printf("%d%s%p%f\r",15,"aa",pi,1.1);
```

```c++
/* 类型相同使用initializer_list<T>来实现max函数 */
cout << max({57,48,60,100,20,18}) << endl; /* 传入一个initializer_list */
template<typename _Tp>
inline _Tp max(initializer_list<_Tp> __1){
    return *max_element(__1.begin(),__1.end());
}
template<typename _ForwardIterator>
inline _ForwardIterator max_element(_ForwardIterator __first,_ForwardIterator __last)
{
    return __max_element(__first,__last,__iter_less_iter()); 
}
template<typename _ForwardIterator,typename _Compare>
_ForwardIterator __max_element(_ForwardIterator __first,_ForwardIterator __last,
                               _Compare __comp){
    if(__first==__last) return __first;
    _ForwardIterator __result=__first;
    while(++__first != __last){
        if(__comp(result,_first)){
            __result = __first;
        }
    }
    return __result; /* 返回最终比较结果 */
}
inline _Iter_less_iter __iter_less_iter(){
    return _Iter_less_iter(); /* 返回仿函数对象 */
}
struct _Iter_less_iter{
    template<typename _Iterator1,typename _Iterator2>
    bool operator(_Iterator1 n__it1, _Iterator1 it2) const {
        return *__it1 < *__it2;
    }
}
```

```c++
/* 类型相同,利用可变参数模板实现max函数 */
int maximum(int n){
    return n;
}
template<typename.. Args>
int maximum(int n,Args... args){ /* 传入参数会被拆分为一个和一包 */
    return std::max(n,maximum(args...)); /* std::max是标准库的比较函数 */
}
/* 使用 */
cout << maximum(57,48,60,100,20,18) << endl;
```

```c++
/* 继承方式,利用可变参数模板实现tuple类 */
template<typename... Values> class tuple; /* 模板的泛化版本声明 */
template<> class tuple<>{} /* 空参数的模板特化版本 */
template<typename Head, typename... Tail> /* 模板的特化版本 */
class tuple<Head,Tail...> :private tuple<Tail...>
{
    typedef tuple<Tail...> inherited;
public:
    tuple(){ }	/* 最后一个空参数调用 */
    /* 利用构造机制,实现传入参数的分解 */
    tuple(Head v,Tail... vtail):m_head(v),inherited(vtail...){ }
    typename Head::type head(){ return m_head;} /* 返回当前类结构存储的数据 */
    //Head head(){ return m_head };	/* Head本身就是模板推导出的类型 */
    inherited& tail() {return *this;} /* 返回当前类结构的下一级类存储结构地址 */
protected:
    Head m_head;	/* 从可变参数中拆分的数据 */
}
/* 使用 */
tuple<int,float,string>t(41,6.3,"nico");
```
```c++
/* 以组合的方式实现可变参数模板 */
template<typename... Values>class tup;
template<> class tup<>{ };
template<typename Head,typename... Tail>
class tup<Head,Tail...> /* 一个和一包 */
{
    typedef tup<Tail...>composited;
protected:
    composited m_tail; /* 利用剩余的一包组合 */
    Head m_head;	/* 拆分出来的数据 */
public:
    tup(){ }
    tup(Head v,Tail... vtail):m_tail(vtail...),m_head(v){} /* 递归组合 */
    Haed head(){ return m_head;} /* 返回当前类结构存储的数据 */
    /* 返回引用,否则会产生拷贝 */
    composited& tail(){ return m_tail; } /* 返回当前类结构组合的下一级类存储结构 */
};
/* 使用 */
tup<int,float,string>it1(41,6.3,"nico")
cout<<it1.head()<<endl;
cout<<it1.tail().head()<<endl;
```

```c++
/* 利用可变参数模板实现tuple类输出为指定格式 */
/* 输出为[[7.5,hello,00000010110] */
cout << make_tuple(7.5,string("hello"),bitset<16>(377)) << endl;

template<typename... Args>
ostream& operator<<(ostream& os, const tuple<Args...>& t){
    os<< "[" /* sizeof...()用于获取不定模板参数个数 */
        PRINT_TUPLE<0,sizeof...(Args),Args...>::print(os,t);
    return os<<"]";
}
template<int IDX,int MAX,typename... Args>
struct PRINT_TUPLE{ /* get<index>(t)获取tuple中第index个元素 */
    static void print(ostream& os, const tuple<Args...>& t){
        os<<get<IDX(t)<<(IDX+1 == MAX ? "":",");
        PRINT_TUPLE<IDX+1,MAX,Args...>::print(os,t); /* 递归分包输出 */
    }
}; 
template<int MAX,typename... Args> /* 特化 */
struct PRINT_TUPLE<MAX,MAX,Args...>{
    static void print(std::ostream& os,const tuple<Args...>& t){ } 
};
```

**右值引用:RvalueReferences**

```c++
/* C++2.0新特性,是一种新的引用类型,用于解决一些非必要性的拷贝 */
/* 左值:Lvalue,既可以出现在表达式左侧,又可以出现在表达式右侧 */
/* 右值:Rvalue,只能出现在表达式右侧 */
int a = 9;  /* 所有具体存储空间的变量都是左值 */
int b = 4;
a = b;		/* 左值既可以放在表达式左边也可以放在表达式右边 */
b = a;
a = a+b;
a+b = 42;  /* 错误:a+b的结果为临时匿名变量,临时变量只能做右值,只能出现在右侧 */
/* 结论:所有匿名的临时变量或临时变量都是右值,只能出现在表达式右侧  */
```

```c++
/* 右值只能出现在右侧,不可对右值取引用 */
int foo(){ return 5; }
...
int x = foo();		/* Ok */
int* p = &foo();	/* 错误:foo()的返回值5是临时匿名对象,是右值,不可对右值取引用 */
foo() = 7;			/* 错误:返回值5是临时对象,是右值,不可在左侧 */
```

**右值引用使用语法:**

```c++
class MyString
{
private:
    char* _data;
    ...
public:
    MyString& operator=(const MyString& str){ /* 拷贝赋值函数 */
        ...
      	return *this;
    }
    MyString& operator=(MyString&& str) noecxept{ /* move拷贝赋值函数 */
        ...
        return *this;   
    }
    ...
}
```

**右值引用的使用:**

```c++
class MyString{
public:
    static size_t DCtor;	/* 默认构造函数调用次数 */
    static size_t Ctor;		/* 构造函数调用次数 */
    static size_t CCtor;	/* 拷贝构造函数调用次数 */
    static size_t CAsgn;	/* 拷贝赋值函数调用次数 */
    static size_t MCtor;	/* move构造函数 */
    static size_t MAsgn;	/* move赋值函数 */
    static size_t Dtor;		/* 析构函数 */
private:
    char* _data;
    size_t _len;
    void _init_data(const char* s){
        _data = new char[_len+1];
        memcpy(_data,s,_len);
        _data[_len] = '\0';
public:
     /* 默认构造函数 */
     MyString():_data(NULL),_len(0){
         ++DCtor;
     }
     /* 普通构造函数 */
     MyString(const char* p):_len(strlen(p)){
         ++Ctor;
         _init_data(p);
     }
     /* 拷贝构造函数 */
     MyString(const MyString& str):_len(str._len){
         ++CCtor;
         _init_data(str._data);
     }
     /* 拷贝赋值函数 */
     MyString& operator=(const MyString& str){
         ++CAsgn;
         if(this != &str){
             if(_data) delete _data;
             _len = str._len;
             _init_data(str._data);	/* 深拷贝 */
         }else{
             
         }
         return *this;
     }
     /* move构造函数,move版本的需要noexcept,否则vector成长时可能不会被调用 */
     MyString(MyString&& str) noexcept:_data(str._data),_len(str._len){
         ++MCtor;	/* 需要确保传入的对象不在会被使用 */
         str._len = 0;
         str._data = NULL;
     }
     /* move赋值函数,move版本的需要noexcept,否则vector成长时可能不会被调用 */
     MyString& operator=(MyString&& str) noexcept {
         ++MAsgn;
         if(this != &str){ /* 需要确保传入的对象不在会被使用 */
             if(_data) delete _data;
             _len = str._len;
             _data = str._data;
             str.len = 0;
             str._data = null; /* 避免析构时删除指针指向的数据 */
		 }
         return *this;
     }
     /* 虚析构函数 */
     virtual ~MyString(){
         if(_data){ /* 此处必须判断null,因为move版本偷完后会将指针设为null */
             delete _data;
         }
     }
     bool operator<(const MyString& rhs) const{
         return std::string(this->_data) < std::string(rhs._data);
     }
     bool operator==(const MyString& rhs) const {
          return std::string(this->_data) = std::string(rhs._data);
     }
};
size_t MyString::DCtor = 0;
size_t MyString::Ctor = 0;
size_t MyString::CCtor = 0;    
size_t MyString::CAsgn = 0;
size_t MyString::MCtor = 0;
size_t MyString::MAsgn = 0;
size_t MyString::Dtor = 0;
 
#include<typeinfo> /* typeid() */
template<typename T>
void output_static_data(const T& myStr){
    cout<<typeid(myStr).name()<<"--"<<endl;
    cout<<"CCtor="<<T::CCtor<<endl;
}   
template<typename M>
void test_moveable(M c,long& value){
    char buf[10];
    /* 容器值类型 */
    typedef typename iterator_traits<typename M::iterator>::value_type Vtype; 
    clock_t timeStart = clock();
    for(long i = 0; i< value; i++){
        snprintf(buf,10,"%d",rand()); /* 生成随机数 */
        auto ite = c.end;
        c.insert(ite,Vtype(buf)); /* 插入临时匿名对象,会调用move版本的insert函数 */
    }
    cout<<"milli-seconds:"<<(clock()-timeStart)<<endl;
    cout<<"size()="<<c1.size()<<endl;
    output_static_data(*(c1.begin()));
    /* 对整个容器的搬移 */
    M c11(c1);  /* 会产生大量内存创建和数据搬移 */
    M C12(std::move(c1)); /* 调用move版本的拷贝构造,需要确保后面不会在使用c1 */
    c11.swap(c12);
}
/* 使用 */
test_moveable(vector<MyString>(),30000000L);
/* vector容器自身的拷贝构造 */
vector(const vector& __x): /* 普通拷贝构造函数 */
    _Base(__x.size(),_Alloc_traits::__S_select_on_copy(__x._M_get_Tp_allicator()))
{   /* 此处会创建内存,并对数据执行深拷贝 */
	this->_M_impl._M_finish=std::__uninitialized_copy_a(__x.begin(),__x.end(),
                                                       this->_M_impl._M_start,
                                                       _M_get_Tp_allocator());        
}
vector(const vector&& __x): /* move版本的拷贝构造函数 */
    _Base(std::move(__x))
{	/* 此处并没有创建内存,仅仅只是交换了存储数据指针 */
    this->_M_impl._M_swap_data(__x._M_impl);
}
```

**不完整的右值传递:**

```c++
void process(int& i){
    cout << "process(int& i)"<< i << endl;
}
void process(int&& i){
    cout << "process(int&& i)"<< i << endl;
}
void forward(int&& i){  /* 右值传入 */
    cout << "forward(int&& i)" << i << endl;
    process(i)	/* 传入的右值变成左值 */
}
```

**完整的右值传递:**

```c++
template<typename _Tp>
constexpr _Tp&& forward(typename std::remove_reference<_Tp>::type& __t) noexcept
{
    return static_cast<_Tp&&>(__t); /* 将非右值强制转换为右值 */
}
template<typename _Tp>
constexpr _Tp&& forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
{
    return static_cast<_Tp&&>(__t); /* 将右值强制转换为右值 */
}
template<typename _Tp> 
constexpr typename std::remove_reference<_Tp>::type&& move(_Tp&& __t) noexcept
{
    return static_cast<typename std::remove_reference<_Tp>::type&&>(__t);
}
/* 完整的右值传递使用 */
template<typename T1,typename T2>
void functionA(T1&& t1,T2&& t2){
    functionB(std::forward<T1>(t1),
              std::forward<T2>(t2));
}
```

**const与constexpr的区别:**

```c++
/* 在C++1.0之前const的语义是"只读""常量",语义不清晰,存在语义重复,所以2.0引入新的关键字constexpr */
/* C++2.0之后:const的确定语义是"只读变量",在"运行时"确定具体数值
			 constexpr的确定语义是"只读常量",在"编译时"即可确定数值*/
/* constexpr关键字出现主要是出于对性能的优化,减少运行时的执行 */
/* 注意:constexpr修饰函数返回值时,返回值不一定是编译期常量,要看函数出入参数是否是编译期常量 */
```

#### 内存管理机制:

**placement new:**

```c++
/* 两个参数的placement new(), 实现在已经分配好的指定内存处创建对象 */
#include <new>
char* buf=new char[sizeof(Complex)*3];		/* 创建内存 */
Complex* pc = new(buf) Complex(1,2);		/* 在指定内存处创建对象 */
...
delete [] buf;
/* placement new两个参数的重载实现 */
void* operator new(size_t,void* loc){
    return loc;  /* 直接返回指定地址 */
}
/* Complex* pc = new(buf) Complex(1,2),这段代码编译器会将其隐式转换为如下形式 */
Complex* pc;
try{
    void* mem = operator new(sizeof(Complex),buf); /* 在指定内存处创建对象 */
    pc = static_cast<Complex*>(mem);
    pc->Complex::Complex(1,2);	/* 调用构造函数来初始化创建的对象 */
}catch(std::bad_alloc){
    ...
}
```

**per-class1内存管理:**

```c++
/* 目的:设计一个类来管理自身分配的内存,以此来减少底层malloc的调用次数,和内存分配时所产生的多余cookice所占用的字节数量 */
#include <cstddef>
#include <iostream>
using namespace std;

class Screen{
public:
    Screen(int x):i(x){};
    int get(){ return i;}
    /* 重写类的内存分配和释放 */
    void* operator new(size_t);
    void operator delete(void*, size_t);
private:
    Screen* next; /* 单向链表,指向下一个内存块 */
    static Screen* freeStore; /* 指向空闲链表的头 */
    static const int screenChunk; /* 一次申请的内存块数量 */
private:
    int i;		/* 数据 */
};
Screen* Screen::freeStore = 0;
const int Screen::screenChunk = 24;

void* Screen::operator new(size_t size){
    Screen* p;
    if(!freeStore){ /* 内存池中的内存已经用完,一次申请screenChunk个内存块 */
        size_t chunk = screenChunk*size;
        freeStore = p = reinterpret_cast<Screen*>(new char[chunk]);
        /* 构建一个链表,将申请的内存用单向链表串接起来 */
        for(;p!=&freeStore[screenChunk-1];++p){
            p->next = p+1;
        }
        p->next = 0;
    }
    p = freeStore;
    freeStore = freeStore->next; /* 空闲链表头指向下一个内存块 */
    return p;		/* 返回当前链表头处内存块 */
}
void Screen::operator delete(void* p, size_t){
    /* 将要释放的内存,插入到链表头部 */
    (static_cast<Screen*>(p))->next = freeStore;
    freeStore = static_cast<Screen*>(p);
}
/* 使用 */
size_t const N = 100;
Screen* p[N];
for(int i = 0; i < 10; i++){
    p[i] = new Screen(i);
}
for(int i = 10; i<10; i++){ /* 输出前10对象的地址,其间隔为8 */
    cout << p[i] << endl;
}
for(int i = 0; i < N; ++i){
    delete p[i];
}
```

**per-class2内存管理:**

```c++
/* 优化per-class1内存管理,per-class1每个对象多占用了一个指针内存,用于维护单向链表,per-class2通过利用union来将数据和链表指针重叠,在内存块空闲时union内存用来维护指针,当被分配使用时,用于存储用户数据 */
class Airplane{
private:
    struct AirplanRep{ /* 数据类型声明 */
        unsigned long miles;
        char type;	
    }
private:
    union{ /* 匿名union变量,注意此处不是声明,是定义匿名变量 */
        AirplanRep rep;  /* 让用户数据和链表指针公用同一块内存 */
        Airplane* next;  /* 嵌入式指针设计方式 */
    }
public:
    unsigned long getMiles(){ return rep.miles; }
    char getType(){ return rep.type; }
    void set(unsigned long m,char t){
        rep.miles = m; rep.type = t;
    }
public:
    /* 重写operator new&operator delete最好为static属性的,因为在类完成构建之前,成员函数不应该被调		 用,但是静态函数可以,虽然次处是由编译器来调用 */
    static void* operator new(size_t size);
    static void operator delete(void* deadObject, size_t size);
private:
    static const int BLOCK_SIZE; /* 一次申请的内存块数量 */
    static Airplane* headOfFreeList;	/* 已经申请的内存的空闲链表头 */
}
Airplane* Airplane::headOfFreeList;
const int Airplane::BLOCK_SIZE = 512;

void* Airplane::operator new(size_t size){
    if(size!=sizeof(Airplane)){ /* 如果有继承发生,类大小会改变,此处不处理此情况 */
        return ::operator new(size);
    }
    Airplane* newBlock = static_cast<Airplane*>* p = headOfFreeList;
    if(p){ /* 如果内存池中有空闲内存块,则将链表头下移,返回一个内存块 */
        headOfFreeList = p->next;
    }else{ /* 内存池中没有空闲内存块,在申请一段内存池 */
        Airplane* newBlock = static_cast<Airplane*> 
            (::operator new(BLOCK_SIZE*sizeof(Airplane)));
        /* 跳过第一个内存块,将其余空闲内存块串成链表 */
        for(int i=1; i<BLOCK_SIZE-1; i++){
            newBlock[i].next = &newBlock[i+1];
        }
        newBlock[BLOCK_SIZE-1].next = 0;
        headOfFreeList = &newBlock[1]; /* 更新空闲链表头 */
        p = newBlock; /* 取得第一个内存块地址 */
    }
    return p;
}
void Airplane::operator delete(void* deadObject, size_t size){
    if(deadObject == 0) return;
    if(size != sizeof(Airplane)){ /* 如果有继承发生 */
        ::operator delete(deadObject);
        return;
    }
    Airplane* carcass = static_cast<Airplane*>(deadObject);
    carcass->next = headOfFreeList; /* 将内存归还到内存池中 */
    headOfFreeList = carcass;
}
/* 使用 */
size_t const N = 100;
Airplane* p[N];
for(int i = 0; i < N; ++i){
    p[i] = new Airplane;
}
p[1]->set(100,'a');
for(int i = 0; i < 10; ++i){ /* 输出前10个对象的地址,其间隔为8 */
    cout << p[i] << endl;
}
for(int i = 0; i < N; ++i){
    delete p[i];
}
```

**per-class3内存管理:**

```c++
/* 优化per-class2,将内存管理部分抽离出来,避免为每一个类都重写一个operator new&operator delete,提高代码的复用 */
class allocator
{
private:
    struct obj{ /* 当分配的内存块空闲时,next指针用于维护单向链表,当被使用时,用于存储数据 */
        struct obj* next; /* 嵌入式指针 */
    }
public:
    void* allocate(size_t);
    void deallocate(void*, size_t);
private:
    obj* freeStore = nullptr; /* 指向链表头 */
    const int CHUNK = 5; /* 一次分配5个内存块 */
}
void* allocator::allocate(size_t size){
    obj* p;
    if(!freeStore){ /* 内存池中的空闲内存已使用完 */
        size_t chunk = CHUNK*size;
        /* 将内存块转换为obj类型,方便进行链表串接 */
        freeStore = p =(obj*)malloc(chunk);
        for(int i=0; i<(CHUNK-1);i++){ /* 串接成单向链表 */
			p->next = (obj*)((char*)p+size);
            p = p->next;
        }
        p->next = nullptr;
    }
    p = freeStore;	/* 返回链表头处内存块 */
    freeStore = freeStore->next; /* 将链表头下移 */
    return p;
}
void allocator::deallocate(void*p, size_t){
    ((obj*)p)->next = freeStore; /* 将内存块转换为obj后归还给内存池 */
    freeStore = (obj*)p;
}
/* 使用 */
class Foo{
public:
    long L;
    string str;
    static allocator myAlloc; /* 维护一个静态分配器 */
public:
    Foo(long l):L(l){}
    static void* operator new(size_t size){
        return myAlloc.allocate(size);  /* 调用分配器进行内存管理 */
    }
    static void operator delete(void* pdead, size_t size){
        return myAlloc.deallocate(pdead,zise);
    }
}
allocator Foo::myAlloc;
```

**per-class4内存管理:**

```c++
/* 继续优化per-class3,每一个类都需要包含一个静态的allocator分配器,和调用分配器成员函数进行内存管理,这些操作都是重复的,可以将其定义为宏,减少重复书写 */
//DECLARE_POOL_ALLOC
#define DECLARE_POOL_ALLOC()\
public:\
	void* operator new(size_t size){ return myAlloc.allocate(size);}\
	void operator delete(void* p){ myAlloc.deallocate(p,0);}\
protected:\
	static allocator myAlloc;

//IMPLEMENT_POOL_ALLOC
#define IMPLEMENT_POOL_ALLOC(class name)\
allocator class_name::myAlloc;
/* 使用 */
class Foo{
    DECLARE_POOL_ALLOC()
public:
    long L;
    string str;
public:
    Foo(long l):L(l){}
};
IMPLEMENT_POOL_ALLOC(Foo)
```

**new handler:**

```c++
/* operator new分配内存失败时会抛出异常,在抛出异常之前会调用new handler处理函数进行处理 */
void* operator new(size_t size, const std::nothrow_t& ) _THROW0()
{
    void* p;
    while((p=malloc(size)) == 0){
        _TRY_BEGIN
            if(_callnewh(size)==0) break; /* 当内存分配失败时,会不断调用new handler处理函数 */
        _CATCH(std::bad_alloc) return(0);
        _CATCH_END
    }
    return(p);
}
typedef void(*new_handler)();	/* new handler异常处理函数类型 */
new_handler set_new_handler(new_handler p) throw();	/* 实在自定义new handler异常处理函数 */
```

**std::alloc:**

```c++
/* 该内存管理器适用于频繁的分配小内存块,且每次分配的内存大小相对固定,std中的容器就是这种情况 */
#include <cstdlib>
#include <cstddef>
#include <new>
#define __THROW_BAD_ALLOC \
		err<<"out of memory"; exit(1)
/* 第一级分配器 */
template<int inst> /* inst在此处完全没有作用 */
class __malloc_alloc_template{ /* 一级分配器是直接调用malloc进行内存分配,不进行内存管理 */
private:
    /* 静态变量声明 */
    static void* oom_malloc(size_t);  
    static void* oom_realloc(void*, size_t);
    static void (*__malloc_alloc_oom_handler)();	/* 内存分配失败时的回调处理 */
public:
    static void* allocate(size_t n){ 
        void *result = malloc(n);	/* 直接调用malloc进行内存分配 */
        if(0==result){
            result = oom_malloc(n);
        }
        return result;
    }
    static void deallocate(void* p, size_t /*n*/){
        free(p);
    }
    static void* reallocate(void* p,size_t /*old_sz*/,size_t new_sz){
        void* result = realloc(p,new_sz); /* 直接调用realloc() */
        if(0==result) result = oom_realloc(p,new_sz);
        return result;
    }
    /* typedef void(*H)();
   	   H set_malloc_handler(H f); */
    static void(*set_malloc_handler(void(*f)()))(){
        void (*old)() = __malloc_alloc_oom_handler;	/* 记录旧的new handler */
    	__malloc_alloc_oom_handler = f;	 /* 将f记录起来,以便日后调用 */
        return (old);
    }
}
/* 静态函数定义 */
template <int inst>
void (*__malloc_alloc_oom_template<inst>::__malloc_alloc_oom_handler)() = 0;
/* 内存分配失败时调用 */
template<int inst>
void* __malloc_alloc_template<inst>::oom_malloc(size_t n){
    void (*my_malloc_handler)();
    voi* result;
    for(;;){ /* 不断的尝试分配内存 */
        my_malloc_handler = __malloc_alloc_oom_handler;/*换个名称,可能是考虑到多线程的情况*/
        if(0 == my_malloc_handler){ __THROW_BAD_ALLOC; }
        /* 对函数名称取地址和解引用得到的都是函数的地址 */
        (*my_malloc_hanlder)();	/* 调用new handler,尝试释放内存 */
        result = malloc(n);		/* 再次尝试分配内存 */
        if(result){
            return (result); 
        }
    }
}
/* 重新调整已分配内存失败时 */
template<int inst>
void* __malloc_alloc_template<inst>::oom_realloc(void* p, size_t n){
    void (*my_malloc_handler)();
    void* result;
    for(;;){
        my_malloc_handler = __malloc_alloc_oom_handler;	/*换个名称,可能是考虑到多线程的情况*/
        if(0 == my_malloc_handler) { __THROW_BAD_ALLOC; }
        /* 对函数名称取地址和解引用得到的都是函数地址 */
        (*my_malloc_handler)(); /* 调用new handler,尝试释放内存 */
        result = realloc(p,n);	/* 再次尝试 */
        if(result){
            return (result);
        }
    }
}
typedef __malloc_alloc_template<0> malloc_alloc;  /* 给一级分配器重新起别名 */
/* 二级分配器 */
enum {__ALIGN = 8}; 		/* 小内存块的上调边界 */
enum {__MAX_BYTES = 128};   /* 小内存块的最大上限 */
enum {__NFREELISTS = __MAX_BYTES/__ALIGN}; /* 内存管理器的链表个数 */
/* 枚举常量应该写成静态常量或者宏定义如static const int x = N */
template <bool threads, int inst>
class __default_alloc_template{
private:
    static size_t ROUND_UP(size_t bytes){ /* 按照8字节向上对齐 */
        return (((bytes)+__ALIGN-1)&~(__ALIGN-1));
    }
private:
    union obj{	/* 嵌入式指针,用于形成链表 */
        union obj* free_list_link;
    }
private:
    static obj* volatile free_list[__NFREELISTS];	/* 管理的链表数量 */
    static size_t FREELIST_INDEX(size_t bytes){ 	/* 根据字节大小索引到对应的第几号链表 */
        return ((((bytes)+(__ALIGN-1))/__ALIGN)-1);
    }
    static void* refill(size_t n);	
    static char* chunk_alloc(size_t size, int &nobjs);	/* 申请size*nobjs的内存块 */
    static char* start_free;	/* 指向空闲内存池的头 */
    static char* end_free;		/* 指向空闲内存池的尾 */
    static size_t heap_size;	/* 累计分配的内存总量 */
public:
    static void* allocate(size_t n){ 
        obj* volatile *my_free_list;
        obj* result;
        if(n > (size_t)__MAX_BYTES){ /* 大内存块直接调用一级分配器进行处理 */
            return (malloc_alloc::allocate(n));
        }
        my_free_list = free_list + FREELIST_INDEX(n);
        result = *my_free_list;
        if(result == 0){ /* 当前链表没有已分配的空闲内存块 */
            void* r = refill(ROUND_UP(n)); /* 申请内存 */
            return r;
        }
        *my_free_list = result->free_list_link;  /* 有空闲内存块,取出一个并返回 */
        return (result);
    }
    /* 该内存释放存在两大缺陷,1:无法判断归还的内存是否是从该内存管理器中分配出去的,任何内存都可以挂入链表
    			   		  2:归还的内存大小,如果不是8的倍数,且下次仍然被使用,将带来灾难*/
    static void deallocate(void* p, size_t n){ /* 将归还的内存串入空闲链表 */
        obj* q = (obj*)p;
        obj* volatile *my_free_list;
        if(n > (size_t)__MAX_BYTES){ /* 大内存块不在管理范围内 */
            malloc_alloc::dellocate(p, n);
            return;
        }
        my_free_list = free_list + FREELIST_INDEX(n); /* 找到要挂入的第几号空闲链表 */
        q->free_list_link = *my_free_list;	/* 挂入链表 */
        *my_free_list = q;
    }
}
/* 分配内存并管理空闲内存池,内存管理核心部分 */
template <bool threads, int inst>		/* nobjs是引用传入 */
char* __default_alloc_template<threads,inst>::chunk_alloc(size_t size, int &nobjs)
{
    char* result;
    size_t total_bytes = size*nobjs; 			/* 本次要申请分配的内存 */
    size_t bytes_left = end_free-start_free; 	/* 线程池中剩余内存块大小 */
    if(bytes_left > total_bytes){ 			/* 从内存池中分配,内存池中的空间足够本次内存分配 */
        result = start_free;
        start_free += total_bytes;		/* 调整内存池中剩余内存大小 */
        return (result);
    }else if(bytes_left >= size){ 	/* 空间不够,但满足一个以上内存块的分配 */
        nobjs = bytes_left/size;	/* 改变内存块的需求数量 */
        total_bytes = size*nobjs;	/* 改变内存需求总量 */
        result = start_free;	
        start_free += total_bytes;	/* 调整内存池中剩余内存大小 */
        return (result);
    }else{ 	/* 内存池中的剩余内存不够,需要申请新内存 */
        /* 本次需要申请的内存大小 */
        size_t bytes_to_get = 2*total_bytes + ROUND_UP(heap_size >> 4); 
        if(bytes_left > 0){ /* 内存池中还有空闲内存,将其视为内存碎片,挂入对应的空闲链表中 */
            obj* volatile *my_free_list = free_list + FREELIST_INDEX(bytes_left);
            /* 挂入对应的空闲链表中 */
            ((obj*)start_free)->free_list_link = *my_free_list;
            *my_free_list = (obj*)start_free;
        }
        start_free = (char*)malloc(bytes_to_get); /* 申请新内存 */
        if(0 == start_free){ 	/* 内存申请失败 */
            int i;
            obj* volatile *my_free_list;
            obj* p;
            /* 向上遍历,从已分配的空闲内存中分割,此处的i从size+__ALIGN处开始遍历更合理 */ 
            for(i = size; i <= __MAX_BYTES; i += __ALIGN){
                my_free_list = free_list + FREELIST_INDEX(i);
                p = *my_free_list; 
                if(0 != p){ /* 该空闲链表中存在空闲内存块 */
                    *my_free_list = p->free_list_link; /* 拿一块出来,更新该空闲链表头指针 */
                    start_free = (char*)p;   	/* 拿了一块空闲内存给空闲内存池 */
                    end_free = start_free + i;
                    retrun (chunk_alloc(size, nobjs)); /* 内存池中有空闲块,再试一次 */
                    /* 此时的内存池中至少能提供一个内存块,剩余内存碎片会被挂入到指定空闲链表中 */
                }
            }
            /* 已经彻底没有内存了,尝试一级分配器的new handler,看看还能否有一线希望 */
            end_free = 0; 
            start_free = (char*)malloc_alloc::allocate(bytes_to_get);
        }
        heap_size += bytes_to_get;  			/* 累加分配总量 */
        end_free = start_free + bytes_to_get;	/* 更新空闲内存池 */
        reruen (chunk_alloc(size, nobjs));		/* 递归在试一次 */
    }
}
template<bool threads, int inst>  /* 申请内存,并将申请到的内存串成链表 */
void __default_alloc_template<threads, inst>::refill(size_t n){ /* n已调整至8的倍数 */
	int nobjs = 20;	/* 预设,一次最大取20个内存块 */
    char* chunk = chunk_alloc(n,nobjs); /* 注意nobjs是引用传入 */
    obj* volatile *my_free_list;
    obj* result;
    obj* current_obj;
    obj* next_obj;
    int i;
    if(1 == nobjs) { /* 只有一块,无法串成链表 */
        return (chunk);
    }
    my_free_list = free_list + FREELIST_INDEX(n);
    result = (obj*)chunk; 	/* 第一个内存块用于返回 */
    *my_free_list = next_obj = (obj*)(chunk+n);
    for(i = 1; ;++i){ /* 将内存块串成链表 */
        current_obj = next_obj;
        next_obj = (obj*)((char*)next_obj+n); /* 下一个内存块 */
        if(nobjs == i){ /* 最后一个 */
            current_obj->free_list_link = 0;
            break;
        }else{
            current_obj->free_list_link = next_obj;
        }
    }
    return (result);
}
/* 静态变量定义 */
template <bool threads, int inst>
char* __default_alloc_template<threads, inst>::start_free = 0;
template <bool threads, int inst>
char* __default_alloc_template<threads, inst>::end_free = 0;
template <bool threads, int inst>
char* __default_alloc_template<threads, inst>::heap_size = 0;
template <bool therads, int inst>
__default_alloc_template<threads, inst>::obj* volatile 
__default_alloc_template<threads, inst>::free_list[__NFREELIST]
    = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0};
/* 二级分配器的名称 */
typedef __default_alloc_template<false,0>alloc;
```



