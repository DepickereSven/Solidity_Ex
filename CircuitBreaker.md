# CircuitBreaker

```Solidity
contract Owned  {
  address owner;

  modifier onlyOwner() {
    if (msg.sender==owner) _
  }

  function owned() {
    owner = msg.sender;
  }

  function changeOwner(address newOwner) onlyOwner {
    owner = newOwner;
  }

  function getOwner() constant returns (address){
    return owner;
  }
}


contract CircuitBreaker is Owned {
    bool stopped;

    function circuitBreaker() {
        stopped = false;
        owned();
    }

    function toggleCircuit() onlyOwner public {
        stopped = !stopped;
    }

    modifier isStopped() {
        if (!stopped) throw;
        _
    }

    modifier notStopped() {
        if (stopped) throw;
        _
    }

}

contract Crowdfunding is CircuitBreaker{
    struct Backer {
        address addr;
        uint amount;
    }

    uint public numBackers;
    uint public deadline;
    string public campaignStatus;
    bool ended;
    uint public goal;
    uint public amountRaised;
    mapping (uint => Backer) backers;

    event Deposit(address _from, uint _amount);
    event Refund(address _to, uint _amount);


    function Crowdfunding(uint _deadline, uint _goal) {
        owner = msg.sender;
        deadline = _deadline;
        goal = _goal;
        campaignStatus = "Funding";
        numBackers = 0;
        amountRaised = 0;
        ended = false;
        circuitBreaker();
    }

    function fund() notStopped{
        Backer b = backers[numBackers++];
        b.addr = msg.sender;
        b.amount = msg.value;
        amountRaised += b.amount;
        Deposit(msg.sender, msg.value);
    }

    function checkGoalReached () onlyOwner returns (bool ended){
        if (ended)
            throw; // this function has already been called

        if(block.timestamp<deadline)
            throw;

        if (amountRaised >= goal) {
            campaignStatus = "Campaign Succeeded";
            ended = true;
            if (!owner.send(this.balance))
                throw; // If anything fails,
                       // this will revert the changes above
        }else{
            uint i = 0;
            campaignStatus = "Campaign Failed";
            ended = true;
            while (i <= numBackers){
                backers[i].amount = 0;
                if (!backers[i].addr.send(backers[i].amount))
                    throw; // If anything fails,
                           // this will revert the changes above
                Refund(backers[i].addr, backers[i].amount);
                i++;
            }
        }
    }

    function kill() {
        if (msg.sender == owner) {
            suicide(owner);
        }
    }

}

```
