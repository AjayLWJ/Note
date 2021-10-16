[toc]

## linux设备驱动中的并发控制

### 并发与竟态	

​		并发( Concurrency)指的是**多个执行单元同时、并行被执行**，而并发的执行单元**对共享资源(硬件资源和软件上的全局变量、静态变量等)的访问则很容易导致竞态**。

​		在linux内核中，主要的竟态发生于如下几种情况：

1. **对称多处理器(SMP)的多个CPU**

   ​		SMP是一种紧耦合、共享存储的系统模型，其体系结构如下图所示，它的**特点是多个CPU使用共同的系统总线，因此可访问共同的外设和储存器**。

   ![image-20201012152437697](.\image\SMP体系结构.png)

   ​		在SMP的情况下，两个核(CPU0和CPU1)的竞态可能发生于CPU0的进程与CPU1的进程之间、CPU0的进程与CPU1的中断之间以及CPU0的中断与CPU1的中断之间，图中任何一条线连接的两个实体都有核间并发可能性。

   ![image-20200629102232342](.\image\SMP下多核之间的竟态.png)

2. **单CPU内进程与抢占它的进程**

   ​		**linux2.6以后的内核支持内核抢占调度**，一个进程在内核执行的时候可能消耗完了自己的时间片，也可能被另一个高优先级进程打断，进程与抢占它的进程访问共享资源的情况类似于SMP的多个CPU。

3. **中断（硬中断、软中断、Tasklet、底半部）与进程之间**

   ​		中断可以打断正在执行的进程，如果中断服务程序访问进程正在访问的资源，则竞态也会发生。

   ​		此外，中断也有可能被新的更高优先级的中断打断，因此，多个中断之间本身也可能引起并发而导致竞态。但是 Linux2.6.35之后，就取消了中断的嵌套。老版本的内核可以在申请中断时，设置标记 IRQF_DISABLED以避免中断嵌套，由于新内核直接就默认不嵌套中断，这个标记反而变得无用了。

​		上述并发的发生除了**SMP是真正的并行**以外，其他的都是**单核上的“宏观并行，微观串行”**，但其引发的实质问题和SMP相似。下图再现了SMP情况下总的竞争状态可能性，既包含某一个核内的，也包括两个核间的竞态。

![image-20200629103529447](.\image\SMP下核间与核内竟态.png)

​		**解决竞态问题的途径是保证对共享资源的互斥访问**，所谓互斥访问是指一个执行单元在访问共享资源的时候，其他的执行单元被禁止访问。
​		**访问共享资源的代码区域**称为**临界区**( Critical sections)，临界区需要被以某种互斥机制加以保护。**中断屏蔽、原子操作、自旋锁、信号量、互斥体等是 Linux设备驱动中可采用的互斥途径**。

### 编译乱序和执行乱序

​		现代的高性能编译器在目标码优化上都具备对指令进行乱序优化的能力。编译器可以对访存的指令进行乱序，减少逻辑上不必要的访存，以及尽量提高Cache命中率和CPU的Load/Store单元的工作效率。

​		**解决编译乱序问题，需要通过 barrier()編译屏障进行**。可以在代码中设置 barrier()屏障，这个屏障可以阻挡编译器的优化。对于编译器来说，设置编译屏障可以保证屏障前的语句和屏障后的语句不乱“串门”。

​		**编译乱序是编译器的行为，而执行乱序则是处理器运行时的行为**。高级的CPU可以根据自己缓存的组织特性，将访存指令重新排序执行。连续地址的访问可能会先执行，因为这样缓存命中率高。有的还允许访存的非阻塞，即如果前面一条访存指令因为缓存不命中，造成长延时的存储访问时，后面的访存指令可以先执行，以便从缓存中取数。因此，即使是从汇编上看顺序正确的指令，其执行的顺序也是不可预知的。

​		**处理器为了解决多核间一个核的内存行为对另外一个核可见的问题，引入了一些内存屏障的指令**。譬如，**ARM处理器的屏障指令包括**：

​		DMB(数据内存屏障)：在DMB之后的显式内存访问执行前，保证所有在DMB指令之前的内存访问完成；

​		DSB(数据同步屏障)：等待所有在DSB指令之前的指令完成(位于此指令前的所有显式内存访问均完成，位于此指令前的所有缓存、跳转预测和TLB维护操作全部完成)；

​		ISB(指令同步屏障)：Flush流水线，使得所有ISB之后执行的指令都是从缓存或内存中获得的。

​		Linux内核的自旋锁、互斥体等互斥逻辑，需要用到上述指令：在请求获得锁时，调用屏障指令；在解锁时，也需要调用屏障指令。

​		在 Linux内核中，定义了读写屏障mb()、读屏障rmb()、写屏障wmb()、以及作用于寄存器读写的 _iormb()、 _iowmb()这样的屏障API。读写寄存器的 readl_relaxed()和 readI()、writel_relaxed()和 writel()API的区别就体现在有无屏障方面。

```c
#define readb(c)		((u8 __v = readb_relaxed(c)； __iormb()； __v；))
#define readw(c)		((u16 __v = readw_relaxed(c)； __iormb()； __v；))
#define readl(c)		((u32 __v = readl_relaxed(c)； __iormb()； __v；))

#define writeb(v，c)		(( __iowmb()； writeb_relaxed(v， c)； ))
#define writew(v，c)		(( __iowmb()； writew_relaxed(v， c)； ))
#define writel(v，c)		(( __iowmb()； writel_relaxed(v， c)； ))
```

### 中断屏蔽

​		在**单CPU范围**内避免竞态的一种简单而有效的方法是**在进入临界区之前屏蔽系统的中断**，但是在驱动编程中**不值得推荐**，驱动通常需要考虑跨平台特点而不假定自己在单核上运行。CPU一般都具备屏蔽中断和打开中断的功能，这项功能可以保证正在执行的内核执行路径不被中断处理程序所抢占，防止某些竞态条件的发生。具体而言，中断屏蔽将使得中断与进程之间的并发不再发生，而且，由于 Linux内核的进程调度等操作都依赖中断来实现，内核抢占进程之间的并发也得以避免了。

​		中断屏蔽的使用方法为：

```c
local_irq_disable()			/* 屏蔽中断 */
. . .
critical section			/* 临界区 */
. . .
local_irq_enable()			/* 开中断 */
```

​		其底层的实现原理是**让CPU本身不响应中断**。

​		由于 Linux的异步IO、进程调度等很多重要操作都依赖于中断，中断对于内核的运行非常重要，在屏蔽中断期间所有的中断都无法得到处理，因此长时间屏蔽中断是很危险的，这有可能造成数据丢失乃至系统崩溃等后果。这就要求**在屏蔽了中断之后，当前的内核执行路径应当尽快地执行完临界区的代码**。

​		`local_irq_disable()`和` local_irq_enabled()`都只能禁止和使能本CPU内的中断，因此，并**不能解决SMP多CPU引发的竟态**。因此，**单独使用中断屏蔽通常不是一种值得推荐的避免竟态的方法**(换句话说，驱动中使用 `local_irq_disable/enable` 通常意味着一个bug)，它适合与下文将要介绍的自旋锁联合使用。

​		与local_irq_disable()不同的是， **local_irq_save(flags))除了进行禁止中断的操作以外，还保存目前CPU的中断位信息**，local_irq_restore(flags)进行的是与 local_irq_save(flags)相反的操作。对于ARM处理器而言，其实就是保存和恢复CPSR。
​		如果只是想禁止中断的底半部，应使用 local_bh_disabled()，使能被  local_bh_disabled()禁止的底半部应该调用local_bh_enabled()。

### 原子操作

​		原子操作可以保证对一个整型数据的修改是排他性的。 Linux内核提供了一系列函数来实现内核中的原子操作，这些函数又分为两类，分别**针对位和整型变量进行原子操作**。位和整型变量的原子操作都依赖于底层CPU的原子操作，因此所有这些函数都与CPU架构密切相关。对于ARM处理器而言，底层使用LDREX和STREX指令。

### 自旋锁

​		自旋锁( Spin Lock)是一种典型的对临界资源进行互斥访问的手段，其名称来源于它的工作方式。为了获得一个自旋锁，在某CPU上运行的代码需先执行一个原子操作，该操作测试并设置(Test-And-Set)某个内存变量。由于它是原子操作，所以在该操作完成之前其他执行单元不可能访问这个内存变量。如果测试结果表明锁已经空闲，则程序获得这个自旋锁并继续执行；如果测试结果表明锁仍被占用，程序将在一个小的循环内重复这个“测试并设置”操作，即进行所谓的“自旋”，通俗地说就是“**在原地打转**”。当自旋锁的持有者通过重置该变量释放这个自旋锁后，某个等待的“测试并设置”操作向其调用者报告锁已释放。

​		理解自旋锁最简单的方法是把它作为一个变量看待，该变量把一个临界区标记为“我当前在运行，请稍等一会”或者标记为“我当前不在运行，可以被使用”。如果A执行单元首先进入例程，它将持有自旋锁;当B执行单元试图进入同一个例程时，将获知自旋锁已被持有，需等到A执行单元释放后才能进入。

​		在ARM体系结构下，自旋锁的实现借用了ldrex指令、strex指令、ARM处理器内存屏蔽指令dmb和dsb、wfe指令和sev指令，可以说既要保证排他性，也要处理好内存屏障。

**Linux中与自旋锁相关的操作主要有以下4种：**

1. **定义自旋锁**
   `spinlock_t lock;`

2. **初始化自旋锁**
   `spin_lock_init(lock)`
   该宏用于动态初始化自旋锁lock。

3. **获得自旋锁**
   `spin_lock (lock)`
   该宏用于获得自旋锁lock，如果能够立即获得锁，它就马上返回，否则，它将在那里自旋，直到该自旋锁的保持者释放。
   `spin_trylock (lock)`
   该宏尝试获得自旋锁lock，如果能立即获得锁，它获得锁并返回true，否则立即返回false，实际上不再“在原地打转”。

4. **释放自旋锁**
   `spin_unlock (lock)`

   该宏释放自旋锁lock，它与`spin_trylock`或`spin_lock`配对使用。

   自旋锁一般这样被使用：

   ```c
   /* 定义一个自旋锁 */
   spinlock_t lock;
   spin_lock_init(&lock);
   
   spin_lock(&lock);			/* 获取自旋锁，保护临界区 */
   . . . /* 临界区 */
   spin_unlock(&lock);			/* 解锁 */
   ```

​		**自旋锁主要针对SMP或单CPU但内核可抢占的情况，对于单CPU和内核不支持抢占的系统，自旋锁退化为空操作**。在单CPU和内核可抢占的系统中，自旋锁持有期间中内核的抢占将被禁止。由于内核可抢占的单CPU系统的行为实际上很类似于SMP系统，因此，在这样的单CPU系统中使用自旋锁仍十分必要。另外，**在多核SMP的情况下，任何一个核拿到了自旋锁，该核上的抢占调度也暂时禁止了，但是没有禁止另外一个核的抢占调度。**

​		尽管用了自旋锁可以保证临界区不受别的CPU和本CPU内的抢占进程打扰，但是得到锁的代码路径在执行临界区的时候，还可能受到中断和底半部(BH)的影响。为了防止这种影响，就需要用到**自旋锁的衍生**。 `spin_lock()/spin_unlock()`是自旋锁机制的基础，它们和关中断` local_irq_disable()`/开中断` local_irq_enable()`、关底半部 `local_bh_disabled()`/开底半部` local_bh_enabled()`、关中断并保存状态字` local_irq_save()`/开中断并恢复状态字` local_irq_restored()`结合就形成了整套自旋锁机制，关系如下：

```c
spin_lock_irq() = spin_lock() + local_ira_disabled()
spin_unlock_irq() = spin_unlock() + local_irq_enable()
spin_lock_irqsave() = spin_lock() + local_irg_save()
spin_unlock_irqrestore() = spin_unlock() + local_irq_restore()
spin_lock_bh() = spin_lock() + local_bh_disable() 
spin_unlock_bh() = spin_unlock() + local_bh_enable()
```

​		`spin_lock_irq()、 spin_lock_irqsave()、 spin_lock_bh()`类似函数会为自旋锁的使用系好“安全带”以避免突如其来的中断驶入对系统造成的伤害。

​		驱动工程师应谨慎使用自旋锁，而且在使用中还要特别注意如下几个问题：

​		1) 自旋锁实际上是**忙等锁**，当锁不可用时，CPU一直循环执行“测试并设置”该锁直到可用而取得该锁，CPU在等待自旋锁时不做任何有用的工作，仅仅是等待。因此，**只有在占用锁的时间极短的情况下，使用自旋锁才是合理的**。当临界区很大，或有共享设备的时候，需要较长时间占用锁，使用自旋锁会降低系统的性能。

​		2) **自旋锁可能导致系统死锁**。引发这个问题最常见的情况是递归使用一个自旋锁，即如果一个已经拥有某个自旋锁的CPU想第二次获得这个自旋锁，则该CPU将死锁。

​		3) **在自旋锁锁定期间不能调用可能引起进程调度的函数**。如果进程获得自旋锁之后再阻塞，如调用`copy_from_user()`、` copy_to_user()`、` kmalloc()`和 `msleep()`等函数，则可能导致内核的崩溃。

​		4) **在单核情况下编程的时候，也应该认为自己的CPU是多核的，驱动特别强调跨平台的概念**。比如，在单CPU的情况下，若中断和进程可能访问同一临界区，进程里调用` spin_lock_irqsave()`是安全的，在中断里其实不调用 ` spin_lock()`也没有问题，因为` spin_lock_irqsave()`可以保证这个CPU的中断服务程序不可能执行。但是，若CPU变成多核，` spin_lock_irqsave()`不能屏蔽另外一个核的中断，所以另外一个核就可能造成并发问题。因此，无论如何，我们在中断服务程序里也应该调用 ` spin_lock()`。

#### 读写自旋锁

​		自旋锁不关心锁定的临界区究竟在进行什么操作，不管是读还是写，它都一视同仁。即便多个执行单元同时读取临界资源也会被锁住。实际上，**对共享资源并发访问时，多个执行单元同时读取它是不会有问题的**，自旋锁的衍生锁**读写自旋锁**( relock)可允许读的并发。读写自旋锁是一种比自旋锁粒度更小的锁机制，**它保留了“自旋”的概念，但是在写操作方面，只能最多有1个写进程，在读操作方面，同时可以有多个读执行单元**。当然，**读和写也不能同时进行。**

​		**读写自旋锁涉及的操作如下：**

1. **定义和初始化读写自旋锁**

   ```c
   rwlock_t my_rwlock;
   rwlock_init(&my_rwlock);		/* 动态初始化 */
   ```

2. **读锁定**

   ```c
   void read_lock(rwlock_t *lock);
   void read_lock_irqsave(rwlock_t *lock， unsigned long flags);
   void read_lock_irq(rwlock_t *lock);
   void read_lock_bh(rwlock_t *lock);
   ```

3. **读解锁**

   ```c
   void read_unlock(rwlock_t *lock);
   void read_unlock_irqrestore(rwlock_t *lock， unsigned long flags);
   void read_unlock_irq(rwlock_t *lock);
   void read_unlock_bh(rwlock_t *lock);
   ```

   ​		在对共享资源进行读取之前，应该先调用读锁定函数，完成之后应调用读解锁函数。

4. **写锁定**

   ```c
   void write_lock(rwlock_t *lock);
   void write_lock_irqsave(rwlock_t *lock， unsigned long flags);
   void write_lock_irq(rwlock_t *lock);
   void write_lock_bh(rwlock_t *lock);
   int write_trylock(rwlock_t *lock);
   ```

5. **写解锁**

   ```c
   void write_unlock(rwlock_t *lock);
   void write_unlock_irqrestore(rwlock_t *lock， unsigned long flags);
   void write_unlock_irq(rwlock_t *lock);
   void write_unlock_bh(rwlock_t *lock);
   ```

   ​		在对共享资源进行写之前，应该先调用写锁定函数，完成之后应调用写解锁函数。和`spin_trylock()`一样，`write_trylock()`也只是尝试获取读写自旋锁，不管成功失败，都会立即返回。

​		读写自旋锁一般这样被使用：

```c
rwlock_t lock;						/* 定义rwlock */
rwlock_init(&lock);					/* 初始化rwlock */

/* 读时获取锁 */
read_lock(&lock);
. . .								/* 临界资源 */
read_unlock(&lock);

/* 写时获取锁 */
write_lock_irqsave(&lock， flags);
. . .								/* 临界资源 */
write_unlock_irqrestore(&lock， flags);
```

#### 顺序锁

​		**顺序锁( seqlock)是对读写锁的一种优化**，若使用顺序锁，**读执行单元不会被写执行单元阻塞**，也就是说，读执行单元在写执行单元对被顺序锁保护的共享资源进行写操作时仍然可以继续读，而不必等待写执行单元完成写操作，写执行单元也不需要等待所有读执行单元完成读操作才去进行写操作。但是，**写执行单元与写执行单元之间仍然是互斥的**，即**如果有写执行单元在进行写操作，其他写执行单元必须自旋在那里，直到写执行单元释放了顺序锁**。

​		对于顺序锁而言，尽管读写之间不互相排斥，但是**如果读执行单元在读操作期间，写执行单元已经发生了写操作，那么，读执行单元必须重新读取数据，以便确保得到的数据是完整的**。所以，在这种情况下，读端可能反复读多次同样的区域才能读到有效的数据。

​		在linux内核中，**写执行单元涉及的顺序锁操**作如下：

1. **获得顺序锁**

   ```c
   void write_seqlock(seqlock_t *sl);
   int write_tryseqlock(seqlock_t *sl);
   write_seqlock_irqsave(lock， flags)
   write_seqlock_irq(lock)
   write_seqlock_bh(lock)
   ```

   其中，

   ```c
   write_seqlock_irqsave() = local_irq_save() + write_seqlock()
   write_seqlock_irq() = local_irq_disable() + write_seqlock()
   write_seqlock_bh() = local_bh_disable() + write_seqlock()
   ```

2. **释放顺序锁**

   ```c
   void write_sequnlock(seqlock_t *sl);
   write_sequnlock_irqrestore(lock， flags)
   write_sequnlock_irq(lock)
   write_sequnlock_bh(lock)
   ```

   其中，

   ```
   write_sequnlock_irqrestore() = write_sequnlock() + local_irq_restore()
   write_sequnlock_irq() = write_sequnlock() + local_irq_enable()
   write_seqlock_bh() = write_sequnlock() + local_bh_enable()
   ```

   写执行单元使用顺序锁的模式如下：

   ```
   write_seqlock(&seqlock_a);
   . . .								/* 写操作代码块 */
   write_sequnlock(&seqlock_a);
   ```

   **读执行单元涉及的顺序锁操作如下：**

1. **读开始**

   ```c
   unsigned read_seqbegin(const seqlock_t *sl);
   read_seqbegin_irqsave(lock， flags)
   ```

   ​		读执行单元在对被顺序锁sl保护的共享资源进行访问前需要调用该函数，该函数返回顺序锁sl的当前序号。其中，

   ```c
   read_seqbegin_irqsave() = local_irq_save() + read_seqbegin()
   ```

2. **重读**

   ```c
   int read_seqretry(const seqlock_t *sl， unsigned iv);
   read_seqretry_irqrestore(lock， iv， flags)
   ```

   ​		读执行单元在访问完被顺序锁sl保护的共享资源后需要调用该函数来检查，在读访问期间是否有写操作。如果有写操作，读执行单元就需要重新进行读操作。其中，

   ```c
   read_seqretry_irqrestore() = read_seqretry() + local_irq_restore()
   ```

   读执行单元使用顺序锁的模式如下：

   ```c
   do {
   	seqnum = read_seqbegin(&seqlock_a);
       /* 读操作代码块 */
       . . .
   } while (read_seqretry(&seqlock_a， seqnm));
   ```

#### 读-复制-更新

​		RCU(Read-Copy-Update，读-复制-更新)，它是基于其原理命名的。不同于自旋锁，使用RCU的读端没有锁、内存屏障、原子指令类的开销，几乎可以认为是直接读(只是简单地标明读开始和读结束)，而RCU的写执行单元在访问它的共享资源
前**首先复制一个副本，然后对副本进行修改，最后使用一个回调机制在适当的时机把指向原来数据的指针重新指向新的被修改的数据**，这个时机就是所有引用该数据的CPU都退出对共享数据读操作的时候。等待适当时机的这一时期称为宽限期( Grace Period)。

​		比如，有下面的一个由`struct foo`结构体组成的链表：

```c
struct foo {
	struct list_head list;
    int a;
    int b;
    int c;
};
```

​		假设进程A要修改链表中某个节点N的成员a、b。**自旋锁的思路**是排他性地访问这个链表，等所有其他持有自旋锁的进程或者中断把自旋锁释放后，进程A再拿到自旋锁访问链表并找到N节点，之后修改它的a、b两个成员，完成后解锁。而**RCU的思路**则不同，它直接制造一个新的节点M，把N的内容复制给M，之后在M上修改a、b，并用M来代替N原本在链表的位置。之后进程A等待在链表前期已经存在的所有读端结束后(即宽限期，通过下文说的 `synchronize_rcu( )`API完成)，再释放原来的N。用代码来描述这个逻辑就是:

```c
struct foo {
	struct list_head list;
    int a;
    int b;
    int c;
};
LIST_HEAD(head);

/* . . . */

p = search(head， key);
if (p == NULL) {
    /* Take appropriate action， unlock， and return */
}

q = kmalloc(sizeof(*p)， GFP_KERNEL);
*q = *p;
q->b = 2;
q->c = 3;
list_replace_rcu(&p->list， &q->list);
synchronize_rcu();
kfree(p);
```

​		RCU可以看作读写锁的高性能版本，相比读写锁，**RCU的优点在于既允许多个读执行单元同时访问被保护的数据，又允许多个读执行单元和多个写执行单元同时访问被保护的数据**。但是，**RCU不能替代读写锁，因为如果写比较多时，对读执行单元的性能提高不能弥补写执行单元同步导致的损失**。因为使用RCU时，写执行单元之间的同步开销会比较大，它需要延迟数据结构的释放，复制被修改的数据结构，它也必须使用某种锁机制来同步并发的其他写执行单元的修改操作。

​		**linux中提供的RCU操作包括如下4中：**

1. **读锁定**

   ```c
   rcu_read_lock()
   rcu_read_lock_bh()
   ```

2. **读解锁**

   ```c
   rcu_read_unlock()
   rcu_read_unlock_bh()
   ```

   使用RCU进行读的模式如下：

   ```c
   rcu_read_lock()
   . . .						/* 读临界区 */
   rcu_read_unlock()
   ```

3. **同步RCU**

   `synchronize_rcu()`

   ​		该函数由RCU写执行单元调用，**它将阻塞写执行单元，直到当前CPU上所有的已经存在(Ongoing)的读执行单元完成读临界区，写执行单元才可以继续下一步操作**。`synchronize_rcu()`并不需要等待后续( Subsequent)读临界区的完成，如图所示。

   ![image-20200702104645269](.\image\synchronize_rcu.png)

4. **挂接回调**

   ```c
   void call_rcu(struct rcu_head *head， void (*fun)(struct rcu_head *rcu));
   ```

   ​		函数`call_rcu()`也由RCU写执行单元调用，与`synchronize_rcu()`不同的是，它不会使写执行单元阻塞，因而可以在中断上下文或软中断中使用。该函数把函数`func`挂接到RCU回调函数链上，然后立即返回。挂接的回调函数会在一个宽限期结束(即所有已经存在的RCU读临界区完成)后被执行。

   #### **没看懂，待补充。。。。。**



### 信号量

​		信号量( Semaphore)是操作系统中最典型的用于同步和互斥的手段，信号量的值可以是0、1或者n。**信号量与操作系统中的经典概念PV操作对应**。

P(S): ① 将信号量S的值减1，即S = S-1;

​		 ② 如果S≥0，则该进程继续执行;否则该进程置为等待状态，排人等待队列。

V(S): ① 将信号量S的值加1，即S = S+1;

​		 ② 如果S>0，唤醒队列中等待信号量的进程。

**Linux中与信号量相关的操作主要有下面几种：**

1. **定义信号量**

   下列代码定义名称为scm的信号量：

   `struct semaphore sem;`

2. **初始化信号量**

   `void sema_init(struct semaphore *sem， int val);`

   该函数初始化信号量，并设置信号量sem的值为val。

3. **获得信号量**

   `void down(struct semaphore *sem);`

   **该函数用于获得信号量sem，它会导致睡眠，因此不能在中断上下文中使用**。

   `int down_interruptible(struct semaphore *sem);`

   该函数功能与down类似，不同之处为，因为 down()进入睡眠状态的进程不能被信号打断，但因为 `down_interruptible()`进入睡眠状态的进程能被信号打断，信号也会导致该函数返回，这时候函数的返回值非0。

   `int down_trylock(struct semaphore *sem);`

   该函数尝试获得信号量sem，如果能够立刻获得，它就获得该信号量并返回0，否则，返回非0值。它不会导致调用者睡眠，可以在中断上下文中使用。
   在使用 `down_interruptible()`获取信号量时，对返回值一般会进行检查，如果非0，通常立即返回`-ERESTARTSYS`，如:

   ```c
   if (down_interruptible(&sem))
   	return -ERESTARTSYS:
   ```

4. **释放信号量**

   `void up(struct semaphore *sem);`

   该函数释放信号量sem，唤醒等待者。

​		作为一种可能的互斥手段，信号量可以保护临界区，它的使用方式和自旋锁类似。与自旋锁相同的是，**只有得到信号量的进程才能执行临界区代码**。但是，与自旋锁不同的是，**当获取不到信号量时，进程不会原地打转而是进入休眠等待状态**。用作互斥时，信号量一般这样被使用：

```c
进程P1			进程P2			......			进程Pn
......			 ......							  ......
P(S);			 P(S);							  P(S);	
临界区;		   临界区;		   					  临界区;
V(S);			 V(S);							  V(S);	
......			 ......			  ......		  ......
```

​		由于新的linux内核倾向于直接使用`mutex`作为互斥手段，信号量用作 互斥不再被推荐使用。

​		信号量也可以用于同步，一个进程A执行`down()`等待信号量，另外一个进程B执行`up()`释放信号量，这样进程A就同步地等待了进程B。其过程类似：

![image-20200702170824464](.\image\进程同步问题.png)

### 互斥体

​		尽管信号量已经可以实现互斥的功能，但是“正宗”的 mutex在 Linux内核中还是真实地存在着。

​		下面代码定义了名为 my mutex的互斥体并初始化它：

```c
struct mutex my mutex;
mutex_init(&my mutex);
```

​		下面的两个函数用于获取互斥体：

```c
void mutex_lock(struct mutex *lock);
int mutex_lock_interruptible(struct mutex *lock);
int mutex_trylock(struct mutex *lock);
```

​		`mutex_lock()`与 `mutex_lock_interruptible()`的区别和 `down()`与 `down_trylock()`的区别完全一致，前者引起的睡眠不能被信号打断，而后者可以。 `mutex_trylock()`用于尝试获得 `mutex`，获取不到`mutex`时不会引起进程睡眠。

​		下列函数用于释放互斥体：

```c
void mutex_unlock(struct mutex *lock);
```

`mutex`的使用方法和信号量用于互斥的场合完全一样：

```c
struct mutex my_mutex;					/* 定义 mutex */
mutex_init(&my_mutex);					/* 初始化 mutex */

mutex_lock(&my_mutex);					/* 获取 mutex */
. . .									/* 临界资源 */
mutex_unlock(&my_mutex);				/* 释放 mutex */
```

​		自旋锁和互斥体都是解决互斥问题的基本手段，面对特定的情况，应该如何取舍这两种手段呢?选择的依据是临界区的性质和系统的特点。
​		从严格意义上说，**互斥体和自旋锁属于不同层次的互斥手段**，前者的实现依赖于后者。在互斥体本身的实现上，为了保证互斥体结构存取的原子性，需要自旋锁来互斥。**所以自旋锁属于更底层的手段**。
​		**互斥体是进程级的**，用于多个进程之间对资源的互斥，虽然也是在内核中，但是该内核执行路径是以进程的身份，代表进程来争夺资源的。如果竞争失败，会发生进程上下文切换，当前进程进入睡眠状态，CPU将运行其他进程。鉴于进程上下文切换的开销也很大，因此，**只有当进程占用资源时间较长时，用互斥体才是较好的选择。**
​		**当所要保护的临界区访问时间比较短时，用自旋锁是非常方便的，因为它可节省上下文切换的时间**。但是CPU得不到自旋锁会在那里空转直到其他执行单元解锁为止，所以**要求锁不能在临界区里长时间停留，否则会降低系统的效率**。

​		由此，可以总结出**自旋锁和互斥体选用的3项原则**。

​		1) 当锁不能被获取到时，使用互斥体的开销是进程上下文切换时间，使用自旋锁的开销是等待获取自旋锁(由临界区执行时间决定)。**若临界区比较小，宜使用自旋锁，若临界区很大，应使用互斥体**。

​		2) **互斥体所保护的临界区可包含可能引起阻塞的代码，而自旋锁则绝对要避免用来保护包含这样代码的临界区**。因为阻塞意味着要进行进程的切换，如果进程被切换出去后，另个进程企图获取本自旋锁，死锁就会发生。

​		3) **互斥体存在于进程上下文，因此，如果被保护的共享资源需要在中断或软中断情况下使用，则在互斥体和自旋锁之间只能选择自旋锁**。当然，如果一定要使用互斥体，则只能通过`mutex_trylock()`方式进行，不能获取就立即返回以避免阻塞。

### 完成量

​		linux提供了完成量，它用于一个执行单元等待另一个执行单元执行完某事。

​		**linux中与完成量相关的操作主要有以下4种：**

1. **定义完成量**

   下列代码定义名为`my_completion`的完成量：

   `struct completion my_completion;`

2. **初始化完成量**

   下列代码初始化或者重新初始化`my_completion`这个完成量的值为0(即没有完成的状态)：

   ```c
   init_completion(&my_completion);
   reinit_completion(&my_completion)
   ```

3. **等待完成量**

   下列函数用于等待一个完成量被唤醒：

   `void wait_for_completion(struct completion *c);`

4. **唤醒完成量**

   下面两个函数用于唤醒完成量：

   ```c
   void completion(struct completion *c);
   void completion_all(struct completion *c);
   ```

   前者唤醒一个等待的执行单元，后者释放所有等待同一完成量的执行单元。

   完成量用于同步的流程一般如下：

   ```c
   进程P1					进程P2
   代码区C1;					wait_for_completion(&done);
   complete(&done);
   						 代码区C2;	
   ```

### 增加并发控制后的globalmen的设备驱动

​		在`globalmen()`的读写函数中，由于要调用`copy_from_user()、copy_to_user()`这些可能导致阻塞的函数，因此不能使用自旋锁，宜使用互斥体。

​		驱动工程师习惯将某设备所使用的自旋锁、互斥体等辅助手段也放在设备结构中，因此，可如下修改`globalmem_dev`结构体的定义。

```c
/* 增加并发控制后的globalmem设备结构体 */
struct globalmem_dev {
    struct cdev cdev;
    unsigned char men[GLOBALMEM_SIZE];
    struct mutex mutex;
}
```

​		并在模块初始化函数中初始化这个信号量。

```c
/* 增加并发控制后的globalmem设备驱动模块加载函数 */
static int __init globalmem_init(void)
{
    int ret;
    dev_t devno = MKDEV(globalmem_major， 0);
    
    if (globalmem_major)
        ret = register_chrdev_region(devno， 1， "globalmem");
    else {
        ret = alloc_chrdev_region(&devno， 0， 1， "globalmem");
        globalmem_major = MAJOR(devno);
    }
    if (ret <  0)
        return ret;
    
    globalmem_devp = kzalloc(sizeof(struct globalmem_dev)， GFP_KERNEL);
    if (!globalmen_devp) {
        ret = -ENOMEM;
        goto fail_malloc;
    }
    
    mutex_init(&globalmem_devp->mutex);
    globalmem_setup_cdev(globalmem_devp， 0);
    return 0;
    
    fail_malloc:
    	unregister_chrdev_region(devno， 1);
    	return ret;
}

module_init(globalmem_init);
```

​		在访问 `globalmem_dev`中的共享资源时，需先获取这个互斥体，访问完成后，随即释放这个互斥体。驱动中新的 `globalmem`读、写操作如下代码所示。

```c
/* 增加并发控制后的globalmem读写操作 */
static ssize_t globalmem_read(struct file *filp， char __user *buf， size_t size， loff_t *ppos)
{
    unsigned long p = *ppos;
    unsigned int count = size;
    int ret = 0;
    struct globalmem_dev *dev = filp->private_data;
    
    if (p >= GLOBALMEM_SIZE)
        return 0;
    if (count > GLOBALMEM_SIZE - p)
        count = GLOBALMEM_SIZE - p;
    
    mutex_lock(&dev->mutex);
    
    if (copy_to_user(buf， dev->mem + p， count)) {
        ret = -EFAULT;
	} else {
        *ppos += count;
        ret = count;
        printk(KERN_INFO "read %u bytes(s) from %lu\n"， count， p);
    }
    
    mutex_unlock(&dev->mutex);
    
    return ret;
}

static ssize_t globalmem_write(struct file *filp， const char __user *buf， size_t size， loff_t *ppos)
{
    unsigned long p = *ppos;
    unsigned int count = size;
    int ret = 0;
    
    if (p >= GLOBALMEM_SIZE)
        return 0;
    if (count > GLOBALMEM_SIZE - p)
        count = GLOBALMEM_SIZE - p;
    
    mutex_lock(&dev->mutex);
    
    if (copy_from_user(dev->mem + p， buf， count)) {
        ret = -EFAULT;
	} else {
        *ppos += count;
        ret = count;
        printk(KERN_INFO "written %u bytes(s) from %lu\n"， count， p);
    }
    
    mutex_unlock(&dev->mutex);
    
    return ret;
}
```

​		除了` globalmem`的读、写操作之外，如果在读、写的同时，另一个执行单元执行MEM_CLEAR IO控制命令，也会导致全局内存的混乱，因此，`globalmem_ioctl`函数也需被重写，如下代码所示。

```c
/* 增加并发控制后的globalmem设备驱动ioctl()函数 */
static long globalmem_ioctl(struct file *filp， unsigned int cmd， unsigned long arg)
{
    struct globalmem_dev *dev = filp->private_data;
    
    switch (cmd) {
        case MEM_CLEAR:
			mutex_lock(&dev->mutex);
			memset(dev->mem， 0， GLOBALMEM_SIZE);
			mutex_unlock(&dev->mutex);
            
			printk(KERN INFo "globalmem is set to zero\n");
			break;
	default
		return -EINVAL;
    }
    
	return 0;
}

```

### 总结

​		并发和竞态广泛存在，中断屏蔽、原子操作、自旋锁和互斥体都是解决并发问题的机制。中断屏蔽很少单独被使用，原子操作只能针对整数进行，因此自旋锁和互斥体应用最为广泛。

​		**自旋锁会导致死循环，锁定期间不允许阻塞，因此要求锁定的临界区小。互斥体允许临界区阻塞，可以适用于临界区大的情况。**