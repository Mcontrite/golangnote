interface + struct

make中间参数为0不会开辟内存

interface + interface + struct

结构体指针变量可以用new初始化

任意数据类型数组 dataStore []interface{}

make([]interface{},0,10)

栈模拟递归

```go
data=Pop()
if data==0{
    rst+=0
}else {
    rst+=data.(Int)
    Push(data-1)
}
```

循环队列最多存储size-1个数据，空一格节点表示满格

出队把值设为0

循环队列为空：q.front==q.rear

循环队列满：（q.rear+1)%size==q.front%size

fmt.Println(“a”>”b”)是地址的比较，意义不大。通常使用strings.Compare(“a1”,”a2”)

奇偶排序

os.Open()路径用`\\`划分