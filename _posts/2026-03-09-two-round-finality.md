---
layout: post
title: BFT with 2-Rround Finality
author: Suyan Qu
categories:
- Byzantine Fault Tolerance
excerpt_separator: <!--more-->
---

This post gives a general overview of BFT protocols with 2-round finality. I was digging into this topic last weekend since there is not a good post explaining why they need $n\ge5f+1$ and ChatGPT cannot explain it to me (so AI cannot replace us...yet). I hope this post can clear out some common questions on this topic so that people don't need to go through all the thinkings I did :). This post is not perfect (since I just learned about the topic as well), so if you have any question or suggestions, feel free to submit PRs or issues. In this blog, we focus on protocols with all-to-all communication, but the same argument should apply to protocols with one-to-all communication. 

## What is 2-Round Finality
Traditional BFT consensus protocols (e.g., PBFT, Tendermint, ...) require at least 3 rounds of communication to commit a block: 
1. The leader broadcasts a proposal. 
2. The followers cast votes. 
3. The followers inform each other that a particular proposal received enough votes to be safely committed. 

This contradicts with CFT consensus protocols (e.g., Raft), where proposals can be committed in 2 rounds of communication. The extra round exists mainly because, unlike in CFT, a Byzantine leader can send different proposals to different followers. This means a follower cannot blindly commit what the leader proposes: they need to validate with each other to ensure that they all commit the same block. 

Because 3-round finality incurs an extra message delay to commit a block, an immediate question is if 2-round finality is possible in BFT so that we experience the same latency as in CFT. The answer is yes, but at a stronger assumption of $n\ge5f+1$.

## How Does Consensus with 2-Round Finality Work
BFT consensus with 2-round finality generally work as follows:
1. The leader broadcasts a proposal. 
2. Each follower, upon first receiving the first proposal of the current view, casts a vote. 

If a validator observes a quorum ($Q_c$) of votes for a proposal, the proposal is considered committed and the validator will advance to the next view; Otherwise, it will send a timeout message that includes the most recent vote it casts. Upon receiving a quorum ($Q_t$) of timeout messages, it will also advance to the next round. To ensure safety, the leader of the new round needs to propose the same block if it observes a timeout quorum from the previous round and sufficiently many ($\ge I$) timeout messages in the quorum voted for the same block from the previous round. Similarly, the followers can only vote for this block in the new round. The basic intuition is that, if some validator commits in the previous round, the leader needs to propose the same block so that other validators will also commit on that block. Notice: it is possible that no correct node commits on this block, but it does not violate safety:
* Safety: if a correct validator commits on a block at a height, all correct validators commit on the same block at the same height. 

One special case is that, in a single view, two correct validators may observe different timeout quorums, each with $I$ votes for different proposals. This could happen if, in the previous round, a malicious leader proposes two different blocks, equivocating correct validators. For liveness, we cannot force these correct validators to vote for different blocks since they will never be able to form a commit quorum. Therefore, when proposing a new block, the leader will also include the commit quorum or the timeout quorum from the previous view. With this design, suppose the leader observes a timeout quorum with $I$ votes for block $A$ and a follower observes a timeout quorum with $I$ votes for block $B$, the leader will propose $A$ again with $I$ votes from previous round for $A$. The follower, upon receiving the proposal, will notice that there exists both $I$ votes for $A$ and $I$ votes for $B$. This information should be able to prove that no correct validator can commit on $A$ or $B$ in the previous round, so the follower is free to vote for either. 

## Why $n\ge5f+1$
Unlike traditional BFT consensus that requires $n\ge3f+1$, consensus with 2-round finality has a stronger assumption of $n\ge5f+1$. To arrive at this bound, we first derive some requirements that $Q_c$, $Q_t$, and $I$ should satisfy. 

First of all, for basic liveness, we should have
$$n-f\ge Q_c$$
$$n-f\ge Q_t$$
Second, if at least $I$ votes for the same block $A$ exists in a timeout quorum, it is possible that a correct validator commits on $A$. To get the threshold value $I$, we consider the extreme case: all nodes not in the timeout quorum are correct and voted for $A$, all $I$ nodes in the timeout quorum voted for node $A$ are correct, and $f$ Byzantine nodes voted for $A$, and they together forms a commit quorum:
$$(n-Q_t)+I+f=Q_c$$
$$I=Q_c+Q_t-n-f$$
Lastly, if a validator observes $I$ votes for block $A$ and $I$ votes for block $B$, it will know that no commit quorum can be formed in the view. Note that this is equivalent to saying that $I$ votes for block $A$ guarantees that no quorum can be formed for blocks other than $A$. Therefore,
$$Q_c+I> n+f$$
Plug in $I$
$$2Q_c+Q_t-n-f> n+f$$
$$2Q_c+Q_t>2n+2f$$
Applying the bounds on $Q_c$ and $Q_t$
$$3(n-f)\ge 2Q_c+Q_t>2n+2f$$
$$3n-3f>2n+2f$$
$$n>5f$$
Note that this bound is achieved when $Q_c=Q_t=n-f$. In this case, $I=n-3f$. 

## Protocols with 2-Round Finality
* [FaB Paxos](https://www.cs.cornell.edu/lorenzo/papers/fab.pdf): proposed in 2006, this protocol extends Paxos for Byzantine tolerance while preserving 2-round finality. It allows multiple proposers, making the problem more complicated than ever. I will probably go through the paper and fill in this part when I have more time :p
* [ChonkyBFT](https://arxiv.org/abs/2503.15380): basically the protocol described in this post but with one-to-all communication: in the first round of communication, the leader proposes a block and validators reply with vote; in the second round, the leader broadcasts the votes it receives and the validators reply with a commit message and advance to the next round. 
* [Kudzu](https://arxiv.org/abs/2505.08771): Kudzu combines the optimistic 2-round finality path with a 3-round finality fallback. If it receives 80% votes in the first round of voting, it commits immediately; otherwise it will wait for 67% votes and proceed with a second round. It thus achieves low latency in the common case and a safety bound of $n\ge 3f+1$. The paper claims a fast path for $n\ge 3f+2p+1$, where $p$ is the number of correct nodes that can crash (i.e., $Q_c=n-p$). Note that if Byzantine nodes vote for conflicting blocks, the protocol can fall back to the 3-round finality case where the $p$ failed nodes do not need to participate in the consensus. 
* [Alpenglow](https://www.anza.xyz/blog/alpenglow-a-new-consensus-for-solana): this is the recently proposed consensus protocol for Solana. Alpenglow is similar to Kudzu, with a 2-round fast path and a 3-round slow path. It similarly assumes that at most 20% of the stake are controlled by adversary, and at most 20% of stake might crash additionally. 
* [Hydrangea](https://eprint.iacr.org/2025/1112): this is a more recent protocol from Supra Research. It additionally introduces a tunable parameter $k\le2f+c-3$ (if $c$ is odd, or $k\le 2f+c-4$ if $c$ is even $\Rightarrow$ so that ). The fast path assumes $n=3f+2c+k+1$ where $f$ is the number of Byzantine nodes and $c$ is the maximum correct nodes that can crash. In Hydragea, $Q_c=n-\lfloor\frac{c+k}{2}\rfloor$, so it provides a way to improve resilience to crash failures by adding more validators. 

## Minimmit
[Minimmit](https://fc26.ifca.ai/preproceedings/171.pdf) is a recent BFT protocol with all-to-all communication and 2-round finality (commits in 1 RTT). Instead of introducing a slow fallback path when $5f+1$ fails, it lets validators proceed to the next view with only $2f+1$ votes. Why is this beneficial? In a WAN network, the latency distribution between each pair of validators will have a long tail. This means that waiting for 80% of the votes will take over 100 ms longer than waiting for only 40% of the votes. In fact, in the paper, by advancing view after receiving 40% of votes, Minimmit achieves 25% lower view latency compared with Simplex (which requires 67% of votes). While it cannot help with latency to finality, the reduced view change latency allows concurrent views, which directly translates to increased throughput. 

The basic intuition is the same as before: if there exists $I=n-3f$ (or $I=2f+1$ when $n=5f+1$) votes for block $A$, then no other blocks can receive $Q_c=n-f$ votes. The paper refers to a commit quorum of $Q_c$ votes as L-notarisation, a small quorum of $I$ votes as M-notarisation, and a timeout quorum of $I$ votes as nullification. The paper uses $Q_t=I$ instead of $Q_t=n-f$, under the premise that a timeout (nullify) message will only be sent if
1. Timeout triggered before casting a vote, or
2. The validator received a total of $I$ nullify messages or vote messages for blocks other than the one it votes for in the same round

This ensures that, if a commit quorum (L-notarisation) is formed before timeout, no timeout quorums can be formed. Finally, validators only vote for block $B$ in view $v$ that extends from block $B'$ proposed in view $v'$ if they have seen a notarisation for $B'$ and all views between $v'$ and $v$ times out. This ensures safety: no blocks other than $B'$ could have been committed in round $v'$ and no blocks (including $B'$) could have been committed in rounds between $v'$ and $v$. 