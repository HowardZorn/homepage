+++
title = "搞个Windows的sudo(伪)"
date = 2020-05-01T17:51:34+08:00
draft = true
[[copyright]]
  owner = "贺若舟"
  date = "2021"
  license = "cc-by-nd-4.0"
+++

今天折腾新装好的Windows，需要安装好多环境。包括Python、C/C++、MSYS。现在在Windows上写入`C:\Program Files`需要管理员+UAC高令牌等级。如果是提供安装包的软件的话，直接双击，装着装着自然会叫你授予权限的。但是如果在命令行下面，像pip之类的软件就不会出现请求授权的弹窗而是直接告诉你失败了。一般Linux用户很少遇到这种情况(感谢polkit)，万一遇到了直接输入<del>`fuck`</del>`sudo`+原来的命令就好了。

但是倒霉的Windows用户就不行了。这个系统原来有的runas.exe这种类似sudo的程序因为UAC的存在已经拿不到高层级的特权了(而且本来大多数用户本身用的帐号就是管理员嘛，至于为什么可以看[runas命令在不同环境下的表现](https://blog.walterlv.com/post/start-process-in-a-specific-trust-level))。对于开启了UAC的电脑来说，switch user都是假的，想办法提升UAC令牌等级才是真。

之后我瞄上了Powershell的[Start-Process](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/start-process?view=powershell-7)，发现它的Verb选项的RunAs参数可以提出UAC授权请求。于是我在Powershell的profile里写下了这样的一段代码：

```powershell
function SwitchUser-Do {
    if($args.Length -lt 1) {
        Write-Warning("program must be provided!")
        Write-Output("Usage: sudo program [args...]")
        return
    }
    $program = $args[0]
    $prog_args = $args[1..($args.Count-1)]
    Write-Output("Program: " + $program)
    if ($args.Length -le 1) {
        Start-Process -FilePath $program -Verb RunAs
    }
    else {
        Write-Output("Arguments: " + $prog_args)
        Start-Process -FilePath $program -Verb RunAs -ArgumentList $prog_args
    }
}

Set-Alias sudo SwitchUser-Do
```

写着这个Powershell脚本总感觉不舒服，可能用微软家的东西就是这种感觉吧……

之后是测试

```sh
PS C:\> sudo cmd
```

弹出UAC，之后出现的控制台窗口中带有管理员字样。OK，收工。

> 参考资料：
> 
> MSDN：
> 
> [Start-Process](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/start-process?view=powershell-7)
> 
> 吕毅的博客中关于UAC的一些文章：
> 
> [应用程序清单 Manifest 中各种 UAC 权限级别的含义和效果](https://blog.walterlv.com/post/requested-execution-level-of-application-manifest.html)
> 
> [Windows 中的 UAC 用户账户控制](https://blog.walterlv.com/post/windows-user-account-control.html)
> 
> [在 Windows 系统上降低 UAC 权限运行程序（从管理员权限降权到普通用户权限）](https://blog.walterlv.com/post/start-process-with-lowered-uac-privileges.html)
> 
> [Windows 下使用 runas 命令以指定的权限启动一个进程（非管理员、管理员）](https://blog.walterlv.com/post/start-process-in-a-specific-trust-level)

