---
title: Red-Black Tree
date: 2018-08-18 20:43:19
header-img: https://i.imgur.com/RAwKwAj.jpg
cdn: 'header-off'
tags:
    - DataStructure
---

## 红黑树的定义
红黑树是平衡二叉搜索树的其中一种，它的节点有一个标志位用来标识该节点是红色或者黑色，
节点颜色的作用是帮树保持平衡。
![RBT](https://upload.wikimedia.org/wikipedia/commons/6/66/Red-black_tree_example.svg)

## 红黑树必须满足的约束条件
除了满足平衡二叉搜索树的约束条件之外，红黑树还必须满足以下几个约束条件（参考维基百科的定义）：

1. Each node is either red or black.（每个节点只能是红色或者黑色）

2. The root is black. （根节点为黑色）This rule is sometimes omitted. Since the root can always be changed from red to black, but not necessarily vice versa, this rule has little effect on analysis.

3. All leaves (NIL) are black.（所有叶子节点都是黑色的）

4. If a node is red, then both its children are black.（红色节点的两个孩子都是黑色的）

5. Every path from a given node to any of its descendant NIL nodes contains the same number of black nodes.（任意一个非叶子节点到的它子孙的叶子节点的路径的黑色节点个数都是相等的）

>The leaf nodes of red–black trees do not contain data. These leaves need not be explicit in computer memory—a null child pointer can encode the fact that this child is a leaf—but it simplifies some algorithms for operating on red–black trees if the leaves really are explicit nodes. To save execution time, sometimes a pointer to a single sentinel node (instead of a null pointer) performs the role of all leaf nodes; all references from internal nodes to leaf nodes then point to the sentinel node.

需要注意的是，红黑树的所有叶子节点都是空节点，不包含数据，且都是黑色的，这些节点可以是概念上存在（不占用内存，用null表示），也可以是真实存在（占用内存，用一个真正的节点表示）。

## 左旋
```angularjs
void rotate_left(struct node* n) {
 struct node* nnew = n->right;  //将要左旋的节点的右孩子存储到nnew变量中
 assert(nnew != LEAF); // since the leaves of a red-black tree are empty, they cannot become internal nodes
 //如过右孩子节点不存在（为叶子节点），就没办法左旋了
 n->right = nnew->left;  //将右孩子节点(nnew)的左孩子(nnew->left)变成要左旋的节点（n）的右孩子(n->right)
 nnew->left = n;  //将要左旋的节点（n）变成右孩子节点（nnew）的左孩子（nnew-left）
 nnew->parent = n->parent;  //将要左旋的节点（n）的父亲(n->parent)设置成右孩子节点（nnew）的父亲（nnew->parent）
 n->parent = nnew;  //将要右孩子节点（nnew）设置成要左旋的节点（n）的父亲(n->parent)
 // (the other related parent and child links would also have to be updated)
}
```
左旋的主要步骤如下：

1. 检查要左旋的节点的右孩子节点是否为空，如果是，则不能进行左旋。

2. 如果右孩子节点有左孩子节点，则将右孩子节点的左孩子节点变成要左旋的节点的右孩子

3. 将要左旋的节点变成第1步说的右孩子节点的左孩子

4. 如果要左旋的节点原来有父亲，则将第1步说的右孩子节点的父亲变成要左旋的节点的父亲，没有的话第1步说的右孩子节点的父亲设置为空

5. 如果要左旋的节点有父亲且是它父亲的左孩子，将则将第1步说的右孩子节点设置成他的父亲节点的左孩子，否则设置成他的父亲节点的右孩子。

6. 将要左旋的节点的父亲节点设置成第1步说的右孩子节点

结合上面的步骤分析一下JDK1.8中HashMap中红黑树的左旋代码：
```angularjs
//红黑树节点的结构
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

static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            //r为要左旋的节点的右孩子节点，pp为p的父亲节点，rl为r的左孩子节点
            //首选判断p是否为空和p的右孩子是否为空，如果是则不进行左旋，返回root节点
            if (p != null && (r = p.right) != null) {
                //将r的左孩子变成p的右孩子
                if ((rl = p.right = r.left) != null)
                    rl.parent = p;  //将r的左孩子的父亲设置成p
                //将r的父亲设置成p的父亲节点
                if ((pp = r.parent = p.parent) == null)
                //如果p没有父亲，说明p是根节点，左旋后r将变成根节点，红黑树的根节点一定是黑色，所以将r设置成黑色
                    (root = r).red = false;
                else if (pp.left == p)
                    //如果p是他的父亲的左子树，则将r设置成p的父亲的左子树
                    pp.left = r;
                else
                  //如果p是他的父亲的右子树，则将r设置成p的父亲的右子树
                    pp.right = r;
                r.left = p;  //将p设置成r的右子树
                p.parent = r;  //将r设置为p的父亲
            }
            return root;  //返回根节点，经过旋转后根节点可能会发生变化
        }
```

## 右旋
```angularjs
void rotate_right(struct node* n) {
 struct node* nnew = n->left;  //将要右旋的节点的右孩子存储到nnew变量中
 assert(nnew != LEAF); // since the leaves of a red-black tree are empty, they cannot become internal nodes
 //如过左孩子节点不存在（为叶子节点），就没办法右旋了
 n->left = nnew->right;  //将左孩子节点(nnew)的右孩子(nnew->right)变成要右旋的节点（n）的左孩子(n->left)
 nnew->right = n;  // //将要右旋的节点（n）变成左孩子节点（nnew）的右孩子（nnew-right）
 nnew->parent = n->parent;  //这里和后面的都不再赘述，逻辑和左旋反过来就行
 n->parent = nnew;
 // (the other related parent and child links would also have to be updated)
}
```
可以看到右旋的操作和左旋基本上是一直的，只是方向反过来了，比如左旋是判断右孩子是否存在，然后将有孩子的左孩子变成左旋节点的右孩子，
而右旋则是判断左孩子是否存在，然后将左孩子的右孩子变成右旋节点的左孩子。
步骤和左旋一样，就是方向反过来，这里也不再赘述。
下面再分析一下JDK1.8的HashMap中的右旋方法：
```angularjs
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            //l是p的左孩子，pp是p的父亲，lr是l的右孩子
            //判断p和p的左孩子是否为空，做过是，则不进行右旋
            if (p != null && (l = p.left) != null) {
                //如果l的右孩子不为空，则将l的右孩子的父亲设置为p节点
                if ((lr = p.left = l.right) != null)
                    lr.parent = p;
                //如果p的父亲为空，说明p是根节点，右旋后l将变成根节点
                //而红黑树的根节点是黑色的，因此需要将l的颜色设置成黑色
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false;
                //如果p是他的父亲的右孩子，则将l设置成p的父亲的右孩子
                else if (pp.right == p)
                    pp.right = l;
                //否则将l设置成p的父亲的左孩子
                else
                    pp.left = l;
                l.right = p;  //将p设置为l的右孩子
                p.parent = l;  //将l设置成p的父亲
            }
            return root;
        }
```

## 插入
插入一个节点后可能会破坏红黑树的结构，需要通过**修改节点颜色**和**旋转**两种方式来保持平衡。
一般插入的节点默认是红色的，因为插入一个红色的节点一般只会使书的结构违背约束4，
此时我们可以通过上面说的两种方法对树的结构进行调整，使得该树不违背红黑树的所有约束条件。

插入一个红色节点可以分为3种可能的情况：

### 情况1
插入的节点为根节点，此时只需要将节点设置为黑色

### 情况2
插入的节点的父节点为黑色，此时不需要进行任何操作，树依旧满足红黑树的约束条件，下图插入的是值为2的节点
![black-f](https://i.imgur.com/jDdATFh.png)

插入红色的节点不会增加黑色节点的个数，所以不会违反约束5，父亲节点是黑色的，所以不会违反约束4，因此不需要进行任何调整

### 情况3
插入的节点的父节点为红色，此时就有两个连续的红色节点，不满足红黑树的第4个约束条件，需要根据相应的情况对树的结构进行调整

#### 3.1 红叔
父节点为红色，叔节点存在，而且叔节点也是红色，需要进行以下调整：

1)将父节点和叔节点涂黑，将祖父节点涂红

2)将祖父节点设置成当前节点，继续对树的结构进行调整。

![red-u](https://i.imgur.com/U4796ja.png)
将父和叔节点涂黑，祖父节点涂红
![root](https://i.imgur.com/BNDWsJX.png)
将祖父节点设为当前节点后，这里发现它已经是根节点了，因此将它涂黑

在上面的图中要插入的节点值为2，它的父节点为红色，叔节点也为红色，通过上面的操作之后，树的结构又重新满足了红黑树的约束。

#### 3.2 黑叔
父亲节点为红色，叔节点不存在或者为黑色

##### 3.2.1 左右
父亲节点为祖节点的左孩子，当前节点为父节点的右孩子
![lr](https://i.imgur.com/QYf3uzi.png)

需要进行以下调整：

1)以父节点为基准进行左旋

2)设置父节点为当前节点

经过上面的两部操作后，树的结构如下图：
![ll](https://i.imgur.com/CIOHNTt.png)

可以看到树的结构依旧不满足红黑树的约束，但是变成了3.2.2的情况，我们可以继续按照3.2.2的情况对其进行调整

##### 3.2.2 左左
父亲节点为祖节点的左孩子，当前节点为父节点的左孩子

![ll](https://i.imgur.com/jwLvpKx.png)
需要进行以下调整：

1)将父节点涂黑

2)将祖父节点涂红

3)以祖父节点为基准进行右旋

经过上面的调整后，树的结构如下图：
![cmp](https://i.imgur.com/H3X9834.png)

可以看到树的结构已经满足了红黑树的约束

##### 3.2.3 右左
父亲节点为祖节点的右孩子，当前节点为父节点的左孩子
![rl](https://i.imgur.com/7PdyFVf.png)

此时需要进行的操作和3.2.1是对称的

1)以父节点为基准进右旋

2)设置父节点为当前节点

调整后的结构如下图：
![rlt](https://i.imgur.com/kHfDmxg.png)

此时树的结构依旧不满足红黑树的约束，但是变成了和3.2.4一样的情况，因此可以按照3.2.4的情况继续进行调整

##### 3.2.4 右右
![rlt](https://i.imgur.com/kHfDmxg.png)

此时需要进行的操作和3.2.2是对称的

1)将父节点涂黑

2)将祖父节点涂红

3)以祖父节点为基准进行左旋

经过上面的调整后，树的结构如下图：
![cmp](https://i.imgur.com/CDZXGPl.png)

可以看到树的结构已经满足了红黑树的约束

### JDK1.8 balanceInsertion

JDK1.8 HashMap中红黑树按的平衡插入方法（节点实际上已插入，该方法只是调整树的结构使树满足红黑树的约束条件）：
```angularjs
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
            x.red = true;  //插入的节点默认为红色
            //root为根节点，x为插入的节点（当前节点），xp为x的父节点
            //xpp为x的祖父节点，xppl为祖父节点的左孩子，xppr为祖父节点的右孩子
            //这里的循环将不断地设置x的祖先节点为当前节点
            //直到当前节点为根节点或者当前节点的父节点为黑色且没有父节点
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                if ((xp = x.parent) == null) {  //当前节点为根节点？
                    x.red = false;  //将根节点涂成黑色
                    return x;  //返回当前节点（根节点）
                }
                else if (!xp.red || (xpp = xp.parent) == null)  //当前节点的父节点为黑色节点或者根节点？
                    return root;  //父节点为黑色或者父节点为黑色，直接返回根节点
                 //上面的判断使得下面的xp一定是红色的 
                if (xp == (xppl = xpp.left)) {  //父节点为祖父节点的左孩子？
                    if ((xppr = xpp.right) != null && xppr.red) {  //叔节点为红色？
                        xppr.red = false;  //将叔节点涂成黑色
                        xp.red = false;  //将父节点涂成黑色
                        xpp.red = true;  //将祖父节点涂成红色
                        x = xpp;  //将父节点设置为当前节点
                    }
                    else {  //叔节点为黑色或者不存在
                        if (x == xp.right) {  //当前节点是父节点的右孩子
                            root = rotateLeft(root, x = xp);  //以父节点为基点进行左旋
                            //左旋后x引用的对象就变成了xp引用的对象的父亲，xpp的引用的对象的左孩子
                            //变成了x引用的对象，因此下面要纠正它们的引用
                            //将xp的指向x的引用对象的父节点，此时xp可能为空即x引用的对象可能没有父亲
                            //如果xp为空，则xpp也设置为空，否则xpp将指向xp引用的对象的父节点
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {  父节点不为空？
                            xp.red = false;  //将父节点涂成黑色
                            if (xpp != null) {  //祖父节点不为空？
                                xpp.red = true;  //将祖父节点涂成红色
                                root = rotateRight(root, xpp);  //以祖父节点为基准进行右旋
                            }
                            //如果上面的祖父节点为空，说明父节点已经是根节点了，已经涂成了黑色，不需要进行操作了
                            //如果祖父节点不为空，则要以祖父节点为基准进行右旋 
                        }
                    }
                }
                else {  //父节点为祖父节点的右孩子，下面的操作和父节点为祖父节点的左孩子时的操作是对称的，因此不再详细分析
                    if (xppl != null && xppl.red) {  //叔节点为红色？
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {//叔节点不存在或者不为红色
                        if (x == xp.left) {  //当前节点为父节点的左孩子？
                            root = rotateRight(root, x = xp);  //以父节点为基准进行右旋
                            //重新调整xp、xpp的引用指向的对象
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
```

## 删除
删除一个节点同样有可能破坏红黑树的结构
