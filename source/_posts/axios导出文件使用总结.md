---
title: axios导出文件使用总结
date: 2019-11-21 09:13:59
tags:
  - Frontend
  - axios
categories:
  - Frontend
---



这篇文章主要是最近几天在工作中遇到的导出文件总结

之前团队在导出文件的时候，由于使用axios导出之后，非文本文件都会乱码，所以采用另外一种方式，但是该方式存在严重缺陷，在经过调研之后，发现axios本身是支持的，所以重新采用通过axios来导出

<!--more-->

# axios导出文件使用总结

## 背景

之前在导出的时候，前端的同学直接使用axios，发现导出的时候，文件乱码而且没有办法获取到文件的内容，所以采用`window.location.href=XXX`直接指向下载地址

如：`window.location.href=http://localhost/test/export`

这种方式非常直观，直接通过浏览器来访问下载，但同时缺点也非常明显

1. 仅支持get方式
2. 无法携带头信息，尤其是系统需要权限访问的时候，只能开放该接口，从而引入不必要的风险
3. 当参数信息多的时候，拼接在地址后面繁琐

尤其是在对接过程中，有一个接口本身的参数信息非常多，而且某些参数本身存在特殊字符，直接通过URL的形式，就需要先将其进行编码，然后接口收到之后，再解码回来，再组装成对象，操作繁琐，而且不通用

故决定调研新的方案

## 解决方案

经过一番调研之后，发现axios本身是支持文件下载的(与文件格式、类型无关)，只是需要额外的配置

axios在发起请求的时候，有一个参数：`responseType`，该类型用于设置axios如何解析HTTP请求返回的数据，默认情况下，该值为`string`，从axios的代码可以看出，如下所示

```js
export interface AxiosRequestConfig {
  ....
  responseType?: string;
  ...
}
```

如此就可以看出前面通过axios下载非文本文件乱码的原因了，返回的数据流被axios解析为string类型的，而此时下载的文件本身是有其特定的类型如PDF、Excel这一类型，因此，在进行导出文件的时候，需要将返回类型配置为：`Blob`

> Blob是一个代表原始数据的不可变类，JS中的File是Blob派生出来的代表文件系统数据的对象
>
> 通过Blob对象，可以直接操作文件数据，此外，Blob对象可以通过URL.createObjectURL(XX)转换为URL，从而将Blob对象转换为一个地址，从而进行下载等操作

示例代码如下

```js
axios({
      url: root + url,
      data: param,
      method: method,
      responseType: 'blob',
      headers: {
        token: StorageUtils.getItem('token')
      }
    }).then((response) => {
      if (response.status !== 200) {
        // 错误处理
        return
      }
      // response解析见下面分析
      const url = window.URL.createObjectURL(response.data)
      
      // 将该url包装成一个连接，并且模拟点击，从而实现下载的功能
      const link = document.createElement('a')
      link.href = url
      document.body.appendChild(link);
      link.click();
      // 释放资源
      window.URL.revokeObjectURL(url);
      document.body.removeChild(link)
  });
```

> response

上面的代码中直接使用`response.data`作为`createObjectURL()`的参数，原因在于在axios中`response`的`data`属性本身代表的就是返回的数据，当然，在配置为`blob`类型之后，`data`就是一个`Blob`对象了

![axios response.png](http://ww1.sinaimg.cn/large/b162e9f4gy1g95dwei3mpj20nu0dxwfo.jpg)

到这里，通过axios导出文件的处理方式就分析清楚了

## 总结

在下载文件的时候，需要将`responseType`配置为`blob`，在获取到下载的数据之后，将其通过`URL.createObjectURL`转换为一个URL，然后通过`<a>`标签或者我们前面提到的`window.location.href=XXX`等能够通过URL发起GET请求操作的工具将下载完毕的字节流保留下来即可

所以说，碰到问题的时候，还是应该先调研、分析，而不是绕开问题，无论怎么绕开，最后还是有很大的概率会碰到该问题，毕竟问题本身没有解决，只是被绕开了，共勉