+++
date = '2024-12-19T00:00:00+08:00'
draft = true
title = 'Java 竟然可以执行注释了？'
+++

昨天看到了一个很有意思的小魔法，使用 Java 来执行注释，当时我就惊了，这是什么套路？我写了好几年的Java也没见过啊。

今天就抽空测试了一下，代码如下。

```java
public class Main {  
    public static void main(String[] args) {  
        /* comment start  
        \u002A\u002F\u0020\u0053\u0079\u0073\u0074\u0065\u006D\u002E\u006F\u0075\u0074\u002E\u0070\u0072\u0069\u006E\u0074\u006C\u006E\u0028\u0022\u0064\u0075\u0073\u0068\u0069\u0078\u0069\u0061\u006E\u0067\u0020\u006E\u0062\u0022\u0029\u003B\u0020\u002F\u002A  
        comment ends */    
    }  
}
```

看起来就是一段普通的Java代码吧，代码块内部前后都有注释符，理论上是不会输出任何内容的。

但是当你执行之后就会发现，是有输出的。

```shell
dushixiang nb
```

为什么会这样呢？

答案是因为 Java 在编译代码的时候，首先会把 unicode 字符转换为普通的字符串，具体解释可以去看 https://docs.oracle.com/javase/specs/jls/se17/html/jls-3.html#jls-3.3 。

转换之后的代码就变成了这样。

```java
public class Main {
    public static void main(String[] args) {
        /* comment start
        */ System.out.println("dushixiang nb"); /*
        comment ends */
    }
}
```

就像是 xss 一样，把前后的注释符号都闭合了，魔法也就这样产生了。