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
5,数组中的元素不重复  
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
思路二：用一个set存放最大子字符串，用一个变量max存储最大子字符串长度。将字符串中的字符按照顺序放入set，放入前查看是否有重复字符，如果有则剔除set的首字符，如果没有则继续放入，放入后对比max和现在字符串的长度，如果max大则不变，否则取现在字符串的长度。    
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
**条件：**  
1,给定一个字符串  
2,字符串可以以空格，‘-’，‘+’，数字开头否则直接返回0  
3,转换得到的整型如果 大于231 − 1，则返回231 – 1，反之小于−231，返回−231  
**目标：**    
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
思路二：将整数从中间拆分成两半，将后一半反转，比较这两个整数是否相等（需要判断位数是奇数还是偶数）  
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
2,每个数组的索引和值代表一个二维坐标（i, ai）,这个坐标和（i, 0）组成一条直线  
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
## 三数和
**条件：**  
1,给定一个包含n个整型的数组  
**目标：**  
1.找到相加等于0的三个元素，放入到一个三元数组  
2.把所有符合条件的三元数组放入一个数组  
3.三元数组不可重复   
**示例：**  
```
Given array nums = [-1, 0, 1, 2, -1, -4],

A solution set is:
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```
**思路：**  
1,将数组排序  
2,循环每个元素  
3,从剩下的元素中的两头找两数和等于循环元素的负数  
4,排除相等的元素避免重复的结果  
**代码：**  
```
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> res = new LinkedList<>();
        Arrays.sort(nums);
        int len = nums.length;
        for(int i = 0; i < len - 2; i++) {
            //排除相等的元素避免重复的结果
            if (i == 0 || (i > 0 && nums[i] != nums[i - 1])) {
                int low = i + 1, hig = nums.length - 1, sum = -nums[i];
                while (low < hig) {
                    if (nums[low] + nums[hig] == sum) {
                        res.add(Arrays.asList(nums[i], nums[low], nums[hig]));
                        //排除相等的元素避免重复的结果
                        while (low < hig && nums[low] == nums[low+1]) low++;
                        while (hig < hig && nums[hig] == nums[hig-1]) hig--;
                        low++;
                        hig--;
                    } else if (nums[low] + nums[hig] < sum) low++;
                    else hig--;
                }
            }
        }
        return res;
    }
}
```
## 数组去重
**条件：**  
1,给定一个整型数组  
**目标：**  
1,把重复的元素去掉输出  
**示例：**  
```
Input: [1，3，1，-1，0，-1，0]
Output: [1，3，-1，0]
```
**思路：**  
1,一个dest存储返回的目标数组  
2,一个嵌套循环，外循环遍历每一个数组元素，内循环检测该元素是否有重复，重复则剔除，没有重复则将其加入dest  
**代码：**  
```
class Solution {
    public int[] deDuplication(int[] nums) {
        int[] dest = new int[nums.length];
        int index = 0;
        for(int i = 0; i < nums.length; i++){
            for(int j = i + 1; j < nums.length; j++){
                if(nums[i] == nums[j])
                    i++;
            }
            dest[index] = nums[i];
            index++;

        }

        return Arrays.copyOf(dest,index);
    }
}
```
## 电话号码的字母组合
**条件：**  
1,给定一个包含2-9的字符串  
2,每个数字映射多个字母（如下图）  
![映射图](\assets\img\algorithms_2.png)   
**目标：**  
返回这个数字组合对应的所有字母组合  
**示例：**  
```
Input: "23"
Output: ["ad", "ae", "af", "bd", "be", "bf", "cd", "ce", "cf"].
```
**思路：**  
使用回溯法。通过查找所有的潜在的可能性得到全部的答案。如果这个潜在的可能性是错的（或者不是最终的答案）回溯法放弃这个可能性并且更改先前步骤的一些参数，继续尝试。  
**代码：**  
```
class Solution {
    //匿名内部类初始化法
    Map<String, String> phone = new HashMap<String, String>() { {
        put("2", "abc");
        put("3", "def");
        put("4", "ghi");
        put("5", "jkl");
        put("6", "mno");
        put("7", "pqrs");
        put("8", "tuv");
        put("9", "wxyz");
    } };
    
    List<String> output = new ArrayList<String>();
        
    public List<String> letterCombinations(String digits) {
        if(digits.length() != 0)
        backtrack("", digits);
        return output;
    }
    
    public void backtrack(String combination, String next_digits){
        if(next_digits.length() == 0){
            output.add(combination);
        }else{
            String digit = next_digits.substring(0, 1);
            String letters = phone.get(digit);
            for(int i = 0; i < letters.length(); i++){
                String letter = letters.substring(i, i + 1);
                backtrack(combination + letter, next_digits.substring(1));
            }
        }
    }
}
```
## 四数和
**条件：**  
1,给出一个包含n个的整型数组  
2,给出一个目标数组  
**目标：**  
找到四个元素相加等于目标数组的所有可能  
**示例：**
```
Given array nums = [1, 0, -1, 0, -2, 2], and target = 0.

A solution set is:
[
  [-1,  0, 0, 1],
  [-2, -1, 1, 2],
  [-2,  0, 0, 2]
]
```
**思路：**  
将四数和转换为两数和  
**代码：**  
```
class Solution {
    private int len;
    public List<List<Integer>> fourSum(int[] nums, int target) {
        len = nums.length;
        Arrays.sort(nums);
        List<List<Integer>> res = kusm(nums, target, 4, 0);
        return res;
    }

    private ArrayList<List<Integer>> kusm(int[] nums, int target, int k, int index) {
        ArrayList<List<Integer>> res = new ArrayList<List<Integer>>();
        if(index >= len){
            return res;
        }
        //两数和
        if (k == 2) {
            int l = index;
            int r = len - 1;
            while (l < r) {
                //如果两数和等于目标数
                if (target == nums[r] + nums[l]) {
                    List<Integer> list = new ArrayList<>();
                    list.add(nums[l]);
                    list.add(nums[r]);
                    res.add(list);
                    //去重
                    while (l < r && nums[l] == nums[l + 1])
                        l++;
                    while (l < r && nums[r] == nums[r - 1])
                        r--;
                    l++;
                    r--;
                }
                //如果两数和大于目标数
                else if (target < nums[r] + nums[l]) {
                    r--;
                } else {
                    l++;
                }
            }
        }
        //多数和
        if (k > 2) {
            for (int i = index; i < len - k + 1; i++) {
                List<List<Integer>> tmp = kusm(nums, target - nums[i], k - 1, i + 1);

                if (tmp != null) {
                    //将当前的元素添加进每一个list中
                    for (List<Integer> t : tmp) {
                        t.add(0, nums[i]);
                    }
                    res.addAll(tmp);
                }
                //去重
                while (i < len - 1 && nums[i] == nums[i + 1])
                    i++;
            }
        }
        return res;
    }
}
```
## 移除倒数第n的节点
**条件：**  
1,给一个单向链表  
2,给定一个整数n，整数的长度小于链表的长度  
**目标：**  
1,移除倒数第n个节点，返回这个链表  
**示例：**  
```
Given linked list: 1->2->3->4->5, and n = 2.

After removing the second node from the end, the linked list becomes 1->2->3->5.
```
**思路：**  
1,遍历链表找出链表的长度len  
2,将第len-n的节点和len-n+2的节点相连  
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
    public ListNode removeNthFromEnd(ListNode head, int n) {
        ListNode dummy = new ListNode(0);
        dummy.next = head;
        int length = 0;
        ListNode first = head;
        while(first != null){
            length++;
            first = first.next;
            
        }
        
        length -= n;
        first = dummy;
        while(length != 0){
            first = first.next;
            length--;
        }
        first.next = first.next.next;
        return dummy.next;
        
    }
}
```
## 有效的多括号  
**条件：**  
给一个仅仅包含‘(’，‘)’，‘{’，‘}’，‘[’和‘]’,的字符串  
**目标：**  
判断这个字符串的括号组成是否合理  
**示例：**  
```
Input: "()[]{}"  
Output: true
Input: "(]"
Output: false
```
**思路：**  
1,遍历这个字符串，取出每一个字符  
2,用一个栈存放取符合条件的字符（如果取出来的是开阔号，就存进去）  
3,如果是闭括号，判断栈顶是不是对应的开阔号，是就弹栈，不是就直接返回错误  
4,遍历结束如果这个栈大小不为0，返回错误，否则返回正确  
**代码：**  
```
class Solution {
    public boolean isValid(String s) {
        Stack<Character> stack = new Stack<>();
        //map存放映射关系
        Map<Character, Character> map = new HashMap<>();
        map.put(')', '(');
        map.put('}', '{');
        map.put(']', '[');

        char c;
        for(int i = 0; i < s.length(); i++){
            c = s.charAt(i);
            if(map.containsKey(c)){
                char top = stack.isEmpty()? '#': stack.pop();
                if(top != map.get(c))
                    return false;
            }
            else{
                stack.push(c);
            }
        }

        if(stack.isEmpty()){
            return true;
        }else {
            return false;
        }

    }
}
```
## 合并两个有序链表  
**条件：**  
给出两个有序单向链表  
**目标：**  
合并这两个单向链表成一个新的单向链表  
**示例：**  
```
Input: 1->2->4, 1->3->4
Output: 1->1->2->3->4->4
```
**思路：**  
递归处理，如果链表一的头节点大于链表二的头节点，返回链表二，反之返回链表一。并且链表的下一节点继续如上处理，直到两个链表有一个链表为空（或者两个都为空）  
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
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if(l1 == null){
            return l2;
        }
        if(l2 == null){
            return l1;
        }
        if(l1.val > l2.val){
            l2.next = mergeTwoLists(l1, l2.next);
            return l2;
        }else{
            l1.next = mergeTwoLists(l1.next, l2);
            return l1;
        }
    }
}
```
## 生成有效多括号  
**条件：**  
给定n对括号  
**目标：**  
返回有效的多括号组成的集合  
**示例：**  
```
Input: 3
Output:
[
  "((()))",
  "(()())",
  "(())()",
  "()(())",
  "()()()"
]
```
**思路一：**  
递归穷举所有组合  
验证每个组合是否正确  
**代码：**  
```
class Solution {
    public List<String> generateParenthesis(int n) {
        List<String> combinations = new ArrayList();
        generateAll(new char[2 * n], 0, combinations);
        return combinations;
    }

    public void generateAll(char[] current, int pos, List<String> result) {
        if (pos == current.length) {
            if (valid(current))
                result.add(new String(current));
        } else {
            current[pos] = '(';
            generateAll(current, pos+1, result);
            current[pos] = ')';
            generateAll(current, pos+1, result);
        }
    }

    public boolean valid(char[] current) {
        int balance = 0;
        for (char c: current) {
            if (c == '(') balance++;
            else balance--;
            if (balance < 0) return false;
        }
        return (balance == 0);
    }
}
```
**思路二：**   
回溯法，只递归生成正确的组合  
如果开阔号的数量小于总数的一半，可以添加开阔号  
如果必括号的数量小于开阔号的数量，可以添加必括号  
**代码：**  
```
class Solution {
    public List<String> generateParenthesis(int n) {
        ArrayList<String> target = new ArrayList<>();
        backTrack(target, 0, 0, "", n);
        return target;
    }
    private void backTrack(ArrayList<String> target,int open, int close, String s, int max){
        if(s.length() == max * 2){
            target.add(s);
            return;
        }
        
        if(open < max){
            backTrack(target, open+1, close, s + "(", max);
        }
        if(close < open){
            backTrack(target, open, close+1, s + ")", max);
        }
    }
}
```
## 链表按k反转  
**条件：**  
1,给出一个整型链表  
2,给出一个数字K  
3,k的大小小于链表的长度  
**目标：**  
将链表从头开始反转，一次反转k个节点  
**示例：**  
```
Given this linked list: 1->2->3->4->5
For k = 2, you should return: 2->1->4->3->5
For k = 3, you should return: 3->2->1->4->5
```
**思路：**  
每次取前k个节点进行反转。如果节点不够，则不进行反转。    
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
    public ListNode reverseKGroup(ListNode head, int k) {
        int count = 0; //计数
        ListNode curr = head; //当前节点
        while(count != k && curr != null){
            curr = curr.next;
            count++;
        }
        //如果节点个数等于k
        if(count == k){
            curr = reverseKGroup(curr, k);
            while(count-- > 0){
                ListNode temp = head.next;
                head.next = curr; //将头节点变成最后一个节点（和下一组头节点相连）
                curr = head; //将改造后的head赋给curr继续下一次循环
                head = temp;//下一次循环的head    
            }  
            head = curr;
        }
        return head;
    }
}
```
## 有序数组去重  
**条件：**  
给定一个有序的整型数组  
**目标：**  
1,只能改变原有的数组，既不能分配额外的存储空间  
2,返回数组的不重复元素的个数  
3,原数组长度保持不变  
**示例：**  
```
Given nums = [1,1,2],

Your function should return length = 2, with the first two elements of nums being 1 and 2 respectively.

It doesn't matter what you leave beyond the returned length.
```
**思路：**  
1,将数组分为两个部分,不重复部分和重复部分。  
2,用两个指针，第一个指针指向不重复部分的末尾，第二个指针指向最新的不重复元素之后的元素。  
3,如果第二个指针指向的元素和第一个指针指向的元素重复则指向下一个。直到指向的元素和第一个指针指向的元素不重复，则将第二个指针指向的元素赋值给第一个指针指向元素的下一个元素。  
**代码：**  
```
class Solution {
    public int removeDuplicates(int[] nums) {
        int i = 0;
        for(int j = 1; j < nums.length; j++){
//找到右边的第一个不重复元素，将不重复的元素添加到第一个重复元素的右边
            if(nums[i] != nums[j]){
                i++;
                nums[i] = nums[j];
            }
        }
        return i+1; 
    }
}
```
## 串联所有字符串的子串  
**条件：**  
1,给出一个字符串s  
2,给出一个字符串数组  
3,字符串数组的每个元素长度相等    
**目标：**  
1,返回能串联所有字符串数组的子字符串的起始下标    
2,串联的顺序是任意的   
**示例：**  
```
Input:
  s = "barfoothefoobarman",
  words = ["foo","bar"]
Output: [0,9]
Explanation: Substrings starting at index 0 and 9 are "barfoor" and "foobar" respectively.
The output order does not matter, returning [9,0] is fine too.
```
**思路：**  
1,用一个map存储字符串数组，键是每一个word，值是每个word出现的次数  
2,遍历s的每个字符，从起始开始截取字符串，字符串的长度为字符串数组中所有字符的长度。  
3,用另一个map按照word的长度存储截取的字符串，作为key,值为每个key出现的次数  
4,对比第一个map是否有相等的key并且key出现的次数是否不大于第一个map  
5,如果两个map的大小相等，则是我们要寻找的子串。  
**代码：**  
```
class Solution {
    public List<Integer> findSubstring(String s, String[] words) {
         //存储结果
        List<Integer> list = new ArrayList<>();
        
         //存储每个word和word出现的次数
        Map<String, Integer> wordsMap = new HashMap<>();
        int arrayLen = words.length;
        for(int i = 0; i < arrayLen; i++){
            wordsMap.put(words[i], wordsMap.getOrDefault(words[i],0) + 1);
        }
       
        if(words.length == 0){
            return list;
        }
        int wordLen = words[0].length();
        for(int i = 0; i < s.length() - arrayLen * wordLen + 1; i++){
            //存储s中每个word和word存在的次数，然后和wordsMap比较
            Map<String, Integer> sMap = new HashMap<>();
            int j = 0;
            while(j != arrayLen){
                String getSubstring = s.substring(i + j * wordLen, i + j * wordLen + wordLen);

                sMap.put(getSubstring, sMap.getOrDefault(getSubstring, 0) + 1);
                if(wordsMap.containsKey(getSubstring) && wordsMap.get(getSubstring) >= sMap.get(getSubstring)){
                    j++;
                }else{
                    break;
                }
            }
            if(j == arrayLen){
                list.add(i);
            }
        }
        
        return list;
    }
}
```
## 下一个排列
**条件：**  
1,给出一个整型数组  
2,给出的数组能组成一个数字  
**目标：**  
1,将数组重新组合,使得这个重新组合的数组代表的数字大小是所有组合中大于原数组的最小值。  
2,如果这个给定的数组已经代表最大值，则返回它的最小值。  
**示例：**  
```
1,2,3 → 1,3,2
3,2,1 → 1,2,3
1,1,5 → 1,5,1
```
**思路：**  
1,从低位到高位找到相邻两个排序为升序的元素a[i]和a[i+1]  
2,如果找不到证明得顶数组为最大值，将全部元素重小到大排列   
3,将a[i]和右边大于a[i]最小的元素替换  
4,将a[i]右边的元素从小到大排列  
**代码：**  
```
class Solution {
    public void nextPermutation(int[] nums) {
        int len = nums.length;
        int i = len - 2;
        while(i >= 0 && nums[i] >= nums[i + 1]){
            i--;
        }
        //如果存在相邻元素为升序排列
        if(i >= 0){
            int j = len -1;
            //找到第一个大于a[i]的元素并且替换
            while(j >=0 && nums[i] >= nums[j]){
                j--;
            }
            swap(nums, i, j);
        }
        //升序排列a[i]右侧的元素，或者升序排列全部的元素
        reverse(nums, i+1);
    }
    
    private void reverse(int[] nums, int start) {
        int i = start, j = nums.length - 1;
        while(i < j){
            swap(nums, i, j);
            i++;
            j--;
        }
    }
    //替换
    private void swap(int[] nums, int i, int j) {
        int temp = nums[i];
        nums[i] = nums[j];
        nums[j] = temp;
    }
}
```
## 最长有效括号
**条件：**  
给定一个只包含‘（’和’）’的字符串  
**目标：**  
返回最长的有效括号格式的字符子串长度  
**示例：**  
```
Input: ")()())"
Output: 4
Explanation: The longest valid parentheses substring is "()()"
```
**思路：**  
动态规划方案：  
1,初始化一个数组dp，其中dp[i]存储以字符串第i个元素结尾的最长的有效字符子串的长度。  
2,如果字符串第i个元素是’(’，则显然dp[i]=0,因为一个有效的括号字符串是以’)’结尾。  
3,如果字符串第i-1个元素是‘(’，第i个元素是‘)’，则dp[i]=dp[i-2]+2  
4,如果字符串第i-1个元素是‘)’，第i个元素是‘)’。    
①获取字符串以i-1元素结尾的有效字符子串的前一个字符，判断是否是’(’，如果是，则证明该结构为“(以i-1元素结尾的有效字符子串)“，是一个有效字符串，此时dp[i]=dp[i-1]+2  
②获取字符串以i-1元素结尾的有效字符子串的前第二个字符的有效字符子串的长度，再和原来的dp[i]相加，此时dp[i] =dp[i-1]+2+dp[dp[i-1] - 2]    
**代码：**   
```
class Solution {
    public int longestValidParentheses(String s) {
        if("".equals(s)){
            return 0;
        }
        int max = 0;
        int[] dp = new int[s.length()];
        dp[0] = 0;
        for(int i = 1; i < dp.length; i++){
            if(s.charAt(i) == '('){
                dp[i] = 0;
            }else if(s.charAt(i) == ')' && s.charAt(i - 1) == '('){
                //是否是第二个元素
                if(i -2 < 0){
                    dp[i] = 2;
                }else{
                    dp[i] = dp[i - 2] + 2;
                }
            }else{
                if(i - dp[i - 1] > 0 && s.charAt(i - dp[i - 1] - 1) == '('){
                    if((i - dp[i -1]) >=2){
                        dp[i] = dp[i - 1] + 2 + dp[i - dp[i - 1] -2];   
                    }else{
                        dp[i] = dp[i - 1] + 2;
                    }
                }else{
                    dp[i] = 0;
                }
            }
            max = max > dp[i]? max:dp[i];
        }
        return max;
    }
}
```
## 有效的数独
**条件：**  
1,一个有效的9×9字符二维数组  
2,每个元素要嘛是1~9整数，要嘛是’.’  
**目标：**  
验证这个二维数组是不是有效的数独。  
即：每行的数子不能重复，每列的数字不能重复，每个3×三方格不能重复  
**示例：**  
```
Input:
[
  ["5","3",".",".","7",".",".",".","."],
  ["6",".",".","1","9","5",".",".","."],
  [".","9","8",".",".",".",".","6","."],
  ["8",".",".",".","6",".",".",".","3"],
  ["4",".",".","8",".","3",".",".","1"],
  ["7",".",".",".","2",".",".",".","6"],
  [".","6",".",".",".",".","2","8","."],
  [".",".",".","4","1","9",".",".","5"],
  [".",".",".",".","8",".",".","7","9"]
]
Output: true
```
**思路：**  
利用set的数据结构特点（不能插入重复的元素）来将数组的元素依次放入一个set，当插入失败则证明重复。  
**代码：**  
```
class Solution {
    public boolean isValidSudoku(char[][] board) {
        Set valid = new HashSet();
        for(int i = 0; i < 9; i++){
            for(int j = 0; j < 9; j++){
                if(board[i][j] != '.'){
                     //每行，每列，每个三×三格子
                    if(!valid.add("row" + i + ":" +board[i][j]) || 
                       !valid.add("column" + j + ":" +board[i][j]) || 
                       !valid.add("box[" + i / 3 + "][" + j / 3 + "]:" +board[i][j])){
                        return false;
                    }
                }
            }
        }
        return true;
    }
}
```

