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
