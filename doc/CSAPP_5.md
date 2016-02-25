# Optimizing Program Performance
刚刚把CSAPP第五章看完，感触还是很深啊。看完这章之后才知道以前以为自己会的好多东西其实都是会的表象啊，深层次的东西还是需要慢慢的发掘吸收的啊！
这一章主要讲的程序优化方面的东西。从底层，（汇编层，CPU模型层）对一个程序进行分析，找出其瓶颈，并针对性的对其进行优化。最终达到一个性能的提升。
废话少说，直接上代码：
Implementation of vector abstract data type
```C++
typedef int data_t;

typedef struct{
	long int len;
	data_t *data;
}vec_rec, *vec_ptr;

#ifdef COMBIN_SUM
	#define IDENT 0
	#define OP +
#else
	#define IDENT 1;
	#define OP *;
#endif

/*Create vector of specified length*/
vec_ptr new_vec(long int len)
{
	/*Allocate header structure*/
	vec_ptr result = (vec_ptr)malloc(sizeof(vec_rec));
	if(!result)
		return NULL; /*Couldn;t allocate storage*/
	result->len = len;
	/*Allocate array*/
	if(len > 0) {
		data_t *data = (data_t*)calloc(sizeof(data_t)*len);
		if(!data){
			free((void*)result);
			return NULL;
		}
		result->data = data;
	}else
		result->data = NULL;
	return result;
}

/*
 * Retrieve vector element and store at dest.
 * Return 0 (out of bounds) or 1 (successful)
*/
int get_vec_element(vec_ptr v, long int index, data_t *dest)
{
	if(index < 0 ||  index >= v->len)
		return 0;
	*dest = v->data[index];
	return 1;
}

/* Return length of vector */
long int vec_length(vec_ptr v)
{
	return v->len;
}
```
上面是一个抽象数据类型 Vector的实现，也是后面例程的基础。
例程的主要目的是实现一个向量内所有元素的相加或者相乘，并返回结果。

```C++
/*Implementation with maximum use of data abstraction*/
void combine1(vec_ptr v, data_t *dest)
{
	long int i;
	
	*dest = IDENT;
	for(i=0; i<vec_length(v); i++) {
		data_t val;
		get_vec_element(v,i,&val);
		*dest = *dest OP　val;
	}
}
```
上面的代码是第一个版本，也是没有经过任何优化的版本。记为**combine1**其性能为(单位:CPE cycles per elements)：
![combine1 性能指标](http://7xprdq.com1.z0.glb.clouddn.com/combin1.png)

下面我们将会按照一定的原则逐步的对这段代码进行优化，从而在保证精度的同时提升函数的执行效率。

### Eliminating Loop Inefficiencies
翻译过来就是消除因为循环而带来的效率低下的情况，比如那种重复的无意义的循环调用等。


我们可以看到在 combine1 中，
> for(i=0; i<vec_length(v); i++)
 
这个for循环在判断终止条件时会调用 *vec_length*这个函数,而在for循环的每一次迭代中都会执行这个判断语句。 从另一个方面看，在for循环的处理过程中，这个向量的长度是不会变的。那么我们就可以只计算一次向量长度，并在判断终止条件中用这个变量来替代函数的调用。于是就有了下面的一版改进:**combine2**
```C++
/*Move call to vec_length out of loop*/
void combine2（vec_ptr v, data_t *dest)
{
	long int i;
	long int length = vec_length(v);
	
	*dest = IDENT;
	for(i=0; i<length; i++){
		data_t val;
		get_vec_element(v,i,&val);
		*dest = *dest OP　val;
	}
}
```
按照惯例，其性能指标为：

![combine2 性能指标](http://7xprdq.com1.z0.glb.clouddn.com/combin2.png)

从上表的比较中我们可以发现，在某些特定的数据类型下（int），这种改写有着显著的性能提升，而在有些数据类型中，则收效甚微。这又涉及到优化中的一个重要的原则：需要评估一个函数的性能瓶颈，并针对性的进行优化。这一块会在以后的部分慢慢讨论。

### Reducing Procedure Calls
**减少函数的调用**，函数调用会导致性能的低下，从而成为程序优化的一个重要模块。在**combine2**中，在每一次迭代中，我们都需要调用*get_vec_element*，去遍历向量中的元素。这个函数需要检查传进来的参数i是否越界，这是一个明显的多余的操作，因为在外层循环中就已经保证了这个i是不会超过向量的长度的，同时函数调用本身还需要消耗相应的资源。

换一种写法，假设我们给抽象数据类型*vec_rec*增加一个函数 *get_vec_start*，返回数据数组的起始地址。那么我们就可以对**combine2**进行改写，得到**combine3**:

```C++
data_t *get_vec_start(vec_ptr v)
{
	return v->data;
}

/*Direct access to vector data*/
void combine3(vec_ptr v, data_t *dest)
{
	long int i;
	long int length = vec_length(v);
	data_t *data = get_vec_start(v);

	*dest = IDENT;
	for(i=0; i<length; i++）｛
		*dest = *dest OP data[i];
	}
}
```
按照惯例，其性能指标为：
![combine3 性能指标](http://7xprdq.com1.z0.glb.clouddn.com/combin3.png)

从结果上来看，这种写法的提升更加让人迷茫，除了整数的加法外，剩余的情况几乎没有太大的变化。产生这种结果的原因和cpu运行时的一个叫 branch predict的机制有关。有机会的话，会在后面的部分写出来。

###Eliminating Unneeded Menory References
**减少不必要的内存引用/解引用**，combine3的代码中，在循环的过程中，累积计算的中间值到dest指针中，以下这段由编译器生成的汇编代码(X86_64)，可以明确的看出这个问题。
```asm
combine3: data_t= float,OP = *
i in %rdx, data in %rax,  dest in %rbp
.L498:							Loop:
	movss (%rbp), %xmm0 		Read product from dest
	mulss (%rax,%rdx,4), %xmm0	Multiply product by data[i]
	movss %xmmo, (%rbp)			Store product at dest
	addq  $1, %rdx				Increment i
	cmpq  %rdx, %r12			Compare i: limit
	jg 	  .L498					if > goto Loop 
```
从上述的汇编代码中我们可以看到，在第i次迭代的过程中，程序读取位于dest位置的的数据，拿他乘以data[i]，然后把乘积存回dest。这里的读写操作就是多余的，因为在每次迭代中，读取进来的值和上一次迭代结束后存回去的值是同一个值。所以我们可以拿掉这个不必要的读写操作，得到**combine4**：
```C++
/*Accumulate result in local variable*/
void combine4(vec_ptr v, data_t *dest)
{
	long int i;
	lont int length = vec_length(v);
	data_t *data = get_vec_start(v);
	data_t acc = IDENT;
	
	for(i=0; i<length; i++){
		acc = acc OP data[i];
	}
	*dest = acc;
}
```
和**combine3**相比，在循环的每一次迭代中，我们把内存操作从两个读一个写操作降到了一个读操作：
```asm
combine4: data_t = float, OP　＝　＊

i in %rdx, data in %rax, limit in %rbp, acc in %xmm0

.L488:							Loop:
	mulss (%rax,%rdx,4), %xmm0	Mutilpy acc by data[i]
	addq  $1, %rdx				Increment i
	cmpq  %rdx, %rbp 			Compare	limit :i
	jg    .L48					if > , goto loop
```
通过这么简单的一个改写，性能提升效果如下：
![combine4 性能指标](http://7xprdq.com1.z0.glb.clouddn.com/combin4.png)

从上面四个代码的改写过程，我们可以看到一个显著的性能提升，而这些改写都是没有依赖任何目标机器的特性。换句话说就是这写原则是通用的。
下面针对现代的cpu特性，我们还可以进一步做一些优化操作：
### Loop Unrolling
**循环展开**，循环展开可以从如下两个方面提升程序的性能，其一是减少了对程序结果没有贡献的额外的操作，比如循环变量和条件分支。 其二是他给我们提供了一个更广泛的优化空间，从而可以减少在主路径上的操作，提升性能。从而我们得到了**combine5**：
```C++
/*Unroll loop by 2*/
void combine5(vec_ptr v, data_t *dest)
{
	long int i;
	long int length = vec_length(v);
	long int limit = length-1;
	data_t *data = get_vec_start(v);
	data_t acc = IDENT;

	/*Combine 2 elements at a time*/
	for(i=0; i<limit; i+=2){
		acc = (acc OP data[i])　OP data[i+1];
	}
	
	/*Finish any remaining elements*/
	for(; i<length; i++) {
		acc = acc OP　data[i];
	}
	*dest = acc;
}
```
其性能提升为：
![combine5 性能指标](http://7xprdq.com1.z0.glb.clouddn.com/combin5.png)
###Enhancing Parallelism
**增加并行**，combine5提升了 int 型的性能， 然而却于float,double 无补。问题出在哪里呢？
请看下面的一版改写**combine6**：
```C++
/*Unroll loop by 2, 2-way parallelism */
void combine6(vec_ptr v, data_t *dest)
{
	long int i;
	long int length = vec_length(v);
	long int limit = length - 1;
	data_t *data = get_vec_start(v);
	data_t acc0 = IDENT;
	data_t acc1 = IDENT;

	/*Combine 2 elements at a time*/
	for(i=0; i<limit; i+=2){
		acc0 = acc0 OP data[i];
		acc1 = acc1 OP data[i+1];
	}

	/*Finish any remaining elements*/
	for(; i<length; i++){
		acc0 = acc0 OP data[i];
	}
	*dest = acc0 OP acc1;
}
```
通过一个上述一个简单的操作，我们可以获取以下的性能提示：
![](http://7xprdq.com1.z0.glb.clouddn.com/combin6.png)

到这一步其实关于这个函数的优化问题就基本上可以告一段落了，读书笔记也到此结束了。接下来要完成的就是针对这从头到尾的这些个优化做一些分析了。比如性能测试表中的数值是怎么得到的，比如为什么有的操作只对int提升效果较大，对float和double却几乎没有什么提升。比如**combine6**的写法有没有什么问题。会不会导致程序的结果不一样等…… 问题是无尽的。要想彻底回答以上种种疑问，就需要对编译器的优化准则，对现代CPU模型有一个深入的了解。嗯，我会重读这一章，再写一个全局意义上的感想与大家分享。[未完待续](#)