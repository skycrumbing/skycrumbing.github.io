---
layout: post
title: JAVA TreeMap（红黑树）源码分析
tags:
- dataStructure
categories: java
description: JAVA TreeMap（红黑树）源码分析
---
## TreeMap  
Java的TreeMap是一种自平衡二叉树结构。为了维护二叉树的平衡结构（使得二叉树不会往线性结构发展），在每次put和remove操作时都需要进行结构调整，这样虽然在插入和删除的时候会多花费一点时间，但是有利与数据的查找和修改。  
顾名思义，红色树结构除了维护数据和数据上下层关系外，还维护了每个节点的颜色属性：红或者黑。   

<!-- more -->

## 插入操作  
在进行put操作时，除了往将数据插入到正确的位置外，如果是一个新的key（插入操作而不是更新操作），还要进行结构调整。结构调整遵循四个规则：  
每个节点要么是红色，要么是黑色。  
根节点必须是黑色  
红色节点不能连续（也即是，红色节点的孩子和父亲都不能是红色）。  
对于每个节点，从该点至null（树尾端）的任何路径，都含有相同个数的黑色节点。  
### 左旋和右旋  
在这之前我们先了解一些二叉树调整数据结构的一种方式：左旋和右旋。  
左旋：将右子节点作为该节点进行微调（大的为父）  
```
/**左旋，将右子节点作为该节点进行微调（大的为父）*/
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
        TreeMap.Entry<K,V> r = p.right;

        //p节点的右节点的左节点置为p的右节点
        p.right = r.left;
        //如果p节点的右节点的左节点不为空,将他的父节点置为p
        if (r.left != null)
            r.left.parent = p;

        //p的父节点置为p节点的右节点
        r.parent = p.parent;
        //如果p为根节点，根节点直接被p的右节点替代
        if (p.parent == null)
            root = r;
        //下面两个选择表示无论p是他父节点的左节还是右节点，都将他置为p节点的右节点
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;

        //p节点的右节点的左节点置为p
        r.left = p;

        //p的父节点值为p节点的左节点
        p.parent = r;
    }
}
```
右旋：将左子节点作为该节点进行微调（小的为父，与左旋对称）  
```
/**右旋，将左子节点调整为该节点进行微调（小的为父）*/
private void rotateRight(Entry<K,V> p) {
    if (p != null) {
        TreeMap.Entry<K,V> l = p.left;
        p.left = l.right;
        if (l.right != null) l.right.parent = p;
        l.parent = p.parent;
        if (p.parent == null)
            root = l;
        else if (p.parent.right == p)
            p.parent.right = l;
        else p.parent.left = l;
        l.right = p;
        p.parent = l;
    }
}
```
### 结构调整  
fixAfterInsertion(Entry<K,V> x)是在插入一个节点之后进行的树结构调整（调整颜色和左右旋）。参数x为插入的新节点  
```
/**调整二叉树
 * 原则：
 * 每个节点要么是红色，要么是黑色。
 * 根节点必须是黑色
 * 红色节点不能连续（也即是，红色节点的孩子和父亲都不能是红色）。
 * 对于每个节点，从该点至null（树尾端）的任何路径，都含有相同个数的黑色节点。
 */
private void fixAfterInsertion(Entry<K,V> x) {
    //初始化赋值为红色
    x.color = RED;
    //如果x不为空 && x不为根节点 && x的颜色为红色
    while (x != null && x != root && x.parent.color == RED) {
        //如果x的父节点等于x的父节点的父节点的左节点（即x的父节点是左节点）
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            //取出x的父节点的兄弟右节点y
            TreeMap.Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            //如果y是红色（根据红色节点不能连续得出：1，x的父节点的父节点为黑色 2，存在红色节点连续的情况）
            //调整颜色：将x的父节点，x的父节点的兄弟节点调整为黑色，x的父节点的父节点调整为红色
            //并且往上按如上规则向上调整x节点的父节点的父节点，直到出现x的父节点的父节点颜色为红色（这样就不存在x的父节点的右兄弟为红色了，但是x的父节点为红色，这时候就要通过左旋微调）
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                //如果x是右节点，x的父节点为红色，左旋x的父节点（左旋之后父节点和父节点的右节点变成了红色，继续调微调）
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }
                //将x的父节点调整为黑色，x的父节点的父节点调整为红色，右旋
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
            //如果x的父节点是右节点，反着来
        } else {
            TreeMap.Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    //如果为空或者是根节点置为黑色
    root.color = BLACK;
}
``` 
## 删除操作
TreeMap是如何删除一个节点的呢，在将到删除之前先谈谈什么是后继节点。  
### 后继节点  
删除一个节点t后,最适合继承他的节点(即大于t节点的所有节点中的最小节点)  
如何找到这个节点呢：  
t的右子树不空，则t的后继是其右子树中最小的那个元素。  
t的右孩子为空，则t的后继是其第一个向左走的祖先。  
```
/**
 * 寻找节点后继
 */
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.right != null) {
        TreeMap.Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } else {
        TreeMap.Entry<K,V> p = t.parent;
        TreeMap.Entry<K,V> ch = t;
        while (p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```
### 删除节点  
删除节点的两种情况   
1，删除点p的左右子树都为空，或者只有一棵子树非空。  
办法：直接将p删除（左右子树都为空时），或者用非空子树替代p（只有一棵子树非空时)  
2，删除点p的左右子树都非空。  
办法：，可以用p的后继节点s代替p，然后使用情况1删除多余的s。  
```
/**
 * Delete node p, and then rebalance the tree.
 */
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // 情况二
    if (p.left != null && p.right != null) {
        TreeMap.Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    TreeMap.Entry<K,V> replacement = (p.left != null ? p.left : p.right);
    //情况一
    //如果删除的节点p有且只有一个子节点
    if (replacement != null) {
        // 将p子节点的父节点置为p的父节点
        replacement.parent = p.parent;
        //如果p的父节点为空，将子树节点置为父节点
        if (p.parent == null)
            root = replacement;
        //如果p是左节点，则将p的子树节点挂在p的父节点的左子树上
        //反之挂在p的父节点的右子树上
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        else
            p.parent.right = replacement;

        // 将p的左右，父节点置为空，为了调整红黑色的结构
        p.left = p.right = p.parent = null;

        // 只有删除点是BLACK的时候，才会触发调整函数，因为删除RED节点不会破坏红黑树的任何约束
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // 如果该树仅有一个节点p，且删除了这个节点
        root = null;
    } else { //  如果删除的节点p没有子节点
        if (p.color == BLACK)
            fixAfterDeletion(p);

        if (p.parent != null) {//如果删除的节点p没有子节点
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```
### 结构调整  
同插入操作，在删除操作后也要调整树的结构。  
```
private void fixAfterDeletion(Entry<K,V> x) {
    while (x != root && colorOf(x) == BLACK) {
        //如果是左节点
        if (x == leftOf(parentOf(x))) {
            //获取右兄弟节点
            TreeMap.Entry<K,V> sib = rightOf(parentOf(x));
            //若果右兄弟节点是红色（违反红色节点不能连续的规则）
            if (colorOf(sib) == RED) {
                //右兄弟节点设置为黑色
                setColor(sib, BLACK);
                //父节点设置为红色
                setColor(parentOf(x), RED);
                //左旋微调
                rotateLeft(parentOf(x));
                //微调后再获取右兄弟节点
                sib = rightOf(parentOf(x));
            }
            //如果右兄弟下的左子树和右子树节点是黑色，将右兄弟节点设置为红色，获取父节点
            if (colorOf(leftOf(sib))  == BLACK &&
                    colorOf(rightOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                //如果右兄弟下的左子树和右子树节点颜色不一样，且右兄弟下的右子树的节点是黑色
                if (colorOf(rightOf(sib)) == BLACK) {
                    setColor(leftOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;
            }
        } else { // symmetric 对称的
            TreeMap.Entry<K,V> sib = leftOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }

            if (colorOf(rightOf(sib)) == BLACK &&
                    colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRight(parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```

