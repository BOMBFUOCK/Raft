

Raft 算法 Golang实现
在实现算法之前，请多次阅读 https://raft.github.io/raft.pdf 
确保能够了解图2中各部分的功能与作用

![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/Figure2.png)

该项目为了简化说明，前提假设为有5个Server




![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/server.png)

1.Server:
        在Raft中生效的节点只有三中状态即，Leader,Candiate,Follower

        请重点思考究竟在投票时各个Server的状态，只有Candiate拥有投票的权利。

        请注意，每个人的选举都是独立进行的，因此你可以认为同一时间有多个选举同时进行。

        注意心跳的时间间隔是不固定的，因此每个Server的选举程序不是同时开始的。

        Follower:     
                复制Leader的Log并将已经committed的Log中的操作对自身kvstore进行执行。
        
        Candiate:
                获取选票同时也可以进行投票，一旦成为Leader立马发送心跳。
        Leader:
                周期性发送心跳，防止发生选举
                复制日志给所有Follower，最先将committed的Log中的操作对自身kvstore进行执行。
        

        
![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/request.png)

2.RequestVote RPC
        同一时间只能存在一位Leader,而根据选取机制，我们可以认为Leader是被Candiate选举得出的。

        明确RPC是并行的，因此在统计票数与状态改变时，要注意票数统计是否重复与状态改变是否重复。应加以判断。

        同时，非常重要的一点是当票数达到当前存活Server的一半时，Candiate将立即成为Leader,当一位Candiate成为Leader后，它将立马结束自身的选举周期，并向其余的所有Candiate发送通知，使其状态成为Follower并结束选举周期。

        在进行RequestVote时，Candiate将会发送投票请求至每一位候选人，注意在此时需要比较RequestVote RPC中Receiver implementation的条件，以防止重复投票。

        如果一轮选举后没有出现Leader，将会将Term+1并发起一轮新的选举。

        相关参数介绍：
                Arguments:
                        term                    Candidate’s term
                        candidateId             candidate requesting vote
                        lastLogIndex            index of candidate’s last log entry (§5.4)是每个候选人自身Log中最后一个Index
                        lastLogTerm             term of candidate’s last log entry (§5.4)是当前候选人的Term

                Results:
                        term currentTerm        for candidate to update itself 在发送请求时，候选人Term会与被要求投票的人的Term进行比对，因此改参数应该是较新的那一个
                        voteGranted             true means candidate received vote 表示该候选人是否要到票

        在实现过程中，如果Server全部存活，则有可能会出现一个候选人两个票，另一个候选人三张票，但是请放心，因为我们给予的心跳间隔是区间随机的，因此并不会出现想象中的冲突。

![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/state.png)

3.State
                currentTerm                     记录每个Server当前的Term
                votedFor                        记录该Server是否投票，思考在成为Follower后其应该是什么，如果不能正确的设置，会出现选举错误。
                log[]                           日志
                commitIndex                     全体Server超过半数成功复制的日志的Index,
                lastApplied                     已经执行操作的Log的最大Index值

                Leader独有的
                nextIndex[]                     记录了每个Server下一个要添加Log的Index,应在成为Leader后立马进行初始化
                matchIndex[]                    记录每个Server已经复制该Leader的Log的最大Index

![Alt text](https://github.com/BOMBFUOCK/Raft/blob/main/png/append.png)

4.AppendEntries RPC

                term                            leader’s term
                leaderId                        so follower can redirect clients
                                                （判断此时Follower的哪一个参数应该发生改变）


                prevLogIndex                    index of log entry immediately preceding new ones
                                                （该参数记录Leader应从哪里开始截取Log发送给Follower,思考与nextIndex的关系）

                prevLogTerm                     term of prevLogIndex entry

                entries[]                       log entries to store (empty for heartbeat may send more than one for efficiency);
                                                Leader截取自身Log发送至Follower进行复制）
                
                leaderCommit                    leader’s commitIndex 即半数以上matchIndex大于的值
                                                例如[3,3,1,5,2]，则为3

                Results:

                term                            currentTerm, for leader to update itself（注意条件中的设置）
                success                         true if follower contained entry matching prevLogIndex and prevLogTerm




5.心跳的作用：
            Follower：
                    如果在一次心跳期间没有收到来自Leader的心跳，将会立马成为Candiate并发起自身的一轮选举。

            Candiate:
                    如果在一个心跳间隔期间没收到来自Leader的心跳且没有成为Leader则发起一轮新的选举。

            Leader：
                    发送心跳告知Follower仍存在Leader。


6.Golang实现时需要注意的点

        注意通道的使用，即通道如果没有东西送进去，其会阻塞
                例如： 
                        value := <-rn.chan:
                中文：https://juejin.cn/post/6844904016254599176

                English:https://blog.logrocket.com/how-use-go-channels/

        注意锁的使用
                请参考https://thesquareplanet.com/blog/students-guide-to-raft/



7.相关课程推荐MIT6.824


