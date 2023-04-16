---
title: 使用echarts绘制2020疫情折线图
date: 2020-04-02 10:38:58
tags: ["frontend", "echarts", "前端"]
categories:
  - ["frontend", "echarts"]
  - ["技术"]
comments: true
---

2020 年开始到现在的时间，真是发生了太多的事情，心情复杂，但又没有什么话好说，只能以有限的方式表达着对这些事件的思考。

最近买了个服务器，也正在着手搭建网站，利用一下我已经学到的前端知识，第一个应用性的内容，就以这次疫情的情况为出发点吧！

<!--more-->

## 了解 echarts

{% blockquote echarts百科 https://segmentfault.com/t/echarts/info %}
ECharts，缩写来自 Enterprise Charts，商业级数据图表，是一个开源的数据可视化工具，一个纯 Javascript 的图表库，能够在 PC 端和移动设备上流畅运行，兼容当前绝大部分浏览器（IE6/7/8/9/10/11，chrome，firefox，Safari 等），底层依赖轻量级的 Canvas 库 ZRender，ECharts 提供直观，生动，可交互，可高度个性化定制的数据可视化图表。创新的拖拽重计算、数据视图、值域漫游等特性大大增强了用户体验，赋予了用户对数据进行挖掘、整合的能力。
{% endblockquote %}

## 需求确定

使用一个绘图库（这要怎么手画啊.jpg），完成 2020 疫情的数据收集，并绘制折线图。

### 准备零：工程建立

使用 IDE 新建一个 html5 工程，目录结构如下所示
{% asset_img 0.png dir %}

其实里面的大部分文件与这次任务并无太大关系，这样直接新建只是为了省事。

### 准备一：绘图库

经过网上一番搜索和对比后，本次统计图绘图任务我选择用 echarts 来完成。它的首页是 [https://www.echartsjs.com/zh/index.html](https://www.echartsjs.com/zh/index.html)，还蛮精美。

{% asset_img 1.png 网站首页 %}

点击“下载”来到它的下载界面。或直接访问 [https://www.echartsjs.com/zh/download.html](https://www.echartsjs.com/zh/download.html)。
{% asset_img 2.png 下载界面 %}

网站提供了三种安装方式：

1. 从下载的源代码或编译产物安装
2. 从 npm 安装（`npm install echarts`）
3. 选择所需模块，在线定制安装

由于这次任务并不需要所有的图表支持，我选择第三种安装方式，点击“在线定制”即可进入定制页面。
{% asset_img 3.png %}

选择需要的图表、坐标系、组件类型，在开发环境中我选择保留 IE8 的兼容性，同时不选择“代码压缩”以方便调试。

{% gp 4-3 %}
{% asset_img 4-chart.png 图表类型 %}
{% asset_img 4-coordinate.png 坐标系 %}
{% asset_img 4-component.png 组件 %}
{% asset_img 4-others.png 其它选项 %}
{% endgp %}

点击“下载”后，会出现一个编译界面，待到用到的所有组件加入完毕后，就可以下载生成的 js 文件了，默认文件名为`echarts.js`，我们把它保存在事先建好网页工程的 js 文件夹中。
{% asset_img 4-compile.png 编译 %}

{% asset_img 4-save.png 下载 %}

同时生成一份“代码压缩”版的 js 以供线上利用，这个默认文件名为`echarts.min.js`

{% asset_img 4-save-compressed.png 下载-压缩版 %}

下载好所需的 js 文件以后，就可以开始工作了~

1.  建立一个空的 html 文档，填写必要的网页信息。
    {% codeblock lang:html index.html %}
    <!doctype html>
    <html class="no-js" lang="zh">

    <head>
      <meta charset="utf-8">
      <title>2020肺炎疫情数据 - 即时更新</title>
      <!--省略其余meta相关代码-->
    </head>

    <body>
      <!--省略jQuery相关代码-->
    </body>

    </html>
    {% endcodeblock %}

2.  引入`echarts.js`
	{% note info %}
    在线上环境时，我们将引入`echarts.min.js`
    {% endnote %}
    ```html index.html
    <!--head标签下-->
    <script src="js/echarts.js"></script>
    ```
3.  为 Echarts 准备一个具有高宽的 DOM 容器
    ```html index.html
    <body>
	    <!-- 为 ECharts 准备一个具备大小（宽高）的 DOM -->
		<div id="main" style="width: 600px;height:400px;"></div>
    </body>
    ```

4.  通过`echarts.init`方法初始化一个 echarts 实例，并通过`setOption`方法生成一个简单的统计图，大概是像这样的格式：
    ```html index.html
    <script type="text/javascript">
    	 // 基于准备好的dom，初始化echarts实例
    	var myChart = echarts.init(document.getElementById('main'));
    	var option = {...}; //这里做好设置和填充数据
    	myChart.setOption(option); //使用刚指定的配置项和数据显示图表
    </script>
    ```

    - `options`中要怎么指定设置和数据呢？别急，我们先来看它的官方示例数据和它渲染出来的图：
        {% tabs code_show %}
        <!-- tab 代码@code -->
        ```javascript main.js
        option = {
            xAxis: {
			    type: 'category',
			    data: ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']
            },
            yAxis: {
        	    type: 'value'
            },
            series: [{
			    data: [820, 932, 901, 934, 1290, 1330, 1320],
			    type: 'line'
            }]
        };
        ```
        <!-- endtab -->
        <!-- tab 示意图@image -->
        {% asset_img 5.png 单行显示 %}
        <!-- endtab -->
        {% endtabs %}

      	- option 中有 xAxis、yAxis、series 三个对象，它们都拥有 type 和 data 两个属性。对于这个简单的示例来说：
            - xAxis 是横轴数据及类型描述
            - yAxis 是纵轴数据及类型描述
            - series 是系列描述，可以包含多组“统计数据”
                {% note info %}
                在 echarts 里，系列（series）是指：一组数值以及他们映射成的图。“系列”这个词原本可能来源于“一系列的数据”，而在 echarts 中取其扩展的概念，不仅表示数据，也表示数据映射成为的图。所以，一个 **系列**包含的要素至少有：一组数值、图表类型（series.type）、以及其他的关于这些数据如何映射成图的参数。
                {% endnote %} 
                * 实际上，这也意味着可以在一个 echarts 对象上同时绘制多组类型各同或各异的统计数据（注意到 series 是一个数组！）！如果把以上例子的 series 数据稍作修改，就可以得到多折线的统计图，这也是本次数据处理任务的重要模型！
                    {% tabs code_show2 %}
                    <!-- tab 代码@code -->
                    ```javascript
                    option = {
                        xAxis: {
                            type: 'category',
                            data: ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']
                        },
                        yAxis: {
                            type: 'value'
                        },
                        series: [
                            {
                                data: [820, 932, 901, 934, 1290, 1330, 1320],
                                type: 'line'
                            },
                            {
                                data: [100,170,250,330,440,114,225],
                                type: 'line'
                            }
                        ]
                    };
                    ```
                    <!-- endtab -->
                    <!-- tab 示意图@image -->
                    {% asset_img 6.png 多行折线图 %}
                    <!-- endtab -->
                    {% endtabs %}

        - xAxis、yAxis、series 实质为统计图的“组件”。在 echarts 中，各种内容都是被抽象为“组件”的，除了前述的 xAxis、yAxis、series 以外，还有 grid、polar、geo 等组件，与本次任务关系并不大。
            - 所有的组件都在 option 对象中声明，可以是一个对象或者数组。
        - 上层的 option 对象描述了图表的各种需求，包括：有什么数据、要画什么图表、图表长什么样子、含有什么组件、组件能操作什么事情等等。这些设置内容通过`setOption`函数绑定到 echarts 对象上。

            ```javascript
            // 用 option 描述数据、数据如何映射成图形、交互行为等。
            // option 是个大的 JavaScript 对象。
            var option = {
                // option 每个属性是一类组件。
				legend: {...},
				grid: {...},
                tooltip: {...},
                toolbox: {...},
                dataZoom: {...},
                visualMap: {...},
                // 如果有多个同类组件，那么就是个数组。例如这里有三个 X 轴。
				xAxis: [
                    // 数组每项表示一个组件实例，用 type 描述“子类型”。
					{type: 'category', ...},
					{type: 'category', ...},
					{type: 'value', ...}
				],
				yAxis: [{...}, {...}],
                // 这里有多个系列，也是构成一个数组。
				series: [
                    // 每个系列，也有 type 描述“子类型”，即“图表类型”。
					{type: 'line', data: [['AA', 332], ['CC', 124], ['FF', 412], /_ ... _/ ]},
					{type: 'line', data: [2231, 1234, 552, /* ... */ ]},
					{type: 'line', data: [[4, 51], [8, 12], /* ... */ ]}
				]
            };
            ```

            - 数据都在`series.data`中，也可通过 dataset 来取得数据。

                ```javascript
                var option = {
            	    dataset: {
            		    source: [
            			    [121, "XX", 442, 43.11],
            			    [663, "ZZ", 311, 91.14],
            			    [913, "ZZ", 312, 92.12],
            			/* ... */
            		    ],
            	    },
            	    xAxis: {},
            	    yAxis: {},
            	    series: [
            		    // 数据从 dataset 中取，encode 中的数值是 dataset.source 的维度 index（即第几列）
            		    { type: "bar", encode: { x: 1, y: 0 } },
            		    { type: "bar", encode: { x: 1, y: 2 } },
            		    { type: "scatter", encode: { x: 1, y: 3 } },
            		    /* ... */
            	    ],
                };
                ```

            > 总的来说，option 表述了：数据、数据如何映射成图形、交互行为。

       - 通过以上这几个步骤，我们可以绘制一个简单的统计图。

### 准备二：数据

本次使用的数据源为[丁香园疫情实时播报网页](https://ncov.dxy.cn/ncovh5/view/pneumonia)。打开网站发现我们所要的数据在这里呈现：
{% asset_img 7.png 网站页面 %}
右键或者 F12 打开开发者模式，发现数据来源为这段 JavaScript 代码

{% asset_img 8.png 页面代码 %}
其中正好有我们需要的数据

```json5
{
	/* .... */
	currentConfirmedCount: 2934, //现存确诊
	confirmedCount: 82691, //累计确诊数
	suspectedCount: 806, //境外输入
	curedCount: 76436, //累计治愈数
	deadCount: 3321, //累计死亡数
	/* .... */
}
```

使用 python requests 库爬下来，扔进 BeautifulSoup 解析一通，然后存到数据库中，其中数据库的结构为

```sql
create table `2020_pneumonia`
(
  last_since datetime null, # 上次数据更新时间
  proved int null, # 已确诊数
  uncertain int null, # 未确诊数
  died int null, # 死亡数
  cured int null # 治愈数
);
```

```python crawl.py
req = requests.get(url='https://ncov.dxy.cn/ncovh5/view/pneumonia', headers=headers, verify=False)
req.encoding = req.apparent_encoding
soup = BeautifulSoup(req.text, 'html.parser')
final_data = {}
try:
  import json
  json_text = soup.find('script', {'id': 'getStatisticsService'}).get_text().replace(
  'try { window.getStatisticsService = ',
  '').replace('catch(e){}', '')
  json_text = json_text[:-1]
  data_dict = json.loads(json_text)
  final_data['confirmed'] = int(data_dict['confirmedCount'])
  final_data['suspected'] = int(data_dict['suspectedCount'])
  final_data['cured'] = int(data_dict['curedCount'])
  final_data['dead'] = int(data_dict['deadCount'])
  final_data['last_since'] = data_dict['modifyTime']  # "modifyTime":单位是 ms!
except Exception as e:
  # json 解析出错，几乎不可能出现（因为这样的话原来以js驱动的网页也显示不出数据了）
  quit(-1)
finally:
  try:
	import datetime,time
	# 省略连库过程
	cur.execute('select last_since from 2020_pneumonia order by last_since desc')
	ret = cur.fetchone()
	logger.info(ret)  # {'last_since': datetime.datetime(2020, 2, 9, 22, 55)}
	ret_time = ret.get('last_since')
	local_time = datetime.datetime.fromtimestamp(final_data['last_since']/1000)
	if ret_time == local_time:
	  logger.info("[*]No need to insert!")
	else:
	  date_in = datetime.datetime.fromtimestamp(final_data['last_since']/1000).strftime("%Y-%m-%d %H:%M:%S")
      proved = final_data['confirmed']
	  uncertain = final_data['suspected']
      died = final_data['dead']
      cured = final_data['cured']
      sql = "insert into 2020_pneumonia values('%s',%d,%d,%d,%d);"%(date_in,proved,uncertain,died,cured)
      cur.execute(sql)
      conn.commit()
      logger.info("[√]Insert success!")
  finally:
    conn.close()
```

### 准备三：数据交互

编写一个 php 接口，借以从数据库中获取数据，其中接口行为设计如下：
{% codeblock %}
POST api.php HTTP /1.1
action=getfulldata 获得全部数据，对应的 SQL 语句为 select _ from 表名，最多加上个 limit
action=getlatest 得到最新数据，对应的 SQL 语句可以写为 select _ from 表名 order by last_since desc limit 1
action=getsingleday&param={Y-m-d} 得到某一天的统计数据，SQL 语句是一个简单的条件查询，不再赘述了
{% endcodeblock %}
稍后会在 html 网页中用 jQuery ajax 技术调用这个接口。

## 编写主网页

1. 在`body`标签中，准备一个`div`容器，一会放我们的 echarts 图表。这个容器需要指明一个高度

    ```html index.html
    <div id="chartbox" style="width:100%;height:100%;margin:0 auto">
    	<div id="main" style="width: 600px;height:calc(70% + 10px);margin:0 auto;"></div>
    </div>
    ```

2. 在`div`容器中初始化 echarts 对象

    ```javascript index.html
    var main = document.getElementById("main");
    var chart = echarts.init(main, "light"); //设置echarts显示主题为亮色主题
    ```

3. 图标样式设置（chartOptions）

    ```javascript main.js
    chartoption = {
    	title: {
    		text: "2020肺炎疫情数据",
    		lineHeight: 40,
    		height: 40,
    		subtext: "数据来源:丁香园·丁香医生",
    		sublink: "https://ncov.dxy.cn/ncovh5/view/pneumonia",
    	},
    	dataZoom: [
    		{
    			id: "dataZoomX",
    			type: "inside",
    			xAxisIndex: [0],
    			filterMode: "empty",
    		},
    		{
    			id: "dataZoomY",
    			type: "inside",
    			yAxisIndex: [0],
    			filterMode: "empty",
    		},
    	],
    	dataset: {
    		dimensions: ["last_since", "proved", "uncertain", "died", "cured"],
    		source: [
    			// {last_since:xx(datetime),proved:xx(int),uncertain:xx(int),died:xx(int),cured:(int)}
    		],
    	},
    	tooltip: {
    		trigger: "axis",
    	},
    	toolbox: {
    		show: true,
    		orient: "vertical",
    		top: 20,
    		feature: {
    			dataZoom: {
    				yAxisIndex: "none",
    			},
    			// dataView: {
    			//     readOnly: true
    			// },
    			restore: {},
    			saveAsImage: {
    				title: "保存为图片…",
    			},
    		},
    	},
    	legend: {
    		type: "scroll",
    		data: ["proved", "uncertain", "died", "cured"],
    		left: "right",
    	},
    	xAxis: {
    		type: "category",
    		// boundaryGap: false,
    	},
    	yAxis: {
    		type: "value",
    	},
    	series: [
    		{
    			type: "line",
    		},
    		{
    			type: "line",
    		},
    		{
    			type: "line",
    		},
    		{
    			type: "line",
    		},
    	],
    	animationEasing: "quarticOut",
    };
    ```

    然后再将设置对象绑定到之前的 chart 对象上

    ```javascript main.js
    chart.setOption(chartoption);
    ```

4. 异步更新及获取数据，根据 echarts 官方文档的说法，只要在 jQuery 等工具异步获取完数据后，再调用 setOption 填入数据和配置项就行，非常方便

    ```javascript main.js
    $(document).ready(function () {
    	$.post("api.php", { action: "getfull" }, "json").done(function (data) {
    		chart.setOption({
    			dataset: {
    				source: data.contents, // 返回数据大致为{"success":true,"contents":[{},{},...]}
    			},
    		});
    	});
    });
    ```

5. 由于数据较多，加载时间较长，我们给 chart 对象添加一个简单的 loading 动画，在数据加载完成后，移除 loading 动画。

    ```javascript main.js
    chart.showLoading();
    $(document).ready(function () {
    	//getting data with jQuery...
    	chart.hideLoading();
    });
    ```

## 效果

- 经过以上步骤后，打开网页，现在显示效果如下，虽然不怎么酷炫，但是已经基本上满足了我们的需求，可以看到大概的数据走向。可以用鼠标滚轮实现局部放大效果。调一调样式，就可以嵌入到一般网页中去使用了。
	
	{% asset_img 9.png 效果 %}

{% asset_img 10.png 局部效果 %}

## 后续的思考和启示

- 其实丁香园网页的开头还有几段 javascript 数据，多加研究的话还有可以值得提取和研究的东西，比如最近新加的国外疫情数据
- 原来页面真的可以全用 javascript 渲染出来……丁香园疫情网页应该用到了类似于 webpack 和客户端渲染的东西，网页`body`元素中全是 javascript 代码
    - 具体来讲，应该是在`script`标签中加载了一些 js 和 css 文件
- 数据获取部分本来想用 php 一起写掉，但奈何 php 中想要爬虫，只有使用 curl 库来进行抓取，且后续正则提取麻烦，索性放弃了
