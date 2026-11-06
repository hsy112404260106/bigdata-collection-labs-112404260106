# 实验二报告：房源采集与 MySQL 存储

- 班级：数据1241
- 姓名：韩双宇
- 学号：112404260106
- 实验日期：2026-10-27
- GitHub 仓库链接：https://github.com/hsy112404260106/bigdata-collection-labs-112404260106.git
- 源码文件名：`lab02/anjuke.ipynb`
- 数据库与表：`smartpark_lab.houseinfo`
- 数据来源：在线教学站
- 访问时间及结果（若失败，填写状态码、页面标题或报错）：2026-10-27 16:10 左右，状态码 200，网页标题「重庆二手房房产网，重庆二手房出售，重庆买房购房交易信息 - 58安居客」，正常返回房源列表。注意：请求头只写 `Mozilla/5.0` 时会返回「访问验证」页（详见"问题与解决"）。
- 快照文件来源与文件名（在线实验可写"不适用"）：不适用（在线采集成功）

> 填写要求：截图放在 `images/` 中，在对应位置插入图片。代码截图要显示文件名、行号或单元位置，用红框标出关键代码，每项写2—3句话解释。只截最终结果或只贴 AI 生成的文字，不能代替实验过程。未完成的项目写明原因，不填写预期结果，不展示个人账号密码。

## 第一阶段：MySQL Docker 更新与启动

- 本次是否需要更新：已完成更新
- 更新或启动遇到的问题：无。`start-course.bat` 打开 JupyterLab，`start-mysql.bat` 显示 `[PASS] L05 MySQL healthy`，`mysql-console.bat` 输入密码后出现 `mysql>`。

### 启动环境与建表

依次使用 `start-course.bat`、`start-mysql.bat`、`mysql-console.bat` 启动课程环境、数据库和 MySQL 终端。

**截图：** MySQL 启动成功提示，以及终端中 `USE smartpark_lab`、`SHOW TABLES`、`DESCRIBE houseinfo` 的执行结果。可分图。

![DESCRIBE houseinfo 表结构与入库前条数](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/mysql.png)

说明：当前使用哪个数据库？`id` 和8个房源字段分别保存什么？

答：当前使用 `smartpark_lab` 数据库。`id` 是数据库自动生成的自增长主键，不参与采集；其余8列分别保存：`title` 房源卖点标题、`house` 楼盘名称、`address` 完整地址、`struct` 户型（如 3室2厅1卫）、`area` 建筑面积（带单位）、`floor` 楼层、`maketime` 建造年代、`price` 每平方米单价（带单位）。本次实验开始前表中为空（before_count = 0），`houseinfo` 表已存在，无需重新建表。

## 第二阶段：房源采集与 CSV 数据存储

### 1. 网页分析与房源解析

#### 选择器是怎样得到的

在 Chrome 中按 F12，使用 **Elements → Copy → Copy selector**。选一个字段，记录从完整路径到相对选择器的过程。

- 选择的字段：标题 title（第一条房源的 h3）
- 复制的原始选择器：`#content > div.property-content > div.property-content-detail > div.property-content-title > h3.property-content-title-name`（一长串从页面根节点出发的路径，含多层 `>`）
- 一条房源的父容器选择器：`div.property`
- 最终使用的相对选择器：标题 `h3`；楼盘 `p.property-content-info-comm-name`；地址 `p.property-content-info-comm-address`；户型 `p.property-content-info-attribute`；面积/楼层/年代 在 `p.property-content-info-text` 属性组中按内容特征识别；单价 `p.property-price-average`

**截图：** 框出该字段的 HTML 节点和房源父容器，展示选择器的匹配结果。

![房源父容器 div.property 高亮，刚好包住一条房源](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/pares-fileds.png)

![在 h3 节点上右键 复制 selector](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/pares-fileds2.png)

说明：删掉了哪些层级？怎样确认父容器刚好包含一条房源？

答：删掉了从 `#content` 根节点开始的全部上层路径，只保留"一条房源内部"的相对路径。确认方法：把鼠标移到 `div.property` 节点上，网页高亮区域刚好罩住一条房源（图片、标题、属性、楼盘地址、价格都在里面），且第二条房源使用相同的 class，页面共 60 个 `div.property`，与每页显示的房源条数一致。

#### 解析8个字段

**截图：** 自己编写的解析代码和首条房源的8个字段输出。代码中必须框出：

- `select()` 和 `select_one()`。
- 面积、单价的定位与取值。
- 建造年代的提取和缺失处理。

可分成几张局部截图。另附面积、单价对应的 HTML 节点或网页原文。

![首条房源字段观察：属性组顺序、总价与单价](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/parse-fields.png)

![parse_property 解析函数与首条8字段输出](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/parse-fn.png)

说明：两个选择方法各用在哪里？怎样确认面积和单价没有取错？

答：`select("div.property")` 用于整页——返回全部房源容器列表；`select_one()` 用于一条房源内部——每个字段只取第一个匹配节点，取不到时返回 None，代码先判空再取文字，避免下标越界。面积按内容特征识别（含 `㎡`），单价用 `p.property-price-average`（文本形如 `16465元/㎡`）；旁边的 `p.property-price-total` 是"148万"总价，不能取错。首条解析结果与网页逐字段对照一致（89.9㎡ / 16465元/㎡ / 2019年建造）。户型文字分散在多个子 span 中，用 `get_text(strip=True)` 拼成 `3室2厅1卫`，不额外插入空格；地址分散在多个 span 中，同样直接取完整文字。

### 2. 下一页采集

- 下一页 `href` 原值：`?page=2`（相对地址，来自 `div.pagination a.next.next-active`）
- 用 `urljoin()` 补全后的地址：`http://www.bspider.top/anjuke/?page=2`（依次翻到 `?page=10`）
- 实际停止原因（明确禁用的末页按钮 / 异常中止）：第 10 页解析完 60 条后，`a.next.next-active` 选择器取不到节点（末页"下一页"按钮已禁用），判定到末页，正常停止。

**截图：** 框出提取 `href`、补全地址、循环更新地址和停止条件的代码。附前两页的运行日志，以及最终停止提示。离线实验展示快照中的href、补全结果和文件读取记录，注明"离线回放"，不算在线翻页成功。

![分页采集日志：10 页逐页累计，末页正常停止](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/pagination-log.png)

说明：怎样进入下一页？怎样防止重复采集或无限循环？若中途失败，说明停在哪一步。

答：每页解析完后用 `select_one('div.pagination a.next.next-active')` 找可点的"下一页"按钮，`get('href')` 取相对地址，`urljoin(current_url, href)` 补全后作为下一轮的请求地址。防重复：把访问过的地址放进 `visited` 集合，重复即停止；防越界：下一页地址必须包含 `bspider.top`；防死循环：设 `MAX_PAGES = 30` 安全上限；每页之间 `time.sleep(1.2)` 间隔 ≥1 秒。本次运行完整翻完 10 页，共 600 条，正常停止，无中途失败。

### 3. 保存 CSV

**截图：** CSV保存与回读代码、回读条数。框出编码、表头和数据写入部分。

![CSV 保存：utf-8-sig、表头、逐条写入](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/csv-save.png)

![DictReader 回读核对：498 行与三条样本](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/csv-read.png)

说明：8个字段的顺序是什么？按8个字段完全相同去重后，保存了多少条记录？

答：字段顺序为 `title, house, address, struct, area, floor, maketime, price`，与 README 要求一致。用 `csv.writer` 写入，编码 `utf-8-sig`、`newline=""`，第一行是表头。采集 600 条中按8字段完全相同去重 102 条（网站列表存在跨页重复房源），去重后保存 498 条。用 `csv.DictReader` 回读核对：数据行 498 行（不含表头），表头与8字段完全一致；抽第1条、第250条、第498条与网页逐字段核对无误。

## 第三阶段：MySQL 数据存储

### 1. 读取 CSV 并入库

本阶段读取第二阶段的 `data/houseinfo.csv`，不重新采集。

**截图：** 框出读取CSV、按8个字段整理入库参数的代码。

![读取 CSV 并按8字段固定顺序整理入库参数](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/db-read-csv.png)

**截图：** 自己编写的入库代码。框出 `executemany()`、`commit()`、`rollback()`，以及关闭游标和连接的操作。另框出避免重复插入的代码。

![executemany 批量入库、commit/rollback 与防重复插入](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/db-insert.png)

说明：每条记录怎样对应8个参数？成功、失败时分别做什么？再次运行时怎样避免重复插入？

答：INSERT 语句写明8个列名和8个 `%s` 参数位置，`records` 是由 CSV 每行按 `FIELDS` 固定顺序取出的8元组列表，顺序与 `%s` 一一对应，数据通过参数传入而不是字符串拼接。写入流程：先用 `executemany` 试插 3 条真实记录并 `commit()`，确认无误后再批量插入剩余 495 条并 `commit()`；任何一步抛异常就 `rollback()` 撤销本次事务，`finally` 中先 `cur.close()` 再 `conn.close()` 释放资源。防重复插入：入库前先 `SELECT` 已有记录的8字段组合存入集合，只插入集合中不存在的记录——重跑同批数据时"本次待插入 0 条"，不会重复增加。

### 2. 核对结果

#### 查询条数和样本

**截图：** MySQL 终端中的入库前后 `SELECT COUNT(*)`、重跑后的条数，以及 `SELECT ... LIMIT 5` 的结果。

![入库后 SELECT COUNT 与前5条样本](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/db-count.png)

![重跑后：before_count=498，同批数据未重复插入](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/db-rerun-count.png)

| 项目 | 实际结果 |
|---|---|
| 成功采集页数 | 10 页 |
| 采集记录数（去重前） | 600 条 |
| 完全相同的重复记录数 | 102 条 |
| CSV数据行数（不含表头） | 498 条 |
| 入库前条数 | 0 条 |
| 入库后条数 / 本次新增条数 | 498 条 / 498 条 |
| 关闭上次程序后，同批数据独立重跑的条数 / 新增条数 | 498 条 / 0 条 |
| 是否采集完整；未完成的原因 | 完整。第 10 页"下一页"按钮已禁用，正常停止；未达到旧报告的 601 条属正常，以实际采集为准 |

说明：首次入库后的条数应等于去重后的CSV数据行数，8个字段的内容也应一致。若不一致，说明原因。总条数以实际结果为准。

答：首次入库后 498 条 = 去重后 CSV 498 行，一致。同批数据独立重跑一次：程序识别库中已有 498 条，"本次待插入 0 条"，重跑后仍是 498 条，新增 0 条，防重复逻辑有效。

#### 逐条核对

选取第一条、跨页一条和最后一条，比较网页、CSV与数据库中的8个字段。快照实验对照教师提供的网页文件。

**截图：** 三条样本的对照结果，框出面积、单价和年代。可分图。

![CSV 三条样本与数据库逐字段核对](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab02/images/db-sample.png)

- 样本1（页码或URL、记录位置）：第 1 页第 1 条 —— 万科物业自营 南坪六公里金域翰林（89.9㎡ / 低层(共27层) / 2019年建造 / 16465元/㎡）
- 样本2（页码或URL、记录位置）：CSV 第 250 条（约第 5 页，跨页）—— 精装修洋房 位置安静（协信城立方玺院，95.8㎡ / 中层(共8层) / 2016年建造 / 15658元/㎡）
- 样本3（页码或URL、记录位置）：CSV 第 498 条（末页最后一条）—— 恒大照母山 精装3房（90.2㎡ / 低层(共32层) / 2015年建造 / 15530元/㎡）

核对结论及差异：网页、CSV、数据库三方逐字段完全一致，无差异。

## 问题与解决

- 遇到的问题、处理方法及结果：① 按 README 示例用 `User-Agent: "Mozilla/5.0"` 请求时，返回的是「访问验证-安居客」页（状态码仍为 200，页面内容为"系统检测到您正在使用网页抓取工具"）。解决：改用完整浏览器 User-Agent（含 Windows/Chrome/Edg 完整串），并在代码里检查网页标题是否含"访问验证"，是则立即中止。改后正常返回房源列表。② 属性组中"南北"（朝向）与"低层(共27层)"（楼层）都是 `p.property-content-info-text`，顺序取下标容易错位。解决：按内容特征识别——含"建造"的是年代、含"㎡"的是面积、含"层"的是楼层，不按固定下标取值。
- 尚未解决的问题、未完成的步骤（没有可写"无"）：无。

## 思考题

1. 本次怎样实现下一页采集？结合自己的代码说明。

   答：每页解析完后，用 `select_one('div.pagination a.next.next-active')` 找处于可点状态的"下一页"按钮，`get('href')` 取到相对地址如 `?page=2`，再用 `urljoin(current_url, href)` 补全成 `http://www.bspider.top/anjuke/?page=2` 作为下一轮请求地址。停止条件有三层：按钮取不到（末页禁用）、地址访问过（visited 集合）、达到 MAX_PAGES=30 安全上限；另有域名白名单防止链接越出教学站。本次在第 10 页因"下一页"按钮禁用而正常停止。

2. 旧代码把 out_list 定义在全局。你把结果列表放在哪里？为什么？

   答：我把 `all_records` 定义在分页采集单元内部、循环开始之前，解析函数 `parse_property` 也只接收单条 `row` 参数并返回 dict。这样数据流是单向的：解析函数不依赖外部状态，采集列表的生命周期限定在采集单元里，不会被其他单元意外修改；后面 CSV、入库单元都从函数返回值/参数显式传递数据，逻辑更清晰，也便于单独重跑某一阶段。

3. 你怎样获取建造年代？节点缺失或顺序改变时，怎样处理？

   答：建造年代在房源属性组 `p.property-content-info-text` 中，我遍历属性组的每一项，按内容特征（含"建造"字样）识别，而不是按固定下标取第 5 项。节点缺失时：`parse_property` 开头先把 8 个字段初始化为空字符串，匹配不到就保持空串入库，不会报错也不会取到别的字段；顺序改变时因为不依赖下标，只要文本特征还在就能正确识别，属性组整体缺失也只会让该字段为空。


