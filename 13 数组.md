# 数组

对应英文原页：https://solidity-by-example.org/array

数组可以具有编译期固定大小，也可以是动态大小。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Array {
    // 初始化数组的几种方式
    uint256[] public arr;
    uint256[] public arr2 = [1, 2, 3];
    // 固定大小的数组，所有元素都初始化为 0
    uint256[10] public myFixedSizeArr;

    function get(uint256 i) public view returns (uint256) {
        return arr[i];
    }

    // Solidity 可以返回整个数组。
    // 但对于长度可能无限增长的数组，应避免使用此函数。
    function getArr() public view returns (uint256[] memory) {
        return arr;
    }

    function push(uint256 i) public {
        // 向数组追加元素
        // 这会使数组长度增加 1。
        arr.push(i);
    }

    function pop() public {
        // 删除数组的最后一个元素
        // 这会使数组长度减少 1
        arr.pop();
    }

    function getLength() public view returns (uint256) {
        return arr.length;
    }

    function remove(uint256 index) public {
        // delete 不会改变数组长度。
        // 它会将该索引处的值重置为默认值，
        // 本例中为 0
        delete arr[index];
    }

    function examples() external pure {
        // 在 memory 中创建数组，只能创建固定大小的数组
        uint256[] memory a = new uint256[](5);

        // 在 memory 中创建嵌套数组
        // b = [[1, 2, 3], [4, 5, 6]]
        uint256[][] memory b = new uint256[][](2);
        for (uint256 i = 0; i < b.length; i++) {
            b[i] = new uint256[](3);
        }
        b[0][0] = 1;
        b[0][1] = 2;
        b[0][2] = 3;
        b[1][0] = 4;
        b[1][1] = 5;
        b[1][2] = 6;
    }
}
```

### 删除数组元素的示例

通过将元素从右向左移动来删除数组元素

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract ArrayRemoveByShifting {
    // [1, 2, 3] -- remove(1) --> [1, 3, 3] --> [1, 3]
    // [1, 2, 3, 4, 5, 6] -- remove(2) --> [1, 2, 4, 5, 6, 6] --> [1, 2, 4, 5, 6]
    // [1, 2, 3, 4, 5, 6] -- remove(0) --> [2, 3, 4, 5, 6, 6] --> [2, 3, 4, 5, 6]
    // [1] -- remove(0) --> [1] --> []

    uint256[] public arr;

    function remove(uint256 _index) public {
        require(_index < arr.length, "index out of bounds");

        for (uint256 i = _index; i < arr.length - 1; i++) {
            arr[i] = arr[i + 1];
        }
        arr.pop();
    }

    function test() external {
        arr = [1, 2, 3, 4, 5];
        remove(2);
        // [1, 2, 4, 5]
        assert(arr[0] == 1);
        assert(arr[1] == 2);
        assert(arr[2] == 4);
        assert(arr[3] == 5);
        assert(arr.length == 4);

        arr = [1];
        remove(0);
        // []
        assert(arr.length == 0);
    }
}
```

通过将最后一个元素复制到待删除位置来删除数组元素

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract ArrayReplaceFromEnd {
    uint256[] public arr;

    // 删除元素会在数组中留下空隙。
    // 保持数组紧凑的一个技巧是
    // 把最后一个元素移到待删除的位置。
    function remove(uint256 index) public {
        // 把最后一个元素移到待删除的位置
        arr[index] = arr[arr.length - 1];
        // 删除最后一个元素
        arr.pop();
    }

    function test() public {
        arr = [1, 2, 3, 4];

        remove(1);
        // [1, 4, 3]
        assert(arr.length == 3);
        assert(arr[0] == 1);
        assert(arr[1] == 4);
        assert(arr[2] == 3);

        remove(2);
        // [1, 4]
        assert(arr.length == 2);
        assert(arr[0] == 1);
        assert(arr[1] == 4);
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
