# leetcode146.-

<img width="1136" height="1551" alt="image" src="https://github.com/user-attachments/assets/7535e881-432d-4e32-8fb7-e5705c9349a8" />


### （1）题目白话文解释

题目的核心诉求是：设计一个类似于“储物盒”的数据结构，这个储物盒有两个特点：

1.  **容量有限**：最多只能装 `capacity` 个物品（键值对）。
2.  **淘汰机制（LRU，最近最少使用）**：当盒子装满，且需要放入新物品时，必须扔掉一个旧物品。扔掉的规则是：**挑出那个“最长时间没有被碰过”的物品扔掉**。
      * “碰过”的定义：不管是刚把物品放进去（`put`），还是从里面读取物品（`get`），都算作一次“碰过”。一旦被碰过，它就变成了“最近刚使用过”的物品，变得非常安全，不容易被扔掉。

**进阶要求解释：**

  * **$O(1)$ 时间复杂度**：意味着无论盒子里装了 10 个物品还是 10000 个物品，执行 `get`（查找）或 `put`（存入）的时间必须是一瞬间完成的，不能用 `for` 循环去挨个翻找哪个物品最旧。

-----

### （2）代码解题思路来源

为了满足苛刻的 $O(1)$ 时间复杂度，单靠一种基础数据结构是无法完成的，必须将**哈希表（字典）和双向链表**结合起来使用：

1.  **为什么需要哈希表？**
      * 如果只用数组或链表，想知道一个 `key` 存不存在，必须从头遍历，时间复杂度是 $O(N)$。
      * 哈希表（Python 中的 `dict`）可以实现 $O(1)$ 瞬间查找。所以用哈希表来记录 `key` 和对应节点的映射关系。
2.  **为什么需要双向链表？**
      * 哈希表是无序的，无法记录谁是最老的、谁是最新的。
      * 单向链表虽然能排队，但如果要删除中间的某个节点，必须找到它的前一个节点，这又需要 $O(N)$ 遍历。
      * **双向链表**完美解决这个问题：只要知道节点本身，通过 `node.prev` 和 `node.next` 瞬间就能把它从队伍中摘出来，并放到队伍末尾。
3.  **核心结构设计**：
      * 链表的\*\*头部（靠近 head）\*\*规定为：最久未使用的数据（淘汰区）。
      * 链表的\*\*尾部（靠近 tail）\*\*规定为：最近刚使用过的数据（安全区）。
      * 每次访问或新增数据，都把对应节点切断，重新接到尾部。当容量超标时，直接把头部第一个真实节点砍掉即可。

-----

### （3）带有注释的代码

```python
class ListNode:
    # 定义双向链表的节点结构
    def __init__(self, key=None, value=None):
        self.key = key
        self.value = value
        self.prev = None  # 指向前一个节点
        self.next = None  # 指向后一个节点

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.hashmap = {}  # 字典，存放 key -> ListNode 的映射
        
        # 新建两个虚拟（哑）节点 head 和 tail，纯粹为了占位，方便增删操作
        self.head = ListNode()
        self.tail = ListNode()
        # 初始化链表，让首尾相连： head <-> tail
        self.head.next = self.tail
        self.tail.prev = self.head

    # 辅助方法：将一个已经存在的节点，转移到链表的最末尾（标记为最新访问）
    def move_node_to_tail(self, key):
        node = self.hashmap[key]
        
        # 步骤 1：把 node 从当前位置摘出来
        node.prev.next = node.next
        node.next.prev = node.prev
        
        # 步骤 2：把 node 塞到 tail 的前面
        node.prev = self.tail.prev
        node.next = self.tail
        self.tail.prev.next = node
        self.tail.prev = node

    def get(self, key: int) -> int:
        # 如果能在哈希表里找到，说明缓存命中了
        if key in self.hashmap:
            # 既然被访问了，就必须把它移到代表“最新”的尾部
            self.move_node_to_tail(key)
            return self.hashmap[key].value
        # 找不到直接返回 -1
        return -1

    def put(self, key: int, value: int) -> None:
        if key in self.hashmap:
            # 场景 A：键已经存在，属于“更新”操作
            self.hashmap[key].value = value
            self.move_node_to_tail(key)
        else:
            # 场景 B：键不存在，属于“新增”操作
            if len(self.hashmap) == self.capacity:
                # 场景 B1：如果容量满了，必须先淘汰最久未使用的元素（即 head 后面的第一个元素）
                evict_node = self.head.next
                # 从哈希表中删除对应记录
                self.hashmap.pop(evict_node.key)
                # 从双向链表中砍掉这个节点
                self.head.next = evict_node.next
                evict_node.next.prev = self.head
                
            # 场景 B2：执行新增。创建新节点，存入哈希表，并直接插到尾部
            new_node = ListNode(key, value)
            self.hashmap[key] = new_node
            
            # 插入到 tail 前面
            new_node.prev = self.tail.prev
            new_node.next = self.tail
            self.tail.prev.next = new_node
            self.tail.prev = new_node
```

-----

### （4）每一行代码详细解释

  * `class ListNode:` 到 `self.next = None`：定义了双向链表的基础构件。不仅存了 `value`，还存了 `key`（因为在淘汰节点时，需要通过节点的 `key` 反向去删除哈希表里的记录）。
  * `self.head = ListNode(); self.tail = ListNode()`：设立“哑节点（Dummy Node）”。这是链表操作的经典技巧，有了这两个永远不删除的门神，后续增删任何节点时，就不需要写繁琐的 `if node.prev is None:` 之类的边界判断了。
  * `move_node_to_tail(self, key)`：核心复用逻辑。先通过 `prev.next = next` 和 `next.prev = prev` 让该节点原先的左右邻居直接牵手，从而孤立该节点。然后重新设定该节点的前驱为 `tail.prev`，后继为 `tail`，最后让 `tail` 及其原前驱接纳它。
  * `get` 方法：利用 `key in self.hashmap` 瞬间判断是否存在。存在则调用 `move_node_to_tail` 刷新其活跃度，并返回值。
  * `put` 方法中 `if len(self.hashmap) == self.capacity:`：这是触发 LRU 淘汰机制的阈值判断。
  * `self.head.next = evict_node.next` 和 `evict_node.next.prev = self.head`：这两步直接跳过了 `evict_node`（最老的节点），让 `head` 和 `evict_node` 的下一个节点直接相连，完成了老节点的物理删除。

-----

### （5）具体数值算例及变量变化表格

以题目示例 `capacity = 2` 为例进行推演。
*注：表格中链表状态省略了 dummy head 和 tail，仅展示真实节点数据 `[key:value]`。左侧为最旧（head端），右侧为最新（tail端）。*

| 执行操作 | 判断逻辑与动作 | 当前双向链表状态 (左旧 -\> 右新) | 哈希表存活的 Keys | 返回结果 | 对应代码核心逻辑 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `put(1, 1)` | 不存在且未满，直接插到尾部 | `[1:1]` | `1` | `null` | 走 `put` 的 `else` 新增逻辑 |
| `put(2, 2)` | 不存在且未满，插到尾部 | `[1:1] <-> [2:2]` | `1, 2` | `null` | 同上 |
| `get(1)` | 存在。刷新活跃度，移至尾部 | `[2:2] <-> [1:1]` | `1, 2` | **`1`** | `get` -\> `move_node_to_tail` |
| `put(3, 3)` | **超载！** 淘汰头部 `[2:2]`。<br>插入新节点 `[3:3]` 至尾部 | `[1:1] <-> [3:3]` | `1, 3` | `null` | `put` -\> `if len == capacity` 淘汰头部并新增 |
| `get(2)` | 哈希表中找不到键 `2` | `[1:1] <-> [3:3]` | `1, 3` | **`-1`** | `get` 返回 `-1` |
| `put(4, 4)` | **超载！** 淘汰头部 `[1:1]`。<br>插入新节点 `[4:4]` 至尾部 | `[3:3] <-> [4:4]` | `3, 4` | `null` | 同第三次 put |
| `get(1)` | 哈希表中找不到键 `1` | `[3:3] <-> [4:4]` | `3, 4` | **`-1`** | `get` 返回 `-1` |
| `get(3)` | 存在。刷新活跃度，移至尾部 | `[4:4] <-> [3:3]` | `3, 4` | **`3`** | `get` -\> `move_node_to_tail` |
