# malloc lab实验记录

本次实验的要求是实现一个动态分配器，实现mm_init、mm_malloc、mm_free和mm_realloc函数的相应功能，其中本书的9.9.12小节中实现一个简单的分配器更是为我们提供了一个模型  
动态分配器实现所需的方法
```
隐式空闲链表
显式空闲链表
分离空闲链表
```
匹配合适空闲块的方法
```
首次适配
下一次适配
最佳适配
```
书中在编写简单的分配器时使用了许多宏来帮助我们访问和遍历空闲链表  
```
#define WSIZE		4
#define DSIZE		8
#define CHUNKSIZE	(1<<12)
#define MAX(x, y)	((x) > (y)? (x) : (y))

/*将大小和已分配位相结合*/
#define PACK(size, alloc)	((size)|(alloc))

/*将p所指向的地址中的内容读取出来*/
#define GET(p)		(*(unsigned int *)(p))
/*将val放入p所指向的地址中*/
#define PUT(p, val)	(*(unsigned int *)(p)=(val))
/*获取p所指向地址处的头部或者脚部的大小和已分配位*/
#define GET_SIZE(p)		(GET(p) & ~0x7)
#define GET_ALLOC(p)	(GET(p) & 0x1)
/*分别表示指向这个块的头部和脚部*/
#define HDRP(bp)	((char *)(bp) - WSIZE)
#define FTRP(bp)	((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)	
/*获取当前块的下一个块和前一个块的头指针*/
#define NEXT_BLKP(bp)	((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))
#define PREV_BLKP(bp)	((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE)))
```
## 隐式空闲链表
隐式空闲链表的代码书上已经给出，但是第一次测试分数时报错realloc函数没有将旧的data赋给新生成的块  
这里用的方法是不考虑前后是否有空闲块以及是否需要进行合并，只是对新要求的size和旧的size进行比对
```
void *mm_realloc(void *ptr, size_t size)
{
    size_t oldsize;
    void *newptr;

    if(size == 0) {
        mm_free(ptr);
        return 0;
    }
    if(ptr == NULL) {
        return mm_malloc(size);
    }

    if(!(newptr = mm_malloc(size))) {
        return 0;
    }

    oldsize = GET_SIZE(HDRP(ptr));

    if(size < oldsize) 
    	oldsize = size;
    
    memcpy(newptr, ptr, oldsize);
    mm_free(ptr);

    return newptr;
}
```
隐式空闲链表的测试结果

![image](https://user-images.githubusercontent.com/37897095/118917459-850c2a80-b963-11eb-9183-8d7bfca16b6c.png)

## 显示空闲链表
显示空闲链表的构造码了半天，结果一直用的节点不知道为啥突然不好使了，等能连上发现写的全无了...代码也已经改成分离适配的了（草）  
就在隐式的基础上，块的构造时多加两个指针指向前驱和后继，然后在进行合并、创建等操作时相应的对空闲链表进行对应的断开和连接  
其中新加的宏定义方便我们对prev和next指针进行操作
```
#define PREV_LINKNODE_RP(bp) ((char *)(bp))
#define NEXT_LINKNODE_RP(bp) ((char *)(bp) + WSIZE)
```
然后那个...traces的截图也无了，记得是在80分左右，因为realloc函数用的还是最原始的那个版本的没有进行条件判断，所以分数是低了点  
## 分离适配空闲链表
但是问题不大，我们需要实现的是分离空闲链表，基本的思路是在heap的头部放置大小类并用序言块进行隔离，对于我们每次malloc请求的size与大小类进行匹配找到对应的范围，再在其中进行首次匹配（因为其匹配是在类所指定的范围内进行，所以基本等同于最佳匹配），并找到合适的块然后进行分割，将剩余的插入合适的空闲链表中，若遍历了所有大小类依然没有找到就向内存申请更大的空间。
### mm_init




