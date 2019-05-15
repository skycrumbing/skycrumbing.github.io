---
layout: post
title: 数据结构
tags:
- dataStructure
categories: java
description: java数据结构
---
## java数据结构
主要针对List,Map,Tree进行一些大致的分析

<!-- more -->

## List
ArrayList 优势在于访问速度（即查找get和更新set）以及存储空间（没有指针），**并不包括在尾部插入数据。**  
因为linkedList是双列表。索引该列表中的操作将从头或者尾遍历列表，使用更接近指定索引的那个。所以add(rear)时间复杂度都是O(1)，即常数时间  
## Map 
### 普通map（LinearMap）  
![LinearMap](\assets\img\dataStructure_1.jpg)  
将Entry（存放键和值的对象）放入一个ArrayList，此时要要修改，删除，查找等操作都需要根据KEY找到这个Entry，这个方法如下：  
```
	private Entry findEntry(Object target) {
		for (Entry entry: entries) {
			if (equals(target, entry.getKey())) {
			return entry;
			}
		}
		return null;
	}
	private boolean equals(Object target, Object obj) {
		if (target == null) {
			return obj == null;
		}
		return target.equals(obj);
	}
```
**因为该方法是线性的，所以修改删除，查找等操作的时间复杂度都是线性的O(n)**  
### HashMap  
![HashMap](\assets\img\dataStructure_2.jpg)  
将原始版的LinearMap拆分成多个LinearMap，尽可能保证每个LinearMap只有一个Entry,每个Entry对象用Key采用hash算法生成索引（index）。  
下面是一个辅助方法，用于将Key对应正确的index：  
```
	protected MyLinearMap<K, V> chooseMap(Object key) {
		int index = 0;
		if (key != null) {
			index = Math.abs(key.hashCode()) % maps.size();
		}
		return maps.get(index);
	}
```
当我们使用put和get方法的时候就可以通个这个辅助方法利用数组的方法找到对应的Entry。  
缺点：  
1，无法确保每个子map(LinearMap)只有一个Entry或者Entry个数是均匀的，因为不同的Key对象的hashcode可能相同。  
2，无法保证hashMap的存储顺序是特定的。  
## 二叉搜索树（BFS）
1，如果 node 有一个左子树，左子树中的所有键都必须小于 node 的键。  
2，如果 node 有一个右子树，右子树中的所有键都必须大于 node 的键。  
二叉搜索树的查询逻辑：  
如果要查找的键target小于当前节点，就查找结点的左子树，如果没有则不存在  
如果要查找的键target大于当前节点，就查找结点的右子树，如果没有则不存在  
![Tree](\assets\img\dataStructure_3.png)  
如果你在上图中查找 target = 4  
从根节点开始，它包含键 8 。  
因为 target 小于 8 ，你走了左边。  
因为 target 大于 3 ，你走了右边。  
因为 target 小于 6 ，你走了左边。  
然后你找到你要找的键。  
在这个例子中，即使树包含九个键，它也最多只需要四次比较来找到目标。也就是说，比较的数量与树的高度成正比，而不是树中的键的数量。 满二叉树的高度和数量的关系：  
n = 2^h – 1  
对两边取以 2 为底的对数  
也就是说：log2(n) ≈ h  
**也就是说搜索一个节点的时间正比与logn，是对数而不再是线性O(n)，大大提升了效率**  
### 非满二叉树
对于一个非满二叉树而言，搜索时间会渐渐升高。如图：  
![非满二叉树](\assets\img\dataStructure_4.jpg)  
前者的高度只有4，后者的高度却是15，这使得搜索一个节点的时间复杂度变成了线性。  
**原因**：每次添加一个新的键时，它都大于树中的所有键，所以我们总是选择右子树，并且总是将新节点添加为最右边的节点的右子节点。结果是一个“不平衡”的树，只包含右子节点。  
**解决方案**：自平衡树  
修改 put ，以便它检测树何时开始变得不平衡，如果是，则重新排列节点。具有这种能力的树被称为“自平衡树”。普通的自平衡树包括 AVL 树（“AVL”是发明者的缩写），以及红黑树，这是 Java TreeMap 所使用的。  



