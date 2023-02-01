---
title: 二月一日刷题打卡
banner_img: /img/bg.jpg
date: 2023-02-01 23:56:53
tag: 刷题
---

## 力扣2325:解密消息

> 给你字符串 key 和 message ，分别表示一个加密密钥和一段加密消息。解密 message 的步骤如下：
>
> 使用 key 中 26 个英文小写字母第一次出现的顺序作为替换表中的字母 顺序 。
>将替换表与普通英文字母表对齐，形成对照表。
> 按照对照表 替换 message 中的每个字母。
> 空格 ' ' 保持不变。
> 例如，key = "happy boy"（实际的加密密钥会包含字母表中每个字母 至少一次），据此，可以得到部分对照表（'h' -> 'a'、'a' -> 'b'、'p' -> 'c'、'y' -> 'd'、'b' -> 'e'、'o' -> 'f'）。
> 返回解密后的消息。
> 
> 输入：key = "the quick brown fox jumps over the lazy dog", message = "vkbs bs t suepuv"
>输出："this is a secret"
> 解释：对照表如上图所示。
> 提取 "the quick brown fox jumps over the lazy dog" 中每个字母的首次出现可以得到替换表。

今天的题目是一套简单题，不过很久没写代码了，原来想用vector简单解决，后面觉得还是map比较简单，从解答方式倒过来写了一下加入字典的方式，跑代码的时候修改了一下map的key属性，很快就出来了。

第一次出来的答案不太对劲，对比了一下发现是空字符串没有加到字典导致字母错了一位，把这个问题修复了就得到答案了。

```c++
class Solution {
public:
    string decodeMessage(string key, string message) {
        unordered_map<char,string> map;
        int flag = 0;
        map[' '] = " ";
        for (auto c:key){
            if(map.count(c) == 0){
                map[c] = flag + 'a';
                flag ++;
            }
        }
        string ret = "";
        for(auto c:message){
            ret += map[c];
        }
        return ret;
    }
};
```

写完了觉得太快了，顺便看了一下python的题解，也是用了字典的方式解决，学习了一下基本算背了出来。以后也要稍微熟悉一下python的刷题方法，最好的条件是两种语言都能用上。

```python
class Solution(object):
    def decodeMessage(self, key, message):
        s = "abcdefghijklmnopqrstuvwxyz"  
        d = {" ":" "}
        i = 0
        for c in key:
            if c not in d:
                d[c] = s[i]
                i += 1

        return "".join(d[c] for c in message)
```

今天的学习到此结束。
