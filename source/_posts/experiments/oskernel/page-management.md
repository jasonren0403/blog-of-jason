---
title: 页面调度算法实现 date: 2019-10-27 22:43:11 tags: ["OS","页面调度"]
categories:

- ["OS","页面调度"]
- ["技术"]
- ["实验","操作系统内核"]
  comments: true

---

## 实验目的

理解操作系统中的几种页面调度算法，尝试用C++语言实现它们。
<!-- more -->

## 步骤

### 重要的数据结构

{% tabs data_structure,1 %}
<!-- tab <code>Page.h</code> -->

```c++ Page.h
#ifndef _PAGE_H
#define _PAGE_H
class CPage  {
public:
	int m_nPageNumber, //硬盘页编号(≤AP)
		m_nPageFaceNumber, //内存页编号(≤PP)，INVALID表示不在内存
		m_nCounter, //页面被使用次数
		m_nTime; //页面已存在内存中的时间
};
#endif
```

<!-- endtab -->
<!-- tab <code>PageControl.h</code> -->

```c++ PageControl.h
#ifndef _PAGECONTROL_H
#define _PAGECONTROL_H
class CPageControl  {
public:
	int m_nPageNumber,m_nPageFaceNumber;  //硬盘页编号，内存页编号
	class CPageControl * m_pNext; //指向下一个被控制页面
};
#endif
```

<!-- endtab -->
{% endtabs %}

### 算法示意

{% tabs algs,1 %}
<!-- tab 先进先出算法FIFO -->

1. 初始化。设置两个数组`page[ap]`和`pagecontrol[pp]`分别表示进程页面数和内存分配的页面数，并产生一个的随机数序列`main[total_instruction]`（当然这个序列由`page[]`
   的下标随机构成），表示待处理的进程页面顺序，`diseffect`置0。
2. 看`main[]`中是否有下一个元素，有就由`main[]`中获取该页面下标，并转到3；没有，就转到7。
3. 如果该`page`页已在内存中，就转到2；否则就到4，同时未命中的`diseffect`加1。
4. 观察`pagecontrol`是否占满，如果占满需将使用队列（6中建立的）中最先进入的（就是队列第一个单元）`pagecontrol`单元“清干净”，同时将对应的`page[]`单元置为“不在内存中”。
5. 将该`page[]`与`pagecontrol[]`建立关系（可以改变`pagecontrol[]`的标示位，也可以采用指针连接。总之，至少要使对应的`pagecontrol`单元包含两个信息：一是它被使用了，二是哪个`page[]`
   单元使用的；`page[]`单元包含两个信息：对应的`pagecontrol`单元号、本`page[]`单元已在内存中）；
6. 将用到的`pagecontrol`置入使用队列（这里的队列当然是一种先进先出的数据结构），返回2；
7. 显示命中率，算法完成。

{% mermaid graph TD %} A(["初始化"]) --> B{"main[] 中是否有下一个元素"} B --> |Y|C["由main获取下一个元素页面下标"]
B --> |N|D(["显示命中率，算法完成"])
C --> E{"该页是否在内存中"} E --> |Y|B E --> |N|F["diseffect++"]
F --> G{Pagecontrol是否占满} G --> |Y|H["将最先进入的单元清干净，并重置状态"]
G --> |N|I["建立page和pagecontrol的关系"]
H --> I I --> J["用到的Pagecontrol置入使用队列"]
J --> B {% endmermaid %}

```c++ FIFO算法
void CMemory::FIFO(const int nTotal_pf){
	int i;
	CPageControl *p;
	initialize(nTotal_pf);
	_pBusypf_head=_pBusypf_tail=NULL;
	for(i=0;i<TOTAL_INSTRUCTION;i++) {
	if(_vDiscPages[_vPage[i]].m_nPageFaceNumber==INVALID){
		_nDiseffect+=1;
    	if(_pFreepf_head==NULL){  // 无空闲页面
	  		p=_pBusypf_head->m_pNext;
	  		_vDiscPages[_pBusypf_head->m_nPageNumber].m_nPageFaceNumber=INVALID;
	  		_pFreepf_head=_pBusypf_head;
	  		_pFreepf_head->m_pNext=NULL;
	  		_pBusypf_head=p;
		}
		p=_pFreepf_head->m_pNext;
		_pFreepf_head->m_pNext=NULL;
		_pFreepf_head->m_nPageNumber=_vPage[i];
		_vDiscPages[_vPage[i]].m_nPageFaceNumber =_pFreepf_head->m_nPageFaceNumber;
		if(_pBusypf_tail==NULL)
			_pBusypf_head=_pBusypf_tail=_pFreepf_head;
		else{
	  		_pBusypf_tail->m_pNext=_pFreepf_head;
	  		_pBusypf_tail=_pFreepf_head;
		}
 		_pFreepf_head=p;
		}
	}
 	cout<<"FIFO: "<<1-(float)_nDiseffect/320<<"\t";
}
```

{% note success %} 易于实现 {% endnote %} {% note danger %} 容易淘汰常用页面 {% endnote %}
<!-- endtab -->
<!-- tab 最近最少使用的算法LRU -->

1. 初始化。主要是进程页面`page[]`和分配的内存页面`pagecontrol[]`，同时产生随机序列`main[]`，`diseffect`置0。
2. 看序列`main[]`是否有下一个元素，有就由`main[]`中获取该页面下标，并转到3；没有，就转到6。
3. 如果该`page`页已在内存中，便改变页面属性，使它保留“最近使用”的信息，转到2；否则就到4，同时未命中的`diseffect`加1。
4. 判断是否有空闲的内存页面，如果有，就返回页指针，转到5；否则在内存页面中找出最长时间没有使用到的页面，将其“清干净”，并返回该页面指针。
5. 在需要处理的`page[]`与4中得到的`pagecontrol[]`建立关系，同时让对应的`page[]`单元保存“最新使用”的信息，返回2；
6. 如果序列处理完成，就输出命中率，算法结束。

{% mermaid graph TD %} A(["初始化"]) --> B{"main[] 中是否有下一个元素"} B --> |Y|C["由main获取下一个元素页面下标"]
B --> |N|D(["显示命中率，算法完成"])
C --> E{"该页是否在内存中"} E --> |Y|E_1["修改页面属性，使它保留“最近使用”的信息"]
E_1 --> B E --> |N|F["diseffect++"]
F --> G{是否有空闲页面} G --> |Y|H["返回该页面指针"]
G --> |N|I["在内存页面中找出最长时间没有使用到的页面,将其清理"]
I --> H H --> J[内存页面和待处理的进程页面建立关系,对应单元保存最新使用信息]
J --> B {% endmermaid %}

```c++ LRU算法
void CMemory::LRU(const int nTotal_pf){
	int i,j,nMin,minj,nPresentTime(0);
 	initialize(nTotal_pf);
	for(i=0;i<TOTAL_INSTRUCTION;i++){
  		if(_vDiscPages[_vPage[i]].m_nPageFaceNumber==INVALID){
  			_nDiseffect++;
  			if(_pFreepf_head==NULL){
      			nMin=32767;
      			for(j=0;j<TOTAL_VP;j++)
                // 得到最近最少使用的页面的页号
                // 循环结束后,iMin表示最近最少使用页面的访问次数;minj表示需要换出的页号
					if(nMin>_vDiscPages[j].m_nTime&&_vDiscPages[j].m_nPageFaceNumber!=INVALID){
						nMin=_vDiscPages[j].m_nTime;
	    				minj=j;
					}

				_pFreepf_head=&_vMemoryPages[_vDiscPages[minj].m_nPageFaceNumber];
				_vDiscPages[minj].m_nPageFaceNumber=INVALID;
				_vDiscPages[minj].m_nTime=-1;
				_pFreepf_head->m_pNext=NULL;
   			}
			_vDiscPages[_vPage[i]].m_nPageFaceNumber=_pFreepf_head->m_nPageFaceNumber;
			_vDiscPages[_vPage[i]].m_nTime=nPresentTime;
			_pFreepf_head=_pFreepf_head->m_pNext;
		}
		else
			_vDiscPages[_vPage[i]].m_nTime=nPresentTime;
		nPresentTime++;
	}
  	cout<<"LRU: "<<1-(float)_nDiseffect/320<<"\t";
}
```

{% note success %} 页面利用率有显著提升 {% endnote %} {% note danger %} 代价很大 {% endnote %}
<!-- endtab -->
<!-- tab 最近未使用算法NRU -->

1. 初始化。设置两个数组`page[ap]`和`pagecontrol[pp]`分别表示进程页面数和内存分配的页面数，并产生一个的随机数序列`main[total_instruction]`（当然这个序列由`page[]`
   的下标随机构成），表示待处理的进程页面顺序，`diseffect`置0，设定循环周期`CLEAR_PERIOD`。
2. 看`main[]`中是否有下一个元素，有就从`main[]`中获得一个CPU将处理页面的序号；没有，就转到8。
3. 如果待处理的页面已在内存中，就转到2；否则`diseffect`加1，并转到4。
4. 看是否有空闲的内存页面，如果有，返回空闲页面指针，转到5；否则，在所有没有被访问且位于内存中的页面中按任意规则（或者取最近的一个页面；或者取下标最小的页面，等等）取出一个，返回清空后的内存页面指针。
5. 在待处理进程页面与内存页面之间建立联系，并标注该页面被访问。
6. 如果CPU已处理了`CLEAR_PERIOD`个页面，就将所有页面均设为“未访问”。
7. 返回2。
8. 如果CPU所有处理工作完成，就返回命中率的结果，算法结束。

{% mermaid graph TD %} A(["初始化"]) --> B{"main[] 中是否有下一个元素"} B --> |Y|C["由main获取下一个元素页面下标"]
B --> |N|D(["显示命中率，算法完成"])
C --> E{"该页是否在内存中"} E --> B E --> |N|F["diseffect++"]
F --> G{是否有空闲页面} G --> |Y|H["返回该页面指针"]
G --> |N|I["在所有没有被访问且位于内存中的页面中按任意规则（或者取最近的一个页面；或者取下标最小的页面，等等）取出一个"]
I --> H H --> J[内存页面和待处理的进程页面建立关系,标注页面被访问]
J --> K{"CPU已经处理了CLEAR_PERIOD个页面"} K --> |N|B K --> |Y|B'["将所有页面设置为'未访问'"]
B'--> B {% endmermaid %}

```c++ NRU算法
void CMemory::NRU(const int nTotal_pf)   {
	int i,j,nDiscPage,nOld_DiscPage;
 	bool bCont_flag;
  	initialize(nTotal_pf);
  	nDiscPage=0;
	for(i=0;i<TOTAL_INSTRUCTION;i++)     {
		if(_vDiscPages[_vPage[i]].m_nPageFaceNumber==INVALID)  	{
  			_nDiseffect++;
  			if(_pFreepf_head==NULL)	    {
      			bCont_flag=true;
      			nOld_DiscPage=nDiscPage;
      			while(bCont_flag)		{
					if(_vDiscPages[nDiscPage].m_nCounter==0&&_vDiscPages[nDiscPage].m_nPageFaceNumber!=INVALID)
	    					bCont_flag=false;
					else {
  						nDiscPage++;
  						if(nDiscPage==TOTAL_VP) nDiscPage=0;
  						if(nDiscPage==nOld_DiscPage)
  						for(j=0;j<TOTAL_VP;j++)
	  						_vDiscPages[j].m_nCounter=0;
	   				}
				}
				_pFreepf_head=&_vMemoryPages[_vDiscPages[nDiscPage].m_nPageFaceNumber];
				_vDiscPages[nDiscPage].m_nPageFaceNumber=INVALID;
      			_pFreepf_head->m_pNext=NULL;
    			}
			_vDiscPages[_vPage[i]].m_nPageFaceNumber=_pFreepf_head->m_nPageFaceNumber;
  			_pFreepf_head=_pFreepf_head->m_pNext;
		}
		else
  			_vDiscPages[_vPage[i]].m_nCounter=1;
    		if(i%CLEAR_PERIOD==0)
				for(j=0;j<TOTAL_VP;j++)
					_vDiscPages[j].m_nCounter=0;
    }
 	cout<<"NRU:"<<1-(float)_nDiseffect/320<<"\t";
}
```

{% note success %} 易于理解，容易实现。 {% endnote %}
<!-- endtab -->
<!-- tab 最佳置换算法OPT -->

1. 初始化。设置两个数组`page[ap]`和`pagecontrol[pp]`分别表示进程页面数和内存分配的页面数，并产生一个的随机数序列`main[total_instruction]`（当然这个序列由`page[]`
   的下标随机构成），表示待处理的进程页面顺序，`diseffect`置0。然后扫描整个页面访问序列，对`vDistance[TOTAL_VP]`数组进行赋值，表示该页面将在第几步被处理。
2. 看`main[]`中是否有下一个元素，有就从`main[]`中获取一个CPU待处理的页面号；没有，就转到6。
3. 如果该`page`页已在内存中，就转到2；否则就到4，同时未命中的`diseffect`加1。
4.
看是否有空闲的内存页面。如果有，就直接返回该页面指针。如果没有，遍历所有未处理的进程页面序列，如果有位于内存中的页面，而以后CPU不再处理，首先将其换出，返回页面指针；如果没有这样的页面，找出CPU最晚处理到的页面，将其换出，返回该内存页面指针。
5. 在内存页面和待处理的进程页面之间建立联系，返回2。
6. 输出命中率，算法结束。（命中率在扫描时完成计算）

{% note info %} 从以上算法描述中，可以看出，OPT是一种理想化的算法，因为操作系统中页面处理顺序是不一定的，所以页面将在哪一步被处理是不可预测的。但是，可以以此为标准来评价其他页面置换算法。 {% endnote %}

{% mermaid graph TD %} A(["初始化"]) --> B{"main[] 中是否有下一个元素"} B --> |Y|C["由main获取下一个元素页面下标"]
B --> |N|D(["显示命中率，算法完成"])
C --> E{"该页是否在内存中"} E --> |Y|B E --> |N|F["diseffect++"]
F --> G{是否有空闲页面} G --> |Y|H["返回该页面指针"]
G --> |N|I["遍历所有未处理的进程页面序列"]
I --> J{"存在位于内存中的页面，而以后CPU不再处理"} J --> |Y|K[将其换出]
J --> |N|L[找出CPU最晚处理到的页面]
L --> K K --> H H --> M[内存页面和待处理的进程页面建立关系]
M --> B {% endmermaid %}

```c++ OPT算法
void CMemory::OPT(const int nTotal_pf)
{
 	int i,j,max,maxpage,nDistance,vDistance[TOTAL_VP];
  	initialize(nTotal_pf);
  	for(i=0;i<TOTAL_INSTRUCTION;i++)    {
    		if(_vDiscPages[_vPage[i]].m_nPageFaceNumber==INVALID){
			_nDiseffect++;
	  		if(_pFreepf_head==NULL)	    {
	     			for(j=0;j<TOTAL_VP;j++)
           				if(_vDiscPages[j].m_nPageFaceNumber!=INVALID)
						vDistance[j]=32767;
					else
		      			vDistance[j]=0;
				nDistance=1;
  				for(j=i+1;j<TOTAL_INSTRUCTION;j++)		{
      				if((_vDiscPages[_vPage[j]].m_nPageFaceNumber!=INVALID)&&(vDistance[_vPage[j]]==32767))
        				vDistance[_vPage[j]]=nDistance;
     				nDistance++;
				}
    			max=-1;
    			for(j=0;j<TOTAL_VP;j++)
				if(max<vDistance[j]){
    					max=vDistance[j];
    					maxpage=j;
				}
			_pFreepf_head=&_vMemoryPages[_vDiscPages[maxpage].m_nPageFaceNumber];
    			_pFreepf_head->m_pNext=NULL;
    			_vDiscPages[maxpage].m_nPageFaceNumber=INVALID;
  			}
		_vDiscPages[_vPage[i]].m_nPageFaceNumber=_pFreepf_head->m_nPageFaceNumber;
	  	_pFreepf_head=_pFreepf_head->m_pNext;
		}
   	}
 	cout<<"OPT:"<<1-(float)_nDiseffect/320<<"\t";
}
```

<!-- endtab -->
{% endtabs %}

## 实验关键里程碑数据与结果

用g++编译`kernel.cpp`，运行截图如下 {% asset_img run.png %}

可以看到，随着页面数的增多，FIFO、LRU、NRU三种算法的命中率无限趋近于OPT算法，且每一种算法的命中率都在单调上升。（第三列输出写错成NUR了）

## 实验难点与收获

这次实验仍然是编程，主要是使用了C++语言，借助类和结构体实现了常见的页面置换算法。实现过程中主要的困难是将算法语言表示转化为代码语言，需要考虑的东西有很多。通过对于实验结果的观察，我对于页面置换算法有了更加直观的理解，了解到其随所处理页面数的增加，命中率也在增加，FIFO、LRU、NRU三种算法各有利弊，总体来说，它们都是操作系统中较为常用的页面置换算法。

## 实验思考

其实这个实验只是模拟了页面比较少的时候，几种页面置换算法的准确率变化，页面置换的顺序也都是随机生成的序列。实际上，我们的操作系统所要操作的页面情况远比模拟情况要复杂，命中率的上升可能也不会是单调的。

如果在输出中加上每次生成的操作序列和每次换入换出的情况的话，就会更加直观了，准确率从何而来也能看得更加清楚了。
