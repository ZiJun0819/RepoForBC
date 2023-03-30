# Solidity Study

## Day1

```javascript
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.7;

contract SimpleStorage{
    //默认internal,uint和int都是默认为uint256和int256
    uint256 internal favoriteNumber = 5;
    string favoriteString = "cat";
    int256 favoriteInt = 6;
    address favoriteAddress = 0x9E8704942C661e3c64bb13B04Fbb278eA0dc8118;
    bytes32 favoriteBytes = "Moth";
    bool favoriteBoolean = true;

    // People public person = People({favoriteNumber: 8, name: "Alice"});
    //默认array是dynamic的，即自适应大小，不过也可以直接定义数组大小
    People[] public persons;

    struct People{
        uint256 favoriteNumber;
        string name;
    }

    mapping(string => uint256) nameToNumber;

    /*
        EVM访问数据的六个place：
        1. Stack
        2. Memory       memory关键字需要修饰string、array、struct、mapping等类型的组合数据，
        表示零时存储变量，因为solidity知道uint、int、bool等数据在函数中是临时的，但是string等数据需要特定表示
        3. Storage      合同中定义的变量默认为storage，可在函数外存在，而Memory只在function中存在
        4. Calldata     类似于memory的作用，但是如果使用calldata修饰变量，变量便不可修改，而memory可以
        5. Code
        6. Logs
    */
    function insertPeople(string memory _name, uint256 _favoriteNumber) public{
        persons.push(People(_favoriteNumber, _name));
        nameToNumber[_name] = _favoriteNumber;
    }
    /*
        函数修饰词：
        1. public       visible externally and internally (creates a getter function for storage/state variables)
        隐私程度最低，公共开放，用于设置值
        2. private      only visible in the current contract，私有
        3. external     only visible externally (only for functions) - i.e. can only be message-called (via this.func)
        只有外部合同可以调用
        4. internal     only visible internally，自己和自己的子类可以使用
    */
    function storeValue(uint256 _favoriteNumber) public{
        favoriteNumber = add(25,6); //这个会消耗gas，即使add函数本身不消耗gas
        favoriteNumber = _favoriteNumber;
        favoriteNumber++;
    }

    /*
        view和pure都是只能读但是不能修改区块链状态
        view:可访问接触合同中已定义的变量，即可以访问存储但是不能进行计算
        pure:可返回函数中定义变量，并且可以对函数传递变量进行操作,但是不能使用或访问存储，只能进行计算
        可以在pure中设计一个算法
    */
    function retriveValue() public view returns(uint256){
        return favoriteNumber;
    }

    function add(uint256 a, uint256 b) public pure returns(uint256){
        a = a + 5;
        b = b - 3;
        return a*b;
    }

    function searchNumByName(string memory _name) public view returns(uint256){
        return nameToNumber[_name];
    }

}
```

> 合同交互，调用其它合同函数

```javascript
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.7;


import "./SimpleStorage.sol";
// 不同合同之间交互以及调用，合同调用过程仅涉及函数调用，不涉及variable
contract StorageFactory {
    SimpleStorage[] public simpleStorageArray;

    // SimpleStorage simpleStorage = new SimpleStorage();
    function createSimpleStorageFun() public{
        SimpleStorage simpleStorage = new SimpleStorage();
        simpleStorageArray.push(simpleStorage);
    }
    /*
    	address
    	ABI: Contract Application Binary Interface
    */
    function sfStore(uint256 _simpleStorageIndex, uint256 _simpleStorageNumber) public{
        simpleStorageArray[_simpleStorageIndex].storeValue(_simpleStorageNumber);
    }
    
    function sfGet(uint256 _simpleStorageIndex) public view returns(uint256){
        // return SimpleStorage(address(simpleStorageArray[_simpleStorageIndex]))..retriveValue();
        return simpleStorageArray[_simpleStorageIndex].retriveValue();
    }
}
```

> 合同继承 inheritance

``` javascript
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.7;

import "./SimpleStorage.sol";

// 继承合同通过is关键字来实现，并且需要通过import导入合同
contract ExtraStorage is SimpleStorage{

    /*
        继承了SimpleStorage内容中的所有东西，可以通过添加override修饰符进行函数功能重写;
        但是父合同中被重写的函数需要增加修饰符（virtual），以表示函数支持重写
    */
    function storeValue(uint256 _favoriteNumber) public override{
        favoriteNumber = _favoriteNumber+5;
    }
}
```

> Fund，美元换算（chainlink），for-loop，library，special function（receive&fallback），{modifier+(error+revert)/(require)}（特定修饰符）,library使用
>
> <u>***存在问题***</u>：==当多个account向特定合同进行funding时，只有合同的创建者的account才可以withdraw全部钱，相当于其他人的money都可转移到合同创建者的account。==
>
> 疑问？：
>
> 1. ==funders = new address[](0);==，最后的(0)何意？	
> 2. library中的函数必须使用internal进行修饰嘛，不可使用public等？why？

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "./ConvertToUSD.sol";

error NotOwner();

contract FundMe{
    //调用library
    using ConvertToUSD for uint256;
     /*
        1. 依据chainlink将一定数量的ETH转变为USD表示的数字  ETH/USD: 0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e
        为保留小数位，ethAmount需用wei，USD的单位也变相使用wei进行表示
    */
    uint256 public constant MINIMUM_USD = 8.366586*10**18;

    address[] public funders;
    mapping(address => uint256) public addressToAmount;

    address public immutable i_owner;
    
    constructor(){
        i_owner = msg.sender;
    }

    function fund() public payable{
        // require(getConversionRate(msg.value) >= MINIMUM_USD, "You need spend more ETH!");
        require(msg.value.getConversionRate() >= MINIMUM_USD, "You need spend more ETH!");
        addressToAmount[msg.sender] += msg.value;
        funders.push(msg.sender);
    }

    function withdraw() public onlyOwner{
        
        for(uint256 funderIndex=0; funderIndex < funders.length; funderIndex++){
            address funder = funders[funderIndex];
            addressToAmount[funder] = 0;
        }

        funders = new address[](0);
        // // transfer
        // payable(msg.sender).transfer(address(this).balance);
        // // send
        // bool sendSuccess = payable(msg.sender).send(address(this).balance);
        // require(sendSuccess, "Send failed");
        // call
        (bool callSuccess, ) = payable(msg.sender).call{value: address(this).balance}("");
        require(callSuccess, "Call failed");
    }
    modifier onlyOwner{
        // require(msg.sender == i_owner, "You are not the money's owner!");
        if (msg.sender != i_owner) revert NotOwner();
        _;  //表示后面的全部代码
    }
    //意外调用直接重定向至fund；可依据合同地址直接进行fund
    receive() external payable{
        fund();
    }
    //有函数调用即存在msg.data，但是合同中不包含该函数，意外调用直接重定向至fund；
    fallback() external payable{
        fund();
    }
    // Ether is sent to contract
    //      is msg.data empty?
    //          /   \ 
    //         yes  no
    //         /     \
    //    receive()?  fallback() 
    //     /   \ 
    //   yes   no
    //  /        \
    //receive()  fallback()
}
```

> ConvertToUSD.sol 是一个library

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

library ConvertToUSD{
    // 通过合同地址确定real world中挂钩的事务

    function getLatestPrice() internal view returns(uint256){
        AggregatorV3Interface priceFeed = AggregatorV3Interface(0xD4a33860578De61DBAbDc8BFdb98FD742fA7028e);
        // (uint80 roundId, int256 price, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) = priceFeed.latestRoundData();
        (, int256 price, , , ) = priceFeed.latestRoundData();
        return uint256(price*10**10); //保留汇率转换的小数位
    }
    //ethAmout单位位wei，从而可以保留许多小数位的eth表示
    function getConversionRate(uint256 ethAmount) internal view returns(uint256){
        uint256 ethPrice = getLatestPrice();
        uint256 convertedUsdValue = (ethPrice*ethAmount)/ 10**18;
        return convertedUsdValue;
    }
}
```

## Day2

``` javascript
command命令行部署合同运行命令
yarn solcjs --bin --abi --include-path node_modules/ --base-path . -o . SimpleStorage.sol

--bin	generate bin file
--abi	generate abi file
--include-path node_modules/	indicating the storage path of the library, including .js file and .sol file, which can be used mutiple times to provide mutiple locations.
--base-path . indicating the storage path of the contract is current path.
-o .	indicating the path of the output file
```

```
yarn add ethers	Helping us connect to blockchain nodes and our wallets.
yarn add fs-extra Helping us read and write file
install ethers.js and fs-extra.js in our local enviroment
```

==\*==

> 当node deploy.js 出错
>
> WSL Ganache 出错解决方案

```
reason: 'could not detect network',
  code: 'NETWORK_ERROR',
  event: 'noNetwork'
```

> **Solution1**
>
> windows下面使用WSL连接linux客户端时需要在终端启动Ganache然后获取RPC SERVER Address以及Account‘s private key

``` 
yarn add Ganache
```

```
yarn run Ganache
```

使用Linux Terminal中提供的Ganache RPC Server Address以及private key来部署js

> **Solution2**
>
> 上面的方案只能在linux Termial界面运行，难以看到UI界面
>
> 如果想要看到UI界面
>
> 1. 依照[帖子1](https://github.com/smartcontractkit/full-blockchain-solidity-course-js/discussions/34#discussioncomment-2848150)设置一个专用于WSL的Ganache的SERVER
> 2. 依据[帖子2](https://github.com/smartcontractkit/full-blockchain-solidity-course-js/discussions/34#discussioncomment-3292228)步骤设置Ganache的防火墙

## zoKrates Deploy

### Plan A

Step1 Generate SNARKs File

```shell
node ./scripts/zokrates-demo.js
```

Step2 Compile solidiy file, which is a verify contract

```shell
yarn solcjs --bin --abi --include-path node_modules/ --base-path . -o ./compiledFiles ./contracts/verify-demo.sol 
```

Step3 Deploy contract on **Local blockchain network** `Ganache` or **Ethereum testNet** `goerli`

```shell
node ./scripts/zokrates-demo-deploy.js
```



1. 与yaxian进行交流，如何制定zoKrates文件即criteria的设定 verify function criteria
2. 雇员与管理者之间的1对1关系
3. 关于MPC的理解，performance的判定会涉及到多方
4. 同yaxian进一步的理解zoKrates的使用
5. stenven boit
6. high level presentation

生成结果，

ChatGPT, how to programing

GPTZero

Unity

create with VR --- digital twin construt