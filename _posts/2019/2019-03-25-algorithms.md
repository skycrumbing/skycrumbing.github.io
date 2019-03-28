---
layout: post
title: 算法笔记
tags:
- algorithms
categories: leetcode
description: nginx部署个人博客
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
## 最大子字符串长度  
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
## 整数反转  
**条件：**  
1,给一个32位的包含正负的整数  
**目标：**  
1,反转这个整数  
**示例：**  
```
 Input: -120
 Output: -21
```
**思路：**  
1,声明一个变量rev存储反转的这个整数  
2,循环取这个整数的末位（%10），再移除这个整数的末位（/10）  
3,将这个整数的末位放到rev的对应位  
4,注意判断这个反转的整数是否大于32位  
**代码：**  
```
 class Solution {
    public int reverse(int x) {
        int rev = 0;
        while(x != 0){
            int pop = x % 10;
            x = x / 10;       
            if(rev > Integer.MAX_VALUE/10 || (rev == Integer.MAX_VALUE/10 && pop > 7))
                return 0;
            if(rev < Integer.MIN_VALUE/10 || (rev == Integer.MIN_VALUE/10 && pop < -8))
                return 0;
            rev = rev*10 + pop;
        }
        return rev;

    }
 }
```
## 字符串转整型  
## 条件：  
1,给定一个字符串  
2,字符串可以以空格，‘-’，‘+’，数字开头否则直接返回0  
3,转换得到的整型如果 大于231 − 1，则返回231 – 1，反之小于−231，返回−231  
## 目标：  
1,将字符串转为整型  
**示例：**  
```
 Input: "   4193 with words"
 Output: 4193
 Explanation: Conversion stops at digit '3' as the next character is not a   numerical digit.
```
**思路：**  
1,去掉首位空格  
2,用一个字符串变量numString拼接数字  
3,将拼接的字符串转为整型  
**代码：**  
```
 class Solution {
    public int myAtoi(String str) {
           //去首尾空格
        str = str.trim();
       String numString = "";
       if(str.length() == 0){
           return 0;
       }
       boolean flag = false;
       for(int i = 0; i < str.length(); i++) {
           if ((!flag && str.charAt(i) == '-') && i == 0) {
               flag = true;//判断正负，如果是负数超过32位返回值和正数超过32位返回值不同
               numString = numString + '-';
           } else if ((!flag && str.charAt(i) == '+') && i == 0) {
               continue;
           } else if (Character.isDigit(str.charAt(i))) {
               numString = numString + str.charAt(i);
           } else {
               break;
           }
       }

         if("".equals(numString) || "-".equals(numString))
            return 0;

        int target = 0;
        try {
            target = Integer.parseInt(numString);
        }catch (NumberFormatException e){
            if(flag){
                target = Integer.MIN_VALUE;
            }else{
                target = Integer.MAX_VALUE;
            }
        }
        return target;
    }
 }
```
## 回文数字  
**条件：**  
1,给定一个整数  
**目标：**  
1,判断是否是回文数字  
**示例：**  
```
 Input: -121
 Output: false
 Explanation: From left to right, it reads -121. From right to left, it  becomes 121-. Therefore it is not a palindrome.
```
**思路：**  
思路一：将整数反转，再和原数字进行比较（详见整数反转）  
思路二：将整数拆分从中间拆分成两半，比较这两个整数是否相等  
**代码：**  
```
 class Solution {
    public boolean isPalindrome(int x) {
        if (x < 0 || (x % 10 == 0 && x != 0)) {
            return false;
        }
        int rev = 0;
        while(x > rev){
            rev = rev * 10 + x % 10;
            x = x / 10;
        }
        //考虑奇数回文和偶数回文
        if(x == rev || x == rev / 10){
            return true;
        }
        return false;

    }
 }
```

## 正则表达式  
**条件：**  
1,给出一个字符串s和一个表达式p  
**目标：**  
1,实现正则匹配，支持'.'和'\*'  
2,'.'匹配任何单一字符  
3,'\*'匹配0个或多个前一个字符  
## 示例：  
```
 Input:
 s = "ab"
 p = ".*"
 Output: true
 Explanation: ".*" means "zero or more (*) of any character (.)".
```
**思路：**  
1,递归实现  
**代码：**  
```
 class Solution {
    public boolean isMatch(String s, String p) {
        if(p.isEmpty())
            return s.isEmpty();

        boolean first_match = false;
        if(!s.isEmpty() && (s.charAt(0) == p.charAt(0) || p.charAt(0) == '.')){
            first_match = true;
        }
        //如果第二个字符是'*',则有两种情况 1.匹配0个前一个字符。2.匹配多个
        if(p.length() > 1 && p.charAt(1)=='*'){
            return (isMatch(s,p.substring(2)) || (first_match &&isMatch(s.substring(1), p)));
        }else{
            return first_match && isMatch(s.substring(1),p.substring(1));
        }
    }
 }
```
## 围成最大面积  
**条件：**  
1,给定一个非负的数组  
2,每个数组的索引和值代表一个二维坐标（i, ai）,这个坐标和（I, 0）组成一条直线  
**目标：**  
1,找到两条直线，和x轴组成一个方形，她们的面积最大，输出这个面积  
**示例：**  
```
 Input: [1,8,6,2,5,4,8,3,7]
 Output: 49
```  
![结果](\assets\img\algorithms_1.jpg)  
**思路：**  
方案一：循环出每一种可能，找到最大值  
**代码：**  
```
 class Solution {
    public int maxArea(int[] height) {
        int maxArea = 0;
        for(int i = 0; i < height.length; i++){
            for(int j = i + 1; j < height.length; j++){
                maxArea = Math.max(maxArea,Math.min(height[i],height[j]) * (j - i));
            }
        }
        return maxArea; 
    }
 }
```
方案二：面积与两个元素的距离以及两个元素中的较小值大小成正比，因此当两个元素的距离变小了，只要两个元素的较小值变大了就能弥补距离的不足。  
**代码：**  
```
 class Solution {
    public int maxArea(int[] height) {
        int maxArea = 0;
        int l = 0, r = height.length - 1;
        while(l < r){
            maxArea = Math.max(maxArea,Math.min(height[l],height[r]) * (r - l));
            //较大的元素不动
            if(height[l] > height[r]){
                r--;
            }else{
                l++;
            }
        }
        return maxArea;
    }
 }
```
## 数字转罗马   
**条件：**  
1,给定一个数字[1,3999]  
2,罗马数字的表示规则如下：  
罗马数字共有7个，I（1）、V（5）、X（10）、L（50）、C（100）、D（500）和M（1000）。   
在较大的罗马数字的右边记上较小的罗马数字，表示大数字加小数字。  
其中2写做II（I+I）；7写做XII(V+II)；27写作XXVII(XX + V + II)  
在较大的罗马数字的左边记上较小的罗马数字，表示大数字减小数字。  
I在v和X左边代表4和9。  
X在L和C左边代表40和90。  
C在D和M左边代表400和900。  
**目标：**  
1,将给出的数字转换成罗马数字输出  
**示例：**  
```
 Input: 1994
 Output: "MCMXCIV"
 Explanation: M = 1000, CM = 900, XC = 90 and IV = 4.
```
**思路：**  
1,将每一种特殊的罗马数字和对应的数字按照从大到小分别罗列出来保存到数组roms和nums  
2,用一个变量rom存放输出值  
3,循环遍历nums找到小于给定数字的最大值，并将对应的罗马数字添加到rom中  
4,将给定值重新赋值为他们之间的差  
4,继续循环，直到给定值为0退出，返回rom  
**代码：**  
```
 class Solution {
    public String intToRoman(int num) {
        int[] nums = {1000, 900,500, 400, 100,90, 50,40, 10,9, 5, 4,1};
        String[] roms =  {"M","CM","D", "CD","C", "XC","L","XL", "X","IX", "V","IV" ,"I"};
        String rom = "";
        for(int i = 0; i < nums.length;){
            if(num / nums[i] > 0){
                rom = rom + roms[i];
                num = num - nums[i];
            }else{  
                i++;
            }
        }
        return rom;
    }
 }
```
## 罗马转数字  
**条件：**  
1,给定一个罗马数字，范围为[1,3339]  
**目标：**  
1,转换为数字  
示例：
```
 Input: "MCMXCIV"
 Output: 1994
 Explanation: M = 1000, CM = 900, XC = 90 and IV = 4.
```
**思路：**  
1,查看表示大数字减小数字的罗马数字是否存在，存在则减去两倍的大数字减小数字之差。  
2,循环罗马数字字符，将其对应的数字取出相加。  
**代码：**  
```
 class Solution {
    public int romanToInt(String rom) {
        int sum = 0;
        if(rom.indexOf("IV") != -1 || rom.indexOf("IX") != -1 ) sum -= 2;
        if(rom.indexOf("XL") != -1 || rom.indexOf("XC") != -1 ) sum -= 20;
        if(rom.indexOf("CD") != -1 || rom.indexOf("CM") != -1 ) sum -= 200;

        char[] c = rom.toCharArray();
        for(int i= 0; i < c.length ;i++ ){
            if(c[i]=='M') sum+=1000;
            if(c[i]=='D') sum+=500;
            if(c[i]=='C') sum+=100;
            if(c[i]=='L') sum+=50;
            if(c[i]=='X') sum+=10;
            if(c[i]=='V') sum+=5;
            if(c[i]=='I') sum+=1;
        }
        return sum;
    }
 }
```
## 最长公共前缀  
**条件：**  
1,给出一个字符串数组  
**目标：**  
1,找出他们最长的公共前缀，没有则是””  
**示例：**  
```
 Input: ["flower","flow","flight"]
 Output: "fl"
```
**思路：**  
1,找第一个字符串作为公共前缀，看看是否符合  
2,不符合则将去掉作为公共前缀的末尾字符，再进行比较  
3,循环2步骤，直到找到符合的为止  
**代码：**  
```
 class Solution {
    public String longestCommonPrefix(String[] strs) {
      if(strs.length == 0){
           return "";
       }
       String pre = strs[0];
       for(int i = 0; i < strs.length; i++){
           while(strs[i].indexOf(pre) != 0){
               pre = pre.substring(0,pre.length()-1);
           }
       }
        return pre;
    }
 }
```








