+++
date = '2025-06-23T22:00:00+08:00'
draft = true
title = 'Golang Json 反序列化漏洞'
+++

众所周知，Java 的 f***json 反序列化漏洞养活了国内大半的安全从业人员。

作为一个前 Java 现 Go选手，Go 中容易写出来的安全漏洞的代码我也搞过几十个案例，最近我又在网络上发现了一个 Go 中 json 反序列化的非预期行为可以绕过身份验证、规避授权控制的案例。

举个例子。

假设 用户 结构体如下

```go
type User struct {  
    Username string `json:"username,omitempty"`  
    Password string `json:"password,omitempty"`  
    IsAdmin  bool   `json:"-,omitempty"`  
}
```

我们希望在用户注册的接口，反序列化的时候忽略 `IsAdmin` 这个字段，将会默认设置为 false。

不懂 Go 语言的也没关系，让AI来解释一下。


![](/images/ai-1.png)


让我们来写一段代码来模拟注册用户的逻辑。

```go
package main  
  
import (  
    "encoding/json"  
    "fmt"    "log"
)  
  
type User struct {  
    Username string `json:"username,omitempty"`  
    Password string `json:"password,omitempty"`  
    IsAdmin  bool   `json:"-,omitempty"`  
}  
  
func register() {  
    // 模拟用户提交的 JSON 字符串
    var input = `{  
    "username": "dushixiang",    
    "password": "password",    
    "-": true
}`  
    // 解析JSON字符串到User结构体  
    var user User  
    err := json.Unmarshal([]byte(input), &user)  
    if err != nil {  
       log.Fatal(err)  
    }  
    // 模拟保存用户到数据库  
    // ...  
  
    fmt.Printf("%+v", user)  
}  
  
func main() {  
    register()  
}
```

您猜怎么着？这段代码将会输出

```shell
{Username:dushixiang Password:password IsAdmin:true}
```

你问我这怎么了？

用户直接注册成你系统的管理员了，你还问我怎么了。

你问我这是什么问题导致的？

背后的原因让人暖心，AI 说对了一半。

~~`-` 和 `omitempty` 确实会冲突，但是结果是  `omitempty`  会让 `-` 作为  `IsAdmin`  字段的 json key 参与反序列化，最终导致了 `IsAdmin` 被用户提交的内容赋值了。~~

~~解决方案也很简单，把 `,omitempty` 移除就好了。~~

经过其他大佬的指点，实际上是 `json: "-"` 和 `json: "-,"` 的行为不一致，加上逗号就是把 `-` 当作 json key。

解决方案也很简单，把 `,` 移除就好了。