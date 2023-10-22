---
layout: post
title: "qemu的hbitmap数据结构介绍"
date:   2022-4-30
tags: [qemu]
comments: true
author: Reynold
notice: 如需转载请通过github联系本人，否则禁止转载

---

# python代码中捕捉requests网络请求

```python
import sys
import requests

def trace_calls(frame, event, arg):
    if event != 'call':
        return
    co = frame.f_code
    arr = ['url','params','data','body']
    func_name = co.co_name
    if func_name == 'request':
        for key, value in frame.f_locals.items():
            if key not in arr: continue
            if type(value) is bytes: value = value.decode('unicode-escape')
            print(f"    {key} = {value}")

sys.settrace(trace_calls)
```

注意使用value.decode('unicode-escape')进行解码，否则无法打印中文