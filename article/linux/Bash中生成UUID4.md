使用`uuidgen`命令。这是系统组件的一部分，大部分发行版默认都有。

读取`/proc/sys/kernel/random/uuid`，每次都会生成新的。`/proc/sys/kernel/random/boot_id`生成的每次系统启动后会刷新，其他时间在第一次访问后生成固定的uuid，后面都返回这个。

用openssl，第二快的：

```bash
uuid=$(openssl rand -hex 16)
echo ${uuid:0:8}-${uuid:8:4}-${uuid:12:4}-${uuid:16:4}-${uuid:20:12}
```

性能测试：用hyperfine测试，warmup为5次：

```
Summary
  cat /proc/sys/kernel/random/uuid ran
    1.01 ± 0.53 times faster than uuidgen
    2.95 ± 1.18 times faster than uuid=$(openssl rand -hex 16);u=${uuid:0:8}-${uuid:8:4}-${uuid:12:4}-${uuid:16:4}-${uuid:20:12}
```

openssl最慢，proc和uuidgen速度差不多。

如果有python3.12+：`python -m uuid`
