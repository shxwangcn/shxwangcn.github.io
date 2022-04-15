---
title: 聊聊golang中的数据处理
date: 2022-04-12 11:40:48 +0800
categories: [golang, io]
tags: golang buffer
mermaid: true
---

本篇文章，我们来聊聊golang中的数据处理。

在golang中操作二进制数据，你肯定用过下面的这些类型或者函数：

- `bytes.Buffer`
- `io.Copy`
- `ioutil.ReadAll` 

这些类或者函数，底层实现是怎样的？在特定的场景下，我们应该如何选择用哪种呢？

## 1 认识golang中的数据读写机制

golang中，所有的读写数据的类，都实现了下面的两个基本接口：

```go
// 将数据从对象读取到指定slice，必须保证 len(p) 足够容纳想要读取的数据
// 若成功，则返回实际读取到的字节数（0 <= n <= len(p))
type Reader interface {
    Read(p []byte) (n int, err error)
}

// 将指定slice的数据写入到对象
// 返回实际写入的字节数（取决于对象内部能容纳多少数据）
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

假定，我们现在要写一个`Copy(in Reader, out Writer)`的函数，实现将数据从 `in` 拷贝到 `out`。

很简单，只要先调用 `A.Read()` ，再调用 `B.Write()` 就完事了。

OK，接下来就可以写代码实现了。不过，在此之前，我们要先确认一下 `Read` 和 `Write` 方法的具体行为。

对于 `Read`：

- 当返回的err不为 nil 时，n 可能 > 0，需要注意处理；
- 当读取到的数据不能填满buf时，返回的 err 可能是 EOF，但也有可能是 nil，取决于具体实现；
- 当没有数据可读时，所有实现都应当返回 `0, EOF` 

这意味着，我们在处理 `Read` 返回时，应当先处理 `n > 0`，再处理 `err != nil`。

对于 `Write`：

- 当返回的 n < len(p) 时，应当返回一个 非 nil 的 err；

接下来，就是我们实现的`CopyData`了：

```go
func CopyData(in Reader, out Writer) (int,error) {
    // 先创建一个足够容纳要拷贝的数据的slice，这里假定不超过1KB
    buf := make([]byte, 1024)

    // 先把数据从 in 拷贝到 buf
    n, err := in.Read(buf)
    if n > 0 {
        // 将buf截断为实际读取到的长度
        buf := buf[:n]

        // 再把数据写到 out
        n, err = out.Write(buf)
        return n, err
    }
    if err == EOF {
        return n, nil
    }
    return n, err
}
```

只要掌握了 `Reader` 和 `Writer` 这两个接口的使用，那么其实就已经掌握了 golang 中操作数据的精髓了：

数据的处理可以是将一系列 Handler 链在一起。起始处是一个 Reader，中间加工环节都是 Writer + Reader，最后是一个 Writer。

![dataflow](/golang/dataflow.png)

## 2 读取长度未知的数据

在上面的示例中，我们第一步中，把数据从 Reader 拷贝到临时 buffer，假定数据不超过某个限制。那么在实际开发中，如果碰到事先不知道数据长度的情况，该怎么办呢？

一般地，对于 Reader 接口的实现而言，它的内部一般都会有个指示当前读取到了哪个位置（偏移量）。每次 Read 操作，都会根据实际已读取的字节数，往前移动偏移量。

> 这里插播另外一个接口：`Seeker`。对于 Reader，偏移量只能单调地往前进，既不能往后退，也不能往前跳过部分数据，在有些场景可能会带来不便。而 Seeker，顾名思义，实现了 Seek() 方法，可以往前跳跃，也可以往后回退，甚至可以实现倒着读数据。

所以，为了读取长度未知的Reader，只要加个循环，不停地以固定size（例如1KB）为单位从Reader拷贝出数据，直到没有数据为止。

我们把上面写的单次拷贝的函数实现改名为 `CopyBlock`，然后再实现下面的`CopyData`：

```go
func CopyData(in Reader, out Writer) (int,error) {
    bytes := 0
    for {
        n, err := CopyBlock(in[bytes:], out[bytes:])
        if n > 0 {
            bytes += n
        }
        if err != nil {
            return bytes, err
        }
    }
    return bytes, nil
}
```

当然，`CopyData`这么基础的函数，肯定是非常有用的，所以官方提供了一个标准实现：`io.Copy()`。

可以看看它的实现代码：

```go
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}

func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	if buf == nil {
		size := 32 * 1024
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
				size = int(l.N)
			}
		}
		buf = make([]byte, size)
	}
	for {
		nr, er := src.Read(buf)
		if nr > 0 {
			nw, ew := dst.Write(buf[0:nr])
			if nw < 0 || nr < nw {
				nw = 0
				if ew == nil {
					ew = errInvalidWrite
				}
			}
			written += int64(nw)
			if ew != nil {
				err = ew
				break
			}
			if nr != nw {
				err = ErrShortWrite
				break
			}
		}
		if er != nil {
			if er != EOF {
				err = er
			}
			break
		}
	}
	return written, err
}
```

可以看到，实现原理跟我们上面实现的`CopyData`基本原理一样，不过它有个优化点：

- 如果 Reader 对象同时也实现了 `WriterTo` 接口，那么直接调用它的 `WriteTo` 方法；
- 亦或者，如果 Writer 对象同时也实现了 `ReaderFrom` 接口，那么直接调用它的 `ReadFrom` 方法

从而无需把数据先拷贝到临时buffer。

## 3 `bytes.Buffer`

聊到现在，我们发现了golang中处理数据的核心，就是 Reader 和 Writer 两个接口了。

那么，如果我们现在有一段二进制数据（存放在 `[]byte` 或者 `string` 中）需要处理，那么我们怎么把它转换成一个 Reader 对象呢？

答案就是golang官方库提供的 `bytes.Buffer` 类型！

它同时实现了 Reader 接口和 Writer 接口，并可以用 `[]byte` 或者 `string` 初始化。

```go
bindata := make([]byte, 1024)
buf := bytes.NewBuffer(bindata)
```

上例中，我们从 bindata 创建了一个 bytes.Buffer 对象。接着，我们就可以将它当成一个 Reader 来处理 bindata。

当作为 Reader 时，bytes.Buffer 内部维护了一个偏移量，以指示当前读取到了哪个位置，这样下次的`Read()`方法可以接着上次位置继续读取。

其实，`bytes.Buffer`类型的定义也是非常简单的：

```go
type Buffer struct {
	buf      []byte // contents are the bytes buf[off : len(buf)]
	off      int    // read at &buf[off], write at &buf[len(buf)]
	lastRead readOp // last read operation, so that Unread* can work correctly.
}
```

> `lastRead`只是为了在调用`Unread*`系列函数来撤销上次的读取操作时，保存上次的读操作类型。所谓“撤销读”操作，其实就是将偏移量回退到上次读操作之前的位置。那么，当然要知道上次是读取了什么类型的数据了（即上次读取了几个字节）。

当作为 Writer 时，`bytes.Buffer`内部会自动分配足够的内存以保存写入的数据，而无需使用者关心。

```go
// 当作为Writer使用时，创建一个零值的buffer即可
buf := bytes.Buffer{}

data := make([]byte, 1024)

// 当写入数据时，buffer内部自动扩容
buf.Write(data)
```

要想了解 Buffer 的内部扩容机制，最好先了解下 [slice 内部的内存分配机制](https://shxwangcn.github.io/posts/golang-slice/)，然后再结合 Buffer 的`Write()`方法的实现代码一起理解即可。

## 4 聊聊链式处理的劣势

不知道大家有没有注意到，Reader + Writer 的方式，只能形成简单的单链处理逻辑。因为一个 Reader，你一旦把它的数据读取了，那么就没法再读取第二遍。

要想在某个节点处分叉，那么必须先把数据拷贝出来(`[]byte`) ，然后再基于该 slice,重新初始化多个新的 Reader 去处理它。

所以这里又有了个新需求：将数据从 Reader 一次性全部拷贝到单个 slice（以下称其为 resultSlice）。这就是官方库里的 `io.ReadAll()`。

```go
// 从 Reader 一直读数据，直到出错或者返回EOF，并返回成功读取到的数据
bytes, err := io.ReadAll(in)
```

因为无法事先知道 Reader 的数据长度，所以 `ReadAll`的内部实现，一种可能的方案，就是在一个循环内，不停地按固定size读取出数据（unitSlice），再将 unitSlice 数据 append 到 resultSlice，直到EOF为止。

这里贴下它的实现：

```go
func ReadAll(r Reader) ([]byte, error) {
	b := make([]byte, 0, 512)
	for {
		if len(b) == cap(b) {
			// Add more capacity (let append pick how much).
			b = append(b, 0)[:len(b)]
		}
		n, err := r.Read(b[len(b):cap(b)])
		b = b[:len(b)+n]
		if err != nil {
			if err == EOF {
				err = nil
			}
			return b, err
		}
	}
}
```

这里的实现比较不直白。它并不是以固定的size去循环读，而是先分配个初始的 512Bytes的 slice，然后有额外的数据时，先依赖 slice本身的内存扩容机制（每扩容一次，需要将数据全部从旧位置复制到新位置）将其扩容，再将数据直接写到扩容后的空间。

## 数据的序列化&反序列化

上面我们说了很多，但是都只是涉及到数据的复制，没有涉及到对数据的处理。实际开发中，我们经常会碰到数据的序列化和反序列化问题。

- 所谓“序列化”，就是将一个具有某种结构（类型）的对象，按照某种规则，转换成一段二进制数据（一般是`[]byte`）；
- 所谓“反序列化”， 就是从一段二进制数据，按照某种规则，将其转换回特定结构（类型）的对象。

### 反序列化

对于反序列化，可以将数据的来源抽象成 `io.Reader`，然后一边读，一边进行数据转换。

打个比方，我们在做服务开发时，肯定会涉及到 API 的调用（HTTP RESTFul API，或者 grpc/brpc等RPC框架）。这里以 HTTP 请求返回 JSON 格式的数据为例：

```go
type User struct {
	Id      string `json:"id"`
	Name    string `json:"name"`
	Address string `json:"address"`
	Tel     string `json:"tel"`
}

func GetUserInfo(id string) (*User, error) {
	resp, err := http.Get("https://example.com/users/" + id)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	user := User{}
}
```

这里，我们需要实现一个 `GetUserInfo` 函数，从服务端查询回JSON格式的数据，然后将其转换成一个`User`类型的对象。

请求返回的 Body 数据存储在一个 `resp.Body` 对象中，它的接口类型是`io.ReadCloser`（ io.Reader + io.Closer）。

一种比较直观的方法就是，我们先把它的数据全部读取出来（`io.ReadAll`），然后调用 `json.Unmarshal` 完成 json 反序列化。

```
	data, err := io.ReadAll(resp.Body)
	err = json.Unmarshal(data, &user)
	if err != nil {
		return nil, err
	}

	return &user, nil
}
```

当然，熟悉使用json库的同学，肯定知道还有另外一种方法完成反序列化：

```go
decoder := json.NewDecoder(resp.Body)
decoder.Decode(&user)
```

这里引入了一种新的类型：`json.Decoder`。创建它时，需要传入一个 Reader 对象；然后在调用`Decode()`时，直接从 Reader 对象一边读，一边反序列化（就跟从 `[]byte` 边读边序列化一样）。

相比于上面的方法，这种方法减少了一次数据复制的开销，性能更强。

### 序列化

对于序列化，将数据的去处，抽象化为 `io.Writer` 对象，然后一边转换，一边将数据写到 Writer 即可。

还是以上面的场景为例，这次我们需要实现一个 `UpdateUserInfo()` 函数，去服务端更新用户信息。

```go
func UpdateUserInfo(user *User) error {
	buf := bytes.Buffer{}

	encoder := json.NewEncoder(&buf)
	err := encoder.Encode(user)
	if err != nil {
		return err
	}

	resp, err := http.Post("https://example.com/users", "application/json", &buf)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("PostFailed: %d(%s)", resp.StatusCode, resp.Status)
	}
	return nil
}
```

思考下这里数据的走向，涉及到两个环节：序列化（结构体对象 -> 二进制数据）和 HTTP发包（二进制数据 -> 网络协议栈）。

所以，这里我们使用 `bytes.Buffer` 来存储二进制数据。在序列化环节，它是一个 Writer；在发包环节，它是一个 Reader。

## 总结

本篇文章，从最基础的 Reader 接口 和 Writer 接口入手，一步步地为大家介绍了 `io.Copy`，`bytes.Buffer` 和 `io.ReadAll` 的使用和内部实现。

尤其需要注意的是，对于数据处理，优先考虑是否将其抽象化为 `Reader` 和 `Writer`。只有在特别必要的场景下，才需要使用`io.ReadAll`将原始数据拷贝出来。