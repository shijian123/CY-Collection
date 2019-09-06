
# Runtime简介


Objective-C 是一门动态语言，这意味着它总是想办法把一些决定工作从编译连接推迟到运行时。也就是说只有编译器是不够的，还需要一个运行时系统 (runtime system) 来执行编译后的代码。这就是 Objective-C Runtime 系统存在的意义，它是整个 Objc 运行框架的一块基石。


## Runtime版本和平台

根据Apple官方文档的描述，Runtime其实有两个版本:“modern”和 “legacy”。我们现在用的 Objective-C 2.0 采用的是现代（modern）版的Runtime系统，运行在 iOS 和 OS X 10.5 之后的64位程序中（苹果自2013年iPhone5s开始，所有的产品都是64位的）。而之前的32位程序采用 Objective-C 1的早期（legacy）版本的 Runtime 系统。这两个版本最大的区别在于当你更改一个类的实例变量的布局时，在legacy版本中你需要重新编译它的子类，而modern版则不需要。

[点击这里](https://opensource.apple.com/source/objc4/)可以看到苹果维护的开源代码，下面的源码分析来自objc4-750.1


## 与Runtime交互
Objective-C 从三种不同的层级上与 runtime 系统进行交互：通过 Objective-C 源代码；通过 Foundation 框架中的NSObject类定义的方法；通过对 runtime 函数的直接调用。

**Objective-C源代码**

在大多数情况下，runtime 系统会在后台自动运行。只需编写和编译Objective-C源代码即可使用它。

runtime 的主要函数是发送消息的函数，消息的执行会使用到一些编译器为实现动态语言特性而创建的数据结构和函数，Objc中的类、方法和协议等在 runtime 中都由一些数据结构来定义（比如objc_msgSend函数及其参数列表中的id和SEL）
	
**NSObject方法**

在Cocoa中绝大多数的类都是继承自NSObject， 最特殊的例外是NSProxy类。
NSObject定义了类层次结构中该类下方所有类的公共接口和行为。NSProxy是专门用于实现代理对象的类，它实现了一些消息转发有关的方法，可以通过继承它来实现一个其他类的替身类或是虚拟出一个不存在的类。这两个基类都遵循了NSObject协议。在NSObject协议中，声明了所有OC对象的公共方法。
```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

@interface NSProxy <NSObject> {
    Class isa;
}

```

runtime在NSObject协议中定义了-些基础操作：
```
//返回描述该类内容的字符串
@property (readonly, copy) NSString *description;

//返回对象的类
- (Class)class OBJC_SWIFT_UNAVAILABLE("use 'type(of: anObject)' instead");

//-isKindOfClass:和-isMemberOfClass: 方法检查对象是否存在于指定的类的继承体系中(是否是其子类或者父类或者当前类的成员变量)
- (BOOL)isKindOfClass:(Class)aClass;
- (BOOL)isMemberOfClass:(Class)aClass;

//检查对象是否实现了指定协议类的方法
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;

//检查对象能否响应指定的消息
- (BOOL)respondsToSelector:(SEL)aSelector;

```

在NSObject的类中还定义了一个方法，用来返回指定方法实现的地址IMP

```
- (IMP)methodForSelector:(SEL)aSelector;
```


**Runtime的函数**

runtime系统是由一系列函数和数据结构组成的，头文件存放于`/usr/include/objc`目录下。许多函数允许你用纯C代码来重复实现 Objc 中同样的功能。虽然有一些方法构成了NSObject类的基础，但是你在写 Objc 代码时一般不会直接用到这些函数的，除非是写一些Objc与其他语言的桥接或是底层的debug工作。在[Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective-c_runtime) 中有对 runtime 函数的详细文档。
 

# NSObject基类

与runtime交互的3种方式，前两种方式都与NSObject有关，那我们就从NSObject基类开始说起，NSObject源码如下：

```
typedef struct objc_class *Class;

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```
从NSObject的源码中，我们看到了一个objc_class，在Xcode10中点击查看，源码如下：

```

/* Types */
//  #if !OBJC_TYPES_DEFINED 表示里面的代码是无效的
#if !OBJC_TYPES_DEFINED

/// An opaque type that represents a method in a class definition.
typedef struct objc_method *Method;

/// An opaque type that represents an instance variable.
typedef struct objc_ivar *Ivar;

/// An opaque type that represents a category.
typedef struct objc_category *Category;

/// An opaque type that represents an Objective-C declared property.
typedef struct objc_property *objc_property_t;

struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

// OBJC2_UNAVAILABLE表示Objective-C 2.0不再使用了
#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;	
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */

#endif
```

从这里可以看出，每个类都是一个objc_class的结构体，在结构体中有一个isa指
针，该指针指向自己所属的类。在objc_class 结构体中还定义了类名，版本等信息，其中
ivars是objc_ivar_list成员变量列表的指针；methodLists是指向objc_method_list指针的指针，*methodLists是指向方法列表的指针。

但是查看苹果的注释后，发现**上面的源码是legacy版本**的！！！通过查找最新的modern版本，其实苹果发布Objc 2.0之后，objc_class的定义就变成下面这份，在[objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-runtime-new.h.auto.html)中查看

```
typedef struct objc_class *Class;

truct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    
    class_rw_t *data() { 
        return bits.data();
    }
    ... 省略其他方法
}
```

objc_class继承于objc_object，那objc_object又是什么呢？它的定义在[objc-private.h](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-private.h.auto.html)中能看到

```
typedef struct objc_object *id;

struct objc_object {
private:
    isa_t isa;

public:

    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();

    // getIsa() allows this to be a tagged pointer object
    Class getIsa();
    ... 此处省略其他方法声明
}
```
而objc_object 结构体又包含了一个 isa 指针，我们继续查看isa_t的源码

```
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;
}
```

我们将上面源码的定义转化成类图，就是

![runtime_ NSObject.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab22ca583a69ac?w=956&h=535&f=png&s=92977)


从上述源码中，我们可以看到，Objective-C 对象都是 C 语言结构体实现的，在objc2.0中，所有的对象都会包含一个`isa_t`联合体。
objc_object在源码中typedef成了id类型，也就是我们常见的id类型。这个结构体中就只包含了一个`isa_t`联合体。
由于objc_class继承于objc_object，在objc_class中也包含了类型为`isa_t`联合体的`isa`指针，因此Objective-C 中类也是一个对象。在objc_class中，除了isa之外，另外3个成员变量，一个是超类的指针，一个是方法缓存，最后一个是这个类的实例方法链表。


## isa
当一个对象的实例方法被调用的时候，会通过isa找到相应的类，然后在该类的class_data_bits_t中去查找方法。class_data_bits_t是指向了类对象的数据区域。在该数据区域内查找相应方法的对应实现。
但是在我们调用类方法的时候，类对象的isa里面是什么呢？为了和对象查找方法的机制一致，遂引入了元类(meta-class)的概念。


## 什么是元类（meta-class）

类对象所属类型就叫做元类，它用来表述类对象本身所具备的元数据。类方法就定义于此处，因为这些方法可以理解成类对象的实例方法。每个类仅有一个类对象，而每个类对象仅有一个与之相关的元类。当你发出一个类似 [NSObject alloc] 的消息时，你事实上是把这个消息发给了一个类对象 (Class Object) ，这个类对象必须是一个元类的实例，而这个元类同时也是一个根元类 (root meta class) 的实例。所有的元类最终都指向根元类为其超类，所有的元类的方法列表都有能够响应消息的类方法，所以当 [NSObject alloc] 这条消息发给类对象的时候，objc_msgSend() 会去它的元类里面去查找能够响应消息的方法，如果找到了，然后对这个类对象执行方法调用。


* 对象的实例方法调用时，通过对象的 isa 在类中获取方法的实现。

* 类对象的类方法调用时，通过类的 isa 在元类中获取方法的实现。

meta-class之所以重要，是因为它存储着一个类的所有类方法。每个类都会有一个单独的meta-class，因为每个类的类方法基本不可能完全相同。

下图很好的描述了对象，类，元类之间的关系:


![class_isa.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab20338ce023a5?w=625&h=656&f=png&s=162477)

图中实线是 super_class指针，虚线是isa指针。

* Root class (class)其实就是NSObject，NSObject是没有超类的，所以Root class(class)的superclass指向nil

* 每个Class都有一个isa指针指向唯一的Meta class

* Root class(meta)的superclass指向Root class(class)，也就是NSObject，形成一个回路。
每个Meta class的isa指针都指向Root class (meta)

通过objc_class可以看到运行时一个类还关联了它的超类指针，类名，成员变量，方法，缓存，还有附属的协议。

## cache_t

```
struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
    ... 省略其他方法
};

#if __LP64__
typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits
#else
typedef uint16_t mask_t;
#endif

struct bucket_t {
private:
    // IMP-first is better for arm64e ptrauth and no worse for arm64.
    // SEL-first is better for armv7* and i386 and x86_64.
#if __arm64__
    MethodCacheIMP _imp;
    cache_key_t _key;
#else
    cache_key_t _key;
    MethodCacheIMP _imp;
#endif

public:
    inline cache_key_t key() const { return _key; }
    inline IMP imp() const { return (IMP)_imp; }
    inline void setKey(cache_key_t newKey) { _key = newKey; }
    inline void setImp(IMP newImp) { _imp = newImp; }

    void set(cache_key_t newKey, IMP newImp);
};

```
![cache_t.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab20338d7ede8b?w=492&h=247&f=png&s=20086)


根据源码，我们可以知道cache_t中存储了一个bucket_t的结构体，和两个unsigned int的变量。

* mask：用来缓存bucket的总数。

* occupied：表明目前实际占用的缓存bucket的个数。

* bucket_t 的结构体中存储了一个unsigned long和一个IMP。IMP是一个函数指针，指向了一个方法的具体实现。

* _buckets其实就是一个散列表，用来存储Method的链表。

Cache的作用主要是为了优化方法调用的性能。当对象receiver调用方法message时，首先根据对象receiver的isa指针查找到它对应的类，然后在类的methodLists中搜索方法，如果没有找到，就使用super_class指针到父类中的methodLists查找，一旦找到就调用方法。如果没有找到，有可能消息转发，也可能忽略它。但这样查找效率太低，因为一个类往往大概只有20%的方法经常被调用，但却占总调用次数的80%。所以使用Cache来缓存经常调用的方法，当调用方法时，优先在Cache查找，如果没有找到，再到methodLists查找


## class_data_bits_t
objc_class 中最复杂的是 bits，class_data_bits_t 结构体所包含的信息太多了，主要包含 class_rw_t， retain/release/autorelease/retainCount 和 alloc 等信息，很多存取方法也是围绕它展开。查看 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-runtime-new.h.auto.html) 源码如下

```
struct class_data_bits_t {

	// Values are the FAST_ flags above.
	uintptr_t bits;
	class_rw_t* data() {
	   return (class_rw_t *)(bits & FAST_DATA_MASK);
	}
... 省略其他方法
}

struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
    
#if SUPPORT_INDEXED_ISA
    uint32_t index;
#endif
	... 省略操作 flags 的相关方法
};

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};

```
![class_data_bits_t.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab20338b7d4473?w=829&h=534&f=png&s=69073)

objc_class结构体中的注释写到 class_data_bits_t相当于 class_rw_t指针加上 rr/alloc 的标志。

```
class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
```
它为我们提供了便捷方法用于返回其中的 class_rw_t *指针：

```
class_rw_t *data() {
    return bits.data();
}
```

class_rw_t 的内容是可以在运行时被动态修改的，它提供了 runtime 对类拓展的能力，可以说运行时对类的拓展大都是存储在这里的。而 class_ro_t 存储的大多是类在编译时就已经确定的信息。二者都存有类的方法、属性（成员变量）、协议等信息，不过存储它们的列表实现方式不同，即rw-readwrite，ro-readonly。如下图


![class_data_bits_t2.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab20338e1016c3?w=634&h=816&f=png&s=70836)

**class_rw_t 和 class_ro_t的关系**

在某个类初始化之前，objc_class->data() 返回的指针指向的其实是个 class_ro_t 结构体。等到 static Class realizeClass(Class cls) 静态方法在类第一次初始化时被调用，它会开辟 class_rw_t 的空间，并将 class_ro_t 指针赋值给 class_rw_t->ro，最后调用methodizeClass方法，把类里面的属性，协议，方法都加载进来。

<!--```
struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

方法method的定义如上。里面包含3个成员变量。SEL是方法的名字name。types是Type Encoding类型编码，

IMP是一个函数指针，指向的是函数的具体实现。在runtime中消息传递和转发的目的就是为了找到IMP，并执行函数
-->
整个运行时过程可以描述如下：

![runtime_process.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab20338e5a8bcb?w=996&h=670&f=png&s=100816)

综上所述，我们总结一下objc-class在objc 1.0和2.0中的差别

![objc1.0&2.0_1.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab2033ac3c563c?w=1000&h=777&f=png&s=154888)

![objc1.0&2.0_2.png.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab2033adf8f6bd?w=1000&h=777&f=png&s=170279)




# 消息

Objc中发送消息是用中括号（[]）把接收者和消息括起来，而直到运行时才会把消息与方法实现绑定。

## objc_msgSend 函数

编译器将消息表达式`[receiver message]`转换为对objc_msgSend函数的调用，即 `objc_msgSend(receiver, selector)` ，其中将 接收器 和 消息中显示的方法名-即方法选择器，作为它的两个主要参数。

如果在消息中还有其他参数，则为：

```
id objc_msgSend(id self, SEL op, ...);
```

**id**

objc_msgSend 第一个参数类型为id，它是一个指向类实例的指针，前面我们已经介绍过了
```
typedef struct objc_object *id;
```


**SEL**

可变参数函数`objc_msgSend`的第二个参数类型为SEL，它是selector在Objc中的表示类型，selector是方法选择器，可以理解为区分方法的 ID，而这个 ID 的数据结构是SEL:

```
typedef struct objc_selector *SEL;
```

其实它就是一个映射到方法的C字符串。你可以用 Objc 编译器命令 @selector() 或者 Runtime 系统的 sel_registerName 函数来获得一个 SEL 类型的方法选择器。

需要注意的是@selector()选择子只与函数名有关，不同类中相同名字的方法所对应的方法选择器是相同的，即使方法名字相同而变量类型不同也会导致它们具有相同的方法选择器，所以 Objc中方法命名有时会带上参数类型（NSNumber）。


**IMP**
IMP在objc.h中的定义是：

```
typedef void (*IMP)(void /* id, SEL, ... */ ); 
```
它就是一个函数指针，由编译器生成。当你发起一个 ObjC 消息之后，最终它会执行的那段代码，就是由这个函数指针指定的，即 IMP 这个函数指针就指向了这个方法的实现。我们如果得到了执行某个实例的某个方法的入口，就可以绕开消息传递阶段，直接执行方法

通过上面的源码我们发现 IMP 指向的方法与 objc_msgSend 函数类型相同，参数都包含 id 和 SEL 类型。每个方法名都对应一个 SEL 类型的方法选择器，而每个实例对象中的 SEL 对应的方法实现肯定是唯一的，通过一组 id 和 SEL 参数就能确定唯一的方法实现地址；反之亦然。


**Method**

Method是一种代表类中的某个方法的类型。

```
typedef struct method_t *Method;
```

 objc_method 存储了方法名，方法类型和方法实现：

```
struct method_t {
    SEL name;
    const char *types;
    IMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```
方法名类型为 SEL，前面提到过相同名字的方法即使在不同类中定义，它们的方法选择器也相同。
方法类型 types 是个char指针，其实存储着方法的参数类型和返回值类型。
imp 指向了方法的实现，本质上是一个函数指针



### 消息发送的步骤
1、检测这个 selector 是不是要忽略的。比如 Mac OS X 开发，有了垃圾回收就不理会 retain, release 这些函数了。

2、检测这个 target 是不是 nil对象。当为nil时，如果这里有相应的nil的处理函数，就跳转到相应的函数中；如果没有处理nil的函数，就自动清理现场并返回。这也是为何在OC中给nil发送消息不会崩溃的原因。

3、如果上面两个都通过了，则开始查找这个类的 IMP，先从 cache 里面找，若找得到就跳到对应的函数去执行。

4、如果在 cache 中找不到就去方法分发表中查询。

5、如果分发表也找不到，就去查询超类的分发表，就这样一直在超类中查找，直到找到NSObject类为止。

6、如果还找不到就要开始进入动态方法解析了，后面会提到。

ps：分发表其实就是Class中的方法列表，它将方法选择器和方法实现地址联系起来


下图为消息发送流程

![消息发送框架](https://user-gold-cdn.xitu.io/2019/5/14/16ab20342cc2879b?w=332&h=544&f=gif&s=11977)

## 调用中的隐藏参数

当objc_msgSend找到方法对应的实现时，它将直接调用该方法实现，并将消息中所有的参数都传递给方法实现,同时,它还将传递两个隐藏的参数:

* 接收消息的对象（也就是self指向的内容）
* 方法选择器（_cmd指向的内容）

之所以说它们是隐藏的是因为在源代码方法的定义中并没有声明这两个参数。它们是在代码被编译时被插入到实现中的。尽管这些参数没有被明确声明，在源代码中我们仍然可以引用它们。在下面的例子中，self接收strange对象的消息，而_cmd引用了strange方法的选择器：

```
- strange
{
    id  target = getTheReceiver();
    SEL method = getTheMethod();
 
    if ( target == self || method == _cmd )
        return nil;
    return [target performSelector:method];
}
```
在这两个参数中，self 更有用。实际上,它是在方法实现中访问 消息接收者对象的实例变量 的途径。

而当方法中的super关键字接收到消息时，编译器会创建一个objc_super结构体：

```
struct objc_super { id receiver; Class class; };
```
这个结构体指明了消息应该被传递给特定超类的定义。但receiver仍然是self本身，这点需要注意，因为当我们想通过[super class]获取超类时，编译器只是将指向self的id指针和class的SEL传递给了objc_msgSendSuper函数，因为只有在NSObject类才能找到class方法，然后class方法调用object_getClass()，接着调用objc_msgSend(objc_super->receiver, @selector(class))，传入的第一个参数是指向self的id指针，与调用[self class]相同，所以我们得到的永远都是self的类型。



## 获取方法地址

避开动态绑定的唯一方法是直接获取方法的地址并调用它，这种做法很少用，除非是需要持续大量重复调用某方法，为避免消息传递的开销而直接调用。

使用NSObject类中定义的方法`methodForSelector:`，可以获取到指向实现方法的IMP，然后使用IMP直接调用

下面的示例显示了如何调用`setFilled:`实现该方法的过程：
```
void（* setter）（id，SEL，BOOL）;
int i;
 
setter =（void（*）（id，SEL，BOOL））[target
    methodForSelector：@selector（setFilled :)];
for（i = 0; i <1000; i ++）
    setter（targetList [i]，@selector（setFilled :)，YES）;
```
当方法被当做函数调用时，就需要我们明确给出传递过程的两个隐藏参数：接收对象（self）和方法选择器（_cmd）。

使用`methodForSelector:`规避动态绑定可以节省消息传递所需的大部分时间。但是只有在特定消息重复多次调用的情况下，节省才有效果，如上面所示的for循环。

PS：`methodForSelector:`方法是由 Cocoa 的 runtime 系统提供的，而不是 Objc 自身的特性。


# 动态方法解析

有时候我们希望动态地提供一个方法的实现。例如我们可以用@dynamic关键字在类的实现文件中修饰一个属性：

```
@dynamic propertyName;
```

这表明我们会为这个属性动态提供存取方法，也就是说编译器不会再默认为我们生成setPropertyName:和propertyName方法，而需要我们动态提供。我们可以重载resolveInstanceMethod:和resolveClassMethod:方法分别添加实例方法实现和类方法实现。

因为当 runtime 系统在Cache和方法分发表中（包括超类）找不到要执行的方法时，runtime 会调用`resolveInstanceMethod:`或`resolveClassMethod:`来给程序员一次动态添加方法实现的机会。我们需要用`class_addMethod`函数完成向特定类添加特定方法实现的操作：

```
void dynamicMethodIMP(id self, SEL _cmd) {
    // implementation ....
}
@implementation MyClass
//动态方法解析
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(resolveThisMethodDynamically)) {
          class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
          return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}
@end
```
上面的例子为`resolveThisMethodDynamically`方法添加了实现内容，也就是dynamicMethodIMP方法中的代码。其中"v@:"表示方法参数编码，v表示Void，@表示OC对象，:表示SEL类型

PS：动态方法解析会在消息转发机制浸入前执行。如果 `respondsToSelector:` 或 `instancesRespondToSelector:`方法被执行，动态方法解析器将会被首先给予一个提供该方法选择器对应的IMP的机会。如果你想让该方法选择器被传送到转发机制，那么就让`resolveInstanceMethod:`返回NO


# 消息转发

将消息发送给不处理该消息的对象是一个错误。但是，在抛出错误之前，runtime系统为接收对象提供了再次处理消息的机会。

下图为消息转发流程图：

![message_forwarding.png](https://user-gold-cdn.xitu.io/2019/5/14/16ab52c0a83ca20d?w=700&h=379&f=png&s=50549)



## 备用接收者

在执行消息转发机制前，runtime系统会再给我们机会，即通过重载`- (id)forwardingTargetForSelector:(SEL)aSelector`方法替换消息的接受者为其他对象：

```

+ (BOOL)instancesRespondToSelector:(SEL)aSelector {
    //返回YES，没有动态方法解析，进入下一步
    return YES;
}

//此方法中替换消息的接受者为其他对象
- (id)forwardingTargetForSelector:(SEL)aSelector {

    if (aSelector == @selector(work)) {
        return [[Person alloc] init];
    }
    return  [super forwardingTargetForSelector:aSelector];
}
```

如果想替换类方法的接受者，需要覆写`+(id)forwardingTargetForSelector:(SEL)aSelector`方法，并返回类对象：

```
+ (BOOL)instancesRespondToSelector:(SEL)aSelector {
    //返回YES，没有动态方法解析，进入下一步
    return YES;
}

// 如果想替换类方法的接受者
+ (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(work)) {
        return NSClassFromString(@"Person");
    }
    return  [super forwardingTargetForSelector:aSelector];
}
```

## 转发
当动态方法解析和备用接收者都不作处理时，消息转发机制会被触发。在这时forwardInvocation:方法会被执行，我们可以重写这个方法来定义我们的转发逻辑：

```

// ****************** 完整消息转发 *************************

+ (BOOL)instancesRespondToSelector:(SEL)aSelector {
    //返回YES，没有动态方法解析，进入下一步
    return YES;
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    //没有设置备用接收者，进入下一步
    return  nil;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if ([NSStringFromSelector(aSelector) isEqualToString:@"travel"]) {
        // 签名，进入forwardInvocation
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

// 消息转发
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL sel = anInvocation.selector;
    
    Person *p = [[Person alloc] init];
    if ([p respondsToSelector:sel]) {
        [anInvocation invokeWithTarget:p];
    }else {
        [self doesNotRecognizeSelector:sel];
    }

}
```
当一个对象由于没有相应的方法实现而无法响应某消息时，运行时系统将通过`forwardInvocation:`消息通知该对象。每个对象都从NSObject类中继承了`forwardInvocation:`方法。然而，NSObject中的方法实现只是简单地调用了`doesNotRecognizeSelector:`。通过实现我们自己的`forwardInvocation:`方法，我们可以在该方法实现中将消息转发给其它对象。

`forwardInvocation:`方法就像一个不能识别的消息的分发中心，将这些消息转发给不同接收对象。或者它也可以象一个运输站将所有的消息都发送给同一个接收对象。它可以将一个消息翻译成另外一个消息，或者简单的”吃掉“某些消息，因此没有响应也没有错误。`forwardInvocation:`方法也可以对不同的消息提供同样的响应，这一切都取决于方法的具体实现。该方法所提供的是将不同的对象链接到消息链的能力。

注意： `forwardInvocation:`方法只有在消息接收对象无法正常响应消息时才会被调用。 所以，如果我们希望一个对象将negotiate消息转发给其它对象，则这个对象不能有negotiate方法。否则，`forwardInvocation:`将不可能会被调用。

## 转发和多继承

转发和继承相似，可以用于为Objc程序添加一些多继承的效果，如下图所示，一个对象通过消息转发，就好似它可以把另一个对象中的方法实现借过来或是“继承”过来一样。


![转发](https://user-gold-cdn.xitu.io/2019/5/14/16ab2033d166368f?w=376&h=241&f=gif&s=6603)

因此从继承层次结构上看，转发消息的对象有两个“继承”方法分支- 它自己的分支和响应消息的对象的分支。在上图中，Warrior和Diplomat没有继承关系，但是Warrior将negotiate消息转发给了Diplomat后，就好似Diplomat是Warrior的超类一样。

消息转发为我们提供了那些想从多继承中获取的大多数功能，但是两者还是存在显著的差异：多继承是在单个对象中组合不同的功能，它倾向于较大且多功能的对象；与之相反，消息转发则为不同的对象分配不同的职责，它将问题分解得很细，只针对想要借鉴的方法才转发，而且转发机制是透明的。

## 替代者对象(Surrogate Objects)

转发不仅能模仿多继承，也能使轻量级对象代表或“覆盖”重量级对象，替代其他对象并向其发送消息

例如有一个操纵大量数据的对象A（创建一个复杂的图像或读取磁盘上文件的内容），由于运行A可能非常耗时，我们希望在确实需要时或者系统资源比较空闲的时候才进行操作。但同时，我们需要拥有A的占位符才能让应用程序中的其他对象正常运行。在这种情况下，我们可以一开始为它创建一个轻量级的Surrogate，而不是一个完整的对象。这个对象可以自己做一些事情，比如回答有关数据的问题，但大多数情况下它只是为较大对象保留一个位置，并在时机来临的时候将消息转发给它。当Surrogate的`forwardInvocation:`方法收到要前往另一个对象的消息时，它会首先确保该对象是否存在，如果不存在则创建它。由于较大对象的所有消息都经过Surrogate，因此对程序的其余部分而言，Surrogate和较大对象是相同的。


## 转发和继承

尽管转发很像继承，但是NSObject类不会将两者混淆。像`respondsToSelector: `和`isKindOfClass:`这类方法只会考虑继承体系，不会考虑转发链。比如上图中一个Warrior对象如果被问到是否能响应negotiate消息：

```
if（[aWarrior respondsToSelector：@selector（negotiate）]）
    ...
```
结果是NO，尽管它能够接受negotiate消息而不报错，但是它是靠转发消息给Diplomat类来响应消息。

如果您希望Warrior能够很像是继承到了Diplomat的negotiate方法，您需要重新实现`respondsToSelector:`和`isKindOfClass:`方法来加入你的转发算法：

```
- (BOOL)respondsToSelector:(SEL)aSelector
{
    if ( [super respondsToSelector:aSelector] )
        return YES;
    else {
        /* Here, test whether the aSelector message can     *
         * be forwarded to another object and whether that  *
         * object can respond to it. Return YES if it can.  */
    }
    return NO;
}
```

除了`respondsToSelector: `和 `isKindOfClass:`之外，`instancesRespondToSelector:`中也应该写一份转发算法。如果使用了协议，`conformsToProtocol:`同样也要加入到这一行列中。类似地，如果一个对象转发它接受的任何远程消息，它得给出一个`methodSignatureForSelector:`来返回准确的方法描述，这个方法会最终响应被转发的消息。比如一个对象能给它的替代者对象转发消息，它需要实现`methodSignatureForSelector:`像下面这样：

```
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector
{
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
       signature = [surrogate methodSignatureForSelector:selector];
    }
    return signature;
}
```


# Method Swizzling

黑魔法 Method Swizzling 是runtime里面很强大的一部分，它可以通过runtime的API实现更改任意的方法，理论上可以在运行时通过类名/方法名hook到任何 OC 方法，替换任何类的实现以及新增任意类

**Method Swizzling原理：**

Method Swizzing 主要用于在运行时将两个Method进行交换， 本质上是对IMP和SEL进行交换。我们可以将Method Swizzling代码写在任何地方，但只有在这段Method Swilzzling代码执行完毕之后互换才起作用


**Method Swizzling 示例**


```
#import <objc/runtime.h>

@implementation UIViewController (Swizzling)

+ (void)load {
 
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // When swizzling a class method, use the following:
        // Class aClass = object_getClass((id)self);
        Class aClass = [self class];
        
        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(cy_viewWillApper:);
        
        Method originalMethod = class_getInstanceMethod(aClass, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(aClass, swizzledSelector);
        
        //判断一下原有类中是否有要替换方法的实现,如果发现方法已经存在，会失败返回；如果方法没有存在,我们则添加被替换的方法的实现
        BOOL didAddMethod = class_addMethod(aClass, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        
        if (didAddMethod) {//说明当前类中没有要替换方法的实现
            class_replaceMethod(aClass, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        }else {//说明当前类中有要替换方法的实现，可以直接进行替换
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
        
    });
}

#pragma mark - Method Swizzling

- (void)cy_viewWillApper:(BOOL)animated {
    [self cy_viewWillApper:animated];
    NSLog(@"viewWillAppear: %@", self);
}

@end
```

注意：如果类中不存在要替换的方法，那就先用`class_addMethod`和`class_replaceMethod`函数进行添加和替换两个方法的实现；如果类中已经有了想要替换的方法，那么就调用`method_exchangeImplementations`函数交换了两个方法的 IMP，这是苹果提供给我们用于实现 Method Swizzling 的便捷方法。`method_exchangeImplementations`方法做的事情与如下的原子操作等价：

```
IMP imp1 = method_getImplementation(m1);
IMP imp2 = method_getImplementation(m2);
method_setImplementation(m1, imp2);
method_setImplementation(m2, imp1);
```

**Method Swizzling注意事项：**

**1、Swizzling应该总在+load中执行**

Objective-C在运行时会自动调用类的两个方法`+load`和`+initialize`。`+load`会在类初始加载时调用， `+initialize`方法是以懒加载的方式被调用的，如果程序一直没有给某个类或它的子类发送消息，那么这个类的 
`+initialize`方法是永远不会被调用的。所以和`+initialize`相比，`+load`能保证在类的初始化过程中被加载。

+load和+initialize的比较表


类型	   	|	+load 					|	+initialize
----------- | ------------------------- | ----------------
调用时机	|  初始化，被添加到runtime时|以懒加载的方式被调用
调用顺序	| 父类->子类->分类			|	父类->子类
调用次数	| 1次	                    |   多次
是否需要显式调用父类实现	|	否		|	否
是否沿用父类的实现			|	否		|	是
分类中的实现	| 	类和分类都执行		|	覆盖类中的方法，只执行分类的实现



**2、Swizzling应该总是在dispatch_once中执行**


Swizzling会改变全局状态，所以在运行时采取一些预防措施，使用dispatch_once就能够确保代码不管有多少线程都只被执行一次，并让其为一个原子操作，线程安全是很重要的。

**举个例子**：如果不写dispatch_once，并同时对NSArray和NSMutableArray中的objectAtIndex:方法都进行了Swizzling，这样可能会导致NSArray中的Swizzling失效。

**原因是**：如果这段Swizzling被执行多次，经过多次的交换IMP和SEL之后，结果可能就是未交换之前的状态。比如说父类A的B方法和子类C的D方法进行交换，交换一次后，父类A持有D方法的IMP，子类C持有B方法的IMP，但是再次交换一次，就又还原了。父类A还是持有B方法的IMP，子类C还是持有D方法的IMP，这样就相当于咩有交换。可以看出，如果不写dispatch_once，偶数次交换以后，相当于没有交换，Swizzling失效！

**3、Swizzling在+load中执行时，不要调用[super load]**

在+ (void)load方法中调用[super load]方法，这会导致父类的Swizzling被重复执行两次，这样父类的Swizzling就会失效。


**4、类方法使用 Class aClass = object_getClass((id)self)问题**

swizzling类方法的时候使用`object_getClass((id)self)`获取aClass

虽然`object_getClass((id)self) `与 `[self class]` 返回的结果类型都是 Class,但前者为元类,后者为其本身,因为此时 self 为 Class 而不是实例。

注意 [NSObject class] 与 [object class] 的区别：

```
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}
```

**5、cy_viewWillAppear:中调用[self cy_viewWillAppear: animated]造成死循环？**


在`cy_viewWillAppear:`方法的定义中看似是递归调用会引发死循环，其实是不会的。因为`[self cy_viewWillAppear:animated]`消息会动态找到`cy_viewWillAppear:`方法的实现，而它的实现已经被我们与`viewWillAppear:`方法实现进行了互换，所以这段代码不仅不会死循环，如果你把`[self cy_viewWillAppear:animated]`换成`[self viewWillAppear:animated]`反而会引发死循环。


# 代码及参考资料

[github源码地址](https://github.com/shijian123/RuntimeCodeDome) 代码涉及：动态方法解析、备用接收者、完整消息转发、Method Swizzling

参考资料：

[runtime官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)

[Objective-C +load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)

[Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)

[Objective-C 消息发送与转发机制原理](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)

[深入解析 ObjC 中方法的结构](https://github.com/draveness/analyze/blob/master/contents/objc/深入解析%20ObjC%20中方法的结构.md#深入解析-objc-中方法的结构)

[神经病院Objective-C Runtime入院第一天——isa和Class](https://www.jianshu.com/p/9d649ce6d0b8) 

[神经病院Objective-C Runtime住院第二天——消息发送与转发](https://www.jianshu.com/p/4d619b097e20)

[神经病院Objective-C Runtime出院第三天——如何正确使用Runtime](http://www.jianshu.com/p/db6dc23834e3)
