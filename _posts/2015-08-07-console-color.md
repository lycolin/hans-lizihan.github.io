---
layout:     post
title:      console 颜色设定
date:       2015-08-07 23:40
summary:    console 颜色设定
categories: bash
---

今天突发奇想要自己写一个属于自己的command-line tool, 奈何并不熟悉 bash 中颜色的命名规范，没关系，那么来看下这个命名规则。

先上一张图 

![console-tomorrow-night-eighties](/assets/img/tomorrow_night_eighties.png)

这张图片是个典型 这是我 iterm2 主题的配色方案。之前我一直有点含糊这个 terminal 的配色是怎么定的呢？现在终于有点明白了。

简单讲就是 纵坐标表示 `文字颜色` (foreground color) 这个是文字的颜色。

每种颜色都有一个正常版本和一个粗体版本。

e.g. 30m 是正常黑(normal) 1;30m 是轻黑(bright)

## foreground

由 30m 到 37m 分别有这些颜色

ansi | color
:--- | :----
[30m | black
[31m | red
[32m | green
[33m | yellow
[34m | blue
[35m | magenta
[36m | cyan
[37m | white

## background

对等地 40m 到 47m 是这些颜色的 `背景颜色` (background color)

ansi | color
:--- | :----
[40m | black
[41m | red
[42m | green
[43m | yellow
[44m | blue
[45m | magenta
[46m | cyan
[47m | white

## General Text Attributes

0m 到 8m 也有特殊意义

| ansi  | Description                                      |
| :-----|:-------------------------------------------------|
| [0m   | Reset all attributes                             |
| [1m   | Set "bright" attribute                           |
| [2m   | Set "dim" attribute                              |
| [4m   | Set "underscore" (underlined text) attribute     |
| [5m   | Set "blink" attribute                            |
| [7m   | Set "reverse" attribute                          |
| [8m   | Set "hidden" attribute                           |

所以 1;30m 是 `bright` 的意思。 这几个 ansi 在不同的 terminal 中支持程度不一样但是一般 1m 是一定会被支持的，所以这个用得最多。

## defaults

`[39m` 这个是默认的文字颜色

对应地

`[49m` 是默认的背景颜色

## control keys

这些是一些平时不知道的 `control key` 平时光管用了根本没有理。。。

|Name | decimal | octal | hex | C-escape | Ctrl-Key | Description |
|:----| :------ | :---- | :-- | :------- | :------- | :-----------|
|BEL  |7        |007    |0x07 |\a        |^G        |Terminal bell
|BS   |8        |010    |0x08 |\b        |^H        |Backspace
|HT   |9        |011    |0x09 |\t        |^I        |Horizontal TAB
|LF   |10       |012    |0x0A |\n        |^J        |Linefeed (newline)
|VT   |11       |013    |0x0B |\v        |^K        |Vertical TAB
|FF   |12       |014    |0x0C |\f        |^L        |Formfeed (also: New page NP)
|CR   |13       |015    |0x0D |\r        |^M        |Carriage return
|ESC  |27       |033    |0x1B |none      |^[        |Escape character
|DEL  |127      |177    |0x7F |none      |none      |Delete character

其中注意第一个 `echo "\007"` 会弹出常常听到的系统提示音 "bipe"

## 给 terminal 上色。

这个有很多种方案，有在 `bash` script 里面用 `tput` 的方案，也有 hardcode 进 `string` 里面然后 `echo` 或者 `printf` 出来的选择

一般来讲在 包中 中为了简单就直接用hardcode (至少 sf-console 是这么干的，node 的包还暂时没有研究)

最简单的方法就是这样的

``` php
// test
<?php 
echo "It is \033[31mnot\033[39m intelligent to use \033[32mhardcoded ANSI\033[39m codes!"
?>
```

php test -> 带有颜色的输出就出来啦~

注意这个格式: 

``` text
\033[31m
 --  --
  |   |
esc  color-code
```

最后来看下 `sf-console` 的实现(忽略其它细节)

``` php
<?php
private static $availableForegroundColors = [
    'black'   => ['set' => 30, 'unset' => 39],
    'red'     => ['set' => 31, 'unset' => 39],
    'green'   => ['set' => 32, 'unset' => 39],
    'yellow'  => ['set' => 33, 'unset' => 39],
    'blue'    => ['set' => 34, 'unset' => 39],
    'magenta' => ['set' => 35, 'unset' => 39],
    'cyan'    => ['set' => 36, 'unset' => 39],
    'white'   => ['set' => 37, 'unset' => 39],
    'default' => ['set' => 39, 'unset' => 39],
];
private static $availableBackgroundColors = [
    'black'   => ['set' => 40, 'unset' => 49],
    'red'     => ['set' => 41, 'unset' => 49],
    'green'   => ['set' => 42, 'unset' => 49],
    'yellow'  => ['set' => 43, 'unset' => 49],
    'blue'    => ['set' => 44, 'unset' => 49],
    'magenta' => ['set' => 45, 'unset' => 49],
    'cyan'    => ['set' => 46, 'unset' => 49],
    'white'   => ['set' => 47, 'unset' => 49],
    'default' => ['set' => 49, 'unset' => 49],
];

// ...

return sprintf("\033[%sm%s\033[%sm", implode(';', $setCodes), $text, implode(';', $unsetCodes));
?>
```

注意还有这个 中它的 `unset` 一律设为的默认值所以每次高亮完了之后就恢复到之前了~

* [bash-hackers](http://wiki.bash-hackers.org/scripting/terminalcodes) 
* [symfony-console](https://github.com/symfony/Console)
