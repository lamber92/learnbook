# Golang实战心得之error

<p align="right">Lamber</p>
<p align="right">2022-05-23</p>



## 前言

本篇基于GO 1.17版本代码进行阐释。



众所周知，C系语言的"error"大多是围绕错误码定义的，比如C语言使用errno(一个整型)来表示错误及传达错误信息，常见的以-1表示。这种设计的结果是当触发的错误所处的代码层级较深，且错误码又与其他类型错误相同时，为了避免混淆，编码者会在底层进行大量的错误码映射直到将错误信息精妙地传递到展示层；或者直接将错误吞没不处理，打印错误日志；又或直接中断程序。

GO吸取了C语言处理错误的失败教训，设计了拓展性极强的错误处理方式，将错误处理的方式，抛给设计程序者自己去决策与处理。

GO面对致命性异常也不再像C程序一样需要分析core dump文件，而使用panic输出尽可能多的信息帮助开发者定位问题，本篇不涉及panic的内容。

下面我们来看下GO的"error"关键字到底是什么？它为什么具备拓展性？又带来了什么问题？我们又是如何利用GO的error设计思想去解决问题的？



## 1.  关键字"error"

### 1.1. error是什么？

- **摘自 Go\src\builtin/builtin.go，error 关键字是一个接口类型。**

```go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
// 内置接口类型是表示错误情况的常规接口，nil 值表示没有错误。
type error interface {
	Error() string
}
```

从接口方法 `Error() string` 可以看出，它仅仅表示用一个字符串就能诠释的错误，它能满足基本编码的需要吗？

我们带着疑问往下探索GO官方提供的两种error实现：`errorString` 与 `wrapError`。



### 1.2. errorString

乍一看对于新手来说，errorString应该会很陌生。

其实，它正是errors包对error接口的实现，想起各种教程中提及的 errors.New() 了吗？

- **摘自 Go\src\errors\errors.go, errorString 是error接口的一种实现**。

```go
package errors

// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
	return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

errorString 本身极其简单，仅仅实现了 error接口 的基础能力：保存一个错误描述字符串，并能将其输出，没有然后。

这种设计仅仅能满足与简单的业务场景，或简短的demo；如果在中型项目中使用errorString，可以想象到每次使用string作错误判断的痛苦吧？

但存在即是合理，它有它的优势：足够轻量级。



- **除了吐槽官方偷懒之外，errors包的另一个文件wrap.go，被许多开源库借鉴，足见errors包设计者的智慧。**

```go
func Unwrap(err error) error {
	u, ok := err.(interface {
		Unwrap() error
	})
	if !ok {
		return nil
	}
	return u.Unwrap()
}
```

`Unwrap` 解析error中包裹的error，即反解嵌套的error。此场景在业务逻辑中比较少出现，但是是必要手段，唯一缺陷是 实现者 必须包含 `Unwrap(error) error` 方法。



```go
func Is(err, target error) bool {
	......
}

func As(err, target error) bool {
	......
}
```

作者想到error的使用者可能会将error嵌套多层，`Is()` 函数可以判断在err嵌套链中是否存在目标error，而 `As()` 函数可以判断目标error是否处于err嵌套链的首位。当然，这两个函数都依赖 `Unwarp()`。



### 1.3. wrapError

```go
// Errorf 根据格式说明符进行格式化，并将字符串作为满足错误的值返回。
func Errorf(format string, a ...interface{}) error {
	p := newPrinter()
	p.wrapErrs = true
	p.doPrintf(format, a)
	s := string(p.buf)
	var err error
	if p.wrappedErr == nil {
		err = errors.New(s)
	} else {
		err = &wrapError{s, p.wrappedErr}
	}
	p.free()
	return err
}

type wrapError struct {
	msg string
	err error
}

func (e *wrapError) Error() string {
	return e.msg
}

func (e *wrapError) Unwrap() error {
	return e.err
}
```

fmt包的作者依然为多次嵌套error提供了反解能力，使用**%w占位符**表达err时，会将传入的err保存到 wrapError 内部的err变量中，并实现了 **Unwrap()** 方法，使得通过 fmt.Errorf() 函数包裹的原始err可以在外部进行比对及反解。

但需要注意几点：

- 必须使用%w占位符才生效，否则fmt.Errorf内部将创建一个新的errors.errorString保存错误信息；
- 当%w个数>1时，多出的占位符对应的err不会生效；
- 传入的err没有实现Unwrap()方法不生效；

结合errors包中的方法，做一个正常使用fmt.Errorf()的示例：

```go
e1 := errors.New("error1")
e2 := fmt.Errorf("error2: %w", e1)

fmt.Println(errors.Unwrap(e2))       // error1
fmt.Println(errors.Unwrap(e2) == e1) // true
```



### 1.4. error接口有什么问题？ 

综合 1.1 ~ 1.3 小节，可以总结出官方提供的error接口实现存在的问题：

- 只保存了简单的字符串信息，无法追溯错误发生的源头；或需要煞费苦心地对比错误的字符串内容是否为自己所需的，从而进行下一步业务操作；

- 为了代码的健壮性考虑，函数返回的每一个错误，都不能忽略。因为按GO的编码习惯，当返回参数包含err非nil时，很可能返回一个nil的对象，此时如果不对err进行判断，很可能会引发panic。这就使得代码中error出现的频率非常高，显得非常冗长。

  下面是常见的代码示例：

  ```go
  if err := doSomething1(); err != nil {
      // 处理error
  }
  if err := doSomething2(); err != nil {
      // 处理error
  }
  if err := doSomething3(); err != nil {
      // 处理error
  }
  ```

  这种处理方式像极了C语言中的处理，显得非常冗长。

  

  这里有人会疑问或者抱怨起来，为什么GO不使用高级语言中的try..catch..finally机制去处理错误？

  在官方的FAQ中已经有解释：https://golang.google.cn/doc/faq

  > ### Why does Go not have exceptions?
  >
  > We believe that coupling exceptions to a control structure, as in the `try-catch-finally` idiom, results in convoluted code. It also tends to encourage programmers to label too many ordinary errors, such as failing to open a file, as exceptional.
  >
  > Go takes a different approach. For plain error handling, Go's multi-value returns make it easy to report an error without overloading the return value. [A canonical error type, coupled with Go's other features](https://golang.google.cn/doc/articles/error_handling.html), makes error handling pleasant but quite different from that in other languages.
  >
  > Go also has a couple of built-in functions to signal and recover from truly exceptional conditions. The recovery mechanism is executed only as part of a function's state being torn down after an error, which is sufficient to handle catastrophe but requires no extra control structures and, when used well, can result in clean error-handling code.
  >
  > See the [Defer, Panic, and Recover](https://golang.google.cn/doc/articles/defer_panic_recover.html) article for details. Also, the [Errors are values](https://blog.golang.org/errors-are-values) blog post describes one approach to handling errors cleanly in Go by demonstrating that, since errors are just values, the full power of Go can be deployed in error handling.

  > ### 为什么 Go 没有异常？
  >
  > 我们相信将异常耦合到控制结构中，就像在`try-catch-finally`成语中一样，会导致代码复杂。它还倾向于鼓励程序员将太多的普通错误（例如无法打开文件）标记为异常。
  >
  > Go 采用了不同的方法。对于简单的错误处理，Go 的多值返回可以很容易地报告错误而不会重载返回值。 [规范的错误类型，加上 Go 的其他特性](https://golang.google.cn/doc/articles/error_handling.html)，使得错误处理令人愉快，但与其他语言中的错误处理完全不同。
  >
  > Go 还有一些内置函数可以发出信号并从真正的异常情况中恢复。恢复机制仅作为函数状态的一部分在错误后被拆除，这足以处理灾难但不需要额外的控制结构，并且如果使用得当，可以产生干净的错误处理代码。
  >
  > 有关详细信息，请参阅[延迟、恐慌和恢复](https://golang.google.cn/doc/articles/defer_panic_recover.html)文章。此外，[Errors are values](https://blog.golang.org/errors-are-values)博客文章描述了一种在 Go 中干净地处理错误的方法，通过演示，由于错误只是值，因此可以在错误处理中部署 Go 的全部功能。

  ​		

  GO设计者之一 Russ Cox 也解释过同样的问题：https://studygolang.com/articles/1674	

  >Go是为编写大型软件而设计的。我们喜欢让程序保持简洁，而不是为大量程序员编写的大型程序不断投入维护成本。基于异常的错误处理的一个诱惑是，对于小的程序它工作得很好。但是在一个庞大的代码库里，对于每一行代码，每一个普通的操作，都需要考虑它们是否会触发一个需要处理的异常。这对于生产效率和工程时间是个很大的拖累。我自己在编写大型Python程序时就遇到这样的问题。必须承认，Go语言里函数返回错误对于调用者来说并不方便，但它们很明白地表明了程序和类型系统里发生错误的可能性。当遇到函数返回错误时，小程序可能只想打印出错误然后退出程序；但是更精心设计的程序通常会根据不同的错误来源作出不同的处理。在这种情况，try和catch的处理方式会比显式的错误返回值处理更冗长。虽然Python语言的10行代码用Go语言来实现确实可能会更冗长，但是，Go语言的首要目标不是编写10行代码的程序。

  从客观的角度来讲，GO在error处理的设计上有弊有利；从我主观的角度来讲，对于弊，只是大多参与到GO项目的编写者因为保留了第一语言的习惯，使用GO有所不适罢了。比较认同上文的一句话：

  > 虽然Python语言的10行代码用Go语言来实现确实可能会更冗长，但是，Go语言的首要目标不是编写10行代码的程序。



综上，也只是阐述了官方提供的error包的两种实现，得益于error接口本身的设计，具备良好的拓展性，可以灵活地实现所需；然而灵活性也带来了多样性，从而给处理众多实现各异三方库中的error时带来了相当的复杂性。我们再往下探究，官方如何引导GO使用者学习error的设计思想，又如何优雅地处理error。



## 2. 处理error

> ### Don’t just check errors, handle them gracefully  
>
> -- Dave Cheney
>
> https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully

不要只检查错误，要优雅地处理它们。



### 2.1. Errors are just values

错误只是值，或者这样理解——把错误视为值。

这本身有很多故事，这里只做部分举例与阐明结论，想知道详细故事请自行查看相关的GoCon。



Rob Pike 在 2015.1.12 发了一篇博客，回应 2014 年秋季在东京举行的 GoCon 中提及error处理的问题，于是写了一篇博客，名为Errors are values：https://go.dev/blog/errors-are-values。

文末总结：

> 然而，关键的教训是错误是值，Go 编程语言的有足够能力去处理它们。
>
> 使用语言来简化您的错误处理。
>
> 但请记住：无论您做什么，都要检查您的错误！



Dave Cheney 为 2016.4.23 在日本举行的 GoCon 针对error的论述做了记录：

> 我花了很多时间思考在 Go 程序中处理错误的最佳方法。我真的希望有一种单一的方法来进行错误处理，我们可以死记硬背地教所有 Go 程序员，就像我们可以教数学或字母表一样。 
>
> 但是，我得出的结论是，没有单一的方法可以处理错误。相反，我认为 Go 的错误处理可以分为三个核心策略。

我们往下逐一分析这三种核心策略：

- **Sentinel errors** —— 哨兵错误

- **Error types** —— 错误类型化

- **Opaque errors** —— 不透明的错误（"黑盒"错误）



#### 2.1.1. Sentinel errors

> 使用标记值是最不灵活的错误处理策略，因为调用者必须使用相等运算符将结果与预先声明的值进行比较。当您想提供更多上下文时，这会出现问题，因为返回不同的错误会破坏相等性检查。
>
> 甚至像使用`fmt.Errorf`为错误添加一些上下文一样有意义的东西也会破坏调用者的相等性测试。相反，调用者将被迫查看`error`'`Error`方法的输出以查看它是否与特定字符串匹配。



Dave Cheney总结了 **Sentinel errors** 的三项关键特征：

- **永远不要检查 error.Error() 的输出**
- **哨兵错误成为公共 API 的一部分**
- **哨兵错误在两个包之间创建依赖关系**

原文的措辞与例子可能比较抽象，我们就用更接地气的示例，由上到下逐一分析这些特征并展开这些特征带来的问题。



- **永远不要检查 error.Error() 的输出**

为什么如此绝对？我们可以反过来想，什么情况下才要检查error.Error()的输出？

在下面例子中有直观感受

```go
func doSomething() error {
    ...... // 执行了步骤1
    return errors.New("步骤1执行异常")
    
    ...... // 执行了步骤2
    return errors.New("步骤2执行异常")
    
    ...... // 执行了步骤3
    ...... // 以此类推
}
```

现在需要新增一个逻辑，doSomething() ；当步骤1执行异常后，执行逻辑A，当步骤2执行异常后，执行逻辑B....；你只能按如下相似的代码判断：

```go
func check() {
    err := doSomething()
    switch err.Error() {
    case "步骤1执行异常":
        doPlanA()
    case "步骤2执行异常":
        doPlanB()
    case .....
    }
}
```

OK，代码执行起来没问题，但是对于后面维护的人来说真是日了狗了；如果这个 doSomething() 被20+处代码引用，后面的维护人可以轻松修改错误的描述且不引发错误吗？当人有人会提出使用常量去定义错误描述，那又可以看看下面这种情况：

```go
func doSomething() error {
    ...... // 执行了步骤1
    return errors.New("步骤1执行异常")
    
    ...... // 执行了步骤2
    return errors.New("步骤2执行异常")
    
    x := getRandom()
    ...... // 执行了步骤3
    return fmt.Errorf("步骤3执行异常. 随机值: %s", x)
    
    // 以此类推
}
```

请问步骤3异常的情况怎么判断呢？还能用常量吗？然后又会有人提出，可以截取前半段字符串匹配啊。这就只会在错误的路上越走越远，然后暗暗骂一句，GO处理错误的方式真垃圾！



好了，看到这里相信你已经可以理解为什么**永远不要检查 error.Error() 的输出**。



在官方的源码中，可以发现有不少局部使用errors.New()定义的error并作为返参，但这些error不是逻辑错误，大多为系统级别触发的error。也就是说在绝大多数场景下，开发者并不需要根据错误去判断错误内容在代码中if...else...，而是需要根据错误信息来处理程序之外的系统级别问题。

在这种前提下，局部地返回纯文本error是可以被理解的，如下面这个示例：

- **摘自Go\src\log\syslog\syslog_unix.go**

```go
// unixSyslog opens a connection to the syslog daemon running on the
// local machine using a Unix domain socket.
func unixSyslog() (conn serverConn, err error) {
	logTypes := []string{"unixgram", "unix"}
	logPaths := []string{"/dev/log", "/var/run/syslog", "/var/run/log"}
	for _, network := range logTypes {
		for _, path := range logPaths {
			conn, err := net.Dial(network, path)
			if err == nil {
				return &netConn{conn: conn, local: true}, nil
			}
		}
	}
	return nil, errors.New("Unix syslog delivery error")  // <-- 如果出错，只需要知道这个func内部发生错误，并知道错误信息即可，并不需要根据错误信息来进行逻辑分支处理。
}
```



那如果前同事写了类似上文 **doSomething()** 的代码，怎么解困呢？

我们来看下GO官方包内部是怎么处理这种问题的：

- **摘自Go\src\io\io.go**

```go
var EOF = errors.New("EOF")
```

- **摘自Go\src\fmt\scan.go**

```go
func (r *stringReader) Read(b []byte) (n int, err error) {
	n = copy(b, *r)
	*r = (*r)[n:]
	if n == 0 {
		err = io.EOF  // 注意这里，返回了一个io.EOF错误，该使用了标记值错误
	}
	return
}
```

- **摘自Go\src\io\io.go，当我们判断这个EOF时，需要判断这个error是否为 io.EOF**

```go
func ReadAll(r Reader) ([]byte, error) {
	......
	for {
		......
		if err != nil {
			if err == EOF {  // 这里直接使用==运算符对比error是否为io.EOF
				err = nil
			}
			return b, err
		}
	}
}
```



看到这里相信你已经可以**举一反一**，我们可以把整个错误提取出来作为全局变量提供外部包使用

```go
Step1Err := errors.New("步骤1执行异常")
Step2Err := errors.New("步骤2执行异常")

func doSomething() error {
    ...... // 执行了步骤1
    return Step1Err
    
    ...... // 执行了步骤2
    return Step2Err
    
    ...... // 执行了步骤3
    ...... // 以此类推
}
```



但这种全局化的做法同时带来了另外一些问题：

- **一种错误描述对应一个全局变量，并定义在包头，增加了包的"厚度"，显得代码比较冗余；**
- **其他包想使用该包内的方法时，需要绑定使用该包的错误，如果该错误的传递层次较多，那么最外部文件中还需要import该包；如果一个逻辑包含了多个使用sentinel errors的包，那么需要把它们都import一遍进行判断，这会使得需要import的包非常多；**

以上两点一一对应了Dave Cheney总结 **Sentinel errors** 的后两种特征：

- **哨兵错误成为公共 API 的一部分**（这个特征还有另外一个原因：使用了非实现error接口的值作为错误，这种做法很奇葩，违反error接口的根本设计理念，忽略）
- **哨兵错误在两个包之间创建依赖关系**



最终Dave Cheney也有自己对 **Sentinel errors** 的使用结论：

> ## Conclusion: avoid sentinel errors
>
> So, my advice is to avoid using sentinel error values in the code you write. There are a few cases where they are used in the standard library, but this is not a pattern that you should emulate.
>
> If someone asks you to export an error value from your package, you should politely decline and instead suggest an alternative method, such as the ones I will discuss next.

> 结论：避免哨兵错误 
>
> 因此，我的建议是避免在您编写的代码中使用哨兵错误。在标准库中使用它们的情况很少，但这不是您应该效仿的模式。 
>
> 如果有人要求你从你的包中导出一个错误值，你应该礼貌地拒绝，而是建议一种替代方法，比如我将在下面讨论的方法。



认真阅读本文的小可爱一定还记得上文  **doSomething()** 示例中还存在 带变参 的错误没有得到解决，这种错误是没办法通过全局变量的方式替换的。需要使用Dave Cheney提到的第二种核心策略：**Error types**。



#### 2.1.2. Error types

> 错误类型是指实现错误接口的类型。

它的一个重要的好处是，类型中除了异常文本描述之外，还可以附带其他字段，从而提供额外的信息。

我们以官方包的**PathError**举例：

- **摘自Go\src\io\fs\fs.go**

```go
// PathError records an error and the operation and file path that caused it.
type PathError struct {
	Op   string
	Path string
	Err  error
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }

func (e *PathError) Unwrap() error { return e.Err }

// Timeout reports whether this error represents a timeout.
func (e *PathError) Timeout() bool {
	t, ok := e.Err.(interface{ Timeout() bool })
	return ok && t.Timeout()
}
```

**PathError** 能够记录出错时的文件路径与操作类型。外层业务逻辑可以使用类型断言来判断错误：

- **摘自Go\src\cmd\go\internal\lockedfile\internal\filelock\filelock.go**

```go
// underlyingError returns the underlying error for known os error types.
func underlyingError(err error) error {
	switch err := err.(type) {
	case *fs.PathError:
		return err.Err
	case *os.LinkError:
		return err.Err
	case *os.SyscallError:
		return err.Err
	}
	return err
}
```



但这种也存在以下问题：

- **该接口的所有实现者都需要依赖于定义错误类型的包。**
- **创建与调用时都是与类型强耦合，降低了API的灵活性。**（这句话可能描述得有点抽象，大白话是：如**PathError**一样的结构虽然实现了error接口，但其创建时仍然是直接返回指针对象，而不是一个error接口，还是对开发者有心智负担）

最终Dave Cheney也有自己对 **Error types** 的使用结论：

>## Conclusion: avoid error types
>
>While error types are better than sentinel error values, because they can capture more context about what went wrong, error types share many of the problems of error values.
>
>So again my advice is to avoid error types, or at least, avoid making them part of your public API.
>
>结论：避免错误类型
>
>虽然错误类型比哨兵错误更好，因为它们可以捕获更多关于出错的上下文，但错误类型仍然携带着需要导包等问题。
>
>所以我的建议是避免错误类型，或者至少避免将它们作为公共 API 的一部分。



最后Dave Cheney提出了一种终极方案：**Opaque errors**，下面我们来分析一下**Opaque errors**到底有多神。



#### 2.1.3. Opaque errors

> 现在我们来到第三类错误处理。在我看来，这是最灵活的错误处理策略，因为它需要您的代码和调用者之间的耦合最少。
>
> 我将这种风格称为不透明错误处理，因为当您知道发生了错误时，您无法查看错误内部。作为调用者，您所知道的关于操作结果的所有信息都是它有效，或者它没有。
>
> 这就是不透明错误处理的全部内容——只需返回错误而不假设其内容。如果您采用这种立场，那么错误处理作为调试辅助工具会变得更加有用。

```go
import “github.com/quux/bar”

func fn() error {
        x, err := bar.Foo()
        if err != nil {
                return err
        }
        // use x
}
```

这就是 **Opaque errors** 的策略，作为调用方，只需要知道 Foo 是否正常执行，一旦err不为空，直接返回错误；

但在真实场景中，调用的三方库会非常多，复杂的业务中难免会遇到根据err来走不同逻辑分支的情形。这时候，光有Foo已经不够用，需要一种方法获知错误是否具备某种行为，而不需要判断错误类型是什么：

```go
type temporary interface {
        Temporary() bool
}
 
// IsTemporary returns true if err is temporary.
func IsTemporary(err error) bool {
        te, ok := err.(temporary)
        return ok && te.Temporary()
}
```

这么做的好处是：

- **业务层不需要import具体类型的包；**
- **业务层不需要知道error的内部类型，再进行具体错误原因的对比；而是直接判断接口是否具备某种行为；**

这种设计方式，也被称为**面向接口编程**。



### 2.2. Don’t just check errors, handle them gracefully

检查并优雅地处理错误，这是本篇的重点内容。



#### 2.2.1. 避免冗长的错误处理代码

Dave Cheney在[[3]](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)中给出一个例子

```go
func AuthenticateRequest(r *Request) error {
        err := authenticate(r.User)
        if err != nil {
                return err
        }
        return nil
}
```

很明显代码可以简化到如下：

```go
func AuthenticateRequest(r *Request) error {
        return authenticate(r.User)
}
```

这是最基本的应对措施。



#### 2.2.2. 携带堆栈信息

尽管代码足够简约，返回的error信息还是太贫瘠，仅包含一个字符串形式的信息，例如它是：**"No such file or directory"**。

当错误产生在代码层次较深时，我们根本不知道这个错误来自于哪里？？？

如果错误中能携带堆栈信息，就能解决这一困境。

这就不得不说一个具有启蒙意义的第三方包：**"github.com/pkg/errors"**，一个携带有堆栈信息的错误管理包。

```go
// New 使用提供的消息返回错误。
// New 还会记录调用时的堆栈跟踪。
func New(message string) error {
	return &fundamental{
		msg:   message,
		stack: callers(),
	}
}

// Errorf 根据格式说明符进行格式化，并将字符串作为满足错误的值返回。
// Errorf 还会记录调用时的堆栈跟踪。
func Errorf(format string, args ...interface{}) error {
	return &fundamental{
		msg:   fmt.Sprintf(format, args...),
		stack: callers(),
	}
}

// fundamental是一个错误，它有一个消息和一个堆栈，但没有调用者。
type fundamental struct {
	msg string
	*stack
}
```

**fundamental** 结构包含堆栈信息 ***stack** 与自定义错误描述 **msg**，只要使用常规的 **New**() 与 **Errorf()** 新建error，就能记录当前位置的堆栈。

除此之外，**fundamental** 还提供了 **Format()**方法，实现了 **Formatter** 接口。使用 **fmt.Printf("%+v", (*fundamental类型的error)** 时可以输出详细的堆栈信息。

- **摘自Go\src\fmt\print.go**

```go
// Formatter is implemented by any value that has a Format method.
// The implementation controls how State and rune are interpreted,
// and may call Sprint(f) or Fprint(f) etc. to generate its output.
type Formatter interface {
   Format(f State, verb rune)
}
```

- **摘自pkg\mod\github.com\pkg\errors@v0.9.1\errors.go**

```go
func (f *fundamental) Format(s fmt.State, verb rune) {
	switch verb {
	case 'v':
		if s.Flag('+') {  // <---可以重点看这里的逻辑，进入这个分支可以输出详细的堆栈。
			io.WriteString(s, f.msg)
			f.stack.Format(s, verb)
			return
		}
		fallthrough
	case 's':
		io.WriteString(s, f.msg)
	case 'q':
		fmt.Fprintf(s, "%q", f.msg)
	}
}
```

效果如下：

```tex
readfile.go:27: could not read config
readfile.go:14: open failed
open /Users/dfc/.settings.xml: no such file or directory
```

这对于定位问题来说就简单多了，不需要反复重现或debug，就能直接知道问题的根源。



#### 2.2.3. 嵌套error链

在1.2小节我们提到过嵌套错误的问题。当业务流程中，需要携带的错误不止一个时，可以用嵌套error的方式实现。

**"github.com/pkg/errors"**也有一套**"包装"**与**"解包"**的方法

- **摘自pkg\mod\github.com\pkg\errors@v0.9.1\errors.go**

```go
// WithStack 在调用 WithStack 时使用堆栈跟踪注释 err。
// 如果 err 为 nil，WithStack 返回 nil。
func WithStack(err error) error {
	if err == nil {
		return nil
	}
	return &withStack{
		err,
		callers(),
	}
}

type withStack struct {
	error
	*stack
}

func (w *withStack) Cause() error { return w.error }

// Unwrap 为 Go 1.13 错误链提供兼容性。
func (w *withStack) Unwrap() error { return w.error }
```

类似地，**withStack** 与 **fundamental** 提供了记录堆栈的能力，但扩展了一套 **Cause()** 与 **WithStack()** 方法，意于装载任意的错误，当必要时可以通过解包获取最内部的错误，或对比整个嵌套链的错误包含有业务场景关注的错误。

- **摘自pkg\mod\github.com\pkg\errors@v0.9.1\errors.go**

```go
// 如果错误没有实现Cause，则返回原来的错误。如果错误为 nil，则将返回 nil 而无需进一步调查。
func Cause(err error) error {
	type causer interface {
		Cause() error
	}

	for err != nil {
		cause, ok := err.(causer)
		if !ok {
			break
		}
		err = cause.Cause()
	}
	return err
}
```



这个 **Cause()** 方法是否与1.2小节中提到的 **Unwrap()** 很相像？其实在Go1.13版本前，是没有Unwrap()方法的，我们可以追溯到Go1.12最后一个发布版本的errors官方包代码，errors包下只有一个errors.go文件及其单元测试文件：

- 摘自 https://github.com/golang/go/blob/go1.12.17/src/errors/errors.go

  ```go
  // Copyright 2011 The Go Authors. All rights reserved.
  // Use of this source code is governed by a BSD-style
  // license that can be found in the LICENSE file.
  
  // Package errors implements functions to manipulate errors.
  package errors
  
  // New returns an error that formats as the given text.
  func New(text string) error {
  	return &errorString{text}
  }
  
  // errorString is a trivial implementation of error.
  type errorString struct {
  	s string
  }
  
  func (e *errorString) Error() string {
  	return e.s
  }
  ```

在Go1.13的第一个发布版本时，errors包下加入了wrap.go文件。**Unwrap()** 正式替代了 Cause() 方法。

有兴趣可以看：https://github.com/golang/go/blob/go1.13/src/errors/wrap.go



### 2.3. Only handle errors once

> I want to mention that you should only handle errors once. Handling an error means inspecting the error value, and making a decision.
>
> 我想提一下，你应该只处理一次错误。处理错误意味着检查错误值并做出决定。



#### 2.3.1. 妥善处理每一个错误 

我们来看一下没处理错误的例子：

```go
func (d *Data) ConvData(b []byte) {
    ...... // 其他逻辑
    ......
    json.Unmarshal(b, d)   // <---此处直接吞掉了Unmarshal()的返回值error
    ......
    ......
}
```

```go
func Unmarshal(data []byte, v interface{}) error {
	// 检查格式是否正确。
	// 避免在发现 JSON 语法错误之前填写半个数据结构。
	var d decodeState
	err := checkValid(data, &d.scan)
	if err != nil {
		return err
	}

	d.init(data)
	return d.unmarshal(v)
}
```

这是一个**非常不好的习惯**，当**b**的内容不是一个合法的json结构时，**Unmarshal()** 的错误没有被外层感知，调用 **ConvData()** 的开发者很可能认为内部不会发生错误，且在**d**在必须非空数据的业务场景时引发了业务错误，并且难以排查。

正如 Rob Pike 所说：**"But remember: Whatever you do, always check your errors!"**



但万事都有极端情况，当你非常确认该错误可以忽略时，请你用 **_** (下划线显式说明忽略该返回值)，起到一定的提示作用。

（goland编辑器针对隐式省略返回值的代码行也会进行高亮提示）：

```go
func (d *Data) ConvData(b []byte) {
    _ = json.Unmarshal(b, d)
}
```



#### 2.3.2. 只处理错误一次

针对一个错误做出多个决定也是有问题的。

```go
func Write(w io.Writer, buf []byte) error {
        _, err := w.Write(buf)
        if err != nil {
                // 带注释的错误进入日志文件
                log.Println("unable to write:", err)
            
                // 未注释的错误返回给调用者
                return err
        }
        return nil
}
```

在此示例中，如果在 `Write` 期间发生错误，会将一行写入日志文件，记录发生错误的文件和行，并将错误也返回给上层。上层调用者可能也会记录日志并返回错误给上上层。最终，会在日志文件中得到一堆重复的行，但在最外层，仅仅得到一个没有任何上下文的原始错误。这样的日志会给排查问题的人带来疑惑，到底发生了几次错误呢？

因此推荐只处理错误一次。



### 2.4. 总结

最后，Dave Cheney给出了他的总结：

> In conclusion, errors are part of your package’s public API, treat them with as much care as you would any other part of your public API.
>
> For maximum flexibility I recommend that you try to treat all errors as opaque. In the situations where you cannot do that, assert errors for behaviour, not type or value.
>
> Minimise the number of sentinel error values in your program and convert errors to opaque errors by wrapping them with `errors.Wrap` as soon as they occur.
>
> Finally, use `errors.Cause` to recover the underlying error if you need to inspect it.

这些总结也成为了处理错误和设计自定义错误结构的准则：

- **首先，错误隶属于包中 公共API 的一部分，请像对待公共 API 的任何其他部分一样小心对待它们。**
- **为了获得最大的灵活性，建议尝试将所有错误视为不透明的。在你不能这样做的情况下，断言错误的行为，而不是类型或值。**

- **最大限度地减少程序中 Sentinel errors 的数量，并通过在错误发生时，立即使用`errors.Wrap`将其包装起来，将错误转换为 opaque errors。**

- **最后，如果您需要检查错误原因，请使用`errors.Cause`来恢复底层错误。**

最终，结合业务场景，目前已经诞生出非常多的自定义error结构方案，下面我们着重分析一下kit-go中的实现。



## 3. kiterror与kitcode

阅读了第2章之后，都应理解应该怎么去处理error。

然而理想是丰满的，现实是骨感的。GO经过了多次迭代，以及庞大生态产生的error实现方式是多样的，而集成优秀组件必然不能因为error处理麻烦而却步。

因此 **kit-go** 中的 **kiterror** 与 **kitcode** 包诞生了。

遵循Dave Cheney的设计思想，采用 **Error types** 与 **Opaque errors** 相兼容的方式，将多样的第三方包error消化为内部error，调用方需要知道错误属于具体哪种类型时，可以直接获取，也可以忽略类型本身而进行行为判断。这种兼容设计已完全满足复杂的业务需要，大大减少了开发者的心智负担。

下面我们来看下kiterror具备哪些能力



### 3.1. 携带错误产生位置的堆栈信息

### 3.2. 携带原始错误，支持常见错误类型自动转换为内部类型

​		不知道方法内部是否已经转换过error，没关系，直接用kiterror.Convert()方法即可！

### 3.3. 自动映射错误类型

​		一种错误码，自适应HTTP/gRPC接口响应

​		接口模式设计，支持自定义扩展

### 3.4. 携带业务子错误类型

### 3.5. 支持返回主错误类型及子错误类型，及支持直接判断错误类型

### 3.6. 支持直接判断错误的行为

### 3.7. 接口访问中间件对错误进行一次性处理





## 参考文献

1. https://golang.google.cn/doc/faq [GO官方FAQ]
2. https://studygolang.com/articles/1674 [Russ Cox回应“Why I'm not leaving python for go”]

3. https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully [Dave Cheney阐述如何优雅处理error]

4. https://go.dev/blog/errors-are-values [Rob Pike阐述错误处理思想]

5. https://github.com/golang/go [go语言官方开源库]
