# Darknet源码阅读
Darknet是一个较为轻型的完全基于C与CUDA的开源深度学习框架，其主要特点就是容易安装，没有任何依赖项（OpenCV都可以不用），移植性非常好，支持CPU与GPU两种计算方式。

更多信息（包括安装、使用）可以参考：[Darknet: Open Source Neural Networks in C](https://pjreddie.com/darknet/)

# 为什么要做这个？

我在阅读Yolo论文的时候对很多实现细节不太明白，但是所幸我找到了讲解Yolo源码的一些博客，工程，给了我很大的启发，所以我打算在他们的基础上从宏观和微观的角度来理解一下DarkNet并将笔记记录到这个工程里。

# DarkNet源码阅读之主线

## 关于分析主线的确定

darknet相比当前训练的c/c++主流框架来讲，具有编译速度快，依赖少，易部署等众多优点，我们先定位到examples/darknet.c的main函数，这是这个框架实现分类，定位，回归，分割等功能的初始入口。

![](image/1.png)

## 目标检测-run_detector

我们这里主要来分析一下目标检测，也就是examples/detector.c中的run_detector函数。可以看到这个函数主要包括了训练-测试-验证三个阶段。

![](image/2.png)

## 训练检测器-train_detector

由于训练，验证和测试阶段代码几乎是差不多的，只不过训练多了一个反向传播的过程。所以我们主要分析一下训练过程，训练过程是一个比较复杂的过程，不过宏观上大致分为解析网络配置文件，加载训练样本图像和labels，开启训练，结束训练保存模型这样一个过程，整体代码如下：

![](image/3.png)

## 解析配置文件(.cfg)

### 配置文件长啥样？

![](image/4.png)

可以看到配置参数大概分为2类：

- 与训练相关的项，以 [net] 行开头的段. 其中包含的参数有: `batch_size, width,height,channel,momentum,decay,angle,saturation, exposure,hue,learning_rate,burn_in,max_batches,policy,steps,scales`。
- 不同类型的层的配置参数. 如` [convolutional], [short_cut], [yolo], [route], [upsample] `层等。

在src/parse.c中我们会看到一行代码，`net->batch /= net->subdivisions;`，也就是说`batch_size` 在 darknet 内部又被均分为 `net->subdivisions`份, 成为更小的`batch_size`。 但是这些小的 `batch_size` 最终又被汇总, 因此 darknet 中的`batch_size = net->batch / net->subdivisions * net->subdivisions`。此外，和这个参数相关的计算训练图片数目的时候是这样，`int imgs = net->batch * net->subdivisions * ngpus;`，这样可以保证`imgs`可以被`subdivisions`整除，因此，通常将这个参数设为8的倍数。从这里也可以看出每个gpu或者cpu都会训练`batch`个样本。

我们知道了参数是什么样子，那么darknet是如何保存这些参数的呢？这就要看下基本数据结构了。

### 基本数据结构？

Darknet是一个C语言实现的神经网络框架，这就决定了其中大多数保存数据的数据结构都会使用链表这种简单高效的数据结构。为了解析网络配置参数, darknet 中定义了三个关键的数据结构类型。list 类型变量保存所有的网络参数, section类型变量保存的是网络中每一层的网络类型和参数, 其中的参数又是使用list类型来表示.  kvp键值对类型用来保存解析后的参数变量和参数值。

- list类型定义在include/darknet.h文件中，代码如下：

  ```c++
  //链表上的节点
  typedef struct node{
      void *val;
      struct node *next;
      struct node *prev;
  } node;
  
  //双向链表
  typedef struct list{
      int size; //list的所有节点个数
      node *front; //list的首节点
      node *back; //list的普通节点
  } list;
  
  ```

- section 类型定义在src/parser.c文件中，代码如下：

  ```c++
  typedef struct{
      char *type;
      list *options;
  }section;
  ```

- kvp 键值对类型定义在src/option_list.h文件中，具体定义如下：

  ```c++
  typedef struct{
      char *key;
      char *val;
      int used;
  } kvp;
  ```

在darknet的网络配置文件(.cfg结尾)中，以`[`开头的行被称为一个段(section)。所有的网络配置参数保存在list类型变量中，list中有很多的sections节点，每个sections中又有一个保存层参数的小list，整体上出现了一种大链挂小链的结构。大链的每个节点为section，每个section中包含的参数保存在小链中，小链的节点值的数据类型为kvp键值对，这里有个图片可以解释这种结构。

![](image/5.jpg)

我们来大概解释下该参数网，首先创建一个list，取名sections，记录一共有多少个section（一个section存储了CNN一层所需参数）；然后创建一个node，该node的void类型的指针指向一个新创建的section；该section的char类型指针指向.cfg文件中的某一行（line），然后将该section的list指针指向一个新创建的node，该node的void指针指向一个kvp结构体，kvp结构体中的key就是.cfg文件中的关键字（如：`batch，subdivisions`等），val就是对应的值；如此循环就形成了上述的参数网络图。

### 解析并保存网络参数到链表中

读取配置文件由src/parser.c中的`read_cfg()`函数实现：

```c++
/*
 * 读取神经网络结构配置文件（.cfg文件）中的配置数据， 将每个神经网络层参数读取到每个
 * section 结构体 (每个 section 是 sections 的一个节点) 中， 而后全部插入到
 * list 结构体 sections 中并返回
 * 
 * \param: filename    C 风格字符数组， 神经网络结构配置文件路径
 * 
 * \return: list 结构体指针，包含从神经网络结构配置文件中读入的所有神经网络层的参数
 * 每个 section 的所在行的开头是 ‘[’ , ‘\0’ , ‘#’ 和 ‘;’ 符号开头的行为无效行, 除此
 *之外的行为 section 对应的参数行. 每一行都是一个等式, 类似键值对的形式.

 *可以看到, 如果某一行开头是符号 ‘[’ , 说明读到了一个新的 section: current, 然后第945行
 *list_insert(options, current);` 将该新的 section 保存起来.

 *在读取到下一个开头符号为 ‘[’ 的行之前的所有行都是该 section 的参数, 在第 957 行 
 *read_option(line, current->options) 将读取到的参数保存在 current 变量的 options 中. 
 *注意, 这里保存在 options 节点中的数据为 kvp 键值对类型.

 *当然对于 kvp 类型的参数, 需要先将每一行中对应的键和值(用 ‘=’ 分割) 分离出来, 然后再
 *构造一个 kvp 类型的变量作为节点元素的数据.
 */

list *read_cfg(char *filename)
{
    FILE *file = fopen(filename, "r");
    //一个section表示配置文件中的一个字段，也就是网络结构中的一层
    //因此，一个section将读取并存储某一层的参数以及该层的type
    if(file == 0) file_error(filename);
    char *line;
    int nu = 0; //当前读取行号
    list *options = make_list(); //options包含所有的神经网络层参数
    section *current = 0; //当前读取到某一层
    while((line=fgetl(file)) != 0){ 
        ++ nu;
        strip(line); //去除读入行中含有的空格符
        switch(line[0]){
            // 以 '[' 开头的行是一个新的 section , 其内容是层的 type 
            // 比如 [net], [maxpool], [convolutional] ...
            case '[':
                current = malloc(sizeof(section));
                list_insert(options, current);
                current->options = make_list();
                current->type = line;
                break;
            case '\0': //空行
            case '#':  //注释
            case ';': //空行
                free(line); // 对于上述三种情况直接释放内存即可
                break;
            default:
                // 剩下的才真正是网络结构的数据，调用 read_option() 函数读取
                // 返回 0 说明文件中的数据格式有问题，将会提示错误
                if(!read_option(line, current->options)){
                    fprintf(stderr, "Config file error line %d, could parse: %s\n", nu, line);
                    free(line);
                }
                break;
        }
    }
    fclose(file);
    return options;
}
```

### 链表的插入操作

保存 section 和每个参数组成的键值对时使用的是 list_insert() 函数, 前面提到了参数保存的结构其实是大链( 节点为 section )上边挂着很多小链( 每个 section 节点的各个参数)。`list_insert()`函数实现了链表插入操作，该函数定义在src/list.c 文件中：

```c++
/*
 * \brief: 将 val 指针插入 list 结构体 l 中，这里相当于是用 C 实现了 C++ 中的 
 *         list 的元素插入功能
 * 
 * \prama: l    链表指针
 *         val  链表节点的元素值
 * 
 * 流程： list 中保存的是 node 指针. 因此，需要用 node 结构体将 val 包裹起来后才可以
 *       插入 list 指针 l 中
 * 
 * 注意: 此函数类似 C++ 的 insert() 插入方式；
 *      而 opion_insert() 函数类似 C++ map 的按值插入方式，比如 map[key]= value
 *      
 *      两个函数操作对象都是 list 变量， 只是操作方式略有不同。
*/
void list_insert(list *l, void *val)
{
	node *new = malloc(sizeof(node));
	new->val = val;
	new->next = 0;
    // 如果 list 的 back 成员为空(初始化为 0), 说明 l 到目前为止，还没有存入数据  
    // 另外, 令 l 的 front 为 new （此后 front 将不会再变，除非删除） 
	if(!l->back){
		l->front = new;
		new->prev = 0;
	}else{
		l->back->next = new;
		new->prev = l->back;
	}
	l->back = new;
	++l->size;
}
```

可以看到, 插入的数据都会被重新包装在一个新的 node : 变量 new 中, 然后再将这个节点插入到链表中。网络结构解析到链表中后还不能直接使用, 如果仅仅想使用某一个参数而不得不每次都遍历整个链表, 这样就会导致程序效率变低, 最好的办法是将其保存到一个结构体变量中, 使用的时候按照成员进行访问。

### 将链表中的网络结构保存到network结构体

- 首先来看看network结构体的定义，在include/darknet.h中：

  ```c++
  //定义network结构
  typedef struct network{
      int n; //网络的层数，调用make_network(int n)时赋值
      int batch; //一批训练中的图片参数，和subdivsions参数相关
      size_t *seen; //目前已经读入的图片张数(网络已经处理的图片张数) 
      int *t;
      float epoch; //到目前为止训练了整个数据集的次数
      int subdivisions;
      layer *layers;  //存储网络中的所有层  
      float *output;
      learning_rate_policy policy; // 学习率下降策略: TODO
      // 梯度下降法相关参数  
      float learning_rate;
      float momentum;
      float decay;
      float gamma;
      float scale;
      float power;
      int time_steps;
      int step;
      int max_batches;
      float *scales;
      int   *steps;
      int num_steps;
      int burn_in;
  
      int adam;
      float B1;
      float B2;
      float eps;
  
      int inputs;
      int outputs;
      int truths;
      int notruth;
      int h, w, c;
      int max_crop;
      int min_crop;
      float max_ratio;
      float min_ratio;
      int center;
      float angle;
      float aspect;
      float exposure;
      float saturation;
      float hue;
      int random;
      //darknet 为每个 GPU 维护一个相同的 network, 每个 network 以 gpu_index 区分
      int gpu_index;
      tree *hierarchy;
      //中间变量，用来暂存某层网络的输入（包含一个 batch 的输入，比如某层网络完成前向，
      //将其输出赋给该变量，作为下一层的输入，可以参看 network.c 中的forward_network() 
      //与 backward_network() 两个函数 ）
      float *input;
      // 中间变量，与上面的 input 对应，用来暂存 input 数据对应的标签数据（真实数据）
      float *truth;
      // 中间变量，用来暂存某层网络的敏感度图（反向传播处理当前层时，用来存储上一层的敏
      //感度图，因为当前层会计算部分上一层的敏感度图，可以参看 network.c 中的 backward_network() 函数） 
      float *delta;
      // 网络的工作空间, 指的是所有层中占用运算空间最大的那个层的 workspace_size, 
      // 因为实际上在 GPU 或 CPU 中某个时刻只有一个层在做前向或反向运算
      float *workspace;
       // 网络是否处于训练阶段的标志参数，如果是则值为1. 这个参数一般用于训练与测试阶段有不
      // 同操作的情况，比如 dropout 层，在训练阶段才需要进行 forward_dropout_layer()
      // 函数， 测试阶段则不需要进入到该函数
      int train;
      int index;// 标志参数，当前网络的活跃层 
      float *cost;
      float clip;
  
  #ifdef GPU
      float *input_gpu;
      float *truth_gpu;
      float *delta_gpu;
      float *output_gpu;
  #endif
  
  } network;
  ```

- 为网络结构体分配内存空间，函数定义在src/network.c文件中：

  ```c++
  //为网络结构体分配内存空间
  network *make_network(int n)
  {
      network *net = calloc(1, sizeof(network));
      net->n = n;
      net->layers = calloc(net->n, sizeof(layer));
      net->seen = calloc(1, sizeof(size_t));
      net->t    = calloc(1, sizeof(int));
      net->cost = calloc(1, sizeof(float));
      return net;
  }
  ```

  在src/parser.c中的`parse_network_cfg()`函数中，从net变量开始，依次为其中的指针变量分配内存。由于第一个段`[net]`中存放的是和网络并不直接相关的配置参数, 因此网络中层的数目为 sections->size - 1，即：`network *net = make_network(sections->size - 1);`

- 将链表中的网络参数解析后保存到 network 结构体，配置文件的第一个段一定是`[net]`段，该段的参数解析由`parse_net_options()`函数完成，函数定义在src/parser.c中。之后的各段都是网络中的层。比如完成特定特征提取的卷积层，用来降低训练误差的shortcur层和防止过拟合的dropout层等。这些层都有特定的解析函数：比如`parse_convolutional()`, `parse_shortcut()`和`parse_dropout()`。每个解析函数返回一个填充好的层l，将这些层全部添加到network结构体的layers数组中。即是：`net->layers[count] = l;`另外需要注意的是这行代码：`if (l.workspace_size > workspace_size) workspace_size = l.workspace_size;`，其中workspace代表网络的工作空间，指的是所有层中占用运算空间最大那个层的workspace。因为在CPU或GPU中某个时刻只有一个层在做前向或反向传播。 输出层只能在网络搭建完毕之后才可以确定，输入层需要考虑`batch_size`的因素，truth是输入标签，同样需要考虑`batch_size`的因素。

  ```c++
  	layer out = get_network_output_layer(net);
      net->outputs = out.outputs;
      net->truths = out.outputs;
      if(net->layers[net->n-1].truths) net->truths = net->layers[net->n-1].truths;
      net->output = out.output;
      net->input = calloc(net->inputs*net->batch, sizeof(float));
      net->truth = calloc(net->truths*net->batch, sizeof(float));
  ```

- 到这里，网络的宏观解析结束。`parse_network_cfg()`(src/parser.c中)函数返回解析好的network类型的指针变量。

### 为啥需要中间数据结构缓存？

这里可能有个疑问，为什么不将配置文件读取并解析到 network 结构体变量中, 而要使用一个中间数据结构来缓存读取到的文件呢？如果不使用中间数据结构来缓存. 将读取和解析流程串行进行的话, 如果配置文件较为复杂, 就会长时间使文件处于打开状态。 如果此时用户更改了配置文件中的一些条目, 就会导致读取和解析过程出现问题。分开两步进行可以先快速读取文件信息到内存中组织好的结构中, 这时就可以关闭文件. 然后再慢慢的解析参数。这种机制类似于操作系统中断的底半部机制, 先处理重要的中断信号, 然后在系统负荷较小时再处理中断信号中携带的任务。

## 加载训练样本数据

darknet的数据加载在src/data.c中实现，`load_data()`函数调用流程如下：`load_data(args)->load_threads()->load_data_in_threads()->load_thread()->load_data_detection()`，前四个函数都是在对线程的调用进行封装，主要是个线程的加载任务量。最底层的数据加载任务由 `load_data_detection()` 函数完成。所有的数据(图片数据和标注信息数据)加载完成之后再拼接到一个大的数组中。在darknet中，图片的存储形式是一个行向量，向量长度为`h*w*3`。同时图片被归一化到[0, 1]之间。

### load_threads()完成线程分配和数据拼接

```c++
// copy from https://github.com/hgpvision/darknet/blob/master/src/data.c#L355
/*
** 开辟多个线程读入图片数据，读入数据存储至ptr.d中（主要调用load_in_thread()函数完成）
** 输入： ptr    包含所有线程要读入图片数据的信息（读入多少张，开几个线程读入，读入图片最终的宽高，图片路径等等）
** 返回： void*  万能指针（实际上不需要返回什么）
** 说明： 1) load_threads()是一个指针函数，只是一个返回变量为void*的普通函数，不是函数指针
**       2) 输入ptr是一个void*指针（万能指针），使用时需要强转为具体类型的指针
**       3) 函数中涉及四个用来存储读入数据的变量：ptr, args, out, buffers，除args外都是data*类型，所有这些变量的
**          指针变量其实都指向同一块内存（当然函数中间有些动态变化），因此读入的数据都是互通的。
** 流程： 本函数首先会获取要读入图片的张数、要开启线程的个数，而后计算每个线程应该读入的图片张数（尽可能的均匀分配），
**       并创建所有的线程，并行读入数据，最后合并每个线程读入的数据至一个大data中，这个data的指针变量与ptr的指针变量
**       指向的是统一块内存，因此也就最终将数据读入到ptr.d中（所以其实没有返回值）
*/
void *load_threads(void *ptr)
{
    int i;
    // 先使用(load_args*)强转void*指针，而后取ptr所指内容赋值给args
    // 虽然args不是指针，args是深拷贝了ptr中的内容，但是要知道ptr（也就是load_args数据类型），有很多的
    // 指针变量，args深拷贝将拷贝这些指针变量到args中（这些指针变量本身对ptr来说就是内容，
    // 而args所指的值是args的内容，不是ptr的，不要混为一谈），因此，args与ptr将会共享所有指针变量所指的内容
    load_args args = *(load_args *)ptr;
    if (args.threads == 0) args.threads = 1;
    // 另指针变量out=args.d，使得out与args.d指向统一块内存，之后，args.d所指的内存块会变（反正也没什么用了，变就变吧），
    // 但out不会变，这样可以保证out与最原始的ptr指向同一块存储读入图片数据的内存块，因此最终将图片读到out中，
    // 实际就是读到了最原始的ptr中，比如train_detector()函数中定义的args.d中
    data *out = args.d;
    // 读入图片的总张数= batch * subdivision * ngpus，可参见train_detector()函数中的赋值
    int total = args.n;
    // 释放ptr：ptr是传入的指针变量，传入的指针变量本身也是按值传递的，即传入函数之后，指针变量得到复制，函数内的形参ptr
    // 获取外部实参的值之后，二者本身没有关系，但是由于是指针变量，二者之间又存在一丝关系，那就是函数内形参与函数外实参指向
    // 同一块内存。又由于函数外实参内存是动态分配的，因此函数内的形参可以使用free()函数进行内存释放，但一般不推荐这么做，因为函数内释放内存，
    // 会影响函数外实参的使用，可能使之成为野指针，那为什么这里可以用free()释放ptr呢，不会出现问题吗？
    // 其一，因为ptr是一个结构体，是一个包含众多的指针变量的结构体，如data* d等（当然还有其他非指针变量如int h等），
    // 直接free(ptr)将会导致函数外实参无法再访问非指针变量int h等（实际经过测试，在gcc编译器下，能访问但是值被重新初始化为0），
    // 因为函数内形参和函数外实参共享一块堆内存，而这些非指针变量都是存在这块堆内存上的，内存一释放，就无法访问了；
    // 但是对于指针变量，free(ptr)将无作为（这个结论也是经过测试的，也是用的gcc编译器），不会释放或者擦写掉ptr指针变量本身的值，
    // 当然也不会影响函数外实参，更不会牵扯到这些指针变量所指的内存块，总的来说，
    // free(ptr)将使得ptr不能再访问指针变量（如int h等，实际经过测试，在gcc编译器下，能访问但是值被重新初始化为0），
    // 但其指针变量本身没有受影响，依旧可以访问；对于函数外实参，同样不能访问非指针变量，而指针变量不受影响，依旧可以访问。
    // 其二，darknet数据读取的实现一层套一层（似乎有点罗嗦，总感觉代码可以不用这么写的:)），具体调用过程如下：
    // load_data(load_args args)->load_threads(load_args* ptr)->load_data_in_thread(load_args args)->load_thread(load_args* ptr)，
    // 就在load_data()中，重新定义了ptr，并为之动态分配了内存，且深拷贝了传给load_data()函数的值args，也就是说在此之后load_data()函数中的args除了其中的指针变量指着同一块堆内存之外，
    // 二者的非指针变量再无瓜葛，不管之后经过多少个函数，对ptr的非指针变量做了什么改动，比如这里直接free(ptr)，使得非指针变量值为0,都不会影响load_data()中的args的非指针变量，也就不会影响更为顶层函数中定义的args的非指针变量的值，
    // 比如train_detector()函数中的args，train_detector()对args非指针变量赋的值都不会受影响，保持不变。综其两点，此处直接free(ptr)是安全的。
    // 说明：free(ptr)函数，确定会做的事是使得内存块可以重新分配，且不会影响指针变量ptr本身的值，也就是ptr还是指向那块地址， 虽然可以使用，但很危险，因为这块内存实际是无效的，
    //      系统已经认为这块内存是可分配的，会毫不考虑的将这块内存分给其他变量，这样，其值随时都可能会被其他变量改变，这种情况下的ptr指针就是所谓的野指针（所以经常可以看到free之后，置原指针为NULL）。
    //      而至于free(ptr)还不会做其他事情，比如会不会重新初始化这块内存为0（擦写掉），以及怎么擦写，这些操作，是不确定的，可能跟具体的编译器有关（个人猜测），
    //      经过测试，对于gcc编译器，free(ptr)之后，ptr中的非指针变量的地址不变，但其值全部擦写为0；ptr中的指针变量，丝毫不受影响，指针变量本身没有被擦写，
    //      存储的地址还是指向先前分配的内存块，所以ptr能够正常访问其指针变量所指的值。测试代码为darknet_test_struct_memory_free.c。
    //      不知道这段测试代码在VS中执行会怎样，还没经过测试，也不知道换用其他编译器（darknet的Makefile文件中，指定了编译器为gcc），darknet的编译会不会有什么问题？？
    //      关于free()，可以看看：http://blog.sina.com.cn/s/blog_615ec1630102uwle.html，文章最后有一个很有意思的比喻，但意思好像就和我这里说的有点不一样了（到底是不是编译器搞得鬼呢？？）。
    free(ptr);
    // 每一个线程都会读入一个data，定义并分配args.thread个data的内存
    data *buffers = calloc(args.threads, sizeof(data));
    // 此处定义了多个线程，并为每个线程动态分配内存
    pthread_t *threads = calloc(args.threads, sizeof(pthread_t));
    for(i = 0; i < args.threads; ++i){
        // 此处就承应了上面的注释，args.d指针变量本身发生了改动，使得本函数的args.d与out不再指向同一块内存，
        // 改为指向buffers指向的某一段内存，因为下面的load_data_in_thread()函数统一了结口，需要输入一个load_args类型参数，
        // 实际是想把图片数据读入到buffers[i]中，只能令args.d与buffers[i]指向同一块内存
        args.d = buffers + i;
        // 下面这句很有意思，因为有多个线程，所有线程读入的总图片张数为total，需要将total均匀的分到各个线程上，
        // 但很可能会遇到total不能整除的args.threads的情况，比如total = 61, args.threads =8,显然不能做到
        // 完全均匀的分配，但又要保证读入图片的总张数一定等于total，用下面的语句刚好在尽量均匀的情况下，
        // 保证总和为total，比如61,那么8个线程各自读入的照片张数分别为：7, 8, 7, 8, 8, 7, 8, 8
        args.n = (i+1) * total/args.threads - i * total/args.threads;
        // 开启线程，读入数据到args.d中（也就读入到buffers[i]中）
        // load_data_in_thread()函数返回所开启的线程，并存储之前已经动态分配内存用来存储所有线程的threads中，
        // 方便下面使用pthread_join()函数控制相应线程
        threads[i] = load_data_in_thread(args);
    }

    for(i = 0; i < args.threads; ++i){
        // 以阻塞的方式等待线程threads[i]结束：阻塞是指阻塞启动该子线程的母线程（此处应为主线程），
        // 是母线程处于阻塞状态，一直等待所有子线程执行完（读完所有数据）才会继续执行下面的语句
        // 关于多线程的使用，进行过代码测试，测试代码对应：darknet_test_pthread_join.c
        pthread_join(threads[i], 0);
    }
    // 多个线程读入所有数据之后，分别存储到buffers[0],buffers[1]...中，接着使用concat_datas()函数将buffers中的数据全部合并成一个大数组得到out
    *out = concat_datas(buffers, args.threads);
    // 也就只有out的shallow敢置为0了，为什么呢？因为out是此次迭代读入的最终数据，该数据参与训练（用完）之后，当然可以深层释放了，而此前的都是中间变量，
    // 还处于读入数据阶段，万不可设置shallow=0
    out->shallow = 0;
    // 释放buffers，buffers也是个中间变量，切记shallow设置为1,如果设置为0,那就连out中的数据也没了
    for(i = 0; i < args.threads; ++i){
        buffers[i].shallow = 1;
        free_data(buffers[i]);
    }
    // 最终直接释放buffers,threads，注意buffers是一个存储data的一维数组，上面循环中的内存释放，实际是释放每一个data的部分内存
    // （这部分内存对data而言是非主要内存，不是存储读入数据的内存块，而是存储指向这些内存块的指针变量，可以释放的）
    free(buffers);
    free(threads);
    return 0;
}
```

### load_data_detection()完成底层的数据加载任务

```c++
/*
** 可以参考，看一下对图像进行jitter处理的各种效果:
** https://github.com/vxy10/ImageAugmentation
** 从所有训练图片中，随机读取n张，并对这n张图片进行数据增强，同时矫正增强后的数据标签信息。最终得到的图片的宽高为w,h（原始训练集中的图片尺寸不定），也就是网络能够处理的图片尺寸，
** 数据增强包括：对原始图片进行宽高方向上的插值缩放（两方向上缩放系数不一定相同），下面称之为缩放抖动；随机抠取或者平移图片（位置抖动）；
** 在hsv颜色空间增加噪声（颜色抖动）；左右水平翻转，不含旋转抖动。
** 输入： n         一个线程读入的图片张数（详见函数内部注释）
**       paths     所有训练图片所在路径集合，是一个二维数组，每一行对应一张图片的路径（将在其中随机取n个）
**       m         paths的行数，也即训练图片总数
**       w         网络能够处理的图的宽度（也就是输入图片经过一系列数据增强、变换之后最终输入到网络的图的宽度）
**       h         网络能够处理的图的高度（也就是输入图片经过一系列数据增强、变换之后最终输入到网络的图的高度）
**       boxes     每张训练图片最大处理的矩形框数（图片内可能含有更多的物体，即更多的矩形框，那么就在其中随机选择boxes个参与训练，具体执行在fill_truth_detection()函数中）
**       classes   类别总数，本函数并未用到（fill_truth_detection函数其实并没有用这个参数）
**       jitter    这个参数为缩放抖动系数，就是图片缩放抖动的剧烈程度，越大，允许的抖动范围越大（所谓缩放抖动，就是在宽高上插值缩放图片，宽高两方向上缩放的系数不一定相同）
**       hue       颜色（hsv颜色空间）数据增强参数：色调（取值0度到360度）偏差最大值，实际色调偏差为-hue~hue之间的随机值
**       saturation 颜色（hsv颜色空间）数据增强参数：色彩饱和度（取值范围0~1）缩放最大值，实际为范围内的随机值
**       exposure  颜色（hsv颜色空间）数据增强参数：明度（色彩明亮程度，0~1）缩放最大值，实际为范围内的随机值
** 返回： data类型数据，包含一个线程读入的所有图片数据（含有n张图片）
** 说明： 最后四个参数用于数据增强，主要对原图进行缩放抖动，位置抖动（平移）以及颜色抖动（颜色值增加一定噪声），抖动一定程度上可以理解成对图像增加噪声。
**       通过对原始图像进行抖动，实现数据增强。最后三个参数具体用法参考本函数内调用的random_distort_image()函数
** 说明2：从此函数可以看出，darknet对训练集中图片的尺寸没有要求，可以是任意尺寸的图片，因为经该函数处理（缩放/裁剪）之后，
**       不管是什么尺寸的照片，都会统一为网络训练使用的尺寸
*/
data load_data_detection(int n, char **paths, int m, int w, int h, int boxes, int classes, float jitter, float hue, float saturation, float exposure)
{
    // paths包含所有训练图片的路径，get_random_paths函数从中随机提出n条，即为此次读入的n张图片的路径
    char **random_paths = get_random_paths(paths, n, m);
    int i;
    // 初始化为0,清楚内存中之前的旧值
    data d = {0};
    d.shallow = 0;
    // 一次读入的图片张数：d.X中每行就是一张图片的数据，因此d.X.cols等于h*w*3
    // n = net.batch * net.subdivisions * ngpus，net中的subdivisions这个参数暂时还没搞懂有什么用，
    // 从parse_net_option()函数可知，net.batch = net.batch / net.subdivision，等号右边的那个batch就是
    // 网络配置文件.cfg中设置的每个batch的图片数量，但是不知道为什么多了subdivision这个参数？总之，
    // net.batch * net.subdivisions又得到了在网络配置文件中设定的batch值，然后乘以ngpus，是考虑多个GPU实现数据并行，
    // 一次读入多个batch的数据，分配到不同GPU上进行训练。在load_threads()函数中，又将整个的n仅可能均匀的划分到每个线程上，
    // 也就是总的读入图片张数为n = net.batch * net.subdivisions * ngpus，但这些图片不是一个线程读完的，而是分配到多个线程并行读入，
    // 因此本函数中的n实际不是总的n，而是分配到该线程上的n，比如总共要读入128张图片，共开启8个线程读数据，那么本函数中的n为16,而不是总数128
    d.X.rows = n;
    //d.X为一个matrix类型数据，其中d.X.vals是其具体数据，是指针的指针（即为二维数组），此处先为第一维动态分配内存
    d.X.vals = calloc(d.X.rows, sizeof(float*));
    d.X.cols = h*w*3;
    // d.y存储了所有读入照片的标签信息，每条标签包含5条信息：类别，以及矩形框的x,y,w,h
    // boxes为一张图片最多能够处理（参与训练）的矩形框的数（如果图片中的矩形框数多于这个数，那么随机挑选boxes个，这个参数仅在parse_region以及parse_detection中出现，好奇怪？    
    // 在其他网络解析函数中并没有出现）。同样，d.y是一个matrix，make_matrix会指定y的行数和列数，同时会为其第一维动态分配内存
    d.y = make_matrix(n, 5*boxes);
    // 依次读入每一张图片到d.X.vals的适当位置，同时读入对应的标签信息到d.y.vals的适当位置
    for(i = 0; i < n; ++i){
        //读入原始的图片
        image orig = load_image_color(random_paths[i], 0, 0);
        // 原始图片经过一系列处理（重排及变换）之后的最终得到的图片，并初始化像素值全为0.5（下面会称之为输出图或者最终图之类的）
        image sized = make_image(w, h, orig.c);
        fill_image(sized, .5);
        // 缩放抖动大小：缩放抖动系数乘以原始图宽高即得像素单位意义上的缩放抖动
        float dw = jitter * orig.w;
        float dh = jitter * orig.h;
        // 缩放抖动大小：缩放抖动系数乘以原始图宽高即得像素单位意义上的缩放抖动
        float new_ar = (orig.w + rand_uniform(-dw, dw)) / (orig.h + rand_uniform(-dh, dh));
        //float scale = rand_uniform(.25, 2);
        
        // 为了方便，引入了一个虚拟的中间图（之所以称为虚拟，是因为这个中间图并不是一个真实存在的变量），
        // 下面两个变量nh,nw其实是中间图的高宽，而scale就是中间图相对于输出图sized的缩放尺寸（比sized大或者小）
        // 中间图与sized 并不是保持长宽比等比例缩放，中间图的长宽比为new_ar，而sized的长宽比为w/h，
        // 二者之间的唯一的关系就是有一条边（宽或高）的长度比例为scale
        float scale = 1;
        //nw, nh为中间图的宽高，new_ar为中间图的宽高比
        float nw, nh;

        if(new_ar < 1){
            // new_ar<1，说明宽度小于高度，则以高度为主，宽度按高度的比例计算
            nh = scale * h;
            nw = nh * new_ar;
        } else {
            // 否则说明高度小于等于宽度，则以宽度为主，高度按宽度比例计算 
            nw = scale * w;
            nh = nw / new_ar;
        }
        // 得到0~w-nw之间的均匀随机数（w-nw可能大于0,可能小于0，因为scale可能大于1,也可能小于1）
        float dx = rand_uniform(0, w - nw);
        // 得到0~h-nh之间的均匀随机数（h-nh可能大于0,可能小于0）
        float dy = rand_uniform(0, h - nh);
        // place_image先将orig根据中间图的尺寸nw,nh进行重排（双线性插值，不是等比例缩放，长宽比可能会变），而后，将中间图放入到sized，
        // dx,dy是将中间图放入到sized的起始坐标位置（dx,dy若大于0,说明sized的尺寸大于中间图的尺寸，这时
        // 可以说是将中间图随机嵌入到sized中的某个位置；dx,dy若小于0,说明sized的尺寸小于中间图的尺寸，这时
        // sized相当于是中间图的一个mask，在中间图上随机抠图）
        place_image(orig, nw, nh, dx, dy, sized);
        // 随机对图像jitter（在hsv三个通道上添加扰动），实现数据增强
        random_distort_image(sized, hue, saturation, exposure);
        // 随机的决定是否进行左右翻转操作来实现数据增强（注意是直接对sized，不是对原始图，也不是中间图）
        int flip = rand()%2;
        if(flip) flip_image(sized);
        // d.X为图像数据，是一个矩阵（二维数组），每一行为一张图片的数据
        d.X.vals[i] = sized.data;

        // d.y包含所有图像的标签信息（包括真实类别与位置），d.y.vals是一个矩阵（二维数组），每一行含一张图片的标签信息
        // 因为对原始图片进行了数据增强，其中的平移抖动势必会改动每个物体的矩形框标签信息（主要是矩形框的像素坐标信息），需要根据具体的数据增强方式进行相应矫正
        // 后面4个参数就是用于数据增强后的矩形框信息矫正（nw,nh是中间图宽高，w,h是最终图宽高）
        fill_truth_detection(random_paths[i], boxes, d.y.vals[i], classes, flip, -dx/w, -dy/h, nw/w, nh/h);

        free_image(orig);
    }
    free(random_paths);
    return d;
}
```

### load_data(args)使用技巧

![](image/6.png)

可以看到在examples/detector.c中的`train_detector()`函数共有3次调用`load_data(args)`，第一次调用是为训练阶段做好数据准备工作，充分利用这段时间来加载数据。第二次调用是在resize操作中，可以看到这里只有random和count同时满足条件的情况下会做resize操作，也就是说resize加载的数据是未进行resize过的，因此，需要调整args中的图像宽高之后再重新调用`load_data(args)`加载数据。反之，不做任何处理，之前加载的数据仍然可用。第三次调用就是在数据加载完成后，将加载好的数据保存起来`train=buffer;`，然后开始下一次的加载工作。这一次的数据就会进行这一次的训练操作(调用`train_network`函数)。

## 前向传播Forward

前向传播的主函数在src/network.c中实现，代码如下：

```c++
/* 
** 前向计算网络net每一层的输出
** netp: 构建好的整个网络
** 遍历net的每一层网络，从第0层到最后一层，逐层计算每层的输出
*/
void forward_network(network *netp)
{
#ifdef GPU
    if(netp->gpu_index >= 0){
        forward_network_gpu(netp);   
        return;
    }
#endif
    network net = *netp;
    int i;
    // 遍历所有层，从第一层到最后一层，逐层进行前向传播，网络共有net.n层
    for(i = 0; i < net.n; ++i){
        // 当前处理的层为网络的第i层
        net.index = i;
        // 获取当前层
        layer l = net.layers[i];
        // 如果当前层的l.delta已经动态分配了内存，则调用fill_cpu()函数将其所有元素初始化为0
        if(l.delta){
            // 第一个参数为l.delta的元素个数，第二个参数为初始化值，为0
            fill_cpu(l.outputs * l.batch, 0, l.delta, 1);
        }
        // 前向传播: 完成当前层前向推理
        l.forward(l, net);
        // 完成某一层的推理时，置网络的输入为当前层的输出（这将成为下一层网络的输入），要注意的是，此处是直接更改指针变量net.input本身的值，
        // 也就是此处是通过改变指针net.input所指的地址来改变其中所存内容的值，并不是直接改变其所指的内容，
        // 所以在退出forward_network()函数后，其对net.input的改变都将失效，net.input将回到进入forward_network()之前时的值。
        net.input = l.output;
        if(l.truth) {
            net.truth = l.output;
        }
    }
    calc_network_cost(netp);
}
```

为了更深入的理解前向传播，我们来理解几个darknet中的经典layer的前向传播实现。

### 前向传播-卷积层

```c++
void forward_convolutional_layer(convolutional_layer l, network net)
{
    int i, j;
    
    // l.outputs=l.out_h*l.out_w*l.out_c在make各网络层函数中赋值(比如make_convolution_layer())
    // 对应每张输入图片的所有特征图的总元素个数(每张输入图片会得到n也即是l.outc张特征图)
    // 初始化输入l.output全为0.0，l.outputs*l.batch为输出的总元素个数，其中l.outputs为batch中一个
    //输入对应的输出的所有元素个数，l.batch为一个batch输入包含的图片张数
    fill_cpu(l.outputs*l.batch, 0, l.output, 1);
    
    // 是否进行二值化操作，这是干吗的?二值网络？
    if(l.xnor){
        binarize_weights(l.weights, l.n, l.c/l.groups*l.size*l.size, l.binary_weights);
        swap_binary(&l);
        binarize_cpu(net.input, l.c*l.h*l.w*l.batch, l.binary_input);
        net.input = l.binary_input;
    }
    
    int m = l.n/l.groups; // 该层的卷积核个数
    int k = l.size*l.size*l.c/l.groups; // 该层每个卷积核的参数元素个数
    int n = l.out_w*l.out_h; // 该层每个特征图的尺寸(元素个数)
    // 该循环即为卷积计算核心代码：所有卷积核对batch中每张图片进行卷积运算
    // 每次循环处理一张输入图片（所有卷积核对batch中一张图片做卷积运算）
    for(i = 0; i < l.batch; ++i){ 
        // 该循环是为了处理分组卷积
        for(j = 0; j < l.groups; ++j){
            // 当前组卷积核(也即权重)，元素个数为l.n*l.c/l.groups*l.size*l.size,
            // 共有l.n行，l.c/l.gropus,l.c*l.size*l.size列
            float *a = l.weights + j*l.nweights/l.groups;
            // 对输入图像进行重排之后的图像数据，所以内存空间申请为网络中最大占用内存
            float *b = net.workspace;
            // 存储一张输入图片（多通道）当前组的输出特征图（输入图片是多通道的，输出
            // 图片也是多通道的，有多少组卷积核就有多少组通道，每个分组后的卷积核得到一张特征图即为一个通道）
            // 这里似乎有点拗口，可以看下分组卷积原理。
            float *c = l.output + (i*l.groups + j)*n*m;
            // 由于有分组卷积，所以获取属于当前组的输入im并按一定存储规则排列的数组b，
            // 以方便、高效地进行矩阵（卷积）计算，详细查看该函数注释（比较复杂）
            // 这里的im实际上只加载了一张图片的数据
            // 关于im2col和sgemm可以看:https://blog.csdn.net/mrhiuser/article/details/52672824
            float *im =  net.input + (i*l.groups + j)*l.c/l.groups*l.h*l.w;
            // 如果这里卷积核尺寸为1，是不需要改变内存排布方式
            if (l.size == 1) {
                b = im;
            } else {
                // 将多通道二维图像im变成按一定存储规则排列的数组b，
                // 以方便、高效地进行矩阵（卷积）计算，详细查看该函数注释（比较复杂）
                // 进行重排，l.c/groups为每张图片的通道数分组，l.h为每张图片的高度，l.w为每张图片的宽度，l.size为卷积核尺寸，l.stride为步长
                // 得到的b为一张图片重排后的结果，也是按行存储的一维数组（共有l.c/l.groups*l.size*l.size行，l.out_w*l.out_h列）
                im2col_cpu(im, l.c/l.groups, l.h, l.w, l.size, l.stride, l.pad, b);
            }
            // 此处在im2col_cpu操作基础上，利用矩阵乘法c=alpha*a*b+beta*c完成对图像卷积的操作
            // 0,0表示不对输入a,b进行转置，
            // m是输入a,c的行数，具体含义为每个卷积核的个数，
            // n是输入b,c的列数，具体含义为每个输出特征图的元素个数(out_h*out_w)，
            // k是输入a的列数也是b的行数，具体含义为卷积核元素个数乘以输入图像的通道数除以分组数（l.size*l.size*l.c/l.groups），
            // a,b,c即为三个参与运算的矩阵（用一维数组存储）,alpha=beta=1为常系数，
            // a为所有卷积核集合,元素个数为l.n*l.c/l.groups*l.size*l.size，按行存储，共有l*n行，l.c/l.groups*l.size*l.size列，
            // 即a中每行代表一个可以作用在3通道上的卷积核，
            // b为一张输入图像经过im2col_cpu重排后的图像数据（共有l.c/l.group*l.size*l.size行，l.out_w*l.out_h列），
            // c为gemm()计算得到的值，包含一张输入图片得到的所有输出特征图（每个卷积核得到一张特征图），c中一行代表一张特征图，
            // 各特征图铺排开成一行后，再将所有特征图并成一大行，存储在c中，因此c可视作有l.n行，l.out_h*l.out_w列。
            // 详细查看该函数注释（比较复杂）
            gemm(0,0,m,n,k,1,a,k,b,n,1,c,n);
        }
    }

    if(l.batch_normalize){
        forward_batchnorm_layer(l, net);
    } else {
        // 加上偏置
        add_bias(l.output, l.biases, l.batch, l.n, l.out_h*l.out_w);
    }

    activate_array(l.output, l.outputs*l.batch, l.activation);
    if(l.binary || l.xnor) swap_binary(&l);
}
```

- im2col_cpu() 函数剖析

  

# 参考资料

https://blog.csdn.net/gzj2013/article/details/84837198

https://blog.csdn.net/u014540717/column/info/13752

https://github.com/hgpvision/darknet