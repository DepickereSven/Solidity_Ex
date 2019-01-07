# AccessRestriction

```Solidity
pragma solidity ^0.4.11;

contract AccessRestriction {
    // These will be assigned at the construction
    // phase, where `msg.sender` is the account
    // creating this contract.
    address public owner = msg.sender;
    uint public creationTime = now;

    // Modifiers can be used to change
    // the body of a function.
    // If this modifier is used, it will
    // prepend a check that only passes
    // if the function is called from
    // a certain address.
    modifier onlyBy(address _account)
    {
        require(msg.sender == _account);
        // Do not forget the "_;"! It will
        // be replaced by the actual function
        // body when the modifier is used.
        _;
    }

    /// Make `_newOwner` the new owner of this
    /// contract.
    function changeOwner(address _newOwner)
        onlyBy(owner)
    {
        owner = _newOwner;
    }

    modifier onlyAfter(uint _time) {
        require(now >= _time);
        _;
    }

    /// Erase ownership information.
    /// May only be called 6 weeks after
    /// the contract has been created.
    function disown()
        onlyBy(owner)
        onlyAfter(creationTime + 6 weeks)
    {
        delete owner;
    }

    // This modifier requires a certain
    // fee being associated with a function call.
    // If the caller sent too much, he or she is
    // refunded, but only after the function body.
    // This was dangerous before Solidity version 0.4.0,
    // where it was possible to skip the part after `_;`.
    modifier costs(uint _amount) {
        require(msg.value >= _amount);
        _;
        if (msg.value > _amount)
            msg.sender.send(msg.value - _amount);
    }

    function forceOwnerChange(address _newOwner)
        costs(200 ether)
    {
        owner = _newOwner;
        // just some example condition
        if (uint(owner) & 0 == 1)
            // This did not refund for Solidity
            // before version 0.4.0.
            return;
        // refund overpaid fees
    }
}

```

# AutomaticExpiration

```Solidity
contract AutoExpires {
    uint expires;

    function AutoExpires(uint t) {
        expires = now + t;
    }

    function expired() returns (bool) {
        return now > expires ? true : false;
    }

    modifier isExpired() {
        if (!expired()) throw;
        _
    }

    modifier notExpired() {
        if (expired()) throw;
        _
    }
}

contract Voting is AutoExpires{
    // Declare a complex type to reresent a single voter.
    struct Voter {
        bool voted;  // if true, that person already voted
        uint vote;   // index of the voted proposal
        bool rightToVote; // if true, that person has right to vote
    }

    // This is a type for a single proposal.
    struct Proposal
    {
        bytes32 name;   // short name
        uint voteCount; // number of accumulated votes
    }

    address public chairperson;

    uint public numProposals;

    mapping(address => Voter) public voters;
    mapping (uint => Proposal) public proposals;

    function Voting(bytes32[] proposalNames, uint _votingTime) {
        chairperson = msg.sender;
        voters[chairperson].rightToVote = true;
        AutoExpires(_votingTime);

        numProposals=proposalNames.length;

        // For each of the provided proposal names,
        // create a new proposal object and add it
        // to the end of the array.
        for (uint i = 0; i < proposalNames.length; i++) {
            Proposal p = proposals[i];
            p.name = proposalNames[i];
            p.voteCount = 0;
        }
    }

    function giveRightToVote(address voter) notExpired {
        if (msg.sender != chairperson || voters[voter].voted) {
            // `throw` terminates and reverts all changes to the state
            throw;
        }
        voters[voter].rightToVote = true;
    }

    function vote(uint proposal)  notExpired {
        Voter sender = voters[msg.sender];

        if (sender.voted)
            throw;
        sender.voted = true;
        sender.vote = proposal;

        proposals[proposal].voteCount += 1;
    }

    function winningProposal() constant isExpired
            returns (uint winningProposal, bytes32 proposalName)
    {
        uint winningVoteCount = 0;
        for (uint p = 0; p < numProposals; p++) {
            if (proposals[p].voteCount > winningVoteCount) {
                winningVoteCount = proposals[p].voteCount;
                winningProposal = p;
                proposalName = proposals[p].name;
            }
        }
    }
}

```

# BankingApplication


```Solidity

contract DougEnabled {
    address DOUG;

    function setDougAddress(address dougAddr) returns (bool result){
        // Once the doug address is set, don't allow it to be set again, except by the
        // doug contract itself.
        if(DOUG != 0x0 && msg.sender != DOUG){
            return false;
        }
        DOUG = dougAddr;
        return true;
    }

    // Makes it so that Doug is the only contract that may kill it.
    function remove(){
        if(msg.sender == DOUG){
            selfdestruct(DOUG);
        }
    }

}

// The Doug contract.
contract Doug {

    address owner;

    // This is where we keep all the contracts.
    mapping (bytes32 => address) public contracts;

    modifier onlyOwner { //a modifier to reduce code replication
        if (msg.sender == owner) // this ensures that only the owner can access the function
            _
    }
    // Constructor
    function Doug(){
        owner = msg.sender;
    }

    // Add a new contract to Doug. This will overwrite an existing contract.
    function addContract(bytes32 name, address addr) onlyOwner returns (bool result) {
        DougEnabled de = DougEnabled(addr);
        // Don't add the contract if this does not work.
        if(!de.setDougAddress(address(this))) {
            return false;
        }
        contracts[name] = addr;
        return true;
    }

    // Remove a contract from Doug. We could also selfdestruct if we want to.
    function removeContract(bytes32 name) onlyOwner returns (bool result) {
        if (contracts[name] == 0x0){
            return false;
        }
        contracts[name] = 0x0;
        return true;
    }

    function remove() onlyOwner {
        address fm = contracts["fundmanager"];
        address perms = contracts["perms"];
        address permsdb = contracts["permsdb"];
        address bank = contracts["bank"];
        address bankdb = contracts["bankdb"];

        // Remove everything.
        if(fm != 0x0){ DougEnabled(fm).remove(); }
        if(perms != 0x0){ DougEnabled(perms).remove(); }
        if(permsdb != 0x0){ DougEnabled(permsdb).remove(); }
        if(bank != 0x0){ DougEnabled(bank).remove(); }
        if(bankdb != 0x0){ DougEnabled(bankdb).remove(); }

        // Finally, remove doug. Doug will now have all the funds of the other contracts,
        // and when suiciding it will all go to the owner.
        selfdestruct(owner);
    }

}

// Interface for getting contracts from Doug
contract ContractProvider {
    function contracts(bytes32 name) returns (address addr) {}
}

// Base class for contracts that only allow the fundmanager to call them.
// Note that it inherits from DougEnabled
contract FundManagerEnabled is DougEnabled {

    // Makes it easier to check that fundmanager is the caller.
    function isFundManager() constant returns (bool) {
        if(DOUG != 0x0){
            address fm = ContractProvider(DOUG).contracts("fundmanager");
            return msg.sender == fm;
        }
        return false;
    }
}

// Permissions database
contract PermissionsDb is DougEnabled {

    mapping (address => uint8) public perms;

    // Set the permissions of an account.
    function setPermission(address addr, uint8 perm) returns (bool res) {
        if(DOUG != 0x0){
            address permC = ContractProvider(DOUG).contracts("perms");
            if (msg.sender == permC ){
                perms[addr] = perm;
                return true;
            }
            return false;
        } else {
            return false;
        }
    }

}

// Permissions
contract Permissions is FundManagerEnabled {

    // Set the permissions of an account.
    function setPermission(address addr, uint8 perm) returns (bool res) {
        if (!isFundManager()){
            return false;
        }
        address permdb = ContractProvider(DOUG).contracts("permsdb");
        if ( permdb == 0x0 ) {
            return false;
        }
        return PermissionsDb(permdb).setPermission(addr, perm);
    }

    // Set the permissions of an account.
    function getPermission(address addr) returns (uint8 perm) {
        if (!isFundManager()){
            throw;
        }
        address permdb = ContractProvider(DOUG).contracts("permsdb");
        if ( permdb == 0x0 ) {
            throw;
        }
        return PermissionsDb(permdb).perms(addr);
    }


}

// The bank database
contract BankDb is DougEnabled {

    mapping (address => uint) public balances;

    function deposit(address addr) returns (bool res) {
        if(DOUG != 0x0){
            address bank = ContractProvider(DOUG).contracts("bank");
            if (msg.sender == bank ){
                balances[addr] += msg.value;
                return true;
            }
        }
        // Return if deposit cannot be made.
        msg.sender.send(msg.value);
        return false;
    }

    function withdraw(address addr, uint amount) returns (bool res) {
        if(DOUG != 0x0){
            address bank = ContractProvider(DOUG).contracts("bank");
            if (msg.sender == bank ){
                uint oldBalance = balances[addr];
                if(oldBalance >= amount){
                    msg.sender.send(amount);
                    balances[addr] = oldBalance - amount;
                    return true;
                }
            }
        }
        return false;
    }

}

// The bank
contract Bank is FundManagerEnabled {

    // Attempt to withdraw the given 'amount' of Ether from the account.
    function deposit(address userAddr) returns (bool res) {
        if (!isFundManager()){
            return false;
        }
        address bankdb = ContractProvider(DOUG).contracts("bankdb");
        if ( bankdb == 0x0 ) {
            // If the user sent money, we should return it if we can't deposit.
            msg.sender.send(msg.value);
            return false;
        }

        // Use the interface to call on the bank contract. We pass msg.value along as well.
        bool success = BankDb(bankdb).deposit.value(msg.value)(userAddr);

        // If the transaction failed, return the Ether to the caller.
        if (!success) {
            msg.sender.send(msg.value);
        }
        return success;
    }

    // Attempt to withdraw the given 'amount' of Ether from the account.
    function withdraw(address userAddr, uint amount) returns (bool res) {
        if (!isFundManager()){
            return false;
        }
        address bankdb = ContractProvider(DOUG).contracts("bankdb");
        if ( bankdb == 0x0 ) {
            return false;
        }

        // Use the interface to call on the bank contract.
        bool success = BankDb(bankdb).withdraw(userAddr, amount);

        // If the transaction succeeded, pass the Ether on to the caller.
        if (success) {
            userAddr.send(amount);
        }
        return success;
    }

}

// The fund manager
contract FundManager is DougEnabled {

    // We still want an owner.
    address owner;

    // Constructor
    function FundManager(){
        owner = msg.sender;
    }

    // Attempt to withdraw the given 'amount' of Ether from the account.
    function deposit() returns (bool res) {
        if (msg.value == 0){
            return false;
        }
        address bank = ContractProvider(DOUG).contracts("bank");
        address perms = ContractProvider(DOUG).contracts("perms");
        if ( bank == 0x0 || perms == 0x0 || Permissions(perms).getPermission(msg.sender) < 1) {
            // If the user sent money, we should return it if we can't deposit.
            msg.sender.send(msg.value);
            return false;
        }

        // Use the interface to call on the bank contract. We pass msg.value along as well.
        bool success = Bank(bank).deposit.value(msg.value)(msg.sender);

        // If the transaction failed, return the Ether to the caller.
        if (!success) {
            msg.sender.send(msg.value);
        }
        return success;
    }

    // Attempt to withdraw the given 'amount' of Ether from the account.
    function withdraw(uint amount) returns (bool res) {
        if (amount == 0){
            return false;
        }
        address bank = ContractProvider(DOUG).contracts("bank");
        address perms = ContractProvider(DOUG).contracts("perms");
        if ( bank == 0x0 || perms == 0x0 || Permissions(perms).getPermission(msg.sender) < 1) {
            // If the user sent money, we should return it if we can't deposit.
            msg.sender.send(msg.value);
            return false;
        }

        // Use the interface to call on the bank contract.
        bool success = Bank(bank).withdraw(msg.sender, amount);

        // If the transaction succeeded, pass the Ether on to the caller.
        if (success) {
            msg.sender.send(amount);
        }
        return success;
    }

    // Set the permissions for a given address.
    function setPermission(address addr, uint8 permLvl) returns (bool res) {
        if (msg.sender != owner){
            return false;
        }
        address perms = ContractProvider(DOUG).contracts("perms");
        if ( perms == 0x0 ) {
            return false;
        }
        return Permissions(perms).setPermission(addr,permLvl);
    }

}

```

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

# Cond-Effects_And_Rejector

```Solidity

contract CrowdFunding {
    struct Backer {
        address addr;
        uint amount;
    }

    address public owner;
    uint public numBackers;
    uint public deadline;
    string public campaignStatus;
    bool ended;
    uint public goal;
    uint public amountRaised;
    mapping (uint => Backer) backers;

    event Deposit(address _from, uint _amount);
    event Refund(address _to, uint _amount);

    modifier onlyOwner()
    {
        if (msg.sender != owner) throw;
        _
    }

    function CrowdFunding(uint _deadline, uint _goal) {
        owner = msg.sender;
        deadline = _deadline;
        goal = _goal;
        campaignStatus = "Funding";
        numBackers = 0;
        amountRaised = 0;
        ended = false;
    }

    function fund() {
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

    function destroy() {
        if (msg.sender == owner) {
            suicide(owner);
        }
    }

    function () {
        // This function gets executed if a
        // transaction with invalid data is sent to
        // the contract or just ether without data.
        // We revert the send so that no-one
        // accidentally loses money when using the
        // contract.
        throw;
    }
}

```

# ERC20Token

```Solidity
contract ERC20Basic {
  uint256 public totalSupply;
  function balanceOf(address who) public view returns (uint256);
  function transfer(address to, uint256 value) public returns (bool);
  event Transfer(address indexed from, address indexed to, uint256 value);
}

```


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


# RunOncePerAccount

```Solidity
contract Registration {
    struct Participant {
        bool registered;  // if true, that person already registered
    }

    mapping(address => Participant) public participants;
    uint public numParticipants;

    modifier onlyOnce() {
        if (participants[msg.sender].registered) throw;
    }

    function Registration() {
        numParticipants = 0;
    }

    function register() onlyOnce{
        Participant p = participants[msg.sender];
        p.registered = true;
        numParticipants++;
    }
}

```

# StateMachine

```Solidity
pragma solidity ^0.4.11;

contract StateMachine {
    enum Stages {
        AcceptingBlindedBids,
        RevealBids,
        AnotherStage,
        AreWeDoneYet,
        Finished
    }

    // This is the current stage.
    Stages public stage = Stages.AcceptingBlindedBids;

    uint public creationTime = now;

    modifier atStage(Stages _stage) {
        require(stage == _stage);
        _;
    }

    function nextStage() internal {
        stage = Stages(uint(stage) + 1);
    }

    // Perform timed transitions. Be sure to mention
    // this modifier first, otherwise the guards
    // will not take the new stage into account.
    modifier timedTransitions() {
        if (stage == Stages.AcceptingBlindedBids &&
                    now >= creationTime + 10 days)
            nextStage();
        if (stage == Stages.RevealBids &&
                now >= creationTime + 12 days)
            nextStage();
        // The other stages transition by transaction
        _;
    }

    // Order of the modifiers matters here!
    function bid()
        payable
        timedTransitions
        atStage(Stages.AcceptingBlindedBids)
    {
        // We will not implement that here
    }

    function reveal()
        timedTransitions
        atStage(Stages.RevealBids)
    {
    }

    // This modifier goes to the next stage
    // after the function is done.
    modifier transitionNext()
    {
        _;
        nextStage();
    }

    function g()
        timedTransitions
        atStage(Stages.AnotherStage)
        transitionNext
    {
    }

    function h()
        timedTransitions
        atStage(Stages.AreWeDoneYet)
        transitionNext
    {
    }

    function i()
        timedTransitions
        atStage(Stages.Finished)
    {
    }
}

```
