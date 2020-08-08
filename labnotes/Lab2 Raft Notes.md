## Lab2 Raft Notes
Author: Yufeng Yang

### Introduction
Implement Raft, a replicated state machine protocol.A set of Raft instances talk to each other with RPC to maintain replicated logs. Raft interface will support an indefinite sequence of numbered commands, also called log entries. The entries are numbered with index numbers. The log entry with a given index will eventually be committed. At that point, Raft should send the log entry to the larger service for it to execute.This lab exactly follow the design in the `extended Raft` paper, with particular attention to Figure 2. The implementation includes saving persistent state and reading it after a node fails and then restarts. 

<b> *** Lab codes are not allowed to put on github *** </b>

### Raft Design
In order to implement a replicated state machine protocol, there should be a `leader` monitoring the replication states of the whole systems. For example, if the application want to append a new command to the system, the `leader` would try to call `AppendEntries` RPC to the followers and make them append these command entries to their logs. However, the `leader` server may crash as well, so Raft system would have a leader election to elect a new leader. To sum up, there are three possible states and three main task in the whole Raft system.

<b>The three possible states are `leader`, `follower` and `candidate`. The three main tasks in Raft system are leader election, log replication and persistency.</b>


```
sequenceDiagram
Follower->>Candidate: ElectionTimeout
Candidate->>Candidate: ElectionTimeout
Candidate->>Leader: Receive votes from majority
Candidate->>Follower: Discover new leader or term
Leader->>Follower: Discover higher term
```

#### 1. Leader Election
Leaders send periodic heartbeats (AppendEntries RPCs that carry no log entries) to all followers in order to maintain their authority. If a follower receives no communication over a period of time called the election timeout, then it assumes there is no viable leader and begins an election to choose a new leader.

A candidate wins an election if it receives votes from a majority of the servers in the full cluster for the same term. Each server will vote for at most one candidate in a given term, on a first-come-first-served basis.

While waiting for votes, a candidate may receive an AppendEntries RPC from another server claiming to be leader. If the leader’s term (included in its RPC) is at least as large as the candidate’s current term, then the candidate recognizes the leader as legitimate and returns to follower state. If the term in the RPC is smaller than the candidate’s current term, then the candidate rejects the RPC and continues in candidate state.

Raft uses randomized election timeouts to ensure that split votes are rare and that they are resolved quickly. To prevent split votes in the first place, election timeouts are chosen randomly from a fixed interval (e.g., 150–300ms).Each candidate restarts its randomized election timeout at the start of an election, and it waits for that timeout to elapse before starting the next election.

#### 2. Log Replication
Once a leader has been elected, it begins servicing client requests. Each client request contains a command to be executed by the replicated state machines. The leader appends the command to its log as a new entry, then issues AppendEntries RPCs in parallel to each of the other servers to replicate the entry. When the entry has been safely replicated (as described below), the leader applies the entry to its state machine and returns the result of that execution to the client.

The leader decides when it is safe to apply a log en- try to the state machines; such an entry is called commit- ted. Raft guarantees that committed entries are durable and will eventually be executed by all of the available state machines. A log entry is committed once the leader that created the entry has replicated it on a majority of the servers.

When sending an AppendEntries RPC, the leader includes the index and term of the entry in its log that immediately precedes the new entries. If the follower does not find an entry in its log with the same index and term, then it refuses the new entries.

In Raft, the leader handles inconsistencies by forcing the followers’ logs to duplicate its own. This means that conflicting entries in follower logs will be overwritten with entries from the leader’s log.

If desired, the protocol can be optimized to reduce the number of rejected AppendEntries RPCs.

#### 3. Persistency
The system will save the persistent variables of all server at a proper time. If a server crash, it will restart with checking the data on the disk and loading it into the memory.

Actually, as long as the server has changed its commit index, it will start to serialize the persistent variables and save the bytes into the disk.

### Project Design
The whole project is split into three parts:
- `raft.go`: responsible for the functions of followers and candidates
- `raft_leader.go`: responsible for the functions of the sole leader
- `raft_apply.go`: updating commit index and applying the commands to the application

#### 1. Follower and Candidate
```go
// In raft.go file

// Raft Struct
type Raft struct {
	mu        sync.Mutex          // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]
	dead      int32               // set by Kill()

	state int                     // Follower || Candidate || Leader
	applyCh chan ApplyMsg         // the channel to send messages of committing commands
	currentTerm int
	votedFor int                  // whom I vote for in this term
	log []Entry                   // the log in memory

	commitIndex int               // commits all entries before commitIndex of logs
	lastApplied int               // the last applied index of entry (apply commands into the application)
	commitCond *sync.Cond         // the condition variable on changing commitIndex

	nextIndex []int               // index of the next log entry to send to that server
	// matchIndex []int

	ElectionTimeoutBase int       // the lower limit of election time
	ElectionTimeoutRange int      // the range of random election time
	HeartBeatTimeout int          // timeout of heartbeat sent by the leader

	timer *time.Timer             // timer for monitoring the election timeout
}

// Create a Raft server and do initialization
func Make(peers []*labrpc.ClientEnd, me int, persister *Persister, applyCh chan ApplyMsg) *Raft {}


// monitoring the election timeout (and call LeaderElection function)
func (rf *Raft) TimerFunc() {}

// Launch a leader election.
func (rf *Raft) LeaderElection() {
    // ...
    rf.state = CANDIDATE
	rf.currentTerm ++
	rf.votedFor = rf.me
	// ...
	
	// concurrently call `RequestVote` RPC to each server (lauch many goroutines)
	// count the votes and see if itself has won the elction
}

// RequestVote RPC arguments structure.
type RequestVoteArgs struct {
	Term int
	CandidateId int
	LastLogIndex int
	LastLogTerm int
}

// RequestVote RPC reply structure.
type RequestVoteReply struct {
	Term int
	VoteGranted bool
}

// RequestVote RPC handler
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
    // Reply false if term < currentTerm
    // If votedFor is null or candidateId, and candidate’s log is at least as up-to-date as receiver’s log, grant vote.
    // "up-to-date" means my last entry's term is higher than args.LastLogTerm. (If both equal, check the last index.)
}

type AppendEntriesArgs struct {
	Term int               // rf.currentTerm
	LeaderId int           // rf.me
	PrevLogIndex int       // rf.nextIndex[server] - 1
	PrevLogTerm int        // rf.log[rf.nextIndex[server] - 1].Term
	Entries []Entry        // rf.log[rf.nextIndex[server]:]
	LeaderCommit int       // rf.commitIndex
}

type AppendEntriesReply struct {
	Term int
	Success bool
}

// AppendEntries RPC handler
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
    // Reply false if term < currentTerm
    // Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm.
    // If an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it
    // Append any new entries not already in the log
    // If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)
}

// save Raft's persistent state to stable storage,
// where it can later be retrieved after a crash and restart.
// see paper's Figure 2 for a description of what should be persistent.
func (rf *Raft) persist() {
	rf.mu.Lock()
	// serialize the persistent variables (rf.currentTerm, rf.votedFor, rf.log)
	rf.persister.SaveRaftState(data)
	rf.mu.Unlock()
}


// restore previously persisted state.
func (rf *Raft) readPersist(data []byte) {
	if data == nil || len(data) < 1 { // bootstrap without any state?
		return
	}
	rf.mu.Lock()
	// deserialize the data from the disks.
	rf.mu.Unlock()
}


```

#### 2. Leader
```go
// in raft_leader.go file

// If the follower is elected as a new leader, it will start a new goroutine called `LeaderJob`
func (rf *Raft) LeaderJob() {
    rf.LeaderInitialization()
	isHeartBeat := true           
	for rf.IsLeader() {
	    // As long as the leader exists, it will call AppendEntries RPC to
	    // some followers or all followers. Not all servers do not match the 
	    // logs with the leader, but heat beat should be sent periodicaly.
		rf.CallAppendEntries(isHeartBeat)
		isHeartBeat = !isHeartBeat
	}
}

// do the initialization of the leader (mainly the rf.nextIndex of the leader)
func (rf *Raft) LeaderInitialization() {
	// ...
	rf.nextIndex = make([]int, len(rf.peers))
	for i := 0; i < len(rf.peers); i ++ {
		rf.nextIndex[i] = len(rf.log)
	}
	// ...
}

// The leader call AppendEntries RPC.
func (rf *Raft) CallAppendEntries(isHeartBeat bool) int {
    // ...
    // for each server except me:
    //     create the RPC args and reply struct
    //     go func(server int) {concurrently call AppendEntries and check reply}
    // update commitIndex and broadcast the rf.commitCond to apply commands in the application
    // ...
}

```

#### 3. Update Commit and Apply (Job of all servers)
```go
// in the `raft_apply.go` file

func (rf *Raft) UpdateCommit() {
    // According to the new commitIndex to apply commands
}
```

### Problems

#### 1. No leader can be elected
- Check the condition of being a leader
- Stop the timer when start to be a leader
- Less than N + 1 members in a group of 2N + 1 servers


#### 2. Having a leader but followers still lauch a few elections
- Seems having a leader but for some reasons it stop sending heart beats (checking conditions of leader's loop)
- timer reseting is not being reached in followers' jobs


#### 3. Logs do not match
- Check conditions of AppendEntries RPC
- Logic is ok but rf.nextIndex[server]-- so slow (considering optimization)
- The follower's uncommited logs should be deleted if they don't match the leaders'.
- Heat beat timeout can be decreased (fast heart beat)
- Every time the servers restart, it should check and load the local logs on the disk