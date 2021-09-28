

# 如何撰写一个可以自动更新的Stata命令

前段时间发现Gitee.com的网页源代码更改了，导致原来写的用于安装gitee命令（用于安装gitee上的程序包）无法使用。不得不修改一下gitee命令，然后再发给Baum更新到ssc上。
但考虑到Gitee.com可能以后还会更改网页源代码，gitee命令也得做相应更新。索性就在gitee命令中加入一段代码，用于检测是否有新版本的程序，并自动更新。代码原理很简单，就是对比一下电脑本地的gitee.ado的版本号与服务器上的版本号，如果本地命令版本号低于线上版本就重新安装gitee进行更新。
完成这工作后，觉得有必要把自动更新程序提取出来写一个独立的命令，嵌入我的其他命令，方便用户进行相关命令的更新，及时修正命令存在的bugs。

于是我开始更进一步的思考。gitee命令做的是简单的爬虫工作，本身就是需要跟网络进行连接，并获取正确的内容。如果Gitee.com更改网页源代码，gitee命令也得做对应的修改，要不用户无法使用。所以对于gitee命令强制更新似乎是可以接受的。

但对于其他命令，强制更新会更改用户电脑上的文件，显然不太合适，应该由用户自己决定。所以在检查到新命令的时候，应该告知用户有新版本的命令，让用户选择是否更新。这就涉及到一个Stata与用户进行交互的问题。利用Stata的Dialog programming，我设计了一个简单的弹窗。但检测到有新版本的命令，Stata会弹出如下窗口，用户在Yes前面打钩✔，点Next进入更新；否则跳过更新。

![image-20210928113537976](https://gitee.com/kerrydu/images/raw/master/image-20210928113537976.png)

更进一步地，我希望加入更新代码后对用户的干扰越小越好。如果没有检测到新版本，命令执行原来的功能，用户不会感知到检测更新的存在。如果存在新版本，用户可以选择更新或者不更新，然后程序继续执行原来的功能。要实现这目标，那么程序就需要进行递归。我用如下简单的流程图来说明：

```flow
st=>start: 命令开始
op=>condition: 是否检测更新(是或否?)
op=>operation: 检测更新
cond=>condition: 存在新版本(是或否?)
sub1=>subroutine: 更新命令
io=>inputoutput: 执行命令原有功能
e=>end: 结束命令
st->op->cond
cond(no)->io->e
cond(yes)->sub1(left)->op
```



另外一个问题是Stata用户可能在工作中会多次地使用一个命令，让命令每次调用都进行检测更新没有必要，也浪费时间和资源。对此，我可以用一个global来记录更新检测的状态，如果是已经检测过更新，那么后续用户再使用命令的时候就会跳过检测更新，执行原有的命令。

最后，我将这一更新方法写成了新的命令updatecmd和updatecmd2，可以作为routine用于嵌入用户的开发命令中，实现自动更新。我以下面例子作为说明。demo_updatecmd命令是个demo，其功能只是在Stata控制台显示"hello, your code written here" 。

```
*! version 0.02 
cap program drop demo_updatecmd
program define demo_updatecmd
    version 16   
    di "hello, your code written here" 
end
```

我们为了使得这个命令可以自动更新，需要在di "hello, your code written here" 上面加入以下代码。其中，第7-8行， local pkg和local cmdname分别放入命令所属的程序包名称和命令本身名称。第9-10行提供程序的安装源。global c_m_d_0 存入命令行所输入的内容。第11-18行在线安装updatecmd程序包。第24行用${up_grade_`pkg'}判断是否已经检测过更新，如果没有检测过，则进入检测更新环节。第26行是核心，调用updatecmd进行检测更新，from()和froma()提供了两个线上程序下载源。如果从from( )检测程序失败，则进一步使用froma( )的程序源。这里主要考虑到国内用户可能上不了github，而国外用户可能用不了gitee。第29行之后则可以写原有命令的内容。Stata命令开发者只需第6-29行放入自己的命令中，并更替第7-10行的参数后，并可实现使用这个自动更新routine。

```
*! version 0.02 
cap program drop demo_updatecmd
program define demo_updatecmd
    version 16   
    **********************************************
    *** 需要更改的参数
    local pkg       updatecmd // updatecmd should be replaced with your package name
    local cmdname   updatecmd
    local from      "https://gitee.com/kerrydu/kgitee/raw/master/"
    local froma     "https://github.com/kerrydu/kgitee/raw/master/" //可缺省
    *******Check whether a new version is available
    global c_m_d_0  `0'  
    //install the updatecmd package if it is missing    
    cap which updatecmd
    if _rc{
     cap net install updatecmd,from("https://github.com/kerrydu/kgitee/raw/master/") replace
     if _rc{
     cap net install updatecmd, from("https://gitee.com/kerrydu/kgitee/raw/master/") replace
     if _rc global up_grade_`pkg' "updatecmd_is_missing"
       }
    }
    //the first run of the command defines global up_grade_`pkg'
    local checkcmd 0
    if "${up_grade_`pkg'}"==""{ 
        local checkcmd 1
        updatecmd `cmdname',  from(`from') froma(`froma') pkg(`pkg') 	
    } 
    if `checkcmd' exit
    ********************************************

    di "hello, your code written here" // the content of your command placed here
   
end
```



需要特别说明的是，命令更新的判断依据是ado文件中的命令版本号。为了正确提取版本号，我在updatecmd中做了一些约定，只有version #.####这样的版本号才能被updatecmd识别出来。因此，使用updatecmd进行自动更新，建议在ado文件的首行，写入这样的版本信息`！* version  #, date`, 其中，#必须为一个实数，不能出现字母，也不能是`#.#.#.#`的形式。在与连老师进行交流后，我又增加了updatecmd2命令，可以用于版本号为`#.#.#.#`的形式。提取到错误版本号可能造成一些未知的错误，为了谨慎起见，我仍排除了版本号中有字母的形式。例如`1.223a`是不能被updatecmd和updatecmd2所识别。

最后，局限个人的能力和时间，我这里只是探索性地提供一个思路来实现Stata命令的自动更新，还存在大量的问题没有解决。例如，弹窗还比较简陋，缺少版本号信息和版本更新的功能说明。感兴趣的朋友可以自行探索。相关程序代码在[kgitee: install Stata package issued by Kerry Du in Gitee.com](https://gitee.com/kerrydu/kgitee)。这是个简单的想法，欢迎大家提供改进意见。









