# Django決済システム実装ガイド

## 目次
1. [決済システムの概要](#1-決済システムの概要)
2. [Stripe決済](#2-stripe決済)
3. [PayPal決済](#3-paypal決済)
4. [Square決済](#4-square決済)
5. [LINE Pay決済](#5-line-pay決済)
6. [PayPay決済](#6-paypay決済)
7. [楽天ペイ決済](#7-楽天ペイ決済)
8. [銀行振込・コンビニ決済](#8-銀行振込コンビニ決済)
9. [サブスクリプション決済](#9-サブスクリプション決済)
10. [セキュリティとベストプラクティス](#10-セキュリティとベストプラクティス)

---

## 1. 決済システムの概要

### 1.1 主要な決済サービス比較

| サービス | 手数料 | 対応通貨 | 対応国 | 実装難易度 | 特徴 |
|---------|--------|---------|--------|-----------|------|
| Stripe | 3.6% | 135+ | 46+ | ★★☆☆☆ | グローバル、API充実、サブスク対応 |
| PayPal | 3.6%+40円 | 25+ | 200+ | ★★★☆☆ | 世界最大、買い手保護 |
| Square | 3.25% | 4 | 8 | ★★☆☆☆ | 実店舗連携、シンプル |
| LINE Pay | 2.45%~ | JPY | 日本 | ★★★☆☆ | 国内普及率高、QRコード |
| PayPay | 3.24%~ | JPY | 日本 | ★★★★☆ | 国内シェア1位、QRコード |
| 楽天ペイ | 3.24% | JPY | 日本 | ★★★☆☆ | 楽天ポイント連携 |

### 1.2 決済フロー

```
┌──────────────────────────────────────────┐
│         決済フロー（基本）                 │
└──────────────────────────────────────────┘

[ユーザー]
    ↓
1. 商品選択・カート追加
    ↓
2. 注文確認画面
    ↓
3. 決済方法選択
    ↓
4. 決済処理
    ├─ クレジットカード
    ├─ 電子マネー
    ├─ 銀行振込
    └─ コンビニ決済
    ↓
5. 決済完了通知
    ↓
6. 注文確定
    ↓
[完了画面]
```

### 1.3 データモデル設計

```python
# payments/models.py

from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()

class Order(models.Model):
    """
    注文モデル
    """
    STATUS_CHOICES = [
        ('pending', '支払い待ち'),
        ('processing', '処理中'),
        ('completed', '完了'),
        ('cancelled', 'キャンセル'),
        ('refunded', '返金済み'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name='ユーザー')
    order_number = models.CharField(max_length=50, unique=True, verbose_name='注文番号')
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending', verbose_name='ステータス')
    total_amount = models.DecimalField(max_digits=10, decimal_places=2, verbose_name='合計金額')
    tax_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0, verbose_name='税額')
    shipping_fee = models.DecimalField(max_digits=10, decimal_places=2, default=0, verbose_name='送料')
    
    # 配送先情報
    shipping_name = models.CharField(max_length=100, verbose_name='配送先氏名')
    shipping_postal_code = models.CharField(max_length=10, verbose_name='郵便番号')
    shipping_address = models.TextField(verbose_name='住所')
    shipping_phone = models.CharField(max_length=20, verbose_name='電話番号')
    
    created_at = models.DateTimeField(auto_now_add=True, verbose_name='作成日時')
    updated_at = models.DateTimeField(auto_now=True, verbose_name='更新日時')
    
    class Meta:
        verbose_name = '注文'
        verbose_name_plural = '注文'
        ordering = ['-created_at']
    
    def __str__(self):
        return f'Order {self.order_number}'


class OrderItem(models.Model):
    """
    注文明細モデル
    """
    order = models.ForeignKey(Order, related_name='items', on_delete=models.CASCADE, verbose_name='注文')
    product_name = models.CharField(max_length=200, verbose_name='商品名')
    product_price = models.DecimalField(max_digits=10, decimal_places=2, verbose_name='単価')
    quantity = models.PositiveIntegerField(default=1, verbose_name='数量')
    subtotal = models.DecimalField(max_digits=10, decimal_places=2, verbose_name='小計')
    
    class Meta:
        verbose_name = '注文明細'
        verbose_name_plural = '注文明細'
    
    def __str__(self):
        return f'{self.product_name} x {self.quantity}'


class Payment(models.Model):
    """
    決済モデル
    """
    PAYMENT_METHOD_CHOICES = [
        ('credit_card', 'クレジットカード'),
        ('paypal', 'PayPal'),
        ('line_pay', 'LINE Pay'),
        ('paypay', 'PayPay'),
        ('bank_transfer', '銀行振込'),
        ('convenience_store', 'コンビニ決済'),
    ]
    
    STATUS_CHOICES = [
        ('pending', '支払い待ち'),
        ('authorized', '承認済み'),
        ('captured', '確定済み'),
        ('failed', '失敗'),
        ('refunded', '返金済み'),
        ('cancelled', 'キャンセル'),
    ]
    
    order = models.OneToOneField(Order, on_delete=models.CASCADE, verbose_name='注文')
    payment_method = models.CharField(max_length=50, choices=PAYMENT_METHOD_CHOICES, verbose_name='決済方法')
    payment_status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending', verbose_name='決済ステータス')
    
    # 決済サービス側の情報
    transaction_id = models.CharField(max_length=200, blank=True, verbose_name='トランザクションID')
    payment_intent_id = models.CharField(max_length=200, blank=True, verbose_name='決済インテントID')
    
    amount = models.DecimalField(max_digits=10, decimal_places=2, verbose_name='決済金額')
    currency = models.CharField(max_length=3, default='JPY', verbose_name='通貨')
    
    paid_at = models.DateTimeField(null=True, blank=True, verbose_name='支払い日時')
    created_at = models.DateTimeField(auto_now_add=True, verbose_name='作成日時')
    updated_at = models.DateTimeField(auto_now=True, verbose_name='更新日時')
    
    # 追加情報（JSON形式）
    metadata = models.JSONField(default=dict, blank=True, verbose_name='メタデータ')
    
    class Meta:
        verbose_name = '決済'
        verbose_name_plural = '決済'
    
    def __str__(self):
        return f'Payment for {self.order.order_number}'


class Refund(models.Model):
    """
    返金モデル
    """
    payment = models.ForeignKey(Payment, related_name='refunds', on_delete=models.CASCADE, verbose_name='決済')
    refund_id = models.CharField(max_length=200, verbose_name='返金ID')
    amount = models.DecimalField(max_digits=10, decimal_places=2, verbose_name='返金額')
    reason = models.TextField(verbose_name='返金理由')
    status = models.CharField(max_length=20, default='pending', verbose_name='ステータス')
    created_at = models.DateTimeField(auto_now_add=True, verbose_name='作成日時')
    
    class Meta:
        verbose_name = '返金'
        verbose_name_plural = '返金'
    
    def __str__(self):
        return f'Refund {self.refund_id}'
```

---

## 2. Stripe決済

### 2.1 Stripeの特徴

- グローバルで最も人気のある決済プラットフォーム
- 豊富なAPIとドキュメント
- サブスクリプション、分割払い、多通貨対応
- PCI DSS準拠でセキュア

### 2.2 Stripe決済のフロー

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │Stripe API│
│         │      │          │      │          │      │          │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. 商品購入     │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. チェックアウトリクエスト         │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. PaymentIntent作成│
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 4. client_secret返却│
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │ 5. チェックアウト画面表示           │
     │                 │    (Stripe.js埋込)                │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 6. カード情報入力│                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 7. カード情報送信（直接Stripeへ）   │
     │                 ├─────────────────────────────────>│
     │                 │                 │                 │
     │                 │                 │                 │ 8. カード検証
     │                 │                 │                 │
     │                 │ 9. PaymentMethod作成               │
     │                 │<─────────────────────────────────┤
     │                 │                 │                 │
     │                 │ 10. 決済確認リクエスト              │
     │                 │     (PaymentMethod ID)            │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 11. 決済実行    │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │                 │ 12. 決済処理
     │                 │                 │                 │     (3Dセキュア等)
     │                 │                 │                 │
     │                 │                 │ 13. 決済成功    │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 14. Webhook送信 │
     │                 │                 │ (payment_intent.succeeded)│
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 15. 注文ステータス更新│
     │                 │                 │     Payment記録 │
     │                 │                 │                 │
     │                 │ 16. 完了画面表示│                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 17. 購入完了    │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
```

### 2.3 インストールと設定

```bash
# インストール
pip install stripe
```

```python
# settings.py

# Stripe設定
STRIPE_PUBLIC_KEY = os.environ.get('STRIPE_PUBLIC_KEY')
STRIPE_SECRET_KEY = os.environ.get('STRIPE_SECRET_KEY')
STRIPE_WEBHOOK_SECRET = os.environ.get('STRIPE_WEBHOOK_SECRET')

# 通貨設定
STRIPE_CURRENCY = 'jpy'
```

### 2.3 基本的な実装

```python
# payments/stripe_service.py

import stripe
from django.conf import settings
from decimal import Decimal

stripe.api_key = settings.STRIPE_SECRET_KEY


class StripeService:
    """
    Stripe決済サービス
    """
    
    @staticmethod
    def create_payment_intent(order, metadata=None):
        """
        PaymentIntentを作成
        """
        try:
            # 金額を最小単位に変換（円→銭）
            amount = int(order.total_amount * 100)
            
            payment_intent = stripe.PaymentIntent.create(
                amount=amount,
                currency=settings.STRIPE_CURRENCY,
                metadata={
                    'order_id': order.id,
                    'order_number': order.order_number,
                    **(metadata or {})
                },
                description=f'Order {order.order_number}',
            )
            
            return payment_intent
        
        except stripe.error.StripeError as e:
            raise Exception(f'Stripe error: {str(e)}')
    
    @staticmethod
    def retrieve_payment_intent(payment_intent_id):
        """
        PaymentIntentを取得
        """
        try:
            return stripe.PaymentIntent.retrieve(payment_intent_id)
        except stripe.error.StripeError as e:
            raise Exception(f'Stripe error: {str(e)}')
    
    @staticmethod
    def confirm_payment(payment_intent_id):
        """
        決済を確定
        """
        try:
            return stripe.PaymentIntent.confirm(payment_intent_id)
        except stripe.error.StripeError as e:
            raise Exception(f'Stripe error: {str(e)}')
    
    @staticmethod
    def create_refund(payment_intent_id, amount=None, reason=None):
        """
        返金を作成
        """
        try:
            refund_params = {
                'payment_intent': payment_intent_id,
            }
            
            if amount:
                refund_params['amount'] = int(amount * 100)
            
            if reason:
                refund_params['reason'] = reason
            
            return stripe.Refund.create(**refund_params)
        
        except stripe.error.StripeError as e:
            raise Exception(f'Stripe error: {str(e)}')
    
    @staticmethod
    def create_customer(email, name=None, metadata=None):
        """
        顧客を作成
        """
        try:
            customer_params = {
                'email': email,
            }
            
            if name:
                customer_params['name'] = name
            
            if metadata:
                customer_params['metadata'] = metadata
            
            return stripe.Customer.create(**customer_params)
        
        except stripe.error.StripeError as e:
            raise Exception(f'Stripe error: {str(e)}')
```

```python
# payments/views.py

from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.views.decorators.csrf import csrf_exempt
from django.http import JsonResponse, HttpResponse
from django.conf import settings
import stripe
import json

from .models import Order, Payment
from .stripe_service import StripeService


@login_required
def create_checkout(request, order_id):
    """
    チェックアウト画面
    """
    order = get_object_or_404(Order, id=order_id, user=request.user)
    
    # PaymentIntentを作成
    try:
        payment_intent = StripeService.create_payment_intent(order)
        
        # Paymentレコードを作成
        payment, created = Payment.objects.get_or_create(
            order=order,
            defaults={
                'payment_method': 'credit_card',
                'amount': order.total_amount,
                'payment_intent_id': payment_intent.id,
            }
        )
        
        if not created:
            payment.payment_intent_id = payment_intent.id
            payment.save()
        
        context = {
            'order': order,
            'client_secret': payment_intent.client_secret,
            'stripe_public_key': settings.STRIPE_PUBLIC_KEY,
        }
        
        return render(request, 'payments/checkout.html', context)
    
    except Exception as e:
        return render(request, 'payments/error.html', {
            'error_message': str(e)
        })


@csrf_exempt
def stripe_webhook(request):
    """
    Stripeからのwebhookを処理
    """
    payload = request.body
    sig_header = request.META.get('HTTP_STRIPE_SIGNATURE')
    
    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
        )
    except ValueError:
        return HttpResponse(status=400)
    except stripe.error.SignatureVerificationError:
        return HttpResponse(status=400)
    
    # イベントタイプに応じた処理
    if event['type'] == 'payment_intent.succeeded':
        payment_intent = event['data']['object']
        handle_payment_success(payment_intent)
    
    elif event['type'] == 'payment_intent.payment_failed':
        payment_intent = event['data']['object']
        handle_payment_failed(payment_intent)
    
    return HttpResponse(status=200)


def handle_payment_success(payment_intent):
    """
    決済成功時の処理
    """
    try:
        payment = Payment.objects.get(payment_intent_id=payment_intent.id)
        payment.payment_status = 'captured'
        payment.transaction_id = payment_intent.id
        payment.save()
        
        # 注文ステータスを更新
        order = payment.order
        order.status = 'completed'
        order.save()
        
        # メール送信などの処理
        # send_order_confirmation_email(order)
    
    except Payment.DoesNotExist:
        pass


def handle_payment_failed(payment_intent):
    """
    決済失敗時の処理
    """
    try:
        payment = Payment.objects.get(payment_intent_id=payment_intent.id)
        payment.payment_status = 'failed'
        payment.save()
        
        # 注文ステータスを更新
        order = payment.order
        order.status = 'cancelled'
        order.save()
    
    except Payment.DoesNotExist:
        pass
```

```html
<!-- templates/payments/checkout.html -->

{% extends "base.html" %}
{% load static %}

{% block extra_head %}
<script src="https://js.stripe.com/v3/"></script>
{% endblock %}

{% block content %}
<div class="checkout-container">
    <h2>お支払い</h2>
    
    <div class="order-summary">
        <h3>注文内容</h3>
        <p>注文番号: {{ order.order_number }}</p>
        <p>合計金額: ¥{{ order.total_amount|floatformat:0 }}</p>
    </div>
    
    <form id="payment-form">
        <div id="card-element">
            <!-- Stripe Elements がここに挿入されます -->
        </div>
        
        <div id="card-errors" role="alert"></div>
        
        <button type="submit" id="submit-button">
            支払う
        </button>
    </form>
</div>

<script>
    const stripe = Stripe('{{ stripe_public_key }}');
    const elements = stripe.elements();
    
    // カード要素を作成
    const cardElement = elements.create('card', {
        style: {
            base: {
                fontSize: '16px',
                color: '#32325d',
            }
        }
    });
    cardElement.mount('#card-element');
    
    // エラー表示
    cardElement.on('change', function(event) {
        const displayError = document.getElementById('card-errors');
        if (event.error) {
            displayError.textContent = event.error.message;
        } else {
            displayError.textContent = '';
        }
    });
    
    // フォーム送信
    const form = document.getElementById('payment-form');
    form.addEventListener('submit', async function(event) {
        event.preventDefault();
        
        const submitButton = document.getElementById('submit-button');
        submitButton.disabled = true;
        
        const clientSecret = '{{ client_secret }}';
        
        const {error, paymentIntent} = await stripe.confirmCardPayment(
            clientSecret,
            {
                payment_method: {
                    card: cardElement,
                }
            }
        );
        
        if (error) {
            // エラー処理
            const displayError = document.getElementById('card-errors');
            displayError.textContent = error.message;
            submitButton.disabled = false;
        } else {
            if (paymentIntent.status === 'succeeded') {
                // 成功時
                window.location.href = '/payments/success/{{ order.id }}/';
            }
        }
    });
</script>
{% endblock %}
```

---

## 3. PayPal決済

### 3.1 PayPal決済のフロー

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │PayPal API│
│         │      │          │      │          │      │          │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. 商品購入     │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. PayPal決済選択│                │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. Payment作成  │
     │                 │                 │    (return_url, │
     │                 │                 │     cancel_url) │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 4. approval_url返却│
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │ 5. PayPalへリダイレクト            │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │                 │ 6. PayPalログイン画面へ移動        │
     │                 ├─────────────────────────────────>│
     │                 │                 │                 │
     │ 7. PayPalログイン│                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │ 8. 決済内容確認  │                 │                 │
     │    画面表示     │                 │                 │
     │<─────────────────────────────────────────────────┤
     │                 │                 │                 │
     │ 9. 支払い承認   │                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │                 │ 10. return_urlへリダイレクト       │
     │                 │     (payment_id, payer_id)       │
     │                 │<─────────────────────────────────┤
     │                 │                 │                 │
     │                 │ 11. 決済実行リクエスト              │
     │                 │     (payment_id, payer_id)       │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 12. 決済実行    │
     │                 │                 │     (execute)   │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 13. 決済完了    │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 14. 注文確定    │
     │                 │                 │     Payment記録 │
     │                 │                 │                 │
     │                 │ 15. 完了画面表示│                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 16. 購入完了    │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
     │                 │                 │                 │
     │ [キャンセル時]  │                 │                 │
     │                 │                 │                 │
     │ 17. キャンセルボタン押下                            │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │                 │ 18. cancel_urlへリダイレクト       │
     │                 │<─────────────────────────────────┤
     │                 │                 │                 │
     │                 │ 19. キャンセル処理│                │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │ 20. キャンセル画面│                │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
```

### 3.2 インストールと設定

```bash
# インストール
pip install paypalrestsdk
```

```python
# settings.py

# PayPal設定
PAYPAL_CLIENT_ID = os.environ.get('PAYPAL_CLIENT_ID')
PAYPAL_CLIENT_SECRET = os.environ.get('PAYPAL_CLIENT_SECRET')
PAYPAL_MODE = 'sandbox'  # 'sandbox' または 'live'
```

### 3.2 実装

```python
# payments/paypal_service.py

import paypalrestsdk
from django.conf import settings
from decimal import Decimal

# PayPal設定
paypalrestsdk.configure({
    'mode': settings.PAYPAL_MODE,
    'client_id': settings.PAYPAL_CLIENT_ID,
    'client_secret': settings.PAYPAL_CLIENT_SECRET
})


class PayPalService:
    """
    PayPal決済サービス
    """
    
    @staticmethod
    def create_payment(order, return_url, cancel_url):
        """
        PayPal決済を作成
        """
        payment = paypalrestsdk.Payment({
            'intent': 'sale',
            'payer': {
                'payment_method': 'paypal'
            },
            'redirect_urls': {
                'return_url': return_url,
                'cancel_url': cancel_url
            },
            'transactions': [{
                'item_list': {
                    'items': [
                        {
                            'name': item.product_name,
                            'sku': str(item.id),
                            'price': str(item.product_price),
                            'currency': 'JPY',
                            'quantity': item.quantity
                        }
                        for item in order.items.all()
                    ]
                },
                'amount': {
                    'total': str(order.total_amount),
                    'currency': 'JPY'
                },
                'description': f'Order {order.order_number}'
            }]
        })
        
        if payment.create():
            return payment
        else:
            raise Exception(payment.error)
    
    @staticmethod
    def execute_payment(payment_id, payer_id):
        """
        PayPal決済を実行
        """
        payment = paypalrestsdk.Payment.find(payment_id)
        
        if payment.execute({'payer_id': payer_id}):
            return payment
        else:
            raise Exception(payment.error)
    
    @staticmethod
    def get_payment(payment_id):
        """
        PayPal決済情報を取得
        """
        return paypalrestsdk.Payment.find(payment_id)
    
    @staticmethod
    def create_refund(sale_id, amount=None):
        """
        返金を作成
        """
        sale = paypalrestsdk.Sale.find(sale_id)
        
        refund_params = {}
        if amount:
            refund_params['amount'] = {
                'total': str(amount),
                'currency': 'JPY'
            }
        
        refund = sale.refund(refund_params)
        
        if refund.success():
            return refund
        else:
            raise Exception(refund.error)
```

```python
# payments/views.py

from .paypal_service import PayPalService


@login_required
def paypal_checkout(request, order_id):
    """
    PayPalチェックアウト
    """
    order = get_object_or_404(Order, id=order_id, user=request.user)
    
    return_url = request.build_absolute_uri(
        f'/payments/paypal/execute/{order.id}/'
    )
    cancel_url = request.build_absolute_uri(
        f'/payments/paypal/cancel/{order.id}/'
    )
    
    try:
        payment = PayPalService.create_payment(order, return_url, cancel_url)
        
        # Paymentレコードを作成
        Payment.objects.create(
            order=order,
            payment_method='paypal',
            amount=order.total_amount,
            payment_intent_id=payment.id,
        )
        
        # PayPalの承認URLにリダイレクト
        for link in payment.links:
            if link.rel == 'approval_url':
                return redirect(link.href)
        
        raise Exception('No approval URL found')
    
    except Exception as e:
        return render(request, 'payments/error.html', {
            'error_message': str(e)
        })


@login_required
def paypal_execute(request, order_id):
    """
    PayPal決済を実行
    """
    order = get_object_or_404(Order, id=order_id, user=request.user)
    payment_id = request.GET.get('paymentId')
    payer_id = request.GET.get('PayerID')
    
    try:
        payment = PayPalService.execute_payment(payment_id, payer_id)
        
        # Paymentレコードを更新
        payment_obj = Payment.objects.get(order=order)
        payment_obj.payment_status = 'captured'
        payment_obj.transaction_id = payment.transactions[0].related_resources[0].sale.id
        payment_obj.save()
        
        # 注文ステータスを更新
        order.status = 'completed'
        order.save()
        
        return redirect('payment_success', order_id=order.id)
    
    except Exception as e:
        return render(request, 'payments/error.html', {
            'error_message': str(e)
        })


@login_required
def paypal_cancel(request, order_id):
    """
    PayPal決済キャンセル
    """
    order = get_object_or_404(Order, id=order_id, user=request.user)
    
    # Paymentレコードを更新
    try:
        payment = Payment.objects.get(order=order)
        payment.payment_status = 'cancelled'
        payment.save()
    except Payment.DoesNotExist:
        pass
    
    return render(request, 'payments/cancelled.html', {
        'order': order
    })
```

---

## 4. Square決済

### 4.1 Square決済のフロー

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │Square API│
│         │      │          │      │          │      │          │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. 商品購入     │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. チェックアウトリクエスト         │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │ 3. チェックアウト画面表示           │
     │                 │    (Square Web SDK埋込)           │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 4. カード情報入力│                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 5. カード情報をトークン化           │
     │                 │    (直接Squareへ)                 │
     │                 ├─────────────────────────────────>│
     │                 │                 │                 │
     │                 │                 │                 │ 6. カード検証
     │                 │                 │                 │
     │                 │ 7. source_id (nonce)返却          │
     │                 │<─────────────────────────────────┤
     │                 │                 │                 │
     │                 │ 8. 決済リクエスト│                 │
     │                 │    (source_id)  │                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 9. 決済実行     │
     │                 │                 │    (create_payment)│
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │                 │ 10. 決済処理
     │                 │                 │                 │
     │                 │                 │ 11. 決済結果    │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 12. 注文確定    │
     │                 │                 │     Payment記録 │
     │                 │                 │                 │
     │                 │ 13. 完了画面表示│                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 14. 購入完了    │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
```

### 4.2 インストールと設定

```bash
# インストール
pip install squareup
```

```python
# settings.py

# Square設定
SQUARE_ACCESS_TOKEN = os.environ.get('SQUARE_ACCESS_TOKEN')
SQUARE_LOCATION_ID = os.environ.get('SQUARE_LOCATION_ID')
SQUARE_ENVIRONMENT = 'sandbox'  # 'sandbox' または 'production'
```

### 4.2 実装

```python
# payments/square_service.py

from square.client import Client
from django.conf import settings
import uuid


class SquareService:
    """
    Square決済サービス
    """
    
    def __init__(self):
        self.client = Client(
            access_token=settings.SQUARE_ACCESS_TOKEN,
            environment=settings.SQUARE_ENVIRONMENT
        )
    
    def create_payment(self, order, source_id):
        """
        Square決済を作成
        """
        amount = int(order.total_amount * 100)  # 円→銭
        
        result = self.client.payments.create_payment(
            body={
                'source_id': source_id,
                'idempotency_key': str(uuid.uuid4()),
                'amount_money': {
                    'amount': amount,
                    'currency': 'JPY'
                },
                'location_id': settings.SQUARE_LOCATION_ID,
                'reference_id': order.order_number,
                'note': f'Order {order.order_number}',
            }
        )
        
        if result.is_success():
            return result.body['payment']
        else:
            raise Exception(result.errors)
    
    def get_payment(self, payment_id):
        """
        Square決済情報を取得
        """
        result = self.client.payments.get_payment(payment_id)
        
        if result.is_success():
            return result.body['payment']
        else:
            raise Exception(result.errors)
    
    def create_refund(self, payment_id, amount=None, reason=None):
        """
        返金を作成
        """
        refund_params = {
            'payment_id': payment_id,
            'idempotency_key': str(uuid.uuid4()),
        }
        
        if amount:
            refund_params['amount_money'] = {
                'amount': int(amount * 100),
                'currency': 'JPY'
            }
        
        if reason:
            refund_params['reason'] = reason
        
        result = self.client.refunds.refund_payment(body=refund_params)
        
        if result.is_success():
            return result.body['refund']
        else:
            raise Exception(result.errors)
```

---

## 5. LINE Pay決済

### 5.1 LINE Pay決済のフロー

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │LINE Pay  │
│         │      │          │      │          │      │API       │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. 商品購入     │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. LINE Pay決済選択│                │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. 決済リクエスト│
     │                 │                 │    (request)    │
     │                 │                 │    署名生成     │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 4. transaction_id│
     │                 │                 │    payment_url  │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │ 5. LINE Payへリダイレクト          │
     │                 │    (payment_url)                  │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │                 │ 6. LINE Pay画面へ移動              │
     │                 ├─────────────────────────────────>│
     │                 │                 │                 │
     │ 7. LINEログイン │                 │                 │
     │    (必要時)     │                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │ 8. 決済内容確認  │                 │                 │
     │    画面表示     │                 │                 │
     │<─────────────────────────────────────────────────┤
     │                 │                 │                 │
     │ 9. 支払い方法選択│                 │                 │
     │    (残高/カード) │                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │ 10. 決済確認    │                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │                 │ 11. confirmUrlへリダイレクト       │
     │                 │     (transactionId)              │
     │                 │<─────────────────────────────────┤
     │                 │                 │                 │
     │                 │ 12. 決済確認API呼出│               │
     │                 │     (confirm)   │                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 13. 決済確定    │
     │                 │                 │     (confirm)   │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 14. 決済完了    │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 15. 注文確定    │
     │                 │                 │     Payment記録 │
     │                 │                 │                 │
     │                 │ 16. 完了画面表示│                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 17. 購入完了    │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
```

### 5.2 LINE Pay APIの実装

```python
# payments/linepay_service.py

import requests
import hmac
import hashlib
import base64
import json
from django.conf import settings


class LinePayService:
    """
    LINE Pay決済サービス
    """
    
    API_URL = 'https://api-pay.line.me'  # 本番環境
    # API_URL = 'https://sandbox-api-pay.line.me'  # サンドボックス
    
    def __init__(self):
        self.channel_id = settings.LINEPAY_CHANNEL_ID
        self.channel_secret = settings.LINEPAY_CHANNEL_SECRET
    
    def _generate_signature(self, uri, body):
        """
        署名を生成
        """
        message = self.channel_secret + uri + json.dumps(body)
        signature = base64.b64encode(
            hmac.new(
                self.channel_secret.encode('utf-8'),
                message.encode('utf-8'),
                hashlib.sha256
            ).digest()
        ).decode('utf-8')
        return signature
    
    def request_payment(self, order, confirm_url, cancel_url):
        """
        決済リクエスト
        """
        uri = '/v3/payments/request'
        
        body = {
            'amount': int(order.total_amount),
            'currency': 'JPY',
            'orderId': order.order_number,
            'packages': [
                {
                    'id': 'package_1',
                    'amount': int(order.total_amount),
                    'products': [
                        {
                            'name': item.product_name,
                            'quantity': item.quantity,
                            'price': int(item.product_price)
                        }
                        for item in order.items.all()
                    ]
                }
            ],
            'redirectUrls': {
                'confirmUrl': confirm_url,
                'cancelUrl': cancel_url
            }
        }
        
        headers = {
            'Content-Type': 'application/json',
            'X-LINE-ChannelId': self.channel_id,
            'X-LINE-Authorization': self._generate_signature(uri, body)
        }
        
        response = requests.post(
            f'{self.API_URL}{uri}',
            headers=headers,
            json=body
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f'LINE Pay error: {response.text}')
    
    def confirm_payment(self, transaction_id, amount):
        """
        決済確認
        """
        uri = f'/v3/payments/{transaction_id}/confirm'
        
        body = {
            'amount': amount,
            'currency': 'JPY'
        }
        
        headers = {
            'Content-Type': 'application/json',
            'X-LINE-ChannelId': self.channel_id,
            'X-LINE-Authorization': self._generate_signature(uri, body)
        }
        
        response = requests.post(
            f'{self.API_URL}{uri}',
            headers=headers,
            json=body
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f'LINE Pay error: {response.text}')
    
    def refund_payment(self, transaction_id, refund_amount=None):
        """
        返金
        """
        uri = f'/v3/payments/{transaction_id}/refund'
        
        body = {}
        if refund_amount:
            body['refundAmount'] = refund_amount
        
        headers = {
            'Content-Type': 'application/json',
            'X-LINE-ChannelId': self.channel_id,
            'X-LINE-Authorization': self._generate_signature(uri, body)
        }
        
        response = requests.post(
            f'{self.API_URL}{uri}',
            headers=headers,
            json=body
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f'LINE Pay error: {response.text}')
```

---

## 6. PayPay決済

### 6.1 PayPay決済のフロー

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │PayPay API│
│         │      │          │      │          │      │          │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. 商品購入     │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. PayPay決済選択│                │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. QRコード作成 │
     │                 │                 │    リクエスト   │
     │                 │                 │    (create)     │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 4. QRコードURL  │
     │                 │                 │    merchant_    │
     │                 │                 │    payment_id   │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │ 5. QRコード画面表示│                │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 6. QRコードスキャン│                 │                 │
     │    (PayPayアプリ起動)                                │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │ 7. 決済内容確認  │                 │                 │
     │    画面表示     │                 │                 │
     │<─────────────────────────────────────────────────┤
     │                 │                 │                 │
     │ 8. 支払い確認   │                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │                 │                 │                 │ 9. 決済処理
     │                 │                 │                 │
     │                 │                 │ 10. Webhook送信 │
     │                 │                 │ (payment.authorized)│
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 11. ステータス確認│
     │                 │                 │     (polling)   │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 12. 決済完了通知│
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 13. 決済確定    │
     │                 │                 │     (capture)   │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 14. キャプチャ完了│
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 15. 注文確定    │
     │                 │                 │     Payment記録 │
     │                 │                 │                 │
     │                 │ 16. 完了画面表示│                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 17. 購入完了    │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
```

### 6.2 PayPay for Developers実装

```python
# payments/paypay_service.py

import requests
import time
import hashlib
import hmac
import uuid
from django.conf import settings


class PayPayService:
    """
    PayPay決済サービス
    """
    
    API_URL = 'https://api.paypay.ne.jp'  # 本番環境
    # API_URL = 'https://stg-api.paypay.ne.jp'  # ステージング
    
    def __init__(self):
        self.api_key = settings.PAYPAY_API_KEY
        self.api_secret = settings.PAYPAY_API_SECRET
        self.merchant_id = settings.PAYPAY_MERCHANT_ID
    
    def _generate_signature(self, method, resource_url, body):
        """
        署名を生成
        """
        timestamp = str(int(time.time()))
        nonce = str(uuid.uuid4())
        
        content_type = 'application/json'
        payload = f'{resource_url}\n{method}\n{nonce}\n{timestamp}\n{content_type}\n{body}'
        
        signature = base64.b64encode(
            hmac.new(
                self.api_secret.encode('utf-8'),
                payload.encode('utf-8'),
                hashlib.sha256
            ).digest()
        ).decode('utf-8')
        
        return {
            'X-ASSUME-MERCHANT': self.merchant_id,
            'Authorization': f'hmac OPA-Auth:{signature}',
            'X-TIMESTAMP': timestamp,
            'X-NONCE': nonce,
            'Content-Type': content_type
        }
    
    def create_payment(self, order, callback_url):
        """
        決済を作成
        """
        resource_url = '/v2/codes'
        method = 'POST'
        
        body = {
            'merchantPaymentId': order.order_number,
            'amount': {
                'amount': int(order.total_amount),
                'currency': 'JPY'
            },
            'orderDescription': f'Order {order.order_number}',
            'orderItems': [
                {
                    'name': item.product_name,
                    'quantity': item.quantity,
                    'unitPrice': {
                        'amount': int(item.product_price),
                        'currency': 'JPY'
                    }
                }
                for item in order.items.all()
            ],
            'redirectUrl': callback_url,
            'redirectType': 'WEB_LINK'
        }
        
        headers = self._generate_signature(method, resource_url, json.dumps(body))
        
        response = requests.post(
            f'{self.API_URL}{resource_url}',
            headers=headers,
            json=body
        )
        
        if response.status_code == 201:
            return response.json()
        else:
            raise Exception(f'PayPay error: {response.text}')
    
    def get_payment_details(self, merchant_payment_id):
        """
        決済詳細を取得
        """
        resource_url = f'/v2/payments/{merchant_payment_id}'
        method = 'GET'
        
        headers = self._generate_signature(method, resource_url, '')
        
        response = requests.get(
            f'{self.API_URL}{resource_url}',
            headers=headers
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f'PayPay error: {response.text}')
    
    def refund_payment(self, merchant_payment_id, merchant_refund_id, amount=None):
        """
        返金
        """
        resource_url = '/v2/refunds'
        method = 'POST'
        
        body = {
            'merchantPaymentId': merchant_payment_id,
            'merchantRefundId': merchant_refund_id,
            'reason': 'Customer request'
        }
        
        if amount:
            body['amount'] = {
                'amount': int(amount),
                'currency': 'JPY'
            }
        
        headers = self._generate_signature(method, resource_url, json.dumps(body))
        
        response = requests.post(
            f'{self.API_URL}{resource_url}',
            headers=headers,
            json=body
        )
        
        if response.status_code == 201:
            return response.json()
        else:
            raise Exception(f'PayPay error: {response.text}')
```

---

## 7. 楽天ペイ決済

### 7.1 楽天ペイ決済のフロー

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │楽天ペイ   │
│         │      │          │      │          │      │API       │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. 商品購入     │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. 楽天ペイ決済選択│                │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. 決済リクエスト│
     │                 │                 │    チェックサム生成│
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 4. checkout_url │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │ 5. 楽天ペイへリダイレクト          │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │                 │ 6. 楽天ペイ画面へ移動              │
     │                 ├─────────────────────────────────>│
     │                 │                 │                 │
     │ 7. 楽天IDログイン│                 │                 │
     │    (必要時)     │                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │ 8. 決済内容確認  │                 │                 │
     │    楽天ポイント利用選択                              │
     │<─────────────────────────────────────────────────┤
     │                 │                 │                 │
     │ 9. 支払い確認   │                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │                 │                 │                 │ 10. 決済処理
     │                 │                 │                 │
     │                 │ 11. success_urlへリダイレクト      │
     │                 │     (order_number, result)       │
     │                 │<─────────────────────────────────┤
     │                 │                 │                 │
     │                 │ 12. 決済結果確認│                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 13. ステータス取得│
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 14. 決済詳細    │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 15. 注文確定    │
     │                 │                 │     Payment記録 │
     │                 │                 │     楽天ポイント付与│
     │                 │                 │                 │
     │                 │ 16. 完了画面表示│                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 17. 購入完了    │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
```

### 7.2 楽天ペイ実装

```python
# payments/rakutenpay_service.py

import requests
import hashlib
from django.conf import settings


class RakutenPayService:
    """
    楽天ペイ決済サービス
    """
    
    API_URL = 'https://api.checkout.rakuten.co.jp'
    
    def __init__(self):
        self.service_key = settings.RAKUTENPAY_SERVICE_KEY
        self.service_secret = settings.RAKUTENPAY_SERVICE_SECRET
    
    def _generate_checksum(self, params):
        """
        チェックサムを生成
        """
        sorted_params = sorted(params.items())
        param_str = '&'.join([f'{k}={v}' for k, v in sorted_params])
        checksum_str = param_str + self.service_secret
        
        return hashlib.md5(checksum_str.encode('utf-8')).hexdigest()
    
    def create_payment(self, order, success_url, error_url):
        """
        決済を作成
        """
        params = {
            'service_key': self.service_key,
            'order_number': order.order_number,
            'amount': int(order.total_amount),
            'tax': int(order.tax_amount),
            'success_url': success_url,
            'error_url': error_url,
        }
        
        # アイテム情報を追加
        for idx, item in enumerate(order.items.all(), 1):
            params[f'item_name_{idx}'] = item.product_name
            params[f'item_price_{idx}'] = int(item.product_price)
            params[f'item_quantity_{idx}'] = item.quantity
        
        # チェックサムを追加
        params['checksum'] = self._generate_checksum(params)
        
        # 決済URLを生成
        payment_url = f'{self.API_URL}/payment?'
        payment_url += '&'.join([f'{k}={v}' for k, v in params.items()])
        
        return payment_url
    
    def verify_payment(self, order_number, result_key):
        """
        決済結果を検証
        """
        params = {
            'service_key': self.service_key,
            'order_number': order_number,
        }
        
        checksum = self._generate_checksum(params)
        
        response = requests.get(
            f'{self.API_URL}/result',
            params={
                **params,
                'checksum': checksum
            }
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f'Rakuten Pay error: {response.text}')
```

---

## 8. 銀行振込・コンビニ決済

### 8.1 銀行振込・コンビニ決済のフロー

```
■ コンビニ決済フロー:

┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │決済代行   │
│         │      │          │      │          │      │サービス   │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. 商品購入     │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. コンビニ決済選択│                │
     │                 │    (セブン/ローソン/ファミマ)        │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. 決済番号発行 │
     │                 │                 │    リクエスト   │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 4. 払込票番号   │
     │                 │                 │    受付番号     │
     │                 │                 │    確認番号     │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │ 5. 決済番号表示 │                 │
     │                 │    期限表示     │                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 6. 決済番号メモ  │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
     │ 7. コンビニ来店  │                 │                 │
     │    端末操作     │                 │                 │
     │    (Loppi/Famiポート等)                              │
     │                 │                 │                 │
     │ 8. 決済番号入力  │                 │                 │
     │    レジで支払い  │                 │                 │
     │                 │                 │                 │
     │                 │                 │ 9. 入金通知     │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 10. 注文確定    │
     │                 │                 │     Payment記録 │
     │                 │                 │                 │
     │                 │ 11. 入金完了メール送信              │
     │<────────────────────────────────┤                 │
     │                 │                 │                 │

■ 銀行振込フロー:

┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │決済代行   │
│         │      │          │      │          │      │サービス   │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. 商品購入     │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. 銀行振込選択  │                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. 仮想口座発行 │
     │                 │                 │    リクエスト   │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 4. 振込先情報   │
     │                 │                 │    (銀行名/支店名│
     │                 │                 │     口座番号)   │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │ 5. 振込先表示   │                 │
     │                 │    期限表示     │                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 6. 振込先メモ   │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
     │ 7. ATM/ネットバンクで振込           │                 │
     │                 │                 │                 │
     │                 │                 │ 8. 入金通知     │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 9. 注文確定     │
     │                 │                 │    Payment記録  │
     │                 │                 │                 │
     │                 │ 10. 入金完了メール送信              │
     │<────────────────────────────────┤                 │
     │                 │                 │                 │
```

### 8.2 GMOペイメントゲートウェイ実装

```python
# payments/gmo_service.py

import requests
import hashlib
from datetime import datetime
from django.conf import settings


class GMOPaymentService:
    """
    GMOペイメントゲートウェイサービス
    """
    
    API_URL = 'https://p01.mul-pay.jp'
    
    def __init__(self):
        self.shop_id = settings.GMO_SHOP_ID
        self.shop_pass = settings.GMO_SHOP_PASS
    
    def create_convenience_store_payment(self, order):
        """
        コンビニ決済を作成
        """
        params = {
            'ShopID': self.shop_id,
            'ShopPass': self.shop_pass,
            'OrderID': order.order_number,
            'Amount': int(order.total_amount),
            'Tax': int(order.tax_amount),
            'CustomerName': order.shipping_name,
            'CustomerKana': '',  # カナ名
            'TelNo': order.shipping_phone,
            'Convenience': 'LAWSON',  # LAWSON, FAMILYMART, SEVENELEVEN等
            'ReceiptsDisp1': f'Order {order.order_number}',
        }
        
        response = requests.post(
            f'{self.API_URL}/payment/ExecTran.idPass',
            data=params
        )
        
        if response.status_code == 200:
            return self._parse_response(response.text)
        else:
            raise Exception(f'GMO error: {response.text}')
    
    def create_bank_transfer_payment(self, order):
        """
        銀行振込を作成
        """
        params = {
            'ShopID': self.shop_id,
            'ShopPass': self.shop_pass,
            'OrderID': order.order_number,
            'Amount': int(order.total_amount),
            'Tax': int(order.tax_amount),
        }
        
        response = requests.post(
            f'{self.API_URL}/payment/VirtualAccount.idPass',
            data=params
        )
        
        if response.status_code == 200:
            return self._parse_response(response.text)
        else:
            raise Exception(f'GMO error: {response.text}')
    
    def _parse_response(self, response_text):
        """
        レスポンスをパース
        """
        params = {}
        for param in response_text.split('&'):
            key, value = param.split('=')
            params[key] = value
        return params
```

---

## 9. サブスクリプション決済

### 9.1 サブスクリプション決済のフロー

```
┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │Stripe API│
│         │      │          │      │          │      │          │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ [初回登録]       │                 │                 │
     │                 │                 │                 │
     │ 1. プラン選択   │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 2. 登録リクエスト│                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. Customer作成 │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 4. Customer ID  │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │ 5. カード情報入力画面              │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 6. カード情報入力│                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 7. PaymentMethod作成（直接Stripe）  │
     │                 ├─────────────────────────────────>│
     │                 │                 │                 │
     │                 │ 8. PaymentMethod ID               │
     │                 │<─────────────────────────────────┤
     │                 │                 │                 │
     │                 │ 9. Subscription作成リクエスト       │
     │                 │    (Customer, PaymentMethod, Price)│
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 10. Subscription作成│
     │                 │                 │     PaymentMethodアタッチ│
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 11. Subscription ID│
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 12. DB記録      │
     │                 │                 │     (Subscription)│
     │                 │                 │                 │
     │                 │ 13. 登録完了画面│                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 14. 登録完了    │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
     │                 │                 │                 │
     │ [毎月の課金]     │                 │                 │
     │                 │                 │                 │
     │                 │                 │ 15. Webhook     │
     │                 │                 │ (invoice.created)│
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │                 │ 16. 自動課金
     │                 │                 │                 │
     │                 │                 │ 17. Webhook     │
     │                 │                 │ (invoice.paid)  │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 18. 請求記録    │
     │                 │                 │     Payment作成 │
     │                 │                 │                 │
     │                 │ 19. 課金完了メール送信              │
     │<────────────────────────────────┤                 │
     │                 │                 │                 │
     │                 │                 │                 │
     │ [解約]          │                 │                 │
     │                 │                 │                 │
     │ 20. 解約リクエスト│                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 21. 解約処理    │                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 22. Subscription │
     │                 │                 │     キャンセル   │
     │                 │                 ├────────────────>│
     │                 │                 │                 │
     │                 │                 │ 23. 解約完了    │
     │                 │                 │<────────────────┤
     │                 │                 │                 │
     │                 │                 │ 24. DB更新      │
     │                 │                 │     (status=cancelled)│
     │                 │                 │                 │
     │                 │ 25. 解約完了画面│                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 26. 解約完了    │                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
```

### 9.2 Stripeサブスクリプション

```python
# payments/subscription_service.py

import stripe
from django.conf import settings

stripe.api_key = settings.STRIPE_SECRET_KEY


class SubscriptionService:
    """
    サブスクリプション管理サービス
    """
    
    @staticmethod
    def create_product(name, description=None):
        """
        商品を作成
        """
        return stripe.Product.create(
            name=name,
            description=description
        )
    
    @staticmethod
    def create_price(product_id, amount, interval='month'):
        """
        価格プランを作成
        
        interval: 'day', 'week', 'month', 'year'
        """
        return stripe.Price.create(
            product=product_id,
            unit_amount=int(amount * 100),
            currency='jpy',
            recurring={
                'interval': interval,
            }
        )
    
    @staticmethod
    def create_subscription(customer_id, price_id, trial_period_days=None):
        """
        サブスクリプションを作成
        """
        subscription_params = {
            'customer': customer_id,
            'items': [{'price': price_id}],
        }
        
        if trial_period_days:
            subscription_params['trial_period_days'] = trial_period_days
        
        return stripe.Subscription.create(**subscription_params)
    
    @staticmethod
    def cancel_subscription(subscription_id, at_period_end=True):
        """
        サブスクリプションをキャンセル
        """
        return stripe.Subscription.modify(
            subscription_id,
            cancel_at_period_end=at_period_end
        )
    
    @staticmethod
    def update_subscription(subscription_id, new_price_id):
        """
        サブスクリプションを更新
        """
        subscription = stripe.Subscription.retrieve(subscription_id)
        
        return stripe.Subscription.modify(
            subscription_id,
            items=[{
                'id': subscription['items']['data'][0].id,
                'price': new_price_id,
            }]
        )
```

```python
# サブスクリプションモデル

class Subscription(models.Model):
    """
    サブスクリプションモデル
    """
    STATUS_CHOICES = [
        ('active', 'アクティブ'),
        ('past_due', '延滞'),
        ('canceled', 'キャンセル'),
        ('unpaid', '未払い'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    stripe_subscription_id = models.CharField(max_length=200)
    stripe_customer_id = models.CharField(max_length=200)
    plan_name = models.CharField(max_length=100)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES)
    current_period_start = models.DateTimeField()
    current_period_end = models.DateTimeField()
    cancel_at_period_end = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        verbose_name = 'サブスクリプション'
        verbose_name_plural = 'サブスクリプション'
```

---

## 10. セキュリティとベストプラクティス

### 10.1 セキュリティチェックリスト

```markdown
□ PCI DSS準拠
  □ クレジットカード情報をサーバーに保存しない
  □ 決済フォームはHTTPSで提供
  □ トークン化された情報のみを扱う

□ データ保護
  □ 決済情報は暗号化して保存
  □ APIキー・シークレットは環境変数で管理
  □ ログに機密情報を記録しない

□ 不正利用対策
  □ 3Dセキュア対応
  □ 異常な決済パターンの検知
  □ レート制限の実装
  □ IPアドレスの記録

□ 運用
  □ Webhookの署名検証
  □ エラーハンドリング
  □ トランザクションログの記録
  □ 定期的なセキュリティ監査
```

### 10.2 共通ユーティリティ

```python
# payments/utils.py

import secrets
import hashlib
from decimal import Decimal


def generate_order_number():
    """
    注文番号を生成
    """
    timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
    random_part = secrets.token_hex(4).upper()
    return f'ORD-{timestamp}-{random_part}'


def calculate_tax(amount, tax_rate=0.10):
    """
    消費税を計算
    """
    return Decimal(amount) * Decimal(tax_rate)


def format_amount(amount):
    """
    金額をフォーマット
    """
    return f'¥{int(amount):,}'


def validate_payment_amount(order, payment_amount):
    """
    決済金額を検証
    """
    if abs(order.total_amount - payment_amount) > Decimal('0.01'):
        raise ValueError('Payment amount mismatch')
    
    return True


def log_payment_event(event_type, order, data):
    """
    決済イベントをログに記録
    """
    import logging
    logger = logging.getLogger('payments')
    
    logger.info(
        f'Payment Event: {event_type}',
        extra={
            'order_id': order.id,
            'order_number': order.order_number,
            'event_data': data
        }
    )
```

### 10.3 エラーハンドリング

```python
# payments/exceptions.py

class PaymentError(Exception):
    """
    決済エラーの基底クラス
    """
    pass


class PaymentAuthorizationError(PaymentError):
    """
    決済承認エラー
    """
    pass


class PaymentCaptureError(PaymentError):
    """
    決済確定エラー
    """
    pass


class RefundError(PaymentError):
    """
    返金エラー
    """
    pass


class InvalidAmountError(PaymentError):
    """
    金額エラー
    """
    pass
```

### 10.4 テスト

```python
# payments/tests.py

from django.test import TestCase
from decimal import Decimal
from .models import Order, Payment
from .stripe_service import StripeService


class PaymentTestCase(TestCase):
    """
    決済機能のテスト
    """
    
    def setUp(self):
        """
        テストデータの準備
        """
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        
        self.order = Order.objects.create(
            user=self.user,
            order_number='TEST-001',
            total_amount=Decimal('1000.00'),
            status='pending'
        )
    
    def test_create_payment_intent(self):
        """
        PaymentIntent作成のテスト
        """
        payment_intent = StripeService.create_payment_intent(self.order)
        
        self.assertIsNotNone(payment_intent.id)
        self.assertEqual(payment_intent.amount, 100000)  # 1000円 = 100000銭
        self.assertEqual(payment_intent.currency, 'jpy')
    
    def test_payment_success(self):
        """
        決済成功時のテスト
        """
        payment = Payment.objects.create(
            order=self.order,
            payment_method='credit_card',
            amount=self.order.total_amount,
            payment_status='pending'
        )
        
        # 決済成功処理
        payment.payment_status = 'captured'
        payment.save()
        
        self.order.status = 'completed'
        self.order.save()
        
        self.assertEqual(payment.payment_status, 'captured')
        self.assertEqual(self.order.status, 'completed')
```

---

## まとめ

Djangoでの決済実装は、様々な決済サービスのAPIを活用することで実現できます。

**重要ポイント:**
1. **セキュリティ最優先** - PCI DSS準拠、機密情報の適切な管理
2. **適切なサービス選択** - ターゲット市場、手数料、実装難易度を考慮
3. **エラーハンドリング** - 決済失敗時の適切な処理とユーザー通知
4. **トランザクション管理** - 決済状態の正確な追跡とログ記録
5. **テスト環境活用** - 本番前に十分なテストを実施
6. **Webhook実装** - 非同期イベントの適切な処理
7. **返金対応** - 返金フローの実装と管理

各決済サービスの公式ドキュメントを参照し、最新のAPIバージョンとセキュリティ要件に準拠した実装を心がけてください。
