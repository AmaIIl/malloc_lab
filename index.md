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

## 分离适配空闲链表
本次实验的重头戏，在显示空闲链表的基础上，通过设置多个空闲链表并且每个空闲链表都有一个对应的范围来与之呼应。一开始在做完隐式的以后尝试去做分离式的，但是效果并不理想，在看了网上很多别的师傅的思路以后自己整理了一个大体的流程图  

![image](https://user-images.githubusercontent.com/37897095/119223292-15e13280-bb2b-11eb-8b8b-b222a2cd73ac.png)

![image](https://user-images.githubusercontent.com/37897095/119223325-5b9dfb00-bb2b-11eb-81ca-64207276dc6c.png)

## mm_init函数
在隐式的基础上，增加了前驱与后继指针来组成双向链表，并且因为增加了两个指针（8字节）的缘故，一个块的最小就是16字节（头部+脚部+前驱+后继）。所以一开始先是在堆的头部存放空闲适配表，其中每一个数（0-8）都对应了一个申请范围的空闲链表，后续我们需要增加新的空闲块或进行适配时都会用到它。程序的最后依然是使用extend_heap函数申请堆空间，但是因为增加了前驱后继的缘故，函数也发生了些许变化。
```
int mm_init(void)
{
    if ((heap_listp = mem_sbrk(12*WSIZE)) == (void *)-1) return -1;
    PUT(heap_listp + (0*WSIZE) , NULL);
    PUT(heap_listp + (1*WSIZE) , NULL);
    PUT(heap_listp + (2*WSIZE) , NULL);
    PUT(heap_listp + (3*WSIZE) , NULL);
    PUT(heap_listp + (4*WSIZE) , NULL);
    PUT(heap_listp + (5*WSIZE) , NULL);
    PUT(heap_listp + (6*WSIZE) , NULL);
    PUT(heap_listp + (7*WSIZE) , NULL);
    PUT(heap_listp + (8*WSIZE) , NULL);
    PUT(heap_listp + (9*WSIZE), PACK(DSIZE, 1));
    PUT(heap_listp + (10*WSIZE), PACK(DSIZE, 1));
    PUT(heap_listp + (11*WSIZE), PACK(0, 1));
    listp = heap_listp;
    heap_listp += (10*WSIZE);

    if (extend_heap(CHUNKSIZE/WSIZE) == NULL) return -1;
    return 0;
}

```

## extend_heap
在隐式空闲链表的功能的基础上，需要对空闲块所再的空闲链表进行增加和删除的操作，其中coalesce函数依旧是分情况去合并空闲块，只是在合并之前需要将旧的空闲块在其对应的空闲链表中删除掉，再将新合并的空闲块加入到对应的空闲链表中
```
static void *extend_heap(size_t words)
{
	char *bp;
	size_t size;

	size = (words % 2) ? (words+1) * WSIZE : words * WSIZE;
	if ((long)(bp = mem_sbrk(size)) == -1) return NULL;

	PUT(HDRP(bp), PACK(size, 0));
	PUT(FTRP(bp), PACK(size, 0));
	PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1));

	PUT(PRED(bp), NULL);
	PUT(SUCC(bp), NULL);
	bp = coalesce(bp);
	bp = add_block(bp);
	return bp;
}
```

## coalesce
```
static void *coalesce(void *bp)
{
	size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
	size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
	size_t size = GET_SIZE(HDRP(bp));

	if (prev_alloc && next_alloc)
	{
		return bp;
	}
	else if (prev_alloc && !next_alloc)
	{
		size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
		delete_block(NEXT_BLKP(bp));
		PUT(HDRP(bp), PACK(size, 0));
		PUT(FTRP(bp), PACK(size, 0));
	}
	else if (!prev_alloc && next_alloc)
	{
		size += GET_SIZE(HDRP(PREV_BLKP(bp)));
		delete_block(PREV_BLKP(bp));
		PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
		PUT(FTRP(bp), PACK(size, 0));
		bp = PREV_BLKP(bp);
	}
	else
	{
		size += GET_SIZE(HDRP(NEXT_BLKP(bp))) + 
				GET_SIZE(HDRP(PREV_BLKP(bp)));
		delete_block(PREV_BLKP(bp));
		delete_block(NEXT_BLKP(bp));
		PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
		PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
		bp = PREV_BLKP(bp);
	}
	return bp;
}
```








