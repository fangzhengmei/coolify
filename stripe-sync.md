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

### 路径 3：外部事件 → 本地状态映射规则

#### 3.1 订阅状态映射（Stripe → 本地）

**核心映射逻辑在 [StripeProcessJob.php:108-136](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L108-L136) 和 [StripeProcessJob.php:278-315](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L278-L315)**

| Stripe 状态 | 本地字段更新 | 说明 |
|------------|-------------|------|
| `active` | `stripe_invoice_paid = true`<br>`stripe_past_due = false` | 正常有效 |
| `past_due` | `stripe_invoice_paid = true`<br>`stripe_past_due = true` | 逾期但仍保留服务 |
| `paused` | `stripe_invoice_paid = false` | 暂停，服务不可用 |
| `incomplete_expired` | `stripe_invoice_paid = false` | 支付超时未完成 |
| `unpaid` | `stripe_invoice_paid = false`<br>→ 调用 `team->subscriptionEnded()` | 未支付，**终止所有服务** |
| `canceled` | → 调用 `team->subscriptionEnded()` | 已取消，**终止所有服务** |

#### 3.2 套餐数量（服务器限制）映射

**[StripeProcessJob.php:262-271](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L262-L271)**

```php
if (str($lookup_key)->contains('dynamic')) {
    $quantity = min(
        (int) data_get($data, 'items.data.0.quantity', 2),
        UpdateSubscriptionQuantity::MAX_SERVER_LIMIT // 100
    );
    $team->update(['custom_server_limit' => $quantity]);
    ServerLimitCheckJob::dispatch($team);
}
```

**映射规则**：
- 仅当 `lookup_key` 包含 `dynamic` 时才同步数量（动态定价套餐）
- 数量被限制在 `[2, 100]` 区间（MIN_SERVER_LIMIT → MAX_SERVER_LIMIT）
- 更新 `custom_server_limit` 字段后，立即触发 `ServerLimitCheckJob` 检查超限

#### 3.3 退款/取消 → 本地状态映射

**[RefundSubscription.php:73-142](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/RefundSubscription.php#L73-L142)**

退款流程的本地状态变更顺序：
1. ✅ 先记录 `stripe_refunded_at = now()`（防重复退款的关键）
2. ❌ 标记 `stripe_invoice_paid = false`
3. ❌ 清除其他状态字段
4. 🔥 调用 `team->subscriptionEnded()` 终止服务

---

### 路径 4：团队套餐限制生效机制

#### 4.1 服务器限制计算

**[Team.php:154-165](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Team.php#L154-L165)**

```php
public function limits(): Attribute
{
    return Attribute::make(
        get: function () {
            if (config('constants.coolify.self_hosted') || $this->id === 0) {
                return 999999999999; // 自托管或根团队无限制
            }
            return $this->custom_server_limit ?? 2; // 默认 2 台
        }
    );
}
```

#### 4.2 ServerLimitCheckJob 超限处理

**[ServerLimitCheckJob.php:28-54](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/ServerLimitCheckJob.php#L28-L54)**

```php
$number_of_servers_to_disable = $servers_count - $this->team->limits;
if ($number_of_servers_to_disable > 0) {
    // 禁用最新创建的服务器（按 created_at 倒序）
    $servers->sortbyDesc('created_at')
        ->take($number_of_servers_to_disable)
        ->each(function ($server) {
            $server->forceDisableServer();
            $this->team->notify(new ForceDisabled($server));
        });
} else {
    // 限额恢复，重新启用被强制禁用的服务器
    $servers->each(function ($server) {
        if ($server->isForceDisabled()) {
            $server->forceEnableServer();
        }
    });
}
```

#### 4.3 subscriptionEnded 完全终止

**[Team.php:220-243](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Models/Team.php#L220-L243)**

```php
public function subscriptionEnded()
{
    // 1. 清除订阅状态
    $this->subscription->update([
        'stripe_subscription_id' => null,
        'stripe_invoice_paid' => false,
        // ... 其他字段重置
    ]);
    // 2. 禁用所有服务器
    foreach ($this->servers as $server) {
        $server->settings()->update([
            'is_usable' => false,
            'is_reachable' => false,
        ]);
        ServerReachabilityChanged::dispatch($server);
    }
}
```

---

## 三、重复回调事件处理机制

### 3.1 重复回调的产生原因

Stripe Webhook **至少投递一次**（at-least-once delivery），可能因：
- 网络超时导致 Stripe 未收到 200 响应
- 同一事件多次投递（通常间隔递增）
- 多个事件类型描述同一次状态变更

### 3.2 幂等性保障措施

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

#### 措施 4：订阅 ID 一致性校验

**[StripeProcessJob.php:279-283](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L279-L283)**

```php
if ($subscription->stripe_subscription_id === $subscriptionId) {
    $subscription->update([/* 状态更新 */]);
}
```

- 仅当事件中的订阅 ID 与本地记录一致时才更新
- 防止旧订阅事件影响新订阅

---

## 四、账单状态竞争条件处理

### 4.1 竞争场景分析

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

### 4.2 竞争防护机制

#### 机制 1：支付失败事件的延迟 + 二次校验

**[StripeProcessJob.php:166-187](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L166-L187)**

```php
// 1. 先调用 Stripe API 确认支付状态
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

#### 机制 2：SubscriptionInvoiceFailedJob 的三次校验

**[SubscriptionInvoiceFailedJob.php:26-64](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/SubscriptionInvoiceFailedJob.php#L26-L64)**

发送失败通知前的三道防线：
1. ✅ 检查订阅状态是否已变为 `active` 或 `trialing`
2. ✅ 检查最近 1 小时内是否有已支付发票
3. ✅ 任何一项通过则自动修正状态并终止流程

```php
if (in_array($stripeSubscription->status, ['active', 'trialing'])) {
    if (! $subscription->stripe_invoice_paid) {
        $subscription->update(['stripe_invoice_paid' => true]);
    }
    return; // 不发送失败通知
}
```

#### 机制 3：invoice.paid 中的状态覆盖优先级

**[StripeProcessJob.php:108-136](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L108-L136)**

`invoice.paid` 事件处理时，会**主动从 Stripe 拉取最新订阅状态**，而不是依赖事件中的数据：
```php
$stripeSubscription = $stripe->subscriptions->retrieve($subscription->stripe_subscription_id);
switch ($stripeSubscription->status) { /* 更新本地状态 */ }
```

这确保即使事件顺序错乱，也以 Stripe 端真实状态为准。

#### 机制 4：VerifyStripeSubscriptionStatusJob 兜底

**[StripeProcessJob.php:132-147](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Jobs/StripeProcessJob.php#L132-L147)**

遇到未知状态或 API 调用失败时，延迟 20 秒后调度校验 Job：
```php
VerifyStripeSubscriptionStatusJob::dispatch($subscription)
    ->delay(now()->addSeconds(20));
```

该 Job 会再次从 Stripe 拉取最新状态并同步，确保最终一致性。

#### 机制 5：UpdateSubscriptionQuantity 中的支付回滚

**[UpdateSubscriptionQuantity.php:154-178](file:///d:/fz/0601-1/solo-dogfeeding/code/100-coolify/app/Actions/Stripe/UpdateSubscriptionQuantity.php#L154-L178)**

用户主动变更套餐数量时，若分摊发票支付失败：
```php
if ($latestInvoice && $latestInvoice->status !== 'paid') {
    // 1. 回滚 Stripe 订阅数量
    $this->stripe->subscriptions->update($subscriptionId, [
        'items' => [['id' => $item->id, 'quantity' => $previousQuantity]],
        'proration_behavior' => 'none',
    ]);
    // 2. 作废未支付发票
    $this->stripe->invoices->voidInvoice($latestInvoice->id);
    // 3. 不更新本地 custom_server_limit
    return ['success' => false, 'error' => 'Payment failed.'];
}
```

这防止了 Stripe 数量已变更但支付失败，导致本地与远端不一致。

---

## 五、完整同步路径总结

### 5.1 订阅创建流程
```
checkout.session.completed
    ↓
Subscription::updateOrCreate(team_id)
    → stripe_invoice_paid = true
    → stripe_past_due = false
```

### 5.2 订阅状态变更流程
```
customer.subscription.updated
    ↓
根据 status 映射本地字段
    ├─ active → stripe_invoice_paid = true
    ├─ past_due → stripe_past_due = true
    ├─ unpaid/canceled → team->subscriptionEnded()
    └─ dynamic 套餐 → 更新 custom_server_limit → ServerLimitCheckJob
```

### 5.3 发票支付流程
```
invoice.paid
    ↓
主动调用 Stripe API 获取最新订阅状态
    ↓
根据 Stripe 返回的真实状态更新本地
    ↓
未知状态 → 延迟 20s → VerifyStripeSubscriptionStatusJob
```

### 5.4 支付失败流程
```
invoice.payment_failed
    ↓
Stripe API 二次确认支付意图状态
    ├─ 已成功/处理中 → 忽略
    └─ 确已失败 → 延迟 60s → SubscriptionInvoiceFailedJob
                                    ↓
                            再次校验订阅状态和最近发票
                                ├─ 已恢复 → 自动修正
                                └─ 确失败 → 发送邮件通知
```

### 5.5 退款取消流程
```
RefundSubscription::execute()
    ↓
1. 检查 stripe_refunded_at（防重复）
2. 调用 Stripe Refund API
3. 立即写入 stripe_refunded_at = now()
4. 调用 Stripe 取消订阅
5. 更新本地状态 stripe_invoice_paid = false
6. 调用 team->subscriptionEnded() 禁用所有服务器
```

### 5.6 用户主动变更套餐数量流程
```
UpdateSubscriptionQuantity::execute(team, quantity)
    ↓
1. 调用 Stripe 更新订阅数量 + 立即开票
2. 检查返回的 latest_invoice 状态
    ├─ paid → 更新 custom_server_limit → ServerLimitCheckJob
    └─ 未支付 → 回滚 Stripe 数量 + 作废发票 + 返回错误
```

---

## 六、关键设计亮点

1. **最终一致性优先**：多处设计兜底校验，不依赖单次事件顺序
2. **状态机单向流转**：失败事件不会覆盖已确认的成功状态
3. **先标记后操作**：退款先写 `stripe_refunded_at` 防重复
4. **延迟 + 重试**：失败事件延迟处理，给 Stripe 自动重试留窗口
5. **数量上下限保护**：服务器限制限制在 [2, 100] 区间，防止异常值
6. **按时间倒序禁用**：超限时先禁用最新服务器，保护用户核心业务
