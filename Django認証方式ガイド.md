# Django認証方式ガイド

## 目次
1. [認証の概要](#1-認証の概要)
2. [Django標準認証システム](#2-django標準認証システム)
3. [セッションベース認証](#3-セッションベース認証)
4. [トークンベース認証](#4-トークンベース認証)
5. [JWT認証](#5-jwt認証)
6. [OAuth2.0認証](#6-oauth20認証)
7. [ソーシャル認証](#7-ソーシャル認証)
8. [多要素認証（2FA）](#8-多要素認証2fa)
9. [カスタム認証バックエンド](#9-カスタム認証バックエンド)
10. [認証方式の比較と選択基準](#10-認証方式の比較と選択基準)

---

## 1. 認証の概要

### 1.1 認証と認可の違い

| 項目 | 認証（Authentication） | 認可（Authorization） |
|------|----------------------|---------------------|
| 目的 | ユーザーの身元確認 | アクセス権限の確認 |
| 確認内容 | 「あなたは誰ですか？」 | 「あなたは何ができますか？」 |
| 実現方法 | パスワード、トークン、生体認証等 | パーミッション、ロール、ACL等 |
| タイミング | ログイン時 | リソースアクセス時 |

### 1.2 認証方式の種類

```
┌─────────────────────────────────────────┐
│         Django認証方式                   │
├─────────────────────────────────────────┤
│ 1. セッションベース認証                  │
│    └─ Cookie + Session                 │
│                                         │
│ 2. トークンベース認証                    │
│    ├─ Simple Token                     │
│    ├─ JWT (JSON Web Token)            │
│    └─ Refresh Token                   │
│                                         │
│ 3. OAuth2.0認証                        │
│    ├─ Authorization Code               │
│    ├─ Implicit                         │
│    └─ Client Credentials               │
│                                         │
│ 4. ソーシャル認証                       │
│    ├─ Google                           │
│    ├─ Facebook                         │
│    ├─ Twitter/X                        │
│    └─ LINE                             │
│                                         │
│ 5. 多要素認証（MFA/2FA）                │
│    ├─ TOTP (Time-based OTP)           │
│    ├─ SMS                              │
│    └─ Email                            │
└─────────────────────────────────────────┘
```

#### 1.2.1 セッションベース認証

**概要:**  
サーバー側でユーザーのログイン状態を管理する、最も伝統的な認証方式。ユーザーがログインすると、サーバーはセッションIDを生成し、これをCookieに保存してクライアントに送信する。

**特徴:**
- サーバー側でセッション情報を管理（メモリ、DB、Redis等）
- Cookieを介してセッションIDをやり取り
- Djangoの標準認証方式
- ステートフル（サーバーが状態を保持）

**メリット:**
- セッション情報をサーバー側で完全制御可能
- セッションの強制無効化が容易
- ブラウザベースのWebアプリに最適
- Django標準でサポートされており実装が容易

**デメリット:**
- スケールアウト時にセッション共有が必要
- サーバーリソースを消費
- モバイルアプリやSPAには不向き
- CSRF対策が必須

**適用シーン:**
- 従来型のWebアプリケーション（サーバーサイドレンダリング）
- Django Admin等の管理画面
- セキュリティ要件が高く、セッション管理を厳密に行いたい場合

---

#### 1.2.2 トークンベース認証

**概要:**  
クライアントがトークン（ランダム文字列）を保持し、APIリクエスト時にHTTPヘッダーに含めて送信する認証方式。サーバーはトークンをDBで照合してユーザーを特定する。

**特徴:**
- トークンはDBに保存され、ユーザーと紐付け
- ステートレス（サーバーは各リクエストで認証）
- REST APIに適している
- Django REST frameworkの`TokenAuthentication`で実装

**メリット:**
- モバイルアプリ・SPA等のAPIベース認証に最適
- Cookieを使わないためCORSに強い
- スケールアウトが容易（セッション共有不要）
- トークンの有効期限を管理可能

**デメリット:**
- トークン漏洩のリスク（XSS攻撃等）
- トークンの安全な保存が必要（LocalStorage vs Cookie）
- DBへのトークン照合クエリが発生
- リフレッシュ機構が標準で含まれない

**適用シーン:**
- モバイルアプリ（iOS/Android）のバックエンドAPI
- シングルページアプリケーション（SPA: React, Vue等）
- サードパーティへのAPI提供
- マイクロサービス間の認証

---

#### 1.2.3 JWT（JSON Web Token）認証

**概要:**  
自己完結型のトークン方式。トークン自体にユーザー情報（ペイロード）と署名が含まれており、サーバーはDBを照合せずにトークンの検証のみで認証できる。

**特徴:**
- トークンに情報を埋め込み、署名で改ざんを防止
- ステートレス（DBアクセス不要）
- Access Token（短命）とRefresh Token（長命）の2トークン構成が一般的
- Base64エンコードされた3部構成（Header.Payload.Signature）

**メリット:**
- 完全にステートレス（DBアクセス不要で超高速）
- 水平スケーリングが非常に容易
- マイクロサービス間での認証情報共有が可能
- モバイル/SPA/APIに最適

**デメリット:**
- トークンの強制無効化が困難（ブラックリスト管理が必要）
- ペイロードサイズが大きくなる（ネットワーク負荷）
- トークン漏洩時のリスクが高い（短い有効期限設定が必須）
- Refresh Token管理の複雑さ

**適用シーン:**
- 大規模なAPIサービス（高トラフィック）
- マイクロサービスアーキテクチャ
- シングルサインオン（SSO）
- モバイルアプリ + REST API構成

---

#### 1.2.4 OAuth2.0認証

**概要:**  
サードパーティアプリケーションが、ユーザーのパスワードを知ることなく、リソースへのアクセス権限を取得するための認可フレームワーク。認証ではなく「認可」が主目的。

**特徴:**
- 4つのロール（Resource Owner, Client, Authorization Server, Resource Server）
- 複数のグラントタイプ（Authorization Code, Implicit, Password, Client Credentials）
- Access TokenとRefresh Tokenを使用
- Scopeで権限を細かく制御

**メリット:**
- ユーザーのパスワードをサードパーティに渡さない
- 権限の細かい制御（Scope）
- トークンの取り消しが可能
- 業界標準プロトコル

**デメリット:**
- 仕様が複雑で実装が難しい
- 適切なグラントタイプの選択が必要
- セキュリティ考慮事項が多い（PKCE, State, Nonce等）
- リダイレクトフローの実装が必要

**適用シーン:**
- サードパーティアプリへのAPI公開
- ソーシャルログイン（Google, Facebook等）
- マイクロサービス間の認可
- モバイルアプリの認証・認可

---

#### 1.2.5 ソーシャル認証（Social Authentication）

**概要:**  
Google、Facebook、Twitter、LINE等の既存のソーシャルアカウントを使ってログインする方式。OAuth2.0やOpenID Connectプロトコルを利用。

**特徴:**
- ユーザーは既存アカウントでログイン可能
- パスワード管理が不要
- プロフィール情報の取得が可能
- django-allauthやsocial-auth-app-djangoで実装

**メリット:**
- ユーザー登録の障壁が低い（コンバージョン率向上）
- パスワード管理のリスク軽減
- ユーザーの信頼性向上（ソーシャルアカウント経由）
- プロフィール情報の自動取得

**デメリット:**
- 外部サービスへの依存
- プライバシー懸念（ユーザーが情報提供を嫌う場合）
- 各プロバイダーの仕様変更リスク
- 複数プロバイダー対応の手間

**適用シーン:**
- BtoC Webサービス（EC、SNS、メディア等）
- 新規ユーザー獲得を重視するサービス
- グローバル展開するサービス
- モバイルアプリ（ネイティブSDK利用）

---

#### 1.2.6 多要素認証（MFA / 2FA）

**概要:**  
パスワード（知識要素）に加えて、SMS・TOTPアプリ（所持要素）や生体認証（生体要素）を組み合わせて認証する方式。セキュリティを大幅に向上。

**特徴:**
- 2つ以上の認証要素を組み合わせ
- TOTP（Time-based OTP: Google Authenticator等）が主流
- SMS、Email、生体認証（指紋、顔認証）も利用可能
- django-otp、django-trench等で実装

**メリット:**
- セキュリティレベルの大幅向上
- フィッシング攻撃やパスワード漏洩への耐性
- 規制要件への対応（金融、医療等）
- 不正アクセスの検知・防止

**デメリット:**
- ユーザーの手間増加（UX低下）
- SMS送信コスト
- スマホ紛失時のアカウントロックリスク
- 実装とサポートの複雑さ

**適用シーン:**
- 金融サービス（銀行、証券、決済等）
- 医療・ヘルスケアシステム（HIPAA準拠）
- 企業内管理システム（機密情報アクセス）
- 高セキュリティが必要なサービス全般

---

#### 1.2.7 各認証方式の比較表

| 認証方式 | ステート | DB照合 | スケール | 実装難易度 | セキュリティ | 適用シーン |
|---------|---------|-------|---------|----------|------------|----------|
| **セッション** | ステートフル | 必要 | △ | ★☆☆☆☆ | ★★★☆☆ | 従来型Web |
| **トークン** | ステートレス | 必要 | ○ | ★★☆☆☆ | ★★★☆☆ | API/モバイル |
| **JWT** | ステートレス | 不要 | ◎ | ★★★☆☆ | ★★★★☆ | 大規模API |
| **OAuth2.0** | ステートレス | 必要 | ○ | ★★★★☆ | ★★★★☆ | API公開 |
| **ソーシャル** | ステートレス | 必要 | ○ | ★★★☆☆ | ★★★☆☆ | BtoC |
| **MFA/2FA** | 併用 | 必要 | △ | ★★★★☆ | ★★★★★ | 高セキュリティ |

**凡例:**
- スケール: ◎=非常に容易、○=容易、△=工夫必要
- 実装難易度: ★が多いほど難しい
- セキュリティ: ★が多いほど高い

### 1.3 認証フロー

```
[ユーザー] → [認証リクエスト] → [Django]
                                    ↓
                            [認証バックエンド]
                                    ↓
                            [ユーザー検証]
                                    ↓
                    ┌───────────────┴────────────┐
                    ↓                            ↓
              [認証成功]                    [認証失敗]
                    ↓                            ↓
        [セッション/トークン発行]          [エラーメッセージ]
                    ↓                            ↓
            [認証情報を返却]              [401 Unauthorized]
```

---

## 2. Django標準認証システム

### 2.1 標準認証システムの構成

```python
# settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',  # 認証フレームワーク
    'django.contrib.contenttypes',
    'django.contrib.sessions',  # セッション管理
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

# 認証バックエンド
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',  # デフォルト
]

# 認証URL設定
LOGIN_URL = '/accounts/login/'
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/accounts/login/'
```

### 2.2 Userモデル

```python
# Django標準のUserモデル
from django.contrib.auth.models import User

# フィールド構成
user = User.objects.create_user(
    username='testuser',      # ユーザー名（必須、ユニーク）
    email='test@example.com', # メールアドレス
    password='testpass123',   # パスワード（ハッシュ化される）
    first_name='太郎',        # 名
    last_name='山田',         # 姓
)

# 追加属性
user.is_staff = False      # スタッフステータス
user.is_active = True      # アクティブステータス
user.is_superuser = False  # スーパーユーザーステータス
user.date_joined           # 登録日時
user.last_login            # 最終ログイン日時
```

### 2.3 カスタムUserモデル

```python
# accounts/models.py

from django.contrib.auth.models import AbstractUser, AbstractBaseUser, BaseUserManager
from django.db import models

# AbstractUserを継承（推奨）
class CustomUser(AbstractUser):
    """
    標準Userモデルを拡張
    """
    email = models.EmailField(unique=True, verbose_name='メールアドレス')
    phone_number = models.CharField(max_length=15, blank=True, verbose_name='電話番号')
    birth_date = models.DateField(null=True, blank=True, verbose_name='生年月日')
    profile_image = models.ImageField(
        upload_to='profile_images/',
        blank=True,
        null=True,
        verbose_name='プロフィール画像'
    )
    
    # メールアドレスでログイン
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']
    
    class Meta:
        verbose_name = 'ユーザー'
        verbose_name_plural = 'ユーザー'
    
    def __str__(self):
        return self.email


# AbstractBaseUserを継承（完全カスタム）
class CustomUserManager(BaseUserManager):
    """
    カスタムユーザーマネージャー
    """
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('メールアドレスは必須です')
        
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user
    
    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        extra_fields.setdefault('is_active', True)
        
        return self.create_user(email, password, **extra_fields)


class FullCustomUser(AbstractBaseUser):
    """
    完全カスタムUserモデル
    """
    email = models.EmailField(unique=True, verbose_name='メールアドレス')
    full_name = models.CharField(max_length=100, verbose_name='氏名')
    is_active = models.BooleanField(default=True, verbose_name='アクティブ')
    is_staff = models.BooleanField(default=False, verbose_name='スタッフ')
    is_superuser = models.BooleanField(default=False, verbose_name='管理者')
    date_joined = models.DateTimeField(auto_now_add=True, verbose_name='登録日時')
    
    objects = CustomUserManager()
    
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['full_name']
    
    class Meta:
        verbose_name = 'ユーザー'
        verbose_name_plural = 'ユーザー'
    
    def __str__(self):
        return self.email
    
    def has_perm(self, perm, obj=None):
        return self.is_superuser
    
    def has_module_perms(self, app_label):
        return self.is_superuser
```

```python
# settings.py

# カスタムUserモデルを指定
AUTH_USER_MODEL = 'accounts.CustomUser'
```

---

## 3. セッションベース認証

### 3.1 セッションベース認証の仕組み

```
┌──────────┐                    ┌──────────┐
│ クライアント │                    │ サーバー  │
└──────────┘                    └──────────┘
     │                                │
     │ 1. ログインリクエスト           │
     │ (username + password)         │
     ├───────────────────────────────>│
     │                                │
     │                    2. 認証処理  │
     │                    ユーザー検証 │
     │                                │
     │ 3. セッションID発行            │
     │ (Set-Cookie: sessionid=xxx)   │
     │<───────────────────────────────┤
     │                                │
     │ セッションIDをCookieに保存     │
     │                                │
     │ 4. リクエスト                  │
     │ (Cookie: sessionid=xxx)       │
     ├───────────────────────────────>│
     │                                │
     │                 5. セッション検証│
     │                 DBからユーザー情報取得│
     │                                │
     │ 6. レスポンス                  │
     │<───────────────────────────────┤
     │                                │
```

### 3.2 ログイン・ログアウトの実装

```python
# accounts/views.py

from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.decorators import login_required
from django.shortcuts import render, redirect
from django.contrib import messages
from django.views.decorators.http import require_http_methods
from django.views.decorators.csrf import csrf_protect

@csrf_protect
@require_http_methods(["GET", "POST"])
def user_login(request):
    """
    ログインビュー
    """
    if request.user.is_authenticated:
        return redirect('dashboard')
    
    if request.method == 'POST':
        email = request.POST.get('email')
        password = request.POST.get('password')
        remember_me = request.POST.get('remember_me')
        
        # 認証
        user = authenticate(request, username=email, password=password)
        
        if user is not None:
            if user.is_active:
                # ログイン処理
                login(request, user)
                
                # Remember Me機能
                if not remember_me:
                    # ブラウザを閉じたらセッション削除
                    request.session.set_expiry(0)
                else:
                    # 2週間セッションを保持
                    request.session.set_expiry(1209600)  # 14日間
                
                # ログ記録
                import logging
                logger = logging.getLogger('auth')
                logger.info(
                    f'Login successful: {user.email} from {request.META.get("REMOTE_ADDR")}'
                )
                
                messages.success(request, 'ログインしました。')
                
                # 次のページへリダイレクト
                next_url = request.GET.get('next', 'dashboard')
                return redirect(next_url)
            else:
                messages.error(request, 'このアカウントは無効化されています。')
        else:
            messages.error(request, 'メールアドレスまたはパスワードが正しくありません。')
            
            # ログ記録
            import logging
            logger = logging.getLogger('auth')
            logger.warning(
                f'Login failed: {email} from {request.META.get("REMOTE_ADDR")}'
            )
    
    return render(request, 'accounts/login.html')


@login_required
def user_logout(request):
    """
    ログアウトビュー
    """
    email = request.user.email
    logout(request)
    
    # ログ記録
    import logging
    logger = logging.getLogger('auth')
    logger.info(f'Logout: {email}')
    
    messages.success(request, 'ログアウトしました。')
    return redirect('login')


@login_required
def dashboard(request):
    """
    ダッシュボード（要ログイン）
    """
    return render(request, 'dashboard.html', {
        'user': request.user
    })
```

```html
<!-- templates/accounts/login.html -->

{% extends "base.html" %}

{% block content %}
<div class="login-container">
    <h2>ログイン</h2>
    
    {% if messages %}
        {% for message in messages %}
            <div class="alert alert-{{ message.tags }}">
                {{ message }}
            </div>
        {% endfor %}
    {% endif %}
    
    <form method="post" action="{% url 'login' %}">
        {% csrf_token %}
        
        <div class="form-group">
            <label for="email">メールアドレス</label>
            <input type="email" 
                   id="email" 
                   name="email" 
                   required
                   class="form-control"
                   placeholder="example@example.com">
        </div>
        
        <div class="form-group">
            <label for="password">パスワード</label>
            <input type="password" 
                   id="password" 
                   name="password" 
                   required
                   class="form-control"
                   placeholder="パスワード">
        </div>
        
        <div class="form-group">
            <label>
                <input type="checkbox" name="remember_me" value="1">
                ログイン状態を保持する
            </label>
        </div>
        
        <button type="submit" class="btn btn-primary">ログイン</button>
        
        <div class="links">
            <a href="{% url 'password_reset' %}">パスワードを忘れた場合</a>
            <a href="{% url 'signup' %}">新規登録</a>
        </div>
    </form>
</div>
{% endblock %}
```

### 3.3 ユーザー登録の実装

```python
# accounts/forms.py

from django import forms
from django.contrib.auth import get_user_model
from django.contrib.auth.forms import UserCreationForm

User = get_user_model()

class SignUpForm(UserCreationForm):
    """
    ユーザー登録フォーム
    """
    email = forms.EmailField(
        required=True,
        widget=forms.EmailInput(attrs={
            'class': 'form-control',
            'placeholder': 'メールアドレス'
        })
    )
    
    first_name = forms.CharField(
        required=True,
        max_length=30,
        widget=forms.TextInput(attrs={
            'class': 'form-control',
            'placeholder': '名'
        })
    )
    
    last_name = forms.CharField(
        required=True,
        max_length=30,
        widget=forms.TextInput(attrs={
            'class': 'form-control',
            'placeholder': '姓'
        })
    )
    
    class Meta:
        model = User
        fields = ('email', 'first_name', 'last_name', 'password1', 'password2')
    
    def clean_email(self):
        email = self.cleaned_data.get('email')
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError('このメールアドレスは既に登録されています。')
        return email
    
    def save(self, commit=True):
        user = super().save(commit=False)
        user.email = self.cleaned_data['email']
        user.username = self.cleaned_data['email']  # emailをusernameとして使用
        
        if commit:
            user.save()
        
        return user


# accounts/views.py

from django.views.generic import CreateView
from django.urls import reverse_lazy
from .forms import SignUpForm

class SignUpView(CreateView):
    """
    ユーザー登録ビュー
    """
    form_class = SignUpForm
    template_name = 'accounts/signup.html'
    success_url = reverse_lazy('login')
    
    def form_valid(self, form):
        response = super().form_valid(form)
        
        # ログ記録
        import logging
        logger = logging.getLogger('auth')
        logger.info(f'New user registered: {form.cleaned_data["email"]}')
        
        messages.success(
            self.request,
            '登録が完了しました。ログインしてください。'
        )
        
        return response
```

### 3.4 パスワードリセット

```python
# accounts/urls.py

from django.urls import path
from django.contrib.auth import views as auth_views

urlpatterns = [
    # パスワードリセット申請
    path('password-reset/',
         auth_views.PasswordResetView.as_view(
             template_name='accounts/password_reset.html',
             email_template_name='accounts/password_reset_email.html',
             subject_template_name='accounts/password_reset_subject.txt',
             success_url=reverse_lazy('password_reset_done')
         ),
         name='password_reset'),
    
    # パスワードリセット申請完了
    path('password-reset/done/',
         auth_views.PasswordResetDoneView.as_view(
             template_name='accounts/password_reset_done.html'
         ),
         name='password_reset_done'),
    
    # パスワードリセット確認
    path('reset/<uidb64>/<token>/',
         auth_views.PasswordResetConfirmView.as_view(
             template_name='accounts/password_reset_confirm.html',
             success_url=reverse_lazy('password_reset_complete')
         ),
         name='password_reset_confirm'),
    
    # パスワードリセット完了
    path('reset/done/',
         auth_views.PasswordResetCompleteView.as_view(
             template_name='accounts/password_reset_complete.html'
         ),
         name='password_reset_complete'),
]
```

---

## 4. トークンベース認証

### 4.1 トークンベース認証の仕組み

```
┌──────────┐                    ┌──────────┐
│ クライアント │                    │ サーバー  │
└──────────┘                    └──────────┘
     │                                │
     │ 1. ログインリクエスト           │
     │ (username + password)         │
     ├───────────────────────────────>│
     │                                │
     │                    2. 認証処理  │
     │                    ユーザー検証 │
     │                                │
     │ 3. トークン発行                │
     │ {token: "xxx..."}             │
     │<───────────────────────────────┤
     │                                │
     │ トークンをローカルストレージに保存│
     │                                │
     │ 4. APIリクエスト               │
     │ Authorization: Token xxx      │
     ├───────────────────────────────>│
     │                                │
     │                 5. トークン検証 │
     │                 DBからトークン照合│
     │                                │
     │ 6. レスポンス                  │
     │ {data: ...}                   │
     │<───────────────────────────────┤
     │                                │
     │ 7. ログアウトリクエスト         │
     ├───────────────────────────────>│
     │                                │
     │                 8. トークン削除 │
     │                                │
     │ 9. レスポンス                  │
     │<───────────────────────────────┤
     │                                │
```

### 4.2 Django REST framework Token認証

```bash
# インストール
pip install djangorestframework
```

```python
# settings.py

INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework.authtoken',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

```python
# マイグレーション実行
python manage.py migrate
```

```python
# api/views.py

from rest_framework import status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.response import Response
from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.authtoken.models import Token
from django.contrib.auth import authenticate

@api_view(['POST'])
@permission_classes([AllowAny])
def login_api(request):
    """
    APIログイン（トークン発行）
    """
    email = request.data.get('email')
    password = request.data.get('password')
    
    if not email or not password:
        return Response(
            {'error': 'メールアドレスとパスワードを入力してください'},
            status=status.HTTP_400_BAD_REQUEST
        )
    
    # 認証
    user = authenticate(username=email, password=password)
    
    if user is not None:
        # トークン取得または作成
        token, created = Token.objects.get_or_create(user=user)
        
        return Response({
            'token': token.key,
            'user_id': user.id,
            'email': user.email,
        }, status=status.HTTP_200_OK)
    else:
        return Response(
            {'error': '認証に失敗しました'},
            status=status.HTTP_401_UNAUTHORIZED
        )


@api_view(['POST'])
@permission_classes([IsAuthenticated])
def logout_api(request):
    """
    APIログアウト（トークン削除）
    """
    # トークン削除
    request.user.auth_token.delete()
    
    return Response(
        {'message': 'ログアウトしました'},
        status=status.HTTP_200_OK
    )


@api_view(['GET'])
@permission_classes([IsAuthenticated])
def profile_api(request):
    """
    ユーザープロフィール取得
    """
    user = request.user
    
    return Response({
        'id': user.id,
        'email': user.email,
        'first_name': user.first_name,
        'last_name': user.last_name,
    }, status=status.HTTP_200_OK)
```

```python
# クライアント側のリクエスト例

import requests

# ログイン
response = requests.post('http://localhost:8000/api/login/', json={
    'email': 'test@example.com',
    'password': 'testpass123'
})

data = response.json()
token = data['token']

# 認証が必要なAPIへのアクセス
headers = {
    'Authorization': f'Token {token}'
}

response = requests.get('http://localhost:8000/api/profile/', headers=headers)
profile = response.json()
print(profile)
```

---

## 5. JWT認証

### 5.1 JWT認証の仕組み

```
┌──────────┐                    ┌──────────┐
│ クライアント │                    │ サーバー  │
└──────────┘                    └──────────┘
     │                                │
     │ 1. ログインリクエスト           │
     │ (username + password)         │
     ├───────────────────────────────>│
     │                                │
     │                    2. 認証処理  │
     │                    ユーザー検証 │
     │                                │
     │ 3. JWT発行                     │
     │ {                              │
     │   access: "eyJ...",            │
     │   refresh: "eyJ..."            │
     │ }                              │
     │<───────────────────────────────┤
     │                                │
     │ JWTをローカルストレージに保存   │
     │                                │
     │ 4. APIリクエスト               │
     │ Authorization: Bearer eyJ...  │
     ├───────────────────────────────>│
     │                                │
     │                 5. JWT検証      │
     │                 署名検証・有効期限確認│
     │                 （DB不要）      │
     │                                │
     │ 6. レスポンス                  │
     │<───────────────────────────────┤
     │                                │
     │ --- Access Token期限切れ ---   │
     │                                │
     │ 7. Refresh Tokenでリクエスト   │
     │ {refresh: "eyJ..."}           │
     ├───────────────────────────────>│
     │                                │
     │                 8. Refresh検証 │
     │                                │
     │ 9. 新しいAccess Token発行      │
     │ {access: "eyJ..."}            │
     │<───────────────────────────────┤
     │                                │
```

### 5.2 JWT（JSON Web Token）の構造

```
JWT構造:
┌─────────────────────────────────────────┐
│ Header.Payload.Signature                │
└─────────────────────────────────────────┘

Header (ヘッダー):
{
  "alg": "HS256",
  "typ": "JWT"
}

Payload (ペイロード):
{
  "user_id": 123,
  "email": "user@example.com",
  "exp": 1735000000,  // 有効期限
  "iat": 1734900000   // 発行日時
}

Signature (署名):
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret_key
)
```

### 5.2 djangorestframework-simplejwt実装

```bash
# インストール
pip install djangorestframework-simplejwt
```

```python
# settings.py

from datetime import timedelta

INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# SimpleJWT設定
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),      # アクセストークン有効期限
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),         # リフレッシュトークン有効期限
    'ROTATE_REFRESH_TOKENS': True,                       # リフレッシュトークンをローテーション
    'BLACKLIST_AFTER_ROTATION': True,                    # 古いトークンをブラックリスト化
    'UPDATE_LAST_LOGIN': True,                           # 最終ログイン日時を更新
    
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'VERIFYING_KEY': None,
    'AUDIENCE': None,
    'ISSUER': None,
    
    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
    
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
    'TOKEN_TYPE_CLAIM': 'token_type',
    
    'JTI_CLAIM': 'jti',
    
    'SLIDING_TOKEN_REFRESH_EXP_CLAIM': 'refresh_exp',
    'SLIDING_TOKEN_LIFETIME': timedelta(minutes=5),
    'SLIDING_TOKEN_REFRESH_LIFETIME': timedelta(days=1),
}
```

```python
# api/urls.py

from django.urls import path
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    # トークン取得
    path('token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    
    # トークンリフレッシュ
    path('token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    
    # トークン検証
    path('token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

```python
# カスタムトークンビュー

from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    """
    カスタムJWTトークンシリアライザー
    """
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # カスタムクレームを追加
        token['email'] = user.email
        token['full_name'] = f'{user.first_name} {user.last_name}'
        token['is_staff'] = user.is_staff
        
        return token
    
    def validate(self, attrs):
        data = super().validate(attrs)
        
        # レスポンスにユーザー情報を追加
        data['user'] = {
            'id': self.user.id,
            'email': self.user.email,
            'first_name': self.user.first_name,
            'last_name': self.user.last_name,
        }
        
        return data


class CustomTokenObtainPairView(TokenObtainPairView):
    """
    カスタムJWTトークン取得ビュー
    """
    serializer_class = CustomTokenObtainPairSerializer
```

```python
# クライアント側のリクエスト例

import requests

# ログイン（トークン取得）
response = requests.post('http://localhost:8000/api/token/', json={
    'email': 'test@example.com',
    'password': 'testpass123'
})

data = response.json()
access_token = data['access']
refresh_token = data['refresh']

# 認証が必要なAPIへのアクセス
headers = {
    'Authorization': f'Bearer {access_token}'
}

response = requests.get('http://localhost:8000/api/profile/', headers=headers)
profile = response.json()

# トークンリフレッシュ
response = requests.post('http://localhost:8000/api/token/refresh/', json={
    'refresh': refresh_token
})

new_data = response.json()
new_access_token = new_data['access']
```

---

## 6. OAuth2.0認証

### 6.1 OAuth2.0認証の仕組み

```
Authorization Code Flow (認可コードフロー):

┌─────────┐         ┌──────────┐        ┌──────────┐       ┌──────────┐
│ユーザー  │         │クライアント│        │認可サーバー│       │リソース  │
│         │         │アプリ     │        │          │       │サーバー   │
└─────────┘         └──────────┘        └──────────┘       └──────────┘
     │                    │                   │                  │
     │ 1. アクセス要求     │                   │                  │
     ├───────────────────>│                   │                  │
     │                    │                   │                  │
     │                    │ 2. 認可リクエスト  │                  │
     │                    │ (client_id,       │                  │
     │                    │  redirect_uri)    │                  │
     │                    ├──────────────────>│                  │
     │                    │                   │                  │
     │                    │ 3. ログイン画面    │                  │
     │                    │    表示           │                  │
     │<───────────────────┴───────────────────┤                  │
     │                                        │                  │
     │ 4. ユーザー認証                        │                  │
     │    (username + password)              │                  │
     ├───────────────────────────────────────>│                  │
     │                                        │                  │
     │ 5. 認可確認画面                        │                  │
     │<───────────────────────────────────────┤                  │
     │                                        │                  │
     │ 6. 権限承認                            │                  │
     ├───────────────────────────────────────>│                  │
     │                                        │                  │
     │                    │ 7. 認可コード      │                  │
     │                    │    返却           │                  │
     │                    │<──────────────────┤                  │
     │                    │                   │                  │
     │                    │ 8. トークンリクエスト│                │
     │                    │ (code,            │                  │
     │                    │  client_secret)   │                  │
     │                    ├──────────────────>│                  │
     │                    │                   │                  │
     │                    │ 9. アクセストークン│                  │
     │                    │    発行           │                  │
     │                    │<──────────────────┤                  │
     │                    │                   │                  │
     │                    │ 10. APIリクエスト  │                  │
     │                    │ (access_token)    │                  │
     │                    ├───────────────────────────────────>│
     │                    │                   │                  │
     │                    │ 11. ユーザー情報   │                  │
     │                    │<───────────────────────────────────┤
     │                    │                   │                  │
     │ 12. サービス提供    │                   │                  │
     │<───────────────────┤                   │                  │
     │                    │                   │                  │
```

### 6.2 django-oauth-toolkit実装

```bash
# インストール
pip install django-oauth-toolkit
```

```python
# settings.py

INSTALLED_APPS = [
    # ...
    'oauth2_provider',
]

# OAuth2設定
OAUTH2_PROVIDER = {
    'SCOPES': {
        'read': 'Read scope',
        'write': 'Write scope',
    },
    'ACCESS_TOKEN_EXPIRE_SECONDS': 3600,  # 1時間
    'REFRESH_TOKEN_EXPIRE_SECONDS': 2592000,  # 30日
    'AUTHORIZATION_CODE_EXPIRE_SECONDS': 600,  # 10分
}
```

```python
# urls.py

from django.urls import path, include

urlpatterns = [
    # ...
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
]
```

```python
# api/views.py

from oauth2_provider.decorators import protected_resource
from django.http import JsonResponse

@protected_resource(scopes=['read'])
def api_protected_view(request):
    """
    OAuth2で保護されたAPIエンドポイント
    """
    user = request.resource_owner
    
    return JsonResponse({
        'id': user.id,
        'email': user.email,
        'message': 'This is protected data'
    })
```

---

## 7. ソーシャル認証

### 7.1 ソーシャル認証の仕組み

```
Google OAuth2認証フロー例:

┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │Djangoアプリ│      │Google     │      │Djangoアプリ│
│         │      │          │      │認証サーバー │      │          │
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ 1. ログイン画面  │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │ 2. Googleで     │                 │                 │
     │    ログイン選択  │                 │                 │
     ├────────────────>│                 │                 │
     │                 │                 │                 │
     │                 │ 3. Google認証   │                 │
     │                 │    ページへ     │                 │
     │                 │    リダイレクト │                 │
     │<────────────────┴────────────────>│                 │
     │                                   │                 │
     │ 4. Googleログイン                 │                 │
     │    (Gmail + Password)            │                 │
     ├──────────────────────────────────>│                 │
     │                                   │                 │
     │ 5. 権限承認画面                   │                 │
     │<──────────────────────────────────┤                 │
     │                                   │                 │
     │ 6. 権限を承認                     │                 │
     ├──────────────────────────────────>│                 │
     │                                   │                 │
     │                 │ 7. 認可コード   │                 │
     │                 │    コールバック │                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │                 │ 8. トークン交換 │                 │
     │                 │    リクエスト   │                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │ 9. アクセストークン│               │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │                 │ 10. ユーザー情報取得│              │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │ 11. プロフィール情報│              │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │                 │ 12. ユーザー作成/更新│            │
     │                 │    セッション確立 │               │
     │                 ├─────────────────────────────────>│
     │                 │                 │                 │
     │ 13. ログイン完了 │                 │                 │
     │    ダッシュボード表示│              │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
```

### 7.2 django-allauth実装

```bash
# インストール
pip install django-allauth
```

```python
# settings.py

INSTALLED_APPS = [
    # ...
    'django.contrib.sites',
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    # ソーシャルプロバイダー
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount.providers.facebook',
    'allauth.socialaccount.providers.twitter',
    'allauth.socialaccount.providers.line',
]

SITE_ID = 1

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]

# allauth設定
ACCOUNT_AUTHENTICATION_METHOD = 'email'
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_EMAIL_VERIFICATION = 'mandatory'
LOGIN_REDIRECT_URL = '/'
ACCOUNT_LOGOUT_REDIRECT_URL = '/accounts/login/'

# ソーシャル認証設定
SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'SCOPE': [
            'profile',
            'email',
        ],
        'AUTH_PARAMS': {
            'access_type': 'online',
        },
        'APP': {
            'client_id': 'YOUR_GOOGLE_CLIENT_ID',
            'secret': 'YOUR_GOOGLE_SECRET',
            'key': ''
        }
    },
    'facebook': {
        'METHOD': 'oauth2',
        'SCOPE': ['email', 'public_profile'],
        'AUTH_PARAMS': {'auth_type': 'reauthenticate'},
        'INIT_PARAMS': {'cookie': True},
        'FIELDS': [
            'id',
            'email',
            'name',
            'first_name',
            'last_name',
        ],
        'EXCHANGE_TOKEN': True,
        'VERIFIED_EMAIL': False,
        'VERSION': 'v12.0',
        'APP': {
            'client_id': 'YOUR_FACEBOOK_APP_ID',
            'secret': 'YOUR_FACEBOOK_SECRET',
            'key': ''
        }
    },
}
```

```python
# urls.py

urlpatterns = [
    # ...
    path('accounts/', include('allauth.urls')),
]
```

```html
<!-- templates/accounts/login.html -->

{% load socialaccount %}

<div class="social-login">
    <h3>ソーシャルログイン</h3>
    
    <!-- Google -->
    <a href="{% provider_login_url 'google' %}" class="btn btn-google">
        Googleでログイン
    </a>
    
    <!-- Facebook -->
    <a href="{% provider_login_url 'facebook' %}" class="btn btn-facebook">
        Facebookでログイン
    </a>
    
    <!-- Twitter -->
    <a href="{% provider_login_url 'twitter' %}" class="btn btn-twitter">
        Twitterでログイン
    </a>
    
    <!-- LINE -->
    <a href="{% provider_login_url 'line' %}" class="btn btn-line">
        LINEでログイン
    </a>
</div>
```

---

## 8. 多要素認証（2FA）

### 8.1 多要素認証（2FA）の仕組み

```
TOTP (Time-based One-Time Password) フロー:

┌─────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ユーザー  │      │ブラウザ   │      │Djangoサーバー│  │認証アプリ │
│         │      │          │      │          │      │(Google   │
│         │      │          │      │          │      │Authenticator)│
└─────────┘      └──────────┘      └──────────┘      └──────────┘
     │                 │                 │                 │
     │ [初回設定]       │                 │                 │
     │                 │                 │                 │
     │ 1. 2FA設定リクエスト│               │                 │
     ├────────────────>│                 │                 │
     │                 │ 2. 設定ページリクエスト│           │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 3. Secret Key生成│
     │                 │                 │    QRコード作成 │
     │                 │                 │                 │
     │                 │ 4. QRコード表示  │                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 5. QRコードをスキャン│              │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │                 │                 │ 6. Secret Key保存│
     │                 │                 │    30秒毎にOTP生成│
     │                 │                 │                 │
     │ 7. OTP確認       │                 │                 │
     │<─────────────────────────────────────────────────┤
     │                 │                 │                 │
     │ 8. OTP入力       │                 │                 │
     ├────────────────>│                 │                 │
     │                 │ 9. OTP送信       │                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 10. OTP検証     │
     │                 │                 │     (時刻ベース) │
     │                 │                 │                 │
     │                 │ 11. 検証結果     │                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 12. 2FA有効化完了│                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
     │                 │                 │                 │
     │ [ログイン時]     │                 │                 │
     │                 │                 │                 │
     │ 13. ログイン     │                 │                 │
     │    (ID + Pass)  │                 │                 │
     ├────────────────>│                 │                 │
     │                 │ 14. 認証リクエスト│                │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 15. 1段階目認証OK│
     │                 │                 │                 │
     │                 │ 16. OTP入力画面  │                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 17. 認証アプリ確認│                 │                 │
     ├─────────────────────────────────────────────────>│
     │                 │                 │                 │
     │ 18. OTP取得      │                 │                 │
     │<─────────────────────────────────────────────────┤
     │    (例: 123456) │                 │                 │
     │                 │                 │                 │
     │ 19. OTP入力      │                 │                 │
     ├────────────────>│                 │                 │
     │                 │ 20. OTP送信      │                 │
     │                 ├────────────────>│                 │
     │                 │                 │                 │
     │                 │                 │ 21. 2段階目認証OK│
     │                 │                 │     セッション確立│
     │                 │                 │                 │
     │                 │ 22. ログイン完了 │                 │
     │                 │<────────────────┤                 │
     │                 │                 │                 │
     │ 23. ダッシュボード│                 │                 │
     │<────────────────┤                 │                 │
     │                 │                 │                 │
```

### 8.2 TOTP（Time-based One-Time Password）実装

```bash
# インストール
pip install django-otp qrcode pillow
```

```python
# settings.py

INSTALLED_APPS = [
    # ...
    'django_otp',
    'django_otp.plugins.otp_totp',
    'django_otp.plugins.otp_static',
]

MIDDLEWARE = [
    # ...
    'django_otp.middleware.OTPMiddleware',
]
```

```python
# accounts/views.py

from django_otp.plugins.otp_totp.models import TOTPDevice
from django_otp.util import random_hex
from django.contrib.auth.decorators import login_required
import qrcode
from io import BytesIO
import base64

@login_required
def setup_2fa(request):
    """
    2段階認証のセットアップ
    """
    user = request.user
    
    # 既存のデバイスを確認
    device = TOTPDevice.objects.filter(user=user, confirmed=False).first()
    
    if not device:
        # 新しいデバイスを作成
        device = TOTPDevice.objects.create(
            user=user,
            name='default',
            confirmed=False
        )
    
    # QRコードの生成
    url = device.config_url
    
    qr = qrcode.QRCode(version=1, box_size=10, border=5)
    qr.add_data(url)
    qr.make(fit=True)
    img = qr.make_image(fill_color="black", back_color="white")
    
    buffer = BytesIO()
    img.save(buffer, format='PNG')
    qr_code_image = base64.b64encode(buffer.getvalue()).decode()
    
    if request.method == 'POST':
        token = request.POST.get('token')
        
        # トークン検証
        if device.verify_token(token):
            device.confirmed = True
            device.save()
            messages.success(request, '2段階認証が有効化されました。')
            return redirect('dashboard')
        else:
            messages.error(request, '認証コードが正しくありません。')
    
    return render(request, 'accounts/setup_2fa.html', {
        'qr_code': qr_code_image,
        'secret_key': device.key,
    })


@login_required
def verify_2fa(request):
    """
    2段階認証の検証
    """
    if request.method == 'POST':
        token = request.POST.get('token')
        
        # ユーザーのデバイスを取得
        device = TOTPDevice.objects.filter(
            user=request.user,
            confirmed=True
        ).first()
        
        if device and device.verify_token(token):
            # 認証成功
            request.session['otp_verified'] = True
            messages.success(request, '2段階認証に成功しました。')
            return redirect('dashboard')
        else:
            messages.error(request, '認証コードが正しくありません。')
    
    return render(request, 'accounts/verify_2fa.html')


@login_required
def disable_2fa(request):
    """
    2段階認証の無効化
    """
    if request.method == 'POST':
        # ユーザーの全デバイスを削除
        TOTPDevice.objects.filter(user=request.user).delete()
        
        messages.success(request, '2段階認証を無効化しました。')
        return redirect('dashboard')
    
    return render(request, 'accounts/disable_2fa.html')
```

```python
# カスタムデコレーター

from functools import wraps
from django.shortcuts import redirect
from django.contrib import messages

def otp_required(view_func):
    """
    2段階認証が必要なビューのデコレーター
    """
    @wraps(view_func)
    def wrapper(request, *args, **kwargs):
        if not request.session.get('otp_verified', False):
            messages.warning(request, '2段階認証が必要です。')
            return redirect('verify_2fa')
        return view_func(request, *args, **kwargs)
    return wrapper


# 使用例
@login_required
@otp_required
def sensitive_view(request):
    """
    2段階認証が必要な機密情報を扱うビュー
    """
    return render(request, 'sensitive_data.html')
```

---

## 9. カスタム認証バックエンド

### 9.1 カスタム認証バックエンドの仕組み

```
カスタム認証バックエンドのフロー:

┌──────────┐                    ┌──────────┐
│ クライアント │                    │ サーバー  │
└──────────┘                    └──────────┘
     │                                │
     │ 1. ログインリクエスト           │
     │ (email/username/phone          │
     │  + password)                  │
     ├───────────────────────────────>│
     │                                │
     │                 2. authenticate()呼び出し│
     │                                │
     │                 ┌──────────────┴────────┐
     │                 │ 認証バックエンド順次実行│
     │                 │                        │
     │                 │ [1] EmailOrUsernameBackend│
     │                 │  ├─ emailで検索        │
     │                 │  ├─ usernameで検索     │
     │                 │  └─ パスワード検証     │
     │                 │       ↓                │
     │                 │    成功 or 失敗        │
     │                 │       ↓                │
     │                 │  失敗の場合、次へ      │
     │                 │                        │
     │                 │ [2] PhoneNumberBackend │
     │                 │  ├─ 電話番号で検索     │
     │                 │  └─ パスワード検証     │
     │                 │       ↓                │
     │                 │    成功 or 失敗        │
     │                 │       ↓                │
     │                 │  失敗の場合、次へ      │
     │                 │                        │
     │                 │ [3] APIKeyBackend      │
     │                 │  ├─ APIキー検索        │
     │                 │  └─ 有効性検証         │
     │                 │       ↓                │
     │                 │    成功 or 失敗        │
     │                 │       ↓                │
     │                 │  失敗の場合、次へ      │
     │                 │                        │
     │                 │ [4] ModelBackend       │
     │                 │  └─ デフォルト認証     │
     │                 │                        │
     │                 └────────────┬───────────┘
     │                              │
     │                 3. いずれかで認証成功  │
     │                    またはすべて失敗   │
     │                              │
     │              ┌───────────────┴────────────┐
     │              ↓                            ↓
     │        [認証成功]                    [認証失敗]
     │              ↓                            ↓
     │    4. セッション確立              5. エラーメッセージ│
     │        ユーザーオブジェクト返却      None返却       │
     │              │                            │
     │ 6. ログイン完了                   7. ログイン失敗    │
     │<─────────────┤                  ┌─────────┘
     │              │                  │
     │              │                  │
     │<─────────────┴──────────────────┘
     │                                │
```

### 9.2 カスタム認証バックエンドの実装

```python
# accounts/backends.py

from django.contrib.auth.backends import ModelBackend
from django.contrib.auth import get_user_model
from django.db.models import Q

User = get_user_model()

class EmailOrUsernameBackend(ModelBackend):
    """
    メールアドレスまたはユーザー名での認証
    """
    def authenticate(self, request, username=None, password=None, **kwargs):
        try:
            # メールアドレスまたはユーザー名で検索
            user = User.objects.get(
                Q(username=username) | Q(email=username)
            )
        except User.DoesNotExist:
            return None
        
        # パスワード検証
        if user.check_password(password) and self.user_can_authenticate(user):
            return user
        
        return None


class PhoneNumberBackend(ModelBackend):
    """
    電話番号での認証
    """
    def authenticate(self, request, phone_number=None, password=None, **kwargs):
        try:
            user = User.objects.get(phone_number=phone_number)
        except User.DoesNotExist:
            return None
        
        if user.check_password(password) and self.user_can_authenticate(user):
            return user
        
        return None


class APIKeyBackend:
    """
    APIキーでの認証
    """
    def authenticate(self, request, api_key=None, **kwargs):
        if not api_key:
            return None
        
        try:
            # APIキーモデルを検索（別途実装が必要）
            from .models import APIKey
            api_key_obj = APIKey.objects.select_related('user').get(
                key=api_key,
                is_active=True
            )
            return api_key_obj.user
        except APIKey.DoesNotExist:
            return None
    
    def get_user(self, user_id):
        try:
            return User.objects.get(pk=user_id)
        except User.DoesNotExist:
            return None
```

```python
# settings.py

AUTHENTICATION_BACKENDS = [
    'accounts.backends.EmailOrUsernameBackend',
    'accounts.backends.PhoneNumberBackend',
    'accounts.backends.APIKeyBackend',
    'django.contrib.auth.backends.ModelBackend',  # デフォルト
]
```

---

## 10. 認証方式の比較と選択基準

### 10.1 認証方式の比較表

| 認証方式 | メリット | デメリット | 適用場面 |
|---------|---------|-----------|---------|
| セッションベース | ・標準で実装済み<br>・サーバー側で完全制御<br>・セキュア | ・スケーラビリティに課題<br>・CSRF対策が必要<br>・モバイルアプリに不向き | Webアプリケーション |
| トークンベース | ・ステートレス<br>・モバイル対応<br>・スケーラブル | ・トークン管理が必要<br>・有効期限管理 | API、モバイルアプリ |
| JWT | ・自己完結型<br>・スケーラブル<br>・多様なクライアント対応 | ・トークンサイズが大きい<br>・無効化が困難<br>・機密情報を含められない | SPA、マイクロサービス |
| OAuth2.0 | ・標準プロトコル<br>・スコープ管理<br>・サードパーティ連携 | ・実装が複雑<br>・設定が多い | API提供、外部連携 |
| ソーシャル認証 | ・ユーザー登録が簡単<br>・パスワード管理不要 | ・外部サービス依存<br>・プライバシー懸念 | BtoC、SNS連携 |
| 多要素認証 | ・セキュリティ向上<br>・不正アクセス防止 | ・ユーザー体験の低下<br>・実装コスト | 重要システム、管理画面 |

### 10.2 選択基準

```
システムタイプ別推奨認証方式:

1. 管理画面・社内システム
   └─ セッションベース + 2FA

2. BtoC Webアプリケーション
   └─ セッションベース + ソーシャル認証

3. モバイルアプリ
   └─ JWT + リフレッシュトークン

4. REST API
   └─ JWT または トークンベース

5. マイクロサービス
   └─ JWT + OAuth2.0

6. 外部API提供
   └─ OAuth2.0

7. 高セキュリティシステム
   └─ セッションベース + 2FA + IP制限
```

### 10.3 実装チェックリスト

```markdown
□ 認証方式の選定
  □ システム要件の確認
  □ セキュリティ要件の確認
  □ スケーラビリティの確認

□ セキュリティ対策
  □ HTTPS通信の強制
  □ CSRF対策の実装
  □ パスワードハッシュ化
  □ セッション/トークンの有効期限設定
  □ ブルートフォース攻撃対策
  □ レート制限の実装

□ ユーザー体験
  □ ログイン画面のUI/UX
  □ エラーメッセージの適切な表示
  □ パスワードリセット機能
  □ Remember Me機能
  □ ソーシャルログインの提供

□ 運用・監視
  □ ログイン試行のログ記録
  □ 異常なアクセスの検知
  □ セッション管理の監視
  □ トークンの定期的なローテーション

□ テスト
  □ 正常系のテスト
  □ 異常系のテスト
  □ セキュリティテスト
  □ 負荷テスト
```

---

## まとめ

Django認証システムは柔軟で強力な機能を提供しています。プロジェクトの要件に応じて適切な認証方式を選択し、セキュリティとユーザビリティのバランスを取ることが重要です。

**重要ポイント:**
1. 標準認証システムをベースにカスタマイズ
2. セキュリティを最優先に設計
3. ユーザー体験を損なわない認証フロー
4. スケーラビリティを考慮した実装
5. 定期的なセキュリティレビューと更新
