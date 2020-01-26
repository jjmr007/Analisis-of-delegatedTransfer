## An Analisis of delegatedTransfer Function and its Advantages over approve Standard Function

A function with the strategy of doing transfer of funds with delegation by the means of signatures, as a possibility and a good idea has been around [*since some time ago*](https://hackernoon.com/you-dont-need-ether-to-transfer-tokens-f3ae373606e1). But concerning stable coins the first team I have notice to have implemented such idea on production is [STASIS-EURS](https://stasis.net/). Let's see what practical uses this new tool may have.  

**The Innovation of the Function _delegatedTransfer_**: many times innovation consists of simple changes that had not been taken seriously before, but when implemented, they highlight how relevant they are for the adoption of a technology.

If the intention of using stable coins, is to promote an adoption that somehow breaks the barrier of the network effect, we need the end user to use this technology without requiring advanced knowledge of what is happening within their application. And one of those barriers is the need to pay commissions of *Gas* in ethers, in order to be able to mobilize funds that exist in another totally different currency: euros and dollars!

How do we explain to the ordinary user that those dollars he wishes to send, will not move from his wallet unless he acquires and places in his same "*address*" a sufficient amount of *ethers*? That is, it was not enough to buy the * Tokens * in stable coin (either dollars or euros), another complex process must also be done to acquire a cryptocurrency, which in principle does not interest the user or does not have to interest to him/her at all; and all this just to buy a magazine, a coffee or pay the utility bill.

This is the technological contribution of the function **_delegatedTransfer_** whose mission is to avoid the end user the need to deal with a cryptocurrency, when his field of action is concentrated in a different currency. The STASIS team developed an application, available for mobile phones with [**_Android_**](https://play.google.com/store/apps/details?id=com.stasis.stasiswallet) operating system, as well as [**_iOS_**](https://apps.apple.com/app/stasis-wallet/id1371949230) mobile phones, that allow the user to send to a *delegate* a request to mobilize funds, through a "client-server" communication channel completely apart from the blockchain (an RPC off-chain call), in order to carry out the transfer of EURS-token. This request is accompanied by a cryptographic signature [ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) under the [Ethereum standard](https://ethereum.stackexchange.com/questions/64380/understanding-ethereum-signatures) comprising three parameters: two 32-byte strings called "* R *" and "* S *" and a one-byte extension number, or "* V *" parameter.

Since this topic already deviates from the objective of this analysis of the strategies for updating contracts used by some stable currencies, I give the briefest possible analysis to the _delegatedTransfer_ function:

**i.- How _delegatedTransfer_ works**. The solidity code of this function is:

```js

function delegatedTransfer (
    address _to, uint256 _value, uint256 _fee,
    uint256 _nonce, uint8 _v, bytes32 _r, bytes32 _s)
  public delegatable payable returns (bool) {
    if (frozen) return false;
    else {
      address _from = ecrecover (
        keccak256 (
          thisAddress (), messageSenderAddress (), _to, _value, _fee, _nonce),
        _v, _r, _s);

      if (_nonce != nonces [_from]) return false;

      if (
        (addressFlags [_from] | addressFlags [_to]) & BLACK_LIST_FLAG ==
        BLACK_LIST_FLAG)
        return false;

      uint256 fee =
        (addressFlags [_from] | addressFlags [_to]) & ZERO_FEE_FLAG == ZERO_FEE_FLAG ?
          0 :
          calculateFee (_value);

      uint256 balance = accounts [_from];
      if (_value > balance) return false;
      balance = safeSub (balance, _value);
      if (fee > balance) return false;
      balance = safeSub (balance, fee);
      if (_fee > balance) return false;
      balance = safeSub (balance, _fee);

      nonces [_from] = _nonce + 1;

      accounts [_from] = balance;
      accounts [_to] = safeAdd (accounts [_to], _value);
      accounts [feeCollector] = safeAdd (accounts [feeCollector], fee);
      accounts [msg.sender] = safeAdd (accounts [msg.sender], _fee);

      Transfer (_from, _to, _value);
      Transfer (_from, feeCollector, fee);
      Transfer (_from, msg.sender, _fee);

      return true;
    }
  }

```

The function takes seven (7) parameters, of which only four (4) of them are variables from the contract environment and the other three (3) constitute simply the ECDSA signature, with the parameters v (uint8), r (bytes32) and s (bytes32). The first four variables are:

 **\_to** (variable type: **_address_**): it is the address where the funds will be transferred. <br>
 **\_value** (**_uint256_**): amount of funds to be transferred. <br>
 **\_fee** (**_uint256_**): fee to be paid to the "*delegate*". <br>
 **\_nonce** (**_uint256_**): single use cryptographic number. <br>
The nonce is a security element that ECDSA signatures require to prevent forgery attacks. For this purpose, the contract that implements **_delegatedTransfer_** must also implement a mapping that keeps the internal nonces account for the addresses that use this function in the contract. In the case of EURSToken, this mapping is an internal variable (**_nonces_**) but it is publicly available using the function:
 
 ```js
 
 function nonce (address _owner) public view delegatable returns (uint256) {
    return nonces [_owner];
  }
 
 ```
 
Finally, what **_delegatedTransfer_** does is verify how many funds the signatory of the message has (the address that originated the signature v, r, s) and confirm that the nonce approved by the signature corresponds to the internal nonce of the contract. In the case of EURSToken, other conditions pertaining to that contract are verified, such as checking that the signatory is not on any AML "*blacklist*" or if the contract is not paused.

If everything is in order, it proceeds with the respective transfers of funds. The amount **_\_value_** is accredited to the account **_\_to_** and the amount **_\_fee_** is accredited to **_msg.sender_** whoever it is, and which perfectly can be a contract or an externally owned account (EOA). The balance of the signatory account is updated with the deductions of **_\_value_** and **_\_fee_**.


**ii.- Why the "*approve*" function does not work but _delegatedTransfer_ does**. According to [ERC20 standard](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) the [recommended configuration](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol) for the function **_approve_** is:

```js

function approve(address spender, uint256 amount) public returns (bool) {
        _approve(_msgSender(), spender, amount);
        return true;
    }

```

where the \_approve function is: 

```js

function _approve(address owner, address spender, uint256 amount) internal {
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");

        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }

```

by the other hand, *\_allowances* is a mapping which registers authorizations or funds delegations:

```js
mapping (address => mapping (address => uint256)) private _allowances;
```

and \_msgSender() is a function that identifies who is invoking the contract:

```js

    function _msgSender() internal view returns (address payable) {
        return msg.sender;
    }

```

What in a few words means that **_approve_** modifies the mapping *\_allowances* assuming as owner of the funds only the **_msg.sender_** address, that is, the entity or element that makes **_directly_** the call to the contract that holds the ERC20 tokens.

There is no way to delegate the management of funds through a contract that intermediates in that transaction; it must be *directly* the owner of these funds. Additionally, the invocation of approve that must necessarily be done through a transaction is only the half the story: once the **_spender_** account has been authorized, it must invoke the function **_transferFrom_** to in deed make use (whatever it is) of the funds. And this must be done in another separate transaction, which requires the interested party to pay twice the minimum amount of gasoline required by a transaction (21,000 units of Gas).

In addition to [other complications](https://blog.smartdec.net/erc20-approve-issue-in-simple-words-a41aaf47bca6) that may arise, this is already difficult to incorporate into an application for users in general, who don't need to know the inner layers of the technology they use.

Unlike this, **_delegatedTransfer_** is a secure function of delegation of funds and at the same time an effective allocation of funds to any entity (whether if contract or normal account), and therefore can be invoked from a contract, through of functions that can invoke other functions, of any other number of contracts that are required **_in a single transaction_**. What makes it possible for the end user to not even have to deal with the purchase of a *cryptocurrency* called Ethereum and additionally what potentially makes the function **_delegatedTransfer_** an ideal substitute for *approve* and *transferFrom* , leaving these functions obsolete in the ERC20 standard.
