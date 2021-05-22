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

并且为了方便我们操作新增加的前驱后继，添加了新的宏以便我们操作
```
/*取出块的前驱指针和后继指针的地址*/
#define PRED(bp) ((char*)(bp) + WSIZE)
#define SUCC(bp) ((char*)bp)
/*取出块的前驱和后继所指向的块*/
#define PRED_BLKP(bp) (GET(PRED(bp)))
#define SUCC_BLKP(bp) (GET(SUCC(bp)))
```

## mm_init函数
在隐式的基础上，增加了前驱与后继指针来组成双向链表，并且因为增加了两个指针（8字节）的缘故，一个块的最小就是16字节（头部+脚部+前驱+后继）。所以一开始先是在堆的头部存放空闲适配表，其中每一个大小类（0-8）都对应了一个申请范围的空闲链表。并且为了方便我们后续对其进行操作，让listp指针指向大小类的头部，这样当我们想对大小类中的空闲列表进行操作时只需要用 listp + (X * WSZIE) 即可。  
程序的最后依然是使用extend_heap函数申请堆空间，但是因为增加了前驱后继的缘故，函数也发生了些许变化。
```
static char *listp;

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

## delete_block
对于旧的空闲块，当我们执行合并操作时需要将其从空闲链表中删除，这里模仿coalesce函数分情况删除
```
static void delete_block(void *bp)
{
	if (PRED_BLKP(bp) != NULL && SUCC_BLKP(bp) != NULL)
	{
		PUT(PRED(SUCC_BLKP(bp)), PRED_BLKP(bp));
		PUT(SUCC(PRED_BLKP(bp)), SUCC_BLKP(bp));
	}
	else if (PRED_BLKP(bp) != NULL && SUCC_BLKP(bp) == NULL)
	{
		PUT(SUCC(PRED_BLKP(bp)), NULL);
	}
	else if (PRED_BLKP(bp) == NULL && SUCC_BLKP(bp) != NULL)
	{
		PUT(PRED(SUCC_BLKP(bp)), NULL);
	}
}
```

## add_block && Index
首先使用Index函数来判断所属的大小类，再将其按照LIFO(后进先出)的规则放在链表的前面，这里需要注意的是当root的后继指向NULL时，我们只需要将空闲块的后继指向空，再将root与空闲块关联起来即可
```
static int Index(size_t size)
{
	if (size < 16) return -1;
	if (size <= 31) return 0;
	if (size <= 63) return 1;
	if (size <= 127) return 2;
	if (size <= 255) return 3;
	if (size <= 511) return 4;
	if (size <= 1023) return 5;
	if (size <= 2047) return 6;
	if (size <= 4095) return 7;
	if (size >= 4096) return 8;
}

static void *add_block(void *bp)
{
	size_t index = Index(GET_SIZE(HDRP(bp)));
	void *root;

	if (index != -1)
	{
		root = listp + (index*WSIZE);
		if (SUCC_BLKP(root) != NULL)
		{
			PUT(PRED(SUCC_BLKP(root)), bp);
			PUT(SUCC(bp), SUCC_BLKP(root));
		}
		else
		{
			PUT(SUCC(bp), NULL);
		}
		PUT(PRED(bp), root);
		PUT(SUCC(root), bp);
	}
	return bp;
}
```
## mm_malloc
mm_malloc函数与隐式空闲链表中所实现的功能一致，在保证字节对齐的前提下，通过首次适配（first_fit）的方法寻找合适的空闲链表，如果找到了就进行分配（place），如果没有找到就调用extend_heap函数来分配新的空闲块在进行分配，而其中的fist_fit函数与place函数与隐式空闲链表的构造相比增加了些许新的功能
```
void *mm_malloc(size_t size)
{
    char *bp;
    size_t asize;
    size_t extend_size;

    if (size == 0) return NULL;

    if (size <= DSIZE)
    	asize = 2 * DSIZE;
    else
    	asize = DSIZE * ((size + (DSIZE) + (DSIZE - 1)) / DSIZE);

    if ((bp = first_fit(asize)) != NULL)
    {
    	place(bp, asize);
    	return bp;
    }

    extend_size = MAX(asize, CHUNKSIZE);
    if ((bp = extend_heap(extend_size/WSIZE)) == NULL) return NULL;
    place(bp, asize);
    return bp;
}
```

## first_fit
首先需要判断所申请的大小所属的大小类（Index），然后保持一个原则：对这个大小类中的空闲链表进行遍历，如果有合适的空闲块则直接返回其对应指针（root），若在这个大小类中没有找到，就去下一个大小类中寻找，因为下一个大小类对应的范围一定是大于当前的大小类的，如果所有的大小类中的空闲链表都遍历了还是没有找到就返回NULL，让mm_malloc函数通过extend_heap函数分配新的空闲块进行使用。
```
static void *first_fit(size_t asize){
	int index = Index(asize);
	void *root;

	while (index <= 8)
	{
		root = listp + (index*WSIZE);
		while ((root = SUCC_BLKP(root)) != NULL)
		{
			if (GET_SIZE(HDRP(root)) >= asize && GET_ALLOC(HDRP(root)) == 0)
			{
				return root;
			}
		}
		index++;
	}
	return NULL;
}
```

## place
place函数在使用空闲块的同时(将头部脚部的使用状态从0改为1)，需要判断申请的空间大小与使用的空闲块的大小是否有剩余，如果剩余满足生成一个最小空闲块的要求，就进行分割  
而因为有了显示空闲链表的概念，我们在使用一个空闲块的时候，需要将其从对应的空闲链表中删除，并在分割出新的空闲块时将其插入到对应的空闲链表中
```
static void place(void *bp, size_t asize)
{
	size_t csize = GET_SIZE(HDRP(bp));
	delete_block(bp);
	if ((csize - asize) >= (2*DSIZE))
	{
		PUT(HDRP(bp), PACK(asize, 1));
		PUT(FTRP(bp), PACK(asize, 1));
		bp = NEXT_BLKP(bp);
		PUT(HDRP(bp), PACK(csize - asize, 0));
		PUT(FTRP(bp), PACK(csize - asize, 0));
		add_block(bp);
	}
	else
	{
		PUT(HDRP(bp), PACK(csize, 1));
		PUT(FTRP(bp), PACK(csize, 1));
	}
}
```

## mm_free
mm_free函数在释放产生新的空闲块的同时，需要对新产生的空闲块进行判断，判断其是否需要合并，在合并完成后将其添加到对应的空闲链表中
```
void mm_free(void *ptr)
{
	size_t size = GET_SIZE(HDRP(ptr));

	PUT(HDRP(ptr), PACK(size, 0));
	PUT(FTRP(ptr), PACK(size, 0));
	ptr = coalesce(ptr);
	add_block(ptr);
}

```

最后的得分
![image](https://user-images.githubusercontent.com/37897095/119224571-94d96980-bb31-11eb-914a-6ab2620628d9.png)
## 小结
csapp的malloc_lab就告一段落了，这个实验给我最大的感觉就是有着许多的提升空间，好比说脚部可以只在空闲块中使用，以及realloc可以为其再增加一些功能提升效率等等，不得不说一套实验做下来收获还是蛮大的，好了菜鸡要去继续入门pwn了，886。








