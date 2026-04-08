# Agent Skill





### 例子 1：订机票 Skill

##### 你有这些 Tool（小工具）：

1. search_flight（查航班）
2. get_price（查价格）
3. book_ticket（订票）
4. send_sms（发短信）

##### 但你现在需要：

> “帮我订一张明天从北京去上海的最便宜机票”

这**不是一个 Tool 能搞定**，需要：

查航班 → 比价 → 选最便宜 → 订票 → 发短信

这一整套流程，就是一个 **Skill**

##### 这个 Skill 名字叫：

**book_flight_skill**

##### 它里面写的是：

1. 用户要从 A 到 B，日期 X
2. 调用 search_flight 查所有航班
3. 调用 get_price 比价
4. 选最便宜的
5. 调用 book_ticket 订票
6. 调用 send_sms 通知用户

------

### 例子 2：代码审查 Skill

##### Tool 有：

- read_file（读代码）
- run_lint（语法检查）
- search_security（查漏洞）

##### 需求：

> “帮我检查这段代码有没有 Bug 和安全问题”

##### Skill 名字：

**code_review_skill**

##### 它做的事：

1. 读文件
2. 检查语法
3. 查安全漏洞
4. 生成报告
5. 给出修改建议

------

### 例子 3：外卖点餐 Skill

#### Tool：

- search_restaurant（搜餐厅）
- get_menu（拿菜单）
- calculate_price（算钱）
- place_order（下单）

#### Skill：

**order_food_skill**

流程：

搜店 → 看菜单 → 算价格 → 下单

### 总结

**Tool 是单个动作，Skill 是一整套任务流程。**

**Agent 靠 Skill 完成复杂任务，Skill 靠 Tool 真正动手。**

