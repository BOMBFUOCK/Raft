Raft 算法 Golang实现
在实现算法之前，请多次阅读 https://raft.github.io/raft.pdf 
确保能够了解图2中各部分的功能与作用

1.Server:
        在Raft中生效的节点只有三中状态即，Leader,Candiate,Follower

        请重点思考究竟在投票时各个Server的状态，只有Candiate拥有投票的权利。

        请注意，每个人的选举都是独立进行的，因此你可以认为同一时间有多个选举同时进行。

        注意心跳的时间间隔是不固定的，因此每个Server的选举程序不是同时开始的。
        

2.RequestVote RPC
        同一时间只能存在一位Leader,而根据选取机制，我们可以认为Leader是被Candiate选举得出的。

        明确RPC是并行的，因此在统计票数与状态改变时，要注意票数统计是否重复与状态改变是否重复。应加以判断。

        同时，非常重要的一点是当票数达到当前存活Server的一半时，Candiate将立即成为Leader,当一位Candiate成为Leader后，它将立马结束自身的选举周期，并向其余的所有Candiate发送通知，使其状态成为Follower并结束选举周期。

        在进行RequestVote时，Candiate将会发送投票请求至每一位候选人，注意在此时需要比较RequestVote RPC中Receiver implementation的条件，以防止重复投票。

        如果一轮选举后没有出现Leader，将会将Term+1并发起一轮新的选举。

3.心跳的作用：
            Follower：
                    如果在一次心跳期间没有收到来自Leader的心跳，将会立马成为Candiate并发起自身的一轮选举。

            Leader：
                    发送心跳告知Follower仍存在Leader。
            


