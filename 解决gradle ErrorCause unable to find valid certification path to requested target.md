gradle坑

# 解决gradle "Error:Cause: unable to find valid certification path to requested target"

这是由于app不信任我们的证书导致https会话失败。
将`jcenter()`修改为：

```
jcenter{
            url 'http://jcenter.bintray.com'
        }
```

