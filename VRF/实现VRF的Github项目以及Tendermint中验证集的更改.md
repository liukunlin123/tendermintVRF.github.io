[Google的开源项目](https://github.com/google/draft-irtf-cfrg-vrf)

[vrf的椭圆曲线密码学实现](https://github.com/vechain/go-ecvrf)

[Go语言简单的vrf实现](https://github.com/yoseplee/vrf)

[Algorand](https://github.com/algorand/go-algorand)



VRF函数

1. Keygen(VRF_GEN): generates key pair(secret key, public key)
2. Evaluate(VRF_EVAL): generates pseudorandom number and its proof
3. Verify(VRF_VER): verify the random number with proof



这些vrf实现的项目里，都没有把vrf和区块链结合在一起，只是简单的做了test，在[Go语言简单的vrf实现](https://github.com/yoseplee/vrf)中，就是所有节点都生成[0, 1]之间的随机数，然后设置个阈值，小于这个阈值的就相当于被选中。在test里面，做了1000次测试，然后进行了统计，但是这种方法在tendermint中不可行，因为这样的话感觉就相当于中心化了。

[vrf的椭圆曲线密码学实现](https://github.com/vechain/go-ecvrf)有个benchmark，但是这个测试只是验证了椭圆这个算法的可行性

Algorand的源码还没有仔细看，只是找到了vrf.go这个文件，里面写的是vrf生成随机数和零知识证明以及验证的一些函数。

暂时还没有找到能选出确定个数的算法。我感觉可能没有这个算法，因为如果要选出确定个数，那么一定是事先知道了全局的信息，而全局信息应该是不可能获知的。



```go
func (app *PersistentKVStoreApplication) EndBlock(req types.RequestEndBlock) types.ResponseEndBlock {
	return types.ResponseEndBlock{ValidatorUpdates: app.ValUpdates}
}

type ResponseEndBlock struct {
	ValidatorUpdates      []ValidatorUpdate       
	ConsensusParamUpdates *types1.ConsensusParams 
	Events                []Event                 
}

type ValidatorUpdate struct {
	PubKey crypto.PublicKey
	Power  int64
}

func (app *PersistentKVStoreApplication) updateValidator(v types.ValidatorUpdate) types.ResponseDeliverTx {
    。。。
    app.ValUpdates = append(app.ValUpdates, v)
}

func UpdateValidator(pk []byte, power int64, keyType string) ValidatorUpdate

InitChain, BeginBlock, execValidatorTx都调用了updateValidator
InitChain是进行初始化
BeginBlock在一定情况下会调用
execValidatorTx一定会调用

在执行完execValidatorTx后，
return app.updateValidator(types.UpdateValidator(pubkey, power, ""))
更新默克尔树 Merkle Tree
修改app.ValUpdates
然后执行EndBlock函数，返回新的验证者集合


```





