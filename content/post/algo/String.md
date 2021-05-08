---
title: "LC刷题: 字符串专题"
date: 2021-04-08T18:22:18+08:00
hero: /images/posts/leetcode.png
menu:
  sidebar:
    name: 字符串专题
    identifier: leetcode-string
    parent: algo
    weight: 10
draft: false
---



### 14. Longest Common Prefix

返回最长公共前缀子串。[原题](https://leetcode.com/problems/longest-common-prefix/description/)

```
Input: ["flower","flow","flight"]
Output: "fl"
```

**水平扫描**：

![horizontal scanning](https://leetcode.com/media/original_images/14_basic.png)

从头开始遍历整个数组，并且两两比较LCP。如果第i次比较的结果是空，则停止迭代返回空字符串；否则就直到遍历结束:

```java
public String longestCommonPrefix(String[] strs){
  if (str.length == 0) return "";
  String prefix = strs[0];
  for (int i = 1; i < strs.length; i++){
    while (strs[i].indexOf(prefix) != 0){
      prefix = prefix.substring(0, prefix.length() - 1);
      if (prefix.isEmpty()) return "";
    }
  }
  return prefix;
}
```

- 时间复杂度：O(S)
- 空间复杂度：O(1)

**2. 垂直扫描**：

水平扫描有个缺点，就是如果很短的string处于数组末尾，那么还是会进行S次比较，性能上差一点。垂直扫描能解决这个问题，也就是比较同一列(每个字符串的每个字符为一列)：

```java
public String longestCommonPrefix(String[] strs){
	if (strs == null || strs.length == 0) return "";
  for (int i = 0; i < strs[0].length(); i++){
    char c = strs[0].charAt(i);
    for (int j = 1; j < strs.length; j++){
      if(i == strs[j].length() || strs[j].charAt(j) != c){
        return strs[0].substring(0, i);
      }
    }
  }
  return strs[0];
}
```

- 时间复杂度：O(S)
- 空间复杂度：O(1)



### 921.Minimum Add to Make Prentheses Valid

给定只包含小括号的字符串，返回需要多少个`(`或`)`使其成为完整的括号表达式。[原题](https://leetcode.com/problems/minimum-add-to-make-parentheses-valid/)

```
Input: "())"
Output: 1
Input: "()))(("
Output: 4
Input: "()"
Output: 0
Input: "((("
Output: 3
```

题目很简单，注意`)`不能被`(`关闭，所有有两个pointer:

```rust
var minAddToMakeValid = function(str) {
    let left = 0;
    let right = 0;
    for (let i = 0, n = str.length; i < n; i++) {
        if (str.charAt(i) === ')') {
            if (left === 0) {
                right++;
            } else {
                left--;
            }
        } else {
            left++;
        }
    }
    return left + right;
};
```



