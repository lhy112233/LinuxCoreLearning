# mmap

## mmap system call
```c
#include "arch/x86/kernel/sys_x86_64.c"
SYSCALL_DEFINE6(mmap, 
                unsigned long, addr, 
                unsigned long, len,
		        unsigned long, prot, 
                unsigned long, flags,
		        unsigned long, fd, 
                unsigned long, off)
{
    // 偏移量off与PAGE_MASK按位比对检查是否对齐
	if (off & ~PAGE_MASK)
		return -EINVAL;

    // 
	return ksys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
}
```  
- `unsigned long addr`: 待映射的虚拟内存区域在进程虚拟内存空间中的起始地址（虚拟内存地址），通常设置成 NULL，意思就是完全交由内核来帮我们决定虚拟映射区的起始地址（要按照 PAGE_SIZE（4K） 对齐）  
- `unsigned long len`：待申请映射的内存区域的大小，如果是匿名映射，则是要映射的匿名物理内存有多大，如果是文件映射，则是要映射的文件区域有多大（要按照 PAGE_SIZE（4K） 对齐）。  
- `unsigned long prot`: 映射区域的保护模式。有 PROT_READ、PROT_WRITE、PROT_EXEC等。
- `unsigned long flags`: 标志位，可以控制映射区域的特性。常见的有 MAP_SHARED 和 MAP_PRIVATE 等。
- `unsigned long fd`: 文件描述符，用于指定映射的文件 (由 open( ) 函数返回)。
- `unsigned long off`: 映射的起始位置，表示被映射对象 (即文件) 从那里开始对映，通常设置为 0，该值应该为大小为PAGE_SIZE（4K）的整数倍。  

```c
    /*
        类似网络子网掩码，对`PAGE_MASK`进行`取反`和`与`操作后判断是否还存在非0情况，
        如有则是不对齐。
    */

	if (off & ~PAGE_MASK)
		return -EINVAL;    
```  
```c
    //要从字节偏移转换为页偏移
    //PAGE_SHIFT是页大小的log2; 
   /*
    eg:
        PAGE_SHIFT = [4k = 4096] -> log2(4k) = 12;
        off = 10`000`000`000`000 = 8096
        off >> PAGE_SHIFT = 10 = 2;
        即由字节偏移`8096`转换为页偏移`2`
   */
   off >> PAGE_SHIFT
```



---

## ksys_mmap_pgoff
```c
#include "mm/mmap.c"
unsigned long ksys_mmap_pgoff(  unsigned long addr, 
                                unsigned long len,
			                    unsigned long prot, 
                                unsigned long flags,
			                    unsigned long fd, 
                                unsigned long pgoff)
{
	struct file *file = NULL;
	unsigned long retval;

	if (!(flags & MAP_ANONYMOUS)) {
		audit_mmap_fd(fd, flags);
		file = fget(fd);
		if (!file)
			return -EBADF;
		if (is_file_hugepages(file)) {
            // 令len向上对齐到 `huge page` 大小. 如果不对齐，底层 hugepage 映射没法做 
			len = ALIGN(len, huge_page_size(hstate_file(file)));
		} else if (unlikely(flags & MAP_HUGETLB)) {
			retval = -EINVAL;
			goto out_fput;
		}
	} else if (flags & MAP_HUGETLB) {
		struct hstate *hs;

		hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
		if (!hs)
			return -EINVAL;

		len = ALIGN(len, huge_page_size(hs));
		/*
		 * VM_NORESERVE is used because the reservations will be
		 * taken when vm_ops->mmap() is called
		 */
		file = hugetlb_file_setup(HUGETLB_ANON_FILE, len,
				mk_vma_flags(VMA_NORESERVE_BIT),
				HUGETLB_ANONHUGE_INODE,
				(flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
		if (IS_ERR(file))
			return PTR_ERR(file);
	}

	retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
out_fput:
	if (file)
		fput(file);
	return retval;
}
```  
- 参数几乎和上面（SYSCALL_DEFINE6）的一致，就是`pgoff`变为了页偏移 (`off >> PAGE_SHIFT`)
- `unsigned long addr`: 期待的起始地址，一般使用`NULL`就行，几乎没有用
- `unsigned long len`:  待申请映射的内存区域的大小，如果是匿名映射，则是要映射的匿名物理内存有多大，如果是文件映射，则是要映射的文件区域有多大（要按照 PAGE_SIZE（4K） 对齐）
- `unsigned long prot`:  映射区域的保护模式。有 PROT_READ、PROT_WRITE、PROT_EXEC等。
- `unsigned long flags`:  标志位，可以控制映射区域的特性。常见的有 MAP_SHARED 和 MAP_PRIVATE 等
- `unsigned long fd`:  文件描述符，用于指定映射的文件 (由 open( ) 函数返回)
- `unsigned long pgoff`:  映射的页数量


```c
    // 判断`不是匿名映射`的分支
    if (!(flags & MAP_ANONYMOUS))
    /*
        flags & MAP_ANONYMOUS ： 如果 `flag` 存在匿名标志则与 `MAP_ANONYMOUS` 进行与操作后则非`0`进而可以隐式转换为true的`bool`对象. 进而判断为`匿名映射`
        !(flags & MAP_ANONYMOUS): 进行 `非` 操作后变为判断`非匿名操作`.
    */
```  

### void audit_mmap_fd(int fd, int flags)
```c
/*
这是给审计子系统用的。
内核可以记录这次 mmap 用了哪个 fd、什么 flags

你可以把它理解成“日志”，不影响核心控制流。
*/

#ifdef CONFIG_AUDITSYSCALL

static inline void audit_mmap_fd(int fd, int flags)
{
	if (unlikely(!audit_dummy_context()))
		__audit_mmap_fd(fd, flags);
}

#else /* CONFIG_AUDITSYSCALL */

static inline void audit_mmap_fd(int fd, int flags){ }

#endif /* CONFIG_AUDITSYSCALL */
```  

### file *fget(unsigned int fd)
```c
// 根据传入的文件描述符 fd，获取对应的 struct file 内核对象指针，并增加其引用计数，从而安全地使用该文件对象

// 若fd无效则返回 NULL

extern struct file *fget(unsigned int fd);
```  

### bool is_file_hugepages(const struct file *file)
```c
/*
它的核心作用是在内核处理文件映射时，识别出哪些文件使用的是巨型页。由于巨型页在内存管理、地址对齐和资源统计方面与普通 4KB 页面有显著不同，内核需要根据这个判断结果来采取不同的处理路径
*/
#include "include/linux/hugetlb.h"
#ifdef CONFIG_HUGETLBFS

static inline bool is_file_hugepages(const struct file *file)
{
	return file->f_op->fop_flags & FOP_HUGE_PAGES;
}

#else /* !CONFIG_HUGETLBFS */

#define is_file_hugepages(file)			false

#endif /* !CONFIG_HUGETLBFS */
```  

### ALIGN(x, a)
```c
/*
    ALIGN(x, a) 是内核中进行向上取整内存对齐的基础工具宏
*/
#include "include/vdso/align.h"
#define ALIGN(x, a)		__ALIGN_KERNEL((x), (a))

```  

### unsigned long huge_page_size(const struct hstate *h)
```c
/*
获取一个指定的 HugeTLB 页面大小（hstate）所对应的页面大小（以字节为单位）
*/
#include "include/linux/hugetlb.h"
#ifdef CONFIG_HUGETLB_PAGE

// h->order：该 HugeTLB 页面大小对应的“阶”（order）。一个页面的大小等于 PAGE_SIZE * 2^order
static inline unsigned long huge_page_size(const struct hstate *h)
{
	return (unsigned long)PAGE_SIZE << h->order;
}

#else	/* CONFIG_HUGETLB_PAGE */

static inline unsigned long huge_page_size(struct hstate *h)
{
	return PAGE_SIZE;
}

#endif	/* CONFIG_HUGETLB_PAGE */
```  

* Tips: `huge_page_size(h) == (1UL << huge_page_shift(h)) == (PAGE_SIZE << huge_page_order(h))`

### hstate *hstate_file(struct file *f)
```c
/*
    1. 通过 file_inode(f) 从 struct file 中获取其对应的 struct inode (索引节点)
    2. 调用 hstate_inode() 函数从 inode 中提取出 hstate 指针

    struct hstate. 是一个用于描述和管理一种特定大小的大页（Huge Page）页面池的内核数据结构
*/

#include "include/linux/hugetlb.h"
#ifdef CONFIG_HUGETLB_PAGE

static inline struct hstate *hstate_file(struct file *f)
{
	return hstate_inode(file_inode(f));
}

#else	/* CONFIG_HUGETLB_PAGE */

static inline struct hstate *hstate_file(struct file *f)
{
	return NULL;
}

#endif	/* CONFIG_HUGETLB_PAGE */
```  

### else if (unlikely(flags & MAP_HUGETLB))
```c
/*
    判断 `flag` 是否有 `MAP_HUGETLB` 标志
    因为 MAP_HUGETLB 不是“随便给任意文件套一个大页模式”，它要求底层对象本身支持 hugetlb 语义
*/
else if (unlikely(flags & MAP_HUGETLB))
```  

### else if (flags & MAP_HUGETLB)
```c
/*
    判断 `flags` 是否存在 `MAP_HUGETLB` 标志.
    通过检查 MAP_HUGETLB 标志来识别用户程序对巨页的请求，并引导到 HugeTLB 子系统的专用处理逻辑，从而绕过标准的 4KB 页分配路径，实现大页映射以减少 TLB 缺失、提升大内存访问性能
*/
else if (flags & MAP_HUGETLB)
```  

### hstate *hstate_sizelog(int page_size_log)
```c
/*
    根据输入的页面大小对数（即以2为底的指数），返回对应的HugeTLB页面状态描述符 struct hstate
    Linux 内核中，每种支持的巨型页大小（如 2MB、1GB）在系统启动或模块加载时只会被创建一次，然后全局复用
    系统内部维持着一个pool，调用函数后会在池中寻找合适的块分配给调用者
    本质是获取一个容器，待往里面填充合适的数据(等待后续分配内存后注入分配内存的信息)
*/
#include "include/linux/hugetlb.h"
#ifdef CONFIG_HUGETLB_PAGE

static inline struct hstate *hstate_sizelog(int page_size_log)
{
	if (!page_size_log)
		return &default_hstate;

	if (page_size_log < BITS_PER_LONG)
		return size_to_hstate(1UL << page_size_log);

	return NULL;
}

#else	/* CONFIG_HUGETLB_PAGE */

static inline struct hstate *hstate_sizelog(int page_size_log)
{
	return NULL;
}

#endif	/* CONFIG_HUGETLB_PAGE */
```

### hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
```c
/*
    从 flags 的高位字段中提取出用户指定的巨型页大小的对数（如 21 表示 2MB，30 表示 1GB），然后返回对应的 hstate 对象。
	此处只是为了查找系统pool中是否存在合适的`hstate`，并不会直接占有
*/
hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);

/*
    flags >> MAP_HUGE_SHIFT: 截断无用信息，将巨型页大小字段右移到最低位
    ... & MAP_HUGE_MASK: 清除无关位，只保留大小编码值
    hstate_sizelog(...): 将对数转换为对应的 struct hstate 指针(类似 `hs = malloc(2^n)`; )
*/
```  

### file *hugetlb_file_setup(const char *name, size_t size, vma_flags_t acct, int creat_flags, int page_size_log)
```c
/*
	创建并返回一个基于 hugetlbfs 的“匿名文件”，用于支持 mmap 映射大页内存
	只有在 huge page + ANONYMOUS 时才会去分配一个透明文件进行映射， 其并不会落盘到磁盘上

	根据 size 计算需要多少 huge pages
	关联 inode / address_space
	标记 VM_ACCOUNT / VM_NORESERVE 等属性（通过 acct）
*/

/*
| 参数              | 含义                        |
| --------------- | ------------------------- |
| `name`          | 文件名（调试/标识用）               |
| `size`          | 映射大小(总映射大小)                      |
| `acct`          | VM 标志（是否计入内存、overcommit等） |
| `creat_flags`   | 创建标志（类似 O_CREAT 行为）       |
| `page_size_log` | huge page 大小的 log2 (单个huge page大小)   |

eg: mmap(NULL, 5MB, ..., MAP_HUGETLB | MAP_HUGE_2MB)
*/
#include "include/linux/hugetlb.h"

#ifdef CONFIG_HUGETLBFS

struct file *hugetlb_file_setup(const char *name, size_t size, vma_flags_t acct,
				int creat_flags, int page_size_log);

#else /* !CONFIG_HUGETLBFS */

static inline struct file *
hugetlb_file_setup(const char *name, size_t size, vma_flags_t acctflag,
		int creat_flags, int page_size_log)
{
	return ERR_PTR(-ENOSYS);
}

#endif /* !CONFIG_HUGETLBFS */
```

### long __must_check IS_ERR(const void *ptr)
```c
#include "tools/virtio/linux/err.h"
static inline long __must_check IS_ERR(const void *ptr)
{
	return IS_ERR_VALUE((unsigned long)ptr);
	// #define IS_ERR_VALUE(x) unlikely((x) >= (unsigned long)-MAX_ERRNO)
}
```  

### long __must_check PTR_ERR(const void *ptr)
```c
#include "tools/virtio/linux/err.h"
static inline long __must_check PTR_ERR(const void *ptr)
{
	return (long) ptr;
}
```  

### void fput(struct file *file)
```c
/*
	减少 struct file 的引用计数（reference count），并在计数归零时触发文件释放流程。
*/
#include "fs/file_table.c"
void fput(struct file *file)
{
	if (unlikely(file_ref_put(&file->f_ref)))
		__fput_deferred(file);
}
```  

---
---

## unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr, unsigned long len, unsigned long prot, unsigned long flag, unsigned long pgoff)
```c
/*
| 参数              | 含义                     |
| --------------- | ------------------------- |
| `file`          | 映射的文件。匿名映射时可能是 NULL |
| `addr`          | 用户期望映射到的起始地址 |
| `len`           | 映射长度 |
| `prot`   		  | 页权限，比如 PROT_READ/WRITE/EXEC |
| `flag`   		  | 映射标志，比如 MAP_PRIVATE、MAP_SHARED、MAP_FIXED |
| `pgoff`  		  | 页为单位的文件偏移，不是字节偏移 |


*/
#include "mm/util.c"
unsigned long vm_mmap_pgoff(struct file *file, 
							unsigned long addr,
							unsigned long len, 
							unsigned long prot, 
							unsigned long flag, 
							unsigned long pgoff)
{
	loff_t off = (loff_t)pgoff << PAGE_SHIFT;
	unsigned long ret;
	struct mm_struct *mm = current->mm;
	unsigned long populate;
	LIST_HEAD(uf);

	ret = security_mmap_file(file, prot, flag);
	if (!ret)
		ret = fsnotify_mmap_perm(file, prot, off, len);
	if (!ret) {
		if (mmap_write_lock_killable(mm))
			return -EINTR;
		ret = do_mmap(file, addr, len, prot, flag, 0, pgoff, &populate,
			      &uf);
		mmap_write_unlock(mm);
		userfaultfd_unmap_complete(mm, &uf);
		if (populate)
			mm_populate(ret, populate);
	}
	return ret;
}

/*
	// 将页偏移转换会字节偏移
	loff_t off = (loff_t)pgoff << PAGE_SHIFT;

	// `current`: 一个宏，表示“当前正在运行的任务（线程）的 task_struct 指针”
	// `current`: #define current get_current()
	// `struct mm_struct`: mm_struct 里面管理当前进程整个用户虚拟地址空间
	struct mm_struct *mm = current->mm;

	// 原地生成一个名为 uf 的双向链表，用于收集和 userfaultfd 相关的 unmap 事件，供映射修改完成后统一处理
	// 等同于 list_head uf{.next = std::addressof(uf), .prev = std::addressof(uf)};
	LIST_HEAD(uf);

	// 这是 LSM 安全钩子，让安全模块先决定这次映射是否允许
	// 返回 `0` 表示通过
	ret = security_mmap_file(file, prot, flag);

	// 如果安全模块没拒绝，再做文件系统通知/权限检查。
	// 让 inotify/fanotify 一类机制感知 mmap 权限事件
	// 让相关监控/审计系统介入
	ret = fsnotify_mmap_perm(file, prot, off, len);

	// 以写模式获取进程地址空间的 mmap 锁，支持被致命信号中断
	// 如果在等待锁时收到了致命信号，可以中断等待并返回。
	if (mmap_write_lock_killable(mm))

	//
	do_mmap(file, addr, len, prot, flag, 0, pgoff, &populate, &uf);

	// 释放进程地址空间的写锁
	mmap_write_unlock(mm);

	// 这个是对 `do_mmap()` 期间累计的 userfaultfd unmap 事件做统一完成处理
	// `do_mmap()` 里面只负责把“哪些范围发生了 unmap/替换”记下来，真正通知 userfaultfd 监听方在这里完成
	userfaultfd_unmap_complete(mm, &uf);

	// 如果 do_mmap() 要求预填充，就调用 mm_populate()。
	mm_populate(ret, populate);
*/
```  

### current
```c
#include "arch/xtensa/include/asm/current.h"

#define current get_current()

static __always_inline struct task_struct *get_current(void)
{
	return current_thread_info()->task;
}

/* how to get the thread information struct from C */
static __always_inline struct thread_info *current_thread_info(void)
{
	struct thread_info *ti;
	 __asm__("extui %0, a1, 0, "__stringify(CURRENT_SHIFT)"\n\t"
	         "xor %0, a1, %0" : "=&r" (ti) : );
	return ti;
}
```

### LIST_HEAD(name)
```c
/*
	原地定义一个双向链表出来
*/
#include "tools/include/linux/list.h"

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

#define LIST_HEAD_INIT(name) { &(name), &(name) }

struct list_head {
	struct list_head *next;
	struct list_head *prev;
};
```

### int security_mmap_file(struct file *file, unsigned long prot, unsigned long flags)
```c
/*
	file:   要映射的文件对象指针。如果是匿名映射（MAP_ANONYMOUS），此参数为 NULL。
	prot:   映射的内存保护标志，是 PROT_READ、PROT_WRITE、PROT_EXEC 等宏的组合。
           （用户态传入的 PROT_ 系列宏，在此处仍为内核态表示）
	flags:  映射的类型标志，例如 MAP_PRIVATE、MAP_SHARED、MAP_FIXED 等。

	return value: 0表示通过, 非0表示拒绝.

	在真正分配虚拟内存区域（VMA）之前，由注册的 LSM（如 SELinux、AppArmor、Smack 等）来决定是否允许此次映射操作。
*/
#include "include/linux/security.h"
static inline int security_mmap_file(struct file *file, unsigned long prot,
				     unsigned long flags)
{
	return 0;
}
```  

### int fsnotify_mmap_perm(struct file *file, int prot, const loff_t off, size_t len)  
```c
/*
	fsnotify_mmap_perm - 在 mmap 系统调用时检查文件映射权限
 	file:   要被映射的文件对象指针（不能为 NULL，因为匿名映射不走此路径）
 	prot:   内存保护标志，是 PROT_READ、PROT_WRITE、PROT_EXEC 等宏的组合
 	off:    文件映射的起始偏移量（字节偏移，loff_t 类型，支持大文件）
 	len:    映射的长度（字节数）

 * 核心作用：
 *   1. 允许 fanotify 监控程序（如防病毒软件）在文件内容被映射到内存前
 *      对文件进行扫描或检查。
 *   2. 如果监控程序判定文件内容有问题（如包含恶意代码），
 *      可以在映射发生前拒绝该操作。
 *   3. 这是 "pre-content" 通知机制的一部分——在文件内容被访问之前
 *      通知监控程序，监控程序可以决定是否允许访问。
*/
#include "include/linux/fsnotify.h"
static inline int fsnotify_mmap_perm(struct file *file, int prot,
				     const loff_t off, size_t len)
{
	return 0;
}
```  

### int mmap_write_lock_killable(struct mm_struct *mm)
```c
/*
	以写模式获取进程地址空间的 mmap 锁，支持被致命信号中断

    mmap_write_lock()          		- 不可中断，进程会一直阻塞直到获取锁
    mmap_write_lock_killable() 		- 可被致命信号 (SIGKILL) 中断 ← 本函数
    mmap_write_lock_interruptible() - 可被任何信号中断 (较少使用)
*/
#include "include/linux/mmap_lock.h"
static inline int mmap_write_lock_killable(struct mm_struct *mm)
{
	int ret;

	__mmap_lock_trace_start_locking(mm, true);
	ret = down_write_killable(&mm->mmap_lock);
	if (!ret)
		mm_lock_seqcount_begin(mm);
	__mmap_lock_trace_acquire_returned(mm, true, ret == 0);
	return ret;
}
```  

### int mmap_write_unlock()
```c
/*
	释放进程地址空间的写锁
*/
#include "include/linux/mmap_lock.h"
static inline void mmap_write_unlock(struct mm_struct *mm)
{
	__mmap_lock_trace_released(mm, true);
	vma_end_write_all(mm);
	up_write(&mm->mmap_lock);
}
```  
### void userfaultfd_unmap_complete(struct mm_struct *mm, struct list_head *uf)
```c
/*

*/
#include "include/linux/userfaultfd_k.h"
static inline void userfaultfd_unmap_complete(struct mm_struct *mm,
					      struct list_head *uf)
{
}
```  

### 
```c

```








