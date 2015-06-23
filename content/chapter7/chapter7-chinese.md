#第七章 无锁并发数据结构设计

**本章主要内容**

>实现无锁并发数据结构的设计<br>
>在无锁结构中内存管理技术<br>
>对无锁数据结构的简单指导<br>

在上一章中，我们了解了在设计并发数据结构时会遇到的问题，根据指导意见方向，确定我们的设计是安全的。我们会检查一些通用数据结构，并且看一下使用互斥锁对共享数据进行保护的实现例子。第一对例子就是使用一个互斥量来保护整个数据结构，不过后面的例子就使用了不止一个锁来保护数据结构不同的部分，并且允许对数据结构进行更高级别的并发访问。

互斥量是一个强大的工具，其可以保证在多线程情况下我们可以安全的访问数据结构，并且不会有条件竞争或破坏不变量的情况存在。对于使用互斥量的代码，其原因也是很简单的：就是让互斥量来保护数据。不过，这并不会如你所想的那样(*a bed of roses*)；你可以回看一下第3章，回顾一下错误的锁使用时如何造成死锁的，还可以看一下基于锁的队列和查询表的例子，看一下细粒度锁是如何潜在的影响真正的并发。如果你能写出一个无锁的并发安全的数据结构，那么就能避免这些潜在的问题。这样的数据结构叫做无锁(*lock-free*)数据结构。

在本章中，我们还会看到原子操作(第5章介绍)的“内存-顺序”特性，并使用这个特性来构建无锁数据结构。当你要设计这样的数据结构时，要格外的小心，因为这样的数据机构不是那么容易正确实现的，并且让其失败的条件很难复现。我们将从无锁数据的定义开始；而后，我们将继续通过几个例子来了解使用无锁数据结构的意义，而后给出一些通用的指导意见。

##7.1 定义和意义

，会使用互斥量，条件变量，以及“期望”来同步“阻塞”(*blocking*)数据的算法和数据结构。应用调用库函数，将会挂起一个执行线程，知道其他线程执行完某个特定的动作。库将调用阻塞操作来对线程进行阻塞，直到阻塞移除，线程才能继续处理自己的任务。通常，操作系统会完全挂起一个阻塞线程(并将其时间片交给其他线程)，直到其被其他线程“去阻”；“去阻”的方式很多，比如解锁一个互斥锁、提示一个条件变量达成或让一个“期望”就绪。

不使用阻塞库函数的数据结构和算法，被称为“无阻塞”(*nonblocking*)结构。“无阻塞”的数据结构并非都是无锁的(*lock-free*)，那么就让我们见识一下各种各样的“无阻塞”数据结构吧！

###7.1.1 非阻塞数据结构

回到第5章中，我们使用`std::atomic_flag`实现了一个简单的自旋锁。让我们再来看一下这段代码。

清单7.1 使用`std::atomic_flag`实现了一个简单的自旋锁
```c++
class spinlock_mutex
{
  std::atomic_flag flag;
public:
  spinlock_mutex():
    flag(ATOMIC_FLAG_INIT)
  {}
  void lock()
  {
    while(flag.test_and_set(std::memory_order_acquire));
  }
  void unlock()
  {
    flag.clear(std::memory_order_release);
  }
};
```

这段代码没有调用任何阻塞函数；lock()只是让循环持续调用test_and_set()，并返回false。这就是为什么取名为自旋锁的原因——代码“自旋”于循环当中。所以，这里没有阻塞调用，任意代码使用互斥量来保护共享数据都是非阻塞的。不过，这自旋锁不是无锁结构。这里还是用了一个锁，并且一次能锁住一个线程。让我们来看一下无锁结构的定义，这将有助于你判断哪些类型的数据结构是无锁的。

###7.1.2 无锁数据结构

将一个数据结构成为无锁结构，那么线程就可以并发的访问这个数据结构。线程不能做相同的操作；一个无锁队列可能允许一个线程进行压入数据，一个线程弹出数据，当有两个线程同时尝试添加元素时，这个数据结构将被破坏。不仅如此，当其中一个访问线程被调度器中途挂起时，其他线程必须能够继续完成他们自己的工作，不用等待挂起线程。

具有比较/交换操作的数据结构，通常在比较/交换实现中都有一个循环。使用比较/交换操作的原因：当有其他线程同时对对应数据的修改时，代码将恢复调用比较/交换操作前的数据。当其他线程被挂起时，且比较/交换操作执行成功，那么这样的代码就是无锁的。当执行失败时，你就需要一个自旋锁了，那么这个结构就是非阻塞且非无锁的结构。

无锁算法中的这中循环会让一些线程处于“饥饿”状态。如有线程在“错误”时间执行，那么第一个线程将会持续的尝试自己所要完成的操作 (其他程序继续执行)。无锁-无等待数据结构，就为了避免这种问题存在的。

###7.1.3 无等待数据结构

无等待数据结构就是，无锁数据结构加上每个线程都能在有限的步数内完成他们所要完成的操作，且不管其他线程是如何工作的。这里，由于会和别的线程产生冲突，所以算法可以进行无数次尝试，因此并不是无等待的。

正确实现一个无锁的结构是十分困难的。因为，要保证每一个线程都能在有限步骤里完成操作，你需要保证每一个操作可以被一次性执行完成，且当一个线程执行这个操作时，不会让其他线程的操作失败。这会让算法中所使用到的操作变的相当复杂。

考虑到获取无锁或无等待的数据结构所有权是很困难的，那么你就需要一些理由来写一个数据结构；需要保证的是所得获益要大于实现成本。那么，就让我们来找一下实现成本和所得获益的平衡点吧！

###7.1.4 无锁数据结构的利与弊

我们使用无锁结构的主要原因是：其能够将并发最大化。使用基于锁的容器，都会让一个线程阻塞或等待第一个线程完成其工作；这里一个互斥锁削弱了结构的并发性。在无锁数据结构中，某些线程可以通过每一步的操作，递进的完成工作。在无等待数据结构中，每一个线程都可以转发进度，无论其他线程当时在做什么；其目的就是不需要等待。这种理想的方式实现起来很难。这种结构因为太简单，而不容易写出来，因为其本质上就是一个自旋锁。

使用无锁数据结构的第二个原因就是鲁棒性。当一个线程在获取一个锁时，被杀死，那么数据结构将被破坏。不过，当线程在无锁数据结构上执行操作，执行到一半死亡时，那么数据结构上的数据没有丢失(除了线程本身的数据)；其他线程依旧可以正常执行。

另一方面，当你不能排除访问数据结构的线程时，就需要小心的确认不变量的状态，或选择替代品来代替不变量保持状态。同时，你还需要关注附加在操作上的顺序约束。为了避免未定义行为，及相关的数据竞争，就必须使用原子操作对修改操作进行限制。不过，仅使用原子操作时不够的；你需要确定能被其他线程看到的修改，是尊顺一个正确的顺序。所有的一切说明，想要在写一个无锁-线程安全的数据结构是十分困难的。

因为，没有任何锁(有可能存在活锁(*live locks*))，死锁问题就不会困扰无锁数据结构。活锁的产生是因为，两个线程同时尝试修改数据结构，不过每个线程所做的修改操作都会让另一个线程重启，所以两个线程就会陷入循环，多次的尝试完成自己的操作。试想有两个人要过独木桥。当两个人从两头向中间走的时候，他们会在中间碰到，然后不得不再走回出发的地方，再次尝试过独木桥。这里，要打破僵局，除非有人先到独木桥的另一端(或因商量好了，或因走的块，或纯粹是运气)，要不这个循环将一直重复下去。不过活锁的存在时间并不长，因为其依赖于精确的线程调度。所以其只是对性能有所消耗，而不是一个长期的问题；不过，这个问题仍需要关注。根据定义，无等待的代码不会被活锁所困扰，因为其操作执行步骤是有上限的。换个角度，无等待的算法要比等待算法的复杂度高，且即使没有其他线程访问数据结构，也可能需要更多步骤来完成对应操作。

这就是无锁-无等待代码的缺点：虽然提高了并发访问的能力，减少了单个线程的等待时间，但是其可能会将整体性能拉低。首先，使用原子操作的无锁代码，要慢于无原子操作的代码，原子操作就相当于无锁数据结构中的锁。不仅如此，硬件必须通过同一个原子变量对线程间的数据进行同步。在第8章，你将看到与“乒乓”缓存相关的原子变量(多个线程访问同时访问)，将会成为一个明显的性能瓶颈。在提交代码之前，对性能相关的方面进行检查是很重要的(最坏的等待时间，平均等待时间，整体执行时间，或者其他指标)，无论是基于锁的数据结构，还是无锁的数据结构。

先来看几个例子。

##7.2 无锁数据结构的例子

为了演示一些在设计无锁数据结构中所使用到的技术，我们将看到一些无锁实现的简单数据结构。这里不仅要在每个例子中描述一个有用的数据结构实现，还将使用这些例子的某些特别的点来阐述对于无锁数据结构的设计。

如之前所提到的，无锁结构体依赖与原子操作和内存顺序相关的保证，来确保数据结构能被多线程以正确的顺序访问。最初，我们对所有原子操作使用默认的memory_order_seq_cst内存序，因为简单，所以使用(所有memory_order_seq_cst都遵循一种顺序)。不过，在后面的例子中，我们将会降低内存序的要求，使用memory_order_acquire, memory_order_release, 甚至memory_order_relaxed。虽然这个例子中没有直接的使用锁，但需要注意的是`std::atomic_flag`能保证实现中无锁的使用。一些平台中的无锁结构实现，实际上在C++的标准库的实现中，使用了内部锁(详见第5章)。在另一些平台上，基于锁的简单数据结构可能会更适合，不过还有很多平台不能一一说明；在选择一种实现前，你需要明确你的需求，并且配置各种选项以满足要求。

那么，回到数据结构上来吧，最简单的数据结构——栈。

###7.2.1 写一个无锁的线程安全栈

栈的要求很简单：查询节点的顺序是添加顺序的逆序——先入后出(LIFO)。所以，要确保一个值安全的添加入栈就十分重要，因为其可能马上被其他线程索引到，并且确保只有一个线程能索引到给定值也是很重要的。最简单的栈就是链表；head指针指向第一个节点(可能是下一个被索引到的节点)，并且每个节点依次指向下一个节点。

在这样的情况下，添加一个节点相对来说很简单：

1. 创建一个新节点。
2. 将当新节点的next指针指向当前的head节点。
3. 让head节点指向新节点。

在只有单线程的上下文中，这种方式没有问题，不过当多线程对栈进行修改时，这几步就不够用了。至关重要的是，当有两个线程同时添加节点的时候，在第二步和第三步的时候会产生条件竞争：一个线程可能在修改head的值时，另一个线程正在执行第2步，并且在第3步中对head进行更新。这就会使之前那个线程的工作被丢弃，亦或是造成更加糟糕后果。在我们了解如何解决这个条件竞争之前，还要注意一个很重要的事：当head被更新，并指向了新节点，另一个线程就能读取到这个节点了。因此，在head设置为指向新节点前，让新节点完全准备就绪就变得很重要了；因为，在这之后你就不能对节点进行修改了。

OK，那如何应对讨厌的条件竞争呢？答案就是：在第3步的时候使用一个原子“比较/交换”操作，来保证当步骤2对head进行读取的时候，不会对head进行修改。当有修改时，你可以循环做“比较/交换”操作。下面的代码就展示了不用锁来实现线程安全的push()函数。

清单7.2 不用锁实现push()
```c++
template<typename T>
class lock_free_stack
{
private:
  struct node
  {
    T data;
    node* next;

    node(T const& data_):  // 1
     data(data_)
    {}
  };

  std::atomic<node*> head;
public:
  void push(T const& data)
  {
    node* const new_node=new node(data); // 2
    new_node->next=head.load();  // 3
    while(!head.compare_exchange_weak(new_node->next,new_node));  // 4
  }
};
```

上面代码近乎能匹配之前所说的三个步骤：创建一个新节点②，设置新节点的next指针指向当前head③，并且设置head指针指向新节点④。node结构用其自身的构造函数来进行数据填充①，必须保证节点在构造完成后随时能被弹出。而后需要使用compare_exchange_weak()来保证在被存储到new_node->next的head指针和之前的一样③。代码的亮点是使用比较/交换操作：当其返回false时，以为这比较失败(例如，head被其他线程锁修改)，new_node->next作为操作的第一个参数，将会更新head的当前值。循环中不需要每次都重新加载head指针，因为编译器会帮你完成这件事。同样的，因为循环可能直接就失败了，所以这里使用compare_exchange_weak要好于使用compare_exchange_strong(详见第5章)。

所以，这里可能就不需要pop()操作了，可以快速检查一下push()的实现是否有违背指导意见。这里唯一一个能抛出异常的地方就是在构造新node的时候①，但其会自行处理，并且链表中的内容没有被修改，所以这里是很安全的。因为在构建数据的时候，是将其作为node的一部分作为存储，并且使用compare_exchange_weak()来更新head指针，这里也没有恶性的条件竞争。在比较/交换成功时，节点已经准备就绪，且随时可以提取。因为这里没有锁，所以就不存在死锁的情况，这里的push()函数实现的很成功。

那么，你现在已经有往栈中添加数据的方法了，现在需要删除数据的方法。其步骤如下，也很简单：

1. 读取当前head指针的值。
2. 读取head->next。
3. 设置head到head->next。
4. 通过索引node，返回data数据。
5. 删除索引节点。

不过在多线程环境下，就不像看起来那么简单了。当有两个线程要从栈中移除数据，两个线程可能在步骤1中读取到同一个head(值相同)。当其中一个线程处理到步骤5，而另一个线程还在处理步骤2时，这个还在处理步骤2的线程将会解引用一个悬空指针。这只是写无锁代码所遇到的最大问题之一，所以现在只能跳过步骤5，让节点泄露。

这样也不会解决所有问题。另一个问题：当两个线程读取到head的值是同一个，他们将返回同一个节点。这就违反了栈结构的意图，所以你需要避免这样的问题。你可以像在push()函数中解决条件竞争那样来解决这个问题：使用比较/交换操作更新head。当比较/交换操作失败时，不是一个新节点已被推入，就是其他线程已经弹出了你想要弹出的节点。无论是那种情况，你都得返回步骤1(比较/交换操作将会重新读取head)。

当比较/交换调用成功，你就可以知道你的线程是弹出给定节点的唯一线程，那么你就可以放心的执行步骤4了。这里先看一下pop()的雏形：

```c++
template<typename T>
class lock_free_stack
{
public:
  void pop(T& result)
  {
    node* old_head=head.load();
    while(!head.compare_exchange_weak(old_head,old_head->next));
    result=old_head->data;
  }
};
```

虽然这段代码很优雅，但是这里还有两个有关节点泄露的问题。首先，这段代码在空链表的时候不工作：当head指针式一个空指针时，当要访问next指针时，其将引起未定义行为。这很容易通过对nullptr的检查进行修复(在while循环中)，要不对空栈抛出一个异常，要不返回一个bool值来表明成功与否。

第二个问题就是异常安全问题。当在第3章中第一次介绍栈结构时，了解了在返回值的时候会出现异常安全问题：当有异常被抛出时，复制的值将丢失。在这种情况下，传入引用是一种可以接受的解决方案，因为这样你就能保证，当有异常抛出时，栈上的值不会丢失。不幸的是，现在你不能这样做；你只能在确定只有单一线程对值进行返回的时候，才进行拷贝，以确保拷贝操作的安全性，这就意味着在拷贝结束后这个节点就被删除了。因此，通过引用的方式获取返回值的方式就没有任何优势了：直接返回值这里也是可以的。想要安全的返回值，你必须使用第3章中的其他方法：返回指向数据值的(智能)指针。

当返回的是一个智能指针，你就能返回nullptr来表明没有值可返回，但是要求在堆上对智能指针进行内存分配。当将分配过程做为pop()的一部分时(也没有更好的选择了)，堆分配时可能会抛出一个异常。与此相反的是，你可以在push()操作中对内存进行分配——不管怎么样，你都得对node进行内存分配。返回一个`std::shared_ptr<>`不会抛出任何异常，所以在pop()中进行分配就是安全的。将上面的观点放在一起，就能看到如下的代码。

清单7.3 带有节点泄露的无锁栈
```c++
template<typename T>
class lock_free_stack
{
private:
  struct node
  {
    std::shared_ptr<T> data;  // 1 指针获取数据
    node* next;

    node(T const& data_):
      data(std::make_shared<T>(data_))  // 2 让std::shared_ptr指向新分配出来的T
    {}
  };

  std::atomic<node*> head;
public:
  void push(T const& data)
  {
    node* const new_node=new node(data);
    new_node->next=head.load();
    while(!head.compare_exchange_weak(new_node->next,new_node));
  }
  std::shared_ptr<T> pop()
  {
    node* old_head=head.load();
    while(old_head && // 3 在解引用前检查old_head是否为空指针
      !head.compare_exchange_weak(old_head,old_head->next));
    return old_head ? old_head->data : std::shared_ptr<T>();  // 4
  }
};
```

智能指针指向当前数据①，所以这里必须在堆上为数据分配内存(在node结构体中)②。而后，在compare_exchage_weak()循环中③，需要在old_head指针前，检查其是否为空。最终，当存在相关节点，那么将会返回相关节点的值；当不存在时，将返回一个空指针④。注意，这个结构是无锁的，但并不是无等待的，因为在push()和pop()函数中都有while循环，那么当compare_exchange_weak()总是失败的前提下，循环将会无限制的循环下去。

###7.2.2 停止内存泄露：使用无锁数据结构管理内存

在我们第一次了解pop()时，我们为了避免条件竞争(当有一个线程删除一个节点的同时，其他线程还持有指向该节点的指针，并要解引用)选择了带有内存泄露的节点。但是，不论什么样的C++程序，存在内存泄露都是不可接受的。因此，我们必须为其做些什么。现在就是解决这个问题的时间了！

基本问题在于，当想要释放一个节点时，需要确认没有其他线程持有这个节点。当只有一个线程在这个特殊的栈实现上调用pop()，就可以放心的释放。当节点添加入栈后，push()就不会与其有任何的关系了，所以线程这里只有调用pop()函数的线程与已加入节点有关，并且能够安全的将其删除。

从另一方面讲，当你需要一个栈实现上同时处理多线程对pop()的调用时，你需要知道节点在什么时候删除的。这实际上就需要你写一个节点专用的垃圾收集器。这听起来有些可怖，同时也相当棘手，不过也不是那么的糟糕：你需要检查节点，并且检查哪些节点通过pop()进行了访问。你不需要对push()中的节点有所担心，因为这些节点只有推到栈上以后，才能被访问到，而多线程只能通过pop()访问同一节点。

当没有线程调用pop()时，这时可以删除栈上的任意节点。因此，当你添加节点到“可删除”列表中是，你就能从中提取数据了。而后，在没有线程通过pop()访问这些节点时，你就可以安全的删除它们了。那怎么知道没有任何线程调用pop()呢？很简单——计数即可。当计数器数值增加时，就是有节点进入；当减少时，就是有节点被删除。这样从“可删除”列表中删除节点就很安全了，直到计数器的值为0。当然，这个计数器必须是原子的，这样它才能在多线程的情况下正确的进行计数。下面的清单中，展示了修改后的pop()函数，其中有些支持功能的实现将在清单7.5中给出。

清单7.4 当没有线程通过pop()访问节点时，对节点进行回收
```c++
template<typename T>
class lock_free_stack
{
private:
  std::atomic<unsigned> threads_in_pop;  // 1 原子变量
  void try_reclaim(node* old_head);
public:
  std::shared_ptr<T> pop()
  {
    ++threads_in_pop;  // 2 在做事之前，计数值加1
    node* old_head=head.load();
    while(old_head &&
      !head.compare_exchange_weak(old_head,old_head->next));
    std::shared_ptr<T> res;
    if(old_head)
    { 
      res.swap(old_head->data);  // 3 回收删除的节点
    }
    try_reclaim(old_head);  // 4 从节点中直接提取数据，而非拷贝指针
    return res;
  }
};
```

这里threads_in_pop①原子变量用来记录有多少线程试图弹出栈中的元素。当pop()②函数调用的时候，计数器加一；当调用try_reclaim()时，计数器减一，当这个函数被节点调用时，说明这个节点已经被删除④。因为暂时不需要将节点本身删除，所以你可以通过swap()函数来删除节点上的数据③(而非只是拷贝指针)，所以当你不再需要这些数据的时候，这些数据会自动删除，而不是持续存在着(因为这里还有对未删除节点的引用)。下面清单中将看一下try_reclaim()是如何实现的。

清单7.5 采用引用计数的回收机制
```c++
template<typename T>
class lock_free_stack
{
private:
  std::atomic<node*> to_be_deleted;

  static void delete_nodes(node* nodes)
  {
    while(nodes)
    {
      node* next=nodes->next;
      delete nodes;
      nodes=next;
    }
  }
  void try_reclaim(node* old_head)
  {
    if(threads_in_pop==1)  // 1
    {
      node* nodes_to_delete=to_be_deleted.exchange(nullptr);  // 2 声明“可删除”列表
      if(!--threads_in_pop)  // 3 是否只有一个线程调用pop()？
      {
        delete_nodes(nodes_to_delete);  // 4
      }
      else if(nodes_to_delete)  // 5
      {
         chain_pending_nodes(nodes_to_delete);  // 6
      }
      delete old_head;  // 7
    }
    else
    {
      chain_pending_node(old_head);  // 8
      --threads_in_pop;
    }
  }
  void chain_pending_nodes(node* nodes)
  {
    node* last=nodes;
    while(node* const next=last->next)  // 9 让next指针指向链表的末尾
    {
      last=next;
    }
    chain_pending_nodes(nodes,last);
  }

  void chain_pending_nodes(node* first,node* last)
  {
    last->next=to_be_deleted;  // 10
    while(!to_be_deleted.compare_exchange_weak(  // 11 用循环来保证last->next的正确性
      last->next,first));
    }
    void chain_pending_node(node* n)
    {
      chain_pending_nodes(n,n);  // 12
    }
};
```

当回收节点时①，threads_in_pop的数值是1，也就是只有当前线程正在访问pop()，所以就可以安全的对节点进行删除了⑦,这个时候将等待的节点删除应该也是安全的。当数值不是1的时候，删除任何节点都是不安全的，所以需要继续向等待列表中添加节点⑧。

假设在某一时刻，threads_in_pop的值为1。那么就可以尝试回收等待列表了；如果不回收，那么这节点就会持续等待，直到这个栈被销毁。要做到回收，首先要通过一个原子exchange操作声明②声明删除列表，并且将计数器减一③。当减一后计数值为0，这就意味着没有其他线程访问等待节点列表。可能会出现新的等待节点，不过你现在不必为其所烦恼了，因为它们将从被安全的回收。而后，你可以使用delete_nodes对链表进行迭代，并将其删除④。

当计数值在减一后不为0，那么回收节点就是不安全的，所以如果存在⑤，就绪要将其挂在等待删除列表的后面⑥。这种情况会发生在多个线程同时访问数据结构的时候。一些线程在第一次测试threads_in_pop①和对“回收”列表的声明②操作间调用pop()，着可能新填入一个已经被一个或多个线程访问的节点到链表中。在图7.1中，线程C添加节点Y到to_be_deleted列表中，即使线程B仍将其引用作为old_head，之后会尝试访问其next指针。在线程A删除节点的时候，会造成线程B端发生未定义的行为。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter7/7-1.png)

图7.1 三个线程同时调用pop()，说明为什么要在try_reclaim()对声明节点进行删除前，对threads_in_pop进行检查。

为了将等待删除的节点添加入等待删除列表中，你需要复用节点的next指针将等待删除节点链接在一起。在这种情况下，将已存在的链表链接到链表后面，可以通过遍历的方式找到链表的末尾⑨，将最后一个节点的next指针替换为当前to_be_deleted指针⑩，并且将链表中的第一个节点作为新的to_be_deleted指针进行存储⑪。这里你需要在循环中使用compare_exchange_weak来确保通过其他线程添加进来的节点不会发生内存泄露。这还有个好处，就是在链表发生改变时，更新next指针很方便。添加单个节点是一种特殊的情况，因为这需要将这个节点作为第一个节点，同时也是最后一个节点进行添加⑫。

在低负荷的情况下，这种方式没有问题，因为在没有线程访问pop()时，这里有一个合适的静态指针。不过，这只是一个瞬时的状态，也就是为什么在回收前，需要检查threads_in_pop计数为0③的原因；同样也是删除节点⑦前进行对计数器检查的原因。删除一个节点是一项耗时的工作，并且你想要其他线程能对列表做的修改越小越好。从第一发现threads_in_pop是1的时候到尝试删除节点的时候，用时很长，这样就会有很多线程有机会调用pop()，这就会让threads_in_pop不为0，这样就会阻止节点的删除操作。

在高负荷的情况，就不会存在静态；因为，其他线程在初始化之后，都能进入pop()。在这样的情况下，to_ne_deleted列表将会无界的增加，并且这里会再次泄露。当这里不存在任何静态的情况，就得位回收节点寻找替代机制。关键是要确定，当没有线程访问一个特殊的线程，那么这个节点就能被回收。现在，最简单的替换机制就是使用“风险指针”(*hazard pointer*)。

###7.2.3 检测使用风险指针(不可回收)的节点

“风险指针”这个术语引用于迈克尔·马吉德的一个技术发现[1]。之所以这样加，是因为删除一个节点可能会让其他引用其的线程处于危险之中。当其他线程持有这个删除的节点的指针，并且解引用进行操作的时候，将会出现未定义行为。基本观点就是，当一个线程去访问一个要被其他线程删除的对象，其会先设置对这个对象设置一个风险指针，而后通知其他线程，删除这个指针是一个危险的行为。一旦这个对象不再被需要，那么风险指针就可以清除了。如果你看过牛津/剑桥的龙舟比赛，那么这里使用到的机制和龙舟比赛开赛时差不多：每个船上的舵手都举起手来，以表示他们还没有准备好。只要有舵手的手是举着的，那么裁判就不能让比赛开始。当所有舵手的手都放下后，比赛才能开始；在比赛还未开始或感觉自己船队的情况有变时，舵手可以再次举手。

当有线程想要删除一个对象，那么它就必须检查系统中其他线程是否持有风险指针。当没有风险指针的时候，那么它就可以安全删除对象了。否则，它就必须等待风险指针的消失了。这样，线程就得周期性的检查其想要删除的对象是否能安全删除。

高水平的描述，让其听起来很简单，那在C++中应该怎么做呢？

那么，首先你需要一个地点能存储指向访问对象的指针，这个地点就是风险指针。这个地点必须能让所有线程看到，需要其中一些线程可以对数据结构进行访问。如何正确和高效的分配这些线程，的确是一个挑战，所以这个问题可以放在后面解决，然后假设你有一个get_hazard_pointer_for_current_thread()的函数，这个函数可以返回风险指针的引用。当你读取一个指针，并且想要解引用它的时候，你就需要这个函数——在这种情况下head数值源于下面的列表：

```c++
std::shared_ptr<T> pop()
{
  std::atomic<void*>& hp=get_hazard_pointer_for_current_thread();
  node* old_head=head.load();  // 1
  node* temp;
  do
  {
    temp=old_head;
    hp.store(old_head);  // 2
    old_head=head.load();  // 3
  } while(old_head!=temp);
  // ...
}
```

在while循环中的好处是，能保证node不会在读取旧head指针①，和设置风险指针的时候被删除。在这中模式下，没有其他线程能知道你对这个特殊的节点进行了访问。幸运的是，当旧head节点将要被删除时，head本身是要改变的，所以可以对head进行检查，并持续循环，直到你head指针中的值与风险指针中的值相同③。使用风险指针，实际上如同依赖已删除对象的引用。当你使用默认的new和delete操作对风险指针进行操作的时候，会出现未定义行为，所以你要确定你的实现是否支持这样的操作，或你需要使用自定义分配器来保证这种用法的正确性。

现在你已经设置了风险指针，那就可以对pop()进行下处理了，基于现在了解到的安全知识，这里不会有其他线程来删除节点。啊哈！那么每一次重新加载old_head时，在你解引用刚刚读取到的指针，你就需要更新风险指针。当你从链表中提取一个节点时，你就可以将风险指针清除了。如果没有其他风险指针引用节点，那么就可以安全的删除节点了；否则，你就需要将其添加到链表中，在之后将其删除。下面的代码就是用这样的方案，完整的实现了pop()。

清单7.6 使用风险指针是实现的pop()
```c++
std::shared_ptr<T> pop()
{
  std::atomic<void*>& hp=get_hazard_pointer_for_current_thread();
  node* old_head=head.load();
  do
  {
    node* temp;
    do  // 1 直到将风险指针设为head指针
    {
      temp=old_head;
      hp.store(old_head);
      old_head=head.load();
    } while(old_head!=temp);
  }
  while(old_head &&
    !head.compare_exchange_strong(old_head,old_head->next));
  hp.store(nullptr);  // 2 当声明完成，清除风险指针
  std::shared_ptr<T> res;
  if(old_head)
  {
    res.swap(old_head->data);
    if(outstanding_hazard_pointers_for(old_head))  // 3 在删除之前对风险指针引用的节点进行检查
    {
      reclaim_later(old_head);  // 4
    }
    else
    {
      delete old_head;  // 5
    }
    delete_nodes_with_no_hazards();  // 6
  }
  return res;
}
```

首先，循环内部会对风险指针进行设置，在当比较/交换操作失败会重载old_head，再次进行设置①。这里使用compare_exchange_strong()，是因为需要在循环内部做一些实际的工作：当compare_exchange_weak()伪失败后，风险指针将被重置，这是没有必要的。这个过程就能保证风险指针在解引用(old_head)之前，能被正确的设置。当你声明了一个风险指针，那么你就可以将其清除了②。如果想要获取一个节点，你需要检查其他线程上的风险指针，看一下是否有其他指针该节点进行了引用③。如果有，那么你就不能删除那个节点，只能将其放在链表中，在之后再进行回收④；要不，你就能直接将这个节点删除了⑤。最后，如果需要对任意节点进行检查，那么你可以调用reclaim_later()。如果链表上没有任何风险指针引用节点，那么就可以安全的删除这些节点了⑥。当有任意节点持有风险指针，就能让下一个调用pop()函数的线程离开。

当然，这些函数——get_hazard_pointer_for_current_thread(), reclaim_later(), outstanding_hazard_pointers_for(), 和delete_nodes_with_no_hazards()——的细节我们还没有看到，那我们就来看看它们是如何工作的。

为线程分配风险指针指针实例的具体方案，是使用get_hazard_pointer_for_current_thread()与程序逻辑的关系并不大(不过会影响效率，接下会看到具体的情况)。那么现在你可以使用一个简单的结构体：固定长度的“线程ID-指针”数组。get_hazard_pointer_for_curent_thread()就可以通过这个数据来找到第一个释放槽，并将当前线程的ID放入到这个槽中。当线程退出时，这个槽再次置空，可以通过默认构造函数`std::thread::id()`将线程ID放入槽中。这个实现就如下所示：

清单7.7 get_hazard_pointer_for_current_thread()函数的简单实现
```c++
unsigned const max_hazard_pointers=100;
struct hazard_pointer
{
  std::atomic<std::thread::id> id;
  std::atomic<void*> pointer;
};
hazard_pointer hazard_pointers[max_hazard_pointers];

class hp_owner
{
  hazard_pointer* hp;

public:
  hp_owner(hp_owner const&)=delete;
  hp_owner operator=(hp_owner const&)=delete;
  hp_owner():
    hp(nullptr)
  {
    for(unsigned i=0;i<max_hazard_pointers;++i)
    {
      std::thread::id old_id;
      if(hazard_pointers[i].id.compare_exchange_strong(  // 6 尝试声明风险指针的所有权
         old_id,std::this_thread::get_id()))
      {
        hp=&hazard_pointers[i];
        break;  // 7
      }
    }
    if(!hp)  // 1
    {
      throw std::runtime_error("No hazard pointers available");
    }
  }

  std::atomic<void*>& get_pointer()
  {
    return hp->pointer;
  }

  ~hp_owner()  // 2
  {
    hp->pointer.store(nullptr);  // 8
    hp->id.store(std::thread::id());  // 9
  }
};

std::atomic<void*>& get_hazard_pointer_for_current_thread()  // 3
{
  thread_local static hp_owner hazard;  // 4 每个线程都有自己的风险指针
  return hazard.get_pointer();  // 5
}
```

对get_hazard_pointer_for_current_thread()的实现看起来很简单③：其有一个hp_owner④类型的thread_local(本线程所有)变量，就是为存储当前线程的风险指针而存在的。这个变量返回对象所持有的指针⑤。下面的工作就是：当第一次有线程调用这个函数时，一个新的hp_owner实例就被创建了。这个实例的构造函数⑥，将会通过查询“所有者/指针”表，寻找没有所有者的入口。其用compare_exchange_strong()来检查一个入口是否有所有权，并且将一个线程放入这个入口②。当compare_exchange_strong()失败，其他线程的所有者将进入这个入口，所以继续下去就行。当交换成功，你就成功的让当前线程进入了这个入口，所以这里对其进行存储，然后停止搜索⑦。当遍历了列表也没有找到空入口①，这就说明有很多线程都在使用风险指针，所以这里将抛出一个异常。

一旦hp_owner实例被一个给定的线程所创建，那么之后的访问将会很快，因为指针在缓存中，所以表不需要再次遍历。

当每个线程退出时，hp_owner的实例将会被销毁。析构函数会在通过`std::thread::id()`设置拥有者ID前，将指针重置为nullptr,这样就允许其他线程对这个入口进行再次使用⑧⑨。

在实现get_hazard_pointer_for_current_thread()后，outstanding_hazard_pointer_for()的实现就简单了：只需要对风险指针表进行搜索，就可以找到入口。

```c++
bool outstanding_hazard_pointers_for(void* p)
{
  for(unsigned i=0;i<max_hazard_pointers;++i)
  {
    if(hazard_pointers[i].pointer.load()==p)
    {
      return true;
    }
  }
  return false;
}
```

这里的实现都不需要对入口的所有者进行验证：没有所有者的入口将会使一个空指针，所以比较代码将总返回false，通过这种方式将代码极大程度的简化。

reclaim_later()和delete_nodes_with_no_hazards()可以对简单的链表进行操作；reclaim_later()只是将节点添加到列表中，delete_nodes_with_no_hazards()就是搜索整个列表，并将无风险指针的入口进行删除。那么下一个清单中将展示它们的具体实现。

清单7.8 回收函数的简单实现
```c++
template<typename T>
void do_delete(void* p)
{
  delete static_cast<T*>(p);
}

struct data_to_reclaim
{
  void* data;
  std::function<void(void*)> deleter;
  data_to_reclaim* next;

  template<typename T>
  data_to_reclaim(T* p):  // 1
    data(p),
    deleter(&do_delete<T>),
    next(0)
  {}

  ~data_to_reclaim()
  {
    deleter(data);  // 2
  }
};

std::atomic<data_to_reclaim*> nodes_to_reclaim;

void add_to_reclaim_list(data_to_reclaim* node)  // 3
{
  node->next=nodes_to_reclaim.load();
  while(!nodes_to_reclaim.compare_exchange_weak(node->next,node));
}

template<typename T>
void reclaim_later(T* data)  // 4
{
  add_to_reclaim_list(new data_to_reclaim(data));  // 5
}

void delete_nodes_with_no_hazards()
{
  data_to_reclaim* current=nodes_to_reclaim.exchange(nullptr);  // 6
  while(current)
  {
    data_to_reclaim* const next=current->next;
    if(!outstanding_hazard_pointers_for(current->data))  // 7
    {
      delete current;  // 8
    }
    else
    {
      add_to_reclaim_list(current);  // 9
    }
    current=next;
  }
}
```

**对风险指针(较好)的回收策略**

###7.2.4 检测使用引用计数的节点

###7.2.5 应用于无锁栈上的内存模型

###7.2.6 写一个无锁的线程安全队列

**push()中多线程的处理**

**让无锁队列帮助更多的线程**

##7.3 对于设计无锁数据结构的指导建议

###7.3.1 指导建议：使用`std::memory_order_seq_cst`的原型

###7.3.2 指导建议：对无锁内存的回收策略

###7.3.3 指导建议：小心[ABA问题](https://en.wikipedia.org/wiki/ABA_problem)

###7.3.4 指导建议：识别忙等待循环和帮助其他线程

##7.4 小结

###
1 “Safe Memory Reclamation for Dynamic Lock-Free Objects Using Atomic Reads and Writes,” Maged M.
Michael, *in PODC ’02: Proceedings of the Twenty-first Annual Symposium on Principles of Distributed Computing*
(2002), ISBN 1-58113-485-1.