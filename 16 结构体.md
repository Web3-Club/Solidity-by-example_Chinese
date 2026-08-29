# 结构体

对应英文原页：https://solidity-by-example.org/structs

你可以通过创建 `struct` 来定义自己的类型。

结构体适合将相关数据组合在一起。

结构体可以在合约外部声明，并在另一个合约中导入。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Todos {
    struct Todo {
        string text;
        bool completed;
    }

    // 'Todo' 结构体数组
    Todo[] public todos;

    function create(string calldata _text) public {
        // 初始化结构体的 3 种方式
        // - 像函数一样调用它
        todos.push(Todo(_text, false));

        // 键值映射
        todos.push(Todo({text: _text, completed: false}));

        // 先初始化一个空结构体，再更新它
        Todo memory todo;
        todo.text = _text;
        // todo.completed 初始化为 false

        todos.push(todo);
    }

    // Solidity 会自动为 'todos' 创建 getter，
    // 因此实际上并不需要这个函数。
    function get(uint256 _index)
        public
        view
        returns (string memory text, bool completed)
    {
        Todo storage todo = todos[_index];
        return (todo.text, todo.completed);
    }

    // 更新 text
    function updateText(uint256 _index, string calldata _text) public {
        Todo storage todo = todos[_index];
        todo.text = _text;
    }

    // 更新 completed
    function toggleCompleted(uint256 _index) public {
        Todo storage todo = todos[_index];
        todo.completed = !todo.completed;
    }
}
```

### 声明并导入结构体

声明结构体的文件

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;
// 此文件保存为 'StructDeclaration.sol'

struct Todo {
    string text;
    bool completed;
}
```

导入上述结构体的文件

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "./StructDeclaration.sol";

contract Todos {
    // 'Todo' 结构体数组
    Todo[] public todos;
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
