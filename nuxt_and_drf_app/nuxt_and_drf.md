# nuxt + django でWebアプリを作ってみた

nuxt+django で　手書きの数字を判定するwebアプリを作ってみました。
  

## 環境構築
mac 10.15.6　を使用しています
### バックエンド
使用したライブラリは,データの管理に  
- django
- django-rest-framework
- django-cors-headers
  
画像の処理と判定に　　
- scikit-learn
- Pillow
  
を用いた  
プロジェクトファイルの仮想環境にインストールしておく

```Bash:
$ django-admin startproject core
$ cd core
$ python3 manage.py startapp app
```
でプロジェクトファイルを作ったのち  
#### core/settings.py

```python:
import os #追加
...

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework', # 追加
    'corsheaders', # 追加
    'app' # 追加
  ] 

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware', # 追加
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

 # MIDDLEWARE　の直下で追加
CORS_ORIGIN_WHITELIST = (
    'http://localhost:3000',
)

...
...

# STATIC_URL の直下で追加
MEDIA_URL = '/media/' # 追加
MEDIA_ROOT = os.path.join(BASE_DIR, 'media') # 追加

```

これで　appの登録とcors-headerの設定をしました。また、画像を使うため、画像ファイルの保存先を指定しました。  
[CORS_ORIGIN_WHITELIST]にフロントエンド側のサーバーを設定することでクロスドメインによるエラーを解消できます。

### フロントエンド
vue-cilを利用してプロジェクトファイルを作ります  
まず、node.jsをインストールしてから、一緒にインストールされたnpmを利用してvue-cilをインストールします
```Bash:
$npm install -g @vue/cil
```
そして、プロジェクトファイルに移動してからNuxtアプリを作ります
```Bash:
$ npx create-nuxt-app image_app
```
途中で出てくる質問には、axiosを使うこと、npmを使うこと、ESLimtを使うこと、また今回はbootstrap Vueを使ったのでそれを使うこと、に注意して選択しました。

## アプリの作成

### バックエンド


**modelの作成**

#### core/app/models.py
```python:
from django.db import models

# Create your models here.

class ImageModel(models.Model):
    name = models.CharField(max_length=128)
    image = models.ImageField(upload_to="media")
    predict = models.IntegerField(blank=True,default=0)
```

djangoではこのmodelsで作った物がデータベースのテーブルに対応づけられます。今回必要な情報は、名前(name)と受け取った写真(image)と予測した結果の値(predict)なので、その三つをfieldに設定します。

**シリアライザーの作成**  
core/serializers.py
```python:
from rest_framework import serializers
from .models import ImageModel


class ImageSerializer(serializers.ModelSerializer):
    class Meta:
        model = ImageModel
        fields = ("id", "name", "image", "predict")
        read_only_fields = ('predict',"id")

```
シリアライザーでは、データの送受信に的するようにmodelインスタンスをjson形式にするための設定をします。  
rest-frameworkにあるserializers.ModelSerializerを継承することで、作成できます。受け取るデータはnameとimageだけなので、idとpredictはread_only_fieldsに設定しておきます。

**viewの作成**  
view.pyを書くまえに、文字を判定するためのファイルを用意しておきます。  
appの下にprocessというディレクトリを用意し、その中にmodelを保存しておくconfigファイル,実際に判定する関数を定義するpredict.pyを作ります。  
今回使用したmodelは「いちばんやさしいPython機械学習の教本 人気講師が教える業務で役立つ実践ノウハウ
いちばんやさしいPython機械学習の教本 」(インプレス)の中にあったものをそのまま使いましたので、それについては省略します。作成はscikit-learnを用いました。作成したモデルのpickleファイル（trained-model.pickle）をconfigファイルにあらかじめ保存しておきます。  
#### app/process/predict.py
```python3:
import numpy
from PIL import Image, ImageEnhance, ImageOps
import  pickle


def predict(img):
    with open("./app/process/config/trained-model.pickle", 'rb') as f:
        clf = pickle.load(f)
    im = Image.open(img)
    im_enhanced = ImageEnhance.Brightness(im).enhance(2)
    im_gray = im_enhanced.convert(mode='L')
    im_8x8 = im_gray.resize((8,8))
    im_inverted = ImageOps.invert(im_8x8)
    X_im2d = numpy.asarray(im_inverted)
    X_im1d = X_im2d.reshape(-1)
    X_multiplied = X_im1d * (16 / 255)
    res = clf.predict([X_multiplied])[0]
    return res

```
画像を受け取り、pillowで画像を前処理してからロードしたmodelを使って予測し、その文字を返します。

#### app/view.py

```python3:
from rest_framework import viewsets
from rest_framework.response import Response
from rest_framework.decorators import action
from rest_framework import status
from .models import ImageModel
from .serializers import ImageSerializer
from .process.predict import predict


class ImageViewSet(viewsets.ModelViewSet):
    queryset = ImageModel.objects.all()
    serializer_class = ImageSerializer

    def create(self,request):
        serializer = self.serializer_class(data = request.data)
        serializer.is_valid(raise_exception=True)
        name = request.data["name"]
        img = request.data["image"]
        res = predict(img)
        item = ImageModel(name=name, image=img, predict=res)
        item.save()
        return Response([res,name], status=status.HTTP_200_OK)
```
django-frame-workの汎用クラスを継承して使いました。viewsets.ModelViewSetは作成したモデルに基づいてそれぞれのrequest(CRUD)を処理してくれます。今回はPOSTされた際に、imageの数字を予測した値を保存したいのでcreateをオーバーライドします。先ほどのpredictを用いて予測した値をデータベースに保存するようにています。

**URLの設定**
#### core/urls.py
```python3:
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static
from app.urls import router as router
urlpatterns = [
    path('admin/', admin.site.urls),
    path("core/", include(router.urls))
]
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL,document_root=settings.MEDIA_ROOT)
```
次にappの中のurlを作成します
#### app/urls.py
```python3:
from rest_framework import routers
from .views import ImageViewSet

router = routers.DefaultRouter()
router.register(r"^image", ImageViewSet)
```
全体のurlはcore/urls.pyに、それぞれのアプリについてのurlはそのアプリ内のulrs.pyに設定していくようです。これで、"ローカルサーバー/core/image"にviewが設定されました。

**管理画面の編集**  
最後にdjangoの管理画面の設定をします。
#### app/admin.py
from django.contrib import admin
from .models import ImageModel

class ImageModelAdmin(admin.ModelAdmin):
    fields = ['name', 'image']

admin.site.register(ImageModel, ImageModelAdmin)


**マイグレーションとスーパーユーザーの設定**  
これまでの設定をデータベースに反映させるためマイグレーションをします。
```Bash:
$ python3 manage.py makemigrations
$ python3 manage.py migrate
```
また、管理画面を操作するために、スーパーユーザーを設定します。
```Bash:
$ python3 manage.py createsuperuser
```
  
プロジェクトファイルは次のようになりました。
![](スクリーンショット%202020-08-21%2017.40.09.png)

### フロントエンド
nuxtではpageファイルのディレクトリ構造がそのままurlの構造になるため、それぞれのファイルのindex.vueを編集することで作成できます。今回は  
- result
- judge
  
のページを作ります。役割はjudgeで判定して、その結果をresultに加えていくという形です。resultに結果をそれぞれ並べるため、それぞれの結果をもつresult_card.vueというコンポーネント をまず作成します。
(デザインについてはほとんどnMoMo'sさんのを参考にさせてもらったので説明はしないです)
#### components/result_card.vue  
```javascript:
<template>
    <div class="resultcard">
        <img :src="result.image" class="card-img-top">
        <div class="card-body">
            <h5>
                判定結果は =>{{ result.predict}}
            </h5>
            <div class='action-buttons'>
                <button @click="onDelete(result.id)" class = 'action-buttons'>
                    削除
                </button>
            </div>
        </div>
    </div>
</template>

<script>
export default {
  props: ['result', 'onDelete']
}
</script>

<style>
.resultcard {
    box-shadow: 0 1rem 1.5rem rgba(0,0,0,.6);
}
</style>
```
それぞれの結果であるresultとそのデータを消す関数onDeleteを受け取ります。  
  
次に結果一覧であるresult画面を作成します。
#### page/results/index.vue
```javascript:
/* eslint-disable */
<template>
    <main class="container mt-5">
        <div class="row">
            <div class="col-12 text-right mb-4">
                <div class="d-flex justify-content-between" >
                    <h3>判定した文字一覧</h3>
                    <nuxt-link to="/judge" class="btn btn-info">
                    判定する</nuxt-link>
                </div>
            </div>
            <template v-for="result in results">
                <div :key="result.id" class="col-lg-3 col-md-4 col-sm 6 mb-4">
                    <ResultCard :onDelete = "deleteResult" :result="result" ></ResultCard>
                </div>
            </template>
        </div>
    </main>
</template>

<script>
import ResultCard from '@/components/result_card'

export default {
  head () {
    return {
      title: 'result list'
    }
  },
  components: {
    ResultCard
  },
  data () {
    return {
      results: []
    }
  },
  async asyncData( { $axios, params } ){ // eslint-disable-line
    try {
      const results = await $axios.$get('/image/')
      return { results }
    } catch (e) {
      return { results: [] }
    }
  },
  methods: {
        async deleteResult (result_id) { // eslint-disable-line
      try {
        await this.$axios.$delete(`/image/${result_id}`) // eslint-disable-line
        const newResults = await this.$axios.$get('/image/')
        this.results = newResults
      } catch (e) {
        console.log(e)
      }
    }
  }

}
</script>
```
先ほど作ったcomponentを登録し、resultとdeleteResultを与えます。  
asyncDataでは先ほど作ったバックエンドの情報をaxiosのgetで受け取ります。その受け取った結果をそれぞれのResultCardに割り当てます。  
また、deleteResultではaxiosのdeleteでresult.idに応じたデータを消しています。その後、消したあとのデータベースの値を受け取っています。
  
次にjudgeのページを作っていきます。
#### page/judge/index.vue
```javascript:
/* eslint-disable */
<template>
    <main class="container my-5">
        <div class="row">
            <div class="col-12 text-center my-3">
                <h2 class="mb-3 display-4 text-uppercase">
                    {{ prejudge.name}}
                </h2>
            </div>
            <div class="col-md-4">
                <form @submit.prevent="judge">
                    <div class="form-group">
                        <label for>名前</label>
                        <input v-model="prejudge.name" type="text">
                    </div>
                    <div class="form-group">
                        <label for>判定したい写真</label>
                        <input @change="onFileChange" type="file" name="file">
                    </div>
                    <button type="submit" class="btn btn-primary">
                        判定
                    </button>
                </form>
            </div>
        </div>
    </main>
</template>

<script>
export default {
  head () {
    return {
      title: '判定'
    }
  },
  data () {
    return {
      prejudge: {
        name: '',
        image: ''
      }
    }
  },
  methods: {
    onFileChange (e) {
      const files = e.target.files || e.dataTransfer.files
      if (!files.length) {
        return
      }
      this.prejudge.image = files[0]
    },
    async judge () {
      const config = {
        headers: { 'content-type': 'multipart/form-data' }
      }
      const formData = new FormData()
      for (const data in this.prejudge) {
        formData.append(data, this.prejudge[data])
      }
      try {
        await this.$axios.$post('/image/', formData, config)
        this.$router.push('/results/')
      } catch (e) {
        console.log(e)
      }
    }
  }
}
</script>
<style scoped>
</style>
/* eslint-disable */
```
入力されたデータをnameはv-modelで、画像はonFileCangeという関数でprejudgeに入れて、送信するために、formdataにします。  
そして、axiosのpostで送信したのち、results画面に$router.pushで戻ります。  

最後に、最初のページであるpages/index.vueとnuxtの設定ファイルnuxt.config.jsのaxiosの部分を設定します。
#### pages/index.vue
```javascript:
<template>
  <header>
    <div class="text-box">
      <h1>手書き文字を判定しよう!</h1>
      <nuxt-link class='btn btn-outline btn-large btn-info' to="/results">
      判定した文字達
      </nuxt-link>
    </div>
  </header>
</template>

<script>
export default {
  head () {
    return {
      title: 'Home page'
    }
  }
}
</script>

<style>
header {
  min-height: 100vh;
  background-image: linear-gradient(
      to right,
      rgba(0, 0, 0, 0.7),
      rgba(0, 0, 0, 0.2)
    ),
    url("/static/icon.png");
  background-position: center;
  background-size: cover;
  position: relative;
}
.text-box {
  position: absolute;
  top: 50%;
  left: 10%;
  transform: translateY(-50%);
  color: #fff;
}
.text-box h1 {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  font-size: 5rem;
}
.text-box p {
  font-size: 2rem;
  font-weight: lighter;
}

</style>
```

#### nuxt.config.js
```javascript:

...
axios: {
    baseURL: "http://localhost:8000/core"
  },
...

```
プロジェクトファイルは次のようになりました。
![](スクリーンショット%202020-08-21%2017.41.18.png)

## アプリの実行

それぞれのサーバーを立ち上げることで起動させることができます。  
djangoについては
```Bash:
$ python3 manage.py runserver
```
nuxtについては
```Bash:
$ npm run dev
```
で起動します。
それぞれの実行画面は次のようになっています。

**http://localhost:3000/**
![](スクリーンショット%202020-08-21%2017.06.25.png)

**http://localhost:3000/results/**
![](スクリーンショット%202020-08-21%2017.11.22.png)
**http://localhost:3000/judge**
![](スクリーンショット%202020-08-21%2017.07.43.png)

  
また、django側は
![](スクリーンショット%202020-08-21%2017.12.20.png)
のようになっておりきちんと動作していることがわかります。


特に参考にさせてもらったサイトは  
<https://nmomos.com/tips/2020/01/16/django-vue-nuxt-ssr/>  
<https://atma.hatenablog.com/entry/2020/03/27/184501>  
です。  
また、「いちばんやさしいPython機械学習の教本 人気講師が教える業務で役立つ実践ノウハウ」(インプレス)  
もかなり参考にさせてもらいました。
