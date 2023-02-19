# LeetCode刷题总结
## 按照题目类型分类

## 一. 位计算
位运算除了提升效率，一般还能利用转换成二进制后的一些特性寻找特定元素。

### 异或
1^1 = 0;  
1^0 = 1;  
0^1 = 1;
0^0 = 0;
相关题目：https://leetcode.cn/problems/WGki4K/  

### 找众数
相关题目：https://leetcode.cn/problems/majority-element/

## 二. 字符串

### 基本概念 
26个小写字母的ascii码范围是 97-122  
大写字母ascii范围是 65-90  
java中判断字符是不是数字或者字母：Character.isLetterOrDigit(c)  
字符串大小写转换：String.toLowerCase() / String.toUpperCase()  
相关题目：https://leetcode.cn/problems/valid-palindrome/  

涉及到字母数字统计的，int[] array = new int[26]会经常用到。  

递归打印所有子串：
    
    public static void allStrings(String s, int start, int end) {
        if (start > end) {
            List<String> l = new ArrayList<>(path);
            answer.add(l);
            return;
        }

        for (int len = 1; len <= end-start+1; len++) {
            path.add(s.substring(start, start+len));
            allStrings(s, start+len, end);
            path.remove(path.size() - 1);
        }
    }


### 动态规划
利用回文串的性质可以写出他的动态转移方程。  
dp(i, j)代表子串[i, j]是否是回文串。  
则
    if(j - i <= 2) dp(i, j) = true;
    else dp(i, j) = (value[i] == value[j]) && dp(i+1, j-1)

相关题目：https://leetcode.cn/problems/palindrome-partitioning/solutions/

字符串是否由字典中的数据组成。  
dp(i)表示subString(0, i)是否可以由字典内数据组成。  
则 dp(i) = dp(j) && dict.contains(j, i)
相关题目：https://leetcode.cn/problems/word-break/

### 回溯
判断一个字符串的构成，dfs回溯是经常会用到的。
相关题目：https://leetcode.cn/problems/word-break/  
https://leetcode.cn/problems/word-break-ii/solutions/


### 字典树
特点：
1. 一个空的根节点。
2. 下面每个节点都是一个字母。
3. 每个节点的end代表是否是一个单词的结尾，pass代表有多少单词经过这里(可选)。
4. 节点信息还可以自己根据题目自定义一些。


## 三. 数组
### 动态规划
涉及到提取出符合要求的子数组，动态规划是经常用到的。  
dp(i-1) -> dp(i)

### 数组翻转
这题如果不用额外空间，数组翻转挺难想到的。  
相关题目：https://leetcode.cn/problems/rotate-array/solutions/

## 四. 二叉树
**二叉搜索**  
利用二叉搜索树左小右大的特点。  
相关题目：https://leetcode.cn/problems/search-a-2d-matrix/

