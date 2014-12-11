#第二章 线程管理

**本章主要内容**

>启用线程，让指定代码在新线程上运行<br>
>等待线程结束与置之不理继续运行<br>
>唯一的线程标识<br>

好的！看来你已经决定在应用中使用并发和多线程了。现在做点什么呢？如何去启动线程？如何知道它们已经结束了？还是，如何持续监视它们？C++标准库让线程管理变得相对容易很多，你将在后面看到，只需要管理`std::thread`对象中关联的线程即可。不过，标准库的灵活性，会让这些任务不会如同想象的那么简单。
在这一章，我将从一些基本的东西开始说起：启动一个线程，等待其结束，或让它在后台运行。再看看怎么给已经启动的线程函数传递参数，以及怎么讲一个线程的所有权从一个`std::thread`对象移交给另一个。最后，我们来看看怎么确定线程数，以及识别特殊的线程。

##2.1 线程管理基础

每一个C++程序至少有一个线程：线程执行**main()**函数。你的程序可以添加额外的线程，使用其他的函数作为入口点。之后这些线程与原始线程（以main()函数作为入口点的线程）同时运行。如同程序在**mian()**函数执行完会退出一样，当线程中的入口函数执行完成，那么线程也就会退出。如你将看到的那样，如果你为一个线程创建了一个`std::thread`对象，你需要等待这个线程结束；但前提是，你得启动它。下面我们就来启动线程。

###2.1.1 启动线程

如同第一章，线程在一个`std::thread`对象创建（为线程指定任务）时启动。在最简单的情况下，任务也会很简单，通常是无参数无返回（*void-returning*）的函数。这种函数在其所属线程上运行，直到函数执行完毕，线程也就结束了。在一些极端情况下，在线程运行时，任务中的函数对象需要通过某种通讯系统进行参数的传递，或者执行一系列独立操作；线程的停止，也是通过通讯系统传递信号进行控制的。线程要做什么或从哪里启动，其实都无关紧要。不过，使用C++线程库启动线程，通常都可以归结为构造了`std::thread`对象：

```c++
void do_some_work();
std::thread my_thread(do_some_work);
```

这只是一个简单的例子。当让，你也要保证`<thread>`头文件包含其中，能让编译器识别`std::thread`类。如同大多数C++标准库一样，`std::thread`可用于任何可调用（*callable*）类型，你可以将带有函数调用符类型的实例传入`std::thread`类中，用来替换默认的构造函数。

```c++
class background_task
{
public:
  void operator()() const
  {
    do_something();
    do_something_else();
  }
};
background_task f;
std::thread my_thread(f);
```

在这个例子中，所提供的函数对象会复制到新线程的存储空间当中，函数对象的执行和调用都在那进行的。函数对象的副本与原始的函数对象行为是相同的，但是函数对象副本得到的结果，有时却与我们期望的不同。
有件事需要注意，当把函数对象传入到线程构造函数中时，需要避免叫做“[最令人头痛的语法解析](http://en.wikipedia.org/wiki/Most_vexing_parse)”(*C++’s most vexing parse*, [中文简介](http://qiezhuifeng.diandian.com/post/2012-08-27/40038339477) )。如果你传递一个临时变量，而不是一个命名的变量；之后C++编译器就会将其解析为一个函数声明，而不是一个对象的定义。

例如：
```c++
std::thread my_thread(background_task());
```

这里相当与声明了一个名为**my_thread**，带有一个参数（是一个函数指针，指向一个没有参数并返回**background_task**对象的函数），并且返回一个`std::thread`对象的函数，而非启动了一个线程。你使用在前面命名函数对象的方式，或使用多组括号①，再或使用新统一的初始化语法②，来避免这个问题。举例如下：

```c++
std::thread my_thread((background_task()));  // 1
std::thread my_thread{background_task()};    // 2
```

使用lambda表达式也能避免这个问题。lambda表达式是C++11的一个性特性，
它允许使用一个可以捕获局部变量的局部函数（可以避免传递参数，参见2.2节）。想要具体的了解lambda表达式，可以阅读附录A的A.5节。之前的例子可以改写为lambda表达式的类型，如下：

```c++
std::thread my_thread([](
  do_something();
  do_something_else();
});
```

当你启动了线程，你需要明确一下自己的决定，是要等待线程结束（*加入*式——参见2.1.2节），还是让其自主运行（*分离*式——参见2.1.3节）。如果你
在`std::thread`对象销毁之前没有做出决定，那么你的程序就会终止（` std::thread`的析构函数会调用`std::terminate()`）。因此，即便是有异常的存在，你也需要确保线程能够正确的加入（*joined*）或分离（*detached*）。在2.1.3节中，会有对应的方法来处理这种情况。需要注意的是，你必须在`std::thread`对象销毁之前做出决定——可能在你加入或分离线程之前，线程就已经结束了，之后如果再去分离它，线程可能会再`std::thread`对象销毁之后继续运行下去。
如果不等待线程结束，你必须保证线程结束之前，可访问数据的有效性。这不是一个新问题——在单线程代码中，在对象销毁之后再去访问，也是未定义的行为——但是，线程的生命周期却增加了这个问题发生的几率。
这种情况很可能会在这种情况下碰到：当线程还没结束，函数已经退出时，线程函数还持有函数局部变量的指针或引用的时候。下面的清单中就展示了这样的一种情况。

清单2.1  函数已经结束，但线程依旧访问局部变量
```c++
struct func
{
  int& i;
  func(int& i_) : i(i_) {}
  void operator() ()
  {
    for (unsigned j=0 ; j<1000000 ; ++j)
    {
      do_something(i);           // 1. 潜在访问隐患：悬空引用
    }
  }
};

void oops()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread my_thread(my_func);
  my_thread.detach();          // 2. 不等待线程结束
}                              // 3. 新线程可能还在运行
```

在这个例子中，因为你已经明确的决定不等待线程结束了（使用了detach() ②），所以当**oops()**函数执行完成时 ③，与新线程中的函数可能还在运行中。如果线程还在运行中，它就会去调用**do_something(i)**函数 ①，这时就会去访问已经销毁了的变量。这挺像一个单线程程序——允许在函数完成后继续持有局部变量的指针或引用，这从来就不是一个好主意——因为这种情况发生的时，并不是很明显，所以这样的代码会使得多线程程序更容易出现错误。
处理这种情况常规方法是使线程函数自身的功能齐全，数据复制到线程中，而不是复制到共享数据中。如果你使用一个可调用的对象作为线程函数，这个对象就会赋值到线程中去，然后原始对象就能立即销毁了。但是，你还需要谨慎的对待对象中包含的指针和或引用，例如清单2.1所示。使用一个能访问局部变量的函数去创建一个线程是一个糟糕的主意，除非能够确定线程会在函数完成前结束。
此外，你可以通过加入的方式来确保线程在函数完成前结束。

###2.1.2 等待线程完成

如果需要等待线程结束，你对相关的`std::tread`实例使用**join()**。在2.1清单中，将`my_thread.detach()`替换为`my_thread.join()`，就可以确保局部变量在线程完成后，才被销毁。在这种情况下，因为原始线程在其生命周期中并没有做什么事，所以使得用一个独立的线程去运行这个函数变得收益甚微，但在实际编程中，原始线程要么有自己的工作要做；要么会启动多个子线程来做一些有用的工作，并等待这些线程结束。
**join()**是简单粗暴的——等待线程完成或不等待。如有你需要对等待中的线程有更细致的控制，比如看一下某个线程是否结束，或者只等待一段时间（超过时间就判定为**超时**）。想要做到这些，你需要使用一些其他的机制，比如条件变量和期待（*futures*），相关的讨论我们会在第四章继续。调用**join()**的行为，还清理了线程相关的存储部分，这样`std::thread`对象将不再与已经完成的线程有任何关联。这就意味着，你只能对一个线程使用一次**join()**;一旦已经使用过**join()**，`std::thread`对象就不能再次加入了，当对其使用**joinable()**时，函数将返回**否**（*false*）。

###2.1.3 特殊情况下的等待

如前面所提到的，你需要对一个还未销毁的`std::thread`对象使用**join()**或**detach()**。如果你想要分离一个线程，你可以在线程启动后，直接使用**detach()**进行分离。如果你打算等待对应线程，你需要细心挑选调用**join()**的地方。如果有异常在线程运行之后，join()调用之前抛出，就意味着很这次调用会被跳过。
为了避免你的应用被抛出的异常所终止，你需要作出一个决定。通常，当你倾向于在无异常的情况下使用**join()**时，你需要在异常处理过程中调用**join()**，从而避免生命周期的问题。我在下面的程序清单中做了一个例子。

清单 2.2 等待线程完成
```c++
struct func; // 定义在清单2.1中
void f()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread t(my_func);
  try
  {
    do_something_in_current_thread();
  }
  catch(...)
  {
    t.join();  // 1
    throw;
  }
  t.join();  // 2
}
```

清单2.2中的代码使用了try/catch块，确保访问本地状态的线程退出后，函数才结束。当函数正常退出时，会执行到②处；当函数执行过程中抛出异常，程序会执行到①处。冗余的try/catch块能轻易的捕获轻量级错误，所以这种情况，并非是理想的。如需确保线程在函数之前结束——查看是否因为线程函数使用了局部变量的引用，或其他原因——而后再确定一下程序可能会退出的途径，无论正常与否，然后提供一个简洁的机制，来做这些事。
一种方式是使用“资源获取即是初始化方式”（RAII，Resource Acquisition Is Initialization），并且提供一个类，在析构函数中使用**join()**，如同下面清单中的代码。看它如何简化**f()**函数。

清单 2.3 使用RAII等待线程完成
```c++
class thread_guard
{
  std::thread& t;
public:
  explicit thread_guard(std::thread& t_):
    t(t_)
  {}
  ~thread_guard()
  {
    if(t.joinable()) // 1
    {
      t.join();      // 2
    }
  }
  thread_guard(thread_guard const&)=delete;   // 3
  thread_guard& operator=(thread_guard const&)=delete;
};

struct func; // 定义在清单2.1中

void f()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread t(my_func);
  thread_guard g(t);
  do_something_in_current_thread();
}    // 4
```

当线程执行到 ④ 处时，局部对象就要被逆序销毁了。因此，**thread_guard**对象**g**是第一个被销毁的，这时线程在析构函数中被加入②到原始线程中。即使**do_something_in_current_thread**抛出一个异常，这个销毁的动作，依旧会发生。
在**thread_guard**的析构函数的测试中，首先判断线程是否已加入①，如果没有会调用**join()**②进行加入。这很重要，因为**join()**只能对给定的对象调用一次，所以对给已加入的线程再次进行加入操作时，将会导致一些错误。
拷贝构造函数和拷贝赋值操作被标记为`=delete`③，这是为了不让编译器自动生成它们。直接对一个对象进行拷贝或赋值是危险的，因为这可能会丢失已经加入了的线程。通过删除声明，任何尝试赋值一个**thread_guard**对象的操作都会引发一个编译错误。想要了解删除函数的更多知识，请参阅附录A的A.2节。
如果你不想等待线程结束，你可以分离（*detaching*）线程，从而避免异常安全（*exception-safety*）的问题。不过，这就打破了线程与`std::thread`对象的联系，即使线程仍然在后台运行着，分离操作也能确保`std::terminate()`不会在`std::thread`对象销毁后被调用，。

###2.1.4 后台运行线程

对一个`std::thread`对象使用**detach()**就会将这个线程搁置在后台运行，这就意味着不能与这个线程产生直接交互。通也就师说，我们不会等待这个线程结束;如果一个线程被分离，那么就不可能让一个`std::thread`对象引用它，所以这个线程就不可能再被加入了。分离的线程的确是运行在后台；因为C++运行库可以保证，当线程退出时，其相关资源的能够正确的回收，所以后台线程的归属和控制都会交给C++运行库处理。
通常，称分离线程为守护线程（*daemon threads*）。在UNIX中守护线程是指，运行在后台，且没有任何可用用户接口的线程。这种线程的特点就是长时间运行；它们的生命周期可能会从某一个应用起始到结束，也会在后台执行诸如监事文件系统的任务，还有可能对未使用的缓存进行清理，亦或对数据结构进行优化。另一方面，它也使用分离线程的另一种机制，确定线程什么时候结束，或者在“发后即忘”（*fire and forget*）的任务在哪里使用到了这个线程。
如你在2.1.2节所见，你可以调用`std::thread`的成员函数**detach()**来分离一个线程。在这之后，对应的`std::thread`对象就与这个实际执行的线程无关了，并且这个线程也无法再加入了：

```c++
std::thread t(do_background_work);
t.detach();
assert(!t.joinable());
```

为了从一个`std::thread`对象中分离一个线程，前提是有一个可进行分离的线程：你不能对没有可执行线程的`std::thread`对象使用**detach()**。这同样也是**join()**的使用条件，并且你需要用同样的方式进行检查——当一个`std::thread`对象使用**t.joinable()**返回的是true，你就可以对其使用**t.detach()**了。

试想如何能让一个文字处理应用同时编辑多个文档。无论是使用用户界面，还是在内部应用内部进行，都有很多的解决方法。虽然，这些窗口看起来是完全独立的，在每一个窗口都有自己独立的菜单选项，但是他们却是运行在同一个应用实例中。一种内部处理方式是，让每个文档处理窗口拥有自己的线程；每个线程运行同样的的代码，不过不同窗口处理的数据需要隔离开来。这样的话，打开一个文档就要启动一个新线程。因为是对独立的文档进行操作，所以没有必要等待其他线程完成。因此，这里就可以让文档处理窗口运行在分离的线程上。
下面代码简要的展示了这种方法：

清单2.4 使用分离线程去处理其他文档
```c++
void edit_document(std::string const& filename)
{
  open_document_and_display_gui(filename);
  while(!done_editing())
  {
    user_command cmd=get_user_input();
    if(cmd.type==open_new_document)
    {
      std::string const new_name=get_filename_from_user();
      std::thread t(edit_document,new_name);  // 1
      t.detach();  // 2
    }
    else
    {
       process_user_input(cmd);
    }
  }
}
```
如果用户选择打开一个新文档，为了让他们的文档迅速打开，你需要启动一个新线程去打开新文档 ① ，并且分离这个线程 ②。和当前线程做出的操作一样，新线程只不过是针对另一个文件而已。所以，你可以复用这个函数（**edit_document**），通过传参的形式打开新的文件。
这个例子也展示了传参启动线程的方法：你不仅可以向`std::thread`构造函数 ① 传递函数名，而且还可以传递函数所需的参数（实参）。虽然也有其他方法可以完成这项功能，比如使用一个带有成员数据的成员函数，去代替一个需要传参的普通函数。不过，在这里C++线程库的解决方式也很简单。

##2.2 向线程函数传递参数

如清单2.4所示，向`std::thread`构造函数中的可调用对象或函数传递一个参数是很简单的。但，这里需要注意的是，默认参数是要拷贝到，线程独立内存中的，一遍新创建的线程在执行中可以访问，即使参数参数是引用的形式。来看一个例子：

```c++
void f(int i,std::string const& s);
std::thread t(f, 3, "hello");
```

以上代码创建了一个调用**f(3, "hello")**的执行线程。注意，这里函数**f**需要一个`std::string`对象作为第二个参数，不过这使用的是字符串字面值，也就是`char const *`类型。之后，字面值向`std::string`对象的转化，是在新线程的上下文中去完成的。这里特别要注意的是，如下代码中，提供一个指向动态变量的指针作为参数传递给线程时。

```c++
void f(int i,std::string const& s);
void oops(int some_param)
{
  char buffer[1024]; // 1
  sprintf(buffer, "%i",some_param);
  std::thread t(f,3,buffer); // 2
  t.detach();
}
```

在这种情况下，buffer②是一个指针变量，并且指向本地变量  ，然后这个本地变量通过buffer传递到新线程 ② 中。并且，函数有很大的可能，会再字面值转化成`std::string`对象之前崩溃（*oops*），从而导致线程做出一些为未定义的行为。解决方案就是在传递到`std::tread`构造函数之前就将字面值转化为`std::string`对象。

```c++
void f(int i,std::string const& s);
void not_oops(int some_param)
{
  char buffer[1024];
  sprintf(buffer,"%i",some_param);
  std::thread t(f,3,std::string(buffer));  // 使用std::string，避免悬垂指针
  t.detach();
}
```

在这种情况中的问题是，你想要依赖与隐式转换将字面值转换为函数期待的`std::string`对象，但因为`std::thread`的构造函数会复制已提供的变量，这里就只复制了没有转换成期望类型的字符串字面值。
不过，也可能会有成功的情况：复制一个引用。成功的传递一个引用，可能发生在线程更新数据结构时。

```c++
void update_data_for_widget(widget_id w,widget_data& data); // 1
void oops_again(widget_id w)
{
  widget_data data;
  std::thread t(update_data_for_widget,w,data); // 2
  display_status();
  t.join();
  process_widget_data(data); // 3
}
```

虽然**update_data_for_widget** ① 的第二个参数期待传入一个引用，但是`std::thread`的构造函数 ② 不知道这事；构造函数无视函数期待的参数类型，并且盲目的拷贝已提供的变量。当线程调用**update_data_for_widget**函数时，传递给函数的参数是data变量内部拷贝的引用，而非数据本身的引用。因此，当线程结束时，内部拷贝数据将会在更新阶段被销毁，并且**process_widget_data**将会接收到没有修改的data变量 ③。使用你所熟悉的`std::bind`，这些问题的解决办法就显而易见了，你可以使用`std::ref`将需要参数转换成引用的形式。这种情况下，如果你改变将线程的调用改成以下形式：

```c++
std::thread t(update_data_for_widget,w,std::ref(data));
```

在这之后，**update_data_for_widget**就会接受到一个data变量的引用，而非一个data变量拷贝的引用。
如果你熟悉`std::bind`,就应该不会对以上传参的形式感到好奇。因为，`std::thread`构造函数和`std::bind`的操作都在标准库中定义好了。这就意味着，你可以传递一个成员函数指针作为线程函数，并提供一个合适的对象指针作为第一个参数：

```c++
class X
{
public:
  void do_lengthy_work();
};
X my_x;
std::thread t(&X::do_lengthy_work,&my_x); // 1
```

这段代码中，新线程将**my_x.do_lengthy_work()**函数作为线程函数；这里，my_x的地址 ① 作为一个指针对象提供给函数。你也可以为成员函数提供参数：`std::thread`构造函数的第三个参数就是成员函数的第一个参数，以此类推（代码如下，译者自加）。

```c++
class X
{
public:
  void do_lengthy_work(int);
};
X my_x;
int num;
std::thread t(&X::do_lengthy_work, &my_x, num);
```

另一个有趣的地方是，提供的参数可以是“移动”（*move*）的，但不能是“拷贝”（*copy*）的。这里的“移动”是指，原始对象中的数据转移给另一对象，而转移的这些数据就不再在原始对象中保存了（译者：比较像在文本编辑时的“剪切”操作）。`std::unique_ptr`就是这样一种类型（译者：C++11添加的一种指针指针），这种类型为动态分配的对象提供内存自动管理机制（译者：类似垃圾回收）。在同一时间点，只允许一个`std::unique_ptr`实现指向一个给定对象，并且当这个实现销毁时，指向的对象也将被删除。移动构造函数（*move constructor*）和移动赋值操作符（*move assignment operator*）允许一个对象在多个`std::unique_ptr`实现中传递（有关“移动”的更多内容，请参考附录A的A.1.1节）。使用移动转移源对象后，就会留下一个空指针（*NULL*）。移动操作可以将对象转换成可接受的类型，例如函数参数，或者函数返回的类型。当源对象是一个临时变量时，移动操作是自动的，但当源对象是一个命名变量，那么转移的时候就需要使用`std::move()`进行移动了。下面的代码展示了`std::move`的用法，它是如何转移一个动态对象到一个线程中去的：

```c++
void process_big_object(std::unique_ptr<big_object>);

std::unique_ptr<big_object> p(new big_object);
p->prepare_data(42);
std::thread t(process_big_object,std::move(p));
```

在`std::thread`的构造函数中指定`std::move(p)`,big_object对象的所有权就被首先转移到新创建线程的的内部存储中，之后传递给**process_big_object**函数。
在标准线程库中，有和`std::unique_ptr`和`std::thread`在所属权上有相似语义的类型。虽然，`std::thread`实例不会如`std::unique_ptr`那样去占有一个动态对象所有权，但是它会去占用一部分资源的所有权：每个实例都有管理一个执行线程的责任。在`std::thread`中，所有权可以在多个实例中互相转移，这是因为这些实例都是可移动（*movable*）的，并且都是不可复制（*aren't copyable*）的。这就保证了，在同一时间中，只有一个相关的执行线程；同时，也允许程序员在不同的对象之间转移所有权。

##2.3 转移线程所有权

假设你要写一个用来在后台启动线程线程的函数，但现在你想通过新线程返回的所有权（译者：大概就是想依赖线程中的得到的某些结果，对函数进行调用）去掉用这个函数，而不是等待这个等待线程结束再去调用；或完全与之相反的想法：创建一个线程，并且想要在一些函数中传递所有权，都必须要等待线程结束才行。不管怎么样，想要完成你的想法，新线程的所有权都需要从一个地方转移到另一个地方。
这就是移动引入`std::thread`的原因。之前说过，在C++标准库中有很多资源占用（*resource-owning*）类型，比如`std::ifstream`,`std::unique_ptr`还有`std::thread`，都是可移动（*movable*）的，而不可拷贝（*cpoyable*）的。这就说明执行线程的所有权是可以在`std::thread`实例中移动的，下面将展示一个例子。在这个例子中，创建了两个执行线程，并且在`std::thread`实例之间（t1,t2和t3）转移所有权：

```c++
void some_function();
void some_other_function();
std::thread t1(some_function);			// 1
std::thread t2=std::move(t1);			// 2
t1=std::thread(some_other_function);	// 3
std::thread t3;							// 4
t3=std::move(t2);						// 5
t1=std::move(t3);						// 6 赋值操作将使程序崩溃
```

首先，一个与t1相关的新线程启动 ① 。当显式使用` std::move()`创建t2后 ② ，t1的说有权就转移给了t2。现在，t1和执行线程已经没有关联了；执行**some_function**的函数现在与t2关联。
然后，与一个临时`std::thread`对象相关的线程启动了 ③ 。这里为什么不显式调用`std::move()`完成所有权的转移呢？因为这里的所有者是一个临时对象——移动操作将会自动的，且隐式的完成。
t3使用默认构造方式创建 ④，也就是没有与任何执行线程关联。当再次调用`std::move()`将与t2关联线程的所有权转移到t3中 ⑤。这里显式的调用了`std::move()`，是因为t2是一个命名对象（与临时对象相反）。在代码中的移动操作完成之后，t1与执行**some_other_function**的线程相关联，t2与任何线程都无关联，t3与执行**some_function**的线程相关联。
代码中，最后一个移动操作，将执行**some_function**线程的所有权转移 ⑥ 给t1。但这时，t1已经有了一个关联的线程（执行**some_other_function**的线程），所以这里会直接调用`std::terminate()`终止程序继续运行。终止操作将调用` std::thread`的析构函数，销毁所有对象（与C++中异常的处理方式很相似）。在2.1.1节时，你需要在线程对象被析构前，显式的等待一个线程完成，或者分离它；在进行复制时也需要满足这些条件（说明：你不能通过赋一个新值给`std::thread`对象的方式来“丢弃”一个线程）。
`std::thread`支持移动，就意味着线程的所有权，可以在函数外很容易的进行转移，就如下面程序清单中的程序一样。

清单2.5 从一个函数中返回一个`std::thread`对象
```c++
std::thread f()
{
  void some_function();
  return std::thread(some_function);
}
std::thread g()
{
  void some_other_function(int);
  std::thread t(some_other_function,42);
  return t;
}
```

同样的，如果所有权可以在函数内部传递，那么这就允许把`std::thread`实例当做参数进行传递，代码如下：

```c++
void f(std::thread t);
void g()
{
  void some_function();
  f(std::thread(some_function));
  std::thread t(some_function);
  f(std::move(t));
}
```

`std::thread`支持移动的好处之一是，你可以创建**thread_guard**类的实例（定义见 清单2.3），并且拥有其线程的所有权。当**thread_guard**对象所持有的线程已经被引用时，移动操作就可以避免很多不爽的情况的发生，并且这也意味着，当某个对象转移了线程的所有权后，它就不能对其进行加入或分离的操作了。这也主要是为了确保线程在范围退出前完成，所以我在下面的代码里定义了*scoped_thread*类。现在，我们来看一下这段代码：

清单2.6 scoped_thread的用法
```c++
class scoped_thread
{
  std::thread t;
public:
  explicit scoped_thread(std::thread t_): 				// 1
    t(std::move(t_))
  {
    if(!t.joinable()) 									// 2
      throw std::logic_error(“No thread”);
  }
  ~scoped_thread()
  {
    t.join();											// 3
  }
  scoped_thread(scoped_thread const&)=delete;
  scoped_thread& operator=(scoped_thread const&)=delete;
};
 
struct func; // 定义在清单2.1中

void f()
{
  int some_local_state;
  scoped_thread t(std::thread(func(some_local_state)));	// 4
  do_something_in_current_thread();
}														// 5
```

这个例子与清单2.3中的很相似，但这里的新线程是直接传递到scoped_thread中去的 ④，而不是创建了一个独立的命名变量。当主线程到达**f()**函数的末尾，**scoped_thread**对象将会销毁，然后加入 ③ 到的构造函数 ① 创建的线程对象中去。而在清单2.3中的**thread_guard**类，就要在析构的时候检查线程是否是“可加入”的。这里，我们把这个检查放在了构造函数中 ② ，并且当线程不可加入时，抛出一个异常。
对`std::thread`对象的容器，如果这个容器是移动敏感的（比如，新标准中的`std::vector<>`），那么移动操作同样适用于这些容器。了解这个后，你就可以写出类似一下清单中的代码了，这部分代码量产了一些线程，并且等待他们结束。

清单2.7 量产一些线程，并且等待它们结束
```c++
void do_work(unsigned id);

void f()
{
  std::vector<std::thread> threads;
  for(unsigned i=0;i<20;++i)
  {
    threads.push_back(std::thread(do_work,i)); // 产生线程
  } 
  std::for_each(threads.begin(),threads.end(),
  				std::mem_fn(&std::thread::join)); // 对每个线程调用join()
}
```




