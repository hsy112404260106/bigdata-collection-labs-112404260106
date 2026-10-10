# Lab 1 实验报告

## 学生信息

- **姓名：**韩双宇
- **学号：**112404260106
- **班级：**数据1241
- **GitHub 用户名：**hsy112404260106
- **仓库地址：**https://github.com/hsy112404260106/bigdata-collection-labs-112404260106

> 文件位置：`lab01/REPORT.md`
> 完整代码保存在 `lab1_html_api.ipynb`。
> 本报告只填写关键发现、关键代码、结果和指定截图，**不要把整份 Notebook 复制进来**。

---

# 任务 1：判断数据加载方式

## 1.1 Device：判断动态加载并找到 API

### 判断结果

- 页面：`/devices`
- 数据加载方式：**动态加载**
- 判断依据：

请用 1～2 句话说明：

1. 在“查看网页源代码”中是否能搜索到页面上的设备名称；
   答：在“查看网页源代码”中搜索页面上已经显示的设备编号 `D00000001` 搜索不到，说明设备记录不是随初始 HTML 一起返回的。

2. 在 Network 的 Fetch/XHR 中是否找到返回设备数据的请求。
   答：打开后并刷新页面后，可以看到一条返回设备数据的请求，因此为动态加载。

### API 信息

- Request URL：`https://115.159.159.82/api/devices?page=1&page_size=50`
- Request Method：`GET`
- Query 参数：`page=1`、`page_size=50`

### 【截图 1】Device API 发现证据

文件名：

```text
images/01_device_api.png
```

**截图内容必须是浏览器 DevTools，不是 Jupyter 代码。**

截图中应能看清：

1. 浏览器的 **Network** 面板；
2. 已选中真正返回设备列表的 Fetch/XHR 请求；
3. 右侧 **Headers** 中的 Request URL；
4. Query String Parameters（或 URL 中的分页参数）。

> 不要截整个桌面；尽量裁剪到 Network 请求和右侧 Headers 区域。

插入：

![Device API 发现证据](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab01/images/01_device_api.png)


---

## 1.2 Company：判断静态加载

- 页面：`/companies`
- 数据加载方式：**静态加载**
- 判断依据：

请说明：使用“右键 → 查看网页源代码”后，是否能搜索到页面中真实显示的企业名称。
答：能找到，可按静态页面处理。

### 【截图 2】Company 静态加载证据

文件名：

```text
images/02_company_source.png
```

**截图内容必须是“查看网页源代码”页面，不是 Elements，也不是 Jupyter。**

截图中应能看清：

1. 浏览器正在显示 `/companies` 的网页源代码；
2. 使用 `Ctrl + F` 搜索一个页面中真实存在的企业名称；
3. 源代码中能看到该企业名称或对应企业记录。

插入：

![Company 静态加载证据](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab01/images/02_company_source.png)

---

# 任务 2：Device API 分页采集

## 2.1 API 分页结构

填写你实际从 JSON 中观察到的字段：

| 含义 | JSON 字段 |
|---|---|
| 当前页 | `page` |
| 每页条数 | `page_size` |
| 设备总数 | `total` |
| 设备列表 | `items` |

---

## 2.2 关键代码

只填写分页采集的核心部分，不要粘贴整个 Notebook。

```python
# 从第一页响应中读取分页信息
total = data["total"]
page_size = data["page_size"]

total_pages = (total + page_size - 1) // page_size

devices = []

for page in range(1, total_pages + 1):
    r = requests.get(
        api_url,
        params={
            "page": page,
            "page_size": page_size,
        },
        timeout=10
    )

    page_data = r.json()

    # 将本页设备加入 devices
    devices.extend(page_data["items"])

print("device count:", len(devices))
```

---

## 2.3 采集结果

- 每页条数：`50`
- 设备总数：`5000`
- 程序计算得到的总页数：`100`
- 最终采集设备数：`5000`
- 唯一 `device_id` 数：`5000`

回答：

> 为什么总页数应由 API 返回的 `total` 和 `page_size` 计算，而不是直接在代码里写死？

答：因为 total（数据总量）和 page_size（每页条数）都是由服务端返回的运行期数据，
数据会随业务增长而变化。写死页数（例如直接写 range(1, 101)）一旦数据量增加就会漏采，
数据量减少则会产生无用请求甚至报错。用 (total + page_size - 1) // page_size
动态计算，可以保证无论数据量怎么变，分页范围都刚好覆盖全部数据，代码也更通用。

---

# 任务 3：Company HTML 解析

## 3.1 企业记录 Selector

- 企业记录 selector：`article.company-card`
- 第 1 页企业记录数：`20`

说明为什么选择这个 selector：

- 主要使用了 `id / class / data-* / 父子层级` 中的哪一种？
- 是否参考过浏览器 Copy selector 或 AI？
- 最终为什么采用现在这个写法？

答：主要使用了 class（company-card）配合标签名 article。
观察过 Copy selector，但它给出的是很长的 nth-child(...) 路径，因此没有采用；最终是根据 DOM 结构自己写的。
因为 class="company-card" 语义明确、在本页中唯一标识“一条企业记录”这个最外层节点，且不依赖它在父节点中是第几个子节点。
相比之下 nth-child 形式的路径一旦页面插入广告位或调整顺序就会失效。
这条 selector 只有“标签 + 一个 class”，短且稳定，符合“优先短、语义明确、对结构变化不敏感”的原则。

---

## 3.2 关键代码

只填写“记录定位 + 一条记录的关键字段”。

```python
cards = soup.select("article.company-card")

card = cards[0]

company_id = card["data-company-id"]                        # 来自 HTML 属性 data-*
name = card.select_one("h2 a").get_text(strip=True)         # 来自文本
status = card.select_one(".badge").get_text(strip=True)
```

行业、区域、设备数量使用 `dt/dd` 结构后，请写出你最终如何得到：

```python
industry = details["所属行业"]
zone = details["所在区域"]
device_count = int(details["设备数量"].replace("台", "").strip())
```

---

## 3.3 字段来源说明

任选 **2 个字段**说明：

1. 字段：`company_id`
   - 来自：**HTML 属性**
   - 使用的方法：企业卡片的 `<article class="company-card" data-company-id="C000001">`
     上带有 data-* 属性，直接用 `card["data-company-id"]` 取出属性值，
     不需要再从文本里做正则提取。

2. 字段：`device_count`
   - 来自：**文本**
   - 使用的方法：先通过 `dl > div > dt/dd` 结构取到 设备数量 对应的文本 "6 台"，
     用 `replace("台", "").strip()` 去掉单位，再用 `int()` 转成整数，
     以便后面和设备表统计出来的实际数量做数值比较。

---

# 任务 4：Company “下一页”分页

## 4.1 下一页元素

- “下一页” selector：`nav.pagination a:last-child`
- 第 1 页 `href` 示例：`/companies?page=2`

### 【截图 3】Company “下一页”元素

文件名：

```text
images/03_company_next.png
```

**截图内容必须是浏览器 DevTools 的 Elements 面板。**

截图中应能看清：

1. 页面底部“下一页”对应的 `<a>` 元素被选中；
2. 该元素的 `class`、`href` 等关键属性；
3. 能判断你为什么使用当前 selector。

> 不需要截 Python 代码；这一张图只证明你找到了正确的 DOM 元素。

> 说明：本截图是在第 2 页截取的，此时“下一页”的 `href` 为 `/companies?page=3`
> （第 1 页时对应为 `/companies?page=2`）。`nav.pagination a:last-child`
> 这一 selector 在每一页都同样适用。

插入：

![Company 下一页元素](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab01/images/03_company_next.png)


---

## 4.2 关键代码

```python
next_link = soup.select_one("nav.pagination a:last-child")

next_href = next_link.get("href")

next_url = urljoin(current_url, next_href)
```

再用 2～3 句话说明：

> 你的程序如何完成“当前页 → 找下一页 href → 请求下一页 → 再解析”的循环？

答：程序用一个 `while` 循环维护 `current_url`。每一轮先 `requests.get(current_url)`
取回当前页 HTML，用 `soup.select("article.company-card")` 解析出本页全部企业并追加进
`companies`；随后在同一份 `soup` 里用 `nav.pagination a:last-child` 找“下一页”链接，
用 `urljoin` 把相对 `href` 拼成绝对 URL 作为下一轮的 `current_url`。
循环条件是“页码计数 ≤ 3 且还能找到下一页”，因此在采满 3 页或翻到最后一页时自动停止。

---

## 4.3 三页采集结果

| 页码 | 企业数 |
|---|---:|
| 1 | 20 |
| 2 | 20 |
| 3 | 20 |
| **合计** | **60** |

- 唯一 `company_id` 数：`60`

---

# 任务 5：结果验证

## 5.1 最终结果

| 检查项 | 结果 |
|---|---:|
| Company 总数 | 60 |
| 唯一 `company_id` | 60 |
| Device 总数 | 5000 |
| 唯一 `device_id` | 5000 |
| Company–Device 不一致企业数 | 0 |

### 【截图 4】最终验证结果

文件名：

```text
images/04_validation.png
```

**截图内容必须是 JupyterLab 的运行结果，不是代码截图，也不是浏览器截图。**

截图中应能同时看清以下输出：

```text
Company 总数
唯一 company_id 数
Device 总数
唯一 device_id 数
Company–Device 不一致企业数
```

建议把这几个 `print()` 放在同一个 Notebook 单元格中运行后截图。

插入：

![最终验证结果](https://gcore.jsdelivr.net/gh/hsy112404260106/bigdata-collection-labs-112404260106@main/lab01/images/04_validation.png)

---

## 5.2 一致性检查关键代码

只填写最关键的比较逻辑：

```python
from collections import Counter

device_counts = Counter(
    x["company_id"] for x in devices
)

mismatch_count = 0

for company in companies:
    cid = company["company_id"]
    expected = company["device_count"]          # 页面上写的设备数
    actual = device_counts.get(cid, 0)

    if expected != actual:
        mismatch_count += 1
```

回答：

> 为什么 `status_code == 200` 仍然不能说明“数据已经采集正确”？

答：`status_code == 200` 只说明这次 HTTP 请求在**传输层**成功了，它并不保证
响应体里的内容是完整的、正确的。例如分页参数写错导致只拿到某一页、
`items` 字段取错拿到空列表、翻页时漏掉最后一页、
或者解析字段时定位错节点导致某个字段为空，这些情况下服务器依然会返回 200。
因此除了状态码，还必须校验**业务层**的指标：采集条数是否等于 `total`、
主键（`device_id` / `company_id`）是否唯一、以及跨数据源的一致性
（Company 页面的 `device_count` 与 Device 表统计值是否吻合）。

---

# 任务 6：保存采集结果

确认已经生成：

```text
data/companies.json
data/devices.json
```

填写：

- `companies.json`：**已生成**
- `devices.json`：**已生成**

不需要截图 JSON 全文，也不需要把 JSON 内容复制到报告。

---

# 实验结论

用 **80～150 字**总结：

1. 静态加载与动态加载页面的处理方式有什么不同；
2. HTML “下一页”分页和 API `page` 分页有什么不同；
3. 你使用了哪些方法验证采集结果。

答：静态页面的数据随 HTML 一起返回，直接 requests + BeautifulSoup 按 CSS Selector
解析即可；动态页面初始 HTML 里没有数据，必须先用 Network 找到 API，再请求 JSON
自行分页。HTML 的“下一页”要解析 DOM 中的 href 逐页翻，页数由页面决定；
API 的 `page` 分页只需改参数，总页数由 `total` 和 `page_size` 精确算出。
验证用条数、主键唯一性和 Company–Device 一致性比对。
