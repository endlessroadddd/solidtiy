
# Gas 优化

Gas 优化是开发以太坊智能合约中非常具有挑战性的任务。以太坊上的计算资源是有限的，每个区块可用的 Gas 有上限（本书编写时约为 1250 万 Gas）。随着去中心化金融应用的兴起，以太坊的利用率逐渐增长，导致 Gas 价格上升，交易手续费居高不下。因此，优化合约交易的 Gas 消耗显得尤为重要。

## 2.5.1 变量打包：像拼积木一样节省空间

想象你有一些积木块，每块都有固定的大小（32 字节）。如果能将多个小积木块拼在一起，就能节省空间和资源。**变量打包**就是这样一个概念：

- **连续存储**：在智能合约中，变量按照固定大小（32 字节）存储。如果能将多个小变量（如 `uint128`）放在同一个存储槽中，可以减少槽的使用数量，节省 Gas。

### 举例说明

```solidity
// 不优化的变量顺序
uint128 a;
uint256 b;
uint128 c;

// 优化后的变量顺序
uint128 a;
uint128 c;
uint256 b;
```

优化后的顺序可以让 `a` 和 `c` 共用一个存储槽，减少 Gas 消耗。

**选择数据类型时的建议**：
- 如果变量可以与其他变量打包，使用较小的数据类型（如 `uint128`）。
- 如果变量无法打包，尽量使用 256 位的数据类型（如 `uint256` 和 `bytes32`），因为 EVM 一次处理 32 字节更高效。

## 2.5.2 选择适合的数据类型：量体裁衣

选择合适的数据类型就像为不同物品选择合适的盒子：

### 1. 使用常量或不可变量

如果某个值在合约运行中不会改变，使用 `constant` 或 `immutable` 可以减少 Gas，因为这些值会被直接包含在合约代码中，无需额外存储。

```solidity
contract C { 
    uint constant X = 32**22 + 8; // 定义常量 
    string constant TEXT = "abc"; 
    uint immutable decimals; // 定义不可变量 

    constructor(uint _d) { 
        decimals = _d; 
    } 
}
```

**区别**：
- 常量 (`constant`) 在编译期确定值。
- 不可变量 (`immutable`) 在部署时确定值。

### 2. 固定长度优于变长

如果能确定数组的大小，优先使用固定大小的数组。

```solidity
uint256[12] monthlys; // 固定大小数组
```

对于字符串或字节，如果长度固定，使用 `bytes1` 至 `bytes32` 类型比 `string` 或 `bytes` 更高效。

### 3. 映射 vs 数组

- **映射（Mapping）**：通常比数组更省 Gas，因为存储、读取和删除操作的 Gas 消耗是固定的。
- **数组（Array）**：适用于存储较小的数据类型，并且通过变量打包节省空间。但动态数组的 Gas 消耗会随着长度增长而增加。

**优化建议**：
- 使用映射存储大多数情况下的数据。
- 当数组元素类型较小且可以打包时，数组也是不错的选择。
- 如果必须使用动态数组，尽量保持数组末尾递增，避免数组移位操作。

## 2.5.3 内存和存储：内存中的操作更快捷

在智能合约中，**内存操作比存储操作便宜**。内存类似于工作台，操作更快更便宜；存储类似于仓库，操作更慢更贵。

### 优化技巧

尽量在内存中进行计算，最后再将结果存回存储。例如：

```solidity
uint num = 0; 

function expensiveLoop(uint x) public { 
    for(uint i = 0; i < x; i++) { 
        num += 1; 
    } 
}
```

优化为：

```solidity
uint num = 0; 

function lessExpensiveLoop(uint x) public { 
    uint temp = num; 
    for(uint i = 0; i < x; i++) { 
        temp += 1; 
    } 
    num = temp; 
}
```

通过使用临时变量 `temp`，减少了对全局变量 `num` 的频繁操作，节省 Gas。

## 2.5.4 减少存储：清理和利用事件

存储是最昂贵的操作之一，减少存储使用可以大幅降低 Gas 消耗。

### 1. 清理存储

删除不需要的数据可以返还一部分 Gas，特别是在清理大数据变量时效果显著。

```solidity
contract DelC { 
    uint[] bigArr; 

    function doSome() public { 
        // 执行一些操作 
        delete bigArr; 
    } 
}
```

同样，当合约不再需要时，可以销毁合约返还 Gas。

### 2. 使用事件储存数据

如果某些数据不需要在链上被频繁访问，可以通过事件（Events）记录，节省 Gas。

**示例**：

不优化的合约：

```solidity
contract Registry { 
    mapping (uint256 => address) public documents; 

    function register(uint256 hash) public { 
        documents[hash] = msg.sender; 
    } 
}
```

优化后的合约：

```solidity
contract DocumentRegistry { 
    event Registered(uint256 hash, address sender); 

    function register(uint256 hash) public { 
        emit Registered(hash, msg.sender); 
    } 
}
```

通过事件记录数据，不仅节省了存储 Gas，还能在需要时通过订阅事件查询数据。

## 2.5.5 其他优化建议：细节决定成败

### 1. 初始化变量

在 Solidity 中，每个变量的赋值都要消耗 Gas。初始化变量时，避免使用默认值。

```solidity
uint256 value; // 更省 Gas
uint256 value = 0; // 更耗 Gas
```

### 2. 优化 `require` 语句

限制 `require` 中错误信息的长度（如不超过 32 字节），可以减少 Gas 消耗。

### 3. 明确函数可见性

显式声明函数的可见性（如 `external` 而不是 `public`）不仅提高安全性，还能优化 Gas 使用。

### 4. 链下计算

尽量将复杂计算移到链下完成，只在链上进行必要的验证，减少 Gas 消耗。例如，在排序列表中，链下计算插入位置，只在链上验证结果。

### 5. 警惕循环

循环会随着数据量的增加而增加 Gas 消耗，甚至可能导致交易失败。优化方法包括：

- 避免使用依赖数据大小的循环。
- 将线性增长的计算量转化为固定大小。
- 限制循环次数，将大数据分拆为小数据段处理。

**示例**：

在质押合约中，设置质押有效期，限制单次计算量大小。

## 总结

Gas 优化就像在有限资源下尽可能高效地完成任务。通过合理安排变量存储、选择合适的数据类型、优化内存和存储操作，以及遵循其他优化建议，可以显著降低智能合约的 Gas 消耗，提高其运行效率和经济性。

希望这个总结能帮助你更好地理解 Gas 优化的核心概念！

---

# Gas Optimization

Gas optimization is a highly challenging task in Ethereum smart contract development. Ethereum has limited computational resources, and each block has a Gas limit (about 12.5 million Gas at the time of writing). With the rise of decentralized finance (DeFi) applications post-2020, Ethereum's utilization has increased, leading to higher Gas prices and transaction fees. Therefore, optimizing Gas consumption in contract transactions has become crucial.

## 2.5.1 Variable Packing: Save Space Like Building Blocks

Imagine you have building blocks, each with a fixed size (32 bytes). If you can stack smaller blocks together, you save space and resources. **Variable packing** works similarly:

- **Sequential Storage**: In smart contracts, variables are stored in fixed-size (32-byte) slots. If you can place multiple smaller variables (like `uint128`) in the same slot, you reduce the number of slots used, saving Gas.

### Example

```solidity
// Unoptimized variable order
uint128 a;
uint256 b;
uint128 c;

// Optimized variable order
uint128 a;
uint128 c;
uint256 b;
```

In the optimized order, `a` and `c` share a storage slot, reducing Gas consumption.

**Data Type Selection Tips**:
- Use smaller data types (`uint128`) when variables can be packed together.
- Use 256-bit data types (`uint256`, `bytes32`) when variables cannot be packed, as the EVM handles 32 bytes more efficiently.

## 2.5.2 Choosing the Right Data Types: Tailoring to Needs

Selecting the appropriate data type is like choosing the right box for different items:

### 1. Use Constants or Immutables

If a value does not change during contract execution, use `constant` or `immutable` to reduce Gas, as these values are embedded directly in the contract code.

```solidity
contract C { 
    uint constant X = 32**22 + 8; // Define a constant
    string constant TEXT = "abc"; 
    uint immutable decimals; // Define an immutable

    constructor(uint _d) { 
        decimals = _d; 
    } 
}
```

**Difference**:
- **Constant**: Value is determined at compile time.
- **Immutable**: Value is set during deployment.

### 2. Fixed-Length Over Variable-Length

If the size of an array is known, prefer fixed-size arrays over dynamic ones.

```solidity
uint256[12] monthlys; // Fixed-size array
```

For strings or bytes, use fixed-length types (`bytes1` to `bytes32`) if the length is known, instead of `string` or `bytes`.

### 3. Mappings vs Arrays

- **Mappings**: Generally more Gas-efficient for storage, reading, and deletion, as their Gas consumption is fixed.
- **Arrays**: Suitable for storing smaller data types and can save space through packing, but dynamic arrays have Gas costs that increase with length.

**Optimization Tips**:
- Use mappings for most data storage needs.
- Use arrays when elements are small and can be packed.
- If using dynamic arrays, keep them append-only to avoid costly shifts.

## 2.5.3 Memory vs Storage: Memory Operations Are Cheaper

In smart contracts, **memory operations are cheaper than storage operations**. Think of memory as your workspace—fast and cheap—while storage is like a warehouse—slow and expensive.

### Optimization Technique

Perform calculations in memory and only write back to storage at the end.

```solidity
uint num = 0; 

function expensiveLoop(uint x) public { 
    for(uint i = 0; i < x; i++) { 
        num += 1; 
    } 
}
```

Optimize to:

```solidity
uint num = 0; 

function lessExpensiveLoop(uint x) public { 
    uint temp = num; 
    for(uint i = 0; i < x; i++) { 
        temp += 1; 
    } 
    num = temp; 
}
```

Using a temporary variable `temp` reduces frequent access to the global `num`, saving Gas.

## 2.5.4 Reducing Storage: Clean Up and Use Events

Storage operations are among the most expensive. Reducing storage usage can significantly lower Gas consumption.

### 1. Clean Up Storage

Deleting unnecessary data can refund some Gas, especially when clearing large data structures.

```solidity
contract DelC { 
    uint[] bigArr; 

    function doSome() public { 
        // Perform some operations 
        delete bigArr; 
    } 
}
```

Similarly, destroying a contract when it's no longer needed can refund Gas.

### 2. Use Events to Store Data

For data that doesn't need to be frequently accessed on-chain, use events instead of storage variables to save Gas.

**Example**:

Unoptimized contract:

```solidity
contract Registry { 
    mapping (uint256 => address) public documents; 

    function register(uint256 hash) public { 
        documents[hash] = msg.sender; 
    } 
}
```

Optimized contract:

```solidity
contract DocumentRegistry { 
    event Registered(uint256 hash, address sender); 

    function register(uint256 hash) public { 
        emit Registered(hash, msg.sender); 
    } 
}
```

By using events, you avoid storing data on-chain, significantly saving Gas while still maintaining a permanent record that can be queried off-chain.

## 2.5.5 Other Optimization Tips: Details Matter

### 1. Initialize Variables

Avoid explicitly initializing variables to their default values to save Gas.

```solidity
uint256 value; // More Gas-efficient
uint256 value = 0; // Less Gas-efficient
```

### 2. Optimize `require` Statements

Limit the length of error messages in `require` statements (e.g., keep them under 32 bytes) to reduce Gas costs.

### 3. Explicit Function Visibility

Clearly declare function visibility (e.g., `external` instead of `public`) to enhance security and optimize Gas usage.

### 4. Off-Chain Computation

Move complex computations off-chain and perform only necessary validations on-chain to reduce Gas consumption. For example, calculate insertion positions off-chain and only verify them on-chain.

### 5. Beware of Loops

Loops that depend on data size or time can lead to increasing Gas consumption and potential transaction failures. Optimize by:

- Avoiding loops that scale with data size.
- Converting linear computations to fixed-size operations.
- Limiting the number of iterations by breaking data into smaller segments.

**Example**:

In staking contracts, set a maximum staking period to limit computation per transaction.

## 总结

Gas 优化就像在有限的资源下尽可能高效地完成任务。通过合理安排变量存储、选择合适的数据类型、优化内存和存储操作，以及遵循其他优化建议，可以显著降低智能合约的 Gas 消耗，提高其运行效率和经济性。

希望这个总结能帮助你更好地理解 Gas 优化的核心概念！
