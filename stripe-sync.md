# Stripe 订阅事件同步到本地团队套餐限制 - 完整路径分析

## 一、整体架构概览

Stripe 事件同步系统采用 **Webhook 入口 + 异步 Job 处理 + 状态机映射** 的三层架构：

```
Stripe 平台 → Webhook 签名验证 → StripeProcessJob（队列）→ 状态映射 → 本地数据库更新 → 团队套餐限制生效
```

涉及的核心文件：
- [Stripe.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Http/Controllers/Webhook/Stripe.php) - Webhook 入口
- [StripeProcessJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php) - 核心事件处理器
- [UpdateSubscriptionQuantity.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/UpdateSubscriptionQuantity.php) - 套餐数量变更 Action
- [RefundSubscription.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/RefundSubscription.php) - 退款处理 Action
- [Team.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Team.php) - 团队模型（含套餐限制）
- [Subscription.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Subscription.php) - 订阅模型

辅助 Job：
- [VerifyStripeSubscriptionStatusJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/VerifyStripeSubscriptionStatusJob.php) - 订阅状态校验
- [SubscriptionInvoiceFailedJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/SubscriptionInvoiceFailedJob.php) - 支付失败处理
- [ServerLimitCheckJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/ServerLimitCheckJob.php) - 服务器数量超限检查

---

## 二、按代码顺序的完整调用路径

### 路径 1：Webhook 入口 → 事件分发

**[Stripe.php:14-36](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Http/Controllers/Webhook/Stripe.php#L14-L36)**

```php
public function events(Request $request)
{
    // 1. 签名验证（防伪造）
    $event = Webhook::constructEvent(
        $request->getContent(),
        $signature,
        $webhookSecret
    );
    // 2. 异步分发到高优先级队列
    StripeProcessJob::dispatch($event);
    return response('Webhook received.', 200);
}
```

**关键点**：
- 首先验证 Stripe 签名，防止伪造请求
- 立即返回 200，避免 Stripe 重试（Stripe 要求在几秒内响应）
- 事件通过 `high` 队列异步处理，不阻塞 Webhook 响应

---

### 路径 2：StripeProcessJob 事件路由

**[StripeProcessJob.php:29-347](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L29-L347)**

Job 启动后，通过 `switch ($type)` 路由到不同事件处理器：

| 事件类型 | 处理逻辑 | 影响本地状态 |
|---------|---------|-------------|
| `checkout.session.completed` | 首次订阅完成 | ✅ 创建订阅记录，标记已支付 |
| `customer.subscription.created` | 订阅创建 | ⚠️ 创建记录，但**不标记已支付** |
| `customer.subscription.updated` | 订阅更新（核心） | ✅ 状态变更、数量变更、套餐变更 |
| `customer.subscription.deleted` | 订阅删除 | ❌ 终止订阅，禁用服务器 |
| `invoice.paid` | 发票支付成功 | ✅ 同步支付状态 |
| `invoice.payment_failed` | 发票支付失败 | ⚠️ 延迟检查后通知 |
| `payment_intent.payment_failed` | 支付意图失败 | ⚠️ 仅记录，不直接操作 |
| `radar.early_fraud_warning.created` | 欺诈预警 | ❌ 立即退款并取消订阅 |

---

## 三、三大事件源对照：字段变更与 subscriptionEnded 触发

### 3.1 invoice.paid — 发票支付成功事件

**代码位置**：[StripeProcessJob.php:87-149](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L87-L149)

**处理步骤与字段变更**：

```
Step 1: 从事件数据提取 customerId → 按 stripe_customer_id 找 Subscription
Step 2: 若本地有 stripe_subscription_id → 主动调用 Stripe API 拉取最新状态（不依赖事件数据）
Step 3: 根据 Stripe 真实 status 映射本地字段
        ┌─────────────┬──────────────────────────────────────────────────┐
        │ Stripe status│ 本地字段变更                                      │
        ├─────────────┼──────────────────────────────────────────────────┤
        │ active      │ stripe_invoice_paid = true                        │
        │             │ stripe_past_due = false                           │
        ├─────────────┼──────────────────────────────────────────────────┤
        │ past_due    │ stripe_invoice_paid = true  ← 注意：仍标记已支付   │
        │             │ stripe_past_due = true                            │
        ├─────────────┼──────────────────────────────────────────────────┤
        │ canceled    │ 仅发内部通知，不更新字段                           │
        │ incomplete… │                                                    │
        │ unpaid      │                                                    │
        ├─────────────┼──────────────────────────────────────────────────┤
        │ default 其他│ 延迟 20s → VerifyStripeSubscriptionStatusJob 兜底 │
        └─────────────┴──────────────────────────────────────────────────┘
Step 4: 若无 stripe_subscription_id → 延迟 20s → VerifyStripeSubscriptionStatusJob
```

**⚠️ 重要结论**：`invoice.paid` **不会触发** `subscriptionEnded()`。即使 Stripe 端已 canceled/unpaid，本事件只发通知不动本地状态，等 `customer.subscription.updated/deleted` 事件来处理。

---

### 3.2 customer.subscription.updated — 订阅更新事件（核心）

**代码位置**：[StripeProcessJob.php:230-323](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L230-L323)

**字段变更全景（按代码执行顺序）**：

```
Step 1: 订阅记录定位
        → 优先按 stripe_customer_id 查
        → 查不到且 status=incomplete_expired → 抛异常
        → 查不到但有 team_id → firstOrCreate 新记录（stripe_invoice_paid = false）

Step 2: 无条件更新字段 [L272-277]
        stripe_feedback = cancellation_details.feedback
        stripe_comment  = cancellation_details.comment
        stripe_plan_id  = items.data.0.plan.id
        stripe_cancel_at_period_end = cancel_at_period_end

Step 3: dynamic 套餐数量更新 [L262-271]
        → lookup_key 包含 'dynamic' 才处理
        → custom_server_limit = min(quantity, 100)  ← 只有上限 100，无下限检查！
        → 触发 ServerLimitCheckJob

Step 4: 状态分支（均带 subscription_id 一致性校验）
        ┌────────────────────┬─────────────────────────────────────────┬────────────────────┐
        │ Stripe status      │ 本地字段变更（需通过 ID 校验）            │ subscriptionEnded? │
        ├────────────────────┼─────────────────────────────────────────┼────────────────────┤
        │ paused             │ stripe_invoice_paid = false             │ ❌ 不触发          │
        │ incomplete_expired │                                         │                    │
        ├────────────────────┼─────────────────────────────────────────┼────────────────────┤
        │ past_due           │ stripe_past_due = true                  │ ❌ 不触发          │
        ├────────────────────┼─────────────────────────────────────────┼────────────────────┤
        │ unpaid             │ stripe_invoice_paid = false             │ ✅ 触发            │
        │                    │                                         │ → 禁用全部服务器  │
        ├────────────────────┼─────────────────────────────────────────┼────────────────────┤
        │ active             │ stripe_past_due = false                 │ ❌ 不触发          │
        │                    │ stripe_invoice_paid = true              │                    │
        └────────────────────┴─────────────────────────────────────────┴────────────────────┘
```

**⚠️ subscriptionEnded 触发条件**：
- **仅当 `status === 'unpaid'` 时触发**，通过 `team->subscriptionEnded()` 调用
- `canceled` 状态不在此事件处理，靠 `customer.subscription.deleted` 事件触发终止

**ID 校验范围**：状态字段变更前必须通过 `$subscription->stripe_subscription_id === $subscriptionId` 校验，涉及 4 处：
- [L279-283](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L279-L283) - paused / incomplete_expired
- [L286-291](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L286-L291) - past_due
- [L294-299](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L294-L299) - unpaid
- [L309-314](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L309-L314) - active

---

### 3.3 customer.subscription.deleted — 订阅删除事件

**代码位置**：[StripeProcessJob.php:324-340](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L324-L340)

```php
$subscription = Subscription::where('stripe_customer_id', $customerId)
    ->where('stripe_subscription_id', $subscriptionId)  // ← 双条件精确匹配
    ->first();

if ($subscription) {
    $team = $subscription->team;
    if ($team) {
        $team->subscriptionEnded();  // ✅ 必触发
    } else {
        throw new RuntimeException("No team found");
    }
} else {
    break;  // 找不到记录静默忽略，不抛异常
}
```

**⚠️ 重要差异**：
- 此事件 **一定触发** `subscriptionEnded()`（只要 team 存在）
- ID 校验方式不是松散比较，而是 `WHERE stripe_customer_id AND stripe_subscription_id` **双条件精确匹配**
- 找不到 subscription 时**静默 break 不抛异常**（与 subscription.updated 的 `throw new RuntimeException` 不同）

---

### 3.4 subscriptionEnded 内部字段变更清单

**代码位置**：[Team.php:220-243](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Team.php#L220-L243)

触发后分两步执行：

```
Step 1: Subscription 表字段重置 [L226-232]
        stripe_subscription_id      = null      ← 断开关联
        stripe_cancel_at_period_end = false
        stripe_invoice_paid         = false     ← 标记无效
        stripe_trial_already_ended  = false
        stripe_past_due             = false

Step 2: 所有服务器禁用 [L233-242]
        server.settings.is_usable    = false
        server.settings.is_reachable = false
        → 触发 ServerReachabilityChanged 事件广播
        → server.unreachable_count = 3（到达阈值）
        → server.unreachable_notification_sent = true
```

**⚠️ 注意**：`subscriptionEnded()` 内部**不检查** `custom_server_limit`，也**不更新**该字段。该字段保留历史值，直到新订阅创建后被新的数量覆盖。

---

## 四、套餐数量变更的上下限差异：Webhook vs UpdateSubscriptionQuantity

### 4.1 Webhook 路径（customer.subscription.updated）—— 只有上限，无下限

**代码位置**：[StripeProcessJob.php:262-271](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L262-L271)

```php
$quantity = min(
    (int) data_get($data, 'items.data.0.quantity', 2),  // 默认值 2，不是下限！
    UpdateSubscriptionQuantity::MAX_SERVER_LIMIT        // 100 —— 唯一的限制
);
```

**关键分析**：
- 调用了 `min(quantity, 100)` → **只有上限 100**
- `data_get(..., 2)` 的 2 是**默认值**（quantity 不存在时取 2），不是下限校验
- 如果 quantity = 0 或 1，Webhook 路径会**原样写入** `custom_server_limit = 0/1`
- 只有通过 Team.limits() accessor 读取时才回退到 `?? 2`，数据库存的是异常值

**测试用例佐证**：[StripeProcessJobTest.php:146-189](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/tests/Feature/Subscription/StripeProcessJobTest.php#L146-L189) 只测试了上限（999→100），没有测试下限场景。

---

### 4.2 UpdateSubscriptionQuantity 路径 —— 上下限都有

**代码位置**：[UpdateSubscriptionQuantity.php:119-199](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/UpdateSubscriptionQuantity.php#L119-L199)

```php
// 入口处先检查下限 [L121-123]
if ($quantity < self::MIN_SERVER_LIMIT) {  // MIN = 2
    return ['success' => false, 'error' => 'Minimum server limit is 2.'];
}
// Stripe API 更新后，Webhook 回调用 min(quantity, 100) 做上限保护
```

**差异总结表**：

| 检查项 | Webhook 路径 | UpdateSubscriptionQuantity |
|-------|-------------|---------------------------|
| 下限 2 | ❌ 无 | ✅ `if ($quantity < 2)` 提前 return |
| 上限 100 | ✅ `min(quantity, 100)` | ✅ 调用后 Webhook 回调保护 |
| 默认值 | ✅ quantity 不存在时取 2 | ❌ 由调用方传入 |
| 测试覆盖 | 仅上限测试 | 上下限都有测试 |

---

## 五、RefundSubscription 四道前置校验全解

**代码位置**：[RefundSubscription.php:24-66](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/RefundSubscription.php#L24-L66)

`checkEligibility()` 按顺序执行四道校验，任何一道不通过立即返回 `ineligible`：

```
前置 1 [L28-30]: stripe_refunded_at 非空？
         └─ 已退过款 → "A refund has already been processed for this team."

前置 2 [L32-34]: stripe_subscription_id 存在？
         └─ 没有关联订阅 → "No active subscription found."

前置 3 [L36-38]: stripe_invoice_paid = true？
         └─ 发票未支付 → "Subscription invoice is not paid."

前置 4 [L40-58]: 调用 Stripe API 检查远程状态
         ├─ 4a. Stripe 查不到订阅 → "Subscription not found in Stripe."
         ├─ 4b. status 不在 ['active', 'trialing'] → 状态非有效
         └─ 4c. 从 start_date 起算超过 30 天窗口 → "The 30-day refund window has expired."
```

**各前置的详细代码对照**：

**前置 1：refunded_at（数据库级防重）**
```php
if ($subscription?->stripe_refunded_at) {
    return $this->ineligible('A refund has already been processed for this team.');
}
```
- 执行顺序：**第一道**，最轻量（纯本地查询）
- 数据来源：`subscriptions.stripe_refunded_at` 字段，`datetime` 类型
- 在 `execute()` 中，调用 Stripe Refund API 后**立即写入**该字段 [L106-110]，即使后续取消订阅失败也已防住重复退款

**前置 2：active/trialing（Stripe 端状态校验）**
```php
$stripeSubscription = $this->stripe->subscriptions->retrieve($subscription->stripe_subscription_id);
if (! in_array($stripeSubscription->status, ['active', 'trialing'])) {
    return $this->ineligible("Subscription status is '{$stripeSubscription->status}'.");
}
```
- 执行顺序：**第四道**中第 4b 步
- 只允许 `active`（正常付费）和 `trialing`（试用中）两种状态
- `past_due` / `canceled` / `unpaid` / `incomplete` 均不允许退款

**前置 3：invoice_paid（本地支付标记）**
```php
if (! $subscription->stripe_invoice_paid) {
    return $this->ineligible('Subscription invoice is not paid.');
}
```
- 执行顺序：**第三道**
- 对应本地 `subscriptions.stripe_invoice_paid` 布尔字段
- 与前置 4 的 Stripe 端状态形成**双重校验**：本地 + 远端都确认已支付

**前置 4：30 天窗口（时间窗口）**
```php
$startDate = Carbon::createFromTimestamp($stripeSubscription->start_date);
$daysSinceStart = (int) $startDate->diffInDays(now());
$daysRemaining = self::REFUND_WINDOW_DAYS - $daysSinceStart; // REFUND_WINDOW_DAYS = 30
if ($daysRemaining <= 0) {
    return $this->ineligible('The 30-day refund window has expired.');
}
```
- 执行顺序：**第四道**中第 4c 步
- 起算时间：Stripe `subscription.start_date`（订阅开始时间，不是首次支付时间）
- 计算方式：`now() - start_date`，取整天数差
- 窗口：精确 30 自然日，第 31 天 00:00 起不可退款
- 返回值还包含 `days_remaining` 供前端展示剩余天数

**四道前置的短路机制**：按顺序校验，前一道失败直接 return，不执行后续校验（避免不必要的 Stripe API 调用）。

---

## 六、ID 校验范围全览

### 6.1 customer.subscription.updated 中的 ID 校验

**校验逻辑**：`$subscription->stripe_subscription_id === $subscriptionId`

**覆盖范围（4 处状态分支）**：

| 状态 | 代码行 | 校验通过时更新 |
|-----|--------|--------------|
| `paused` / `incomplete_expired` | [L279-283](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L279-L283) | `stripe_invoice_paid = false` |
| `past_due` | [L286-291](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L286-L291) | `stripe_past_due = true` |
| `unpaid` | [L294-299](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L294-L299) | `stripe_invoice_paid = false` + `subscriptionEnded()` |
| `active` | [L309-314](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L309-L314) | `stripe_past_due = false` + `stripe_invoice_paid = true` |

**⚠️ 不覆盖的范围**：
- L272-277 的字段更新（`stripe_feedback` / `stripe_comment` / `stripe_plan_id` / `stripe_cancel_at_period_end`）**没有 ID 校验**，直接更新
- L262-271 的 dynamic 套餐数量更新**没有 ID 校验**，直接更新 team 的 `custom_server_limit`
- `canceled` 状态本事件不处理，无校验机会

**场景风险**：如果用户新订阅刚创建完成，旧订阅的 `subscription.updated` 延迟事件到达 —— feedback/comment/plan_id/cancel_at_period_end 和 custom_server_limit 会被旧数据覆盖，但状态字段（invoice_paid/past_due）和 subscriptionEnded 不会被触发。

---

### 6.2 customer.subscription.deleted 中的 ID 校验

**校验逻辑**：`WHERE stripe_customer_id = ? AND stripe_subscription_id = ?`

**代码位置**：[L327](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L327)

```php
$subscription = Subscription::where('stripe_customer_id', $customerId)
    ->where('stripe_subscription_id', $subscriptionId)
    ->first();
```

- 更严格：**双条件精确匹配**，不是先查 customer 再比较 id
- 找不到时 `break` 静默跳过，不抛异常
- 找到了才调用 `subscriptionEnded()`

---

### 6.3 ID 校验差异总结

| 事件 | 校验方式 | 覆盖范围 | 不匹配时 |
|-----|---------|---------|---------|
| subscription.updated | `===` 松散比较 | 仅 4 个状态分支（invoice_paid/past_due + subscriptionEnded） | 跳过字段更新但仍执行无校验部分（feedback/plan_id/数量） |
| subscription.deleted | WHERE 双条件精确匹配 | 整个事件逻辑（subscriptionEnded） | 静默 break，什么都不做 |

---

## 七、重复回调事件处理机制

### 7.1 重复回调的产生原因

Stripe Webhook **至少投递一次**（at-least-once delivery），可能因：
- 网络超时导致 Stripe 未收到 200 响应
- 同一事件多次投递（通常间隔递增）
- 多个事件类型描述同一次状态变更

### 7.2 幂等性保障措施

#### 措施 1：updateOrCreate / firstOrCreate 防重复

**[StripeProcessJob.php:77-85](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L77-L85)**

```php
Subscription::updateOrCreate(
    ['team_id' => $teamId],  // 唯一键：每个 team 只有一条订阅
    [/* 要更新的字段 */]
);
```

- 以 `team_id` 为唯一键，重复事件只会更新同一条记录
- 测试用例：[StripeProcessJobTest.php:58-89](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/tests/Feature/Subscription/StripeProcessJobTest.php#L58-L89)

#### 措施 2：状态字段的幂等更新

所有状态更新使用 `->update()` 直接覆盖，无增量操作：
```php
$subscription->update(['stripe_invoice_paid' => true]);
```
重复执行结果相同。

#### 措施 3：退款标记防重复退款

**[RefundSubscription.php:28-30](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/RefundSubscription.php#L28-L30)**

```php
if ($subscription?->stripe_refunded_at) {
    return $this->ineligible('A refund has already been processed for this team.');
}
```

- 退款前先检查 `stripe_refunded_at` 字段
- 退款操作**先写入该字段**，再执行 Stripe API 调用
- 即使后续取消订阅失败，也能防止重复退款

#### 措施 4：订阅 ID 一致性校验（详见第六章）

通过状态分支前的 ID 一致性比较，防止旧订阅事件误改新订阅的关键状态。

---

## 八、账单状态竞争条件处理

### 8.1 竞争场景分析

**典型竞争时序**：
```
时间线：
T1: 用户触发支付 → Stripe 处理中
T2: Stripe 发送 invoice.payment_failed（卡余额不足）
T3: 用户充值后，Stripe 自动重试成功
T4: Stripe 发送 invoice.paid
T5: 两个事件几乎同时到达，并发处理
```

若 `invoice.payment_failed` 后处理，会覆盖 `invoice.paid` 的正确状态。

### 8.2 竞争防护机制

#### 机制 1：支付失败事件的延迟 + 二次校验

**[StripeProcessJob.php:166-187](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L166-L187)**

```php
// 1. 先调用 Stripe API 确认支付意图状态
$paymentIntent = $stripe->paymentIntents->retrieve($paymentIntentId);
if (in_array($paymentIntent->status, ['processing', 'succeeded', 'requires_action'])) {
    break; // 状态已改变，不处理失败
}

// 2. 新建订阅且 5 分钟内的失败，延迟 60 秒再检查
if (! $subscription->stripe_invoice_paid && $subscription->created_at->diffInMinutes(now()) < 5) {
    SubscriptionInvoiceFailedJob::dispatch($team)->delay(now()->addSeconds(60));
    break;
}
```

#### 机制 2：SubscriptionInvoiceFailedJob 的三道防线

**[SubscriptionInvoiceFailedJob.php:26-64](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/SubscriptionInvoiceFailedJob.php#L26-L64)**

发送失败通知前依次检查：
1. ✅ 检查订阅状态是否已变为 `active` 或 `trialing` → 是则自动修正并返回
2. ✅ 检查最近 1 小时内是否有已支付发票 → 有则自动修正并返回
3. ✅ 两道都没通过才认定为真失败，发送邮件通知

```php
if (in_array($stripeSubscription->status, ['active', 'trialing'])) {
    if (! $subscription->stripe_invoice_paid) {
        $subscription->update(['stripe_invoice_paid' => true, 'stripe_past_due' => false]);
    }
    return; // 不发送失败通知
}
```

#### 机制 3：invoice.paid 中的状态覆盖优先级

**[StripeProcessJob.php:108-136](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L108-L136)**

`invoice.paid` 事件处理时，**主动从 Stripe 拉取最新订阅状态**，而不是依赖事件数据中的 snapshot：
```php
$stripeSubscription = $stripe->subscriptions->retrieve($subscription->stripe_subscription_id);
switch ($stripeSubscription->status) { /* 用 Stripe 实时状态更新本地 */ }
```
这确保即使事件顺序错乱，也以 Stripe 端真实状态为准。

#### 机制 4：VerifyStripeSubscriptionStatusJob 兜底

**[StripeProcessJob.php:132-147](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L132-L147)**

遇到未知状态或 API 调用失败时，延迟 20 秒后调度校验 Job：
```php
VerifyStripeSubscriptionStatusJob::dispatch($subscription)
    ->delay(now()->addSeconds(20));
```
退避策略：`backoff = [10, 30, 60]` 秒，最多重试 3 次。

#### 机制 5：UpdateSubscriptionQuantity 中的支付回滚

**[UpdateSubscriptionQuantity.php:154-178](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/UpdateSubscriptionQuantity.php#L154-L178)**

用户主动变更套餐数量时，若分摊发票支付失败：
```php
if ($latestInvoice && $latestInvoice->status !== 'paid') {
    // 1. 回滚 Stripe 订阅数量到 previousQuantity
    $this->stripe->subscriptions->update($subscriptionId, [/* 原数量 */, 'proration_behavior' => 'none']);
    // 2. 作废未支付发票（voidInvoice）
    $this->stripe->invoices->voidInvoice($latestInvoice->id);
    // 3. 不更新本地 custom_server_limit
    return ['success' => false, 'error' => 'Payment failed.'];
}
```
防止 Stripe 数量已变更但支付失败，导致本地与远端不一致。

---

## 九、完整同步路径总结

### 9.1 三大事件源字段变更对照表

| 事件源 | 触发 subscriptionEnded | 直接更新字段 | 需 ID 校验 |
|-------|----------------------|-------------|-----------|
| invoice.paid | ❌ 从不 | stripe_invoice_paid, stripe_past_due | ❌ 无（但主动拉 Stripe） |
| subscription.updated | ✅ 仅 status=unpaid | stripe_feedback, stripe_comment,<br>stripe_plan_id, stripe_cancel_at_period_end,<br>stripe_invoice_paid, stripe_past_due,<br>custom_server_limit（dynamic） | ✅ 状态字段 4 处<br>❌ 其他字段和数量 |
| subscription.deleted | ✅ 总是（找到 team） | 无直接字段，全部走 subscriptionEnded() | ✅ WHERE 双条件匹配 |
| RefundSubscription | ✅ 总是 | stripe_refunded_at, stripe_feedback, stripe_comment,<br>stripe_invoice_paid 等（见 3.4） | ✅ 四道前置校验 |

### 9.2 订阅创建流程
```
checkout.session.completed
    ↓
Subscription::updateOrCreate(team_id)
    → stripe_invoice_paid = true
    → stripe_past_due = false
```

### 9.3 订阅状态变更流程
```
customer.subscription.updated
    ↓
Step 1: 无条件更新 feedback/comment/plan_id/cancel_at_period_end
Step 2: dynamic 套餐 → custom_server_limit = min(qty, 100) → ServerLimitCheckJob
Step 3: 按 status 分支（需通过 subscription_id 一致性校验）
        ├─ active → stripe_invoice_paid = true, stripe_past_due = false
        ├─ past_due → stripe_past_due = true
        ├─ paused/incomplete_expired → stripe_invoice_paid = false
        └─ unpaid → stripe_invoice_paid = false + subscriptionEnded()
```

### 9.4 发票支付流程
```
invoice.paid
    ↓
主动调用 Stripe subscriptions.retrieve 获取最新状态（不信任事件数据）
    ↓
    ├─ active/past_due → 更新对应字段
    ├─ canceled/unpaid/incomplete_expired → 仅发通知，不动作
    └─ 其他/异常 → 延迟 20s → VerifyStripeSubscriptionStatusJob（3次重试兜底）
```

### 9.5 退款取消流程
```
RefundSubscription::execute(team)
    ↓
checkEligibility() 四道前置（短路式）：
    1. stripe_refunded_at 为空？
    2. stripe_subscription_id 存在？
    3. stripe_invoice_paid = true？
    4. Stripe status ∈ [active, trialing] 且 start_date 距今 ≤ 30 天？
    ↓
1. Stripe refunds.create（退款）
2. 写入 stripe_refunded_at = now() ← 立即写库防重复
3. Stripe subscriptions.cancel（取消订阅）
4. 更新本地状态字段（invoice_paid = false 等）
5. team->subscriptionEnded() → 清除订阅 + 禁用全部服务器
```

### 9.6 用户主动变更套餐数量流程
```
UpdateSubscriptionQuantity::execute(team, quantity)
    ↓
前置校验：quantity >= 2（MIN_SERVER_LIMIT）
前置校验：subscription 存在且 stripe_invoice_paid = true
    ↓
1. Stripe subscriptions.update + proration_behavior=always_invoice
2. 检查 latest_invoice.status
    ├─ paid → custom_server_limit = quantity → ServerLimitCheckJob
    └─ 未支付 → 回滚 Stripe 数量 + voidInvoice + 返回错误
    ↓
后续：Stripe 会发送 customer.subscription.updated webhook
    → Webhook 用 min(quantity, 100) 再做一次上限保护
```

---

## 十、关键设计亮点

1. **最终一致性优先**：多处兜底校验 + 延迟重试，不依赖单次事件投递顺序
2. **状态机单向流转**：失败事件有二次确认机制，不会覆盖已确认的成功状态
3. **先标记后操作**：退款先写 `stripe_refunded_at` 防重复，即使后续步骤失败也安全
4. **延迟 + 退避**：支付失败延迟 60 秒，给 Stripe 自动重试留足时间窗口
5. **上下限保护不对称**：Webhook 路径只设上限（信任 Stripe），用户主动操作加下限（防误输入）
6. **按时间倒序禁用**：服务器超限时先禁用最新创建的，保护用户的核心业务服务器
7. **ID 校验分级**：subscriptionEnded/invoice_paid 等破坏性操作强制校验，feedback/comment 等元数据宽松处理
