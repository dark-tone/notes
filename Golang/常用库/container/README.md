# list
### 说明
实际上是一个循环链表。

可用于实现stack，queue等类型。

### 主要对象
``` GO
type List struct {
	root Element
	len  int
}

type Element struct {
	next, prev *Element
	list *List
	Value interface{}
}

```
### 常用API
- func New() *List</br>   初始化
- func (l *List) Front() *Element{} </br> 返回头部元素（不删除）
- func (l *List) Back() *Element{}</br>  返回末尾元素
- func (l *List) Remove(e *Element) interface{} {}</br> 删除元素
- func (l *List) PushFront(v interface{}) *Element{}</br> 把元素压入头部
- func (l *List) PushBack(v interface{}) *Element{}</br> 把元素压入尾部
- func (e Element) Next() Element{}<br> 后一个元素
- func (e Element) Prev() Element{} <br> 前一个元素

### 示例
``` GO
// 遍历
for e := a.Front(); e != nil; e = e.Next() {
        fmt.Println(e.Value)
    }
```

# heap

# ring