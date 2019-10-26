---
title: 用php实现curl的data-binary
date: 2017-10-01 20:45:50
tags: [php,influxdb]
---
如果我想往influxdb中插入一条数据，教程告诉我可以这样：
```sh
curl -i -XPOST 'http://127.0.0.1:8086/write?db=metrics' -u admin:admin --data-binary 'test,host=localhost count=1'
```

但是现在我需要用php来实现。经过查阅网上资料，有人说可以先将`test,host=localhost count=1`转为stream，然后php进行curl的时候设置`CURLOPT_INFILE`、`CURLOPT_INFILESIZE`、`CURLOPT_UPLOAD`。这样操作下来虽然influxb虽然返回了204，但是数据并没有正确插入。

最后我使用了以下的代码实现了功能：
```php
$ch = curl_init();//init
curl_setopt($ch, CURLOPT_TIMEOUT, 1); //一秒超时
curl_setopt($ch, CURLOPT_URL, "http://127.0.0.1:8086/write?db=metrics");
curl_setopt($ch, CURLOPT_BINARYTRANSFER, TRUE);//二进制传输
curl_setopt($ch, CURLOPT_USERPWD, '123');//权限认证
curl_setopt($ch, CURLOPT_POSTFIELDS, 'test,host=localhost count=1');//写操作只能用post
curl_exec($ch);
curl_close($ch);
```