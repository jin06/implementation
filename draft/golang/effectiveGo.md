
## 介绍
原文地址 https://go.dev/doc/effective_go 


#### Redeclaration and reassignment
重新声明和重新分配

```golang
f, err := os.Open(name)
d, err := f.Stat()
```

err在第一行已经声明过，第二行不会重新声明err，只是重新分配值