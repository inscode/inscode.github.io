---
title: Golang爬虫系列一：使用Go抓取豆瓣电影top250数据信息
date: 2019-08-10 18:19:03
tags:
    - Golang
    - 爬虫
categories:
    - Golang
---

###### 声明：以下内容仅供技术学习

简述：爬取豆瓣电影 `top 250` 的基础信息，包括电影🎬名，评分，评价人数，电影链接。
<!--more-->


```golang
package main

import (
     "fmt"
     "strconv"
     "net/http"
     "io"
     "regexp"
     "os"
)

func main() {
     var start int //起始页
     var end int    //结束页
     fmt.Print("请输入爬取的起始页:")
     fmt.Scanln(&start)
     fmt.Print("请输入爬取的结束页:")
     fmt.Scanln(&end)
     toWorking(start, end)
}

func toWorking(start, end int) {
     fmt.Printf("起始结束页是%d : %d\n", start, end)

     pageChan := make(chan int)
     for i := start; i <= end; i++ {
          go SpiderDoubanPage(i, pageChan)
     }

     for i := start; i <= end; i++ {
          //开启协程
          fmt.Printf("第 %d 页 爬取完成\n", <-pageChan)
     }
}

func SpiderDoubanPage(i int, pageChan chan int) {
     //https://movie.douban.com/top250?start=0&filter=
     //https://movie.douban.com/top250?start=25&filter=
     //https://movie.douban.com/top250?start=50&filter=
     url := "https://movie.douban.com/top250?start=" + strconv.Itoa((i-1)*25)

     //爬取url页面内容【横向爬取】
     result, err := httpGetDoubanUrlContent(url)
     if err != nil {
          fmt.Println("err message：", err)
          return
     }

     //fmt.Println(result)
     //使用正则表达式解析内容---电影名称
     film_pat := regexp.MustCompile(`width="100" alt="(.*?)"`)
     FilmNames := film_pat.FindAllStringSubmatch(result, -1)

     //使用正则表达式解析内容---电影分数
     score_pat := regexp.MustCompile(`v:average">(?s:(.*?))<`)
     filmScores := score_pat.FindAllStringSubmatch(result, -1)

     //使用正则表达式解析内容---评价人数
     people_pat := regexp.MustCompile(`<span>(?s:(\d*?))人评价</span>`)
     peopleNum := people_pat.FindAllStringSubmatch(result, -1)

     //使用正则表达式解析内容---电影详情页url
     film_url_pat := regexp.MustCompile(`<a href="(.*?)" class="">`)
     filmUrls := film_url_pat.FindAllStringSubmatch(result, -1)

     //使用正则表达式解析内容---电影序号
     film_order_pat := regexp.MustCompile(`<em class="">(\d*)</em>`)
     filmOrders := film_order_pat.FindAllStringSubmatch(result, -1)

     saveToFile(i, filmOrders, FilmNames, filmScores, peopleNum, filmUrls)

     pageChan <- i

}

func httpGetDoubanUrlContent(url string) (result string, err error) {
     resp, err1 := http.Get(url)
     if err1 != nil {
          err = err1
          return
     }

     defer resp.Body.Close()

     buffer := make([]byte, 8192)
     for {
          n, err2 := resp.Body.Read(buffer)
          if n == 0 {
                fmt.Println("读取网页完成")
                break
          }

          if err2 != nil && err2 != io.EOF {
                err = err2
                return
          }
          result += string(buffer[:n])
     }
     return
}

func saveToFile(idx int, filmOrders, filmNames, filmScores, peopleNum, filmUrls [][]string) {
     //将读取到的数据存储存储为文件
     dir, _ := os.Getwd()
     filePath := dir + "douban_top_" + strconv.Itoa(idx) + ".txt"

     file, err := os.Create(filePath)
     if err != err {
          fmt.Println("os Create err：", err.Error())
          return
     }

     defer file.Close() //保存好一个文件就关闭一个文件

     file.WriteString(
          "序号" +
                "\t" + "电影名称" +
                "\t" + "电影评分" +
                "\t" + "评价人数" +
                "\t" + "链接地址" +
                "\n")
     for i := 0; i < len(filmNames); i++ {
          file.WriteString(
                filmOrders[i][1] +
                     "\t" + filmNames[i][1] +
                     "\t" + filmScores[i][1] +
                     "\t" + peopleNum[i][1] +
                     "\t" + filmUrls[i][1] +
                     "\n")
     }
}


```

##### 使用 `linux` 命令合并爬取的文件

- `cat crawlerdouban_top_*.txt>top250.txt` //文件合并

- `cat top250.txt |grep -v '序号'>douban_top250_movie_list.txt` //去取指定内容

- `sort -k 1 -n douban_top250_movie_list.txt>douban_top250_movie_list_ordered.txt` //排序

- 看心情在文件头部加上 `序号     电影名称     电影评分     评价人数     链接地址`

- 结果如下

```txt
序号     电影名称     电影评分     评价人数     链接地址
1    肖申克的救赎    9.7    1535695    https://movie.douban.com/subject/1292052/
2    霸王别姬    9.6    1136522    https://movie.douban.com/subject/1291546/
3    这个杀手不太冷    9.4    1388824    https://movie.douban.com/subject/1295644/
4    阿甘正传    9.4    1199279    https://movie.douban.com/subject/1292720/
5    美丽人生    9.5    702825    https://movie.douban.com/subject/1292063/
6    千与千寻    9.3    1231099    https://movie.douban.com/subject/1291561/
7    泰坦尼克号    9.4    1140842    https://movie.douban.com/subject/1292722/
8    辛德勒的名单    9.5    623915    https://movie.douban.com/subject/1295124/
9    盗梦空间    9.3    1189561    https://movie.douban.com/subject/3541415/
10    忠犬八公的故事    9.3    795328    https://movie.douban.com/subject/3011091/
11    机器人总动员    9.3    788458    https://movie.douban.com/subject/2131459/
12    三傻大闹宝莱坞    9.2    1075170    https://movie.douban.com/subject/3793023/
13    放牛班的春天    9.3    748223    https://movie.douban.com/subject/1291549/
14    海上钢琴师    9.2    872907    https://movie.douban.com/subject/1292001/
15    楚门的世界    9.2    832134    https://movie.douban.com/subject/1292064/
16    大话西游之大圣娶亲    9.2    835440    https://movie.douban.com/subject/1292213/
17    星际穿越    9.3    854952    https://movie.douban.com/subject/1889243/
18    龙猫    9.2    739493    https://movie.douban.com/subject/1291560/
19    熔炉    9.3    492270    https://movie.douban.com/subject/5912992/
20    教父    9.3    536821    https://movie.douban.com/subject/1291841/
21    无间道    9.2    686756    https://movie.douban.com/subject/1307914/
22    疯狂动物城    9.2    961506    https://movie.douban.com/subject/25662329/
23    当幸福来敲门    9.1    870538    https://movie.douban.com/subject/1849031/
24    怦然心动    9.0    971877    https://movie.douban.com/subject/3319755/
25    触不可及    9.2    569502    https://movie.douban.com/subject/6786002/
26    蝙蝠侠：黑暗骑士    9.1    561534    https://movie.douban.com/subject/1851857/
27    乱世佳人    9.3    394279    https://movie.douban.com/subject/1300267/
28    活着    9.2    444310    https://movie.douban.com/subject/1292365/
29    控方证人    9.6    199996    https://movie.douban.com/subject/1296141/
30    少年派的奇幻漂流    9.0    854147    https://movie.douban.com/subject/1929463/
31    指环王3：王者无敌    9.2    432241    https://movie.douban.com/subject/1291552/
32    摔跤吧！爸爸    9.0    848439    https://movie.douban.com/subject/26387939/
33    天空之城    9.1    507243    https://movie.douban.com/subject/1291583/
34    鬼子来了    9.2    361061    https://movie.douban.com/subject/1291858/
35    十二怒汉    9.4    257052    https://movie.douban.com/subject/1293182/
36    天堂电影院    9.2    416891    https://movie.douban.com/subject/1291828/
37    飞屋环游记    9.0    773645    https://movie.douban.com/subject/2129039/
38    大话西游之月光宝盒    9.0    675104    https://movie.douban.com/subject/1299398/
39    哈尔的移动城堡    9.0    570115    https://movie.douban.com/subject/1308807/
40    搏击俱乐部    9.0    543030    https://movie.douban.com/subject/1292000/
41    罗马假日    9.0    576438    https://movie.douban.com/subject/1293839/
42    末代皇帝    9.2    354693    https://movie.douban.com/subject/1293172/
43    寻梦环游记    9.0    811262    https://movie.douban.com/subject/20495023/
44    闻香识女人    9.0    493633    https://movie.douban.com/subject/1298624/
45    辩护人    9.2    340911    https://movie.douban.com/subject/21937445/
46    素媛    9.2    322715    https://movie.douban.com/subject/21937452/
47    窃听风暴    9.1    340301    https://movie.douban.com/subject/1900841/
48    死亡诗社    9.0    414748    https://movie.douban.com/subject/1291548/
49    两杆大烟枪    9.1    369786    https://movie.douban.com/subject/1293350/
50    飞越疯人院    9.1    373765    https://movie.douban.com/subject/1292224/
51    指环王2：双塔奇兵    9.0    405233    https://movie.douban.com/subject/1291572/
52    教父2    9.2    291589    https://movie.douban.com/subject/1299131/
53    指环王1：魔戒再现    9.0    454776    https://movie.douban.com/subject/1291571/
54    狮子王    9.0    441123    https://movie.douban.com/subject/1301753/
55    V字仇杀队    8.9    634942    https://movie.douban.com/subject/1309046/
56    美丽心灵    9.0    442940    https://movie.douban.com/subject/1306029/
57    饮食男女    9.1    312944    https://movie.douban.com/subject/1291818/
58    海豚湾    9.3    246512    https://movie.douban.com/subject/3442220/
59    情书    8.9    544709    https://movie.douban.com/subject/1292220/
60    何以为家    9.1    353458    https://movie.douban.com/subject/30170448/
61    钢琴家    9.1    293055    https://movie.douban.com/subject/1296736/
62    大闹天宫    9.3    190080    https://movie.douban.com/subject/1418019/
63    本杰明·巴顿奇事    8.9    577017    https://movie.douban.com/subject/1485260/
64    哈利·波特与魔法石    8.9    466720    https://movie.douban.com/subject/1295038/
65    看不见的客人    8.8    635451    https://movie.douban.com/subject/26580232/
66    黑客帝国    8.9    434713    https://movie.douban.com/subject/1291843/
67    西西里的美丽传说    8.8    552815    https://movie.douban.com/subject/1292402/
68    小鞋子    9.2    218270    https://movie.douban.com/subject/1303021/
69    美国往事    9.2    248262    https://movie.douban.com/subject/1292262/
70    拯救大兵瑞恩    9.0    370031    https://movie.douban.com/subject/1292849/
71    让子弹飞    8.8    937467    https://movie.douban.com/subject/3742360/
72    音乐之声    9.0    340645    https://movie.douban.com/subject/1294408/
73    致命魔术    8.8    497893    https://movie.douban.com/subject/1780330/
74    猫鼠游戏    8.9    404409    https://movie.douban.com/subject/1305487/
75    七宗罪    8.8    593674    https://movie.douban.com/subject/1292223/
76    被嫌弃的松子的一生    8.9    445775    https://movie.douban.com/subject/1787291/
77    低俗小说    8.8    494334    https://movie.douban.com/subject/1291832/
78    沉默的羔羊    8.8    497708    https://movie.douban.com/subject/1293544/
79    蝴蝶效应    8.8    548478    https://movie.douban.com/subject/1292343/
80    春光乍泄    8.9    367593    https://movie.douban.com/subject/1292679/
81    勇敢的心    8.9    388081    https://movie.douban.com/subject/1294639/
82    天使爱美丽    8.7    657882    https://movie.douban.com/subject/1292215/
83    穿条纹睡衣的男孩    9.0    256883    https://movie.douban.com/subject/3008247/
84    剪刀手爱德华    8.7    675841    https://movie.douban.com/subject/1292370/
85    心灵捕手    8.8    402941    https://movie.douban.com/subject/1292656/
86    禁闭岛    8.8    537386    https://movie.douban.com/subject/2334904/
87    布达佩斯大饭店    8.8    500379    https://movie.douban.com/subject/11525673/
88    阿凡达    8.7    787025    https://movie.douban.com/subject/1652587/
89    入殓师    8.8    401708    https://movie.douban.com/subject/2149806/
90    幽灵公主    8.9    336781    https://movie.douban.com/subject/1297359/
91    加勒比海盗    8.7    519035    https://movie.douban.com/subject/1298070/
92    摩登时代    9.3    148862    https://movie.douban.com/subject/1294371/
93    致命ID    8.7    455343    https://movie.douban.com/subject/1297192/
94    断背山    8.7    448806    https://movie.douban.com/subject/1418834/
95    阳光灿烂的日子    8.8    379539    https://movie.douban.com/subject/1291875/
96    重庆森林    8.7    471704    https://movie.douban.com/subject/1291999/
97    第六感    8.8    326415    https://movie.douban.com/subject/1297630/
98    狩猎    9.1    196797    https://movie.douban.com/subject/6985810/
99    喜剧之王    8.7    513460    https://movie.douban.com/subject/1302425/
100    玛丽和马克思    8.9    290154    https://movie.douban.com/subject/3072124/
101    消失的爱人    8.7    525034    https://movie.douban.com/subject/21318488/
102    告白    8.7    467513    https://movie.douban.com/subject/4268598/
103    小森林 夏秋篇    9.0    226137    https://movie.douban.com/subject/25814705/
104    大鱼    8.8    347484    https://movie.douban.com/subject/1291545/
105    一一    9.0    210191    https://movie.douban.com/subject/1292434/
106    阳光姐妹淘    8.8    378032    https://movie.douban.com/subject/4917726/
107    爱在黎明破晓前    8.8    326958    https://movie.douban.com/subject/1296339/
108    请以你的名字呼唤我    8.8    327024    https://movie.douban.com/subject/26799731/
109    射雕英雄传之东成西就    8.7    384745    https://movie.douban.com/subject/1316510/
110    甜蜜蜜    8.8    307227    https://movie.douban.com/subject/1305164/
111    侧耳倾听    8.9    256439    https://movie.douban.com/subject/1297052/
112    红辣椒    8.9    210659    https://movie.douban.com/subject/1865703/
113    驯龙高手    8.7    438648    https://movie.douban.com/subject/2353023/
114    倩女幽魂    8.7    399855    https://movie.douban.com/subject/1297447/
115    超脱    8.8    256836    https://movie.douban.com/subject/5322596/
116    杀人回忆    8.8    314334    https://movie.douban.com/subject/1300299/
117    海蒂和爷爷    9.1    142601    https://movie.douban.com/subject/25958717/
118    恐怖直播    8.7    349058    https://movie.douban.com/subject/21360417/
119    菊次郎的夏天    8.8    284490    https://movie.douban.com/subject/1293359/
120    爱在日落黄昏时    8.8    285141    https://movie.douban.com/subject/1291990/
121    7号房的礼物    8.8    276596    https://movie.douban.com/subject/10777687/
122    小森林 冬春篇    9.0    196346    https://movie.douban.com/subject/25814707/
123    风之谷    8.8    246043    https://movie.douban.com/subject/1291585/
124    哈利·波特与死亡圣器(下)    8.7    408422    https://movie.douban.com/subject/3011235/
125    我不是药神    9.0    1181641    https://movie.douban.com/subject/26752088/
126    幸福终点站    8.7    310524    https://movie.douban.com/subject/1292274/
127    蝙蝠侠：黑暗骑士崛起    8.7    415939    https://movie.douban.com/subject/3395373/
128    上帝之城    8.9    200345    https://movie.douban.com/subject/1292208/
129    萤火之森    8.8    261708    https://movie.douban.com/subject/5989818/
130    借东西的小人阿莉埃蒂    8.8    308863    https://movie.douban.com/subject/4202302/
131    超能陆战队    8.6    565299    https://movie.douban.com/subject/11026735/
132    唐伯虎点秋香    8.5    545655    https://movie.douban.com/subject/1306249/
133    神偷奶爸    8.5    572719    https://movie.douban.com/subject/3287562/
134    无人知晓    9.1    133082    https://movie.douban.com/subject/1292337/
135    怪兽电力公司    8.7    363701    https://movie.douban.com/subject/1291579/
136    电锯惊魂    8.7    287704    https://movie.douban.com/subject/1417598/
137    岁月神偷    8.7    394423    https://movie.douban.com/subject/3792799/
138    玩具总动员3    8.8    278994    https://movie.douban.com/subject/1858711/
139    血战钢锯岭    8.7    472814    https://movie.douban.com/subject/26325320/
140    谍影重重3    8.8    254835    https://movie.douban.com/subject/1578507/
141    疯狂原始人    8.7    518689    https://movie.douban.com/subject/1907966/
142    七武士    9.2    108944    https://movie.douban.com/subject/1295399/
143    英雄本色    8.7    276917    https://movie.douban.com/subject/1297574/
144    喜宴    8.9    195228    https://movie.douban.com/subject/1303037/
145    真爱至上    8.6    430547    https://movie.douban.com/subject/1292401/
146    萤火虫之墓    8.7    272296    https://movie.douban.com/subject/1293318/
147    东邪西毒    8.6    362487    https://movie.douban.com/subject/1292328/
148    傲慢与偏见    8.5    430151    https://movie.douban.com/subject/1418200/
149    时空恋旅人    8.7    302907    https://movie.douban.com/subject/10577869/
150    贫民窟的百万富翁    8.6    497991    https://movie.douban.com/subject/2209573/
151    黑天鹅    8.5    549429    https://movie.douban.com/subject/1978709/
152    记忆碎片    8.6    373474    https://movie.douban.com/subject/1304447/
153    心迷宫    8.7    274327    https://movie.douban.com/subject/25917973/
154    纵横四海    8.8    226602    https://movie.douban.com/subject/1295409/
155    教父3    8.8    191865    https://movie.douban.com/subject/1294240/
156    荒蛮故事    8.8    223635    https://movie.douban.com/subject/24750126/
157    完美的世界    9.1    123076    https://movie.douban.com/subject/1300992/
158    达拉斯买家俱乐部    8.7    264828    https://movie.douban.com/subject/1793929/
159    雨人    8.7    267637    https://movie.douban.com/subject/1291870/
160    三块广告牌    8.7    493952    https://movie.douban.com/subject/26611804/
161    花样年华    8.6    337477    https://movie.douban.com/subject/1291557/
162    被解救的姜戈    8.7    357721    https://movie.douban.com/subject/6307447/
163    卢旺达饭店    8.9    161524    https://movie.douban.com/subject/1291822/
164    你的名字。    8.4    757251    https://movie.douban.com/subject/26683290/
165    海边的曼彻斯特    8.6    294826    https://movie.douban.com/subject/25980443/
166    我是山姆    8.9    155512    https://movie.douban.com/subject/1306861/
167    头脑特工队    8.7    340013    https://movie.douban.com/subject/10533913/
168    你看起来好像很好吃    8.8    202385    https://movie.douban.com/subject/4848115/
169    恋恋笔记本    8.5    385616    https://movie.douban.com/subject/1309163/
170    哪吒闹海    9.0    129640    https://movie.douban.com/subject/1307315/
171    无敌破坏王    8.7    306834    https://movie.douban.com/subject/6534248/
172    虎口脱险    8.9    141090    https://movie.douban.com/subject/1296909/
173    冰川时代    8.5    379209    https://movie.douban.com/subject/1291578/
174    二十二    8.7    168873    https://movie.douban.com/subject/26430107/
175    海洋    9.0    116507    https://movie.douban.com/subject/3443389/
176    雨中曲    9.0    125013    https://movie.douban.com/subject/1293460/
177    爆裂鼓手    8.7    324492    https://movie.douban.com/subject/25773932/
178    未麻的部屋    8.9    147477    https://movie.douban.com/subject/1395091/
179    模仿游戏    8.6    355368    https://movie.douban.com/subject/10463953/
180    一个叫欧维的男人决定去死    8.8    195643    https://movie.douban.com/subject/26628357/
181    忠犬八公物语    9.1    96747    https://movie.douban.com/subject/1959195/
182    燃情岁月    8.8    185142    https://movie.douban.com/subject/1295865/
183    人工智能    8.6    258634    https://movie.douban.com/subject/1302827/
184    魔女宅急便    8.6    281054    https://movie.douban.com/subject/1307811/
185    房间    8.8    229618    https://movie.douban.com/subject/25724855/
186    穿越时空的少女    8.6    250139    https://movie.douban.com/subject/1937946/
187    天书奇谭    9.1    91667    https://movie.douban.com/subject/1428581/
188    恐怖游轮    8.4    459129    https://movie.douban.com/subject/3011051/
189    魂断蓝桥    8.8    164770    https://movie.douban.com/subject/1293964/
190    黑客帝国3：矩阵革命    8.7    225709    https://movie.douban.com/subject/1302467/
191    海街日记    8.7    214152    https://movie.douban.com/subject/25895901/
192    猜火车    8.5    306564    https://movie.douban.com/subject/1292528/
193    罗生门    8.8    175470    https://movie.douban.com/subject/1291879/
194    完美陌生人    8.5    333893    https://movie.douban.com/subject/26614893/
195    阿飞正传    8.5    295833    https://movie.douban.com/subject/1305690/
196    头号玩家    8.7    823877    https://movie.douban.com/subject/4920389/
197    香水    8.5    362383    https://movie.douban.com/subject/1760622/
198    功夫    8.4    465611    https://movie.douban.com/subject/1291543/
199    可可西里    8.7    171199    https://movie.douban.com/subject/1308857/
200    朗读者    8.5    333438    https://movie.douban.com/subject/2213597/
201    谍影重重2    8.6    219036    https://movie.douban.com/subject/1308767/
202    浪潮    8.7    173378    https://movie.douban.com/subject/2297265/
203    牯岭街少年杀人事件    8.8    149429    https://movie.douban.com/subject/1292329/
204    谍影重重    8.6    261073    https://movie.douban.com/subject/1304102/
205    战争之王    8.6    224942    https://movie.douban.com/subject/1419936/
206    地球上的星星    8.9    120790    https://movie.douban.com/subject/2363506/
207    疯狂的石头    8.4    452026    https://movie.douban.com/subject/1862151/
208    初恋这件小事    8.3    633902    https://movie.douban.com/subject/4739952/
209    青蛇    8.5    303900    https://movie.douban.com/subject/1303394/
210    惊魂记    8.9    120359    https://movie.douban.com/subject/1293181/
211    终结者2：审判日    8.6    198690    https://movie.douban.com/subject/1291844/
212    源代码    8.4    520617    https://movie.douban.com/subject/3075287/
213    爱在午夜降临前    8.8    167120    https://movie.douban.com/subject/10808442/
214    步履不停    8.8    139427    https://movie.douban.com/subject/2222996/
215    新龙门客栈    8.5    258004    https://movie.douban.com/subject/1292287/
216    奇迹男孩    8.6    323039    https://movie.douban.com/subject/26787574/
217    小萝莉的猴神大叔    8.5    271364    https://movie.douban.com/subject/26393561/
218    追随    8.9    108391    https://movie.douban.com/subject/1397546/
219    一次别离    8.7    168140    https://movie.douban.com/subject/5964718/
220    无耻混蛋    8.5    294062    https://movie.douban.com/subject/1438652/
221    再次出发之纽约遇见你    8.5    245335    https://movie.douban.com/subject/6874403/
222    釜山行    8.4    561902    https://movie.douban.com/subject/25986180/
223    血钻    8.6    196331    https://movie.douban.com/subject/1428175/
224    东京物语    9.2    72955    https://movie.douban.com/subject/1291568/
225    撞车    8.6    219372    https://movie.douban.com/subject/1388216/
226    彗星来的那一夜    8.5    276087    https://movie.douban.com/subject/25807345/
227    城市之光    9.3    65047    https://movie.douban.com/subject/1293908/
228    2001太空漫游    8.7    149476    https://movie.douban.com/subject/1292226/
229    梦之安魂曲    8.7    144757    https://movie.douban.com/subject/1292270/
230    新世界    8.7    149430    https://movie.douban.com/subject/10437779/
231    绿里奇迹    8.7    145109    https://movie.douban.com/subject/1300374/
232    疯狂的麦克斯4：狂暴之路    8.6    296326    https://movie.douban.com/subject/3592854/
233    聚焦    8.8    172697    https://movie.douban.com/subject/25954475/
234    E.T. 外星人    8.5    211210    https://movie.douban.com/subject/1294638/
235    这个男人来自地球    8.5    244977    https://movie.douban.com/subject/2300586/
236    末路狂花    8.7    138086    https://movie.douban.com/subject/1291992/
237    黑鹰坠落    8.6    169479    https://movie.douban.com/subject/1291824/
238    发条橙    8.5    237973    https://movie.douban.com/subject/1292233/
239    遗愿清单    8.6    191251    https://movie.douban.com/subject/1867345/
240    变脸    8.4    272586    https://movie.douban.com/subject/1292659/
241    勇闯夺命岛    8.6    182137    https://movie.douban.com/subject/1292728/
242    国王的演讲    8.4    435891    https://movie.douban.com/subject/4023638/
243    我爱你    9.0    84663    https://movie.douban.com/subject/5908478/
244    黄金三镖客    9.1    71082    https://movie.douban.com/subject/1401118/
245    千钧一发    8.7    131207    https://movie.douban.com/subject/1300117/
246    非常嫌疑犯    8.6    163295    https://movie.douban.com/subject/1292214/
247    秒速5厘米    8.3    412652    https://movie.douban.com/subject/2043546/
248    驴得水    8.3    502838    https://movie.douban.com/subject/25921812/
249    卡萨布兰卡    8.6    160541    https://movie.douban.com/subject/1296753/
250    四个春天    8.9    87331    https://movie.douban.com/subject/27191492/
```
