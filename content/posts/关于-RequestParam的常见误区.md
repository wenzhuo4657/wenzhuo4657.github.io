---
title: 关于@RequestParam的常见误区
date: 2025-03-29 15:35:29
layout : 'singlet'
---













首先说结论，@RequestParam不仅可以结束queryString的参数，还可以接收表单数据。







## 关于http载荷的区分



- queryString

  `http://example.com/path?key1=value1&key2=value2&key3=value3`

  无论是get还是post，他在url上的表现都是上述，而关于get和post的区别主要显示在http报文上字段填充。

  ```
  GET /search?q=springboot&page=1 HTTP/1.1
  Host: example.com
  Accept: application/json
  ```

  ```
  路径和请求体可以共存，但是他们都统一可以使用作为queryString参数进行键值对解析。
  POST /submit?id=123 HTTP/1.1
  Host: example.com
  Content-Type: application/json
  
  {"name": "Alice"}
  ```

  

事实上，queryStirng参数的核心在于键值，而不是位于请求体、路径。

- multipart/form-data

  文件上传

- application/x-www-form-urlencoded

  表单文件