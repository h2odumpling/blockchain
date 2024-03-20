# solidity 与 Evm
1. solidity 通过编译变成abi与二进制bin文件
2. bin文件发送至链使用evm进行解析，详见evm.codes

# abi
solidity中的代码可以直接访问的底层编译器
1. 可以将各类数据编译成二进制文件或二进制打包文件，二进制文件占空间较大，但仍可恢复成原来的各类数据，类似于操作性较差的序列号反序列过程，而打包文件变为一个整体，无法通过abi进行拆分
2. 可以调起方法，类似于反射
   ```solidity
   function transfer(address a, uint256 i) public{
        //do something
   }
   //返回solidity对函数识别的hash
   function getSelector() public pure returns(bytes4 selector){
        return bytes4(keccak256(bytes("transfer(address,uint256)")));
   }
   //通过函数hash进行方法调用，类似反射
   function callTransfer(address a, uint256 i) public pure returns(bytes memory){
        return abi.encodeWithSelector(getSelector(), a, i);
   }
   //通过函数签名进行方法调用，类似于反射
   function callTransferWithSign(address a, uint256 i) public pure returns(bytes memory){
        return abi.encodeWithSignature("transfer(address,uint256)", someAddress, amount);
   }
   //控制反射成功情况
   function callTransferDirectly(address a, uint256 i) public pure returns(bytes memory){
        (bool success, bytes memory data) = address(this).call(
            abi.callTransfer(getSelector(), a, i)
        );
        return data;
   }
   ```

# 版本特效
## 0.8.0
0.8.0以后，数值上溢或下溢会直接报错，之前版本会处于循环状态，需使用safeMath进行验证，如不使用溢出检查，可以使用unchecked方法
```solidity
function add() public {
    uint8 num = 255;
    num+=1; //报错
    unchecked{num+=1}; //不报错，num=0，unchecked执行效率更高因为无需进行验证
}
```

# 比较
比较只能用于基础类型如int256、bytes32、bool、address等，无法用于数组或字符串（字符串本质是[]bytes32），比较快速的字符串或数组比较方法是使用hash
```solidity
assert(keccak256(abi.encodePacked(ourToken.name())) == keccak256(abi.encodePacked("OurToken")));
```

# 可见性
1. public
2. private
   仅合约内可见
3. external
   仅外部可调用
4. internal
   仅内部或子合约可调用

# 合约结构
1. 版权申明
2. 版本限制
   ```solidity
   pragma ^0.8.17;
   pragma >=0.8.17 <0.9.0; //等价于^0.8.17
   pragma 0.8.17; //指定版本为0.8.17
   ```
3. 关键字
    ```solidity
    comtract TinyStorage{
        uint256 myData; //状态变量
        address public baseSender;

        constructor (){ //构造器，只在部署时执行一次
            baseSender = msg.sender;    //消息发送者，部署时为部署者
        }

        function getAddressThis() public view returns(address){ //view 表示从storage中读取，view中不能更新状态变量
            reutrn address(this);   //返回当前的合约地址，this表示当前合约
        }

        function getDefauleAddress() public pure returns(string){   //pure表示不从storage中读取变量
            return "1235";
        }

        string myName;

        function store(string memory _name) public{ //memory表示存储在内存中，且可以在函数中改变
            _name = "zz";
            myName = _name;
        }

        mapping(string => uint256) public nToA;

        struct Person{
            string Name;
            uint256 Age;
        }

        Person[] Plist;

        function addPerson(string calldata _name, uint256 _age) public {
            Plist.push(Person(_name, _age));
            nToA[_name] = _age;
        }

        function fund() public payable {
            myValue += 2;   //当require未通过时，require之前执行的内容会被回滚，到回滚前的所有计算都需要支付Gas
            require(msg.value > 1e18, "didn't send enough ETH");
        }

        modifier OnlyUser{
            require(require(funderAmounts[msg.sender] > 0, "you don't have enough deposit"););
            _;  //下划线表示回到函数并执行
        }

        receive() external payable { } //每次接收到交易请求且receive已设置时都会触发，甚至值为0

        fallback() external payable { } //每次请求方法但未找到对应方法时都会触发
    }
    ```
    存储类型
    * calldata，临时存储，只可被读取，不能在函数中修改
    * memory，临时存储，可以被读取，也可在函数中修改
    * storage，永久存储，可以被读取，也可在函数中修改，状态变量就是storage状态
4. 导入
   ```solidity
   import "./xx.sol";
   import "https://xx.sol"
   ```
5. 接口

# 发起交易的方式
1. transfer\
   tranfer发送交易，交易失败会自动回滚
   ```solidity
   payable(msg.sender).transfer(address(this).balance);
   ```
2. send\
   send发送交易，交易失败返回布尔值，可通过布尔值手动回滚
   ```solidity
   sendSuccess = payable(msg.sender).send(address(this).balance);
   reuqire(sendSuccess, "send failed");
   ```
3. call\
   call发送交易，相当于调用指定地址的方法，单纯发起交易可以调用空方法，并且可控性最高，是目前最推荐的发起交易的方式
   ```solidity
   (bool callSuccess,) = payable(msg.sender).call{value: address(this).balance}("");
   require(callSuccess, "call failed")
   ```

# Gas优化
合约运行中的存储原理，所有可变状态变量都存储在额外storage中，
1. 回滚操作尽量靠前执行
2. 尽量使用unchecked进行计算
3. 多使用常量constant及设置后不可改变immutable变量
4. 使用自定义异常及revert代替require
5. 在可以使用abi.encodePacked时代替abi.encode使用

# 开发相关
1. Vscode插件
   1. Solidity
   2. Even Batter TOML
2. foundry安装
   ```sh
   curl -L https://foundry.paradigm.xyz | bash
   foundryup
   ```
   包含forge(项目初始化及开发测试编译)、anvil(本地区块链evm)、cast(已部署合约的交互)、chisel(在终端直接执行solitity)