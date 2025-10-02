# 学校管理与支付系统数据库详细设计说明书

## 1. 概述
本数据库用于支撑学校管理平台的核心业务，包括：
- 学校、校区、年级、班级管理
- 学生、家长、教职工信息管理
- 家长端报名/缴费（小程序、H5）
- 微信支付对接与回执管理

数据库采用 **MySQL 8.x**，字符集统一为 `utf8mb4`，存储引擎为 `InnoDB`，支持事务与外键约束。

---

## 2. 表结构设计

### 2.1 基础字典与地址

#### division（行政区划表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| code | VARCHAR(12) | PK | 行政区划代码（GB/T 2260） |
| name_zh | VARCHAR(100) | NOT NULL | 中文名称 |
| level | TINYINT | NOT NULL | 1=省/直辖市，2=地级市，3=区/县 |
| parent_code | VARCHAR(12) | FK→division.code | 上级行政区划代码 |

#### address（地址表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 地址ID |
| country_code | CHAR(2) | DEFAULT 'CN' | 国家代码 |
| province_code | VARCHAR(12) | FK→division.code | 省代码 |
| city_code | VARCHAR(12) | FK→division.code | 市代码 |
| district_code | VARCHAR(12) | FK→division.code | 区代码 |
| address_line | VARCHAR(200) |  | 详细地址 |
| postal_code | VARCHAR(20) |  | 邮编 |
| latitude | DECIMAL(9,6) |  | 纬度 |
| longitude | DECIMAL(9,6) |  | 经度 |

---

### 2.2 人员与联系方式

#### person（人员表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 人员ID |
| full_name_zh | VARCHAR(100) | NOT NULL | 中文姓名 |
| full_name_py | VARCHAR(100) |  | 姓名拼音 |
| gender | ENUM('M','F','U') |  | 性别 |
| dob | DATE |  | 出生日期 |
| national_id_hash | VARCHAR(256) |  | 身份证号哈希 |
| email | VARCHAR(120) |  | 邮箱 |
| status | VARCHAR(20) | DEFAULT 'active' | 状态 |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | DATETIME | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 更新时间 |

#### phone（电话表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 电话ID |
| number_e164 | VARCHAR(20) | UNIQUE, NOT NULL | E.164 格式手机号 |
| is_verified | BOOLEAN | DEFAULT FALSE | 是否已验证 |
| verified_at | DATETIME |  | 验证时间 |

#### person_phone（人员-电话关联表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| person_id | BIGINT | PK, FK→person.id | 人员ID |
| phone_id | BIGINT | PK, FK→phone.id | 电话ID |
| label | VARCHAR(20) |  | 标签（mobile/home） |
| is_primary | BOOLEAN | DEFAULT FALSE | 是否主号 |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

---

### 2.3 学校与班级

#### school（学校表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 学校ID |
| name_zh | VARCHAR(150) | UNIQUE, NOT NULL | 学校名称 |
| short_name_zh | VARCHAR(80) |  | 学校简称 |
| code | VARCHAR(50) |  | 学校编码 |
| address_id | BIGINT | FK→address.id | 地址ID |
| established_at | DATE |  | 建校日期 |
| status | VARCHAR(20) | DEFAULT 'active' | 状态 |

#### campus（校区表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 校区ID |
| school_id | BIGINT | FK→school.id | 学校ID |
| name_zh | VARCHAR(100) | NOT NULL | 校区名称 |
| address_id | BIGINT | FK→address.id | 地址ID |

#### grade_level（年级表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 年级ID |
| school_id | BIGINT | FK→school.id | 学校ID |
| stage | VARCHAR(20) | NOT NULL | 学段（primary/middle/high） |
| level | SMALLINT | NOT NULL | 年级序号 |
| name_zh | VARCHAR(50) | NOT NULL | 年级名称 |
| duration_years | SMALLINT |  | 学制年限 |

#### class（班级表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 班级ID |
| school_id | BIGINT | FK→school.id | 学校ID |
| campus_id | BIGINT | FK→campus.id | 校区ID |
| cohort_year | INT | NOT NULL | 入学年份 |
| grade_level_id | BIGINT | FK→grade_level.id | 年级ID |
| name_zh | VARCHAR(80) | NOT NULL | 班级名称 |
| homeroom_staff_id | BIGINT |  | 班主任ID |

---

### 2.4 学生与家长

#### student（学生表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 学生ID |
| person_id | BIGINT | UNIQUE, FK→person.id | 人员ID |
| student_no | VARCHAR(50) |  | 学号 |
| current_school_id | BIGINT | FK→school.id | 当前学校 |
| status | VARCHAR(20) | DEFAULT 'active' | 状态 |

#### guardian_link（监护关系表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 关系ID |
| student_id | BIGINT | FK→student.id | 学生ID |
| guardian_person_id | BIGINT | FK→person.id | 家长ID |
| relationship | VARCHAR(30) | NOT NULL | 关系（father/mother/guardian） |
| is_primary | BOOLEAN | DEFAULT FALSE | 是否主联系人 |
| contact_consent | BOOLEAN | DEFAULT TRUE | 是否同意联系 |

---

### 2.5 报名与支付

#### activation_submission（激活提交表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 提交ID |
| submitted_by_person_id | BIGINT | FK→person.id | 提交人 |
| student_name_raw | VARCHAR(100) |  | 学生姓名（原始） |
| gender_raw | VARCHAR(10) |  | 性别（原始） |
| counselor_phone_raw | VARCHAR(30) |  | 辅导员手机号 |
| parent_name_raw | VARCHAR(100) |  | 家长姓名 |
| parent_phone_raw | VARCHAR(30) |  | 家长手机号 |
| teacher_name_raw | VARCHAR(100) |  | 老师姓名 |
| school_name_raw | VARCHAR(150) |  | 学校名称 |
| province_raw | VARCHAR(50) |  | 省 |
| city_raw | VARCHAR(50) |  | 市 |
| district_raw | VARCHAR(50) |  | 区 |
| cohort_year_raw | INT |  | 年届 |
| class_name_raw | VARCHAR(80) |  | 班级名称 |
| agreement_code | VARCHAR(50) |  | 协议编码 |
| agreement_version | VARCHAR(20) |  | 协议版本 |
| status | VARCHAR(20) | DEFAULT 'approved' | 状态 |
| reviewed_by_staff_id | BIGINT | FK→staff.id | 审核人 |
| reviewed_at | DATETIME |  | 审核时间 |
| notes | TEXT |  | 备注 |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | DATETIME | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 更新时间 |

---

#### payment_order（支付订单表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 订单ID |
| order_no | VARCHAR(64) | UNIQUE, NOT NULL | 业务订单号 |
| student_id | BIGINT | FK→student.id | 学生ID |
| payer_person_id | BIGINT | FK→person.id | 支付人（家长）ID |
| school_id | BIGINT | FK→school.id | 学校ID |
| class_id | BIGINT | FK→class.id | 班级ID |
| description | VARCHAR(200) |  | 订单描述 |
| amount_total_cents | INT | NOT NULL | 总金额（分） |
| currency | VARCHAR(8) | DEFAULT 'CNY' | 币种 |
| status | VARCHAR(20) | DEFAULT 'created' | 状态（created/paying/success/failed/expired/refunded） |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | DATETIME | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 更新时间 |

---

#### payment_order_item（支付订单明细表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 明细ID |
| order_id | BIGINT | FK→payment_order.id | 订单ID |
| item_code | VARCHAR(50) |  | 缴费项编码 |
| item_name | VARCHAR(120) | NOT NULL | 缴费项名称 |
| amount_cents | INT | NOT NULL | 金额（分） |
| quantity | INT | DEFAULT 1 | 数量 |

---

#### wechat_payment（微信支付流水表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 流水ID |
| order_id | BIGINT | FK→payment_order.id | 订单ID |
| appid | VARCHAR(64) | NOT NULL | 微信 AppID |
| mchid | VARCHAR(32) | NOT NULL | 商户号 |
| openid | VARCHAR(128) |  | 用户 openid |
| prepay_id | VARCHAR(128) |  | 预支付交易单号 |
| transaction_id | VARCHAR(64) | UNIQUE | 微信支付单号 |
| trade_state | VARCHAR(32) |  | 支付状态（SUCCESS/NOTPAY/CLOSED/…） |
| amount_payer_cents | INT |  | 实付金额（分） |
| success_at | DATETIME |  | 支付成功时间 |
| notify_raw | JSON |  | 微信回调原文 |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | DATETIME | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 更新时间 |

---

#### payment_receipt（支付回执表）
| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT | PK, AI | 回执ID |
| order_id | BIGINT | FK→payment_order.id | 订单ID |
| receipt_no | VARCHAR(64) | UNIQUE | 回执编号 |
| url | TEXT |  | 回执文件地址 |
| issued_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | 生成时间 |

---

## 3. 索引与约束设计

- **唯一约束**：
  - `school.name_zh` 唯一
  - `phone.number_e164` 唯一
  - `payment_order.order_no` 唯一
  - `wechat_payment.transaction_id` 唯一

- **外键约束**：
  - 所有外键均使用 `ON DELETE CASCADE` 或 `ON DELETE RESTRICT`，确保数据一致性
  - 例如：删除学生会级联删除其班级成员关系、监护关系、缴费订单等

- **常用索引**：
  - `person(full_name_zh)`
  - `student(student_no)`
  - `payment_order(status, created_at)`
  - `wechat_payment(trade_state)`

---

## 4. 数据流与表更新顺序（无后台审核版）

1. **家长提交表单**  
   - `INSERT INTO activation_submission (...)`
   - 自动创建/匹配 `person`、`student`、`guardian_link`、`class_membership`

2. **生成缴费单（固定 380 元）**  
   - `INSERT INTO payment_order (...)`
   - `INSERT INTO payment_order_item (...)`

3. **调用微信统一下单**  
   - `INSERT INTO wechat_payment (...)`

4. **微信回调成功**  
   - `UPDATE wechat_payment SET trade_state='SUCCESS', success_at=NOW()`
   - `UPDATE payment_order SET status='success'`
   - `INSERT INTO payment_receipt (...)`

---

## 5. 安全与合规建议

- **敏感信息加密**：手机号、身份证号等建议加密存储
- **审计日志**：保留所有关键表的变更记录
- **幂等控制**：支付回调、订单生成需加幂等锁
- **数据脱敏**：后台列表展示时对手机号等敏感字段做脱敏处理

---

## 7. 表关系概览（ER 模型）

### 核心实体
- **person**：人员主表（学生、家长、教职工统一管理）
- **student**：学生信息
- **staff**：教职工信息
- **school / campus / grade_level / class**：学校组织结构
- **guardian_link**：学生与家长的关系
- **activation_submission**：家长提交的原始报名/缴费信息
- **payment_order / payment_order_item**：缴费订单与明细
- **wechat_payment**：微信支付流水
- **payment_receipt**：支付回执

### 主要关系
- `student.person_id → person.id`  
- `guardian_link.student_id → student.id`  
- `guardian_link.guardian_person_id → person.id`  
- `class_membership.student_id → student.id`  
- `class_membership.class_id → class.id`  
- `payment_order.student_id → student.id`  
- `payment_order.payer_person_id → person.id`  
- `payment_order_item.order_id → payment_order.id`  
- `wechat_payment.order_id → payment_order.id`  
- `payment_receipt.order_id → payment_order.id`  

---

## 8. 数据流与业务场景映射

### 家长报名缴费流程（无后台审核）
1. **家长提交表单**  
   - 写入 `activation_submission`（status=approved）
   - 自动匹配/创建 `person`（学生、家长）、`student`、`guardian_link`、`class_membership`
2. **生成缴费单**  
   - 固定金额 380 元（38000 分）
   - 写入 `payment_order` 和 `payment_order_item`
3. **调用微信统一下单**  
   - 写入 `wechat_payment`（预支付信息）
4. **微信回调成功**  
   - 更新 `wechat_payment.trade_state='SUCCESS'`
   - 更新 `payment_order.status='success'`
   - 写入 `payment_receipt`

---

## 9. 安全与合规建议

- **数据安全**
  - 手机号、身份证号等敏感信息建议加密存储
  - 后台展示时对敏感字段进行脱敏
- **支付安全**
  - 微信支付 APIv3 密钥、证书妥善保管
  - 回调接口需验签与幂等处理
- **审计追踪**
  - 保留 `activation_submission` 原始数据
  - 对订单、支付状态变更记录操作日志

---

## 10. 版本与维护

- **版本号**：v1.0（2025-09-14）
- **维护人**：Bin
- **更新记录**：
  - v1.0：初版数据库设计，支持学校管理与微信支付全流程

---
