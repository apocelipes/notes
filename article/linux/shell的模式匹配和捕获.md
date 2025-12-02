## Bash

```text
Bash              Regular Expression
  ?(pattern-list)   (...|...)?
  *(pattern-list)   (...|...)*
  +(pattern-list)   (...|...)+
  @(pattern-list)   (...|...) 严格要求模式只出现一次
  !(pattern-list)   "!" 结果取反
```

pattern用`|`分隔。`*`代表零或多个字符，`?`代表单个字符。可以用在`[[ ==/!= ]]`里。

需要额外开启选项才能支持，`shopt -s extglob`。

例子，列出所有图片：`ls +([[:alnum:]])@(.jpg|.webp)`。

`[[ ~= pattern ]]`进行正则匹配，右边的pattern会被当做posix兼容正则表达式。匹配后`BASH_REMATCH`数组会有所有捕获分组，第0个元素是字符串匹配到整个正则的部分，剩下的元素是分组捕获的结果。

```bash
for img in *
do
    if [[ $img =~ .*\.(png|gif) ]]; then
        echo ${BASH_REMATCH[@]}
    fi
done
```

正则不能加引号，加了引号会被转义成普通字符串，可以用变量（变量内容会被当成正则，不会转义）。不匹配`$?`是1，正则语法错误是2。不支持默认是非贪心匹配，不可修改。

## ZSH

正则匹配语法一样，这是捕获变量变成了`$match`数组：

```sh
string="hello-world-123"
if [[ $string =~ (hello)-(world)-([0-9]+) ]]; then
    echo "Full match: ${match[0]}"
    echo "Group 1: ${match[1]}"
    echo "Group 2: ${match[2]}"
    echo "Group 3: ${match[3]}"
fi
```

模式匹配需要`setopt extended_glob`。

`ls (#i)*.pnG`，大小写不敏感。额外支持`[a-z]`和`[^a-z]`。

- `^pattern`，和`!pattern`效果一样
- `ls run<200-300>`，匹配连续数字范围
- `(pattern-list)`，相当于`@(pattern-list)`，模式必须出现一次
- `(pattern-list|)`，加了一个空的分组，相当于`?(pattern-list)`，模式出现零次或一次
- `(pattern-list)#`，零次或多次
- `(pattern-list)##`，一次或多次
- `pattern(xxx)`，根据文件属性匹配文件，比如`(-)`是普通文件、`(p)`匹配命名管道，`(x)`匹配有执行权限的文件，`(X)`大写X匹配没有执行权限的。

也可以`setopt ksh_glob`改回bash的语法。

`**`递归匹配当前目录和所有子目录，bash在4.0之后可以通过`shopt -s globstar`开启相同功能。
