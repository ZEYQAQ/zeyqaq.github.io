Lint（代码静态检查）就是：不运行程序，直接检查代码，提前找出潜在问题和不符合规范的写法。 可以理解为代码的 “自动质检”。主要涉及变量名的规范、冗余定义、无用导入和不推荐的API用法检出。

在go中，常用golangci-lint，它整合多种检查器。进入包含go.mod的项目根目录运行：

```
golangci-lint run ./...
```

就会自动检查并输出类似：

```
main.go:12:14: Error return value of `os.WriteFile` is not checked (errcheck)
```

最优解决办法当然是修复所有报错

```
golangci-lint run --fix ./...

# 检查自动修改了什么
git diff

# 修复后重新检查、测试
golangci-lint run ./...
go test ./...
```

但是如果有豁免，或者不想因为报错而卡发布，可以按照函数范围、git路径忽略错误。

注意，Lint仅是静态分析，并不能代替测试，也不能保证代码没有bug。