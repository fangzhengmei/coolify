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
- [CancelSubscription.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/CancelSubscription.php) - 取消订阅 Action（2 处调用点）
- [VerifyStripeSubscriptionStatusJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/VerifyStripeSubscriptionStatusJob.php) - 订阅状态校验
- [SyncStripeSubscriptionsJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/SyncStripeSubscriptionsJob.php) - 全量同步 Job
- [CloudFixSubscription.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Console/Commands/Cloud/CloudFixSubscription.php) - 命令行修复（3 处调用点）
- [Actions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Livewire/Subscription/Actions.php) - 前端取消入口
- [Team.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Team.php) - 团队模型（含套餐限制）
- [Subscription.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Subscription.php) - 订阅模型
- [Server.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Server.php) - 服务器模型（邮件抑制）

辅助 Job：
- [SubscriptionInvoiceFailedJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/SubscriptionInvoiceFailedJob.php) - 支付失败处理
- [ServerLimitCheckJob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/ServerLimitCheckJob.php) - 服务器数量超限检查

---

## 二、subscriptionEnded 十一处调用点全景

`subscriptionEnded()` 是终止团队所有服务的核心方法，共 **11 处调用点**，分五类：

### 2.1 Webhook 触发（2 处）

| # | 调用位置 | 触发事件 | 触发条件 |
|---|---------|---------|---------|
| 1 | [StripeProcessJob.php:302](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L302) | `customer.subscription.updated` | `status === 'unpaid'` 且通过 subscription_id 校验 |
| 2 | [StripeProcessJob.php:331](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L331) | `customer.subscription.deleted` | 通过 `WHERE stripe_customer_id AND stripe_subscription_id` 双条件匹配 |

### 2.2 Action 入口触发（6 处 = Refund 1 + Cancel 2 + Sync 1 + Verify 1 + Livewire 1）

| # | 调用位置 | 触发场景 | 触发条件 |
|---|---------|---------|---------|
| 3 | [RefundSubscription.php:128](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/RefundSubscription.php#L128) | 退款申请成功 | 四道前置校验全部通过（见第五章） |
| 4 | [VerifyStripeSubscriptionStatusJob.php:87](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/VerifyStripeSubscriptionStatusJob.php#L87) | 兜底校验发现已取消 | Stripe 端 status ∈ `['canceled', 'incomplete_expired', 'unpaid']` |
| 5 | [SyncStripeSubscriptionsJob.php:99](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/SyncStripeSubscriptionsJob.php#L99) | 全量同步发现不一致 | `$fix = true` 且 Stripe 端 `status === 'canceled'` |
| 6 | [CancelSubscription.php:168](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/CancelSubscription.php#L168) | 用户账户删除 | `cancelSingleSubscription()` 内，取消后调用 |
| 7 | [CancelSubscription.php:198](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/CancelSubscription.php#L198) | 按 subscriptionId 强制取消 | `cancelById()` 静态方法，取消后调用 |
| 8 | [Actions.php:145](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Livewire/Subscription/Actions.php#L145) | 用户在订阅页面点击"立即取消" | 密码验证通过 + Stripe cancel API 成功 |

### 2.3 CloudFix 命令行触发（3 处）

| # | 调用位置 | 触发场景 | 触发条件 |
|---|---------|---------|---------|
| 9 | [CloudFixSubscription.php:209](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Console/Commands/Cloud/CloudFixSubscription.php#L209) | 修复已取消订阅 | `--fix-canceled-subs` 且 Stripe 返回 `status === 'canceled'` |
| 10 | [CloudFixSubscription.php:281](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Console/Commands/Cloud/CloudFixSubscription.php#L281) | 修复缺失订阅（Stripe 查不到） | `--fix-canceled-subs` 且 Stripe 返回 `resource_missing` |
| 11 | [CloudFixSubscription.php:727](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Console/Commands/Cloud/CloudFixSubscription.php#L727) | 通用修复入口 | `fixSubscription()` 私有方法内，status 非有效时调用 |

### 2.4 十一处调用点差异对照表

| 调用点 # | 前置校验 | 先更新字段 | team 存在性检查 |
|---------|---------|-----------|----------------|
| 1 Webhook updated (unpaid) | subscription_id === | `stripe_invoice_paid = false` | `if ($team)` |
| 2 Webhook deleted | WHERE 双条件 | 无（全部走 subscriptionEnded） | `if ($team)` |
| 3 RefundSubscription | 4 道前置 | 7 个字段全量重置 | `if ($team)` |
| 4 VerifyJob (3 case) | Stripe API 拉取状态 | `stripe_invoice_paid = false` | `if ($team)` |
| 5 SyncJob (`$fix=true`) | 全量比对 + status=canceled | `stripe_invoice_paid = false` | `$subscription->team?->` 安全调用 |
| 6 CancelSubscription #168 | owner 权限 + invoice_paid | 6 个字段重置 | `if ($subscription->team)` |
| 7 CancelSubscription #198 | isCloud() + subscription 存在 | 4 个字段重置 | `if ($subscription->team)` |
| 8 Actions (Livewire) | 密码验证 + Stripe API | 6 个字段重置 | 直接调用（currentTeam 必然存在） |
| 9 CloudFix #209 | --fix-canceled-subs + status=canceled | 无，直接调 | 直接调用（$team 已查） |
| 10 CloudFix #281 | --fix-canceled-subs + resource_missing | 无，直接调 | 直接调用（$team 已查） |
| 11 CloudFix #727 | fixSubscription() 方法内 | 无，直接调 | 直接调用（$team 已传入） |

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
        │             │ stripe_cancel_at_period_end = Stripe 返回值        │
        ├─────────────┼──────────────────────────────────────────────────┤
        │ past_due    │ stripe_invoice_paid = true  ← 注意：仍标记已支付   │
        │             │ stripe_past_due = true                            │
        │             │ stripe_cancel_at_period_end = Stripe 返回值        │
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

### 3.2 customer.subscription.created — 订阅创建事件

**代码位置**：[StripeProcessJob.php:203-229](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L203-L229)

```php
// Step 1: 从 metadata 提取 team_id 和 user_id
$teamId = data_get($data, 'metadata.team_id');
$userId = data_get($data, 'metadata.user_id');

// Step 2: 无 metadata 时报错（区分"已存在"与"完全缺失"）
if (! $teamId || ! $userId) {
    $subscription = Subscription::where('stripe_customer_id', $customerId)->first();
    if ($subscription) {
        throw new \RuntimeException("Subscription already exists for customer: {$customerId}");
    }
    throw new \RuntimeException('No team id or user id found');
}

// Step 3: 校验 user 是否为 team 的 admin/owner
$team = Team::find($teamId);
$found = $team->members->where('id', $userId)->first();
if (! $found->isAdmin()) {
    throw new \RuntimeException("User {$userId} is not an admin or owner of team {$team->id}");
}

// Step 4: 按 team_id updateOrCreate（invoice_paid 设为 false）
Subscription::updateOrCreate(
    ['team_id' => $teamId],
    [
        'stripe_subscription_id' => $subscriptionId,
        'stripe_customer_id' => $customerId,
        'stripe_invoice_paid' => false,  // ← 创建时不标记已支付
    ]
);
```

**与 checkout.session.completed 的分叉对比**：

| 对比项 | `checkout.session.completed` | `customer.subscription.created` |
|-------|-----------------------------|--------------------------------|
| 代码位置 | [L61-86](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L61-L86) | [L203-229](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L203-L229) |
| 触发时机 | Checkout 支付**完成后** | Stripe 端订阅对象**创建时**（可能支付未完成） |
| ID 来源 | `client_reference_id` 解析 `userId:teamId` | `metadata.team_id` + `metadata.user_id` |
| 权限校验 | `$found->isAdmin()` | `$found->isAdmin()` |
| 创建方式 | `updateOrCreate(['team_id' => $teamId], ...)` | `updateOrCreate(['team_id' => $teamId], ...)` |
| `stripe_invoice_paid` | ✅ **设置为 true** | ❌ **设为 false** |
| `stripe_past_due` | `false` | 不设置（保持默认） |
| 典型时序 | 先收到 subscription.created → 支付完成 → 收到 checkout.session.completed 覆盖 | 通常先于 checkout 事件到达 |

---

### 3.3 customer.subscription.updated — 订阅更新事件（核心）

**代码位置**：[StripeProcessJob.php:230-323](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L230-L323)

**字段变更全景（按代码执行顺序）**：

```
Step 1: 订阅记录定位
        → 优先按 stripe_customer_id 查
        → 查不到且 status=incomplete_expired → 抛异常
        → 查不到但有 team_id → firstOrCreate 新记录（stripe_invoice_paid = false）

Step 2: 无条件更新字段 [L272-277] ← 无 ID 校验！
        stripe_feedback = cancellation_details.feedback
        stripe_comment  = cancellation_details.comment
        stripe_plan_id  = items.data.0.plan.id
        stripe_cancel_at_period_end = cancel_at_period_end

Step 3: dynamic 套餐数量更新 [L262-271] ← 无 ID 校验！
        → lookup_key 包含 'dynamic' 才处理
        → custom_server_limit = min(quantity, 100)  ← 只有上限 100，无下限检查！
        → 触发 ServerLimitCheckJob

Step 4: 状态分支（均带 subscription_id 一致性校验 [L279/L286/L294/L309]）
        ┌────────────────────┬─────────────────────────────────────────┬────────────────────┐
        │ Stripe status      │ 本地字段变更（需通过 ID 校验）            │ subscriptionEnded? │
        ├────────────────────┼─────────────────────────────────────────┼────────────────────┤
        │ paused             │ stripe_invoice_paid = false             │ ❌ 不触发          │
        │ incomplete_expired │                                         │                    │
        ├────────────────────┼─────────────────────────────────────────┼────────────────────┤
        │ past_due           │ stripe_past_due = true                  │ ❌ 不触发          │
        ├────────────────────┼─────────────────────────────────────────┼────────────────────┤
        │ unpaid             │ stripe_invoice_paid = false             │ ✅ 触发（调用点1） │
        │                    │                                         │ → 禁用全部服务器  │
        ├────────────────────┼─────────────────────────────────────────┼────────────────────┤
        │ active             │ stripe_past_due = false                 │ ❌ 不触发          │
        │                    │ stripe_invoice_paid = true              │                    │
        └────────────────────┴─────────────────────────────────────────┴────────────────────┘
```

**⚠️ subscriptionEnded 触发条件（调用点 1）**：
- **仅当 `status === 'unpaid'` 时触发**
- `canceled` 状态不在此事件处理，靠 `customer.subscription.deleted` 事件触发终止

**⚠️ unpaid 分支的 ID 校验范围**：注意 unpaid 的 subscriptionEnded 调用 [L300-306](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L300-L306) 在 if 校验块**之外**，即：即使 subscription_id 不匹配导致 stripe_invoice_paid 没更新，只要 status=unpaid 且找到了 team，**仍然会触发 subscriptionEnded**。

**ID 校验范围**：状态字段变更前必须通过 `$subscription->stripe_subscription_id === $subscriptionId` 校验，涉及 4 处：
- [L279-283](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L279-L283) - paused / incomplete_expired
- [L286-291](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L286-L291) - past_due
- [L294-299](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L294-L299) - unpaid（字段更新，不含 subscriptionEnded）
- [L309-314](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L309-L314) - active

---

### 3.4 customer.subscription.deleted — 订阅删除事件

**代码位置**：[StripeProcessJob.php:324-340](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L324-L340)

```php
$subscription = Subscription::where('stripe_customer_id', $customerId)
    ->where('stripe_subscription_id', $subscriptionId)  // ← 双条件精确匹配
    ->first();

if ($subscription) {
    $team = $subscription->team;
    if ($team) {
        $team->subscriptionEnded();  // ✅ 必触发（调用点2）
    } else {
        throw new RuntimeException("No team found");
    }
} else {
    break;  // 找不到记录静默忽略，不抛异常
}
```

**⚠️ 重要差异（调用点 2）**：
- 此事件 **一定触发** `subscriptionEnded()`（只要 team 存在）
- ID 校验方式不是松散比较，而是 `WHERE stripe_customer_id AND stripe_subscription_id` **双条件精确匹配**
- 找不到 subscription 时**静默 break 不抛异常**（与 subscription.updated 的 `throw new RuntimeException` 不同）

---

### 3.5 radar.early_fraud_warning.created — 欺诈预警（直接退款取消，不走 RefundSubscription）

**代码位置**：[StripeProcessJob.php:38-60](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L38-L60)

```php
case 'radar.early_fraud_warning.created':
    $stripe = new StripeClient(config('subscription.stripe_api_key'));
    $id = data_get($data, 'id');
    $charge = data_get($data, 'charge');

    // Step 1: 直接调 Stripe Refunds API 退款（不走 RefundSubscription Action）
    if ($charge) {
        $stripe->refunds->create(['charge' => $charge]);
    }

    // Step 2: 通过 payment_intent 反查 customer
    $pi = data_get($data, 'payment_intent');
    $piData = $stripe->paymentIntents->retrieve($pi, []);
    $customerId = data_get($piData, 'customer');

    // Step 3: 取消订阅 + 更新本地状态
    $subscription = Subscription::where('stripe_customer_id', $customerId)->first();
    if ($subscription) {
        $subscriptionId = data_get($subscription, 'stripe_subscription_id');
        $stripe->subscriptions->cancel($subscriptionId, []);
        $subscription->update([
            'stripe_invoice_paid' => false,
        ]);
        send_internal_notification("Early fraud warning: Refunded + canceled");
    } else {
        send_internal_notification("Early fraud warning: subscription not found");
        throw new \RuntimeException("Early fraud warning: subscription not found");
    }
    break;
```

**⚠️ 重要纠正**：radar 事件**不走 RefundSubscription Action**，也**不调用 subscriptionEnded()**，处理方式是：
1. 直接 `$stripe->refunds->create()` 退款
2. 直接 `$stripe->subscriptions->cancel()` 取消订阅
3. 仅更新 `stripe_invoice_paid = false`
4. 后续由 Stripe 发送的 `customer.subscription.deleted` webhook 来触发 subscriptionEnded（调用点 2）

因此 radar 链路是：**radar webhook → 直接退款取消 → Stripe 发 subscription.deleted → subscriptionEnded（调用点2）**

---

### 3.6 subscriptionEnded 内部字段变更清单

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
        → server.unreachable_notification_sent = true  ← 抑制邮件标记（见第九章）
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
前置 1 [L28-30]: stripe_refunded_at 为空？
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
| `unpaid`（字段更新） | [L294-299](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L294-L299) | `stripe_invoice_paid = false` |
| `active` | [L309-314](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L309-L314) | `stripe_past_due = false` + `stripe_invoice_paid = true` |

**⚠️ 不覆盖的范围**：
- L272-277 的字段更新（`stripe_feedback` / `stripe_comment` / `stripe_plan_id` / `stripe_cancel_at_period_end`）**没有 ID 校验**，直接更新
- L262-271 的 dynamic 套餐数量更新**没有 ID 校验**，直接更新 team 的 `custom_server_limit`
- **unpaid 的 subscriptionEnded 调用 [L300-306]** 在 if 块**之外**，即：即使 ID 不匹配，只要找到了 team 也会触发 subscriptionEnded
- `canceled` 状态本事件不处理，无校验机会

**场景风险**：如果用户新订阅刚创建完成，旧订阅的 `subscription.updated` 延迟事件到达（status=unpaid）—— feedback/comment/plan_id/cancel_at_period_end 和 custom_server_limit 会被旧数据覆盖，且 **subscriptionEnded 一定会被触发**（因为在 if 块外），只有 invoice_paid/past_due 字段更新受 ID 校验保护。

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
| subscription.updated | `===` 松散比较 | 仅 4 个状态分支的字段更新（不含 unpaid 的 subscriptionEnded） | 跳过字段更新，但仍执行无校验部分（feedback/plan_id/数量 + unpaid subscriptionEnded） |
| subscription.deleted | WHERE 双条件精确匹配 | 整个事件逻辑（subscriptionEnded） | 静默 break，什么都不做 |

---

## 七、paymentIntent 白名单四项 + payment_failed 派发条件

### 7.1 paymentIntent 白名单四项

**代码位置**：[StripeProcessJob.php:172](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L172)

在 `invoice.payment_failed` 处理中，先调用 Stripe API 二次确认支付意图状态，若处于以下四种状态之一则认为支付仍在进行/已成功，**不处理失败**：

```php
if (in_array($paymentIntent->status, [
    'processing',           // 处理中
    'succeeded',            // 已成功
    'requires_action',      // 需要用户额外操作（如 3DS 验证）
    'requires_confirmation' // 需要确认（用户在支付流程中）
])) {
    break; // 状态已改变，不处理失败
}
```

**四项状态含义**：
| 状态 | 含义 | 处理方式 |
|-----|------|---------|
| `processing` | 银行/卡组织处理中 | 忽略，等待后续事件 |
| `succeeded` | 支付已成功 | 忽略，状态已恢复 |
| `requires_action` | 需要用户交互（3DS、验证码） | 忽略，等待用户完成 |
| `requires_confirmation` | 需要商户确认支付 | 忽略，支付流程未结束 |

---

### 7.2 payment_failed 派发：以 `! stripe_invoice_paid` 为最终闸门

**代码位置**：[StripeProcessJob.php:176-187](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L176-L187)

两种派发策略，**两种都以 `! $subscription->stripe_invoice_paid` 为准入条件**：

```php
// 场景 A：新建订阅且 5 分钟内的支付失败 → 延迟 60 秒派发
// 准入条件（两个都要满足）：
//   1. ! $subscription->stripe_invoice_paid  ← 关键闸门
//   2. $subscription->created_at->diffInMinutes(now()) < 5
if (! $subscription->stripe_invoice_paid && $subscription->created_at->diffInMinutes(now()) < 5) {
    SubscriptionInvoiceFailedJob::dispatch($team)->delay(now()->addSeconds(60));
    break;
}

// 场景 B：其他支付失败 → 立即派发（无 delay）
// 准入条件（只要一个）：
//   ! $subscription->stripe_invoice_paid  ← 关键闸门（唯一条件）
if (! $subscription->stripe_invoice_paid) {
    SubscriptionInvoiceFailedJob::dispatch($team);
    break;
}
```

**⚠️ 关键纠正**：`delay(60s)` 的延迟派发有两个条件（新建+5分钟内），但两种派发**最终都要通过 `! stripe_invoice_paid` 这个闸门**。如果本地 stripe_invoice_paid 已经是 true（说明在此期间 invoice.paid 事件已先处理完），则两种派发都不会执行，直接 fall through 到 break 结束。

**两种派发策略对比**：

| 场景 | 准入条件 | 派发方式 | 原因 |
|-----|---------|---------|------|
| A | `invoice_paid = false` + 创建时间 < 5 分钟 | `delay(60s)` | 给 Stripe 自动重试和用户操作留足时间 |
| B | `invoice_paid = false`（且不满足 A） | 立即 `dispatch` | 非首次支付失败，直接通知用户 |

**补充**：`payment_intent.payment_failed` 事件 [L189-202](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L189-L202) 仅做检查不派发 Job：如果 stripe_invoice_paid 已经是 true 则直接 return，否则只打日志不发通知。

---

## 八、VerifyStripeSubscriptionStatusJob 拆三 case + customer 反查 self-recovery

**代码位置**：[VerifyStripeSubscriptionStatusJob.php:26-103](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/VerifyStripeSubscriptionStatusJob.php#L26-L103)

### 8.1 Job 配置

```php
public int $tries = 3;           // 最多重试 3 次
public array $backoff = [10, 30, 60]; // 退避：10s → 30s → 60s
```

### 8.2 Step 1：customer 反查 self-recovery（补救缺失的 subscription_id）

**代码位置**：[L28-46](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/VerifyStripeSubscriptionStatusJob.php#L28-L46)

```php
// 如果本地有 stripe_customer_id 但还没有 stripe_subscription_id，
// 主动向 Stripe 反查该 customer 的最新订阅来补全 ID
if (! $this->subscription->stripe_subscription_id &&
    $this->subscription->stripe_customer_id) {
    try {
        $stripe = new \Stripe\StripeClient(config('subscription.stripe_api_key'));
        $subscriptions = $stripe->subscriptions->all([
            'customer' => $this->subscription->stripe_customer_id,
            'limit' => 1,
        ]);

        if ($subscriptions->data) {
            // ✅ self-recovery：把查到的最新 subscription_id 回填本地
            $this->subscription->update([
                'stripe_subscription_id' => $subscriptions->data[0]->id,
            ]);
        }
    } catch (\Exception $e) {
        // 静默忽略，继续执行（没有 ID 就 return 退出）
    }
}

// 反查后仍没有 ID，直接退出
if (! $this->subscription->stripe_subscription_id) {
    return;
}
```

### 8.3 Step 2：拆三 case 状态映射

**代码位置**：[L58-97](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/VerifyStripeSubscriptionStatusJob.php#L58-L97)

```php
switch ($stripeSubscription->status) {
    // ─── Case 1: active ───
    case 'active':
        $this->subscription->update([
            'stripe_invoice_paid' => true,
            'stripe_past_due' => false,
            'stripe_cancel_at_period_end' => $stripeSubscription->cancel_at_period_end,
        ]);
        break;

    // ─── Case 2: past_due ───
    case 'past_due':
        // 保留有效状态，但标记逾期（invoice_paid 仍为 true，与 invoice.paid 一致）
        $this->subscription->update([
            'stripe_invoice_paid' => true,   // ← 注意：past_due 仍算已支付
            'stripe_past_due' => true,
            'stripe_cancel_at_period_end' => $stripeSubscription->cancel_at_period_end,
        ]);
        break;

    // ─── Case 3: canceled / incomplete_expired / unpaid（合并处理）───
    case 'canceled':
    case 'incomplete_expired':
    case 'unpaid':
        $this->subscription->update([
            'stripe_invoice_paid' => false,
            'stripe_past_due' => false,
        ]);
        $team = $this->subscription->team;
        if ($team) {
            $team->subscriptionEnded();  // ✅ 调用点 4
        }
        break;

    default:
        send_internal_notification("Unknown status: {$stripeSubscription->status}");
        break;
}
```

**三 case 字段变更对照表**：

| Case | Stripe status | invoice_paid | past_due | cancel_at_period_end | subscriptionEnded |
|------|--------------|-------------|----------|---------------------|-------------------|
| 1 | active | ✅ true | ❌ false | ✅ 同步 Stripe 值 | ❌ |
| 2 | past_due | ✅ true | ✅ true | ✅ 同步 Stripe 值 | ❌ |
| 3 | canceled/incomplete_expired/unpaid | ❌ false | ❌ false | 不更新 | ✅ 触发（调用点4） |

**重试与自愈总结**：
1. **触发时机**：invoice.paid 遇到未知状态、或 webhook 处理异常时延迟 20s 调度
2. **customer 反查**：有 customer_id 但缺 subscription_id 时，主动从 Stripe 拉取回填（self-recovery）
3. **重试策略**：3 次重试，退避 10s→30s→60s
4. **自愈能力**：三 case 双向修正（本地滞后/超前都能修正），Case 3 触发 subscriptionEnded
5. **极端情况**：3 次重试全部失败后，进入 failed_jobs 表，需人工介入

---

## 九、服务器邮件抑制机制单列

当 `subscriptionEnded()` 被触发时，会批量禁用团队所有服务器并触发 `ServerReachabilityChanged` 事件。为避免向用户发送大量"服务器不可达"邮件，系统设计了三层抑制机制：

### 9.1 第一层：subscriptionEnded 内直接打标记

**代码位置**：[Team.php:238-241](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Team.php#L238-L241)

```php
foreach ($this->servers as $server) {
    $server->settings()->update(['is_usable' => false, 'is_reachable' => false]);
    ServerReachabilityChanged::dispatch($server);
    $server->unreachable_count = 3;           // ← 直接设为阈值
    $server->unreachable_notification_sent = true;  // ← 打抑制标记
    $server->save();
}
```

**两个关键设置**：
- `unreachable_count = 3`：直接达到发送通知的阈值（正常需 ≥2）
- `unreachable_notification_sent = true`：标记"已发送过通知"，阻止后续重复发送

---

### 9.2 第二层：isReachableChanged 检查抑制标记

**代码位置**：[Server.php:1235-1252](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Server.php#L1235-L1252)

```php
public function isReachableChanged()
{
    $this->refresh();
    $unreachableNotificationSent = (bool) $this->unreachable_notification_sent;
    $isReachable = (bool) $this->settings->is_reachable;

    if ($isReachable === true) {
        // 服务器恢复：仅当之前发过不可达通知时，才发恢复通知
        if ($unreachableNotificationSent === true) {
            $this->sendReachableNotification();  // 内部会重置标记为 false
        }
        return;
    }

    // 服务器不可达：需要同时满足两个条件才发通知
    // 1. unreachable_count >= 2（已达到阈值）
    // 2. unreachable_notification_sent === false（未发过）
    if ($this->unreachable_count >= 2 && ! $unreachableNotificationSent) {
        $this->sendUnreachableNotification();  // 内部会设置标记为 true
    }
}
```

**抑制逻辑真值表**：

| 场景 | is_reachable | unreachable_count | unreachable_notification_sent | 结果 |
|-----|-------------|------------------|-------------------------------|------|
| subscriptionEnded 触发 | false | 3 | true | ❌ 不发送（抑制生效） |
| 服务器真的不可达（第一次） | false | 1 | false | ❌ 不发送（count < 2） |
| 服务器真的不可达（第二次） | false | 2 | false | ✅ 发送 Unreachable 通知 |
| 服务器真的不可达（第三次） | false | 3 | true | ❌ 不发送（已发过） |
| 服务器从不可达恢复 | true | 0 | true | ✅ 发送 Reachable 通知（并重置标记） |
| 服务器从不可达恢复 | true | 0 | false | ❌ 不发送（之前没发过不可达） |

---

### 9.3 第三层：sendUnreachableNotification 内写标记

**代码位置**：[Server.php:1262-1274](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Server.php#L1262-L1274)

```php
public function sendUnreachableNotification()
{
    $this->unreachable_notification_sent = true;  // ← 发之前先打标记，防止并发重复
    $this->save();
    $this->team->notify(new Unreachable($this));
}

public function sendReachableNotification()
{
    $this->unreachable_notification_sent = false;  // ← 发恢复通知时重置标记
    $this->save();
    $this->team->notify(new Reachable($this));
}
```

**关键设计**：发送通知**之前**先写数据库标记，即使通知发送失败也不会重复触发。

---

## 十、重复回调事件处理机制

### 10.1 重复回调的产生原因

Stripe Webhook **至少投递一次**（at-least-once delivery），可能因：
- 网络超时导致 Stripe 未收到 200 响应
- 同一事件多次投递（通常间隔递增）
- 多个事件类型描述同一次状态变更

### 10.2 幂等性保障措施

#### 措施 1：updateOrCreate / firstOrCreate 防重复

**[StripeProcessJob.php:77-85](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L77-L85)**

```php
Subscription::updateOrCreate(
    ['team_id' => $teamId],  // 唯一键：每个 team 只有一条订阅
    [/* 要更新的字段 */]
);
```

- 以 `team_id` 为唯一键，重复事件只会更新同一条记录
- `subscription.created` 和 `checkout.session.completed` 都使用 `updateOrCreate` 按 `team_id` 定位
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

通过状态分支前的 ID 一致性比较，防止旧订阅事件误改新订阅的关键字段（但 unpaid 的 subscriptionEnded 在 if 块外，不受此保护）。

---

## 十一、账单状态竞争条件处理

### 11.1 竞争场景分析

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

### 11.2 竞争防护机制

#### 机制 1：支付失败事件的白名单四项 + 延迟 + invoice_paid 闸门

**[StripeProcessJob.php:166-187](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L166-L187)**

```php
// 1. 先调用 Stripe API 确认支付意图状态（白名单四项）
$paymentIntent = $stripe->paymentIntents->retrieve($paymentIntentId);
if (in_array($paymentIntent->status, ['processing', 'succeeded', 'requires_action', 'requires_confirmation'])) {
    break; // 状态已改变，不处理失败
}

// 2. 新建订阅且 5 分钟内的失败，延迟 60 秒再检查
if (! $subscription->stripe_invoice_paid && $subscription->created_at->diffInMinutes(now()) < 5) {
    SubscriptionInvoiceFailedJob::dispatch($team)->delay(now()->addSeconds(60));
    break;
}

// 3. 最终闸门：只有 stripe_invoice_paid 仍为 false 时才派发失败通知
if (! $subscription->stripe_invoice_paid) {
    SubscriptionInvoiceFailedJob::dispatch($team);
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

#### 机制 4：VerifyStripeSubscriptionStatusJob 兜底 + customer 反查 self-recovery

**[StripeProcessJob.php:132-147](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L132-L147)**

遇到未知状态或 API 调用失败时，延迟 20 秒后调度校验 Job：
```php
VerifyStripeSubscriptionStatusJob::dispatch($subscription)
    ->delay(now()->addSeconds(20));
```
退避策略：`backoff = [10, 30, 60]` 秒，最多重试 3 次。如果本地缺少 subscription_id，会先通过 customer_id 反查 Stripe 回填（self-recovery）。

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

## 十二、完整同步路径总结

### 12.1 十一处 subscriptionEnded 调用点汇总表

| # | 触发源分类 | 触发源 | 调用位置 | 触发条件 |
|---|----------|-------|---------|---------|
| 1 | Webhook | customer.subscription.updated | [StripeProcessJob.php:302](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L302) | status=unpaid（在 ID 校验 if 块外，即使 ID 不匹配也触发） |
| 2 | Webhook | customer.subscription.deleted | [StripeProcessJob.php:331](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L331) | 双条件匹配成功 |
| 3 | Action | RefundSubscription | [RefundSubscription.php:128](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/RefundSubscription.php#L128) | 四道前置全部通过 |
| 4 | Job | VerifyStripeSubscriptionStatusJob | [VerifyStripeSubscriptionStatusJob.php:87](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/VerifyStripeSubscriptionStatusJob.php#L87) | Case 3: status ∈ [canceled, incomplete_expired, unpaid] |
| 5 | Job | SyncStripeSubscriptionsJob | [SyncStripeSubscriptionsJob.php:99](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/SyncStripeSubscriptionsJob.php#L99) | $fix=true + Stripe 端 status=canceled |
| 6 | Action | CancelSubscription::cancelSingleSubscription | [CancelSubscription.php:168](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/CancelSubscription.php#L168) | 用户账户删除 |
| 7 | Action | CancelSubscription::cancelById | [CancelSubscription.php:198](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/CancelSubscription.php#L198) | 按 ID 强制取消 |
| 8 | Livewire | Subscription/Actions | [Actions.php:145](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Livewire/Subscription/Actions.php#L145) | 用户点击"立即取消" |
| 9 | Command | CloudFixSubscription | [CloudFixSubscription.php:209](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Console/Commands/Cloud/CloudFixSubscription.php#L209) | --fix-canceled-subs + status=canceled |
| 10 | Command | CloudFixSubscription | [CloudFixSubscription.php:281](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Console/Commands/Cloud/CloudFixSubscription.php#L281) | --fix-canceled-subs + resource_missing |
| 11 | Command | CloudFixSubscription::fixSubscription | [CloudFixSubscription.php:727](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Console/Commands/Cloud/CloudFixSubscription.php#L727) | fixSubscription() 通用修复入口 |

### 12.2 三大事件源字段变更对照表

| 事件源 | 触发 subscriptionEnded | 直接更新字段 | 需 ID 校验 |
|-------|----------------------|-------------|-----------|
| invoice.paid | ❌ 从不 | stripe_invoice_paid, stripe_past_due, stripe_cancel_at_period_end | ❌ 无（但主动拉 Stripe） |
| subscription.created | ❌ 从不 | stripe_subscription_id, stripe_customer_id, stripe_invoice_paid=false | ❌ 无（按 team_id updateOrCreate） |
| subscription.updated | ✅ status=unpaid（调用点1，在 if 块外） | stripe_feedback, stripe_comment,<br>stripe_plan_id, stripe_cancel_at_period_end,<br>stripe_invoice_paid, stripe_past_due,<br>custom_server_limit（dynamic） | ✅ 状态字段 4 处<br>❌ 其他字段/数量/unpaid subscriptionEnded |
| subscription.deleted | ✅ 总是（调用点2） | 无直接字段，全部走 subscriptionEnded() | ✅ WHERE 双条件匹配 |
| RefundSubscription | ✅ 总是（调用点3） | stripe_refunded_at, stripe_feedback, stripe_comment,<br>stripe_invoice_paid 等 | ✅ 四道前置校验 |
| VerifyJob (Case 3) | ✅ status ∈ [canceled, incomplete_expired, unpaid]（调用点4） | stripe_invoice_paid, stripe_past_due | ✅ Stripe API 二次校验 |
| SyncJob | ✅ $fix=true + status=canceled（调用点5） | stripe_invoice_paid, stripe_past_due | ✅ 全量比对 |
| CancelSubscription #168 | ✅ 总是（调用点6） | 6 个字段重置 | ✅ owner 权限 |
| CancelSubscription #198 | ✅ 总是（调用点7） | 4 个字段重置 | ✅ isCloud() + 存在性检查 |
| Actions (Livewire) | ✅ 总是（调用点8） | 6 个字段重置 | ✅ 密码验证 |
| CloudFix × 3 | ✅ 总是（调用点9/10/11） | 无，直接调 | ✅ --fix 标志 |

### 12.3 订阅创建流程（subscription.created 与 checkout.session.completed 分叉）
```
customer.subscription.created （先到达）
    ↓
从 metadata 提取 team_id, user_id
    ↓
$found->isAdmin() 权限校验
    ↓
Subscription::updateOrCreate(['team_id' => $teamId], ...)
    → stripe_invoice_paid = false（未支付）
    ↓
（支付完成后）
    ↓
checkout.session.completed （后到达）
    ↓
从 client_reference_id 解析 userId:teamId
    ↓
$found->isAdmin() 权限校验
    ↓
Subscription::updateOrCreate(['team_id' => $teamId], ...)
    → stripe_invoice_paid = true  ✅ 覆盖为已支付
    → stripe_past_due = false
```

### 12.4 订阅状态变更流程
```
customer.subscription.updated
    ↓
Step 1: 无条件更新 feedback/comment/plan_id/cancel_at_period_end
Step 2: dynamic 套餐 → custom_server_limit = min(qty, 100) → ServerLimitCheckJob
Step 3: 按 status 分支（字段更新需通过 subscription_id 校验，unpaid 的 subscriptionEnded 在 if 外）
        ├─ active → stripe_invoice_paid = true, stripe_past_due = false
        ├─ past_due → stripe_past_due = true
        ├─ paused/incomplete_expired → stripe_invoice_paid = false
        └─ unpaid → stripe_invoice_paid = false（if 内）+ subscriptionEnded()（if 外，调用点1）
```

### 12.5 radar 欺诈预警退款链（不走 RefundSubscription，直接操作）
```
radar.early_fraud_warning.created
    ↓
Step 1: 直接调 $stripe->refunds->create(['charge' => $charge]) 退款
Step 2: 通过 payment_intent 反查 customer
Step 3: $stripe->subscriptions->cancel() 直接取消订阅
Step 4: 本地仅更新 stripe_invoice_paid = false
    ↓
（后续由 Stripe 发送 customer.subscription.deleted）
    ↓
customer.subscription.deleted webhook → subscriptionEnded()（调用点2）
```

### 12.6 支付失败流程
```
invoice.payment_failed
    ↓
Stripe API 二次确认 paymentIntent.status
    ├─ ∈ [processing, succeeded, requires_action, requires_confirmation] → break（白名单