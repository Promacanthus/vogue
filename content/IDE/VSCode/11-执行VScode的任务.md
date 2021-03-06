---
title: 11-执行VScode的任务
date: 2020-04-14T10:09:14.278627+08:00
draft: false
---

从命令面板中调用`Task: Configure Task Runner`。

这将在工作空间（`.vscode`）文件夹中级工创建`task.json`文件，用下面的内容替换自动生成的`JSON`文件中的内容。

```JSON
{
    "version": "0.1.0",
    "command": "go",
    "isShellCommand": true,
    "showOutput": "silent",
    "tasks": [
        {
            "taskName": "install",
            "args": [ "-v", "./..."],
            "isBuildCommand": true
        },
        {
            "taskName": "test",
            "args": [ "-v", "./..."],
            "isTestCommand": true
        }
    ]
}
{
```

```json
{
    "version": "2.0.0",
    "type": "shell",
    "echoCommand": true,
    "cwd": "${workspaceFolder}",
    "tasks": [
        {
            "label": "rungo",
            "command": "go run ${file}",
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
    ]
}
```

- 通过 `ctrl + shift + b` 来运行 `go install -v ./...` 并且会在输出窗口中返回结果
- 通过  `ctrl + shift + t` 来运行 `go test -v ./...` 并且会在输出窗口中返回结果

可以使用同样的技巧来调用其他构建和测试工具。例如，在`makefile`中定义构建过程，则调用`makefile`目标，或者调用类似`go generate`之类的工具。

有关在VSCode中配置任务的更多信息，查看[这里](https://code.visualstudio.com/docs/editor/tasks.)。

可以定义特殊的仅运行某些测试的任务：

```json
{
    "version": "0.1.0",
    "command": "go",
    "isShellCommand": true,
    "showOutput": "silent",
    "suppressTaskName": true,
    "tasks": [
        {
            "taskName": "Full test suite",
            "args": [ "test", "v", "./..."]
        },
        {
            "taskName": "Blog User test suite",
            "args": [ "test", "./...", "-test.v", "-test.run", "User"]
        }
    ]
}
```

该任务将会用`User`中的名字来运行所有的测试。
