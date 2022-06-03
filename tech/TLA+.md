[toc]



# | Concepts

逻辑运算

- `/\` AND
- `\/` OR
- `/=` 不等于



集合

- `\cup`   并集、增加元素

  ```
  # 增加元素
  RRAndNode == 
    /\ reg_nodeIds' = reg_nodeIds \cup {reg_nextId}
    /\ reg_nextId' = reg_nextId + 1
  ```

- `\`  删除元素

- `\cap` 交集

- `\notin` 过滤

  ```
  # 判断是否包含在集合中
  IsConnected(n) == n \notin node_disconnected
  ```

- `\A` 所有元素

  ```
  # 判断对于集合中所有元素都满足某个条件：本例中判断所有元素都比传入的 nodeId大
  HasMinId(nodeId) == 
    \A n \in reg_nodeIds : nodeId <= n
  ```

- `\E` 存在某个元素 

  ```
  # 存在某个节点满足某些条件
  \E nd \in LiveNodes : \/ StopConnectedNode(nd)
                        \/ StopIsolatedNode(nd)
                        \/ ExpireNode(nd)
  ```

  





Map

- `function` 字典

- `domain` 字典 key

  ```
  # 例如NodeData是一个 Record：
  Phase == { "-", "stopped", "started" }
  Role == { "offline", "none", "follower", "leader" }
  NodeData == [id : Nat, role : Role, aVersion : Nat, rbPhase : Phase]
  
  
  NoNode == [id |-> 0, role |-> "offline", aVersion |-> 0, rbPhase |-> "-"]
  
  # creates a function whose domain is the set N 
  # and where each node points to a record with an id of 0, role offline etc.
  Init == 
    node_data = [n \in N |-> NoNode]
  ```

- 修改 function：指定要修改的元素，同时还要标明其他元素不变，使用 `EXCEPT!`

  ```
  # 只修改节点 n，其他节点保持不变
  node_data' = [node_data EXCEPT ![n] = [id |-> reg_nextId, 
                                         role |-> "none", 
                                         aVersion |-> reg_aVersion, 
                                         rbPhase |-> "-" ]]
  ```

  



# | TLA+ Toolbox

> https://github.com/tlaplus/tlaplus/releases





# | References

Tutorials

- 文字教程
  https://learntla.com/introduction/ 
- 视频教程
  http://lamport.azurewebsites.net/tla/tla.html 
  https://lamport.azurewebsites.net/video/videos.html 

Cases

- Rebalanser: Partition assigner
  https://jack-vanlightly.com/blog/2019/1/27/building-a-simple-distributed-system-formal-verification
  https://github.com/Rebalanser/wiki/tree/master/tlaplus/LeaderResourceBarrier
- BookKeeper TLA+
  https://github.com/Vanlightly/bookkeeper-tlaplus 
  https://medium.com/splunk-maas/modelling-and-verifying-the-bookkeeper-protocol-tla-series-part-3-ef8a9850ad63 
- Kafka
  https://www.slideshare.net/ConfluentInc/hardening-kafka-replication 
- https://sookocheff.com/post/tlaplus/getting-started-with-tlaplus/ 





