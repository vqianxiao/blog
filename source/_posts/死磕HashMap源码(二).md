---
layout:    post
title:     死磕HashMap源码(二)
category:  源码解析
description: 死磕HashMap源码(二) 红黑树相关的操作
tags: JDK
date: 2021/08/12 19:42:10


---

Jdk8以前，使用的是链表去解决hash冲突的，这样就会导致一个问题，如果这个链表很长，那么从链表中定位到数据的时间复杂度就是O(n)，链表越长性能越差。因此Jdk8对HashMap进行了优化，引入了自平衡的红黑树结构，让定位元素的时间复杂度优化成O(logn)，这样就可以提升元素的查找效率，但是也不是完全摒弃了链表，因为元素不多的情况下，链表的插入速度更快。

看下树化的源码

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //这里还要进行判断 如果桶容量小于64还是会进行扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            //这里只是将原来的Node替换成了TreeNode
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            //这里进行红黑树的转化
            hd.treeify(tab);
    }
}

TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}
```

通过上面的代码可以看到，链表转红黑树其实是有两个条件的，一个是链表长度大于8，还有一个就是桶容量要大于64，否则，就只会扩容不会树化。

然后treeifyBin方法中将Node节点转化为TreeNode，此时只是节点类型转化，并没有实际的树化，并且记录了链表的顺序。

然后来看下TreeNode的结构，方法我没粘，需要的时候再看。

```java
 static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
     TreeNode<K,V> parent;  // red-black tree links
     TreeNode<K,V> left;
     TreeNode<K,V> right;
     TreeNode<K,V> prev;    // needed to unlink next upon deletion
     boolean red;
     TreeNode(int hash, K key, V val, Node<K,V> next) {
         super(hash, key, val, next);
     }
 }
```

上面看了链表树化的操作，然后来看下树转链表的操作。

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    //这里遍历树节点 然后转化成Node节点 hd是头 tl是尾 根据树节点保存的顺序恢复链表的顺序
    for (Node<K,V> q = this; q != null; q = q.next) {
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}

Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
    return new Node<>(p.hash, p.key, p.value, next);
}
```

我们已经知道了链表转树的条件，那么树是什么时候转成链表呢？其实可以猜个大概，因为扩容和删除节点的都会让树节点变少。

1.扩容的时候。如果扩容那部分仔细看的话，就知道扩容的时候，其实就是把桶里的元素的index进行重新分配，让它们更分散，也包括树。这个时候就需要把树像链表一样拆开，然后去看这些元素哪些该喊0哪些该喊1，进而给它们重新分配桶，这样一个大树可能就变成了一颗小树，如果小于等于非树化阈值，那么就转成链表。

2.删除节点的时候。

扩容的时候判断条件就是小于等于`UNTREEIFY_THRESHOLD`就会将树转成链表。但是删除的时候不是通过这个阈值控制的，可以看下条件。

```java
//movable 这个参数删除的时候是写死的为true 所以只要root.right == null 或者root.left == null 或者 root.left.left == null
//可以根据红黑树的性质去推断，当根节点的右节点为空时，从根节点出发路径有一个黑色节点。那么此时从根节点出发，到左边的NIL节点应该只有一个红色节点
//此时树只有2个元素。从根出发左节点为空同理。也是只有2个元素。
//如果左节点的左节点为空那么左节点为黑色，右节点为红色及它的两个子节点为黑色，那么此时最多5个元素。
//但是为什么不用UNTREEIFY_THRESHOLD去计算 我还是没有想明白。不过作用是类似的，只是少了个树里6个元素的情况
if (root == null
    || (movable
        && (root.right == null
            || (rl = root.left) == null
            || rl.left == null))) {
    tab[index] = first.untreeify(map);  // too small
    return;
}
```



为了深入理解红黑树，需要先从二叉查找树说起。

#### BST

二叉查找树（Binary Search Tree 简称BST）是一颗二叉树，它的左子节点的值比父节点的值要小，右节点的值要比父节点的值大。它的高度决定了它的查找效率。在理想的情况下，二叉查找树增删改查的时间复杂度为O(logN)（其中N为节点数），最坏的情况下为O(N)。当它的高度为logN+1时，我们就说二叉查找树是平衡的。

![](https://vqianxiao.github.io/blog/images/hashmap/bst.png)

BST的查找

```java
T  key = a search key
Node root = point to the root of a BST

 //其实思路就是找到根 然后去判断 相等返回。小于就从左子树查找，大于就从右子树查找 直到当前节点指针为空或者找到对应的节点
while(true){
    if(root==null){
    	break;
    }
    if(root.value.equals(key)){
    	return root;
    }
    else if(key.compareTo(root.value)<0){
    	root = root.left;
    }
    else{
    	root = root.right;
    }
}
return null;
```

BST的插入

```java
Node node = create a new node with specify value
Node root = point the root node of a BST
Node parent = null;

//find the parent node to append the new node
while(true){
   if(root==null)break;
   parent = root;
   if(node.value.compareTo(root.value)<=0){
      root = root.left;  
   }else{
      root = root.right;
   } 
}
if(parent!=null){
   if(node.value.compareTo(parent.value)<=0){//append to left
      parent.left = node;
   }else{//append to right
	  parent.right = node;
   }
}

```

插入操作先通过循环找到要插入的节点的父节点，然后对比父节点，小的就插入到父节点的左节点，大的就插入到父节点的右节点。

BST的删除

删除步骤如下：

1.查找要删除的节点

2.如果待删除的节点是叶子节点，则直接删除

3.如果删除的节点不是叶子节点，则先找到待删除节点的中序遍历的后继节点，用该后继节点的值替换待删除的节点的值，然后删除后继节点。

![](https://vqianxiao.github.io/blog/images/hashmap/bstdel.png)

BST的缺陷

BST的主要问题，数据在插入的时候，会导致树倾斜，不同的插入顺序会导致树的高度不一样，而树的高度直接影响了树的查找效率。理想的高度是logN，最坏的情况就是所有的节点都在一条斜线上，这样的树的高度为N。

#### RBTREE

基于BST的问题，一种新的树——平衡二叉查找树产生了。平衡树在插入和删除的时候，会通过旋转操作将高度保持在logN。其中两款具有代表性的平衡树分别为AVL树和红黑树。AVL树由于实现比较复杂，而且插入和删除性能差，在实际环境下的应用不如红黑树。红黑树应用非常广泛，在Linux内核中的完全公平调度器、高精度计时器、ext3文件系统等等，各种语言的函数库如Java的TreeMap和TreeSet，C++ STL的map、multimap、multiset等。值得一提的是，Java8中HashMap的实现也因为用RBTree取代链表，性能有所提升。

> 1.任何一个节点都有颜色，节点是红色或者黑色
>
> 2.根节点是黑色的
>
> 3.父子节点之间不能出现两个连续的红节点（红色节点的子节点只能是黑色节点） 
>
> 4.从任意节点到叶子节点的所有路径都包含相同数目的黑色节点
>
> 5.所有叶子节点都是黑色节点（叶子是NIL结点，默认省略）

RBTree理论上还是一颗BST树，但是在对BST的插入和删除操作时为维持树的平衡，即保证树的高度在[logN,logN+1]（理论上，极端情况下可以持续爱你RBTree的高度达到2*logN，但实际上很难遇到）。这样RBTree的查找时间复杂度始终保持在O(logN)从而接近于理想的BST。RBTree的删除和插入操作的时间复杂度也是O(logN)，RBTree的查找操作和BST的查找操作一致。

RBTree的旋转操作

旋转操作（Rotate）的目的是使节点颜色符合定义，让RBTree的高度达到平衡。Rotate分为left-rotate（左旋）和right-rotate（右旋），区分左旋和右旋的方法是：待旋转的节点从左边上升到父节点就是右旋，待旋转的节点从右边上升到父节点就是左旋。

![](https://vqianxiao.github.io/blog/images/hashmap/rotateRBTree.png)

RBTree的查找操作

RBTree的查找操作和BST的查找操作是一样的。

#### RBTree的插入操作

RBTree的插入操作和BST的插入方式一致，不过在插入后，可能会导致树的不平衡，这时就需要对树进行旋转操作和颜色修复（也叫变色），使得它符合RBTree的定义。

新插入的节点是红色的，插入修复操作如果遇到父节点的颜色为黑则修复操作结束。也就是说，只有在父节点为红色结点的时候需要插入修复操作。

插入修复操作分为以下三种情况，而且新插入的结点的父节点都是红色的：

1.叔叔结点也为红色

2.叔叔结点为空，且祖父结点、父节点和新节点处于一条斜线上

3.叔叔结点为空，且祖父结点、父节点和新节点不出与一条斜线上

##### 插入操作-case1 第一种情况（叔叔节点也为红色）

将父节点和叔叔节点与祖父节点的颜色互换，这样就符合了RBTree的定义。即维持了高度的平衡，修复后颜色也符合RBTree定义的第三条和第四条。下图中，操作完成后A节点变成了新的节点。如果A节点的父节点不是黑色的话，则继续做修复操作。

![](https://vqianxiao.github.io/blog/images/hashmap/insertcase1.png)

##### 插入操作-case2 第二种情况（叔叔节点为空，且祖父节点、父节点和新节点处于一条斜线上）

将B节点进行右旋操作，并且和父节点A互换颜色，通过该修复操作BRTree的高度和颜色都符合红黑树的定义。如果B和C节点都是右节点的话，只要将操作变成左旋就可以了。

![](https://vqianxiao.github.io/blog/images/hashmap/insertcase2.png)

##### 插入操作-case3 第三种情况（叔叔结点为空，且祖父结点、父节点和新节点不出与一条斜线上）

将C节点进行左旋，这样就将第三种情况转换成第二种情况了，然后针对第二种情况进行操作处理就可以了。case2操作做了一个右旋操作和颜色互换来达到目的。如果树的结构是下图的镜像结，则只需要将对应的左旋变成右旋，右旋变成左旋即可。

![](https://vqianxiao.github.io/blog/images/hashmap/insertcase3.png)

**插入操作的总结**

插入后的修复操作时一个向root节点回溯的操作，一旦涉及的节点都符合了红黑树的定义，修复操作结束。之所以会向上回溯是由于case1操作会将父节点、叔叔节点和祖父节点进行变色，有可能会导致祖父节点不平衡（红黑树定义三）。这个时候需要对祖父节点为起点进行调节（向上回溯）。

祖父节点调节后如果还是遇到它的祖父颜色问题，操作就会继续向上回溯，直到root节点为止，根据定义root节点永远是黑色的。在向上的回溯过程中，针对插入的3种情况进行调节，直到符合红黑树的定义为止。直到涉及的节点都符合了红黑树的定义，修复操作结束。

如果上面的3种情况对应的操作是在右子树上，对对应的镜像操作就可以了。

#### RBTree的删除操作

删除操作首先要做的也是BST的删除操作，删除操作会删除对应的节点，如果是叶子结点就直接删除，如果是非叶子结点，会用对应的中序遍历的后继节点来顶替要删除节点的位置。删除后就需要做删除修复操作，使树符合红黑树的定义，符合定义的红黑树高度是平衡的。

删除修复操作在遇到被删除的节点是红色节点或者到达root节点时，修复操作完毕。

删除修复操作时针对删除黑色节点才有的，当黑色节点被删除后会让整个树不符合RBTree的定义的第四条。需要做的处理时从兄弟节点上借调黑色的节点过来，如果兄弟节点没有黑色节点可以借调的话，就只能向上追溯，将每一级的黑色节点数减去一个，使得整棵树符合红黑树的定义。

删除操作总体思想时从兄弟节点借调黑色节点使树保持局部的平衡，如果局部的平衡达到了，就看整体的树是否是平衡的，如果不平衡就接着向上追溯调整。

删除修复操作分为四种情况，并且只针对黑色节点的删除：

1.待删除的节点的兄弟节点是红色的节点

2.待删除的节点的兄弟节点是黑色的节点，且兄弟节点的子节点都是黑色的

3.待调整的节点的兄弟节点是黑色的节点，且兄弟节点的左子节点是红色的，右节点是黑色的（兄弟节点在右边），如果兄弟节点在左边的话，就是兄弟节点的右子节点是红色的，左节点是黑色的

4.待调整的节点的兄弟节点是黑色的节点，且右子节点是红色的（兄弟节点在右边），如果兄弟节点在左边，则对应的就是左节点是红色的

##### 删除操作-case1（待删除的节点的兄弟节点是红色的节点）

由于兄弟节点是红色结点的时候，无法借调黑色节点，所以需要将兄弟节点提升到父节点，由于兄弟节点是红色的，根据RBTree的定义，兄弟节点的子节点是黑色的，就可以从它的子节点借调了。

case1这样转换以后就会变成后面的case2，case3，或者case4进行处理了。上升操作需要对C做一个左旋操作，如果是镜像结构的树只需要做对应的右旋操作即可。

之所以要做case1操作是因为兄弟节点是红色的，无法借到一个黑色节点来填补删除的黑色节点。

![](https://vqianxiao.github.io/blog/images/hashmap/delcase1.png)

##### 删除操作-case2 （待删除的节点的兄弟节点是黑色的节点，且兄弟节点的子节点都是黑色的）

case2的删除操作是由于兄弟节点可以消除一个黑色节点，因为兄弟节点和兄弟节点之间的子节点都是黑色的，所以可以将兄弟节点变红，这样就可以保证树的局部的颜色符合定义了。这个时候需要将父节点A变成新的节点，继续向上调整，直到整棵树的颜色符合RBTree的定义为止。

case2这种情况下之所以要将兄弟节点变红，是因为如果把兄弟节点借调过来，会导致兄弟的结构不符合RBTree的定义，这样的情况下只能是将兄弟节点也变成红色来达到颜色的平衡。当将兄弟节点也变红之后，达到了局部的平衡了，但是对于祖父节点来说是不符合定义4的（因为了个黑色的节点）。这样就需要回溯到父节点，接着进行修复操作。

![](https://vqianxiao.github.io/blog/images/hashmap/delcase2.png)

##### 删除操作-case3（待调整的节点的兄弟节点是黑色的节点，且兄弟节点的左子节点是红色的，右节点是黑色的（兄弟节点在右边），如果兄弟节点在左边的话，就是兄弟节点的右子节点是红色的，左节点是黑色的）

case3的删除操作是一个中间步骤，它的目的是将左边的红色节点借调过来，这样就可以转换成case4状态了，在case4状态下可以将D、E节点都借调过来，通过将两个节点变成黑色来保证红黑树的整体平衡。

之所以说case3是一个中间状态，是因为根据红黑树的定义来说，下图并不是平衡的，它是通过case2操作完后向上回溯出现的状态。之所以会出现case3和后面的case4的情况，是因为可以通过借调侄子节点的红色，变成黑色来符合定义4。

![](https://vqianxiao.github.io/blog/images/hashmap/delcase3.png)

##### 删除操作-case4 （待调整的节点的兄弟节点是黑色的节点，且右子节点是红色的（兄弟节点在右边），如果兄弟节点在左边，则对应的就是左节点是红色的）

case4的操作是真正的节点借调操作，通过将兄弟节点以及兄弟节点的右节点借调过来，并将兄弟节点的右子节点变成红色来达到借调两个黑色节点的目的，这样的话，整棵树还是符合RBTree定义的。

case4这种情况的发生只有在待删除的节点的兄弟节点为黑，且子节点不全部为黑，才有可能借调到两个节点来做黑色节点使用，从而保持整棵树都符合红黑树的定义。

![](https://vqianxiao.github.io/blog/images/hashmap/delcase4.png)

**删除操作的总结**

红黑树的删除操作是最复杂的操作，复杂的地方就在于当删除了黑色节点的时候，如何从兄弟节点去借调节点，以保证树的颜色符合定义。由于红色的兄弟节点是没法借调出黑色节点的，这样只能通过旋转操作让他上升到父节点，而由于它是红节点，所以它的子节点就是黑的，可以借调。

对于兄弟节点是黑色节点的可以分成3种情况来处理，当所有的兄弟节点的子节点都是黑色节点时，可以直接将兄弟节点变红，这样局部的红黑树颜色是符合定义的。但是整棵树不一定是符合红黑树定义的，需要向上追溯继续调整。

对于兄弟节点的子节点为左红右黑或者（全部为红）右红左黑这两种情况，可以先将前面的情况通过选择转换为后一种情况，在后一种情况下，因为兄弟节点为黑，兄弟节点的右节点为红，可以借调出两个节点出来做黑节点，这样就可以保证删除了黑节点，整棵树还是符合红黑树的定义的，因为黑色节点的个数没有改变。

红黑树的删除操作是遇到删除的节点为红色，或者追溯调整到了root节点，这时删除的修复工作完毕。

**红黑树总结**

作为平衡二叉查找树里面众多的实现之一，红黑树无疑是最简洁、实现最为简单的。红黑树通过引入颜色的概念，通过颜色这个约束条件的使用来保持树的高度稳定。作为平衡二叉查找树，旋转是一个必不可少的操作，通过旋转可以降低树的高度，在红黑树里面还可以转换颜色。

红黑树里面的插入和删除的操作比较难理解，这是要注意记住一点：操作之前红黑树是平衡的，颜色是符合定义的，在操作的时候就需要向兄弟节点、父节点、侄子节点借调和互换颜色，要达到这个目的，就需要不断的进行旋转。所以红黑树的插入删除操作需要不停的旋转，一旦借调了别的节点，删除和插入的节点就会达到局部的平衡（局部符合红黑树的定义），但是被借调的节点就不会平衡了，这时就需要以被借调的节点为起点继续进行调整，直到整棵树都是平衡的。在整个修复的过程中，插入可以分3种情况，删除分4种情况。

整个红黑树的查找，插入和删除都是O(logN)的，原因就是整个红黑树的高度是logN，查找从根到叶，走过的路径是树的高度，删除和插入操作是从根到叶的，所以经过的路径都是logN。

>参考自 https://tech.meituan.com/2016/12/02/redblack-tree.html