# 大数据采集与处理实验

- 姓名：韩双宇
- 学号：112404260106
- 班级：数据1241
- GitHub：hsy112404260106

## 仓库结构

```text
bigdata-collection-labs-112404260106/
├── README.md
└── lab01/
    ├── lab1_html_api.ipynb      # 实验一完整代码
    ├── REPORT.md                # 实验一实验报告
    ├── data/
    │   ├── companies.json       # 采集到的 60 家企业
    │   └── devices.json         # 采集到的 5000 台设备
    └── images/                  # 4 张证据截图
        ├── 01_device_api.png
        ├── 02_company_source.png
        ├── 03_company_next.png
        └── 04_validation.png
```

## Lab 1 概要

| 数据源 | 加载方式 | 采集方式 | 结果 |
|---|---|---|---|
| Device（`/devices`） | 动态加载（API） | `GET /api/devices?page=N&page_size=50` | 5000 条 |
| Company（`/companies`） | 静态加载（HTML） | BeautifulSoup + “下一页”翻页 | 3 页 60 条 |
