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
