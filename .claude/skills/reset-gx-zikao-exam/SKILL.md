---
name: reset-gx-zikao-exam
description: 按准考证号批量重置广西自考学生考试(UserQuiz batchReset)。生产环境写操作:先只读查询并展示待重置清单,用户回复"确认"后才调用重置接口,再通过 search 接口校验结果。
disable-model-invocation: true
---

# 任务:重置广西自考学生考试(UserQuiz batchReset)

用户在调用本 skill 时提供的参数见下表;未提供的按默认值处理,标注"缺失则问我"的必须询问用户。

## 角色与总原则
你是运维助手,负责按准考证号批量重置学生考试。这是**生产环境写操作**,必须:
1. 先只读查询,把待重置清单展示给我确认,**我回复"确认"后才能调用重置接口**。
2. 绝不输出、打印或写入文件任何数据库账号密码。
3. 所有 SQL 只做 SELECT;写操作只通过重置接口完成,结果校验只通过 search 接口完成。
4. 任何一步失败就停下来告诉我,不要自行重试或绕过。

## 一、输入参数
| 参数 | 说明 | 本次默认值 |
|---|---|---|
| 学校 | symbol 或 school id,二选一 | school id = `114357` |
| 学生范围 | ① 若干准考证号;或 ② Excel 路径(有一列准考证号,对应 `exam_ticket_num`) | 由我提供 |
| 过滤条件(可选) | 如"只重置考试时间在某时间之后的",**考试时间一律指 `start_time` 字段** | 无 |
| 目标状态 `status` | `0`=未开始(NOT_START),`1`=已保存未提交(SAVED) | **`1`(保存)**;我明确说"重置为未开始"才用 `0` |
| `resetSubmittedCount` | 仅 status=0 时有意义 | false |
| `resetPassFaceRecognition` | 仅 status=0 时有意义 | false |
| `resetUserId` | 操作人用户 ID(Integer) | 固定为 `0` |
| `platform` | 操作来源平台(String) | 固定为 `gxzk` |
| 导出路径 | 涉及导出时使用 | `E:\temp-file`(我不指定时) |

## 二、连接信息(账号密码一律读环境变量)
- 用户名:`PROD_MYSQL_USER`,密码:`PROD_MYSQL_PW`
- global 库:
  `jdbc:mysql://qs-pro-inst1.chlhkgnyqqkn.rds.cn-north-1.amazonaws.com.cn:3306/global?useSSL=false&allowMultiQueries=true&rewriteBatchedStatements=true&cacheServerConfiguration=true&useLocalSessionState=true&useUnicode=yes&characterEncoding=UTF-8`
- npdb 库:
  `jdbc:mysql://qs-pro-inst1.chlhkgnyqqkn.rds.cn-north-1.amazonaws.com.cn:3306/npdb?useSSL=false&allowMultiQueries=true&rewriteBatchedStatements=true&cacheServerConfiguration=true&useLocalSessionState=true&useUnicode=yes&characterEncoding=UTF-8`
- 先检查两个环境变量是否存在(只判断是否为空,不要回显值);缺失则停止并提示我。

## 三、执行步骤

### Step 1:确定 school_id
- 给的是 **symbol**:连 global 库,`SELECT id FROM school WHERE symbol = ?`,得到 school_id。查不到或返回多条则停止。
- 给的是 **id**:直接作为 school_id。

### Step 2:获取学校库连接
连 npdb 库:
`SELECT url, schema_name FROM school_db_setting WHERE school_id = ?`
用 `url` + `schema_name` 拼出学校库连接(沿用上面的 JDBC 参数、同一组环境变量账号密码)。

### Step 3:解析学生清单
- 若给的是 Excel:读取准考证号列(列名不明确时先列出表头问我),去除空值、首尾空格并去重,告诉我有效条数。
- 在学校库先用 `DESCRIBE` / `SHOW COLUMNS` 确认 `student`、`examination_statistic` 的字段,不要凭猜测写字段名。
- 通过 `student.exam_ticket_num` 找到学生,再关联 `examination_statistic` 取得 `user_quiz_id`。
  - 关联键以 DESCRIBE 结果为准。
  - 考试时间过滤使用 `start_time`(优先 `examination_statistic.start_time`;若该表无此字段,再确认其他表并告诉我)。例如"2026-10-01 之后" => `start_time >= '2026-10-01 00:00:00'`,边界(含/不含)不明确时问我。
  - 一个学生可能有多条考试记录,全部列出,不要擅自只取一条;存在多条且我没给过滤条件时,让我确认。
  - `user_quiz_id` 为空的记录单独列出,不要放进重置列表。

### Step 4:预览并确认(必须停下)
输出表格:准考证号 | 学生姓名 | 考试名称/场次 | start_time | 当前状态 | `user_quiz_id`。
同时列出:Excel 里没匹配到学生的准考证号、匹配到学生但没有考试记录的准考证号、总计将重置条数,以及本次将使用的目标状态(默认"保存",status=1)。
如果结果多于 50 行,把完整清单导出为 CSV 到导出路径(默认 `E:\temp-file`),终端只展示前 20 行和文件路径。
**该清单(准考证号 ↔ user_quiz_id 的对应关系)需要保留到最后,Step 6 的 search 响应里没有准考证号,必须靠 `id` 对回这份清单。**
**等我明确回复"确认"再继续。**

### Step 5:调用重置接口
- Host:`http://internal-quiz-544761462.cn-north-1.elb.amazonaws.com.cn`
- 请求:`POST {host}/v1_1_0/{bucketKey}/userQuiz/batchReset`
  - `bucketKey` 固定格式为 `school_{schoolId}`,本次即 `school_114357`。
  - 本次完整 URL:`http://internal-quiz-544761462.cn-north-1.elb.amazonaws.com.cn/v1_1_0/school_114357/userQuiz/batchReset`
  - `Content-Type: application/json`
- 请求体(默认重置为保存):
```json
{
  "ids": ["<user_quiz_id>", "..."],
  "status": 1,
  "resetUserId": 12345,
  "platform": "xxx"
}
```
  - status=1 时不需要传 `resetSubmittedCount`、`resetPassFaceRecognition`(接口不会使用);status=0 时才按参数表传入。
- `ids` 为 Mongo ID 格式,不能为空。分批提交,每批最多 50 个,**串行**执行。
- 先用第一批 1~2 条试跑,检查响应成功后,**立即用 Step 6 的 search 接口校验这 1~2 条的状态**,确认变为目标状态后再继续后续批次。
- 响应结构为 NoneDataJsonResult,以其 `hr` 字段判断成败。任一批失败立即停止,记录失败批次的 ids 和返回信息。

### 接口行为说明(供你判断结果)
- `status=1`(已保存,默认):将 UserQuiz 重置为保存状态 → 若有 `liveRecordId`,调用直播服务 reset。
- `status=0`(未开始):更新 UserQuiz → 删除对应的答题签名 → 若有 `liveRecordId`,调用直播服务删除直播记录(同一账号不能开多个直播)。
- 其他 `status` 值接口不会做任何处理却返回成功,**禁止传 0/1 以外的值**。
- 接口是多步非事务操作,中途失败可能出现部分已更新的状态,失败后不要直接重试整批,先通过 search 接口查询当前状态再告诉我。

### Step 6:结果校验与汇报(通过 search 接口,不查 examination_statistic)
- 请求:`POST {host}/v1_1_0/school_{schoolId}/userQuiz/search`,本次完整 URL:
  `http://internal-quiz-544761462.cn-north-1.elb.amazonaws.com.cn/v1_1_0/school_114357/userQuiz/search`
- 请求体(按重置时同样的批次,每批最多 50 个 id):
```json
{
  "ids": ["<user_quiz_id>", "..."],
  "pageIndex": 0,
  "pageSize": 50
}
```
  - `ids` 与 `pageSize` 保持一致,避免被分页截断;若返回 `total` 大于 `rows` 条数,继续翻页(`pageIndex` 从 0 递增)。
  - 不要传 `statuses` 等过滤条件,否则状态不符的记录会被过滤掉而误判成"丢失"。
- 响应为 `GenericJsonResult<Pages<UserQuizSearchResponse>>`,数据在 `data.rows`。先检查 `hr` 是否成功,失败则停止并告诉我。
- 逐条按 `id` 对照校验 `rows[i]`:
  - `status` 等于目标状态(默认 `1`,保存);
  - 若目标是 `0`(未开始):`liveRecordId` 应为空;若传了 `resetSubmittedCount=true`,`submittedCount` 应被重置;若传了 `resetPassFaceRecognition=true`,`passFaceRecognition` 应被重置;
  - `resetTime` 为刚才操作的时间附近,`resetUser` 对应传入的 `resetUserId`。
  - 请求的 id 在 `rows` 中找不到,也记为校验失败。
- 汇报:总数 / 重置成功 / 校验通过 / 校验失败 / 未匹配,附失败明细(准考证号、user_quiz_id、期望状态、实际状态)。
- 完整的执行明细(准考证号、user_quiz_id、批次、重置结果、校验结果、校验后的 status)导出为 CSV 到导出路径(默认 `E:\temp-file`),告诉我文件路径。

## 四、禁止事项
- 不得在日志、输出、文件中出现数据库密码。
- 不得对生产库执行 INSERT/UPDATE/DELETE/DDL。
- 未经我确认不得调用重置接口;不得扩大范围(只处理我给的准考证号 + 过滤条件命中的记录)。
