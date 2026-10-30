# 编译产物

| 文件 | 平台 | 说明 |
|---|---|---|
| `zteonu-linux-amd64` | Linux x86-64 | Debian 上原生编译，statically-ish 动态链接 glibc |

## 校验

```bash
file zteonu-linux-amd64
# ELF 64-bit LSB executable, x86-64, statically/dynamically linked, stripped

chmod +x zteonu-linux-amd64
./zteonu-linux-amd64 version
```

## 重新生成

```bash
cd ../src
go build -trimpath -ldflags '-s -w' -o ../bin/zteonu-linux-amd64 .
```

源码与编译产物的对应关系：本二进制由 `src/` 中的源码编译而来（Go 1.27.3，linux/amd64）。
