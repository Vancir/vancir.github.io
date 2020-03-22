---
title: Lisa类沙箱分析框架源码分析
date: 2019-11-28 22:52:42
tags:
---

# Lisa 沙盒框架源码分析

Lisa是一个沙盒框架, 通过QEMU来模拟多种CPU架构, 并自动地对恶意软件进行分析. 集成了多种工具来进行分析, 代码并不复杂, 但是可以通过此来了解一个正常对恶意软件的分析流程. 

## 交互设计

Lisa除开提供了web界面外, 还提供了cli工具和交互式的shell, 不过交互式shell其实并没有完成, 但我也有借此了解到如何通过使用`cmd`包来设计一个交互式的工具. 

### 命令行(CLI)

使用 `click`模块来进行参数的指定. 这个模块其实还蛮好用的, 特别是对函数做测试的时候, 能快速地进行输入和检查输出是否符合预期. 大致的模式就是如下:

``` python
import click

@click.group()
def manage():
    """LiSa Manager - to get info about command run:
       lisa <command> --help"""

@manage.command()
@click.option('-o', '--output-file', help='Output file path.')
@click.option('-p', '--pretty', is_flag=True, help='Indented json.')
@click.argument('file', type=click.Path(exists=True))
def run_analysis(output_file, pretty, file):
    pass
  
if __name__ == '__main__':
    manage()
```

### 交互式Shell

使用`cmd`模块可以如此方便地使用一个交互的shell着实让我觉得很惊喜. 不过使用的方法也很简单. 

```python
import cmd
class LisaShell(cmd.Cmd):
  """Interactive shell for QEMU IoT sandbox."""

    intro = ('Welcome to interactive shell of LiSa [Linux Sandbox].\n'
             'Type help or ? to list commands.\n')
    prompt = '(lisa) '
    def do_start_guest(self, arg):
        """Starts guest machine."""
        pass
    def do_exit(self, arg):
        """Quits shell."""
        print('Exiting.')
        return True

if __name__ == '__main__':
    LisaShell().cmdloop()
```

## 全局变量管理

对于一些预定的全局变量, 直接写在`config.py`文件内, 这样对于一些固定的信息是不错的主意, 但是当一些内容是动态获取或者说动态配置时, 这样就是十分笨拙的实现方式了. 

对于日志来说我觉得这十分不错, 作者定义好了一个字典型的logging配置项. 这样能方便地全局输出日志. 因为不这样的话, 其实各个获取的logger它是没法都输出到stdout来的. 

``` python
import logging
logging_config = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'default': {
            'format': ('%(asctime)s %(process)s:%(module)s '
                       '[%(levelname)s] - %(message)s'),
            'datefmt': '%Y-%m-%d %H:%M:%S'
        }
    },
    'handlers': {
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',
            'formatter': 'default',
            'stream': 'ext://sys.stdout'
        }
    },
    'loggers': {
        '': {
            'level': 'DEBUG',
            'handlers': ['console']
        }
    }
}
logging.config.dictConfig(logging_config)
log = logging.getLogger()
```

## 对分析器的抽象化和插件化

既然是恶意软件的分析框架, 分析器是少不了的, 并且需要有一定的扩展性, 而不用总是对现有的分析器进行缝缝补补. 

### 良好的抽象

Lisa实现了一个分析器的抽象类.

``` python 
from abc import ABC, abstractmethod

class AbstractSubAnalyzer(ABC):
    """Abstract base class for sub-analyzers.

    :param file: AnalyzedFile's object.
    """

    def __init__(self, file):
        self._file = file
        self._output = {}

    @abstractmethod
    def run_analysis(self):
        raise NotImplementedError

    @property
    def file(self):
        """Analyzed file."""
        return self._file

    @property
    def output(self):
        """Analysis output."""
        return self._output
```

对于需要抽象的方法设置了装饰器`@abstractmethod`, 并且如果抽象出的新类没有对抽象方法进行重写的话, 会抛出一个`NotImplementedError`, 这样的代码习惯是十分良好的让程序更有健壮性. 而对于需要封装的类成员, 则设置`property`属性来对上层隐藏. 

### 简陋的插件化

Lisa维护了分析器的模块列表, 在进行分析的时候, 就动态加载这些分析器模块进行分析. 不过逻辑虽然简单粗暴, 但也能实现插件的功能. 

``` python
from importlib import import_module
analyzers_config = [
    'lisa.analysis.static_analysis.StaticAnalyzer',
    'lisa.analysis.dynamic_analysis.DynamicAnalyzer',
    'lisa.analysis.network_analysis.NetworkAnalyzer',

    # 'lisa.analysis.virustotal.VirusTotalAnalyzer'

    # custom modules
]
def create_analyzer(analyzer_path, file_path):
    """Imports analyzer class and creates analyzer object

    :param analyzer_path: Path to analyzer in format 'module.Class'.
    :returns: Instantiated analyzer object.
    """
    mod_name, class_name = analyzer_path.rsplit('.', 1)

    analyzer_module = import_module(mod_name)
    analyzer_class = getattr(analyzer_module, class_name)

    return analyzer_class(file_path)
```

通过`import_module`来载入路径指定的模块, 然后获取相应的分析器并返回. 

## 静态分析

使用`radare2`框架提供的Python绑定包`r2pipe`来对恶意软件进行静态分析. 其实这样利用现成的框架来分析是富有好处的, 比如判断文件的架构, Lisa自己通过读取目标文件的文件头字段来判断. 但是这样的工作是很笨拙的. 因为像分析框架它是一次处理把需要的信息都摘取出来, 而自己实现的话, 就需要频繁地重复读取, 开销其实是很大的. 

Lisa将静态分析的功能全部交由`radare2`来完成, 读取的基础信息有这些: `架构`, `字节序`, `文件格式`, `机器平台`,`文件类型`, `文件大小`, `OS平台`, `是否静态链接`, `解释器`, `语言`, `是否strip`, `是否有重定位信息`, `最短操作码长度`, `最长操作码长度`, `入口点`. 而此外还有`导入表`, `导出表`, `依赖库`, `重定位项`, `符号表`, `区段信息`也被收集了. 当然也少不了程序中的`字符串`信息, 不过Lisa是使用`strings`来获取的, 我记得`radare2`也是有集成字符串输出的功能的. 

## 动态分析

动态分析这块, 因为有实现了QEMU虚拟机管理类`QEMUGuest`, 但它的虚拟机是冷启动方式来进行, 而实际上是可以通过快照模式, 所以我自己设计了`QEMU`管理的基类并在此上做设计. 

Lisa会为每一个分析的样本创建一个文件夹, 专门用来存放分析所用的文件以及产出的信息. 因为需要为待测样本选择合适的虚拟机, 因此需要在分析前确定样本的(`架构`, `位数`以及`字节序`) , 才能确保说待测文件能在虚拟机内正常执行. 

Lisa使用了`e2cp`和`e2mkdir`两个工具, 它们都是`ext2tools`工具箱的子件, 它能在无需启动虚拟机的情况下, 直接将待测样本写入到`ext2`的文件系统类. 而`qcow`文件也能通过`guestfs`进行类似的修改. 但是在实际测试中, 它与虚拟机的快照恢复相冲突, 无法兼容热启动. 

来学习一个命令来禁用`ipv6`吧: `echo 1 > /proc/sys/net/ipv6/conf/eth0/disable_ipv6`

还有`tcpdump`抓取网卡流量: `tcpdump -i eth0 -w /stap/capture.pcap &`

然后我们来看`SystemTap`这个内核调试工具以及Lisa所使用的stp模块. 

``` c
probe syscall.*
{
  if (!target_process()) next

  thread_argstr[tid()] = argstr

  if (name in syscalls_nonreturn)
    report_syscall(name, argstr, "")
}

probe syscall.*.return
{
  if (!target_process()) next

  report_syscall(name, thread_argstr[tid()], retstr)
}

function report_syscall(name, argstr, retstr)
{
  printf("SYSCALL\n");
  printf("%s\n", execname());
  printf("%s\n", name);
  printf("%d\n", pid());
  printf("%s\n", argstr);
  printf("%s\n", retstr);
}
```

它会将程序所使用到的系统调用信息都记录下来. 

``` c
probe syscall.{clone,fork}.return
{
  if (first_clone == 0) {
    if (pid() == target()) {
      processes[retval] = pid()
      report_process(retval, pid());
      first_clone = 1
    }
  }

  if (!target_process()) next 
  processes[retval] = pid()
  report_process(retval, pid());
}


function report_process(pid, parent)
{
  printf("PROCESS\n");
  printf("%d:%d\n", pid, parent);
}
```

而对于`clone`和`fork`则会将相应的进程信息也记录下来

``` c
probe syscall.{open,openat} {
  if (!target_process()) next

  report_openfile(filename);
}

function report_openfile(filename)
{
  printf("OPENFILE\n");
  printf("%s\n", filename);
}
```

当然还有对于文件的访问也会记录行为. 因为有重定向输出, 所以这些信息都会保存到`bev.out`文件内.

---

不光是Lisa, 还有另外的两个沙箱框架Limon和Detux也有一些可以借鉴的亮点:

## 内存分析

使用`Volatility`工具来对内存dump进行分析, 使用的命令有如下: 

`linux_psxview`, `linux_pslist`, `linux_pidhashtable`, `linux_pstree`, `linux_psaux`, `linux_psenv`, `linux_threads`, `linux_netstat`, `linux_ifconfig`, `linux_list_raw`, `linux_library_list`, `linux_ldrmodules`, `linux_lsmod`, `linux_check_modules`, `linux_hidden_modules`, `linux_kernel_opened_files`, `linux_check_creds`, `linux_keyboard_notifiers`, `linux_check_tty`, `linux_check_syscall`, `linux_bash`, `linux_check_fop`, `linux_check_afinfo`, `linux_netfilter`, `linux_check_inline_kernel`, `linux_malfind`, `linux_plthook`, `linux_apihooks`

## VMware命令行模式

`/usr/bin/vmrun`是VMWare的命令行工具, 可以借此来管理VMWare虚拟机. 在Limon中使用了这几个命令: `revertToSnapshot`, `start`, `copyFileFromHostToGuest`, `copyFileFromGuestToHost`, `captureScreen`, `suspend`, `stop`, `listDirectoryInGuest`, `listProcessesInGuest`, `runProgramInGuest`, `killProcessInGuest`. 

而获取VMWare虚拟机的内存文件的话, 只需要找到相应的`vmem`文件即可. 

另外还集成了`sysdig`来监视linux的活动, `tcpdump`来抓取流量包(包括dns), 添加/删除`iptables`规则将大量的端口转发出来, wireshark的`tshark`命令行工具来抓包, `INetSim`来模拟网络服务. 



`detux`沙箱也差不多, 主要也是qemu启动然后ssh连接作分析, 不过里面实现的packet parser对流量包的解析, 获取TCP, UDP, ICMP的流量, 获取URL信息和DNS信息, 还是可以参考的. 

## 其他的功能

### YARA规则匹配

``` python
import yara
def yararules(self, rulesfile):
    rules = yara.compile(rulesfile)
    matches = rules.match(self.file)
    return matches
```

### VirusTotal扫描

``` python
import json
import urllib2
import urllib
def virustotal(self, key):
    url = "https://www.virustotal.com/api/get_file_report.json"
    md5 = self.md5
    parameters = {'resource' : md5, "key" : key}
    encoded_parameters = urllib.urlencode(parameters)
    try:
        request = urllib2.Request(url, encoded_parameters)
        response = urllib2.urlopen(request)
        json_obj = response.read()
        json_obj_dict = json.loads(json_obj)
        if json_obj_dict['result'] ==0:
            print "\t  " + "No match found for " + self.md5
        else:
            avresults = json_obj_dict['report'][1]
            return avresults

    except urllib2.URLError as error:
        print "Cannot get results from Virustotal: " + str(error)

```

### MD5散列计算

``` python
import hashlib
def md5sum(self):
    if os.path.exists(self.file):
        f = open(self.file, 'rb')
        m = hashlib.md5(f.read())
        self.md5 = m.hexdigest()
        return self.md5
    else:
        print "No such file or directory:", self.file
        sys.exit()
```

此外还收集了以下信息:

* ssdeep: `ssdeep file`
* ssdeep compare: `ssdeep -m master_ssdeep_file file`
* ascii string: `strings -a file`
* unicode string: `strings -a -el file`
* dependencies: `ldd file`
* elf header: `readelf -h file`
* program header: `readelf -l file`
* section header: `readelf -S file`
* symbols: `readelf -s file`

## Buildroot

Buildroot项目能够帮助快速生成嵌入式设备的Linux镜像, 里面不仅支持大量的Linux架构, 并且也提供了大量适合QEMU模拟的Linux架构版本, 特别是资料极少的Sparc和SH4, 都有相应的支持, 不过我记得有一些限制, 好像是版本太旧了? 还是什么类似的原因, 我没有很好地运用起来. 

但总归是一个极其提高生产效率的项目, 如果以后有需要, 在官方没有提供简单快速方式的情况下, 我肯定会第一时间考虑这个项目. 

## 单元测试

使用PyTest进行单元测试. 它的优点就在于非常方便, 不需要引用额外的包, 只需要保证文件是以`test`开头, 相应的单元测试函数也是以`test`开头即可. 提供了`fixture`可以方便地进行测试. 

## SSH连接

看到的三个沙盒分析框架, 或多或少都会有SSH连接的需求. 可以使用Pexpect来执行ssh命令, 也可以使用Paramiko包的SSHClient和SCPClient专门用于建立ssh连接和传输文件. 但其实我觉得SSH也是一个非常不方便且笨拙的方式, 因为它要求虚拟机能启动好网络, 并且需要设置好网络将22端口转发出来. 就效率而言是非常低的. 