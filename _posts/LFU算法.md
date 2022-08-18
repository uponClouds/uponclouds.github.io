# LFU算法

LFU(Least Frequently Used)算法是一种经典的缓存淘汰策略算法，它的核心思想是淘汰最近最少使用的对象。在操作系统中，它可以用作进行页面置换算法。

## 算法题例

(LeetCode-460)实现 `LFUCache` 类：

- `LFUCache(int capacity)` - 用数据结构的容量 `capacity` 初始化对象
- `int get(int key)` - 如果键 `key` 存在于缓存中，则获取键的值，否则返回 `-1`
- `void put(int key, int value)` - 如果键 `key` 已存在，则变更其值；如果键不存在，请插入键值对。当缓存达到其容量 `capacity` 时，则应该在插入新项之前，移除最不经常使用的项。在此问题中，当存在平局（即两个或更多个键具有相同使用频率）时，应该去除最近最久未使用的键

为了确定最不常使用的键，可以为缓存中的每个键维护一个使用计数器 。使用计数最小的键是最久未使用的键。当一个键首次插入到缓存中时，它的使用计数器被设置为 `1` (由于 `put` 操作)。对缓存中的键执行` get` 或 `put` 操作，使用计数器的值将会递增。函数 `get` 和 `put` 必须以 `O(1)` 的平均时间复杂度运行。

### 题解

首先`LFU`算法要求我们在`get`时要达到 `O(1)`时间复杂度，那么首先想到的就是利用 `HashMap`来进行储存。而且我们在进行`get`时，相当于访问了这个元素，那么它的访问频率也需要更新。在进行`put`时，在空间不足时要删除某个对象，要想达到`O(1)`的时间复杂度那就必须构建一个 **频率-->对象** 的一个映射，这样一来就需要两个`HashMap`来处理了。同时我们要注意的是，多个元素可能会有相同的频率值，因此 **频率-->对象** 映射中的 `value`部分是若干个对象，那么我们可以采用双链表来将这些对象串起来，以方便我们进行增删。 

在使用 `Java`来编写算法时，可以使用 `LinkedHashSet`集合来代替双链表进行处理，当然，为了更好地理解，首先还是自己编写一个双向链表来处理，代码如下：

```java
public class LFUCache {
    // 基本节点
    private class Node {
        // freq 为对象的频率，每当get或put相同key时就会增1
        int key, value, freq;
        Node next, pre;
        public Node(int key, int value) {
            this.key = key;
            this.value = value;
            this.freq = 1;
        }
    }
    
    // 双向节点链表
    private class NodeList {
        // 为了方便增删，设置head与tail为链表的头和尾
        Node head, tail;
        int size;
        public NodeList() {
            head = new Node(0, 0);
            tail = new Node(0, 0);
            head.next = tail;
            tail.pre = head;
            size = 0;
        }
      
        // 删除链表中第一个节点
        public void removeFirst() {
            Node delNode = head.next;
            if (delNode == tail) {
                throw new IllegalStateException("the NodeList is empty.");
            }
            removeNode(delNode);
        }

        // 在链表尾部插入节点
        public void addLast(Node node) {
            Node pre = tail.pre;
            pre.next = node;
            node.pre = pre;
            tail.pre = node;
            node.next = tail;
            size++;
        }

        // 删除链表中指定节点（非head、tail节点）
        public void removeNode(Node node) {
            Node pre = node.pre;
            Node next = node.next;
            pre.next = next;
            next.pre = pre;
            size--;

            // help GC
            node.pre = null;
            node.next = null;
        }
    }
  	
    Map<Integer, Node> nodeMap;
    // 频率哈希表，key为频率值
    Map<Integer, NodeList> freqMap;
    // 能够储存的最多节点值数
    int capacity;
    // 当前包含节点的最小频率，用于淘汰节点
    int minFreq;
    // 当前节点数量
    int currSize;
  
    public LFUCache(int capacity) {
        nodeMap = new HashMap<>(capacity);
        freqMap = new HashMap<>();
        freqMap.put(1, new NodeList());
        this.capacity = capacity;
        currSize = 0;
        minFreq = 1;
    }

    public int get(int key) {
        Node node = nodeMap.get(key);
        if (node == null) return -1;
        // 将对应节点的频率更新
        freqInc(node);
        return node.value;
    }

    public void put(int key, int value) {
        // 当所给空间大小为0时，跳过，啥也不做
        if (capacity == 0) return;

        Node node = nodeMap.get(key);
        if (node == null) {
            // 空间不足，需要先删除最少使用的元素
            if (capacity == currSize) {
                // 删除节点后为什么不去判断当前链表是否为
                // 空然后更新minFreq呢？
                // 因为我们接下来是要插入节点的，所以交给#109行代码去做就好啦
                removeLeastFreqNode();
                currSize--;
            }
            // 新建节点，并且存入freqMap和nodeMap
            currSize++;
            node = new Node(key, value);
            freqMap.get(1).addLast(node);
            nodeMap.put(key, node);

            // 考虑这种情况：当现有元素频率都是2或2以上时，minFreq为2，
            // 若新增了一个元素，则要将其置为1
            if (minFreq > 1) minFreq = 1;
            return;
        }
        // 走到这里表示存在这个key，只要更新value值
        // 和freqMap中对应的频率即可
        node.value = value;
        freqInc(node);
    }

    /**
     * 用于将节点node从当前频率链表中删除，然后添加到新的频率链表中去
     */
    private void freqInc(Node node) {
        int freq = node.freq;
        node.freq++;
        NodeList oldList = freqMap.get(freq);

        // 如果删除节点后当前链表空了，且最小频率是当前这个频率，则更新最小频率
        oldList.removeNode(node);
        if (oldList.size == 0 && minFreq == freq) {
            minFreq = freq + 1;
        }

        // 把节点添加到新频率（即旧频率+1）链表
        freqMap.putIfAbsent(freq + 1, new NodeList());
        NodeList newList = freqMap.get(freq + 1);
        newList.addLast(node);
    }

    private void removeLeastFreqNode() {
        NodeList list = freqMap.get(minFreq);
        Node delNode = list.head.next;
        nodeMap.remove(delNode.key);
        list.removeFirst();
    }
}
```



