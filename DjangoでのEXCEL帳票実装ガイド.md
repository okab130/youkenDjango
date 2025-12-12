# DjangoでのEXCEL帳票実装ガイド

## 目次
1. [EXCEL帳票の概要](#1-excel帳票の概要)
2. [openpyxl実装](#2-openpyxl実装)
3. [xlsxwriter実装](#3-xlsxwriter実装)
4. [pandas実装](#4-pandas実装)
5. [テンプレートベース生成](#5-テンプレートベース生成)
6. [高度な書式設定](#6-高度な書式設定)
7. [大量データ処理](#7-大量データ処理)
8. [実用的な帳票例](#8-実用的な帳票例)

---

## 1. EXCEL帳票の概要

### 1.1 主要ライブラリ比較

| ライブラリ | 読込 | 書込 | 数式 | グラフ | 難易度 | 特徴 |
|-----------|------|------|------|--------|--------|------|
| openpyxl | ○ | ○ | ○ | ○ | ★★☆☆☆ | 最も汎用的、Excel2010+ |
| xlsxwriter | × | ○ | ○ | ○ | ★★☆☆☆ | 書込専用、高速 |
| pandas | ○ | ○ | △ | △ | ★☆☆☆☆ | データ分析向け |
| xlrd/xlwt | ○ | ○ | × | × | ★★☆☆☆ | .xls形式（旧Excel） |
| pyexcel | ○ | ○ | △ | × | ★☆☆☆☆ | シンプルAPI |

### 1.2 用途別推奨ライブラリ

```
用途別推奨:

1. 複雑な帳票（書式・数式・グラフ）
   └─ openpyxl（読み書き可能）

2. 大量データ出力（高速）
   └─ xlsxwriter（書込専用）

3. データ分析・集計
   └─ pandas（DataFrame活用）

4. テンプレート編集
   └─ openpyxl（既存ファイル編集）

5. シンプルなデータ出力
   └─ pandas（簡単）
```

### 1.3 基本的なワークフロー

```python
# 基本的なEXCEL生成フロー

1. ワークブック作成
   workbook = Workbook()

2. シート取得/作成
   sheet = workbook.active

3. データ書込
   sheet['A1'] = 'データ'

4. 書式設定
   cell.font = Font(bold=True)

5. ファイル保存/レスポンス
   workbook.save('output.xlsx')
```

---

## 2. openpyxl実装

### 2.1 インストール

```bash
pip install openpyxl
```

### 2.2 基本実装

```python
# reports/excel_generator.py

from openpyxl import Workbook
from openpyxl.styles import Font, Alignment, PatternFill, Border, Side
from openpyxl.utils import get_column_letter
from django.http import HttpResponse
from datetime import datetime
import io

class InvoiceExcelGenerator:
    """
    請求書EXCEL生成クラス
    """
    
    def generate_invoice(self, invoice):
        """
        請求書EXCEL生成
        """
        # ワークブック作成
        wb = Workbook()
        ws = wb.active
        ws.title = "請求書"
        
        # ヘッダー
        self._create_header(ws, invoice)
        
        # 請求先・請求元情報
        self._create_party_info(ws, invoice)
        
        # 明細ヘッダー
        self._create_items_header(ws)
        
        # 明細データ
        self._create_items_data(ws, invoice)
        
        # 合計
        self._create_totals(ws, invoice)
        
        # 列幅調整
        self._adjust_column_width(ws)
        
        # HTTPレスポンス
        return self._create_response(wb, invoice)
    
    def _create_header(self, ws, invoice):
        """
        ヘッダー作成
        """
        # タイトル
        ws['A1'] = '請求書'
        ws['A1'].font = Font(name='メイリオ', size=20, bold=True)
        ws['A1'].alignment = Alignment(horizontal='center')
        ws.merge_cells('A1:F1')
        
        # 請求書番号
        ws['A2'] = f'請求書番号: {invoice.invoice_number}'
        ws['A2'].font = Font(name='メイリオ', size=10)
        
        # 発行日
        ws['A3'] = f'発行日: {invoice.created_at.strftime("%Y年%m月%d日")}'
        ws['A3'].font = Font(name='メイリオ', size=10)
    
    def _create_party_info(self, ws, invoice):
        """
        請求先・請求元情報
        """
        # 請求先
        ws['A5'] = '【請求先】'
        ws['A5'].font = Font(name='メイリオ', size=12, bold=True)
        
        ws['A6'] = invoice.customer_name
        ws['A7'] = invoice.customer_address
        
        # 請求元
        ws['D5'] = '【請求元】'
        ws['D5'].font = Font(name='メイリオ', size=12, bold=True)
        
        ws['D6'] = '株式会社サンプル'
        ws['D7'] = '東京都渋谷区...'
    
    def _create_items_header(self, ws):
        """
        明細ヘッダー作成
        """
        headers = ['品目', '数量', '単価', '金額']
        header_cells = ['A9', 'B9', 'C9', 'D9']
        
        # ヘッダー背景色
        header_fill = PatternFill(start_color='D3D3D3', end_color='D3D3D3', fill_type='solid')
        
        # 罫線
        thin_border = Border(
            left=Side(style='thin'),
            right=Side(style='thin'),
            top=Side(style='thin'),
            bottom=Side(style='thin')
        )
        
        for cell_ref, header in zip(header_cells, headers):
            cell = ws[cell_ref]
            cell.value = header
            cell.font = Font(name='メイリオ', size=11, bold=True)
            cell.fill = header_fill
            cell.alignment = Alignment(horizontal='center')
            cell.border = thin_border
    
    def _create_items_data(self, ws, invoice):
        """
        明細データ作成
        """
        thin_border = Border(
            left=Side(style='thin'),
            right=Side(style='thin'),
            top=Side(style='thin'),
            bottom=Side(style='thin')
        )
        
        row = 10
        for item in invoice.items.all():
            ws[f'A{row}'] = item.product_name
            ws[f'B{row}'] = item.quantity
            ws[f'C{row}'] = item.unit_price
            ws[f'D{row}'] = item.subtotal
            
            # 数値セルの書式設定
            ws[f'B{row}'].number_format = '#,##0'
            ws[f'C{row}'].number_format = '¥#,##0'
            ws[f'D{row}'].number_format = '¥#,##0'
            
            # 罫線
            for col in ['A', 'B', 'C', 'D']:
                ws[f'{col}{row}'].border = thin_border
            
            row += 1
        
        return row
    
    def _create_totals(self, ws, invoice):
        """
        合計作成
        """
        last_row = 10 + invoice.items.count()
        
        # 小計
        ws[f'C{last_row + 1}'] = '小計'
        ws[f'D{last_row + 1}'] = invoice.subtotal
        ws[f'D{last_row + 1}'].number_format = '¥#,##0'
        
        # 消費税
        ws[f'C{last_row + 2}'] = '消費税（10%）'
        ws[f'D{last_row + 2}'] = invoice.tax_amount
        ws[f'D{last_row + 2}'].number_format = '¥#,##0'
        
        # 合計
        ws[f'C{last_row + 3}'] = '合計金額'
        ws[f'D{last_row + 3}'] = invoice.total_amount
        ws[f'C{last_row + 3}'].font = Font(name='メイリオ', size=12, bold=True)
        ws[f'D{last_row + 3}'].font = Font(name='メイリオ', size=12, bold=True)
        ws[f'D{last_row + 3}'].number_format = '¥#,##0'
    
    def _adjust_column_width(self, ws):
        """
        列幅調整
        """
        ws.column_dimensions['A'].width = 30
        ws.column_dimensions['B'].width = 10
        ws.column_dimensions['C'].width = 15
        ws.column_dimensions['D'].width = 15
    
    def _create_response(self, wb, invoice):
        """
        HTTPレスポンス作成
        """
        # メモリ上にEXCEL保存
        output = io.BytesIO()
        wb.save(output)
        output.seek(0)
        
        # レスポンス作成
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = f'attachment; filename="invoice_{invoice.invoice_number}.xlsx"'
        
        return response
```

### 2.3 Django Viewでの使用

```python
# reports/views.py

from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404
from .models import Invoice
from .excel_generator import InvoiceExcelGenerator

@login_required
def download_invoice_excel(request, invoice_id):
    """
    請求書EXCEL生成・ダウンロード
    """
    invoice = get_object_or_404(Invoice, id=invoice_id, user=request.user)
    
    generator = InvoiceExcelGenerator()
    return generator.generate_invoice(invoice)
```

---

## 3. xlsxwriter実装

### 3.1 インストール

```bash
pip install xlsxwriter
```

### 3.2 基本実装

```python
# reports/xlsxwriter_generator.py

import xlsxwriter
from django.http import HttpResponse
import io

class SalesReportGenerator:
    """
    売上レポートEXCEL生成（xlsxwriter）
    """
    
    def generate_report(self, sales_data):
        """
        売上レポート生成
        """
        # メモリ上にワークブック作成
        output = io.BytesIO()
        workbook = xlsxwriter.Workbook(output, {'in_memory': True})
        worksheet = workbook.add_worksheet('売上レポート')
        
        # 書式定義
        formats = self._create_formats(workbook)
        
        # タイトル
        worksheet.write('A1', '月次売上レポート', formats['title'])
        worksheet.merge_range('A1:F1', '月次売上レポート', formats['title'])
        
        # ヘッダー
        headers = ['日付', '商品名', '数量', '単価', '売上', '累計']
        for col, header in enumerate(headers):
            worksheet.write(2, col, header, formats['header'])
        
        # データ
        row = 3
        cumulative = 0
        for data in sales_data:
            worksheet.write(row, 0, data['date'], formats['date'])
            worksheet.write(row, 1, data['product_name'])
            worksheet.write(row, 2, data['quantity'], formats['number'])
            worksheet.write(row, 3, data['unit_price'], formats['currency'])
            worksheet.write(row, 4, data['sales'], formats['currency'])
            
            cumulative += data['sales']
            worksheet.write(row, 5, cumulative, formats['currency'])
            
            row += 1
        
        # グラフ追加
        self._add_chart(workbook, worksheet, len(sales_data))
        
        # 列幅調整
        worksheet.set_column('A:A', 12)
        worksheet.set_column('B:B', 25)
        worksheet.set_column('C:C', 10)
        worksheet.set_column('D:F', 15)
        
        workbook.close()
        output.seek(0)
        
        # レスポンス
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = 'attachment; filename="sales_report.xlsx"'
        
        return response
    
    def _create_formats(self, workbook):
        """
        書式作成
        """
        return {
            'title': workbook.add_format({
                'bold': True,
                'font_size': 16,
                'align': 'center',
                'valign': 'vcenter',
                'fg_color': '#4472C4',
                'font_color': 'white',
            }),
            'header': workbook.add_format({
                'bold': True,
                'align': 'center',
                'valign': 'vcenter',
                'fg_color': '#D3D3D3',
                'border': 1,
            }),
            'date': workbook.add_format({
                'num_format': 'yyyy/mm/dd',
                'border': 1,
            }),
            'number': workbook.add_format({
                'num_format': '#,##0',
                'border': 1,
            }),
            'currency': workbook.add_format({
                'num_format': '¥#,##0',
                'border': 1,
            }),
        }
    
    def _add_chart(self, workbook, worksheet, data_count):
        """
        グラフ追加
        """
        chart = workbook.add_chart({'type': 'column'})
        
        # データ系列追加
        chart.add_series({
            'name': '日次売上',
            'categories': f'=売上レポート!$A$4:$A${3 + data_count}',
            'values': f'=売上レポート!$E$4:$E${3 + data_count}',
        })
        
        # グラフタイトル・軸ラベル
        chart.set_title({'name': '日次売上推移'})
        chart.set_x_axis({'name': '日付'})
        chart.set_y_axis({'name': '売上金額'})
        
        # グラフ配置
        worksheet.insert_chart('H3', chart)
```

---

## 4. pandas実装

### 4.1 インストール

```bash
pip install pandas openpyxl
```

### 4.2 基本実装

```python
# reports/pandas_generator.py

import pandas as pd
from django.http import HttpResponse
import io

class DataExportGenerator:
    """
    データエクスポート（pandas）
    """
    
    def export_to_excel(self, queryset, columns, filename='export.xlsx'):
        """
        QuerySetをEXCELエクスポート
        """
        # DataFrameに変換
        df = pd.DataFrame(list(queryset.values(*columns)))
        
        # 列名を日本語に変換（オプション）
        df.columns = self._translate_columns(df.columns)
        
        # メモリ上にEXCEL作成
        output = io.BytesIO()
        
        with pd.ExcelWriter(output, engine='openpyxl') as writer:
            df.to_excel(writer, sheet_name='データ', index=False)
            
            # ワークシート取得して書式設定
            workbook = writer.book
            worksheet = writer.sheets['データ']
            
            # ヘッダー書式
            for cell in worksheet[1]:
                cell.font = Font(bold=True)
                cell.fill = PatternFill(start_color='D3D3D3', end_color='D3D3D3', fill_type='solid')
            
            # 列幅自動調整
            for column in worksheet.columns:
                max_length = 0
                column_letter = column[0].column_letter
                for cell in column:
                    if cell.value:
                        max_length = max(max_length, len(str(cell.value)))
                worksheet.column_dimensions[column_letter].width = max_length + 2
        
        output.seek(0)
        
        # レスポンス
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = f'attachment; filename="{filename}"'
        
        return response
    
    def _translate_columns(self, columns):
        """
        列名を日本語に変換
        """
        translation = {
            'id': 'ID',
            'name': '名前',
            'email': 'メールアドレス',
            'created_at': '作成日時',
            'updated_at': '更新日時',
        }
        return [translation.get(col, col) for col in columns]
    
    def export_multi_sheet(self, data_dict, filename='multi_sheet.xlsx'):
        """
        複数シートのEXCELエクスポート
        """
        output = io.BytesIO()
        
        with pd.ExcelWriter(output, engine='openpyxl') as writer:
            for sheet_name, df in data_dict.items():
                df.to_excel(writer, sheet_name=sheet_name, index=False)
        
        output.seek(0)
        
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = f'attachment; filename="{filename}"'
        
        return response
```

### 4.3 集計レポート

```python
# reports/aggregation_report.py

import pandas as pd

class AggregationReportGenerator:
    """
    集計レポート生成
    """
    
    def generate_summary_report(self, orders):
        """
        注文集計レポート
        """
        # DataFrameに変換
        df = pd.DataFrame(list(orders.values(
            'created_at',
            'customer_name',
            'total_amount',
            'status'
        )))
        
        # 日付型に変換
        df['created_at'] = pd.to_datetime(df['created_at'])
        
        # 月別集計
        df['month'] = df['created_at'].dt.to_period('M')
        monthly_summary = df.groupby('month').agg({
            'total_amount': ['sum', 'mean', 'count']
        }).reset_index()
        
        # ステータス別集計
        status_summary = df.groupby('status').agg({
            'total_amount': 'sum',
            'customer_name': 'count'
        }).reset_index()
        
        # EXCEL出力
        output = io.BytesIO()
        
        with pd.ExcelWriter(output, engine='openpyxl') as writer:
            # 元データ
            df.to_excel(writer, sheet_name='明細', index=False)
            
            # 月別集計
            monthly_summary.to_excel(writer, sheet_name='月別集計', index=False)
            
            # ステータス別集計
            status_summary.to_excel(writer, sheet_name='ステータス別', index=False)
        
        output.seek(0)
        
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = 'attachment; filename="summary_report.xlsx"'
        
        return response
```

---

## 5. テンプレートベース生成

### 5.1 テンプレートファイルの準備

```python
# reports/template_generator.py

from openpyxl import load_workbook
from django.conf import settings
import os
from django.http import HttpResponse
import io

class TemplateBasedGenerator:
    """
    テンプレートベースEXCEL生成
    """
    
    def __init__(self, template_name):
        """
        テンプレートファイル読込
        """
        template_path = os.path.join(
            settings.BASE_DIR,
            'templates',
            'excel',
            template_name
        )
        self.wb = load_workbook(template_path)
    
    def fill_invoice_template(self, invoice):
        """
        請求書テンプレート埋め込み
        """
        ws = self.wb.active
        
        # ヘッダー情報埋め込み
        ws['B2'] = invoice.invoice_number
        ws['B3'] = invoice.created_at.strftime('%Y年%m月%d日')
        
        # 請求先情報
        ws['B5'] = invoice.customer_name
        ws['B6'] = invoice.customer_address
        
        # 明細埋め込み
        start_row = 10
        for idx, item in enumerate(invoice.items.all()):
            row = start_row + idx
            ws[f'A{row}'] = item.product_name
            ws[f'B{row}'] = item.quantity
            ws[f'C{row}'] = item.unit_price
            ws[f'D{row}'] = item.subtotal
        
        # 合計
        total_row = start_row + invoice.items.count() + 2
        ws[f'C{total_row}'] = invoice.subtotal
        ws[f'C{total_row + 1}'] = invoice.tax_amount
        ws[f'C{total_row + 2}'] = invoice.total_amount
        
        return self._create_response(invoice.invoice_number)
    
    def _create_response(self, filename):
        """
        HTTPレスポンス作成
        """
        output = io.BytesIO()
        self.wb.save(output)
        output.seek(0)
        
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = f'attachment; filename="invoice_{filename}.xlsx"'
        
        return response
```

---

## 6. 高度な書式設定

### 6.1 条件付き書式

```python
# reports/conditional_formatting.py

from openpyxl.formatting.rule import ColorScaleRule, CellIsRule
from openpyxl.styles import PatternFill

def apply_conditional_formatting(worksheet, data_range):
    """
    条件付き書式適用
    """
    # カラースケール（売上金額）
    color_scale = ColorScaleRule(
        start_type='min',
        start_color='FFFFFF',
        end_type='max',
        end_color='63BE7B'
    )
    worksheet.conditional_formatting.add(f'E4:E{data_range}', color_scale)
    
    # セル値による書式（在庫警告）
    red_fill = PatternFill(start_color='FF6B6B', end_color='FF6B6B', fill_type='solid')
    warning_rule = CellIsRule(
        operator='lessThan',
        formula=['10'],
        fill=red_fill
    )
    worksheet.conditional_formatting.add(f'C4:C{data_range}', warning_rule)
```

### 6.2 データ検証

```python
# reports/data_validation.py

from openpyxl.worksheet.datavalidation import DataValidation

def add_data_validation(worksheet):
    """
    データ検証追加
    """
    # ドロップダウンリスト
    status_validation = DataValidation(
        type='list',
        formula1='"完了,処理中,保留"',
        allow_blank=False
    )
    worksheet.add_data_validation(status_validation)
    status_validation.add('F4:F100')
    
    # 数値範囲
    quantity_validation = DataValidation(
        type='whole',
        operator='between',
        formula1=1,
        formula2=1000
    )
    worksheet.add_data_validation(quantity_validation)
    quantity_validation.add('C4:C100')
```

### 6.3 数式とピボットテーブル

```python
# reports/formulas.py

from openpyxl.utils import get_column_letter

def add_formulas(worksheet, data_rows):
    """
    数式追加
    """
    # 合計行
    total_row = data_rows + 4
    
    # SUM関数
    worksheet[f'E{total_row}'] = f'=SUM(E4:E{data_rows + 3})'
    
    # 平均
    worksheet[f'E{total_row + 1}'] = f'=AVERAGE(E4:E{data_rows + 3})'
    
    # 最大値
    worksheet[f'E{total_row + 2}'] = f'=MAX(E4:E{data_rows + 3})'
    
    # ラベル
    worksheet[f'D{total_row}'] = '合計'
    worksheet[f'D{total_row + 1}'] = '平均'
    worksheet[f'D{total_row + 2}'] = '最大値'
```

---

## 7. 大量データ処理

### 7.1 ストリーミング書き込み

```python
# reports/streaming_writer.py

from openpyxl import Workbook
from openpyxl.writer.excel import save_virtual_workbook

class StreamingExcelWriter:
    """
    大量データストリーミング書込
    """
    
    def generate_large_report(self, queryset):
        """
        大量データレポート生成
        """
        wb = Workbook(write_only=True)
        ws = wb.create_sheet()
        
        # ヘッダー
        ws.append(['ID', '名前', '金額', '日付'])
        
        # データをバッチで処理
        batch_size = 1000
        for i in range(0, queryset.count(), batch_size):
            batch = queryset[i:i + batch_size]
            for item in batch:
                ws.append([
                    item.id,
                    item.name,
                    item.amount,
                    item.date
                ])
        
        # 保存
        output = io.BytesIO()
        wb.save(output)
        output.seek(0)
        
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = 'attachment; filename="large_report.xlsx"'
        
        return response
```

### 7.2 非同期生成（Celery）

```python
# reports/tasks.py

from celery import shared_task
from django.core.mail import EmailMessage
from .excel_generator import InvoiceExcelGenerator

@shared_task
def generate_and_email_excel(invoice_id, recipient_email):
    """
    EXCEL生成とメール送信（非同期）
    """
    from .models import Invoice
    
    invoice = Invoice.objects.get(id=invoice_id)
    
    # EXCEL生成
    generator = InvoiceExcelGenerator()
    output = io.BytesIO()
    
    wb = generator._create_workbook(invoice)
    wb.save(output)
    output.seek(0)
    
    # メール送信
    email = EmailMessage(
        '請求書をお送りします',
        '添付ファイルをご確認ください。',
        'noreply@example.com',
        [recipient_email]
    )
    email.attach(
        f'invoice_{invoice.invoice_number}.xlsx',
        output.read(),
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
    )
    email.send()
    
    return True
```

---

## 8. 実用的な帳票例

### 8.1 在庫一覧表

```python
# reports/inventory_report.py

from openpyxl import Workbook
from openpyxl.styles import Font, Alignment, PatternFill

class InventoryReportGenerator:
    """
    在庫一覧表生成
    """
    
    def generate(self, products):
        """
        在庫一覧表生成
        """
        wb = Workbook()
        ws = wb.active
        ws.title = "在庫一覧"
        
        # タイトル
        ws['A1'] = '在庫一覧表'
        ws['A1'].font = Font(size=16, bold=True)
        ws.merge_cells('A1:G1')
        
        # ヘッダー
        headers = ['商品コード', '商品名', 'カテゴリ', '在庫数', '単価', '在庫金額', 'ステータス']
        for col, header in enumerate(headers, 1):
            cell = ws.cell(row=3, column=col)
            cell.value = header
            cell.font = Font(bold=True)
            cell.fill = PatternFill(start_color='366092', end_color='366092', fill_type='solid')
            cell.font = Font(color='FFFFFF', bold=True)
            cell.alignment = Alignment(horizontal='center')
        
        # データ
        row = 4
        total_value = 0
        
        for product in products:
            stock_value = product.stock * product.price
            total_value += stock_value
            
            ws[f'A{row}'] = product.code
            ws[f'B{row}'] = product.name
            ws[f'C{row}'] = product.category.name
            ws[f'D{row}'] = product.stock
            ws[f'E{row}'] = product.price
            ws[f'F{row}'] = stock_value
            
            # ステータス判定
            if product.stock < 10:
                status = '要発注'
                ws[f'G{row}'].fill = PatternFill(start_color='FF6B6B', end_color='FF6B6B', fill_type='solid')
            elif product.stock < 50:
                status = '警告'
                ws[f'G{row}'].fill = PatternFill(start_color='FFD93D', end_color='FFD93D', fill_type='solid')
            else:
                status = '正常'
            
            ws[f'G{row}'] = status
            
            # 数値書式
            ws[f'D{row}'].number_format = '#,##0'
            ws[f'E{row}'].number_format = '¥#,##0'
            ws[f'F{row}'].number_format = '¥#,##0'
            
            row += 1
        
        # 合計行
        ws[f'E{row}'] = '合計在庫金額'
        ws[f'F{row}'] = total_value
        ws[f'E{row}'].font = Font(bold=True)
        ws[f'F{row}'].font = Font(bold=True)
        ws[f'F{row}'].number_format = '¥#,##0'
        
        # 列幅調整
        ws.column_dimensions['A'].width = 15
        ws.column_dimensions['B'].width = 30
        ws.column_dimensions['C'].width = 15
        ws.column_dimensions['D'].width = 12
        ws.column_dimensions['E'].width = 12
        ws.column_dimensions['F'].width = 15
        ws.column_dimensions['G'].width = 12
        
        # レスポンス
        return self._create_response(wb)
    
    def _create_response(self, wb):
        output = io.BytesIO()
        wb.save(output)
        output.seek(0)
        
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = 'attachment; filename="inventory_report.xlsx"'
        
        return response
```

### 8.2 勤怠管理表

```python
# reports/attendance_report.py

from openpyxl import Workbook
from datetime import datetime, timedelta

class AttendanceReportGenerator:
    """
    勤怠管理表生成
    """
    
    def generate_monthly_report(self, employee, year, month):
        """
        月次勤怠表生成
        """
        wb = Workbook()
        ws = wb.active
        ws.title = f"{year}年{month}月"
        
        # タイトル
        ws['A1'] = f'{year}年{month}月 勤怠管理表'
        ws['A1'].font = Font(size=14, bold=True)
        ws.merge_cells('A1:H1')
        
        # 従業員情報
        ws['A2'] = f'氏名: {employee.name}'
        ws['E2'] = f'社員番号: {employee.employee_number}'
        
        # ヘッダー
        headers = ['日付', '曜日', '出勤', '退勤', '休憩', '勤務時間', '残業', '備考']
        for col, header in enumerate(headers, 1):
            cell = ws.cell(row=4, column=col)
            cell.value = header
            cell.font = Font(bold=True)
            cell.fill = PatternFill(start_color='D3D3D3', end_color='D3D3D3', fill_type='solid')
        
        # 日付データ
        start_date = datetime(year, month, 1)
        if month == 12:
            end_date = datetime(year + 1, 1, 1) - timedelta(days=1)
        else:
            end_date = datetime(year, month + 1, 1) - timedelta(days=1)
        
        row = 5
        total_hours = 0
        total_overtime = 0
        
        current_date = start_date
        while current_date <= end_date:
            # 勤怠データ取得
            attendance = employee.attendances.filter(date=current_date).first()
            
            ws[f'A{row}'] = current_date.strftime('%m/%d')
            ws[f'B{row}'] = ['月', '火', '水', '木', '金', '土', '日'][current_date.weekday()]
            
            if attendance:
                ws[f'C{row}'] = attendance.clock_in.strftime('%H:%M')
                ws[f'D{row}'] = attendance.clock_out.strftime('%H:%M')
                ws[f'E{row}'] = attendance.break_time
                ws[f'F{row}'] = attendance.work_hours
                ws[f'G{row}'] = attendance.overtime_hours
                ws[f'H{row}'] = attendance.note
                
                total_hours += attendance.work_hours
                total_overtime += attendance.overtime_hours
            else:
                # 土日の背景色
                if current_date.weekday() >= 5:
                    for col in range(1, 9):
                        ws.cell(row=row, column=col).fill = PatternFill(
                            start_color='E8E8E8',
                            end_color='E8E8E8',
                            fill_type='solid'
                        )
            
            row += 1
            current_date += timedelta(days=1)
        
        # 合計行
        ws[f'E{row}'] = '合計'
        ws[f'F{row}'] = total_hours
        ws[f'G{row}'] = total_overtime
        ws[f'E{row}'].font = Font(bold=True)
        ws[f'F{row}'].font = Font(bold=True)
        ws[f'G{row}'].font = Font(bold=True)
        
        # 列幅調整
        ws.column_dimensions['A'].width = 10
        ws.column_dimensions['B'].width = 8
        ws.column_dimensions['C'].width = 10
        ws.column_dimensions['D'].width = 10
        ws.column_dimensions['E'].width = 10
        ws.column_dimensions['F'].width = 12
        ws.column_dimensions['G'].width = 10
        ws.column_dimensions['H'].width = 30
        
        return self._create_response(wb, employee, year, month)
    
    def _create_response(self, wb, employee, year, month):
        output = io.BytesIO()
        wb.save(output)
        output.seek(0)
        
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        filename = f'attendance_{employee.employee_number}_{year}{month:02d}.xlsx'
        response['Content-Disposition'] = f'attachment; filename="{filename}"'
        
        return response
```

### 8.3 見積書

```python
# reports/quotation_generator.py

from openpyxl import Workbook
from openpyxl.styles import Font, Alignment, PatternFill, Border, Side

class QuotationGenerator:
    """
    見積書生成
    """
    
    def generate(self, quotation):
        """
        見積書生成
        """
        wb = Workbook()
        ws = wb.active
        ws.title = "見積書"
        
        # 見積書タイトル
        ws['A1'] = '御見積書'
        ws['A1'].font = Font(size=20, bold=True)
        ws['A1'].alignment = Alignment(horizontal='center')
        ws.merge_cells('A1:F1')
        
        # 見積先
        ws['A3'] = quotation.customer_name + ' 御中'
        ws['A3'].font = Font(size=12)
        
        # 見積情報
        ws['E3'] = f'見積番号: {quotation.quotation_number}'
        ws['E4'] = f'見積日: {quotation.created_at.strftime("%Y年%m月%d日")}'
        ws['E5'] = f'有効期限: {quotation.valid_until.strftime("%Y年%m月%d日")}'
        
        # 会社情報
        ws['E7'] = '株式会社サンプル'
        ws['E8'] = '〒123-4567'
        ws['E9'] = '東京都渋谷区...'
        ws['E10'] = 'TEL: 03-1234-5678'
        
        # 件名
        ws['A12'] = '件名: ' + quotation.subject
        ws['A12'].font = Font(size=11, bold=True)
        
        # 明細ヘッダー
        headers = ['No.', '品目', '仕様', '数量', '単価', '金額']
        thin_border = Border(
            left=Side(style='thin'),
            right=Side(style='thin'),
            top=Side(style='thin'),
            bottom=Side(style='thin')
        )
        
        for col, header in enumerate(headers, 1):
            cell = ws.cell(row=14, column=col)
            cell.value = header
            cell.font = Font(bold=True)
            cell.fill = PatternFill(start_color='4472C4', end_color='4472C4', fill_type='solid')
            cell.font = Font(color='FFFFFF', bold=True)
            cell.alignment = Alignment(horizontal='center')
            cell.border = thin_border
        
        # 明細データ
        row = 15
        for idx, item in enumerate(quotation.items.all(), 1):
            ws[f'A{row}'] = idx
            ws[f'B{row}'] = item.product_name
            ws[f'C{row}'] = item.specification
            ws[f'D{row}'] = item.quantity
            ws[f'E{row}'] = item.unit_price
            ws[f'F{row}'] = item.subtotal
            
            # 数値書式
            ws[f'D{row}'].number_format = '#,##0'
            ws[f'E{row}'].number_format = '¥#,##0'
            ws[f'F{row}'].number_format = '¥#,##0'
            
            # 罫線
            for col in range(1, 7):
                ws.cell(row=row, column=col).border = thin_border
            
            row += 1
        
        # 合計欄
        ws[f'E{row + 1}'] = '小計'
        ws[f'F{row + 1}'] = quotation.subtotal
        ws[f'F{row + 1}'].number_format = '¥#,##0'
        
        ws[f'E{row + 2}'] = '消費税（10%）'
        ws[f'F{row + 2}'] = quotation.tax_amount
        ws[f'F{row + 2}'].number_format = '¥#,##0'
        
        ws[f'E{row + 3}'] = '合計金額'
        ws[f'F{row + 3}'] = quotation.total_amount
        ws[f'E{row + 3}'].font = Font(size=12, bold=True)
        ws[f'F{row + 3}'].font = Font(size=12, bold=True)
        ws[f'F{row + 3}'].number_format = '¥#,##0'
        
        # 備考
        if quotation.note:
            ws[f'A{row + 5}'] = '【備考】'
            ws[f'A{row + 6}'] = quotation.note
        
        # 列幅調整
        ws.column_dimensions['A'].width = 8
        ws.column_dimensions['B'].width = 25
        ws.column_dimensions['C'].width = 30
        ws.column_dimensions['D'].width = 10
        ws.column_dimensions['E'].width = 15
        ws.column_dimensions['F'].width = 15
        
        return self._create_response(wb, quotation)
    
    def _create_response(self, wb, quotation):
        output = io.BytesIO()
        wb.save(output)
        output.seek(0)
        
        response = HttpResponse(
            output.read(),
            content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
        response['Content-Disposition'] = f'attachment; filename="quotation_{quotation.quotation_number}.xlsx"'
        
        return response
```

---

## まとめ

DjangoでのEXCEL帳票生成は、用途に応じて適切なライブラリを選択することが重要です。

**推奨事項:**

1. **基本的な帳票** - openpyxlで柔軟な作成
2. **高速書込** - xlsxwriterで大量データ処理
3. **データ分析** - pandasで集計レポート
4. **テンプレート活用** - 既存EXCELファイルを編集
5. **非同期処理** - 大量生成はCeleryで非同期化
6. **書式設定** - 条件付き書式、データ検証を活用
7. **エラーハンドリング** - ファイル生成エラーを適切に処理

業務アプリケーションにおいて、EXCEL帳票は非常に重要な機能です。適切な実装で使いやすい帳票システムを構築してください。
