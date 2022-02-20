# async/await

在 `fn`、`closure`、`block`前使用 `async` 关键字，会将标记的代码转化为一个 `Future`。因此，`async` 标记的代码不会立即运行，只有在 `Future` 上调用 `.await` 时才会计算运行 `Future`。而在 `await` 一个 `Future` 时，会暂停当前函数的执行，直到 `executor` 完成对该 `Future` 的计算。

以上是对 `async/await` 语义的基本介绍。在本章中，我们将会更加深入地介绍 `async/await` 的使用和它们的底层原理。
