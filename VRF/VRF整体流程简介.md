<center><h3>VRF整体流程简介</h3></center>
##### 1.所有节点生成一对密钥（公钥PK，私钥SK）

- 在网上没有找到具体的生成方法，我谈一下自己的看法：生成密钥的这个函数需要的参数应该是可以唯一标识节点的某个节点属性，如节点ID。这个节点ID类似于在本机生成SSH密钥时，我们要输入的邮箱地址。

##### 2.所有节点调用$VRF_{Hash}$和$VRF_{Proof}$分别生成随机数输出和零知识证明

- 随机数输出 $result = VRF_{Hash}(SK, info)$
- 零知识证明 $proof = VRF_{Proof}(SK,info)$
- 这里的info我还是不太清楚到底是什么，应该也是节点的某个属性值。

##### 3.根据每个节点生成的随机数输出，按照某种规则选择委员会成员

##### 4.被选中的节点将自己的 value 和 proof 递交给验证者

##### 5.验证者调用$VRF_{P2H}$进行验证

- 验证者计算$result = VRF_{P2H}(proof)$是否成立，如果成立，继续下面的步骤，否则终止。

##### 6.被选中的节点将自己的 PK 和 info 递交给验证者

##### 7.验证者调用$VRF_{Verify}$进行验证

- 验证者计算$VRF_{Verify}(PK,info,proof)$，True表示验证通过，False表示验证未通过。



<center><h3>以Algorand选举共识委员会成员为例介绍VRF的部分流程</h3></center>
SK: 用户私钥，seed: 伪随机种子，t: 期望被选为该role的用户的数量，role: 角色，w: 用户权重，W: 总权重

1.每个用户首先使用VRF函数计算$<hash, \pi>:=VRF_{SK}(seed||role)$ ，hash值为随机数输出，随机范围为$[0, 2^{hashlen}-1]$，$\pi$为零知识证明。

2.为了能够使用户被选中的概率和其所拥有的货币相对应，Algorand将用户User按照其拥有多少单位的货币分割成sub-users，也就是：假设用户i拥有$w_i$的货币, 那么该用户就拥有$w_i$个sub-users。每个单位货币被选中的概率为$p = \frac{t}{W}$。（POS的思想）

3.用户的w个sub-users中被选中k个的概率遵循二项分布：$B(k;w,p) = \tbinom{k}{w}p^k(1-p)^{w-k}$，其中$\sum_{k=0}^{w}B(k;w,p) = 1$。

4.为了确定用户w个sub-users中多少个sub-user被选中，抽签算法将[0,1) 区间分割成连续的区间：$I^j = [\sum_{k=0}^{j}B(k;w,p), \sum_{k=0}^{j+1}B(k;w,p))$。

5.如果$hash/2^{hashlen}$落在区间$I^{j}$那么该用户共有j个sub-users被选中，该数字j能够通过$\pi$被其他用户验证, j >0 就证明本节点在本轮次中被选中为共识委员会成员。

![Alt text](E:\大创\报告文档\Algorand中的VRF.png)

![Alt text](E:\大创\报告文档\Algorand中的VRF2.png)