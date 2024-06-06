---
slug: tsxx
title: tsxx [设计与实现]
authors: [yang]
tags: [node, typescript, tsx, hmr]
---

### 介绍

[tsxx](https://github.com/sia-fl/tsxx) 是一个 ts js 文件执行脚本，
基于 tsx 开发。能直接运行 ts js 文件，支持 hmr。

安装方式非常简单，只需要全局安装 tsxx 即可。

```bash
npm install -g tsxx
```

然后就可以直接运行 ts js 文件了。

```bash
tsxx ./src/index.ts
```

😇 修改文件，会发现他会自动刷新。

### 和 nodemon 的区别

nodejs 生态中有热重启工具 nodemon，但是 nodemon 会重启整个 node 进程，
而 tsxx 只会重启当前文件，这样可以保持当前文件的状态。

假定我们有文件 a、b、c。
- a 依赖 b
- b 依赖 c

我们修改 c 时，nodemon 会重启整个进程，而 tsxx 只会重启 c，这样可以保持 a、b 的状态。

```
+---+       +---+       +---+
| a | ----> | b | ----> | c |
+---+       +---+       +---+

nodemon 会重启整个进程时：

+---+       +---+       +---+
| a | ----> | b | ----> | c |
+---+       +---+       +---+
↑           ↑           ↑
|           |           |
+-----------+-----------+
|
重启

tsxx 只会重启 c 时：

+---+       +---+       +---+
| a | ----> | b | ----> | c |
+---+       +---+       +---+
                        ↑
                        |
                        重新引入
```

### 实现原理

其原理是，当 nodejs `import` 或者 `require` 任何 `资源` 的时候， 
他调用一系列的生命周期函数，在这些生命周期函数中，我们可以获取到每一个资源，
当我们要干预加载的时候，我们可以通过重写这些生命周期函数来实现。
比如：


```typescript
const Module = require('node:module')

const _oldResolveFilename = Module._resolveFilename

Module._resolveFilename = function (request: string, parent: NodeModule) {
  const filename = _oldResolveFilename(request, parent)
  // do something
  return filename
}
```

在这个过程中 tsxx 通过重写 `Module._load` 来记录每个资源的上下级关系，当资源发生变化时，
我们将其未使用的下级资源全部卸载，然后重新加载该文件，这里所谓的未使用是指该下级资源没有被其他资源引用。
这意味着，在上述用例中，如果我们修改的是 b 文件，b 和 c 都会被卸载，然后重新加载 b 文件。

```
+---+       +---+       +---+
| a | ----> | b | ----> | c |
+---+       +---+       +---+
            ↑           ↑
            |           |
            +-----------+
            |
            卸载

+---+       +---+
| a | ----> | b |
+---+       +---+
            ↑
            |
            +
            |
            重新引入 b，此时 b 依赖的下级资源也会被重新引入
```

在 `tsxx` 代码中使用了大量的 cjs (commonjs) 模块，因为我们需要在脚本加载前做一些 hack 操作，而 esm 是异步的，无法在加载前做 hack 操作。

### 结语

[tsxx](https://github.com/sia-fl/tsxx) 不算是让你去熟悉 module 资源加载生命周期的好项目，在这里介绍 [module-alias](https://github.com/ilearnio/module-alias) 这个项目。
