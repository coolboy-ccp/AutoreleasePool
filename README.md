# AutoreleasePool

## 关于@autoreleasepool{}的实际执行
```
void *objc = objc_autoreleasePoolPush();
...
objc_autoreleasePoolPop(objc)
```
```
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
 
void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);

```
## AutoreleasePoolPage, 双向链表
* [源码下载](https://opensource.apple.com/tarballs/objc4/)
* 结构
```
class AutoreleasePoolPage {
   magic_t const magic; //完整性校验
   id *next;
   pthread_t const thread; //当前线程
   AutoreleasePoolPage * const parent; //上一个节点
   AutoreleasePoolPage *child; //下一个节点
   uint32_t const depth;
   uint32_t hiwat
}
```
 ---------------------------------------------------------------------------------------
### 栈  ![img1](/autoreleasePoolPage.jpg)
### 入栈流程

* push()
```
static inline void *push() 
{
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
```
* autoreleaseFast
```
static inline id *autoreleaseFast(id obj)
   {
       //获取hotPage
       AutoreleasePoolPage *page = hotPage();
       //hotPage存在，且未满
       if (page && !page->full()) {
           return page->add(obj);
       } else if (page) { //hotPage存在,但已满
           return autoreleaseFullPage(obj, page);
       } else { //hotPage不存在
           return autoreleaseNoPage(obj);
       }
   }
```
* add(obj)
```
//初始化时next指向begin(一个空地址), add obj的操作将obj插入到next地址，next上移到下一个空地址
id *add(id obj)
   {
       assert(!full());
       unprotect();
       id *ret = next;  // faster than `return next-1` because of aliasing
       *next++ = obj;
       protect();
       return ret;
   }
```

* autoreleaseFullPage
```
// 循环查找，直到找到不满的page。如果page -> child == nil, new page, 新page必定不满，结束循环
// set hotPage = page
// add obj
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    // The hot page is full. 
    // Step to the next non-full page, adding a new page if necessary.
    // Then add the object to that page.
    assert(page == hotPage());
    assert(page->full()  ||  DebugPoolAllocation);

    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```

* autoreleaseNoPage
```
static __attribute__((noinline))
   id *autoreleaseNoPage(id obj)
   {
       ...
       AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
       setHotPage(page);
       ...
       return page->add(obj);
   }
```
* POOL_BOUNDARY是一个边界对象 nil,之前的源代码变量名是 POOL_SENTINEL哨兵对象,用来区别每个page即每个 AutoreleasePoolPage边界

* AutoreleasePoolPage::push()
   调用push方法会将一个POOL_BOUNDARY入栈，并且返回其存放的内存地址.
   push就是压栈的操作,先加入边界对象,然后添加person1对象,然后是person2对象...以此类推

* AutoreleasePoolPage::pop(ctxt);
   调用pop方法时传入一个POOL_BOUNDARY的内存地址，会从最后一个入栈的对象开始发送release消息，直到遇到这个POOL_BOUNDARY(因为是双向链表,所以可以向上寻找)
 ---------------------------------------------------------------------------------------
### 出栈

* pop()
```
static inline void pop(void *token) 
   {
       AutoreleasePoolPage *page;
       id *stop;

       if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
           // Popping the top-level placeholder pool.
           if (hotPage()) {
               // Pool was used. Pop its contents normally.
               // Pool pages remain allocated for re-use as usual.
               pop(coldPage()->begin());
           } else {
               // Pool was never used. Clear the placeholder.
               setHotPage(nil);
           }
           return;
       }

       page = pageForPointer(token);
       stop = (id *)token;
       if (*stop != POOL_BOUNDARY) {
           if (stop == page->begin()  &&  !page->parent) {
               // Start of coldest page may correctly not be POOL_BOUNDARY:
               // 1. top-level pool is popped, leaving the cold page in place
               // 2. an object is autoreleased with no pool
           } else {
               // Error. For bincompat purposes this is not 
               // fatal in executables built with old SDKs.
               return badPop(token);
           }
       }

       if (PrintPoolHiwat) printHiwat();

       page->releaseUntil(stop);

       // memory: delete empty children
       if (DebugPoolAllocation  &&  page->empty()) {
           // special case: delete everything during page-per-pool debugging
           AutoreleasePoolPage *parent = page->parent;
           page->kill();
           setHotPage(parent);
       } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
           // special case: delete everything for pop(top) 
           // when debugging missing autorelease pools
           page->kill();
           setHotPage(nil);
       } 
       else if (page->child) {
           // hysteresis: keep one empty child if page is more than half full
           if (page->lessThanHalfFull()) {
               page->child->kill();
           }
           else if (page->child->child) {
               page->child->child->kill();
           }
       }
   }
```
* releaseUntil
```
//循环release obj，直到next == stop
    void releaseUntil(id *stop) 
    {
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
        
        while (this->next != stop) {
            // Restart from hotPage() every time, in case -release 
            // autoreleased more objects
            AutoreleasePoolPage *page = hotPage();

            // fixme I think this `while` can be `if`, but I can't prove it
            while (page->empty()) {
                page = page->parent;
                setHotPage(page);
            }

            page->unprotect();
            id obj = *--page->next;
            memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
            page->protect();

            if (obj != POOL_BOUNDARY) {
                objc_release(obj);
            }
        }

        setHotPage(this);

#if DEBUG
        // we expect any children to be completely empty
        for (AutoreleasePoolPage *page = child; page; page = page->child) {
            assert(page->empty());
        }
#endif
    }
```

* kill
```
//从链表尾部开始向前删除
void kill() 
  {
      // Not recursive: we don't want to blow out the stack 
      // if a thread accumulates a stupendous amount of garbage
      AutoreleasePoolPage *page = this;
      while (page->child) page = page->child;

      AutoreleasePoolPage *deathptr;
      do {
          deathptr = page;
          page = page->parent;
          if (page) {
              page->unprotect();
              page->child = nil;
              page->protect();
          }
          delete deathptr;
      } while (deathptr != this);
  }
```
 ---------------------------------------------------------------------------------------
### runloop
App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。




