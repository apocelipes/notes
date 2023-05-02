最近翻开源代码的时候看到了一种很有意思的switch用法，分享一下。

注意这里讨论的不是`typed switch`，也就是case语句后面是类型的那种。

直接看代码：

```golang
func (s *systemd) Status() (Status, error) {
	exitCode, out, err := s.runWithOutput("systemctl", "is-active", s.unitName())
	if exitCode == 0 && err != nil {
		return StatusUnknown, err
	}

	switch {
	case strings.HasPrefix(out, "active"):
		return StatusRunning, nil
	case strings.HasPrefix(out, "inactive"):
		// inactive can also mean its not installed, check unit files
		exitCode, out, err := s.runWithOutput("systemctl", "list-unit-files", "-t", "service", s.unitName())
		if exitCode == 0 && err != nil {
			return StatusUnknown, err
		}
		if strings.Contains(out, s.Name) {
			// unit file exists, installed but not running
			return StatusStopped, nil
		}
		// no unit file
		return StatusUnknown, ErrNotInstalled
	case strings.HasPrefix(out, "activating"):
		return StatusRunning, nil
	case strings.HasPrefix(out, "failed"):
		return StatusUnknown, errors.New("service in failed state")
	default:
		return StatusUnknown, ErrNotInstalled
	}
}
```

你也可以在这找到它：[代码链接](https://github.com/kardianos/service/blob/master/service_systemd_linux.go#L247)

简单解释下这段代码在做什么：调用systemctl命令检查指定的服务的运行状态，具体做法是过滤systemctl的输出然后根据得到的字符串的前缀判断当前的运行状态。

有意思的在于这个switch，首先它后面没有任何表达式；其次在每个case后面都是个函数调用表达式，返回值都是bool类型的。

虽然看起来很怪异，但这段代码肯定没有语法问题，可以编译通过；也没有语义或者逻辑问题，因为人家用的好好的，这个项目接近4000个星星不是大家乱点的。

这里就不卖关子了，直接公布答案：

1. 如果`switch`后面没有任何表达式，那么它等价于这个：`switch true`；
2. case表达式按**从上到下从左到右**的顺序求值；
3. 如果case后面的表达式求出来的值和switch后面的表达式的值一样，那么就进入这个分支，其他case被忽略（除非用了fallthrough，但这会直接跳进下一个case的分支，不会执行下一个case上的表达式）。

那么上面那一串代码就好理解了：

1. 首先是`switch true`，期待有个case能求出true这个值；
2. 从上到下执行`strings.HasPrefix`，如果是false就往下到下一个case，如果是true就进入这个case的分支。

它等价于下面这段：

```golang
func (s *systemd) Status() (Status, error) {
	exitCode, out, err := s.runWithOutput("systemctl", "is-active", s.unitName())
	if exitCode == 0 && err != nil {
		return StatusUnknown, err
	}

    if strings.HasPrefix(out, "active") {
        return StatusRunning, nil
    }
    if strings.HasPrefix(out, "inactive") {
        // inactive can also mean its not installed, check unit files
		exitCode, out, err := s.runWithOutput("systemctl", "list-unit-files", "-t", "service", s.unitName())
		if exitCode == 0 && err != nil {
			return StatusUnknown, err
		}
		if strings.Contains(out, s.Name) {
			// unit file exists, installed but not running
			return StatusStopped, nil
		}
		// no unit file
		return StatusUnknown, ErrNotInstalled
    }
    if strings.HasPrefix(out, "activating") {
		return StatusRunning, nil
    }
    if strings.HasPrefix(out, "failed") {
        return StatusUnknown, errors.New("service in failed state")
    }

	return StatusUnknown, ErrNotInstalled
}
```

可以看到，光从可读性上来说的话两者很难说谁更优秀；两者同样需要注意把常见的情况放在最前面来减少不必要的匹配（这里的switch-case不能像给整数常量时那样直接进行跳转，实际执行和上面给出的if语句是差不多的）。

那么我们再来看看两者的生成代码，通常我不喜欢去研究编译器生成的代码，但这次是个小例外，对于执行流程上很接近的两段代码，编译器会怎么处理呢？

我们做个简化版的例子：

```golang
func status1(cmdOutput string, flag int) int {
    switch {
    case strings.HasPrefix(cmdOutput, "active"):
        return 1
    case strings.HasPrefix(cmdOutput, "inactive"):
        if flag > 0 {
            return 2
        }
        return -1
    case strings.HasPrefix(cmdOutput, "activating"):
        return 1
    case strings.HasPrefix(cmdOutput, "failed"):
        return -1
    default:
        return -2
    }
}

func status2(cmdOutput string, flag int) int {
    if strings.HasPrefix(cmdOutput, "active") {
        return 1
    }
    if strings.HasPrefix(cmdOutput, "inactive") {
        if flag > 0 {
            return 2
        }
        return -1
    }
    if strings.HasPrefix(cmdOutput, "activating") {
        return 1
    }
    if strings.HasPrefix(cmdOutput, "failed") {
        return -1
    }

    return -2
}
```

这是switch版本的汇编：

```assembly
main_status1_pc0:
        TEXT    main.status1(SB), ABIInternal, $40-24
        CMPQ    SP, 16(R14)
        PCDATA  $0, $-2
        JLS     main_status1_pc273
        PCDATA  $0, $-1
        SUBQ    $40, SP
        MOVQ    BP, 32(SP)
        LEAQ    32(SP), BP
        FUNCDATA        $0, gclocals·wgcWObbY2HYnK2SU/U22lA==(SB)
        FUNCDATA        $1, gclocals·J5F+7Qw7O7ve2QcWC7DpeQ==(SB)
        FUNCDATA        $5, main.status1.arginfo1(SB)
        FUNCDATA        $6, main.status1.argliveinfo(SB)
        PCDATA  $3, $1
        MOVQ    CX, main.flag+64(SP)
        MOVQ    AX, main.cmdOutput+48(SP)
        MOVQ    BX, main.cmdOutput+56(SP)
        PCDATA  $3, $-1
        MOVL    $6, DI
        LEAQ    go:string."active"(SB), CX
        PCDATA  $1, $0
        CALL    strings.HasPrefix(SB)
        NOP
        TESTB   AL, AL
        JNE     main_status1_pc258
        MOVQ    main.cmdOutput+48(SP), AX
        MOVQ    main.cmdOutput+56(SP), BX
        LEAQ    go:string."inactive"(SB), CX
        MOVL    $8, DI
        NOP
        CALL    strings.HasPrefix(SB)
        TESTB   AL, AL
        JEQ     main_status1_pc147
        MOVQ    main.flag+64(SP), CX
        TESTQ   CX, CX
        JLE     main_status1_pc130
        MOVL    $2, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status1_pc130:
        MOVQ    $-1, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status1_pc147:
        MOVQ    main.cmdOutput+48(SP), AX
        MOVQ    main.cmdOutput+56(SP), BX
        LEAQ    go:string."activating"(SB), CX
        MOVL    $10, DI
        CALL    strings.HasPrefix(SB)
        TESTB   AL, AL
        JNE     main_status1_pc243
        MOVQ    main.cmdOutput+48(SP), AX
        MOVQ    main.cmdOutput+56(SP), BX
        LEAQ    go:string."failed"(SB), CX
        MOVL    $6, DI
        PCDATA  $1, $1
        CALL    strings.HasPrefix(SB)
        TESTB   AL, AL
        JEQ     main_status1_pc226
        MOVQ    $-1, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status1_pc226:
        MOVQ    $-2, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status1_pc243:
        MOVL    $1, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status1_pc258:
        MOVL    $1, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status1_pc273:
        NOP
        PCDATA  $1, $-1
        PCDATA  $0, $-2
        MOVQ    AX, 8(SP)
        MOVQ    BX, 16(SP)
        MOVQ    CX, 24(SP)
        CALL    runtime.morestack_noctxt(SB)
        MOVQ    8(SP), AX
        MOVQ    16(SP), BX
        MOVQ    24(SP), CX
        PCDATA  $0, $-1
        JMP     main_status1_pc0
```

我把inline给关了，不然hasprefix内联出来的东西会导致整个汇编代码难以阅读。

上面的代码还是很好理解的，“active”和“inactive”的case被放在一起，如果匹配到了就跳转进入对应的分支；“activing”和“failed”的case也放在了一起，匹配到之后的操作与前面两个case一样（实际上上面两个case的匹配执行完就会跳转到这两个，至于为啥要多一次跳转我没深究，可能是为了提高`L1d`的命中率，一大块指令可能会导致缓存里放不下从而付出更新缓存的代价，而有流水线优化的情况下一个jmp带来的开销可能低于缓存未命中的惩罚，不过这在实践里很难测量，权当我在自言自语也行）。最后那一串带ret的语句块就是对应的case的分支。

再来看看if的代码：

```nasm
main_status2_pc0:
        TEXT    main.status2(SB), ABIInternal, $40-24
        CMPQ    SP, 16(R14)
        PCDATA  $0, $-2
        JLS     main_status2_pc273
        PCDATA  $0, $-1
        SUBQ    $40, SP
        MOVQ    BP, 32(SP)
        LEAQ    32(SP), BP
        FUNCDATA        $0, gclocals·wgcWObbY2HYnK2SU/U22lA==(SB)
        FUNCDATA        $1, gclocals·J5F+7Qw7O7ve2QcWC7DpeQ==(SB)
        FUNCDATA        $5, main.status2.arginfo1(SB)
        FUNCDATA        $6, main.status2.argliveinfo(SB)
        PCDATA  $3, $1
        MOVQ    CX, main.flag+64(SP)
        MOVQ    AX, main.cmdOutput+48(SP)
        MOVQ    BX, main.cmdOutput+56(SP)
        PCDATA  $3, $-1
        MOVL    $6, DI
        LEAQ    go:string."active"(SB), CX
        PCDATA  $1, $0
        CALL    strings.HasPrefix(SB)
        NOP
        TESTB   AL, AL
        JNE     main_status2_pc258
        MOVQ    main.cmdOutput+48(SP), AX
        MOVQ    main.cmdOutput+56(SP), BX
        LEAQ    go:string."inactive"(SB), CX
        MOVL    $8, DI
        NOP
        CALL    strings.HasPrefix(SB)
        TESTB   AL, AL
        JEQ     main_status2_pc147
        MOVQ    main.flag+64(SP), CX
        TESTQ   CX, CX
        JLE     main_status2_pc130
        MOVL    $2, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status2_pc130:
        MOVQ    $-1, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status2_pc147:
        MOVQ    main.cmdOutput+48(SP), AX
        MOVQ    main.cmdOutput+56(SP), BX
        LEAQ    go:string."activating"(SB), CX
        MOVL    $10, DI
        CALL    strings.HasPrefix(SB)
        TESTB   AL, AL
        JNE     main_status2_pc243
        MOVQ    main.cmdOutput+48(SP), AX
        MOVQ    main.cmdOutput+56(SP), BX
        LEAQ    go:string."failed"(SB), CX
        MOVL    $6, DI
        PCDATA  $1, $1
        CALL    strings.HasPrefix(SB)
        TESTB   AL, AL
        JEQ     main_status2_pc226
        MOVQ    $-1, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status2_pc226:
        MOVQ    $-2, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status2_pc243:
        MOVL    $1, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status2_pc258:
        MOVL    $1, AX
        MOVQ    32(SP), BP
        ADDQ    $40, SP
        RET
main_status2_pc273:
        NOP
        PCDATA  $1, $-1
        PCDATA  $0, $-2
        MOVQ    AX, 8(SP)
        MOVQ    BX, 16(SP)
        MOVQ    CX, 24(SP)
        CALL    runtime.morestack_noctxt(SB)
        MOVQ    8(SP), AX
        MOVQ    16(SP), BX
        MOVQ    24(SP), CX
        PCDATA  $0, $-1
        JMP     main_status2_pc0
```

除了函数名子不一样之外，其他是一模一样的，可以说两者在生成代码上也没有区别。

你可以在这里看到代码和他们的编译产物：[Compiler Explorer](https://go.godbolt.org/z/9MGMe8T8e)

既然生成代码是一样的，那性能就没必要测量了，因为肯定是一样的。

最后总结一下这种不常用的switch写法，形式如下：

```golang
switch {
case 表达式1: // 如果是true
    do works1
case 表达式2: // 如果是true
    do works2
default:
    都不是true就会到这里
}
```

考虑到在性能上这并没有什么优势，而且对于初次见到这个写法的人可能不能很快理解它的含义，所以这个写法的使用场景我目前能想到的只有一处：

**如果你的数据有固定的2种以上的前缀/后缀/某种模式，因为没法用固定的常量去表示这种情况，那么用case加上一个简单的表达式（函数调用之类的）会比用if更紧凑，也能更好地表达语义，case越多效果越明显。比如我在开头举的那个例子。**

如果你的代码不符合上述情况，那还是老老实实用if会更好。

话说回来，虽然你机会没啥机会写出这种switch语句，但最好还是得看懂，不然下回看见它就只能干瞪眼了。

##### 参考

<https://go.dev/ref/spec#Switch_statements>
