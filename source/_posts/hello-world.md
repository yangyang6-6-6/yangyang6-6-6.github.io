---
title: 课堂点名系统 — 统计与数据分析模块开发记录
date: 2026-06-26 20:00:00
---

# 基于有状态的课堂点名系统

> 面向对象程序设计课程设计

---

## 项目简介

本系统是一个面向高校课堂教学的辅助工具，旨在解决传统随机点名中"有的同学被点名多次、有的同学从未被点到"的不公平问题。系统通过记录每位学生的历史被点名次数和回答问题次数，采用加权随机算法，确保每位学生获得均等的点名机会。当连续多人无法回答问题时，自动启动救场机制，避免课堂僵局。

---

## 技术栈

| 类别 | 具体技术 |
|------|---------|
| 开发语言 | Java 8+ |
| GUI框架 | Java Swing (JFrame / JTabbedPane / JTable) |
| 数据持久化 | Java对象序列化 (ObjectOutputStream / ObjectInputStream) |
| 第三方库 | Apache POI 5.3.0 (Excel读写) |
| 架构模式 | MVC三层架构 + DAO数据访问模式 |
| 项目管理 | Git + GitHub |

---

## 系统架构

![](/images/architecture.svg)

*黄色标记为我负责的模块*

---

## 我负责的模块

### 1. 共享模型

**Student.java** 学生实体

```java
public class Student implements Serializable {
    private String studentNo;    // 学号
    private String name;         // 姓名
    private String className;    // 班级
    private int totalCalled;     // 被点名总次数
    private int totalAnswered;   // 答出总次数
    private boolean onLeave;     // 是否请假

    public double getAnswerRate() {
        if (totalCalled == 0) return 0.0;
        return (double) totalAnswered / totalCalled * 100;
    }
}
```

**RollCallRecord.java** 点名记录

```java
public class RollCallRecord implements Serializable {
    private String studentNo;     // 学号
    private String studentName;   // 学生姓名
    private String courseName;    // 课程名
    private boolean answered;     // 是否答出
    private String callTime;      // 点名时间
}
```

### 2. 统计服务 — StatisticsService.java

**汇总统计**

使用 `LinkedHashMap` 收集四项核心指标，保持插入顺序：

```java
public Map<String, Object> getSummary() {
    List<Student> students = dao.loadStudents();
    List<RollCallRecord> records = dao.loadRecords();

    Map<String, Object> map = new LinkedHashMap<>();
    map.put("totalStudents", students.size());
    map.put("totalCalls", records.size());

    long answered = records.stream()
        .filter(RollCallRecord::isAnswered)
        .count();
    map.put("totalAnswered", answered);

    double rate = records.isEmpty() ? 0 :
        (double) answered / records.size() * 100;
    map.put("overallRate", String.format("%.1f", rate));
    return map;
}
```

**点名频次分布**

按被点名次数将学生分组，五个区间：0次 / 1~3次 / 4~6次 / 7~10次 / 10次以上。使用 `LinkedHashMap` 确保分组顺序固定。

```java
Map<String, Integer> dist = new LinkedHashMap<>();
dist.put("0次", 0);
dist.put("1~3次", 0);
dist.put("4~6次", 0);
dist.put("7~10次", 0);
dist.put("10次以上", 0);

for (Student s : students) {
    int n = s.getTotalCalled();
    if (n == 0) dist.merge("0次", 1, Integer::sum);
    else if (n <= 3) dist.merge("1~3次", 1, Integer::sum);
    else if (n <= 6) dist.merge("4~6次", 1, Integer::sum);
    else if (n <= 10) dist.merge("7~10次", 1, Integer::sum);
    else dist.merge("10次以上", 1, Integer::sum);
}
```

### 3. 统计界面 — StatisticsPanel.java

界面顶部为四张汇总卡片，使用 `GridLayout(1, 4)` 布局：

- 学生总数
- 总点名次数
- 总答出次数
- 总体成功率

中间为 `JTable` 排名表，按被点名次数降序排列：

```java
students.sort((a, b) -> b.getTotalCalled() - a.getTotalCalled());
```

表格列：排名、学号、姓名、班级、被点名、答出、成功率。

底部为刷新按钮，点击后重新加载所有数据。

```java
JPanel cards = new JPanel(new GridLayout(1, 4, 15, 0));

String[] cols = {"排名", "学号", "姓名", "班级",
                 "被点名", "答出", "成功率"};
```

---

## 项目亮点

1. **加权随机算法** — 权重 = 100 / (被点名次数 + 1)，累积权重法选择，确保点名机会均等
2. **双重救场机制** — 连续 3 人未答出时从高回答率学生中选取；再连续 3 人失败时提示"题目太难"
3. **请假管理** — 标记请假学生，点名时自动排除
4. **MVC 架构** — 界面、业务逻辑、数据访问三层分离

---

## 解答问题

### Q1: 为什么用 LinkedHashMap 而不是 HashMap？

频次分布需要按固定顺序展示（0次 → 1~3次 → 4~6次 → 7~10次 → 10次以上），HashMap 不保证顺序，而 LinkedHashMap 维护了插入顺序，确保每次展示分组顺序一致。

### Q2: Stream API 在这段代码中起了什么作用？

```java
long answered = records.stream()
    .filter(RollCallRecord::isAnswered)
    .count();
```

一行代码完成"遍历集合 → 过滤出已答出记录 → 计数"的全过程，比传统 for 循环加 if 判断更简洁，语义也更清晰。

### Q3: Lambda 表达式简化了什么？

```java
students.sort((a, b) -> b.getTotalCalled() - a.getTotalCalled());
```

传统写法需要创建一个 Comparator 匿名内部类，Lambda 用一句话表达"按被点名次数降序排列"，代码量减少一半以上。

### Q4: 成功率是怎么计算的？

```java
public double getAnswerRate() {
    if (totalCalled == 0) return 0.0;
    return (double) totalAnswered / totalCalled * 100;
}
```

如果学生从未被点名过（分母为 0），直接返回 0 避免除零异常；否则用答出数除以被点名数再乘以 100，得出百分比形式的成功率。

### Q5: MVC 架构在这个模块中如何体现？

- **Model**：Student（学生数据）、RollCallRecord（点名记录）— 承载数据
- **View**：StatisticsPanel（统计界面）— 负责数据可视化展示
- **Controller/Service**：StatisticsService（统计服务）— 业务逻辑计算，介于 Model 和 View 之间

---

## 界面截图

以下截图来自个人 GitHub 上传的代码运行效果：

![](/images/statistics-code.png)

![](/images/statistics-panel.png)

---

## 总结

通过这个项目，我对 Java Swing 桌面应用开发、Stream API 数据处理、MVC 架构设计有了更深入的理解。该模块完整覆盖了从数据采集、计算到可视化展示的全链路流程。

> Java 8+ | Swing | Stream API | Lambda | Apache POI | Git
>
> 课程设计 2026

---

## 项目仓库

- **完整项目仓库**：[github.com/qluliuliuliu/classroom-rollcall-system](https://github.com/qluliuliuliu/classroom-rollcall-system)
- **个人Git仓库**：[github.com/yangyang6-6-6/Java-](https://github.com/yangyang6-6-6/Java-)
