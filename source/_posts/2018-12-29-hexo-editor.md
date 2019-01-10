---
title: Hexo Editor 使用相关
date: 2018-12-29 11:55:27
tags:
- hexo
- next theme
---
<!-- more -->
# HexoEditor基本使用配置
[HexoEdtor主页](https://github.com/zhuzhuyule/HexoEditor)

## 图片本地存储于上传
### 图片快照及上传流程
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2018-12-29-hexo-editor/gif-uploadImage.gif)
### 创建本地存储文件夹
```sh
ln -s /tmp/hexoEditor ~/Documents/HexoEditor/picTmp
```

## Hexo模式
设置 -> Hexo -> Hexo模式
预览时是否展示以下Hexo Header信息
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2018-12-29-hexo-editor/20181229011226937.png)

# shortcut

# ShortCut
| Key                    | Method              | explanation            |
| :--------------------: | :------------------ | :-------------- |
| `Tab`                  | tabAdd              | add indentation        |
| `Shift` - `Tab`        | tabSubtract         | reduce indentation        |
| `Ctrl` - `B`           | toggleBlod          | toggle blod        |
| `Ctrl` - `I`           | toggleItalic        | toggle italic        |
| `Ctrl` - `D`           | toggleDelete        | delete current line        |
| `Ctrl` - <code>\`</code>         | toggleComment       | toggle comment        |
| `Ctrl` - `L`           | toggleUnOrderedList | toggle unordered list    |
| `Ctrl` - `Alt` - `L`   | toggleOrderedList   | toggle ordered list    |
| `Ctrl` - `]`           | toggleHeader        | downgrade title        |
| `Ctrl` - `[`           | toggleUnHeader      | upgrade title        |
| `Ctrl` - `=`           | toggleBlockquote    | add blockquote        |
| `Ctrl` - ` - `         | toggleUnBlockquote  | reduce blockquote        |
| `Ctrl` - `U`           | drawLink            | add hyperlink    |
| `Ctrl` - `Alt` - `U`   | drawImageLink       | add image       |
| `Ctrl` - `T`           | drawTable(row col)  | add table(row column) |
| `Ctrl` - `V`           | pasteOriginContent  | paste origin content       |
| `Shift` - `Ctrl` - `V` | pasteContent        | auto paste content      |
| `Alt` - `F`            | formatTables        | format tables      |
| `Ctrl` - `N`            |         | new md document      |
| `Ctrl` - `H`            |         | new hexo document      |
| `Ctrl` - `O`            |         | open md document      |
| `Ctrl` - `S`            |         | save md document      |
| `Shift` - `Ctrl` - `S`            |         | save as      |
| `Alt` - `Ctrl` - `S`            |         | open settings      |
| `Ctrl` - `W`            |         | toggle write mode      |
| `Ctrl` - `P`            |         | toggle preview mode       |
| `Ctrl` - `R`            |         | toggle read mode       |

* **tip**: In mac OS, plase replace `Ctrl` key with `Cmd` key.
