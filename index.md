# malloc lab实验记录
在学习了csapp的第九章以后对于c中的堆管理器有了新的认识和了解  
本次实验的要求是实现一个动态分配器，实现mm_init、mm_malloc、mm_free和mm_realloc函数的相应功能，其中本书的9.9.12小节中实现一个简单的分配器更是为我们提供了一个模型  
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

