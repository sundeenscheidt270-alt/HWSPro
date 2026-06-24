# 附件E: 规则总表（RuleBook）

> **PRD主档**: [PRD_Main.md](./PRD_Main.md)  
> **版本**: v1.0.0  
> **更新日期**: 2025-12-28

---

## 规则分类索引

- [1. 商品编码规则](#1-商品编码规则)
- [2. 商品上架规则](#2-商品上架规则)
- [3. 价格管理规则](#3-价格管理规则)
- [4. 批次管理规则](#4-批次管理规则)
- [5. 临期预警规则](#5-临期预警规则)
- [6. 商品淘汰规则](#6-商品淘汰规则)
- [7. 数据同步规则](#7-数据同步规则)
- [8. 权限控制规则](#8-权限控制规则)

---

## 1. 商品编码规则

### Rule-001: 商品编码生成规则
**规则描述**：商品编码由PMS系统统一生成，ERP不可自行生成或修改。

**规则内容**：
- 编码格式：`<类别码><品牌码><顺序号>`，共12位
- 示例：`01ABC0001234`
  - `01`: 休闲零食类
  - `ABC`: 品牌码
  - `0001234`: 7位顺序号

**执行场景**：商品新建时

**规则实现**：
- ERP接收PMS推送的商品编码，不允许手动输入
- 商品编码字段设置为只读
- 系统校验：必须符合12位格式

**异常处理**：
- 如编码重复：阻断接收，通知PMS修正
- 如编码格式错误：阻断接收，记录错误日志

---

### Rule-002: 条码唯一性规则
**规则描述**：商品条码（barcode）在系统中必须唯一。

**规则内容**：
- 一个条码只能对应一个商品
- 条码必须符合EAN-13或EAN-8标准
- 条码格式：13位或8位数字

**执行场景**：
- 商品新建时
- 商品修改条码时

**规则实现**：
- 数据库唯一约束：barcode字段
- 前端校验：输入时实时检查
- 后端校验：保存前校验

**异常处理**：
- 条码重复：提示"条码已存在，请检查"，阻断保存
- 条码格式错误：提示"条码格式不正确"，阻断保存

---

## 2. 商品上架规则

### Rule-003: 商品上架前置条件
**规则描述**：商品必须满足以下条件才能上架销售。

**规则内容**（必须全部满足）：
1. 商品状态为`PENDING_CONFIG`
2. 已配置至少一个有效价格（总部价）
3. 已关联至少一个供应商（is_primary=1）
4. 已关联分类（category_id非空）
5. 已设置保质期天数（shelf_life_days>0）
6. 已设置温区（temperature_zone非空）
7. 已设置箱规（case_pack>0）

**执行场景**：商品上架操作时

**规则实现**：
```python
def can_go_live(product):
    checks = [
        product.status == 'PENDING_CONFIG',
        product.has_valid_price(),
        product.has_primary_supplier(),
        product.category_id is not None,
        product.shelf_life_days > 0,
        product.temperature_zone is not None,
        product.case_pack > 0
    ]
    return all(checks)
```

**异常处理**：
- 任一条件不满足：阻断上架，提示缺少的配置项
- 示例："商品无法上架，原因：未配置总部价格"

---

### Rule-004: 商品下架规则
**规则描述**：商品下架时的库存检查和后续处理。

**规则内容**：
- 商品状态变更为`SUSPENDED`
- 下架时不检查库存（有库存也可下架）
- 下架后，门店POS不可继续销售
- 已有库存保留，等待处理（调拨/报损）

**执行场景**：商品下架操作时

**规则实现**：
1. 状态变更：`ACTIVE` → `SUSPENDED`
2. 推送状态变更给POS/WMS
3. POS收到变更后，禁止扫码销售
4. 已锁定库存（订单）继续履约

**异常处理**：
- 已有未完成订单：提示"有X个订单待履约"，但允许下架

---

## 3. 价格管理规则

### Rule-005: 价格优先级规则
**规则描述**：当商品有多个价格时，按优先级取价。

**规则内容**（优先级从高到低）：
1. **促销价**（PROMOTION_PRICE）：活动期间有效
2. **门店价**（STORE_PRICE）：特定门店价格
3. **区域价**（REGION_PRICE）：区域公司价格
4. **会员价**（MEMBER_PRICE）：会员专享
5. **总部价**（HQ_PRICE）：全国统一价，兜底价格

**执行场景**：
- POS扫码结算时
- 电商下单时
- 价格查询时

**规则实现**：
```sql
SELECT price 
FROM product_price 
WHERE product_id = ? 
  AND is_active = 1
  AND CURDATE() BETWEEN effective_date AND IFNULL(expiry_date, '9999-12-31')
  -- 优先级排序
  AND (
    (price_type = 'PROMOTION_PRICE' AND org_id = ?) OR
    (price_type = 'STORE_PRICE' AND org_id = ?) OR
    (price_type = 'REGION_PRICE' AND org_id = ?) OR
    (price_type = 'MEMBER_PRICE') OR
    (price_type = 'HQ_PRICE')
  )
ORDER BY 
  FIELD(price_type, 'PROMOTION_PRICE', 'STORE_PRICE', 'REGION_PRICE', 'MEMBER_PRICE', 'HQ_PRICE')
LIMIT 1;
```

**异常处理**：
- 无任何有效价格：阻断销售，提示"商品未配置价格"

---

### Rule-006: 价格生效时间规则
**规则描述**：价格变更需指定生效时间，生效前可修改/删除。

**规则内容**：
- 价格必须指定生效日期（effective_date）
- 生效日期≥当前日期（可以是未来）
- 生效前的价格可以修改/删除
- 已生效的价格不可修改/删除，只能失效

**执行场景**：价格新增/修改/删除时

**规则实现**：
```python
def can_edit_price(price):
    return date.today() < price.effective_date

def can_delete_price(price):
    return date.today() < price.effective_date
```

**异常处理**：
- 修改已生效价格：提示"价格已生效，不可修改"
- 删除已生效价格：提示"价格已生效，不可删除，请设置失效日期"

---

### Rule-007: 价格有效期重叠检查
**规则描述**：同一商品+同一价格类型+同一组织，有效期不能重叠。

**规则内容**：
- 新增价格时，检查是否与现有价格有效期重叠
- 有效期计算：`[effective_date, expiry_date]`，expiry_date为NULL视为无限期

**执行场景**：价格新增/修改时

**规则实现**：
```sql
SELECT COUNT(*) FROM product_price
WHERE product_id = ?
  AND price_type = ?
  AND IFNULL(org_id, 0) = IFNULL(?, 0)
  AND is_active = 1
  AND (
    (? BETWEEN effective_date AND IFNULL(expiry_date, '9999-12-31'))
    OR (IFNULL(?, '9999-12-31') BETWEEN effective_date AND IFNULL(expiry_date, '9999-12-31'))
  );
  -- 如果> 0，说明有重叠
```

**异常处理**：
- 发现重叠：阻断保存，提示"价格有效期与现有价格重叠：YYYY-MM-DD ~ YYYY-MM-DD"

---

## 4. 批次管理规则

### Rule-008: 批次管理强制规则
**规则描述**：食品类商品必须启用批次管理，非食品类可选。

**规则内容**：
- 食品类（category.is_food=1）：is_batch_managed强制=1
- 非食品类：is_batch_managed可配置

**执行场景**：商品建档时、修改时

**规则实现**：
- 前端：食品类商品，批次管理复选框默认勾选且禁用
- 后端：保存时校验，食品类必须is_batch_managed=1

**异常处理**：
- 食品类商品取消批次管理：阻断保存，提示"食品类商品必须启用批次管理"

---

### Rule-009: 批次号规则
**规则描述**：批次号由"商品编码+生产日期"组成，保证唯一性。

**规则内容**：
- 批次号格式：`<商品编码><生产日期YYYYMMDD>`
- 示例：`01ABC000123420250315`
  - `01ABC0001234`: 商品编码
  - `20250315`: 生产日期2025-03-15

**执行场景**：采购入库时、商品收货时

**规则实现**：
```python
def generate_batch_no(product_code, production_date):
    return f"{product_code}{production_date.strftime('%Y%m%d')}"
```

**异常处理**：
- 批次号重复：允许（同一批次多次入库）

---

## 5. 临期预警规则

### Rule-010: 临期预警触发规则（核心规则）
**规则描述**：根据保质期剩余比例触发临期预警。

**规则内容**：
- 预警阈值：保质期剩余≤30%
- 计算公式：
  ```
  保质期剩余比例 = (到期日期 - 当前日期) / 保质期天数 × 100%
  ```
- 示例：
  - 保质期365天的商品，剩余110天（30%）触发预警
  - 保质期30天的商品，剩余9天（30%）触发预警

**执行场景**：系统每日凌晨2点自动扫描

**规则实现**：
```sql
SELECT 
    i.product_id,
    p.product_name,
    i.batch_no,
    i.expire_date,
    DATEDIFF(i.expire_date, CURDATE()) AS days_remaining,
    p.shelf_life_days,
    ROUND(DATEDIFF(i.expire_date, CURDATE()) / p.shelf_life_days * 100, 2) AS remaining_percent
FROM inventory i
JOIN product p ON i.product_id = p.product_id
WHERE i.quantity_total > 0
  AND DATEDIFF(i.expire_date, CURDATE()) / p.shelf_life_days <= 0.3
  AND DATEDIFF(i.expire_date, CURDATE()) > 0;
```

**异常处理**：
- 已过期（剩余天数≤0）：不纳入临期预警，直接报损

---

### Rule-011: 临期预警推送规则
**规则描述**：临期预警生成清单后，推送给相关人员。

**规则内容**：
- 推送对象：商品部、采购部、门店店长
- 推送方式：站内信+邮件
- 推送时间：每日8:00
- 清单内容：商品、批次、库存量、剩余天数、建议处理方式

**执行场景**：临期扫描完成后

**规则实现**：
1. 生成临期清单（JSON）
2. 调用消息服务推送
3. 记录推送日志

**异常处理**：
- 推送失败：重试3次，仍失败则记录日志

---

## 6. 商品淘汰规则

### Rule-012: 商品淘汰条件
**规则描述**：满足以下条件的商品可以淘汰。

**规则内容**（满足任一条件）：
1. 近6个月无销售
2. 近3个月动销率<5%
3. 商品部手动淘汰决策

**执行场景**：商品淘汰评估时

**规则实现**：
```sql
-- 查找可淘汰商品
SELECT product_id, product_name
FROM product p
LEFT JOIN sales_order_line sol ON p.product_id = sol.product_id 
    AND sol.order_date >= DATE_SUB(CURDATE(), INTERVAL 6 MONTH)
WHERE sol.product_id IS NULL  -- 6个月无销售
  AND p.status = 'ACTIVE';
```

**异常处理**：
- 有库存可淘汰：允许淘汰，库存需处理（调拨/报损）

---

### Rule-013: 商品淘汰后处理规则
**规则描述**：商品淘汰后的库存和订单处理。

**规则内容**：
- 状态变更为`OBSOLETE`
- 禁止新增采购订单
- 禁止新增销售订单
- 已有订单继续履约
- 库存处理：报损/调拨至其他门店

**执行场景**：商品淘汰时

**规则实现**：
1. 状态变更：`ACTIVE/SUSPENDED` → `OBSOLETE`
2. 推送状态变更 to POS/WMS
3. 生成库存处理任务

---

## 7. 数据同步规则

### Rule-014: PMS→ERP同步规则
**规则描述**：PMS商品数据同步到ERP的规则。

**规则内容**：
- 同步方式：消息队列（MQ），异步
- 同步时机：PMS商品审核通过时立即推送
- 同步字段：商品基础信息（编码、名称、分类、规格等）
- 不同步字段：价格、库存（ERP自行维护）

**执行场景**：PMS推送消息时

**规则实现**：
1. 监听MQ队列：`pms.product.sync`
2. 接收消息JSON
3. 解析并保存/更新商品
4. 返回ACK确认

**异常处理**：
- 消息格式错误：记录错误日志，发送NACK，不重试
- 业务校验失败（如编码重复）：记录错误日志，发送NACK，人工介入
- 系统异常：发送NACK，重试（最多3次）

---

### Rule-015: ERP→POS/WMS同步规则
**规则描述**：ERP商品数据推送到POS/WMS的规则。

**规则内容**：
- 同步方式：实时接口调用（RESTful API）
- 同步时机：
  - 商品上架时
  - 价格变更时
  - 状态变更时
- 同步字段：
  - To POS：商品基础信息+价格
  - To WMS：商品基础信息+温区+批次规则

**执行场景**：触发事件发生时

**规则实现**：
1. 监听商品变更事件
2. 调用POS/WMS接口
3. 记录调用日志

**异常处理**：
- 调用失败：重试3次
- 仍失败：记录错误日志，生成待处理任务

---

### Rule-016: 数据对账规则
**规则描述**：每日与PMS/POS进行数据对账。

**规则内容**：
- 对账时间：每日凌晨3点
- 对账对象：PMS（商品主数据）、POS（价格）
- 对账方式：全量数据MD5比对
- 差异处理：生成差异报告，人工核对

**执行场景**：定时任务

**规则实现**：
```python
def reconcile_with_pms():
    erp_products = get_all_products_md5()  # ERP商品MD5
    pms_products = call_pms_api()          # PMS商品MD5
    diff = compare(erp_products, pms_products)
    if diff:
        send_alert(diff)  # 发送差异告警
```

---

## 8. 权限控制规则

### Rule-017: 商品操作权限规则
**规则描述**：不同角色对商品的操作权限。

**规则内容**：

| 角色 | 查看 | 新建 | 修改 | 上架 | 下架 | 淘汰 | 删除 |
|------|------|------|------|------|------|------|------|
| 商品专员 | ✓ | ✓ | ✓ | - | - | - | - |
| 商品主管 | ✓ | ✓ | ✓ | ✓ | ✓ | - | - |
| 商品总监 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 采购员 | ✓ | - | - | - | - | - | - |
| 门店店长 | ✓ | - | - | - | - | - | - |

**执行场景**：所有商品操作时

**规则实现**：
- 基于RBAC（Role-Based Access Control）
- 后端接口权限校验
- 前端按钮显示/隐藏

---

### Rule-018: 价格数据脱敏规则
**规则描述**：供应商合同价仅特定角色可见。

**规则内容**：
- 可见角色：采购员、采购主管、商品总监、财务
- 不可见角色：门店店长、仓管员

**执行场景**：查询供应商商品目录时

**规则实现**：
```python
def get_supplier_product(user):
    data = query_supplier_product()
    if user.role not in ['采购员', '采购主管', '商品总监', '财务']:
        data.contract_price = '***'  # 脱敏
    return data
```

---

## 9. 规则配置化清单

以下规则支持配置化（系统管理员可修改）：

| 规则编号 | 规则名称 | 可配置参数 | 默认值 | 说明 |
|---------|---------|-----------|--------|------|
| Rule-010 | 临期预警触发规则 | 预警阈值 | 30% | 保质期剩余比例 |
| Rule-011 | 临期预警推送规则 | 推送时间 | 08:00 | 每日推送时间 |
| Rule-012 | 商品淘汰条件 | 无销售天数阈值 | 180天 | 近X天无销售可淘汰 |
| Rule-014 | PMS同步规则 | 重试次数 | 3次 | 失败重试次数 |
| Rule-016 | 数据对账规则 | 对账时间 | 03:00 | 每日对账时间 |

---

## 10. 规则变更记录

| 版本 | 日期 | 变更规则 | 变更内容 | 影响范围 |
|------|------|---------|---------|---------|
| v1.0.0 | 2025-12-28 | 全部 | 初始版本 | 全部模块 |
