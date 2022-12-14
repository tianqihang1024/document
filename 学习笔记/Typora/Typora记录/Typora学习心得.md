### 1.标题

| 快捷键                 | 操作         | 备注                   |
| :--------------------- | :----------- | :--------------------- |
| `Ctrl + 0`             | 普通文本     |                        |
| `Ctrl + 1 — Ctrl + 6`  | 一到六级标题 |                        |
| `> / Ctrl + Shift + Q` | 引用         |                        |
| `--------`             | 分割线       |                        |
| `Ctrl + T`             | 表格         | 可拖拽，可网页表格赋值 |
| `[^1]/[^1]:`           | 脚标/脚注    |                        |

### 2.文本

| 快捷键            | 操作     | 备注         |
| :---------------- | :------- | :----------- |
| `Ctrl + B`        | **加粗** |              |
| `Ctrl + I`        | *斜体*   |              |
| `Ctrl + U`        | 下划线   |              |
| `==高亮==`        | 高亮     | 偏好设置选中 |
| `shift + Alt + 5` | 删除线   |              |

### 3.代码

| 快捷键             | 操作       | 备注               |
| :----------------- | :--------- | :----------------- |
| `~~~`              | 代码块     |                    |
| `shift + Ctrl + ~` | 行内代码块 |                    |
| `flow/mermaid`     | 流程图     | 详见流程图文件源码 |

### 4.列表

| 快捷键              | 操作                          | 备注 |
| :------------------ | :---------------------------- | :--- |
| `- / * / +`         | 无序；TAB 下一级；回车 上一级 |      |
| `数字 + . + spance` | 有序                          |      |
| `- + spance + [ ]`  | 任务列表                      |      |

### 5.特殊

| 快捷键             | 操作                                                         | 备注                    |
| :----------------- | :----------------------------------------------------------- | :---------------------- |
| `Ctrl + Shift + I` | 图片                                                         |                         |
| `：up`             | ⬆️`emoji`图标                                                 |                         |
| `<!--注释-->`      | 与`html`注释一样                                             | 转为`pdf`等格式时不显示 |
| `Ctrl + K`         | 超链接/[百度](https://www.cnblogs.com/LDBKY/p/www.baidu.com) |                         |
| `X^6^`             | 上标6                                                        | 偏好设置选中            |
| `H~2~O`            | 下标2                                                        | 偏好设置选中            |
| `Ctrl + Shift + M` | 公式块                                                       | 偏好设置选中            |
| `$内联公式$`       | $e$2                                                         | 偏好设置选中            |



> 设置高亮

\1. 打开 文件 > 偏好设置 > 高级设置 > conf.user.json

\2. 加入代码: "Highlight": "Ctrl + q" 在 "keyBinding": 后面的大括号里面