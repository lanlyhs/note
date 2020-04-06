# sync.Pool

sync.Pool 类型可以被称为临时对象池，它的值可以被用来存储临时的对象。与 Go 语言的很多同步工具一样，sync.Pool 类型也属于结构体类型，它的值在被真正使用之后，就不应该再被复制了。

这里的“临时对象”的意思是：不需要持久使用的某一类值。这类值对于程序来说可有可无，但如果有的话会明显更好。它们的创建和销毁可以在任何时候发生，并且完全不会影响到程序的功能。

我们可以把临时对象池当作针对某种数据的缓存来用。实际上，在我看来，临时对象池最主要的用途就在于此。

sync.Pool 类型只有两个方法 Put 和 Get。Put 用于在当前的池中存放临时对象，它接受一个 interface{} 类型的参数；而 Get 则被用于从当前的池中获取临时对象，它会返回一个 interface{}类型的值。

更具体地说，这个类型的 Get 方法可能会从当前的池中删除掉任何一个值，然后把这个值作为结果返回。如果此时当前的池中没有任何值，那么这个方法就会使用当前池的 New 字段创建一个新值，并直接将其返回。

sync.Pool 类型的 New 字段代表着创建临时对象的函数。它的类型是没有参数但有唯一结果的函数类型，即：func() interface{}。

这里的 New 字段的实际值需要我们在初始化临时对象池的时候就给定。否则，在我们调用它的 Get 方法的时候就有可能会得到 nil。

### 池清理函数

sync 包在被初始化的时候，会向 Go 语言运行时系统注册一个函数，这个函数的功能就是清除所有已创建的临时对象池中的值。我们可以把它称为池清理函数。

一旦池清理函数被注册到了 Go 语言运行时系统，后者在每次即将执行垃圾回收时就都会执行前者。

### 池汇总列表

另外，在 sync 包中还有一个包级私有的全局变量。这个变量代表了当前的程序中使用的所有临时对象池的汇总，它是元素类型为 `*sync.Pool` 的切片。我们可以称之为池汇总列表。

在一个临时对象池的 Put 方法或 Get 方法第一次被调用的时候，这个池就会被添加到池汇总列表中。正因为如此，池清理函数总是能访问到所有正在被真正使用的临时对象池。

池清理函数会遍历池汇总列表。对于其中的每一个临时对象池，它都会先将池中所有的私有临时对象和共享临时对象列表都置为 nil，然后再把这个池中的所有本地池列表都销毁掉。

最后，池清理函数会把池汇总列表重置为空的切片。如此一来，这些池中存储的临时对象就全部被清除干净了。

## 问题 1：临时对象池存储值所用的数据结构是怎样的？

在临时对象池中，有一个多层的数据结构。正因为有了它的存在，临时对象池才能够非常高效地存储大量的值。

这个数据结构的顶层，我们可以称之为本地池列表，不过更确切地说，它是一个数组。这个列表的长度，总是与 Go 语言调度器中的 P 的数量相同。

P 存在的一个很重要的原因是为了分散并发程序的执行压力，而让临时对象池中的本地池列表的长度与 P 的数量相同的主要原因也是分散压力。这里所说的压力包括了存储和性能两个方面。在说明它们之前，我们先来探索一下临时对象池中的那个数据结构。

### private / shared / sync.Mutex

在本地池列表中的每个本地池都包含了三个字段（或者说组件），它们是：存储私有临时对象的字段 private 、代表了共享临时对象列表的字段 shared ，以及一个 sync.Mutex 类型的嵌入字段。

在程序调用临时对象池的 Put 方法或 Get 方法的时候，总会先试图从该临时对象池的本地池列表中，获取与之对应的本地池，依据的就是与当前的 goroutine 关联的那个 P 的 ID。

换句话说，一个临时对象池的 Put 方法或 Get 方法会获取到哪一个本地池，完全取决于调用它的代码所在的 goroutine 关联的那个 P。

## 问题 2：临时对象池是怎样利用内部数据结构来存取值的？

一个本地池的 shared 字段原则上可以被任何 goroutine 中的代码访问到，不论这个 goroutine 关联的是哪一个 P。这也是我把它叫做共享临时对象列表的原因。

相比之下，一个本地池的 private 字段，只可能被与之对应的那个 P 所关联的 goroutine 中的代码访问到，所以可以说，它是 P 级私有的。

### put

临时对象池的 Put 方法总会先试图把新的临时对象，存储到对应的本地池的 private 字段中，以便在后面获取临时对象的时候，可以快速地拿到一个可用的值。

只有当这个 private 字段已经存有某个值时，该方法才会去访问本地池的 shared 字段。Put 方法会在互斥锁的保护下，把新的临时对象追加到共享临时对象列表的末尾。

### get

临时对象池的 Get 方法，总会先试图从对应的本地池的 private 字段处获取一个临时对象。只有当这个 private 字段的值为 nil 时，它才会去访问本地池的 shared 字段。它会在互斥锁的保护下，试图把该共享临时对象列表中的最后一个元素值取出并作为结果。

不过，这里的共享临时对象列表也可能是空的，这可能是由于这个本地池中的所有临时对象都已经被取走了，也可能是当前的临时对象池刚被清理过。无论原因是什么，Get 方法都会去访问当前的临时对象池中的所有本地池，它会去逐个搜索它们的共享临时对象列表。只要发现某个共享临时对象列表中包含元素值，它就会把该列表的最后一个元素值取出并作为结果返回。

即使这样也可能无法拿到一个可用的临时对象，比如，在所有的临时对象池都刚被大清洗的情况下就会是如此。这时，Get 方法就会使出最后的手段调用可创建临时对象的那个函数。

这个函数是由临时对象池的 New 字段代表的，并且需要我们在初始化临时对象池的时候给定。如果这个字段的值是 nil，那么 Get 方法此时也只能返回 nil 了。