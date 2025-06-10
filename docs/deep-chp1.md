<style>
.tab{
    overflow:hidden;
}
.tab button{
    background-color:inherit;
    float:left;
    border:none;
    outline:none;
    cursor:pointer;
    padding:14px 16px;
}
.tab button:hover{
    background-color:#ddd;
}
.tab button.active{
    background-color:#ccc;
}
</style>
<script>
function openTab(evt,tabid,trg){
    var buttonName=["button-cpp","button-py"];
    var tabName=["tab-cpp","tab-py"];
    var i;
    var tablist=document.getElementsByClassName("tabcontent");
    for(i=0;i<tablist.length;i++){
        tablist[i].style.display="none";
    }
    tablist=document.getElementsByClassName(buttonName[Number(!tabid)]);
    for(i=0;i<tablist.length;i++){
        tablist[i].className=tablist[i].className.replace(" active","");
    }
    tablist=document.getElementsByClassName(buttonName[Number(tabid)]);
    for(i=0;i<tablist.length;i++){
        if(!tablist[i].className.includes("active")){
            tablist[i].className+=" active";
        }
    }
    tablist=document.getElementsByClassName(tabName[Number(tabid)]);
    for(i=0;i<tablist.length;i++){
        tablist[i].style.display="block";
    }
}
</script>

# Chapter 1 算号代码解析与优化

## 1.1 完整的算号流程

算号代码可在[教程附录](res-namecalc-code.md)中获取，或使用玩家【sqrt2802】改写的 python 版本 [sqrtools](https://github.com/sqrt2802/sqrtools)。本节将讲述完整的算号过程，并截取一些对应的代码片段供读者参考。阅读时可以尝试回顾进阶教程第 8 章讲述的内容，前面给出的算号规则/结论都能在代码上体现出来。

### 1.1.1 名字信息的数据结构

算号代码中的“号”一般以结构体或 class 的形式存储。一个号有三个特征性编码序列，在代码中体现为三个数组：

- val，长度为 256；
- namebase 和 namebonus（本节中均省略下划线），长度均为 128。

需要注意的是，这三个数组均为（C/C++ 中的）unsigned char 类型，每项取值范围为 0~255，当某项被赋予一个超过上限的值时，会自动对 256 取模。后续的计算处理中也经常用到这一类型的临时变量，如果你使用其他语言或 int 类型储存它们，需要在每次赋值时手动取模一次。

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">unsigned char val[256],namebase[128],namebonus[128];</code><code class="language-text tabcontent tab-py">self.__val=[]
self.namebase:int=[0]*128
self.namebonus:int=[0]*128</code></pre>

### 1.1.2 val 与字符串的载入

val 数组的初始状态是 0~255 的自然数列。

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">for(int i=0;i<256;++i)
    val[i]=i;</code><code class="language-text tabcontent tab-py">self.__val=list(range(256))</code></pre>

在加载名字字符串时，会根据字符串内容按一定规则对 val 进行打乱，不同字符串对应不同的打乱方式。因此，所有号 val 的数字组成都是相同的，但顺序各异。

输入的名字字符串首先会被在 @ 符号处切断，分成名字主体与战队名两部分。无战队号的“战队名”和名字本身相同，例如 `1` 和 `1@1` 是等效的。

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">// 注意此处没有考虑无战队名的情况
for(int i=0;i<256;++i)
    name[i]=team[i]=0;
namelen=1,teamlen=1,bonuslen=0;
for(int i=0,f=0;namein[i];++i){
    if(namein[i]=='@')
        f=1;
    else if(!f)
        name[namelen++]=namein[i];
    else
        team[teamlen++]=namein[i];
}</code><code class="language-text tabcontent tab-py">namein=list(namein.rpartition('@'))
if namein[1]=='@':
    if namein[2]=='':
        namein[2]=namein[0]
else:
    namein[0]=namein[2]</code></pre>

名字主体和战队名会分别按 UTF-8 编码拆成单个字节组成的序列，然后将每个字节的十六进制数码转换为十进制。在 C/C++ 中你可以直接以数字形式按字节读取字符串，但对于默认编码不是 UTF-8 的系统（例如简体中文 Windows），需要进行一些编码转换处理。

需要注意的是，在完成转换之后，序列的最前面会被补一个 0。补上前导零的序列长度数值会在后续步骤中用到，我们将其存储在临时变量 `namelen` 和 `teamlen` 中。这一“序列长度”与肉眼可见的“有几个字”并不相同，一些在 UTF-8 编码中占据多个字节的字符（常见的有汉字、小语种文字、特殊符号、emoji 等）会在序列中变为多个数。我们常说“单个汉字的长度是 2”，意思就是常见汉字在 UTF-8 编码中占据两个字节，多一个汉字会使序列长度 +2。

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">//（教程附录中的代码采取了读入时自动分流并计数的措施，见上一段代码）</code><code class="language-text tabcontent tab-py">namestr=list(namein[0].encode())
teamstr=list(namein[2].encode())
namestr.insert(0,0)
teamstr.insert(0,0)
namelen=len(namestr)
teamlen=len(teamstr)</code></pre>

val 的打乱通过多次交换的方式实现，代码如下：

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">unsigned char s;
for(int i=s=0;i<256;++i){
    s+=team[i%teamlen]+val[i];
    std::swap(val[i],val[s]);
}
for(int i=0;i<2;++i){
    for(int j=s=0;j<256;++j){
        s+=name[j%namelen]+val[j];
        std::swap(val[j],val[s]);
    }
}</code><code class="language-text tabcontent tab-py">s=0
for i in range(256):
    s+=(teamstr[i%teamlen]+self.__val[i])
    s%=256
    self.__val[i],self.__val[s]=self.__val[s],self.__val[i]</code></pre>

从上面的代码可以看出，val 会先按战队名序列打乱一次，再按名字主体序列打乱两次，这三次操作的规则是基本相同的。

### 1.1.3 namebase/bonus 和八围属性生成

namebase 是由 val 生成的。val 序列中特定哈希值满足条件的 128 项会按顺序转化为 namebase：

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">for(int i=0;i<256;++i){
    unsigned char m=val[i]*181+160;
    if(m>=89&&m<217)
        namebase[bonuslen++]=m&63;
}</code><code class="language-text tabcontent tab-py">s=0
for i in range(256):
    m=(self.__val[i]*181+160)%256
    if m>=89 and m<217:
        self.namebase[s]=m&63
        s+=1</code></pre>

由于 val 的数字组成恒定，满足条件的 128 项和它们转化后的结果也是确定的。因此，所有号 namebase 的数字组成都是 0~63 各两个，但顺序各异。

val 和 namebase 都在上述计算完成后固定下来，不受后续数值计算、组队加成等步骤的影响。

namebonus 的初始状态是一份 namebase 的拷贝：

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">memcpy(namebonus,namebase,sizeof(namebase));</code><code class="language-text tabcontent tab-py">self.namebonus[:]=self.namebase[:]</code></pre>

namebonus 不受数值计算的影响，但会在组队加成时改变。号的数值实际上由 namebonus 决定，但在单号计算中 namebonus 与 namebase 等价，因此通常说的“单号数值取决于 namebase” 是没有问题的。

namebonus 的前 32 项是八围属性决定区间，从前到后分别对应 HP、攻、防、速、敏、魔、抗、智。计算这些属性的方式已在前文中提及，此处不再赘述。

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">// 注意此处七围属性没有 +36
int propcnt=0;
unsigned char r[32];
memcpy(r,namebonus,sizeof(r));
for(int i=10;i<31;i+=3){
    std::sort(r+i,r+i+3);
    propbonus[propcnt++]=r[i+1];
}
std::sort(r,r+10);
propbonus[propcnt++]=154;
for(int i=3;i<7;++i)
    propbonus[propcnt-1]+=r[i];</code><code class="language-text tabcontent tab-py">propcnt=1
if usebonus==True:
    r=self.namebonus[0:32]
else:
    r=self.namebase[0:32]
for i in range(10,31,3):
    r[i:i+3]=sorted(r[i:i+3])
    self.nameprop[propcnt]=r[i+1]
    propcnt+=1
r[0:10]=sorted(r[0:10])
self.nameprop[0]=154
for i in range(3,7):
    self.nameprop[0]+=r[i]
for i in range(1,8):
    self.nameprop[i]+=36</code></pre>

可以看到代码中为了避免影响 namebonus，使用了先拷贝属性决定区间再排序的方式。

### 1.1.4 技能的生成

虽然从结果上看号的 16 个技能座位上种类与熟练度一一对应绑定，但在代码层面上二者是分别独立生成的，几乎互不影响。

每个号具有一个决定技能种类的 id 序列（变量名一般简写为 sklid 等）和一个决定技能熟练度的 freq 序列，二者均为 int 数组，每项对应一种技能。

id 数组的初始状态是 0~39 的 40 项自然数列，生成技能时根据 val 序列按特定规则对其进行打乱：

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">Rand rand(0,0,val);
for(int i=0;i<40;++i)
    nameskill[i].init(i);
for(int s=0,i=0;i<2;++i){
    for(int j=0;j<40;++j){
        s=(s+rand(40)+nameskill[j].id)%40;
        swap(nameskill[j].id,nameskill[s].id);
    }
}</code><code class="language-text tabcontent tab-py">self.__sklid=list(range(0,40))
s=0
for i in range(2):
    for j in range(40):
        s=(s+randgen()+self.__sklid[j])%40
        self.__sklid[j],self.__sklid[s]=self.__sklid[s],self.__sklid[j]</code></pre>

由于这一计算过程需要额外打乱 val，为了防止影响原 val 序列，我们采取先拷贝再操作副本的方法。

虽然 id 序列有 40 项，但只有前 16 项会按顺序填入最终的“技能座位”中。显然，只受 val 影响的 id 序列不会在组队加成中改变，因此不在 16 个技能座位中的技能是无法通过加成学会的。

技能 id 与种类之间的对照关系如下：

id|0|1|2|3|4|5|6|7|8|9
:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:
技能|火球|冰冻|雷击|地裂|吸血|投毒|连击|会心|瘟疫|命轮
**id**|**10**|**11**|**12**|**13**|**14**|**15**|**16**|**17**|**18**|**19**
技能|狂暴|魅惑|加速|减速|诅咒|治愈|苏生|净化|铁壁|蓄力
**id**|**20**|**21**|**22**|**23**|**24**|**25**|**26**|**27**|**28**|**29**
技能|聚气|潜行|血祭|分身|幻术|防御|守护|反弹|护符|护盾
**id**|**30**|**31**|**32**|**33**|**34**|**35**|**36**|**37**|**38**|**39**
技能|反击|吞噬|亡灵|垂死|隐匿|(空技能)|(空技能)|(空技能)|(空技能)|(空技能)

namebonus 的第 65~128 项是技能熟练度决定区间。freq 数组只有 16 项，生成过程如下：

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python (sqrtools 3.2.0)</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">int last=-1;
for(int i=0,j=0;i<64;i+=4,++j){
    unsigned char p=min({a[i],a[i+1],a[i+2],a[i+3]}),q=min({b[i],b[i+1],b[i+2],b[i+3]});
    if(p>10){
        if(nameskill[j].id<35)
            nameskill[j].freq=p-10;
        if(q<=10)
            nameskill[j].e=true;
        else if(nameskill[j].id<25)
            last=j;
    }
}
if(last!=-1){
    nameskill[last].e=true;
    nameskill[last].freq*=2;
}
unsigned char u;
u=nameskill[14].freq;
if(u>0&&(!nameskill[14].e)){
    nameskill[14].freq+=min({namebonus[60],namebonus[61],u});
    nameskill[14].e=true;
}
u=nameskill[15].freq;
if(u>0&&(!nameskill[15].e)){
    nameskill[15].freq+=min({namebonus[62],namebonus[63],u});
    nameskill[15].e=true;
}</code><code class="language-text tabcontent tab-py">self.__sklfreq=[0]*16
self.__sklflag=[True]*16
last=-1
j=0
for i in range(64,128,4):
    q=min(self.namebase[i],self.namebase[i+1],self.namebase[i+2],self.namebase[i+3])
    if usebonus==True:
        p=min(self.namebonus[i],self.namebonus[i+1],self.namebonus[i+2],self.namebonus[i+3])
    else:
        p=q
    if p>10:
        if self.__sklid[j]<35:
            self.__sklfreq[j]=p-10
        if q<=10:
            self.__sklflag[j]=False
        elif self.__sklid[j]<25:
            last=j
    j+=1
if last!=-1:
    self.__sklflag[last]=False
    self.__sklfreq[last]*=2
if usebonus==True:
    info=self.namebonus
else:
    info=self.namebase
if self.__sklfreq[14]>0 and self.__sklflag[14]:
    self.__sklfreq[14]+=min(info[60],info[61],self.__sklfreq[14])
    self.__sklflag[14]=False
if self.__sklfreq[15]>0 and self.__sklflag[15]:
    self.__sklfreq[15]+=min(info[62],info[63],self.__sklfreq[15])
    self.__sklflag[15]=False</code></pre>

来自 namebase 的临时变量 q 的作用是检验熟练度数值的来源，p>10 而 q≤10 的情况表明这一技能是通过组队加成获得的（加成前其 id 已在技能座位中，但其熟练度为零），此类技能会被打上 flag 标记并跳过后续的末尾熟练度加成环节。

“空技能”的熟练度会被强制设为 0，一般通过将整个 freq 数组置零初始化并在生成中跳过空技能位填数的方式来实现这一点。

### 1.1.5 组队加成

对于任意**队名相同**的多人组，加成时会两两进行 namebase 扫描操作。在进阶教程第 8 章介绍的加成方式中，组队加成会对 namebase 直接取 max。这种说法只是方便理解的简化版本，实际上并不准确。真正的加成过程在识别到符合要求的 namebase 位点后会在 namebonus 的对应位置上进行取 max 操作：

<div class="tab">
<button class="button-cpp" onclick="openTab(event,false)">C++</button>
<button class="button-py" onclick="openTab(event,true)">Python</button>
</div>
<pre><code class="language-text tabcontent tab-cpp">//（C++ 代码暂缺）</code><code class="language-text tabcontent tab-py">def calcbonus(target,addon):
    for i in range(7,128):
        if addon[i-1]==target.namebase[i]:
            target.namebonus[i]=max(target.namebonus[i],addon[i])
    return
for i in range(len(name)):
    for j in range(i+1,len(name)):
        calcbonus(name[i],name[j].namebase)
        calcbonus(name[j],name[i].namebase)</code></pre>

由于扫描的序列始终是不变的 namebase，某一次扫描产生的加成不会增加新的识别位点（否则加成就太多了）。

## 1.2 测号器卡常加速简介

前一节提到的算号代码虽然能正常工作但速度较慢，与现行测号器之间有很大差距。
实际的测号器使用了一系列代码优化手段提速，借用算法研究中“卡常数”的称呼，我们称这些优化过程为“卡常加速”。

本节会介绍一些常见的测号器卡常思路（实际上是帮助你读测号器源码的），但不能覆盖现有的全部优化手段，我们也并不鼓励玩家自制测号器（因为你重新造出来的轮子未必跑得更快）。

### 1.2.1 字符串滚动加速

算号的起点是名字字符串，而在测号器中字符串是按照设定格式批量生成的。如果每算一次号都重新写入一遍完整的名字，显然会大大减慢测速。由于测号格式中总有固定部分，我们可以用以下方法减少重复工作：

**预加载战队名**<br>
这是最常见的加速技巧。战队名会先独立载入 val 中，把按战队名打乱的 val 缓存下来，每次算号时在它的基础上直接加载名字主体，就能显著提速。需要注意的是，名字主体的载入需要打乱两次，因此前缀不能被预加载。

**逐字替换更新名字**<br>
在滚动更新字符串时，可以不改变前/后缀，直接对可变部分进行单个字的替换。这一方法在顺序模式测号时尤其有用，只需累加末位并按需进位即可。

### 1.2.2 算号算法优化

顾名思义。一些常见的优化有：

**namebase 提前截断**<br>
测号先筛八围，而 namebase 的八围决定区间恰好在开头部分。在算号时 namebase 先只填前 32 位，算出八围后若过筛则填满剩余部分继续处理，若不过筛则直接丢弃。

**属性剪枝**<br>
在测号器中你可能会看到这样的代码：
```cpp
V = 0;
#define median(x, y, z) x<y?(x<z?(y<z?y:z):x):(y<z?(x<z?x:z):y)
V += median(name_base[28], name_base[29], name_base[30]);
if (V < 24) return;
V += median(name_base[13], name_base[14], name_base[15]);
V += median(name_base[16], name_base[17], name_base[18]);
V += median(name_base[25], name_base[26], name_base[27]);
if (V < 165) return;
V += median(name_base[10], name_base[11], name_base[12]);
V += median(name_base[19], name_base[20], name_base[21]);
V += median(name_base[22], name_base[23], name_base[24]);
if (V < 250) return;
sort(name_base, name_base + 10);
V += (154 + name_base[3] + name_base[4] + name_base[5] + name_base[6]) / 3;
```
它的意思也显而易见，如果先计算出的几个属性已经弱到了让这个号不可能强（或者说，过八围筛子）的程度，可以直接丢掉。

### 1.2.3 编译器卡常

同样顾名思义，在不改变算法复杂度的前提下让编译器生成更快的可执行文件。例如：

**命令行参数**<br>
老生常谈。除了烂大街的 `-Ofast` 和指令集火车头，如果你能在测号的电脑上直接编译，也可以试试 `-march=native`。

**刺激编译器优化**<br>
在测号器中你可能会看到这样的代码：
```cpp
for (int i = 0; i < 96; i += 8) {
    ual[i + 0] = val[i + 0] * 181 + 160;
    ual[i + 1] = val[i + 1] * 181 + 160;
    ual[i + 2] = val[i + 2] * 181 + 160;
    ual[i + 3] = val[i + 3] * 181 + 160;
    ual[i + 4] = val[i + 4] * 181 + 160;
    ual[i + 5] = val[i + 5] * 181 + 160;
    ual[i + 6] = val[i + 6] * 181 + 160;
    ual[i + 7] = val[i + 7] * 181 + 160;
}
```
这样手动展开循环的目的是主动触发编译器的优化机制，虽然理论上 `-Ofast` 自带了循环展开功能，但为了保险起见还是手动展开一些关键部位为好。

**选择编译器**<br>
玄学，但确实管用。即使在代码和编译参数完全相同的情况下，不同编译器给出的产物测速之间也存在超出允差的区别。“只要能抓耗子就是好猫”，不论是 gcc、clang 还是 icc、msvc 或者其他的神秘编译器，只要能让你的测号器跑得更快就值得一试。除此之外还有更加玄学的编译器版本号选择，由于加速幅度有限，在此不作赘述。

### 1.2.4 pbb 测号器源码

pbb 测号器已经开源，如果你想自己编译或二次开发，可从下列链接下载：

- 适配 windows，使用 pthread：<https://pastebin.ubuntu.com/p/fQpmd9QmqS/>
- 适配 windows，不使用 pthread：<https://pastebin.ubuntu.com/p/dr6M3V8tsk/>
- 适配 linux，使用 pthread：<https://pastebin.ubuntu.com/p/523KnJHYx9/>

<script>openTab(event,false)</script>
