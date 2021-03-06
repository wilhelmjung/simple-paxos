# Consensus algorithm:
## Phase 1:
 A. A proposer selects a proposal number n and sends a
   prepare request with number n to a majority of
   acceptors

B. If an acceptor receives a prepare request with number
   n greater than that of any prepare request to which it
   has already responded, then it responds to the request
   with a promise not to accept any more proposals
   numbered less than n and with the highest-numbered
   proposal (if any) that it has accepted.

## Phase 2:
A. If the proposer receives a response to its prepare
   request (numbered n) from a majority of acceptors,
   then it sends an accept request to each of those
   acceptors from a proposal numbered n with a value v,
   where v is the value of the highest-numbered proposal
   among the responses, or is any value if the responses
   reported no proposals.

B. If an acceptor receives an accept request for a
   proposal numbered n, it accepts the proposal unless it
   has already responded to a prepare request having a
   number greater than n.

## Message flows:

```
Message flow: Multi-Paxos Collapsed Roles, start
(first instance with new leader)

Client      Servers
   |         |  |  | --- First Request ---
   X-------->|  |  |  Request
   |         X->|->|  Prepare(N)
   |         |<-X--X  Promise(N,I,{Va,Vb})
   |         X->|->|  Accept!(N,I,Vn)
   |         |<-X--X  Accepted(N,I)
   |<--------X  |  |  Response
   |         |  |  |

Message flow: Multi-Paxos Collapsed Roles, steady state
(subsequent instances with same leader)

Client      Servers
   X-------->|  |  |  Request
   |         X->|->|  Accept!(N,I+1,W)
   |         |<-X--X  Accepted(N,I+1)
   |<--------X  |  |  Response
   |         |  |  |


Message types:

client_value

client_request:
    instance_id uint32
    value client_value

// phase 1a:
pxs_req_prepare:
    instance_id uint32
    proposal_num uint32 // n

// phase 1b: rsp with a promise or already accepted proposal.
// acceptor_state: 
//   promised_proposal_num *uint32
//   value *client_value
// if promised_proposal_num = null then { promised_proposal_num = n; ... }
pxs_rsp_promise:
    instance_id uint32
    proposal_num uint32 // n
 
// if promised_proposal_num > n then:
// this is an optimization
pxs_rsp_prepare_nack:
    instance_id uint32
    proposal_num uint32 // higher-numbered proposal.

// if n > promised_proposal_num then:
// respond with proposal or null.
pxs_rsp_prepare_ack:
    instance_id uint32
    proposal_num uint32 // highest number (< n) proposal.
    value client_value

//phase2a
pxs_req_accept:
    instance_id uint32
    proposal_num uint32
    value client_value // value of highest-numbered proposal among responses.

//phase2b
pxs_rsp_accepted: // to proposer
    instance_id uint32
    proposal_num uint32
    value client_value // from proposer

//phase2c
pxs_noti_chosen: // to learners
    instance_id uint32
    proposal_num uint32
    value client_value // from proposer

```

## Q: How to elect master/leader proposer?
## A: Using timeout:
   1. Every proposer has an election timeout(T1), once timeout it should propose itself as leader.
   2. The leader proposer should have a shorter timeout(T0 < T1).
   3. Within the term(T0 ~ T1), acceptors should not accept any proposal from proposer other than the leader proposer.
   4. The chosen of LeaderID should be carried out by basic Paxos algorithm(p1 and p2 phase).

## Q: Should proposer propose multiple instances in parallel?
## A: Leader proposer can do this, because the following instance_id is unused by any proposer.
   For leader proposer it is safe to use these instance_id.

## About instance_id:
```
1. Leader proposer should remember the highest instance_id it has ever proposed.
2. Other proposers should update the highest instance_id that leader has proposed.
3. The instance_id should be persisted on stable storage.
4. instance_id should be updated once proposal is chosen. ???

Node/Cluster states:
{key="LeaderID", value=?}
//leader proposer and other proposer election timeout
electionTimeout0, electionTimeout1 int
```
