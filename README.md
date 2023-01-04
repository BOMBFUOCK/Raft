Raft Algorithm Golang Implementation

[中文版](https://github.com/BOMBFUOCK/Raft/blob/main/READMEchinese.md)


Before implementing the algorithm, please read https://raft.github.io/raft.pdf several times 
to ensure that you understand the function and role of each part in Figure 2

![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/Figure2.png)

This project assumes that there are 5 Servers for the sake of simplicity




![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/server.png)
1. Server:
        There are only three states of nodes in effect in Raft, i.e. Leader, Candiate, Follower.

        Please focus on the exact state of each Server at the time of voting, only Candiate has the right to vote.

        Note that each individual election takes place independently, so you can assume that there are multiple elections taking place at the same time.

        Note that the time interval between heartbeats is variable, so the election process does not start at the same time for each Server.

        Follower:     
                Copies the Leader's Log and executes the actions in the already committed Log against its own kvstore.
        
        Candiate:
                Gets votes and can also vote, and sends a heartbeat as soon as it becomes a Leader.
        Leader:
                Sends heartbeats periodically to prevent elections from happening
                Copy logs to all Follower, first to execute the actions in the committed logs to its own kvstore.
        

        
![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/request.png)
2. RequestVote RPC
        There can only be one Leader at a time, and according to the selection mechanism, we can assume that the Leader is elected by Candiate.

        It is clear that the RPC is parallel, so when counting votes and state changes, it is important to note whether the vote count is duplicated and whether the state change is duplicated. This should be judged.

        It is also important to note that when the number of votes reaches half of the currently active servers, the Candiate will immediately become the Leader, and when a Candiate becomes the Leader, it will immediately end its own election cycle and send a notification to all remaining Candiates to make its state Follower and end the election cycle.

        During the RequestVote, the Candiate will send a poll request to each candidate, noting that at this point it is necessary to compare the conditions of the Receiver implementation in the RequestVote RPC in order to prevent duplicate votes.

        If there is no Leader after one round, a new round of elections will be initiated with Term+1.

        The relevant parameters are described in
                Arguments:
                        term                            Candidate's term
                        candidateId                     Candidate requesting vote
                        lastLogIndex                    index of candidate's last log entry (§5.4) is the last index in each candidate's own log
                        lastLogTerm                     term of candidate's last log entry (§5.4) is the Term of the current candidate

                Results:
                        term                            currentTerm for candidate to update itself When the request is sent, the candidate's Term is compared with the Term of the person who was asked to vote, so the parameter should be the newer one

                        voteGranted                     true means candidate received vote means whether the candidate received a vote

        In the implementation, if all the Servers survive, it is possible that one candidate will have two votes and the other three, but rest assured that since the heartbeat interval we give is interval random, there will be no imaginary conflict.

![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/state.png)
3. State

                currentTerm                             records the current Term of each Server
                votedFor                                records whether or not the Server voted, thinking about what it should be when it becomes Follower, and if it is not set correctly, an election error will occur.
                log[]                                   logs
                commitIndex                             The index of the logs for which more than half of all Servers have been successfully replicated,

                lastApplied                             Maximum Index of logs that have performed an operation

                Unique to Leader

                nextIndex[]                             records the Index of the next log to be added to each Server, which should be initialized as soon as it becomes a Leader

                matchIndex[]                            records the maximum Index of each Server that has copied the Leader's Logs

![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/append.png)

4. AppendEntries RPC

                term leader's term
                leaderId so follower can redirect clients
                                                (determine which of the Follower's parameters should change at this point)


                prevLogIndex index of log entry immediately preceding new ones
                                                (this parameter records where the Leader should start intercepting logs to send to the Follower, thinking about the relationship to nextIndex)

                prevLogTerm term of prevLogIndex entry

                entries[] log entries to store (empty for heartbeat may send more than one for efficiency);
                                                (Leader intercepts its own Log and sends it to Follower for replication)
                
                leaderCommit leader's commitIndex i.e. the value of half or more matchIndexes greater than
                                                For example, if [3,3,1,5,2], then 3

                Results:

                term currentTerm, for leader to update itself (note the setting in the condition)
                success true if follower contained entry matching prevLogIndex and prevLogTerm




5. Role of heartbeat.
            Follower.
                    If no heartbeat is received from the Leader during a heartbeat, it will immediately become a Candiate and initiate its own round of elections.

            Candiate:
                    If no heartbeat is received from the Leader during a heartbeat interval and it does not become the Leader then a new election is initiated.

            Leader.
                    Sends a heartbeat to inform the Follower that a Leader still exists.


6. Points to note when implementing Golang

        Pay attention to the use of channels, i.e. a channel will block if nothing is fed into it
                For example 
                        value := <-rn.chan:
                Chinese:https://juejin.cn/post/6844904016254599176

                English:https://blog.logrocket.com/how-use-go-channels/

        Note the use of locks
                Please refer to https://thesquareplanet.com/blog/students-guide-to-raft/



7. Related courses recommended MIT6.824


 Translated with www.DeepL.com/Translator (free version)