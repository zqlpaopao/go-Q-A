# Golang读写文件操作

### 读文件

使用golang语言去读取一个文件默认会有多种方式，这里主要介绍以下几种。

#### 使用`ioutil`直接读取

需要引入`io/ioutil`包，该包默认拥有以下函数供用户调用。

```javascript
func NopCloser(r io.Reader) io.ReadCloser
func ReadAll(r io.Reader) ([]byte, error)
func ReadDir(dirname string) ([]os.FileInfo, error)
func ReadFile(filename string) ([]byte, error)
func TempDir(dir, prefix string) (name string, err error)
func TempFile(dir, prefix string) (f *os.File, err error)
func WriteFile(filename string, data []byte, perm os.FileMode) error
```

读文件，我们可以看以下三个函数:

```javascript
//从一个io.Reader类型中读取内容直到返回错误或者EOF时返回读取的数据，当err == nil时，数据成功读取到[]byte中
//ReadAll函数被定义为从源中读取数据直到EOF，它是不会去从返回数据中去判断EOF来作为读取成功的依据
func ReadAll(r io.Reader) ([]byte, error)

//读取一个目录，并返回一个当前目录下的文件对象列表和错误信息
func ReadDir(dirname string) ([]os.FileInfo, error)

//读取文件内容，并返回[]byte数据和错误信息。err == nil时，读取成功
func ReadFile(filename string) ([]byte, error)
```

读取文件示例:

```javascript
$ cat readfile.go
package main
import (
    "fmt"
    "io/ioutil"
    "strings"
)
func main() {
   Ioutil("mytestfile.txt")
    }

func Ioutil(name string) {
    if contents,err := ioutil.ReadFile(name);err == nil {
        //因为contents是[]byte类型，直接转换成string类型后会多一行空格,需要使用strings.Replace替换换行符
        result := strings.Replace(string(contents),"\n","",1)
        fmt.Println(result)
        }
    }

$ go run readfile.go
xxbandy.github.io @by Andy_xu
```

#### 借助`os.Open`进行读取文件

由于`os.Open`是打开一个文件并返回一个文件对象，因此其实可以结合`ioutil.ReadAll(r io.Reader)`来进行读取。 `io.Reader`其实是一个包含`Read`方法的接口类型，而文件对象本身是实现了了`Read`方法的。

我们先来看下`os.Open`家族的相关函数

```javascript
//打开一个需要被读取的文件，如果成功读取，返回的文件对象将可用被读取，该函数默认的权限为O_RDONLY，也就是只对文件有只读权限。如果有错误，将返回*PathError类型
func Open(name string) (*File, error)

//大部分用户会选择该函数来代替Open or Create函数。该函数主要用来指定参数(os.O_APPEND|os.O_CREATE|os.O_WRONLY)以及文件权限(0666)来打开文件，如果打开成功返回的文件对象将被用作I/O操作
func OpenFile(name string, flag int, perm FileMode) (*File, error)
```

使用`os.Open`家族函数和`ioutil.ReadAll()`读取文件示例:

```javascript
func OsIoutil(name string) {
      if fileObj,err := os.Open(name);err == nil {
      //if fileObj,err := os.OpenFile(name,os.O_RDONLY,0644); err == nil {
        defer fileObj.Close()
        if contents,err := ioutil.ReadAll(fileObj); err == nil {
            result := strings.Replace(string(contents),"\n","",1)
            fmt.Println("Use os.Open family functions and ioutil.ReadAll to read a file contents:",result)
            }

        }
}

# 在main函数中调用OsIoutil(name)函数就可以读取文件内容了
$ go run readfile.go
Use os.Open family functions and ioutil.ReadAll to read a file contents: xxbandy.github.io @by Andy_xu
```

然而上述方式会比较繁琐一些，因为使用了`os`的同时借助了`ioutil`,但是在读取大文件的时候还是比较有优势的。不过读取小文件可以直接使用文件对象的一些方法。

不论是上边说的`os.Open`还是`os.OpenFile`他们最终都返回了一个`*File`文件对象，而该文件对象默认是有很多方法的，其中读取文件的方法有如下几种:

```javascript
//从文件对象中读取长度为b的字节，返回当前读到的字节数以及错误信息。因此使用该方法需要先初始化一个符合内容大小的空的字节列表。读取到文件的末尾时，该方法返回0，io.EOF
func (f *File) Read(b []byte) (n int, err error)

//从文件的off偏移量开始读取长度为b的字节。返回读取到字节数以及错误信息。当读取到的字节数n小于想要读取字节的长度len(b)的时候，该方法将返回非空的error。当读到文件末尾时，err返回io.EOF
func (f *File) ReadAt(b []byte, off int64) (n int, err error)
```

使用文件对象的`Read`方法读取：

```javascript
func FileRead(name string) {
    if fileObj,err := os.Open(name);err == nil {
        defer fileObj.Close()
        //在定义空的byte列表时尽量大一些，否则这种方式读取内容可能造成文件读取不完整
        buf := make([]byte, 1024)
        if n,err := fileObj.Read(buf);err == nil {
               fmt.Println("The number of bytes read:"+strconv.Itoa(n),"Buf length:"+strconv.Itoa(len(buf)))
               result := strings.Replace(string(buf),"\n","",1)
               fmt.Println("Use os.Open and File's Read method to read a file:",result)
            }
    }
}
```

#### 使用`os.Open`和`bufio.Reader`读取文件内容

`bufio`包实现了缓存IO,它本身包装了`io.Reader`和`io.Writer`对象，创建了另外的Reader和Writer对象，不过该种方式是带有缓存的，因此对于文本I/O来说，该包是提供了一些便利的。

先看下`bufio`模块下的相关的`Reader`函数方法：

```javascript
//首先定义了一个用来缓冲io.Reader对象的结构体，同时该结构体拥有以下相关的方法
type Reader struct {
}

//NewReader函数用来返回一个默认大小buffer的Reader对象(默认大小好像是4096) 等同于NewReaderSize(rd,4096)
func NewReader(rd io.Reader) *Reader

//该函数返回一个指定大小buffer(size最小为16)的Reader对象，如果 io.Reader参数已经是一个足够大的Reader，它将返回该Reader
func NewReaderSize(rd io.Reader, size int) *Reader


//该方法返回从当前buffer中能被读到的字节数
func (b *Reader) Buffered() int

//Discard方法跳过后续的 n 个字节的数据，返回跳过的字节数。如果0 <= n <= b.Buffered(),该方法将不会从io.Reader中成功读取数据。
func (b *Reader) Discard(n int) (discarded int, err error)

//Peekf方法返回缓存的一个切片，该切片只包含缓存中的前n个字节的数据
func (b *Reader) Peek(n int) ([]byte, error)

//把Reader缓存对象中的数据读入到[]byte类型的p中，并返回读取的字节数。读取成功，err将返回空值
func (b *Reader) Read(p []byte) (n int, err error)

//返回单个字节，如果没有数据返回err
func (b *Reader) ReadByte() (byte, error)

//该方法在b中读取delimz之前的所有数据，返回的切片是已读出的数据的引用，切片中的数据在下一次的读取操作之前是有效的。如果未找到delim，将返回查找结果并返回nil空值。因为缓存的数据可能被下一次的读写操作修改，因此一般使用ReadBytes或者ReadString，他们返回的都是数据拷贝
func (b *Reader) ReadSlice(delim byte) (line []byte, err error)

//功能同ReadSlice，返回数据的拷贝
func (b *Reader) ReadBytes(delim byte) ([]byte, error)

//功能同ReadBytes,返回字符串
func (b *Reader) ReadString(delim byte) (string, error)

//该方法是一个低水平的读取方式，一般建议使用ReadBytes('\n') 或 ReadString('\n')，或者使用一个 Scanner来代替。ReadLine 通过调用 ReadSlice 方法实现，返回的也是缓存的切片，用于读取一行数据，不包括行尾标记（\n 或 \r\n）
func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)

//读取单个UTF-8字符并返回一个rune和字节大小
func (b *Reader) ReadRune() (r rune, size int, err error)
```

示例：

```javascript
func BufioRead(name string) {
    if fileObj,err := os.Open(name);err == nil {
        defer fileObj.Close()
        //一个文件对象本身是实现了io.Reader的 使用bufio.NewReader去初始化一个Reader对象，存在buffer中的，读取一次就会被清空
        reader := bufio.NewReader(fileObj)
        //使用ReadString(delim byte)来读取delim以及之前的数据并返回相关的字符串.
        if result,err := reader.ReadString(byte('@'));err == nil {
            fmt.Println("使用ReadSlince相关方法读取内容:",result)
        }
        //注意:上述ReadString已经将buffer中的数据读取出来了，下面将不会输出内容
        //需要注意的是，因为是将文件内容读取到[]byte中，因此需要对大小进行一定的把控
        buf := make([]byte,1024)
        //读取Reader对象中的内容到[]byte类型的buf中
        if n,err := reader.Read(buf); err == nil {
            fmt.Println("The number of bytes read:"+strconv.Itoa(n))
            //这里的buf是一个[]byte，因此如果需要只输出内容，仍然需要将文件内容的换行符替换掉
            fmt.Println("Use bufio.NewReader and os.Open read file contents to a []byte:",string(buf))
        }


    }
}
```

#### 读文件所有方式示例

```javascript
/**
 * @File Name: readfile.go
 * @Author:
 * @Email:
 * @Create Date: 2017-12-16 16:12:01
 * @Last Modified: 2017-12-17 12:12:02
 * @Description:读取指定文件的几种方法，需要注意的是[]byte类型在转换成string类型的时候，都会在最后多一行空格，需要使用result := strings.Replace(string(contents),"\n","",1) 方式替换换行符
 */
package main
import (
    "fmt"
    "io/ioutil"
    "strings"
    "os"
    "strconv"
    "bufio"
)

func main() {
   Ioutil("mytestfile.txt")
   OsIoutil("mytestfile.txt")
   FileRead("mytestfile.txt")
   BufioRead("mytestfile.txt")
    }


func Ioutil(name string) {
    if contents,err := ioutil.ReadFile(name);err == nil {
        //因为contents是[]byte类型，直接转换成string类型后会多一行空格,需要使用strings.Replace替换换行符
        result := strings.Replace(string(contents),"\n","",1)
        fmt.Println("Use ioutil.ReadFile to read a file:",result)
        }
    }

func OsIoutil(name string) {
      if fileObj,err := os.Open(name);err == nil {
      //if fileObj,err := os.OpenFile(name,os.O_RDONLY,0644); err == nil {
        defer fileObj.Close()
        if contents,err := ioutil.ReadAll(fileObj); err == nil {
            result := strings.Replace(string(contents),"\n","",1)
            fmt.Println("Use os.Open family functions and ioutil.ReadAll to read a file :",result)
            }

        }
}


func FileRead(name string) {
    if fileObj,err := os.Open(name);err == nil {
        defer fileObj.Close()
        //在定义空的byte列表时尽量大一些，否则这种方式读取内容可能造成文件读取不完整
        buf := make([]byte, 1024)
        if n,err := fileObj.Read(buf);err == nil {
               fmt.Println("The number of bytes read:"+strconv.Itoa(n),"Buf length:"+strconv.Itoa(len(buf)))
               result := strings.Replace(string(buf),"\n","",1)
               fmt.Println("Use os.Open and File's Read method to read a file:",result)
            }
    }
}

func BufioRead(name string) {
    if fileObj,err := os.Open(name);err == nil {
        defer fileObj.Close()
        //一个文件对象本身是实现了io.Reader的 使用bufio.NewReader去初始化一个Reader对象，存在buffer中的，读取一次就会被清空
        reader := bufio.NewReader(fileObj)
        //使用ReadString(delim byte)来读取delim以及之前的数据并返回相关的字符串.
        if result,err := reader.ReadString(byte('@'));err == nil {
            fmt.Println("使用ReadSlince相关方法读取内容:",result)
        }
        //注意:上述ReadString已经将buffer中的数据读取出来了，下面将不会输出内容
        //需要注意的是，因为是将文件内容读取到[]byte中，因此需要对大小进行一定的把控
        buf := make([]byte,1024)
        //读取Reader对象中的内容到[]byte类型的buf中
        if n,err := reader.Read(buf); err == nil {
            fmt.Println("The number of bytes read:"+strconv.Itoa(n))
            //这里的buf是一个[]byte，因此如果需要只输出内容，仍然需要将文件内容的换行符替换掉
            fmt.Println("Use bufio.NewReader and os.Open read file contents to a []byte:",string(buf))
        }


    }
}
```

### 写文件

那么上述几种方式来读取文件的方式也支持文件的写入，相关的方法如下:

#### 使用`ioutil`包进行文件写入

```javascript
// 写入[]byte类型的data到filename文件中，文件权限为perm
func WriteFile(filename string, data []byte, perm os.FileMode) error
```

示例:

```javascript
$ cat writefile.go
/**
 * @File Name: writefile.go
 * @Author:
 * @Email:
 * @Create Date: 2017-12-17 12:12:09
 * @Last Modified: 2017-12-17 12:12:30
 * @Description:使用多种方式将数据写入文件
 */
package main
import (
    "fmt"
    "io/ioutil"
)

func main() {
      name := "testwritefile.txt"
      content := "Hello, xxbandy.github.io!\n"
      WriteWithIoutil(name,content)
}

//使用ioutil.WriteFile方式写入文件,是将[]byte内容写入文件,如果content字符串中没有换行符的话，默认就不会有换行符
func WriteWithIoutil(name,content string) {
    data :=  []byte(content)
    if ioutil.WriteFile(name,data,0644) == nil {
        fmt.Println("写入文件成功:",content)
        }
    }

# 会有换行符
$ go run writefile.go
写入文件成功: Hello, xxbandy.github.io!
```

#### 使用`os.Open`相关函数进行文件写入

因为`os.Open`系列的函数会打开文件，并返回一个文件对象指针，而该文件对象是一个定义的结构体，拥有一些相关写入的方法。

文件对象结构体以及相关写入文件的方法:

```javascript
//写入长度为b字节切片到文件f中，返回写入字节号和错误信息。当n不等于len(b)时，将返回非空的err
func (f *File) Write(b []byte) (n int, err error)
//在off偏移量出向文件f写入长度为b的字节
func (f *File) WriteAt(b []byte, off int64) (n int, err error)
//类似于Write方法，但是写入内容是字符串而不是字节切片
func (f *File) WriteString(s string) (n int, err error)
```

`注意：`使用WriteString()j进行文件写入发现经常新内容写入时无法正常覆盖全部新内容。(是因为字符串长度不一样)

示例：

```javascript
//使用os.OpenFile()相关函数打开文件对象，并使用文件对象的相关方法进行文件写入操作
func WriteWithFileWrite(name,content string){
    fileObj,err := os.OpenFile(name,os.O_RDWR|os.O_CREATE|os.O_TRUNC,0644)
    if err != nil {
        fmt.Println("Failed to open the file",err.Error())
        os.Exit(2)
    }
    defer fileObj.Close()
    if _,err := fileObj.WriteString(content);err == nil {
        fmt.Println("Successful writing to the file with os.OpenFile and *File.WriteString method.",content)
    }
    contents := []byte(content)
    if _,err := fileObj.Write(contents);err == nil {
        fmt.Println("Successful writing to thr file with os.OpenFile and *File.Write method.",content)
    }
}
```

`注意:`使用`os.OpenFile(name string, flag int, perm FileMode)`打开文件并进行文件内容更改，需要注意`flag`相关的参数以及含义。

```javascript
const (
        O_RDONLY int = syscall.O_RDONLY // 只读打开文件和os.Open()同义
        O_WRONLY int = syscall.O_WRONLY // 只写打开文件
        O_RDWR   int = syscall.O_RDWR   // 读写方式打开文件
        O_APPEND int = syscall.O_APPEND // 当写的时候使用追加模式到文件末尾
        O_CREATE int = syscall.O_CREAT  // 如果文件不存在，此案创建
        O_EXCL   int = syscall.O_EXCL   // 和O_CREATE一起使用, 只有当文件不存在时才创建
        O_SYNC   int = syscall.O_SYNC   // 以同步I/O方式打开文件，直接写入硬盘.
        O_TRUNC  int = syscall.O_TRUNC  // 如果可以的话，当打开文件时先清空文件
)
```

#### 使用`io`包中的相关函数写入文件

在`io`包中有一个`WriteString()`函数，用来将字符串写入一个`Writer`对象中。

```javascript
//将字符串s写入w(可以是一个[]byte)，如果w实现了一个WriteString方法，它可以被直接调用。否则w.Write会再一次被调用
func WriteString(w Writer, s string) (n int, err error)

//Writer对象的定义
type Writer interface {
        Write(p []byte) (n int, err error)
}
```

示例:

```javascript
//使用io.WriteString()函数进行数据的写入
func WriteWithIo(name,content string) {
    fileObj,err := os.OpenFile(name,os.O_RDWR|os.O_CREATE|os.O_APPEND,0644)
    if err != nil {
        fmt.Println("Failed to open the file",err.Error())
        os.Exit(2)
    }
    if  _,err := io.WriteString(fileObj,content);err == nil {
        fmt.Println("Successful appending to the file with os.OpenFile and io.WriteString.",content)
    }
}
```

#### 使用`bufio`包中的相关函数写入文件

`bufio`和`io`包中很多操作都是相似的，唯一不同的地方是`bufio`提供了一些缓冲的操作，如果对文件I/O操作比较频繁的，使用`bufio`还是能增加一些性能的。

在`bufio`包中，有一个`Writer`结构体，而其相关的方法也支持一些写入操作。

```javascript
//Writer是一个空的结构体，一般需要使用NewWriter或者NewWriterSize来初始化一个结构体对象
type Writer struct {
        // contains filtered or unexported fields
}

//NewWriterSize和NewWriter函数
//返回默认缓冲大小的Writer对象(默认是4096)
func NewWriter(w io.Writer) *Writer

//指定缓冲大小创建一个Writer对象
func NewWriterSize(w io.Writer, size int) *Writer

//Writer对象相关的写入数据的方法

//把p中的内容写入buffer,返回写入的字节数和错误信息。如果nn<len(p),返回错误信息中会包含为什么写入的数据比较短
func (b *Writer) Write(p []byte) (nn int, err error)
//将buffer中的数据写入 io.Writer
func (b *Writer) Flush() error

//以下三个方法可以直接写入到文件中
//写入单个字节
func (b *Writer) WriteByte(c byte) error
//写入单个Unicode指针返回写入字节数错误信息
func (b *Writer) WriteRune(r rune) (size int, err error)
//写入字符串并返回写入字节数和错误信息
func (b *Writer) WriteString(s string) (int, error)
注意：如果需要再写入文件时利用缓冲的话只能使用bufio包中的Write方法
```

示例:

```javascript
//使用bufio包中Writer对象的相关方法进行数据的写入
func WriteWithBufio(name,content string) {
    if fileObj,err := os.OpenFile(name,os.O_RDWR|os.O_CREATE|os.O_APPEND,0644);err == nil {
        defer fileObj.Close()
        writeObj := bufio.NewWriterSize(fileObj,4096)
        //
       if _,err := writeObj.WriteString(content);err == nil {
              fmt.Println("Successful appending buffer and flush to file with bufio's Writer obj WriteString method",content)
           }

        //使用Write方法,需要使用Writer对象的Flush方法将buffer中的数据刷到磁盘
        buf := []byte(content)
        if _,err := writeObj.Write(buf);err == nil {
            fmt.Println("Successful appending to the buffer with os.OpenFile and bufio's Writer obj Write method.",content)
            if  err := writeObj.Flush(); err != nil {panic(err)}
            fmt.Println("Successful flush the buffer data to file ",content)
        }
        }
}
```

#### 写文件全部示例

```javascript
/**
 * @File Name: writefile.go
 * @Author:
 * @Email:
 * @Create Date: 2017-12-17 12:12:09
 * @Last Modified: 2017-12-17 23:12:10
 * @Description:使用多种方式将数据写入文件
 */
package main
import (
    "os"
    "io"
    "fmt"
    "io/ioutil"
    "bufio"
)

func main() {
      name := "testwritefile.txt"
      content := "Hello, xxbandy.github.io!\n"
      WriteWithIoutil(name,content)
      contents := "Hello, xuxuebiao\n"
      //清空一次文件并写入两行contents
      WriteWithFileWrite(name,contents)
      WriteWithIo(name,content)
      //使用bufio包需要将数据先读到buffer中，然后在flash到磁盘中
      WriteWithBufio(name,contents)
}

//使用ioutil.WriteFile方式写入文件,是将[]byte内容写入文件,如果content字符串中没有换行符的话，默认就不会有换行符
func WriteWithIoutil(name,content string) {
    data :=  []byte(content)
    if ioutil.WriteFile(name,data,0644) == nil {
        fmt.Println("写入文件成功:",content)
        }
    }

//使用os.OpenFile()相关函数打开文件对象，并使用文件对象的相关方法进行文件写入操作
//清空一次文件
func WriteWithFileWrite(name,content string){
    fileObj,err := os.OpenFile(name,os.O_RDWR|os.O_CREATE|os.O_TRUNC,0644)
    if err != nil {
        fmt.Println("Failed to open the file",err.Error())
        os.Exit(2)
    }
    defer fileObj.Close()
    if _,err := fileObj.WriteString(content);err == nil {
        fmt.Println("Successful writing to the file with os.OpenFile and *File.WriteString method.",content)
    }
    contents := []byte(content)
    if _,err := fileObj.Write(contents);err == nil {
        fmt.Println("Successful writing to thr file with os.OpenFile and *File.Write method.",content)
    }
}


//使用io.WriteString()函数进行数据的写入
func WriteWithIo(name,content string) {
    fileObj,err := os.OpenFile(name,os.O_RDWR|os.O_CREATE|os.O_APPEND,0644)
    if err != nil {
        fmt.Println("Failed to open the file",err.Error())
        os.Exit(2)
    }
    if  _,err := io.WriteString(fileObj,content);err == nil {
        fmt.Println("Successful appending to the file with os.OpenFile and io.WriteString.",content)
    }
}

//使用bufio包中Writer对象的相关方法进行数据的写入
func WriteWithBufio(name,content string) {
    if fileObj,err := os.OpenFile(name,os.O_RDWR|os.O_CREATE|os.O_APPEND,0644);err == nil {
        defer fileObj.Close()
        writeObj := bufio.NewWriterSize(fileObj,4096)
        //
       if _,err := writeObj.WriteString(content);err == nil {
              fmt.Println("Successful appending buffer and flush to file with bufio's Writer obj WriteString method",content)
           }

        //使用Write方法,需要使用Writer对象的Flush方法将buffer中的数据刷到磁盘
        buf := []byte(content)
        if _,err := writeObj.Write(buf);err == nil {
            fmt.Println("Successful appending to the buffer with os.OpenFile and bufio's Writer obj Write method.",content)
            if  err := writeObj.Flush(); err != nil {panic(err)}
            fmt.Println("Successful flush the buffer data to file ",content)
        }
        }
}
```