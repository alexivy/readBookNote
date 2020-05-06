- [编辑器](#编辑器)
  - [生成目录](#生成目录)
  - [显示空格](#显示空格)
- [markdown](#markdown)
  - [换行](#换行)
  - [目录](#目录)
  - [代码](#代码)
  - [表格](#表格)
  - [链接、跳转、图片](#链接跳转图片)
  - [上下标](#上下标)
  - [特殊符号](#特殊符号)
  - [折叠内容](#折叠内容)
  - [其他](#其他)


<p id="jumpflag" ></p>本文简介：这篇是github中编写markdown的一些知识。  

# 编辑器
[ visual studio code ] + extension [ Markdown All in One ]  

可实时预览，可生成适配github的目录（设置Toc: Github Compatibility），便捷使用git。  

## 生成目录  

ctrl + shift + P ==>输入create table of contents即可搜索到  
## 显示空格  

[ visual studio code ]中显示空格，在file=>preference=>setting中设置Render Whitespace为all。  

# markdown  

## 换行  

两个空格+换行。或者两个换行。

## 目录  

格式：  

     - [编辑器](#编辑器)
     - [markdown](#markdown)
      - [代码](#代码)
    

效果：
 - [编辑器](#编辑器)
 - [markdown](#markdown)
   - [代码](#代码)



## 代码
格式：  

    方法1：前有一空行，且行首为四个空格或一个TAB

        code A

        code B
    方法2：前有一空行，```（Tab键上面那个键）开头结尾

    ```
    code C
    ```
    
    
效果：  
方法1：  
行首为一个TAB

    code A  
行首为四个空格

    code B  
方法2：  
\`\`\`开头结尾

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

## 链接、跳转、图片  
格式：  

    网络地址：  
    [微博](http://weibo.com)  
    图片：（一般链接前加!号）  
    ![支持图片](https://img1.doubanio.com/view/photo/s_ratio_poster/public/p2570059839.webp)  
    其他文档位置标题位置：（相对路径）  
    [我的笔记](README.md#My-Note) //中文无法跳转，空格用‘-’代替/  
    对应位置：# MY Note 
    本文档标题位置：  
    [编辑器](#编辑器)  
    对应位置：# 编辑器
    本文档非标题位置： 
    [本文简介](#jumpflag)
    对应位置：<p id="jumpflag"></p>
效果：  
网络地址：  
[微博](http://weibo.com)  
图片：（一般链接前加!号）  
![支持图片](https://img1.doubanio.com/view/photo/s_ratio_poster/public/p2570059839.webp)  
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

## 特殊符号  
格式：  

    转义字符方式：
    \<
    html方式：
    &lt;

效果：

转义字符方式：  
\<  
html方式：  
&lt;  
附：[HTML特殊字符编码对照表](https://www.jb51.net/onlineread/htmlchar.htm)

## 折叠内容  
格式：

    <details>
        <summary>点击查看折叠的内容</summary>
        <!-- 内容与标签间空一行 -->
        支持正文内容

    ```
    支持代码
    ```
    ![支持图片](https://img1.doubanio.com/view/photo/s_ratio_poster/public/p2570059839.webp)
    </details>
效果：

<details>
        <summary>点击查看折叠的内容</summary>
        <!-- 内容与标签间空一行 -->
        支持正文内容  

```
支持代码
```
![支持图片](https://img1.doubanio.com/view/photo/s_ratio_poster/public/p2570059839.webp)
</details>

## 其他  

标签间加一空行可解决部分markdown解析问题。  


持续补充中...  