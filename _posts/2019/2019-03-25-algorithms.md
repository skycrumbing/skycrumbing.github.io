---
layout: post
title: 算法笔记
tags:
- algorithms
categories: leetcode
description: 算法笔记
---
LeetCode算法笔记，持续更新...(*￣０￣)ノ  

<!-- more -->

## 两数和  
**条件：**  
1,给出一个整型数组nums  
2,给出一个目标整数target  
3,数组中有且仅有两个元素相加的值等于目标整数  
4,不可重复使用一个元素两次  
**目标：**  
返回相加等于目标整数的两个元素索引  
**示例：**  
```
 Input   nums = [2, 7, 11, 15], target = 9
 Output   [0,1]
```
**思路：**  
1,将数组存入hashmap(键为数组的值，值为数组的索引。降低查询的时间复杂度)  
2,从第一个元素开始循环查找是否存在目标元素与给出的元素相加为target  
**代码：**  
```
	class Solution {
		public int[] twoSum(int[] nums, int target) {
			Map<Integer, Integer> map = new HashMap<>();
			for(int i = 0; i < nums.length; i++){
				map.put(nums[i],i);
			}
			for(int i = 0; i < nums.length; i++){
				int half = target - nums[i];
					if(map.containsKey(half) && map.get(half) != i){
						return new int[]{i,map.get(half)};
					}
			}
			throw new IllegalArgumentException("No two sum solution");
		}
	}
```
## 两数相加  
**条件：**  
1,给出两个非空链表代表两个非负整数  
2,两个链表的存放顺序和代表整数的位数相反，如第一个存放代表最低位  
**目标：**  
返回两个链表的和  
**示例：**  
```
 Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
 Output: 7 -> 0 -> 8
 Explanation: 342 + 465 = 807.
```
**思路：**  
1,用一个ListNode来存放两数和  
2,从两个ListNode的头节点开始依次做加法，将和的个位放入存放结果的ListNode子节点。若有一个ListNode的子节点不存在则判定值为0，直到两个链表的子节点都为空。  
3,用一个变量carry存两数和的十位，节点做加法时需要把carry加进去。  
4,链表相加完毕判断是否还需要进位，若carry大于0,则再增加一个子节点存放  
**代码：**  
```
	/**
	 * Definition for singly-linked list.
	 * public class ListNode {
	 *     int val;
	 *     ListNode next;
	 *     ListNode(int x) { val = x; }
	 * }
	 */
	class Solution {
	public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
			int carry = 0;
			ListNode sumListNode = new ListNode(0);
			ListNode temp = sumListNode;
			while(l1 != null || l2 != null){
				int x = (l1 == null? 0:l1.val);
				int y = (l2 == null? 0:l2.val);
				int sum = carry + x + y;
				carry = sum / 10;
				temp.next = new ListNode(sum % 10);
				temp = temp.next;
				if(l1 != null) l1 = l1.next;
				if(l2 != null) l2 = l2.next;
			}
			if(carry > 0){
				temp.next = new ListNode(carry);
			}
			return sumListNode.next;
		}
	}
```
## 最大无重复字符串长度  
**条件：**  
1,给出一个字符串  
**目标：**  
1,找出没有重复字符的最大子字符串的长度  
**示例：**  
```
 Input: "pwwkew"
 Output: 3
 Explanation: The answer is "wke", with the length of 3.
```
**思路：**  
思路一：用一个变量max存储最大子字符串长度，迭代每一种可能的子字符串，验证是否有重复字符，如果没有则查看他的长度是否大于max，大于则替换max  
问题：如果字符串长度过长，这种迭代会占用大量的时间  
思路二：用一个set存放最大子字符串，用一个变量max存储最大子字符串长度。将字符串中的字符按照顺序放入set，放入前查看是否有重复字符，如果有则剔除set的首字符，如果没有则继续放入，放入后max的值加1  
**代码：**  
```
	public class Solution {
		public int lengthOfLongestSubstring(String s) {
			int n = s.length();
			int max = 0, i = 0, j=0;
			Set<Character> set = new HashSet<>();
			while(i < n && j < n){
				if(set.contains(s.charAt(j))){
					set.remove(s.charAt(i++));
				}else{
					set.add(s.charAt(j++));
					max = max > j - i? max : j - i;
				}
			}
			return max;
		}
	}
```
## 最大回文字串  
**条件：**  
1,给出一个字符串  
**目标：**  
2,返回它的最大回文子串  
**示例：**  
```
 Input: "babad"
 Output: "bab"
 Note: "aba" is also a valid answer.
```
**思路：**  
1,回文有两种，奇数（aba）,偶数（aa）  
2，用start和end存放最大回文长度的索引  
3,循环字符串的字符，计算以该字符为中心的左右字符相等的最大长度（包括奇数回文和偶数回文）  
4,如果长度比当前最长的回文长度还长，则将start和end更新  
5,截取这个最大回文子串  
**代码：**  
```
	class Solution {
		public String  longestPalindrome(String s) {
		   if(s == null || s.length() < 1)
					return "";
				int start = 0, end = 0;
				for(int i = 0; i < s.length(); i++){
					//奇数回文
					int len1 = expandFromCenter(s, i, i);
					//偶数回文
					int len2 = expandFromCenter(s, i, i + 1);
					int len = Math.max(len1, len2);
					if(len > end - start + 1){
						start = i - (len - 1) / 2;
						end = i + len / 2;
					}
				}
				return s.substring(start,end + 1);
			}
		private int expandFromCenter(String s, int left, int right) {
		   int L = left, R = right;
			while (L >= 0 && R < s.length() && s.charAt(L) == s.charAt(R)){
				L--;
				R++;
			}
			return R - L - 1;
		}
	}
```
## ”Z”子型转换  
**条件：**  
1,给定一个字符串  
2,给定一个行数（整数）  
**目标：**  
1,将字符串转换为“Z”子型输出  
2,输出的”Z”子型行数取决于给定的行数  
**示例：**  
```
 Input: s = "PAYPALISHIRING", numRows = 4
 Output: "PINALSIGYAHRPI"
 Explanation:

 P     I    N
 A   L S  I G
 Y A   H R
 P     I
```
**思路：**  
1,初始化一个list存放每一行的字符串（list的长度取决于给定的行数和字符串的长度）  
2,按顺序将字符串中的每个字符存入这个list  
3,往上存或者往下存取决于是否在最顶行或者最底行  
4,将list中的字符串全部按顺序拼接成一个字符串  
**代码：**  
```
	class Solution {
		public String convert(String s, int numRows) {
			if(numRows == 1)
				return s;
			List<StringBuffer> bufferList = new ArrayList<>();
			int rows = s.length() < numRows? s.length(): numRows;
			for(int i = 0; i < rows; i ++){
				bufferList.add(new StringBuffer());
			}
			//是否往下
			boolean downRow = false;
			int curRow = 0;
			for(Character c: s.toCharArray()){
				bufferList.get(curRow).append(c);
				if(curRow == 0 || curRow == numRows - 1){
					downRow = !downRow;
				}
				curRow += downRow? 1: -1;
			}

			StringBuffer ret = new StringBuffer();
			for (StringBuffer buffer: bufferList){
				ret.append(buffer);
			}
			return ret.toString();
		}
	}
```



