传入参数：peer
函数的作用：为该peer创建几个go Routine，
go conR.gossipDataRoutine(peer, peerState)
go conR.gossipVotesRoutine(peer, peerState)
go conR.queryMaj23Routine(peer, peerState)
负责消息的传递和状态的同步
输出为空
被调用：TestPEXReactorAddRemovePeer


P2P还有一个非常重要的功能就是进行节点发现——PEX，PEX也是一个 Reactor 的具体实现
作为种子模式的pex就是每隔一段时间就会检查当前peer集合中是否有连接时间超过28小时的peer， 如果有，除非设置了持久连接否则就断开， 然后去爬去新的节点， 如果从地址簿中爬去到新的节点就发送一份请求地址交换的报文过去。
作为非种子节点就是一直进行连接，尽量保存足够的连接数量。 如果数量不够就先从地址簿从取一些地址进行连接。如果地址簿发现保存的地址数量太少就会尝试向已知的peer发送请求新peer的报文请求。如果这个时候还是没有足够的peer就只能向种子节点去请求。

快速同步（FastSync）：在当前节点落后于区块链的最新状态时，需要进行节点间同步，快速同步只下载区块并检查验证人的默克尔树，比运行实时一致性八卦协议快得多。一旦追上其它节点的状态，守护进程将切换出快速同步并进入正常共识模式。在运行一段时间后，如果此节点至少有一个 peer，并且其区块高度至少与最大的报告的 peer 高度一样高，则认为该节点已追上区块链最新状态（即 caught up）。
node(s)：网络中任意多个节点个体或节点集合；
peer(s)：由nodes相互连接组成的对等节点；
validator(s)：拜占庭容错共识协议的参与者；

MempoolReactor 用来在 peer 之间对 mempool 交易进行广播。