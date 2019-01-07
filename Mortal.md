
# Mortal

```Solidity
contract Mortal {
  function kill() {
    if (msg.sender == owner) suicide(owner);
  }
}


contract MyContract is Mortal{
  function test() {

  }
}

```
