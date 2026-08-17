# Python http.server 踩坑：end_headers() 忘加括号导致 HTTP 响应异常

> **日期**: 2026-08-06  
> **分类**: 踩坑记录  
> **标签**: 无  
> **来源**: hermes

---

## 背景/问题

用 Python 标准库 `http.server` 写了一个简单的 Web 服务，启动后访问页面一直加载不出来或返回异常内容。代码看起来完全没问题——每个路由都有 `send_response`、`send_header`、`end_headers`，逻辑是对的。

## 原因分析

问题出在一个极容易忽略的细节上：**`self.end_headers` 少写了一对括号 `()`**。

```python
# 错误写法——不会报错，但 HTTP 响应头不会正确结束
self.end_headers
self.wfile.write(body)

# 正确写法——加上括号才是方法调用
self.end_headers()
self.wfile.write(body)
```

`end_headers` 是 `BaseHTTPRequestHandler` 的一个**方法**，不是属性。写 `self.end_headers`（不带括号）只是一个**引用表达式**，Python 会计算这个表达式然后丢弃结果——**什么都不会发生**。

这意味着 HTTP 响应头没有被正确终止，浏览器收到的数据格式是错乱的（头和正文混在一起），所以页面无法正常渲染。

## 为什么这个 bug 特别坑

1. **Python 不会报错**——`self.end_headers` 是合法的表达式（引用一个方法对象）
2. **服务能正常启动**——没有语法错误
3. **日志看起来正常**——服务端访问日志一切如常
4. **只有客户端才发现问题**——浏览器显示空白或乱码

## 解决方案

全局搜索 `end_headers`，确保每一处都带了括号：

```python
class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/":
            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=utf-8")
            body = self._render_index().encode("utf-8")
            self.send_header("Content-Length", str(len(body)))
            self.end_headers()    # 别忘了 () ！！！
            self.wfile.write(body)
        elif self.path == "/health":
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            body = b'{"status":"ok"}'
            self.send_header("Content-Length", str(len(body)))
            self.end_headers()    # 每个分支都要检查
            self.wfile.write(body)
```

## 避坑提示

- **Python 方法调用必须加括号**——`end_headers` 特别容易中招，因为它看起来像"结束头部"这个动作，直觉上不像个函数
- **类似的还有 `self.connection.close`**——也是方法，不带括号不执行
- **调试技巧**：如果页面加载异常但服务没报错，用 `curl -v http://localhost:8080/` 看原始 HTTP 响应，如果头和正文混在一起，大概率就是 `end_headers()` 没调用
- **另一个常见坑**：`wfile.write()` 需要的是 `bytes` 不是 `str`，字符串要 `.encode("utf-8")` 再写入
