- [编辑器](#编辑器)
- [markdown](#markdown)
  - [目录](#目录)
  - [代码](#代码)
  - [表格](#表格)
  - [链接与跳转](#链接与跳转)
  - [上下标](#上下标)


<p id="jumpflag" ></p>本文简介：这篇是github中编写markdown的一些知识。  

# 编辑器
[ visual studio code ] + extension [ Markdown All in One ]   
可实时预览，可生成适配github的目录，集成git。  
[ visual studio code ]中显示空格，在file=>preference=>setting中设置Render Whitespace为all。  

# markdown  

## 目录  

格式：  

    - [编辑器](#编辑器)
    - [markdown](#markdown)
      - [代码](#代码)

效果：
- [编辑器](#编辑器)
- [markdown](#markdown)
  - [目录](#目录)
  - [代码](#代码)
  - [表格](#表格)
  - [链接与跳转](#链接与跳转)
  - [上下标](#上下标)


## 代码
格式：  

    前有一空行，且行首为四个空格或一个TAB
    或```开头结尾

        code A

        code B
    ```
    code C
    ```
效果：  
行首为一个TAB

    code A  
行首为四个空格

    code B  
（```）开头结尾
```
code C
```
## 表格  
格式：  
```
| 表头1 | 表头2 | 表头3 |  
| :- | :-: | -: |  
|左对齐|居中|右边对齐|  
|1行<br>2行| <br>2行| 1行 |  
```
效果：
| qq | dianhua | id |  
| :- | :-: | -: |  
|左对齐|居中|右边对齐|  
|1行<br>2行| <br>2行| 1行 |  

## 链接与跳转  
格式：  

    网络地址：  
    [微博](http://weibo.com)  
    其他文档位置标题位置：  
    [我的笔记](README.md#My-Note) //中文无法跳转，空格用‘-’代替/  
    对应位置：# MY Note 
    本文档标题位置：  
    [编辑器](#编辑器)  
    对应位置：# 编辑器
    本文档非标题位置： 
    [本文简介](#jumpflag)
    对应位置：<p id='jumpflag'></p>
效果：  
网络地址：  
[微博](http://weibo.com)  
其他文档位置标题位置：  
[我的笔记](README.md#My-Note)  
本文档标题位置：  
[编辑器](#编辑器)  
本文档非标题位置：  
[本文简介](#jumpflag)  

注意：字母大小写可能有影响。

## 上下标  
格式：  

    H<sub>2</sub>O
    Coca Cola<sup>TM</sup>
效果：  
H<sub>2</sub>O  
Coca Cola<sup>TM</sup>  



持续补充中...  