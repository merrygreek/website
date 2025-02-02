---
title: receive函数
last_update:

    date: 1/29/2023

---

`receive` 函数是 Solidity 中的一种特殊函数，它主要被用来接收 Ether 转账。还有一个 `fallback` 函数也可以用来接收 Ether 转账，下一节我们会介绍。

## receive 函数定义语法

`receive` 函数的定义是固定的，其可见性必（*visibility*）须为 `external`，状态可变性（*state mutability*）必须为 `payable`。同时要注意 `receive` 函数不需要 `function` 前缀

```solidity

receive() external payable {
    // 函数体
}

```

## 合约没有定义 receive 和 fallback 函数时，不能对其转账

如果一个合约既没有定义 `receive` 函数，也没有定义 `fallback` 函数，那么我们不能对它发起转账。这种情况下所有试图转账的操作都会被 revert 。如下面的示例所示：


:::tip `Callee` 没有定义 `receive` 和 `fallback` 函数，三种对其转账的方法都失败

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity ^0.8.17;

// Callee 没有定义 receive 和 fallback 函数
contract Callee {}

contract Caller {
    address payable callee;

    // 注意： 记得在部署的时候给 Caller 合约转账一些 Wei，比如 100
    constructor() payable{
        callee = payable(address(new Callee()));
    }

    // 失败
    function tryTransfer() external {
        callee.transfer(1);
    }

    // 失败
    function trySend() external {
        bool success = callee.send(1);
        require(success, "Failed to send Ether");
    }

    // 失败
    function tryCall() external {
        (bool success, bytes memory data) = callee.call{value: 1}("");
        require(success, "Failed to send Ether");
    }
}

```

:::

要注意我们上面提到的转账指的是单纯的转账（*msg.data为空*）。单纯的转账有三种方法：

- `send(amount)` （gas 固定为 2300，错误时 revert)
- `transfer(amount)` （gas 固定为 2300, 返回布尔值） 
- `call{value: amount}("")`（gas 可以随意设定，返回布尔值）

这三种方法都是发送 amount 数量的 Wei 到目标账户。

我们提到如果合约既没有定义 `receive` 函数，也没有定义 `fallback` 函数，那么对其转账会失败。但是如果是进行普通函数调用（*msg.data不为空*），那么还是可以成功的。例如你可以用 call 函数进行普通函数调用：

```solidity

// 调用 foo() 函数 (msg.data 不为空)
call( abi.encodeWithSignature("foo()") );

// 调用 foo() 函数，并转账 1 Wei (msg.data 不为空)
call{value: 1}( abi.encodeWithSignature("foo()") );

```

注意第二种函数调用中，还同时给目标合约转了 1 Wei， 这也是允许的，因为这是一个函数调用，而不是单纯的转账。

## 注意 Gas 不足的问题

定义 `receive` 函数的时候要注意 Gas 不足的问题。前面我们有提到, `send` 和 `transfer` 的 Gas 是固定为 2300 的。所以剩余的 Gas 往往不够进行复杂的操作。如果函数体要执行较复杂的操作，那么可能会抛出 `Out of Gas` 异常。例如，进行下面的这些操作所消耗的 Gas 会超过 2300：

- 修改状态变量
- 创建合约
- 调用其他相对复杂的函数
- 发送 Ether 到其他账户

例如下面的 `receive` 函数永远会被 revert， 因为所使用的 Gas 已经超过 2300。

:::tip `receive` 函数消耗过多 Gas

```solidity

// 用send,transfer函数转账到该合约都会被 revert
// 原因是消耗的 Gas 超过了 2300
contract Example {
    uint a;
    receive() external payable {
        a += 1;
    }
}


```

:::