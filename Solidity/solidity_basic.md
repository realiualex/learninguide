# Solidity 基础 

## 初步认识

在线学习IDE: https://remix.ethereum.org/

solidity 里所有的 function都要在 contract下，一个contract就对应以太坊上一个合约地址。一般来说，一个dApp可能会有多个合约。每次合约部署，合约地址都会更新。对于 dApp的开发来说，一个常见的做法是使用代理或入口合约，它的地址是永远不变的，这个代理合约有一个链接到其他实现合约的指针，当需要升级某个功能的时候，开发者可以在新的地址合约，然后更新这个指针。（在代理合约里，需要定一个专门的函数，来更新这个地址，这个地址一般是一个状态变量。实现合约可以更新这个代理合约里的状态变量，由此2个合约就能实现）

### hello world 示例

一个示例

```solidity
pragma solidity >=0.8.2 <0.9.0;

contract HelloWorld {
    string myName;
    function setName(string memory name) public{
         myName = name;
    }
    function getName() public view returns(string memory) {
        return myName;
    }

    function ss() public pure returns(uint104) {
        return 1+2;
    }
}
```

从上面示例里，我们能看到，一个合约里定义 function的时候，我们需要指明该function的可见度，比如我用的是 public，则表示区块链上任何人都能调用该方法。如果是 private，则表示只有合约本身能调用。internal表示只有当前合约以及其派生合约才能调用。external表示只有外部合约才能调用该方法。

**思考**: public 的方法，允许其他合约调用，可以作为代理合约，也可以修改 storage 类型的数据，类似区块链预言机Oracle，就可以通过这种方法，将现实世界的数据，写入到区块链。同时我们也发现，view 的数据是不会消耗gas fee，但是如果是 update的操作，则消耗 gas fee。

数据位置data location: 有三种，storage, memory和 call data。storage就是存在区块链上的数据，写入和更新需要 gas fee。memory使在内存里，只有合约执行的时候才有，执行完就释放，一般用于函数参数，局部变量或函数执行期间创建的数组等。calldata 用于另外一个合约传过来的参数，calldata是只读的。

扩展：派生合约指的是从父合约继承来的新合约。示例：B就是A的派生合约

```solidity
contract A {
// ...
}
contract B is A {
// ...
}
```



同时function还有一些修饰符，比如 view,pure或payable，view表示只读当前合约里的变量，而不修改变量值，pure表示不读链上的信息，pure 函数里可以放一些逻辑计算。因此都不消耗gas fee。

**思考**: 由于pure类型的函数不消耗gas，这里会有一个安全风险，比如有人部署了一个pure function，里面是很复杂的计算，那么只有部署的时候消耗gas，之后执行的时候不消耗（链下调用执行不消耗，如果是另外一个合约调用这个pure 函数，那么另外一个合约还是要付gas fee的），不停的执行这个合约，会对以太坊产生什么影响吗？

其实此时对以太坊整个网络是没任何影响的，因为 pure 函数不会跟以太坊区块链上的数据交互，所以这类攻击只会影响特定的以太坊节点，节点维护者可以选择对这类攻击做一些处理，但该攻击不会对链上其他节点产生影响。



## 合约之间的调用



合约之间的调用需要调用方支付 gas fee，哪怕是A合约调用B合约的 pure function，也要A合约支付（其实是A合约的外部账号EOA Account支付）

**思考**：A合约调用B合约，A合约在链上，EOA account在本地，怎么付费给B合约？

**答**：合约并不会自发调用，一切合约间的调用，均是EOA Account外部发起的。EOA Account是唯一拥有私钥的实体，可以签名广播交易。链上的合约，只是规定了特定的逻辑的代码，必须通过外部账户的互动才能触发这个逻辑



### 合约调用示例

比如我们有一个这样的合约，此时我们想通过另外一个合约，改变当前合约里的myData，假设当前合约文件名为 helloworld.sol，此时需要先将这个合约部署，然后得到合约的地址

```solidity
// SPDX-License-Identifier: MIT

pragma solidity >=0.8.2 <0.9.0;

contract HelloWorld {
    struct Data {
        uint number;
        string name;
    }
    Data public myData;

    function getData() public view returns(Data memory) {
        return myData;
    }
    function setData(uint _number, string memory _name) public {
        myData.number = _number;
        myData.name = _name;
    }
}
```

在另外一个合约里，如果我们有当前合约的代码，我们需要导入，然后定义一个helloWorldAddress合约地址，在部署的时候，需要将这个合约地址传进去

```solidity
// SPDX-License-Identifier: MIT

pragma solidity >=0.8.2 <0.9.0;
import "./helloworld.sol"; // 引入 HelloWorld.sol 文件

contract Updater {
    
    address public helloWorldAddress;

    // 构造函数，设置 HelloWorld 合约的地址
    constructor(address _helloWorldAddress) {
        helloWorldAddress = _helloWorldAddress;
    }

    // 更新 HelloWorld 合约中的 myData
    function updateData(uint _number, string memory _name) public {
        // 调用 HelloWorld 合约的 setData 方法
        HelloWorld(helloWorldAddress).setData(_number, _name); // 直接调用 HelloWorld 合约的函数
    }
}
```

但有时候，我们调用的合约是别人写的，并不是我们自己写的，此时我们可以通过区块链浏览器，获得这个合约的ABI (application binary interface，类似API)，通过 ABI 就能知道要调用的方法里的函数名，以及数据类型

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.2 <0.9.0;

// 定义与 HelloWorld 合约交互的接口
interface IHelloWorld {
    function setData(uint _number, string memory _name) external;
    function myData() external view returns (uint, string memory);
}

contract Updater {
    address public helloWorldAddress;

    // 构造函数，设置 HelloWorld 合约的地址
    constructor(address _helloWorldAddress) {
        helloWorldAddress = _helloWorldAddress;
    }

    // 更新 HelloWorld 合约中的 myData
    function updateData(uint _number, string memory _name) public {
        // 创建接口实例
        IHelloWorld helloWorld = IHelloWorld(helloWorldAddress);
        // 调用 HelloWorld 合约的 setData 方法
        helloWorld.setData(_number, _name);
    }

    // 获取 HelloWorld 合约中的 myData
    function getData() public view returns (uint, string memory) {
        // 创建接口实例
        IHelloWorld helloWorld = IHelloWorld(helloWorldAddress);
        // 调用 HelloWorld 合约的 myData 方法
        return helloWorld.myData();
    }
}

```



### 发币
用别人写好的库，很容易发一个币(无需实现 transfer等方法，因为继承了openzeppelin 这个库，别人都已经写好了)，在 constructor里定义了初始供应量，这个初始供应量的单位是10^-18 次方，这个要注意

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract AlexLiuCoin is ERC20, Ownable {
    constructor(uint256 initialSupply) ERC20("AlexToken", "ALEX") Ownable(msg.sender) {
        _mint(msg.sender, initialSupply);
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function burn(uint256 amount) public {
        _burn(msg.sender, amount);
    }
}

```
之后在 MetaMask里，可以添加这个币的合约地址，将其加进去，就能看到token了



### 获得合约data
我们可以获得合约data，然后对这个data 签名，之后将签名广播出去，就完成了合约部署。
```
brew install truffle
mkdir solidity-code && cd solidity-code
truffle init

```
然后把自己的代码放在solidity-code/contracts 文件夹下，在solidity-code/migrations 路径创建一个 deployment.js，内容(contracts路径下文件名为AlexCoin.sol)
```js
// migrations/2_deploy_contracts.js

const AlexCoin = artifacts.require("AlexCoin");

module.exports = async function (deployer) {
    // 部署合约
    await deployer.deploy(AlexLiuCoin, 1000000); // 传入初始供应量

    // 获取合约实例
    const instance = await AlexLiuCoin.deployed();

    // 获取合约的 ABI 和字节码
    const bytecode = instance.constructor.bytecode;
    const abi = instance.constructor.abi;

    // 获取数据字段
    const data = instance.constructor.contract._jsonInterface[0].bytecode; // 获取部署数据
    console.log("Contract deployment data:", data);

    // 如果需要对数据进行签名或其他操作，可以在这里继续处理
};

```

项目跟路径创建一个 getData.js，内容如下
```js
// getData.js

const path = require('path');
const fs = require('fs');

// 获取合约的 ABI 和字节码
const contractPath = path.join(__dirname, 'build', 'contracts', 'AlexLiuCoin.json');
const contractJson = JSON.parse(fs.readFileSync(contractPath, 'utf8'));

// 获取合约的字节码
const contractData = contractJson.bytecode;

console.log("Contract deployment data:", contractData);

```

在 truffle-config.js里，指定solidity版本
```js
module.exports = {
    compilers: {
        solc: {
            version: "0.8.20", // 使用的 Solidity 版本
        }
    }
};

```

之后执行 truffle compile 进行编译，编译之后，能看到 build/contracts下有很多ABI json文件，之后 node getData.js 就可以得到data了


## 合约调用

可以使用 cast 命令来调用合约。 安装cast命令前，需要安装 foundry toolchain，然后执行 foundryup 命令就安装好了
```
https://getfoundry.sh/introduction/installation/
```

下面的命令，是执行 0xc9934593a3e435bE4d4eD43D124fe84fAEd76992 合约里的 execute 方法，execute 支持3个参数，分别是 address, unit256, bytes，第一个address的值是 [0x08210F9170F89Ab7658F0B5E3fF39b0E03C594D4](https://sepolia.etherscan.io/address/0x08210f9170f89ab7658f0b5e3ff39b0e03c594d4#writeProxyContract)，这是一个ERC20代币合约地址，第二个 uint256为0，第三个 bytes的值是 cast calldata 的执行结果。cast calldata 后面的 transfer(address, uint256)，是0x08210F9170F89Ab7658F0B5E3fF39b0E03C594D4 这个合约里提供的方法
```shell
cast send --gas-limit 1000000 --private-key xxx --rpc-url https://1rpc.io/sepolia 0xc9934593a3e435bE4d4eD43D124fe84fAEd76992 "execute (address,uint256,bytes)" 0x08210F9170F89Ab7658F0B5E3fF39b0E03C594D4 0 $(cast calldata "transfer(address,uint256)" 0xF79ef8dcebAd5cbA8E8Af563D46590498C50d7Bf 1)
```

在上述ERC20代币的 implement contract里，可以看到 transfer 这个function 的input，我们在调用的时候，用的是 type，而并非name，而且严格按照顺序，因此是 transfer(address, uint256)，而并非 transfer(to, value)。这个要注意.

```json
[
    {
        "inputs":
        [
            {
                "internalType": "address",
                "name": "to",
                "type": "address"
            },
            {
                "internalType": "uint256",
                "name": "value",
                "type": "uint256"
            }
        ],
        "name": "transfer",
        "outputs":
        [
            {
                "internalType": "bool",
                "name": "",
                "type": "bool"
            }
        ],
        "stateMutability": "nonpayable",
        "type": "function"
    }
]
```

同样我们也能看到，这个合约支持 executeBatch 的功能，从executeBatch 的ABI，我们能看出支持 的参数是 executeBatch((address,uint256,bytes)[])，那么我们的交易可以这样写
```shell
cast send --gas-limit 1000000 --private-key xxxx --rpc-url https://1rpc.io/sepolia 0xc9934593a3e435bE4d4eD43D124fe84fAEd76992 "executeBatch((address,uint256,bytes)[])" '[(0x08210F9170F89Ab7658F0B5E3fF39b0E03C594D4,0,0xa9059cbb000000000000000000000000f79ef8dcebAd5cbA8E8Af563D46590498C50d7Bf0000000000000000000000000000000000000000000000000000000000000001),(0x08210F9170F89Ab7658F0B5E3fF39b0E03C594D4,0,0xa9059cbb000000000000000000000000f79ef8dcebAd5cbA8E8Af563D46590498C50d7Bf0000000000000000000000000000000000000000000000000000000000000002)]'
```

我们也可以通过 abi-encode 命令，提前把abi hash结果拿到手，然后cast的时候直接把这个结果发出去
```shell
ABI_BUILD_RESULT=`cast abi-encode "executeBatch((address,uint256,bytes)[])" \
'[(0x08210F9170F89Ab7658F0B5E3fF39b0E03C594D4,0,0xa9059cbb000000000000000000000000f79ef8dcebAd5cbA8E8Af563D46590498C50d7Bf0000000000000000000000000000000000000000000000000000000000000001)]'`

cast send --gas-limit 1000000 --private-key xxxx --rpc-url https://1rpc.io/sepolia 0xc9934593a3e435bE4d4eD43D124fe84fAEd76992 $ABI_BUILD_RESULT
```


executeBatch 的 ABI 结构
```json
 {
        "inputs":
        [
            {
                "components":
                [
                    {
                        "internalType": "address",
                        "name": "target",
                        "type": "address"
                    },
                    {
                        "internalType": "uint256",
                        "name": "value",
                        "type": "uint256"
                    },
                    {
                        "internalType": "bytes",
                        "name": "data",
                        "type": "bytes"
                    }
                ],
                "internalType": "struct Call[]",
                "name": "calls",
                "type": "tuple[]"
            }
        ],
        "name": "executeBatch",
        "outputs":
        [
            {
                "internalType": "bytes[]",
                "name": "returnData",
                "type": "bytes[]"
            }
        ],
        "stateMutability": "payable",
        "type": "function"
    }
```

## 故障排查

如果合约在执行的时候，有这样的报错 "0x0 Transaction mined but execution failed"，表示交易已经被矿工打包，并且在区块链上被确认，但是在执行过程中失败了，这时候通常会发生 revert，也就是合约执行出错导致回滚。举个例子，比如合约B调用合约A，但是输入的参数不对，就会出现这个情况



## 工具
区块链交易debug工具
https://dashboard.tenderly.co/

当你有一笔交易，或者执行某一个合约报错的时候，区块链浏览器一般显示的信息很有限，比如下面测试的这个，只能看出合约执行失败，但为什么失败，没有太多线索
https://sepolia.etherscan.io/tx/0x3fa7fcbd2e24b32bf952792a2e082f25cf66024f39c4fd76e629a3afc5d0a09e

但在上面的tenderly.co工具里，可以输入tx id进行查询，就能很清楚的看到执行哪个合约文件的哪个方法报错了，在这里还可以直接 看合约内容，并提供了搜索功能。能很方便debug.



## 参考资料 

https://docs.alchemy.com/docs/when-to-use-storage-vs-memory-vs-calldata-in-solidity

