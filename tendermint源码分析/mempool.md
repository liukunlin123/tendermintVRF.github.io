共识引擎调用的两个方法Reap和Update. 先入为主想一想共识引擎和内存池的关系，应该是从内存池取出交易-->执行交易--->打包交易--->告诉内存池应该移除的交易。 所以Reap作用应该就是从内存池中取出交易。
块的request
块高度
request和peer是关联的
tendermint会同时进行600个区块的下载
也就是说BlockPool启动之后 会一直循环 然后看看是不是有600个routine在进行块请求
// 如果有了 就尝试移除那些被标记为超时的peer，如果没有超过600个routine则继续创建一个请求。
一个请求对应一个块高度
PeekTwoBlocks从pool.height和pool.height+1对应的request取出块内容
RedoRequest 撤销某个块高度对应请求的结果 如果这个request已经绑定了某个peer， 通知绑定这个peer下的所有request均进行撤销请求，然后将这个peer从容器中删除。 这个函数是因为区块交易没通过才会被调用的。 后面会分析到。 request的撤销是通过这个通道置位来标识。后面分析request的routine来解释它是怎么和这个通道进行联系的。

添加一个区块到对应的高度的request。同时将对应的peer超时时间置位。当对应高度的request被添加一个块内容, 说明这个请求的块已经拿到, 这个时候将pool.numPending-1 表示这个请求已经不用挂起了, 同时置位这个通道表示request已经接受到块内容。

SetPeerHeight(peerID p2p.ID, height int64) 其实是更新某个peer对应的最高的区块高度。 这个函数的调用应该也是在Rector的Receive中被调用。 设想一下场景， 本节点向连接的所有peer发送了一个块高度请求， 然后有一些peer回应了自己当前所属的最高块高度。这个时候调用这个函数。

pickIncrAvailablePeer 这个函数就是给一个request找一个合适的peer进行绑定。 同时增加这个peer的numpending值(相当于是引用值)。这个引用值啥用呢，当引用值从0到1 则启动定时器。 当引用值每次减少一个(未减少到0)这个重置定时器。 这个定时器的作用就是为了体现peer是否超时。 也即是表示对于peer的一次块请求是否超时了， 如果超时了我们就在前面的makeRequestersRoutine函数中看到了就是把这个peer给移除掉（removeTimedoutPeers）。

就是把{  BlockRequest{height, peerID} 通过requestsCh这个通道发给Reactor 告诉Reactor 找个peer去要块内容  }

一个是向\<errorsCh\>通道发送了 peerError{err, peerID} 对应的错误信息 由Reactor的来读取 Reactor读到这个内容然后告诉P2P的Switch删除这个peer
// 另一个是pool的routine根据didTimeout将其从容器中移除。

每隔10S 向所有的已知peer发送一次 区块高度状态的请求 如果有peer回复了
			// 自己的当前区块高度 就会把高度和对应的peer加入BlockPool的peer容器中 

Reactor的主任务应该就是读取区块请求, 向指定的peer发送区块下载, 查询下一个区块是否已经下载，如果已经下载则处理完后进行校验。 如果校验成功则保存到数据库中，同时提交给state组件进行区块重复

State的成员变量应用的直接是一个值拷贝，意思很明确，成员函数都不会修改State成员属性。列一下主要函数功能: Copy 拷贝一个新的State数据返回 GetValidators 获取当前验证者集合和上一个区块的验证者集合 看一下根据State内容创建区块MakeBlock这个函数。

总结一下ApplyBlock的功能:

根据当前状态和区块内容来验证当前区块是否符合要求
提交区块内容到ABCI的应用层, 返回应用层的回应
根据当前区块的信息，ABCI回应的内容，生成下一个State
再次调用ABCI的Commit返回当前APPHASH
持久化此处状态同时返回下一次状态的内容

注意这个地方 我们是根据此次提交的区块信息 来验证上一个块的内容  
		// 迭代上一个区块保存的所有验证者 确保每一个验证者签名正确 
		// 最后确认所有的有效的区块验证者的投票数要大于整个票数的2/3

注意这个是执行ABCI后的回调 之前在Mempool也分析过回调
	// 这里不在进行追踪 直接说结论 也就是在提交每一个交易给ABCI之后 然后在调用此函数 
	// 这个回调只是统计了哪些交易在应用层被任务是无效的交易
	// 从这里我们也可以看出来 应用层无论决定提交的交易是否有效 tendermint都会将其打包到区块链中

到了这个地方， State的主要功能就算分析完了，State其实就是代表本节点执行的区块链的最新状态，它同时也是下一个区块执行的验证基础。 这个组件里非常重要的函数就是ApplyBlock了, 基于tendermint来创建自己的区块链， 我们要实现的几个重要接口函数中， 如果从数量上来说这里调用的最多， 从开始一个新区块，到提交交易，再到接收这个区块，最终确认区块。可以说重要的步骤都是在这里完成的。也就是说当tendermint的一个区块被生成之后，此函数是必须被调用的。 因为之后这个函数被调用之后，APP层才算完成了打包区块的。 想象一下一个新区块被生成也就两个地方， 如果他不是验证者，那么它只能是一个同步者，也即是只能下载区块。 这个我们已经见到过就是在Blockchain中下载新区块的地方， 如果他是一个验证者， 那么应该在共识模块被调用。 后面我们去分析共识算法的地方在执行去看看。
State中的区块状态，发送给应用层交易做什么，判断交易有效无效？，那共识节点在后面确认区块有什么作用。
**状态机**？？
所有的节点都存在reactor，负责消息的处理和节点的通信

SwitchToConsensus每个节点在启动之前
Reactor结构里储存了验证节点的个数，通过P2P来识别节点，并添加到Reactor结构里（reactor.go里的addPeer）
handleTimeout处理超时情况(state.go)
进入新的轮次时，判断以下条件
1. 判断是否需要等待交易且为第一轮，如果是，则等待内存池的交易
2. app hash值是否改变
如果不是，则立刻进入提议阶段
```
waitForTxs := cs.config.WaitForTxs() && round == 0 && !cs.needProofBlock(height)
	if waitForTxs {
		if cs.config.CreateEmptyBlocksInterval > 0 {
			cs.scheduleTimeout(cs.config.CreateEmptyBlocksInterval, height, round,
				cstypes.RoundStepNewRound)
		}
	} else {
		cs.enterPropose(height, round)
	}
```
```
// WaitForTxs returns true if the consensus should wait for transactions before entering the propose step
func (cfg *ConsensusConfig) WaitForTxs() bool {
	return !cfg.CreateEmptyBlocks || cfg.CreateEmptyBlocksInterval > 0
}
```
```
// needProofBlock returns true on the first height (so the genesis app hash is signed right away)
// and where the last block (height-1) caused the app hash to change
func (cs *State) needProofBlock(height int64) bool {
	if height == cs.state.InitialHeight {
		return true
	}

	lastBlockMeta := cs.blockStore.LoadBlockMeta(height - 1)
	if lastBlockMeta == nil {
		panic(fmt.Sprintf("needProofBlock: last block meta for height %d not found", height-1))
	}
	return !bytes.Equal(cs.state.AppHash, lastBlockMeta.Header.AppHash)
}
```
