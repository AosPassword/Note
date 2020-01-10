# Solidity

## 一.简介

- Solidity是一个面向合约，为了实现智能合约而创建的高级编程语言，设计的目的是能在以太坊虚拟机（EVM）上运行

- Solidity是**静态类型语言**，支持继承，库和复杂的用户定义类型等特性
- 内含的类型除了常有的类型，还包括address等独有类型

## 二.Solidity语言特性

​		语法接近于JS，面向对象。

- 以太坊底层基于账户，而不是UTXO，所以增加了一个address用于定位用户和合约账户
- 语言内嵌支持支付，提供payable等关键字
- 数据的每一个状态都可以永久存储
- 一旦错误，所有执行会被回撤

## 三.Solidity源码和智能合约

- 用Solidity编写的源码使用编译器编译为字节码，同时会产生智能合约的二进制接口规范（ABI）。是链上合约和外部交互的接口
- 部署的时候要用交易的方式，把字节码部署到以太坊网络（给0账户发），每次成功部署都会产生一个新的智能合约账户
- 用JS编写的DApp通常通过web3.js+ABI调用智能合约中的函数来实现数据的读取和修改

## 四.Solidity编译器

​		最好用的是Remix，而solc是Solidity源码库的构建目标之一，类似于javac。

## 五.一个简单的合约

```
pragma solidity > 0.4.22								//版本信息

contract Setter{
	uint data;									//默认uint256
	constructor(uint init){
		data = init; 
	}
	function setData(uint input) public{
		data = input;
	}
	function getData() public view returns(uint){	//view表示此函数只读
		return data;
	}
	function add(uint a,uint b) public pure returns (uint sum,uint origin_a,uint origin_b){
		//pure表示此为纯计算函数
		return (a+b,a,b);
	}
}
```

## 六.整个简单的子货币

```
pragma solidity >0.4.22

contract MyCoin{

	mapping(address => uint)public balances;
	//event Sent(address form,address to,uint amount);
	
	constructor(uint init) public {
		balances[msg.sender] = init;
	}
	function mint(address receiver,uint amount) public {
		require(msg.sender = minter);
		balances[receiver] += amount;
	}
	function send(address receiver,uint amount) public return(bool success){
		require(balances[msg.sender]>= amount);
		require(balances[receiver] + amount >= balances[receiver]);
		balances[msg.sender] -= amount;
		balances[receiver] += amount;
		
		//emit Sent(msg.sender,receiver,amount);
		return true;
	}
}
```

​		event Sent（address form，address to，uint amount）

- 声明一个时间，他会在send函数最后一行触发
- 用户可以监听区块链上正在发送的时间，而不用花费太多的成本。一旦它发出，监听该事件的lintener都将收到通知
- 所有的时间都包含了from，to，amount三个参数，可以方便的追踪事务
- emit Sent即触发Sent事件



## 七.康康ERC20
###Method
>- The following specifications use syntax from Solidity `0.4.17` (or above)
>- Callers MUST handle `false` from `returns (bool success)`. Callers MUST NOT assume that `false` is never returned!
####name
返回token的名字，比如说“Mytoken”，范例有“BitCoin”（当然比特币肯定不在以太坊上发行233）
这个方法是可选的，可能存在也可能不存在，这个方法有助于提高可用性，但是接口和其他合约必须不能期望这个方法是存在的
``function name() public view returns (string)``
####symbol
返回token的标志，比如说“BTC”，详细细节和name一样
``function symbol() public view returns (string)``
####decimals
返回令牌的小数位，比如说八位，比如说以太币就是18位，最小单位对应wei嘛。以太坊也建议别人搞成18位，这样方便和以太坊适配，用什么uint默认256位就很好处理。
这个玩意也是可选的，也有助于提高可用性，但是接口和其他合约必须不能期望这个方法是存在的.
``function decimals() public view returns (uint8)``
####totalSupply
Returns the total token supply
返回提供的token总量，这个必须有，不知道币的总量谁敢买你的币
``function totalSupply() public view returns (uint256)``
####balanceOf
返回用户账户里面有多少币，这个也必须有，这都没有做个锤子币
####transfer
这个方法用来给别人的账户转币，如果调用者的账户里的币比他要转的少，那一定得throw。
如果一个人要传0个币，那必须正常的传输并且触发transfer事件
很明显这个也必须有，一个币没法流通那还整个锤子
``function transfer(address _to, uint256 _value) public returns (bool success)``

####transfrom
``function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)``
将_value的token从_from转到to，必须触发transfer事件
这个transfrom方法用来撤回工作流，让合约用你的名义来转移币
说白了就是A可以用这个方法将B的币打到C的账户上
但是，在A调用这个方法的时候，必须检查B是否给了A这个转币的权限和限额，如果没有，则应当throw
####approve
``function approve(address _spender, uint256 _value) public returns (bool success)``
这个方法用于B授予A（即spender）转自己账户里的钱的权利，value表示A能动B的多少钱。假如设定了1000，A要用B的账户给C转2000，就会throw，如果转500，就成功
如果B多次调用这个方法，后一次的value将会覆盖前一次。
为了防备攻击，这个方法要有些特殊措施，这个之后再说
####allowance
```function allowance(address _owner, address _spender) public view returns (uint256 remaining)``
返回owner让spender可以用的金额。

###Event
####transfer
``event Transfer(address indexed _from, address indexed _to, uint256 _value)``

token的transfer必须触发event，包括0value的transfer

#### Approval

```
event Approval(address indexed _owner, address indexed _spender, uint256 _value)

```

成功调用approve之后必须被触发

### 举个实例

从官方那里抄来的。

感觉其实还是挺简单的。

```
pragma solidity ^0.5.0;

/**
 * @dev Interface of the ERC20 standard as defined in the EIP. Does not include
 * the optional functions; to access them see {ERC20Detailed}.
 */
interface IERC20 {
    /**
     * @dev Returns the amount of tokens in existence.
     */
    function totalSupply() external view returns (uint256);

    /**
     * @dev Returns the amount of tokens owned by `account`.
     */
    function balanceOf(address account) external view returns (uint256);

    /**
     * @dev Moves `amount` tokens from the caller's account to `recipient`.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * Emits a {Transfer} event.
     */
    function transfer(address recipient, uint256 amount) external returns (bool);

    /**
     * @dev Returns the remaining number of tokens that `spender` will be
     * allowed to spend on behalf of `owner` through {transferFrom}. This is
     * zero by default.
     *
     * This value changes when {approve} or {transferFrom} are called.
     */
    function allowance(address owner, address spender) external view returns (uint256);

    /**
     * @dev Sets `amount` as the allowance of `spender` over the caller's tokens.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * IMPORTANT: Beware that changing an allowance with this method brings the risk
     * that someone may use both the old and the new allowance by unfortunate
     * transaction ordering. One possible solution to mitigate this race
     * condition is to first reduce the spender's allowance to 0 and set the
     * desired value afterwards:
     * https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
     *
     * Emits an {Approval} event.
     */
    function approve(address spender, uint256 amount) external returns (bool);

    /**
     * @dev Moves `amount` tokens from `sender` to `recipient` using the
     * allowance mechanism. `amount` is then deducted from the caller's
     * allowance.
     *
     * Returns a boolean value indicating whether the operation succeeded.
     *
     * Emits a {Transfer} event.
     */
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);

    /**
     * @dev Emitted when `value` tokens are moved from one account (`from`) to
     * another (`to`).
     *
     * Note that `value` may be zero.
     */
    event Transfer(address indexed from, address indexed to, uint256 value);

    /**
     * @dev Emitted when the allowance of a `spender` for an `owner` is set by
     * a call to {approve}. `value` is the new allowance.
     */
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

```
pragma solidity ^0.5.0;

/**
 * @dev Wrappers over Solidity's arithmetic operations with added overflow
 * checks.
 *
 * Arithmetic operations in Solidity wrap on overflow. This can easily result
 * in bugs, because programmers usually assume that an overflow raises an
 * error, which is the standard behavior in high level programming languages.
 * `SafeMath` restores this intuition by reverting the transaction when an
 * operation overflows.
 *
 * Using this library instead of the unchecked operations eliminates an entire
 * class of bugs, so it's recommended to use it always.
 */
library SafeMath {
    /**
     * @dev Returns the addition of two unsigned integers, reverting on
     * overflow.
     *
     * Counterpart to Solidity's `+` operator.
     *
     * Requirements:
     * - Addition cannot overflow.
     */
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");

        return c;
    }

    /**
     * @dev Returns the subtraction of two unsigned integers, reverting on
     * overflow (when the result is negative).
     *
     * Counterpart to Solidity's `-` operator.
     *
     * Requirements:
     * - Subtraction cannot overflow.
     */
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        return sub(a, b, "SafeMath: subtraction overflow");
    }

    /**
     * @dev Returns the subtraction of two unsigned integers, reverting with custom message on
     * overflow (when the result is negative).
     *
     * Counterpart to Solidity's `-` operator.
     *
     * Requirements:
     * - Subtraction cannot overflow.
     *
     * _Available since v2.4.0._
     */
    function sub(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
        require(b <= a, errorMessage);
        uint256 c = a - b;

        return c;
    }

    /**
     * @dev Returns the multiplication of two unsigned integers, reverting on
     * overflow.
     *
     * Counterpart to Solidity's `*` operator.
     *
     * Requirements:
     * - Multiplication cannot overflow.
     */
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        // Gas optimization: this is cheaper than requiring 'a' not being zero, but the
        // benefit is lost if 'b' is also tested.
        // See: https://github.com/OpenZeppelin/openzeppelin-contracts/pull/522
        if (a == 0) {
            return 0;
        }

        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");

        return c;
    }

    /**
     * @dev Returns the integer division of two unsigned integers. Reverts on
     * division by zero. The result is rounded towards zero.
     *
     * Counterpart to Solidity's `/` operator. Note: this function uses a
     * `revert` opcode (which leaves remaining gas untouched) while Solidity
     * uses an invalid opcode to revert (consuming all remaining gas).
     *
     * Requirements:
     * - The divisor cannot be zero.
     */
    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        return div(a, b, "SafeMath: division by zero");
    }

    /**
     * @dev Returns the integer division of two unsigned integers. Reverts with custom message on
     * division by zero. The result is rounded towards zero.
     *
     * Counterpart to Solidity's `/` operator. Note: this function uses a
     * `revert` opcode (which leaves remaining gas untouched) while Solidity
     * uses an invalid opcode to revert (consuming all remaining gas).
     *
     * Requirements:
     * - The divisor cannot be zero.
     *
     * _Available since v2.4.0._
     */
    function div(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
        // Solidity only automatically asserts when dividing by 0
        require(b > 0, errorMessage);
        uint256 c = a / b;
        // assert(a == b * c + a % b); // There is no case in which this doesn't hold

        return c;
    }

    /**
     * @dev Returns the remainder of dividing two unsigned integers. (unsigned integer modulo),
     * Reverts when dividing by zero.
     *
     * Counterpart to Solidity's `%` operator. This function uses a `revert`
     * opcode (which leaves remaining gas untouched) while Solidity uses an
     * invalid opcode to revert (consuming all remaining gas).
     *
     * Requirements:
     * - The divisor cannot be zero.
     */
    function mod(uint256 a, uint256 b) internal pure returns (uint256) {
        return mod(a, b, "SafeMath: modulo by zero");
    }

    /**
     * @dev Returns the remainder of dividing two unsigned integers. (unsigned integer modulo),
     * Reverts with custom message when dividing by zero.
     *
     * Counterpart to Solidity's `%` operator. This function uses a `revert`
     * opcode (which leaves remaining gas untouched) while Solidity uses an
     * invalid opcode to revert (consuming all remaining gas).
     *
     * Requirements:
     * - The divisor cannot be zero.
     *
     * _Available since v2.4.0._
     */
    function mod(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
        require(b != 0, errorMessage);
        return a % b;
    }
}
```



```
pragma solidity ^0.4.24;

import "./IERC20.sol";
import "../../math/SafeMath.sol";

/**
 * @title Standard ERC20 token
 *
 * @dev Implementation of the basic standard token.
 * https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md
 * Originally based on code by FirstBlood: https://github.com/Firstbloodio/token/blob/master/smart_contract/FirstBloodToken.sol
 */
contract ERC20 is IERC20 {
  using SafeMath for uint256;///这是个什么东西

  mapping (address => uint256) private _balances;

  mapping (address => mapping (address => uint256)) private _allowed;

  uint256 private _totalSupply;

  /**
  * @dev Total number of tokens in existence
  */
  function totalSupply() public view returns (uint256) {
    return _totalSupply;
  }

  /**
  * @dev Gets the balance of the specified address.
  * @param owner The address to query the balance of.
  * @return An uint256 representing the amount owned by the passed address.
  */
  function balanceOf(address owner) public view returns (uint256) {
    return _balances[owner];
  }

  /**
   * @dev Function to check the amount of tokens that an owner allowed to a spender.
   * @param owner address The address which owns the funds.
   * @param spender address The address which will spend the funds.
   * @return A uint256 specifying the amount of tokens still available for the spender.
   */
  function allowance(
    address owner,
    address spender
   )
    public
    view
    returns (uint256)
  {
    return _allowed[owner][spender];
  }

  /**
  * @dev Transfer token for a specified address
  * @param to The address to transfer to.
  * @param value The amount to be transferred.
  */
  function transfer(address to, uint256 value) public returns (bool) {
    require(value <= _balances[msg.sender]);
    require(to != address(0));

    _balances[msg.sender] = _balances[msg.sender].sub(value);
    _balances[to] = _balances[to].add(value);
    emit Transfer(msg.sender, to, value);
    return true;
  }

  /**
   * @dev Approve the passed address to spend the specified amount of tokens on behalf of msg.sender.
   * Beware that changing an allowance with this method brings the risk that someone may use both the old
   * and the new allowance by unfortunate transaction ordering. One possible solution to mitigate this
   * race condition is to first reduce the spender's allowance to 0 and set the desired value afterwards:
   * https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
   * @param spender The address which will spend the funds.
   * @param value The amount of tokens to be spent.
   */
  function approve(address spender, uint256 value) public returns (bool) {
    require(spender != address(0));

    _allowed[msg.sender][spender] = value;
    emit Approval(msg.sender, spender, value);
    return true;
  }

  /**
   * @dev Transfer tokens from one address to another
   * @param from address The address which you want to send tokens from
   * @param to address The address which you want to transfer to
   * @param value uint256 the amount of tokens to be transferred
   */
  function transferFrom(
    address from,
    address to,
    uint256 value
  )
    public
    returns (bool)
  {
    require(value <= _balances[from]);
    require(value <= _allowed[from][msg.sender]);
    require(to != address(0));

    _balances[from] = _balances[from].sub(value);
    _balances[to] = _balances[to].add(value);
    _allowed[from][msg.sender] = _allowed[from][msg.sender].sub(value);
    emit Transfer(from, to, value);
    return true;
  }

  /**
   * @dev Increase the amount of tokens that an owner allowed to a spender.
   * approve should be called when allowed_[_spender] == 0. To increment
   * allowed value is better to use this function to avoid 2 calls (and wait until
   * the first transaction is mined)
   * From MonolithDAO Token.sol
   * @param spender The address which will spend the funds.
   * @param addedValue The amount of tokens to increase the allowance by.
   */
  function increaseAllowance(
    address spender,
    uint256 addedValue
  )
    public
    returns (bool)
  {
    require(spender != address(0));

    _allowed[msg.sender][spender] = (
      _allowed[msg.sender][spender].add(addedValue));
    emit Approval(msg.sender, spender, _allowed[msg.sender][spender]);
    return true;
  }

  /**
   * @dev Decrease the amount of tokens that an owner allowed to a spender.
   * approve should be called when allowed_[_spender] == 0. To decrement
   * allowed value is better to use this function to avoid 2 calls (and wait until
   * the first transaction is mined)
   * From MonolithDAO Token.sol
   * @param spender The address which will spend the funds.
   * @param subtractedValue The amount of tokens to decrease the allowance by.
   */
  function decreaseAllowance(
    address spender,
    uint256 subtractedValue
  )
    public
    returns (bool)
  {
    require(spender != address(0));

    _allowed[msg.sender][spender] = (
      _allowed[msg.sender][spender].sub(subtractedValue));
    emit Approval(msg.sender, spender, _allowed[msg.sender][spender]);
    return true;
  }

  /**
   * @dev Internal function that mints an amount of the token and assigns it to
   * an account. This encapsulates the modification of balances such that the
   * proper events are emitted.
   * @param account The account that will receive the created tokens.
   * @param amount The amount that will be created.
   */
  function _mint(address account, uint256 amount) internal {
    require(account != 0);
    _totalSupply = _totalSupply.add(amount);
    _balances[account] = _balances[account].add(amount);
    emit Transfer(address(0), account, amount);
  }

  /**
   * @dev Internal function that burns an amount of the token of a given
   * account.
   * @param account The account whose tokens will be burnt.
   * @param amount The amount that will be burnt.
   */
  function _burn(address account, uint256 amount) internal {
    require(account != 0);
    require(amount <= _balances[account]);

    _totalSupply = _totalSupply.sub(amount);
    _balances[account] = _balances[account].sub(amount);
    emit Transfer(account, address(0), amount);
  }

  /**
   * @dev Internal function that burns an amount of the token of a given
   * account, deducting from the sender's allowance for said account. Uses the
   * internal burn function.
   * @param account The account whose tokens will be burnt.
   * @param amount The amount that will be burnt.
   */
  function _burnFrom(address account, uint256 amount) internal {
    require(amount <= _allowed[account][msg.sender]);

    // Should https://github.com/OpenZeppelin/zeppelin-solidity/issues/707 be accepted,
    // this function needs to emit an event with the updated approval.
    _allowed[account][msg.sender] = _allowed[account][msg.sender].sub(
      amount);
    _burn(account, amount);
  }
}
```


## 八.Ballot投票

从Remix上抄来的源码，感觉也挺简单的

```
pragma solidity >=0.4.22 <0.6.0;
contract Ballot {
	///投票者
    struct Voter {
        uint weight;
        bool voted;
        uint8 vote;
        address delegate;//选个人代表你去投票
    }
    struct Proposal {
        uint voteCount;
    }

    address chairperson;
    mapping(address => Voter) voters;
    Proposal[] proposals;

    /// Create a new ballot with $(_numProposals) different proposals.
    ///初始化chairperson和proposals数量
    constructor(uint8 _numProposals) public {
        chairperson = msg.sender;
        voters[chairperson].weight = 1;
        proposals.length = _numProposals;
    }

    /// Give $(toVoter) the right to vote on this ballot.授予别人投票权
    /// May only be called by $(chairperson).只可以被chairperson调用
    function giveRightToVote(address toVoter) public {
        if (msg.sender != chairperson || voters[toVoter].voted) return;
        voters[toVoter].weight = 1;
    }

    /// Delegate your vote to the voter $(to).将你的投票权给别人，让别人代表你投票
    function delegate(address to) public {
        Voter storage sender = voters[msg.sender]; // assigns reference，在storage存储
        if (sender.voted) return;
        ///我代理的代理就是我的代理
        while (voters[to].delegate != address(0) && voters[to].delegate != msg.sender)
            to = voters[to].delegate;
            //此处并不会产生单向环，因为产生单向环的那一刻，其to必为msg.sender
            //会在下一步return，此次delegeate失败
        if (to == msg.sender) return;
        sender.voted = true;
        sender.delegate = to;
        Voter storage delegateTo = voters[to];
        if (delegateTo.voted)
            proposals[delegateTo.vote].voteCount += sender.weight;
        else
            delegateTo.weight += sender.weight;
    }

    /// Give a single vote to proposal $(toProposal).
    function vote(uint8 toProposal) public {
        Voter storage sender = voters[msg.sender];
        if (sender.voted || toProposal >= proposals.length) return;
        sender.voted = true;
        sender.vote = toProposal;
        proposals[toProposal].voteCount += sender.weight;
    }
	///选出投票数最多的proposal
    function winningProposal() public view returns (uint8 _winningProposal) {
        uint256 winningVoteCount = 0;
        for (uint8 prop = 0; prop < proposals.length; prop++)
            if (proposals[prop].voteCount > winningVoteCount) {
                winningVoteCount = proposals[prop].voteCount;
                _winningProposal = prop;
            }
    }
}
```

## 九.理解Solidity

### pragma

- 版本杂注，表明要求的编译器版本
- 例如``pragma solidity ^ 0.4.0``指的是，不允许低于0.4.0且不高于0.5.0版本的编译器编译

### import

```
import "filename";
```

导入filename中的所有全局变量到当前全局作用域中

```
import * as symbolName from "filename";
```

创建也给新的全局符号symbolName，其成员与filename的全局符号相同

```
import {symbol1 as alias,symbol2} from "filename";
```

创建新的全局符号alias和symbol2，分别从filename中引用symbol1和symbol2

```
import "filename" as symbolName;
```

等同于``import * as symbolName from "filename";``

### Solidity值类型

- bool
- int/uint,支持uint8到uint256
- 定长浮点型（fixed/ufixed），uinxedMxN与fixedMxN，M表示该类型占用位数，N为可用小数位数，M是8-256，N是0-80，必须是8的倍数
- address，20字节的值，160位
- 定长字节数组，bytes（16进制表示），从bytes1到bytes32
- 枚举enum，默认从0递增，一般用俩模拟合约状态
- 函数function
- string，不定长字符串

### Solidity引用类型

- 数组Array，分为定长和定长。对于storage数组来讲，元素类型可以是任意的，而对于内存型memory数组来说，元素类型不能是map。
- 结构（struct），和c差不多
- 映射map，可以视作hash表，在实际的初始化过程中就会创建每个可能的key，并将其映射到字节形式全是0的值

###address

- 存储20字节的值
- 0.5.0引入了address payable，必须是payable的地址才可以调用transfer，或者是接受别人发的币，但是普通的地址无法发送或者接受ether
- 一个普通的address无法转换为address payable，但是address payable可以转化为address，但是可以用uint160进行转换
- 0.5.0之前，合约直接继承address，但是之后，合约不再由地址类型直接派生而出，如果合约有payable回退函数，可以显示转换为address或者address payable。但是如果没有，就无法给合约转钱
- 有挺多的成员变量和方法
  - address.balance，返回uint256，即地址的ether余额（wei） 
  - address payable.transfer（uint256 amount），合约给address以太币,失败则throw异常
  - address payable.send(uint256 amount)，合约给address以太币，失败返回false，send和transfer调用时消耗2300gas，不可调
  - address.call(bytes memory) return (bool,bytes memory)
    - 调用函数，失败返回false，默认发送所有可用gas，这个发的gas是可调的
  - address.delegatecall(bytes memory) return (bool,bytes memory)
    - 代理调用函数，就是EVM中所说的委托调用，例如可以用来传递msg.sender
    - 默认发送所有的可用gas，可调
  - address.staticcall(bytes memory) return (bool,bytes memory)
    - 静态调用，没有状态改变
  - 这个call的函数都少用

### Bytes，Emun，Struct

比较简单，懒得写了

### Array

- 固定大小n，固定元素类型T，数组写作T[n]，如果是动态数组大小则为T[]，五个uint动态数组组成的数组是``uint[][5]``
- 访问第三个动态数组中的第二个uint，则用``T[2][1]``
- 越界访问，会调用失败回退
- 添加新元素用.push(),或者将.length增大
- 变长的storage数组和bytes有push，可以将一个新元素假如数组末端，返回当前长度
- storage中的动态数组的length可变，但是memory中的动态数组的length不可变

### Mapping

key=>value

- key可以是任何**基本**类型，可以是任何内置值类型加上字符数组和字符串。不能用用户定义的或者复杂的类型，比如枚举，映射，结构，以及除了bytes和string之外的任何数组类型
- value可以是任何类型，包括映射

### Solidity数据位置

- 所有的复杂类型，即数组，结构和映射，都有一个额外属性，数据位置，表示其是存储在memory中的还是storage中
- 大多数情况下数据有默认位置，也可以通过加关键字修改
- 函数参数，包括返回参数的数据位置默认为memory，**局部变量**的数据的默认位置是**storage**，**状态变量**的数据位置**强制storage**
- calldata是只读的，不会永久存储的位置，用来存储函数参数。外部函数的参数（非返回参数）的数据位置强制为calldata，只是用来将外部函数和内部函数区分的，和memory其实差不多‘
- public的函数参数一定是memory类型，如果要求是storage类型，则函数必须是private或者internal，为了防止随意的公开调用占用资源

```
pragma solidity ^0.4.0

contract C{
///都在storage
	uint[] data1;
	uint[] data2;
	
	function appendOne() public{
		append(data1);
	}
	function appendTwo() public{
		append(data2);
	}
	function append(uint[] storage d) internal{
	///如果d前面不加storage，则d默认为memory，无法使用push
	///如果把internal改成publc，则编译器报错，因为public函数的参数强制为memory
		d.push(233);
	}
}
```

再试一个神奇的代码

```
pragma solidity ^0.4.0

contract Bug{
	uint public a;
	uint public b;
	uint[] public data;
	function f() public{
		uint[] x;
		x.push(2);
		data = x;
	}
}
```

当以上代码调用f()之后，data会变成2，但是a会变成1，b则保持默认的0。如果再一次调用f()，那么a会变成2，而b依旧是0

uint[] x是一个变长数组的指针，并且她所在的空间是stroage，他在没有定义的情况下是一个野指针，以太坊默认指向的位置是整个程序的储存空间的头位置，也就是a的位置。而因为storage这个存储空间可以看作一个巨大的hash表，uint[]在storage中会直接存自己长度，也就是说这个x这个指针会将这个变长数组的长度存到他所指向的空间位置，如果要拿到变长数组的内容，需要根据这个指针指向的位置和这个位置对应的长度结合，作为key从hash表里取出来。

所以a在这里存的其实是x这个变长数组的length。

另外再举一个例子：

```
pragma solidity ^0.4.0;
contract Con{
	uint[] x;
	function f(uint[] memoryArray) public{
		x = memoryArray;
		uint[] y = x;
		y.length = 2;
		y = memoryArray;//这里有错误
	}
}
```

在上面的一个例子里，因为memoryArray是默认存储在memory中的，而x是存储在storage中的，当uint[] y = x，y成为了一个指向x的指针，储存在storage中，而x  = memoryArray并不是将x的指针指向了memoryArray，因为它们本身就不在同一个存储空间中，x会直接将memoryArray中的数据完全拷贝到自己的存储空间中。

当你使用y = memoryArray的时候，y作为一个storage中的指针，无法拷贝存储在memory中的数据，所以编译器会报错。

### Solidity可见性

可见性可以指定为external，public，internal，private，对于状态变量，不能设置为external，默认为internal

属性不能使用external来修饰，可以用public，internal和private

如果对一个属性使用public关键字，其实就是编译器默认给这个属性添加了一个public的getter方法。并且这个方法是view的。

如果也给函数被设定成external，则此函数只能外部调用而不能内部调用。但是我们可以通过this.f()（this属于外部访问）来调用。

### Solidity函数可变性

- pure，纯函数，不改变或修改状态
- view，不允许修改状态
- payable，允许从消息调用中接受以太币
- constant，与view相同，一般只修饰状态变量，不允许赋值（初始化可以赋值）。 如果修饰函数，则和view一样

#### 什么叫修改状态

- 修改状态变量
- 产生事件
- 创建其他合约
- 自毁
- 通过调用发送以太币
- 调用任何没有被pure和view修饰的函数
- 低级调用call
- 内联汇编（不太懂）

####什么是读取状态

- 读取状态变量
- 访问this.balance或者address.balance
- 访问block，tx，msg中的任意成员（除了msg.sig和msg.data)
- 调用任何未标记为pure的函数
- 使用包含某些操作码的内联汇编（不懂）

### Modifier（函数修饰器）

例子

```
pragma solidity ^0.4.22;
contract Purchase{
	address public seller;
	constructor() public {
		seller = msg,sender;
	}
	modifier onlySeller(){
		require(msg.sender == seller,"only seller can call");
		_;///表示在调用别的函数之前先检查require，之后再调用别的函数
		///其实就是一个切面嘛，和Spring里面的AOP差不多
		///下划线代指被修饰的其他函数的主体
	}
	function f() public view onlySeller{
	}
}
```

### 回退函数（fallback）

- 没有名字，没有参数，没有fanhuizhi
- 如果在一个合约的调用时，没有其他函数和给定的函数标识符匹配，则调用回退函数
- 当合约收到以太币，但没有任何数据的时候，则调用回退函数。也因此回退函数必须标记为payable，如果不存在遮掩固定函数，则合约不能通过常规交易中接受以太币
  - 要小心回退函数调用其他合约的回退函数，其他合约的回退函数再调用你的函数，the DAO就是这么发生的
- 上下文通常只有很少的gas用来完成回退函数，所以回退函数的调用应尽量廉价

### Event

- 事件是一以太坊EVM提供的一种日志基础设施，事件可以操作记录，存储为日志，可以实现一些简单的交互功能，比如说通知UI，返回函数调用结果等。
- 当定义的事件触发的时候，我们可以将事件存储到EVM的交易日志中，日志是区块链中的一种特殊的数据结构，日志和合约相关联，和合约存储合并存入区块链中，只要某个区块可以访问，其相关的日志即可访问。但是合约中并不能直接访问日志和时间数据
- 可以通过日志实现简单支付验证SPV，如果一个外部实体提供了一个带有这种证明的合约，他可以检查日志是否真的存储于区块链中













 











