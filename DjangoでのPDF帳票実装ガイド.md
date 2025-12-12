# DjangoでのPDF帳票実装ガイド

## 目次
1. [PDF帳票の概要](#1-pdf帳票の概要)
2. [ReportLab実装](#2-reportlab実装)
3. [WeasyPrint実装](#3-weasyprint実装)
4. [xhtml2pdf実装](#4-xhtml2pdf実装)
5. [帳票テンプレート管理](#5-帳票テンプレート管理)
6. [日本語フォント対応](#6-日本語フォント対応)
7. [高度な帳票機能](#7-高度な帳票機能)
8. [パフォーマンス最適化](#8-パフォーマンス最適化)

---

## 1. PDF帳票の概要

### 1.1 主要ライブラリ比較

| ライブラリ | 難易度 | 日本語対応 | HTML対応 | 特徴 |
|-----------|--------|-----------|---------|------|
| ReportLab | ★★★☆☆ | ○ | × | 低レベルAPI、細かい制御可能 |
| WeasyPrint | ★★☆☆☆ | ○ | ○ | HTML/CSS to PDF、美しい出力 |
| xhtml2pdf | ★☆☆☆☆ | ○ | ○ | Djangoテンプレート活用 |
| PyPDF2 | ★★★☆☆ | ○ | × | PDF編集・結合 |
| PDFKit | ★☆☆☆☆ | ○ | ○ | wkhtmltopdfラッパー |

### 1.2 用途別推奨ライブラリ

```
用途別推奨:

1. 請求書・納品書
   └─ WeasyPrint (HTML/CSS活用)

2. 帳票レポート（複雑なレイアウト）
   └─ ReportLab (細かい制御)

3. 既存HTMLページのPDF化
   └─ WeasyPrint または PDFKit

4. PDF結合・編集
   └─ PyPDF2

5. シンプルな帳票
   └─ xhtml2pdf (Djangoテンプレート)
```

---

## 2. ReportLab実装

### 2.1 インストール

```bash
pip install reportlab
pip install pillow  # 画像処理用
```

### 2.2 基本実装

```python
# reports/pdf_generator.py

from reportlab.lib.pagesizes import A4, letter
from reportlab.lib.units import mm
from reportlab.pdfgen import canvas
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.lib.colors import HexColor
from django.http import HttpResponse
from datetime import datetime
import os

class InvoicePDF:
    """
    請求書PDF生成クラス
    """
    
    def __init__(self):
        # 日本語フォント登録
        font_path = os.path.join(
            os.path.dirname(__file__),
            'fonts',
            'NotoSansJP-Regular.ttf'
        )
        pdfmetrics.registerFont(TTFont('NotoSansJP', font_path))
        
    def generate_invoice(self, invoice_data):
        """
        請求書PDF生成
        """
        response = HttpResponse(content_type='application/pdf')
        response['Content-Disposition'] = f'attachment; filename="invoice_{invoice_data["number"]}.pdf"'
        
        # PDFキャンバス作成
        p = canvas.Canvas(response, pagesize=A4)
        width, height = A4
        
        # ヘッダー
        self._draw_header(p, width, height, invoice_data)
        
        # 請求先情報
        self._draw_billing_info(p, invoice_data)
        
        # 請求明細
        self._draw_items(p, invoice_data)
        
        # 合計金額
        self._draw_totals(p, invoice_data)
        
        # フッター
        self._draw_footer(p, width, height)
        
        p.showPage()
        p.save()
        
        return response
    
    def _draw_header(self, p, width, height, data):
        """
        ヘッダー描画
        """
        # タイトル
        p.setFont('NotoSansJP', 24)
        p.drawString(50*mm, height - 30*mm, '請求書')
        
        # 請求書番号
        p.setFont('NotoSansJP', 10)
        p.drawString(50*mm, height - 40*mm, f'請求書番号: {data["number"]}')
        
        # 発行日
        p.drawString(50*mm, height - 45*mm, f'発行日: {data["date"]}')
        
        # 会社ロゴ（オプション）
        if data.get('logo_path'):
            p.drawImage(
                data['logo_path'],
                width - 80*mm,
                height - 40*mm,
                width=60*mm,
                height=20*mm,
                preserveAspectRatio=True
            )
    
    def _draw_billing_info(self, p, data):
        """
        請求先情報描画
        """
        y_position = 200*mm
        
        # 請求先
        p.setFont('NotoSansJP', 12)
        p.drawString(50*mm, y_position, '【請求先】')
        
        p.setFont('NotoSansJP', 10)
        y_position -= 5*mm
        p.drawString(50*mm, y_position, data['customer_name'])
        
        y_position -= 5*mm
        p.drawString(50*mm, y_position, data['customer_address'])
        
        # 請求元
        y_position = 200*mm
        p.setFont('NotoSansJP', 12)
        p.drawString(120*mm, y_position, '【請求元】')
        
        p.setFont('NotoSansJP', 10)
        y_position -= 5*mm
        p.drawString(120*mm, y_position, data['company_name'])
        
        y_position -= 5*mm
        p.drawString(120*mm, y_position, data['company_address'])
    
    def _draw_items(self, p, data):
        """
        明細描画
        """
        # テーブルヘッダー
        y_position = 170*mm
        p.setFont('NotoSansJP', 10)
        
        # ヘッダー背景
        p.setFillColor(HexColor('#f0f0f0'))
        p.rect(30*mm, y_position - 5*mm, 150*mm, 7*mm, fill=1)
        
        # ヘッダーテキスト
        p.setFillColor(HexColor('#000000'))
        p.drawString(35*mm, y_position, '品目')
        p.drawString(100*mm, y_position, '数量')
        p.drawString(120*mm, y_position, '単価')
        p.drawString(150*mm, y_position, '金額')
        
        # 明細行
        y_position -= 10*mm
        for item in data['items']:
            p.drawString(35*mm, y_position, item['name'])
            p.drawString(100*mm, y_position, str(item['quantity']))
            p.drawString(120*mm, y_position, f'¥{item["price"]:,}')
            p.drawString(150*mm, y_position, f'¥{item["subtotal"]:,}')
            y_position -= 7*mm
    
    def _draw_totals(self, p, data):
        """
        合計金額描画
        """
        y_position = 80*mm
        
        # 小計
        p.setFont('NotoSansJP', 10)
        p.drawString(120*mm, y_position, '小計')
        p.drawString(150*mm, y_position, f'¥{data["subtotal"]:,}')
        
        # 消費税
        y_position -= 7*mm
        p.drawString(120*mm, y_position, f'消費税（{data["tax_rate"]}%）')
        p.drawString(150*mm, y_position, f'¥{data["tax"]:,}')
        
        # 合計
        y_position -= 10*mm
        p.setFont('NotoSansJP', 14)
        p.drawString(120*mm, y_position, '合計金額')
        p.drawString(150*mm, y_position, f'¥{data["total"]:,}')
    
    def _draw_footer(self, p, width, height):
        """
        フッター描画
        """
        p.setFont('NotoSansJP', 8)
        p.drawString(
            50*mm,
            20*mm,
            'お支払いは請求書発行日より30日以内にお願いいたします。'
        )
```

### 2.3 Django Viewでの使用

```python
# reports/views.py

from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404
from .models import Invoice
from .pdf_generator import InvoicePDF

@login_required
def download_invoice_pdf(request, invoice_id):
    """
    請求書PDF生成・ダウンロード
    """
    invoice = get_object_or_404(Invoice, id=invoice_id, user=request.user)
    
    # PDFデータ準備
    invoice_data = {
        'number': invoice.invoice_number,
        'date': invoice.created_at.strftime('%Y年%m月%d日'),
        'customer_name': invoice.customer_name,
        'customer_address': invoice.customer_address,
        'company_name': '株式会社サンプル',
        'company_address': '東京都渋谷区...',
        'items': [
            {
                'name': item.product_name,
                'quantity': item.quantity,
                'price': item.unit_price,
                'subtotal': item.subtotal
            }
            for item in invoice.items.all()
        ],
        'subtotal': invoice.subtotal,
        'tax_rate': 10,
        'tax': invoice.tax_amount,
        'total': invoice.total_amount,
    }
    
    # PDF生成
    pdf_generator = InvoicePDF()
    return pdf_generator.generate_invoice(invoice_data)
```

---

## 3. WeasyPrint実装

### 3.1 インストール

```bash
pip install WeasyPrint
```

### 3.2 HTML/CSSテンプレート

```html
<!-- templates/reports/invoice.html -->

<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <style>
        @page {
            size: A4;
            margin: 20mm;
        }
        
        body {
            font-family: "Noto Sans JP", sans-serif;
            font-size: 12pt;
        }
        
        .header {
            text-align: center;
            margin-bottom: 30px;
        }
        
        .header h1 {
            font-size: 28pt;
            margin: 0;
        }
        
        .invoice-info {
            display: flex;
            justify-content: space-between;
            margin-bottom: 30px;
        }
        
        .billing-address, .company-info {
            width: 45%;
        }
        
        .section-title {
            font-weight: bold;
            border-bottom: 2px solid #333;
            margin-bottom: 10px;
            padding-bottom: 5px;
        }
        
        table {
            width: 100%;
            border-collapse: collapse;
            margin: 20px 0;
        }
        
        th, td {
            border: 1px solid #ddd;
            padding: 10px;
            text-align: left;
        }
        
        th {
            background-color: #f0f0f0;
            font-weight: bold;
        }
        
        .text-right {
            text-align: right;
        }
        
        .totals {
            margin-top: 20px;
            float: right;
            width: 40%;
        }
        
        .totals table {
            margin: 0;
        }
        
        .total-row {
            font-weight: bold;
            font-size: 14pt;
        }
        
        .footer {
            clear: both;
            margin-top: 50px;
            text-align: center;
            font-size: 10pt;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>請求書</h1>
        <p>請求書番号: {{ invoice.invoice_number }}</p>
        <p>発行日: {{ invoice.created_at|date:"Y年m月d日" }}</p>
    </div>
    
    <div class="invoice-info">
        <div class="billing-address">
            <div class="section-title">【請求先】</div>
            <p>{{ invoice.customer_name }} 様</p>
            <p>{{ invoice.customer_address }}</p>
        </div>
        
        <div class="company-info">
            <div class="section-title">【請求元】</div>
            <p>株式会社サンプル</p>
            <p>東京都渋谷区...</p>
            <p>TEL: 03-1234-5678</p>
        </div>
    </div>
    
    <table>
        <thead>
            <tr>
                <th>品目</th>
                <th class="text-right">数量</th>
                <th class="text-right">単価</th>
                <th class="text-right">金額</th>
            </tr>
        </thead>
        <tbody>
            {% for item in invoice.items.all %}
            <tr>
                <td>{{ item.product_name }}</td>
                <td class="text-right">{{ item.quantity }}</td>
                <td class="text-right">¥{{ item.unit_price|floatformat:0|intcomma }}</td>
                <td class="text-right">¥{{ item.subtotal|floatformat:0|intcomma }}</td>
            </tr>
            {% endfor %}
        </tbody>
    </table>
    
    <div class="totals">
        <table>
            <tr>
                <td>小計</td>
                <td class="text-right">¥{{ invoice.subtotal|floatformat:0|intcomma }}</td>
            </tr>
            <tr>
                <td>消費税（10%）</td>
                <td class="text-right">¥{{ invoice.tax_amount|floatformat:0|intcomma }}</td>
            </tr>
            <tr class="total-row">
                <td>合計金額</td>
                <td class="text-right">¥{{ invoice.total_amount|floatformat:0|intcomma }}</td>
            </tr>
        </table>
    </div>
    
    <div class="footer">
        <p>お支払いは請求書発行日より30日以内にお願いいたします。</p>
    </div>
</body>
</html>
```

### 3.3 Django Viewでの実装

```python
# reports/views.py

from django.template.loader import render_to_string
from weasyprint import HTML, CSS
from django.http import HttpResponse
import tempfile

@login_required
def download_invoice_weasyprint(request, invoice_id):
    """
    WeasyPrintで請求書PDF生成
    """
    invoice = get_object_or_404(Invoice, id=invoice_id, user=request.user)
    
    # HTMLレンダリング
    html_string = render_to_string('reports/invoice.html', {
        'invoice': invoice
    })
    
    # PDF生成
    pdf_file = HTML(string=html_string).write_pdf()
    
    # レスポンス
    response = HttpResponse(pdf_file, content_type='application/pdf')
    response['Content-Disposition'] = f'attachment; filename="invoice_{invoice.invoice_number}.pdf"'
    
    return response
```

---

## 4. xhtml2pdf実装

### 4.1 インストール

```bash
pip install xhtml2pdf
```

### 4.2 実装

```python
# reports/views.py

from django.template.loader import get_template
from xhtml2pdf import pisa
from django.http import HttpResponse
from io import BytesIO

@login_required
def download_invoice_xhtml2pdf(request, invoice_id):
    """
    xhtml2pdfで請求書PDF生成
    """
    invoice = get_object_or_404(Invoice, id=invoice_id, user=request.user)
    
    # テンプレート取得
    template = get_template('reports/invoice.html')
    html = template.render({'invoice': invoice})
    
    # PDF生成
    result = BytesIO()
    pdf = pisa.pisaDocument(BytesIO(html.encode("UTF-8")), result)
    
    if not pdf.err:
        response = HttpResponse(result.getvalue(), content_type='application/pdf')
        response['Content-Disposition'] = f'attachment; filename="invoice_{invoice.invoice_number}.pdf"'
        return response
    
    return HttpResponse('PDF生成エラー', status=500)
```

---

## 5. 帳票テンプレート管理

### 5.1 テンプレートモデル

```python
# reports/models.py

from django.db import models

class ReportTemplate(models.Model):
    """
    帳票テンプレートモデル
    """
    TEMPLATE_TYPE_CHOICES = [
        ('invoice', '請求書'),
        ('receipt', '領収書'),
        ('delivery_note', '納品書'),
        ('estimate', '見積書'),
        ('statement', '明細書'),
    ]
    
    name = models.CharField(max_length=100, verbose_name='テンプレート名')
    template_type = models.CharField(
        max_length=20,
        choices=TEMPLATE_TYPE_CHOICES,
        verbose_name='種別'
    )
    html_template = models.TextField(verbose_name='HTMLテンプレート')
    css_template = models.TextField(blank=True, verbose_name='CSSテンプレート')
    is_active = models.BooleanField(default=True, verbose_name='有効')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        verbose_name = '帳票テンプレート'
        verbose_name_plural = '帳票テンプレート'
    
    def __str__(self):
        return f'{self.name} ({self.get_template_type_display()})'
```

### 5.2 動的テンプレート使用

```python
# reports/pdf_service.py

from django.template import Template, Context
from weasyprint import HTML, CSS

class DynamicReportGenerator:
    """
    動的帳票生成サービス
    """
    
    @staticmethod
    def generate_pdf(template_id, context_data):
        """
        テンプレートIDとコンテキストからPDF生成
        """
        template_obj = ReportTemplate.objects.get(id=template_id)
        
        # テンプレートレンダリング
        template = Template(template_obj.html_template)
        context = Context(context_data)
        html_string = template.render(context)
        
        # CSS適用
        css = CSS(string=template_obj.css_template) if template_obj.css_template else None
        
        # PDF生成
        pdf_file = HTML(string=html_string).write_pdf(stylesheets=[css] if css else None)
        
        return pdf_file
```

---

## 6. 日本語フォント対応

### 6.1 Notoフォントのインストール

```bash
# Notoフォントダウンロード
wget https://github.com/googlefonts/noto-cjk/raw/main/Sans/OTF/Japanese/NotoSansJP-Regular.otf

# プロジェクトのfontsディレクトリに配置
mkdir -p static/fonts
mv NotoSansJP-Regular.otf static/fonts/
```

### 6.2 WeasyPrintでのフォント設定

```python
# settings.py

# WeasyPrint用フォント設定
WEASYPRINT_BASEURL = os.path.join(BASE_DIR, 'static')

# フォントディレクトリ
FONT_CONFIG = os.path.join(BASE_DIR, 'static', 'fonts')
```

```html
<!-- テンプレートでのフォント指定 -->
<style>
    @font-face {
        font-family: 'Noto Sans JP';
        src: url('fonts/NotoSansJP-Regular.otf');
    }
    
    body {
        font-family: 'Noto Sans JP', sans-serif;
    }
</style>
```

---

## 7. 高度な帳票機能

### 7.1 ページ番号付与

```python
# ReportLabでのページ番号

def add_page_number(canvas, doc):
    """
    ページ番号を追加
    """
    page_num = canvas.getPageNumber()
    text = f"Page {page_num}"
    canvas.drawRightString(200*mm, 20*mm, text)

# 使用例
from reportlab.platypus import SimpleDocTemplate

doc = SimpleDocTemplate("output.pdf", pagesize=A4)
doc.build(story, onFirstPage=add_page_number, onLaterPages=add_page_number)
```

### 7.2 ヘッダー・フッター

```html
<!-- WeasyPrintでのヘッダー・フッター -->
<style>
    @page {
        @top-center {
            content: "請求書";
            font-size: 14pt;
        }
        
        @bottom-right {
            content: "Page " counter(page) " of " counter(pages);
        }
    }
</style>
```

### 7.3 バーコード・QRコード

```python
# reports/barcode_utils.py

import qrcode
import barcode
from barcode.writer import ImageWriter
from io import BytesIO
import base64

def generate_qr_code(data):
    """
    QRコード生成
    """
    qr = qrcode.QRCode(version=1, box_size=10, border=5)
    qr.add_data(data)
    qr.make(fit=True)
    
    img = qr.make_image(fill_color="black", back_color="white")
    
    # Base64エンコード
    buffer = BytesIO()
    img.save(buffer, format='PNG')
    img_str = base64.b64encode(buffer.getvalue()).decode()
    
    return f'data:image/png;base64,{img_str}'

def generate_barcode(data, barcode_type='code128'):
    """
    バーコード生成
    """
    barcode_class = barcode.get_barcode_class(barcode_type)
    barcode_instance = barcode_class(data, writer=ImageWriter())
    
    buffer = BytesIO()
    barcode_instance.write(buffer)
    img_str = base64.b64encode(buffer.getvalue()).decode()
    
    return f'data:image/png;base64,{img_str}'
```

### 7.4 透かし（ウォーターマーク）

```python
# reports/watermark.py

from PyPDF2 import PdfReader, PdfWriter
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4
from io import BytesIO

def add_watermark(input_pdf, watermark_text):
    """
    PDFに透かしを追加
    """
    # 透かしPDF作成
    watermark_buffer = BytesIO()
    c = canvas.Canvas(watermark_buffer, pagesize=A4)
    
    width, height = A4
    c.saveState()
    c.translate(width/2, height/2)
    c.rotate(45)
    c.setFont('Helvetica', 60)
    c.setFillGray(0.5, 0.5)
    c.drawCentredString(0, 0, watermark_text)
    c.restoreState()
    c.save()
    
    # 元PDFと透かしを結合
    watermark_buffer.seek(0)
    watermark_pdf = PdfReader(watermark_buffer)
    input_pdf_reader = PdfReader(input_pdf)
    output = PdfWriter()
    
    for page in input_pdf_reader.pages:
        page.merge_page(watermark_pdf.pages[0])
        output.add_page(page)
    
    output_buffer = BytesIO()
    output.write(output_buffer)
    output_buffer.seek(0)
    
    return output_buffer
```

---

## 8. パフォーマンス最適化

### 8.1 非同期処理（Celery）

```python
# reports/tasks.py

from celery import shared_task
from django.core.mail import EmailMessage
from .pdf_service import DynamicReportGenerator

@shared_task
def generate_and_email_report(template_id, context_data, recipient_email):
    """
    帳票PDF生成とメール送信（非同期）
    """
    # PDF生成
    pdf_file = DynamicReportGenerator.generate_pdf(template_id, context_data)
    
    # メール送信
    email = EmailMessage(
        '帳票PDFをお送りします',
        '添付ファイルをご確認ください。',
        'noreply@example.com',
        [recipient_email]
    )
    email.attach('report.pdf', pdf_file, 'application/pdf')
    email.send()
    
    return True
```

### 8.2 キャッシング

```python
# reports/views.py

from django.core.cache import cache
import hashlib

@login_required
def download_cached_pdf(request, invoice_id):
    """
    キャッシュを使用したPDF生成
    """
    invoice = get_object_or_404(Invoice, id=invoice_id, user=request.user)
    
    # キャッシュキー生成
    cache_key = f'invoice_pdf_{invoice_id}_{invoice.updated_at.timestamp()}'
    
    # キャッシュから取得
    pdf_content = cache.get(cache_key)
    
    if not pdf_content:
        # PDF生成
        html_string = render_to_string('reports/invoice.html', {'invoice': invoice})
        pdf_content = HTML(string=html_string).write_pdf()
        
        # キャッシュに保存（1時間）
        cache.set(cache_key, pdf_content, 3600)
    
    response = HttpResponse(pdf_content, content_type='application/pdf')
    response['Content-Disposition'] = f'attachment; filename="invoice_{invoice.invoice_number}.pdf"'
    
    return response
```

### 8.3 バッチ処理

```python
# reports/batch_processor.py

from django.db import transaction
from concurrent.futures import ThreadPoolExecutor
import logging

logger = logging.getLogger(__name__)

class BatchReportProcessor:
    """
    バッチ帳票処理
    """
    
    @staticmethod
    def process_batch(invoice_ids, max_workers=4):
        """
        複数請求書のPDF一括生成
        """
        results = []
        
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            futures = []
            
            for invoice_id in invoice_ids:
                future = executor.submit(
                    BatchReportProcessor._generate_single_pdf,
                    invoice_id
                )
                futures.append(future)
            
            for future in futures:
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    logger.error(f'PDF生成エラー: {str(e)}')
        
        return results
    
    @staticmethod
    def _generate_single_pdf(invoice_id):
        """
        単一PDF生成
        """
        invoice = Invoice.objects.get(id=invoice_id)
        
        html_string = render_to_string('reports/invoice.html', {'invoice': invoice})
        pdf_file = HTML(string=html_string).write_pdf()
        
        # ファイル保存
        file_path = f'media/invoices/invoice_{invoice.invoice_number}.pdf'
        with open(file_path, 'wb') as f:
            f.write(pdf_file)
        
        return file_path
```

---

## まとめ

DjangoでのPDF帳票生成は、用途に応じて適切なライブラリを選択することが重要です。

**推奨事項:**
1. **HTML/CSS活用** - WeasyPrintでデザイナーフレンドリーな帳票作成
2. **日本語対応** - Notoフォント等の適切なフォント使用
3. **テンプレート管理** - データベースでテンプレート管理し柔軟性確保
4. **非同期処理** - 大量PDF生成はCeleryで非同期化
5. **キャッシング** - 同一PDFの再生成を避ける
6. **エラーハンドリング** - PDFエラーを適切に処理

業務アプリケーションにおいて、PDF帳票は重要な機能です。適切な実装で高品質な帳票システムを構築してください。
