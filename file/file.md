# Golang 读写文件

在Go中（就此而言，大多数底层语言和某些动态语言（如Node））返回字节流。 不将所有内容自动转换为字符串的好处是，其中之一是避免昂贵的字符串分配，这会增加GC压力。

为了使本文更加简单，我将使用string(arrayOfBytes)将bytes数组转换为字符串。 但是，在发布生产代码时，不应将其作为一般建议。

# 读文件1

## 1 读取整个文件到内存

首先，标准库提供了多种功能和实用程序来读取文件数据。我们将从os软件包中提供的基本情况开始。这意味着两个先决条件：

-  该文件必须容纳在内存中
-  我们需要预先知道文件的大小，以便实例化一个足以容纳它的缓冲区。

有了os.File对象的句柄，我们可以查询大小并实例化一个字节列表。

```go
func main() {
	file, err := os.Open("filetoread.txt")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer file.Close()

	fileinfo, err := file.Stat()
	if err != nil {
		fmt.Println(err)
		return
	}

	filesize := fileinfo.Size()
	buffer := make([]byte, filesize)

	bytesread, err := file.Read(buffer)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("bytes read: ", bytesread)
	fmt.Println("bytestream to string: ", string(buffer))
}

```

## 2 以块的形式读取文件

虽然大多数情况下可以一次读取文件，但有时我们还是想使用一种更加节省内存的方法。例如，以某种大小的块读取文件，处理它们，并重复直到结束。在下面的示例中，使用的缓冲区大小为100字节。

```go
const BufferSize = 100

func main() {
	file, err := os.Open("filetoread.txt")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer file.Close()
	buffer := make([]byte, BufferSize)
	for {
		bytesread, err := file.Read(buffer)
		if err != nil {
			if err != io.EOF {
				fmt.Println(err)
			}
			break
		}
		fmt.Println("bytes read: ", bytesread)
		fmt.Println("bytestream to string: ", string(buffer[:bytesread]))
	}
}
```

与完全读取文件相比，主要区别在于：

-  读取直到获得EOF标记，因此我们为err == io.EOF添加了特定检查
-  我们定义了缓冲区的大小，因此我们可以控制所需的“块”大小。 如果操作系统正确地将正在读取的文件缓存起来，则可以在正确使用时提高性能。
-  如果文件大小不是缓冲区大小的整数倍，则最后一次迭代将仅将剩余字节数添加到缓冲区中，因此调用buffer [：bytesread]。 在正常情况下，bytesread将与缓冲区大小相同。

对于循环的每次迭代，都会更新内部文件指针。 下次读取时，将返回从文件指针偏移开始直到缓冲区大小的数据。 该指针不是语言的构造，而是操作系统之一。 在Linux上，此指针是要创建的文件描述符的属性。 所有的read / Read调用（分别在Ruby / Go中）在内部都转换为系统调用并发送到内核，并且内核管理此指针。

## 3 并发读取文件块

如果我们想加快对上述块的处理该怎么办？

一种方法是使用多个go例程。与串行读取块相比，我们需要做的另一项工作是我们需要知道每个例程的偏移量。注意当目标缓冲区的大小大于剩余的字节数时，ReadAt的行为与Read的行为略有不同。

这里并没有限制goroutine的数量，它仅由缓冲区大小来定义。实际上，此数字可能会有上限。

```GO
const BufferSize = 100

type chunk struct {
	bufsize int
	offset  int64
}

func main() {
	file, err := os.Open("filetoread.txt")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer file.Close()
	fileinfo, err := file.Stat()
	if err != nil {
		fmt.Println(err)
		return
	}
	filesize := int(fileinfo.Size())
	concurrency := filesize / BufferSize
	chunksizes := make([]chunk, concurrency)
	for i := 0; i < concurrency; i++ {
		chunksizes[i].bufsize = BufferSize
		chunksizes[i].offset = int64(BufferSize * i)
	}
	if remainder := filesize % BufferSize; remainder != 0 {
		c := chunk{bufsize: remainder, offset: int64(concurrency * BufferSize)}
		concurrency++
		chunksizes = append(chunksizes, c)
	}
	var wg sync.WaitGroup
	wg.Add(concurrency)
	for i := 0; i < concurrency; i++ {
		go func(chunksizes []chunk, i int) {
			defer wg.Done()
			chunk := chunksizes[i]
			buffer := make([]byte, chunk.bufsize)
			bytesread, err := file.ReadAt(buffer, chunk.offset)
			if err != nil {
				fmt.Println(err)
				return
			}
			fmt.Println("bytes read, string(bytestream): ", bytesread)
			fmt.Println("bytestream to string: ", string(buffer))
		}(chunksizes, i)
	}
	wg.Wait()
}
```

与以前的任何方法相比，这种方法要多得多：

-  我正在尝试创建特定数量的Go例程，具体取决于文件大小和缓冲区大小（在本例中为100）。
-  我们需要一种方法来确保我们正在“等待”所有执行例程。 在此示例中，我使用的是wait group。
-  在每个例程结束的时候，从内部发出信号，而不是break for循环。因为我们延时调用了wg.Done(),所以在每个例程返回的时候才调用它。

注意：始终检查返回的字节数，并重新分配输出缓冲区。

使用Read()读取文件可以走很长一段路，但是有时您需要更多的便利。Ruby中经常使用的是IO函数，例如each_line,each_char, each_codepoint 等等.通过使用Scanner类型以及bufio软件包中的关联函数，我们可以实现类似的目的。

bufio.Scanner类型实现带有“ split”功能的函数，并基于该功能前进指针。例如，对于每个迭代，内置的bufio.ScanLines拆分函数都会使指针前进，直到下一个换行符为止.

在每个步骤中，该类型还公开用于获取开始位置和结束位置之间的字节数组/字符串的方法。

```go
const BufferSize = 100

type chunk struct {
	bufsize int
	offset  int64
}

func main() {
	file, err := os.Open("filetoread.txt")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer file.Close()
	scanner := bufio.NewScanner(file)
	scanner.Split(bufio.ScanLines)
	for {
		read := scanner.Scan()
		if !read {
			break
		}
		fmt.Println("read byte array: ", scanner.Bytes())
		fmt.Println("read string: ", scanner.Text())
	}
}
```

因此，要以这种方式逐行读取整个文件，可以使用如下所示的内容：

```go
func main() {
	file, err := os.Open("filetoread.txt")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer file.Close()
	scanner := bufio.NewScanner(file)
	scanner.Split(bufio.ScanLines)
	var lines []string
	for scanner.Scan() {
		lines = append(lines, scanner.Text())
	}
	fmt.Println("read lines:")
	for _, line := range lines {
		fmt.Println(line)
	}
}
```

## 4 逐字扫描

bufio软件包包含基本的预定义拆分功能：

-  ScanLines (默认)
-  ScanWords
-  ScanRunes(对于遍历UTF-8代码点（而不是字节）非常有用)
-  ScanBytes

因此，要读取文件并在文件中创建单词列表，可以使用如下所示的内容：

```go
func main() {
	file, err := os.Open("filetoread.txt")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer file.Close()
	scanner := bufio.NewScanner(file)
	scanner.Split(bufio.ScanWords)
	var words []string
	for scanner.Scan() {
		words = append(words, scanner.Text())
	}
	fmt.Println("word list:")
	for _, word := range words {
		fmt.Println(word)
	}
}
```

ScanBytes拆分函数将提供与早期Read()示例相同的输出。 两者之间的主要区别是在扫描程序中，每次需要附加到字节/字符串数组时，动态分配问题。 可以通过诸如将缓冲区预初始化为特定长度的技术来避免这种情况，并且只有在达到前一个限制时才增加大小。 使用与上述相同的示例：

```go
func main() {
	file, err := os.Open("filetoread.txt")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer file.Close()
	scanner := bufio.NewScanner(file)
	scanner.Split(bufio.ScanWords)
	bufferSize := 50
	words := make([]string, bufferSize)
	pos := 0
	for scanner.Scan() {
		if err := scanner.Err(); err != nil {
			fmt.Println(err)
			break
		}
		words[pos] = scanner.Text()
		pos++
		if pos >= len(words) {
			newbuf := make([]string, bufferSize)
			words = append(words, newbuf...)
		}
	}
	fmt.Println("word list:")
	for _, word := range words[:pos] {
		fmt.Println(word)
	}
}
```

因此，我们最终要进行的切片“增长”操作要少得多，但最终可能要根据缓冲区大小和文件中的单词数在结尾处留出一些空插槽，这是一个折衷方案。

## 5 将长字符串拆分为单词

bufio.NewScanner使用满足io.Reader接口的类型作为参数，这意味着它将与定义了Read方法的任何类型一起使用。
标准库中返回reader类型的string实用程序方法之一是strings.NewReader函数。当从字符串中读取单词时，我们可以将两者结合起来：

```go
func main() {
	longstring := "This is a very long string. Not."
	var words []string
	scanner := bufio.NewScanner(strings.NewReader(longstring))
	scanner.Split(bufio.ScanWords)
	for scanner.Scan() {
		words = append(words, scanner.Text())
	}
	fmt.Println("word list:")
	for _, word := range words {
		fmt.Println(word)
	}
}
```

## 6 扫描以逗号分隔的字符串

手动解析CSV文件/字符串通过基本的file.Read()或者Scanner类型是复杂的。因为根据拆分功能bufio.ScanWords，“单词”被定义为一串由unicode空间界定的符文。读取各个符文并跟踪缓冲区的大小和位置（例如在词法分析中所做的工作）需要大量的的工作和操作。

但这可以避免。 我们可以定义一个新的拆分函数，该函数读取字符直到读者遇到逗号，然后在调用Text（）或Bytes（）时返回该块。bufio.SplitFunc函数的函数[签名](http://www.zzvips.com/qq/qm/)如下所示：

```go
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)
```

为简单起见，我展示了一个读取字符串而不是文件的示例。 使用上述签名的CSV字符串的简单阅读器可以是：

```go
func main() {
	csvstring := "name, age, occupation"
	ScanCSV := func(data []byte, atEOF bool) (advance int, token []byte, err error) {
		commaidx := bytes.IndexByte(data, ',')
		if commaidx > 0 {
			buffer := data[:commaidx]
			return commaidx + 1, bytes.TrimSpace(buffer), nil
		}
		if atEOF {
			if len(data) > 0 {
				return len(data), bytes.TrimSpace(data), nil
			}
		}
		return 0, nil, nil
	}
	scanner := bufio.NewScanner(strings.NewReader(csvstring))
	scanner.Split(ScanCSV)
	for scanner.Scan() {
		fmt.Println(scanner.Text())
	}
}
```

## 7 ioutil

如果只想将文件读入缓冲区怎么办？

ioutil是标准库中的软件包，其中包含一些使它成为单行的功能。

读取整个文件

```go
func main() {
	bytes, err := ioutil.ReadFile("filetoread.txt")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Bytes read: ", len(bytes))
	fmt.Println("String read: ", string(bytes))
}
```

读取文件的整个目录（不适用于大文件）

```go
func main() {
	filelist, err := ioutil.ReadDir(".")
	if err != nil {
		log.Fatal(err)
	}
	for _, fileinfo := range filelist {
		if fileinfo.Mode().IsRegular() {
			bytes, err := ioutil.ReadFile(fileinfo.Name())
			if err != nil {
				log.Fatal(err)
			}
			fmt.Println("Bytes read: ", len(bytes))
			fmt.Println("String read: ", string(bytes))
		}
	}
}
```





# 读文件2

## ioutil.ReadFile()

利用ioutil.ReadFile直接从文件读取到[]byte中

```go
func Read0()  (string){
    f, err := ioutil.ReadFile("file/test")
    if err != nil {
        fmt.Println("read fail", err)
    }
    return string(f)
}
```

## os.Open、[]byte、Read()

先从文件读取到file中，在从file读取到buf, buf在追加到最终的[]byte

```go
func Read1()  (string){
    //获得一个file
    f, err := os.Open("file/test")
    if err != nil {
        fmt.Println("read fail")
        return ""
    }
    //把file读取到缓冲区中
    defer f.Close()
    var chunk []byte
    buf := make([]byte, 1024)
    for {
        //从file读取到buf中
        n, err := f.Read(buf)
        if err != nil && err != io.EOF{
            fmt.Println("read buf fail", err)
            return ""
        }
        //说明读取结束
        if n == 0 {
            break
        }
        //读取到最终的缓冲区中
        chunk = append(chunk, buf[:n]...)
    }
    return string(chunk)
    //fmt.Println(string(chunk))
}
```

## os.Open、bufio.NewReader、Read()

先从文件读取到file, 在从file读取到Reader中，从Reader读取到buf, buf最终追加到[]byte

```go
//先从文件读取到file, 在从file读取到Reader中，从Reader读取到buf, buf最终追加到[]byte，这个排第三
func Read2() (string) {
    fi, err := os.Open("file/test")
    if err != nil {
        panic(err)
    }
    defer fi.Close()
    r := bufio.NewReader(fi)
    var chunks []byte
    buf := make([]byte, 1024)
    for {
        n, err := r.Read(buf)
        if err != nil && err != io.EOF {
            panic(err)
        }
        if 0 == n {
            break
        }
        //fmt.Println(string(buf))
        chunks = append(chunks, buf...)
    }
    return string(chunks)
    //fmt.Println(string(chunks))
}
```

## os.Open、ioutil.ReadAll()

读取到file中，再利用ioutil将file直接读取到[]byte中

```go
//读取到file中，再利用ioutil将file直接读取到[]byte中, 这是最优
func Read3()  (string){
    f, err := os.Open("file/test")
    if err != nil {
        fmt.Println("read file fail", err)
        return ""
    }
    defer f.Close()
    fd, err := ioutil.ReadAll(f)
    if err != nil {
        fmt.Println("read to fd fail", err)
        return ""
    }
    return string(fd)
}
```

# 写文件

## io.WriteString()

使用 io.WriteString 写入文件

```go
func Write0()  {
    fileName := "file/test1"
    strTest := "测试测试"

    var f *os.File
    var err error

    if CheckFileExist(fileName) {  //文件存在
        f, err = os.OpenFile(fileName, os.O_APPEND, 0666) //打开文件
        if err != nil{
            fmt.Println("file open fail", err)
            return
        }
    }else {  //文件不存在
        f, err = os.Create(fileName) //创建文件
        if err != nil {
            fmt.Println("file create fail")
            return
        }
    }
    //将文件写进去
    n, err1 := io.WriteString(f, strTest)
    if err1 != nil {
        fmt.Println("write error", err1)
        return
    }
    fmt.Println("写入的字节数是：", n)
}
```

## ioutil.WriteFile()

使用 ioutil.WriteFile 写入文件

```go
func Write1()  {
    fileName := "file/test2"
    strTest := "测试测试"
    var d = []byte(strTest)
    err := ioutil.WriteFile(fileName, d, 0666)
    if err != nil {
        fmt.Println("write fail")
    }
    fmt.Println("write success")
}
```

## File(Write, WriteString)

使用 File(Write, WriteString) 写入文件

```go
func Write2()  {

    fileName := "file/test3"
    strTest := "测试测试"
    var d1 = []byte(strTest)

    f, err3 := os.Create(fileName) //创建文件
    if err3 != nil{
        fmt.Println("create file fail")
    }
    defer f.Close()
    n2, err3 := f.Write(d1) //写入文件(字节数组)

    fmt.Printf("写入 %d 个字节n", n2)
    n3, err3 := f.WriteString("writesn") //写入文件(字节数组)
    fmt.Printf("写入 %d 个字节n", n3)
    f.Sync()
}
```

## bufio.NewWriter()

使用 bufio.NewWriter 写入文件

```go
func Write3()  {
    fileName := "file/test3"
    f, err3 := os.Create(fileName) //创建文件
    if err3 != nil{
        fmt.Println("create file fail")
    }
    w := bufio.NewWriter(f) //创建新的 Writer 对象
    n4, err3 := w.WriteString("bufferedn")
    fmt.Printf("写入 %d 个字节n", n4)
    w.Flush()
    f.Close()
}
```

检查文件是否存在：

```go
func CheckFileExist(fileName string) bool {
    _, err := os.Stat(fileName)
    if os.IsNotExist(err) {
        return false
    }
    return true
}
```





# 复制文件

主要介绍三种复制文件的方法：

1. 使用`io.Copy()`方法
2. 一次性读取输入文件，然后再一次性写入目标文件
3. 使用buffer一块块地复制文件

## 1 io.Copy

```go
func copy(src, dst string) (int64, error) {
   sourceFileStat, err := os.Stat(src)
   if err != nil {
      return 0, err
   }

   if !sourceFileStat.Mode().IsRegular() {
      return 0, fmt.Errorf("%s is not a regular file", src)
   }

   source, err := os.Open(src)
   if err != nil {
      return 0, err
   }
   defer source.Close()

   destination, err := os.Create(dst)
   if err != nil {
      return 0, err
   }

   defer destination.Close()
   nBytes, err := io.Copy(destination, source)
   return nBytes, err
```

这种方法虽然简单，但是缺少灵活性。如果文件太大，不是一种很好的方法。

## 2 ioutil.WriteFile()、ioutil.ReadFile()

```go
input, err := ioutil.ReadFile(sourceFile)
if err != nil {
   fmt.Println(err)
   return
}

err = ioutil.WriteFile(destinationFile, input, 0644)
if err != nil {
   fmt.Println("Error creating", destinationFile)
   fmt.Println(err)
   return
}
```

不是很高效的方法。

## 3 os.Read()、os.Write()

```go
buf := make([]byte, BUFFERSIZE)
for {
   n, err := source.Read(buf)
   if err != nil && err != io.EOF {
      return err
   }
   if n == 0 {
      break
   }

   if _, err := destination.Write(buf[:n]); err != nil {
      return err
   }
}
```