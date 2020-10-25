本專案紀錄使用 Python 搭配 Django 框架實作 Line 聊天機器人 
同時記錄如何使用 [Ngrok](https://ngrok.com) 工具做本機測試  
也會紀錄如何部署到 [Heroku](https://dashboard.heroku.com) 雲端平台做測試   

![Result](https://github.com/quien697/linebot-echobot/blob/main/images/Result.jpg)

- - -

# 開發

1. 更新pip

  ```
  $ python3 -m pip install --upgrade pip
  ```

2. 創建資料夾

  ```
  $ mkdir linebot-echobot
  $ cd linebot-echobot
  ```

3. 建立虛擬環境

   建立虛擬環境，並命名為 venv

   ```
   linebot-echobot$ python3 -m venv venv
   ```

   啟動虛擬環境

   ```
   linebot-echobot$ source ./venv/bin/activate
   ```


4. 安裝 Django 模組 與 line-bot-sdk

  ```
  (venv) linebot-echobot$ pip install Django line-bot-sdk
  ```

5. 建立 Django 專案

  ```
  (venv) linebot-echobot$ django-admin startproject echobot
  (venv) linebot-echobot$ cd echobot
  (venv) linebot-echobot/echobot# python manage.py startapp echobot_app
  ```

6. 撰寫程式碼

   echobot/urls.py

   ```python
   from django.contrib import admin
   from django.urls import path, include
    
   urlpatterns = [
       path('admin/', admin.site.urls),
       path('echobot/', include('echobot_app.urls')),
   ]
   ```

   echobot_app/urls.py (預設沒有這個檔案，需要自行建立)

   ```python
   from django.urls import path
   from . import views
   
   urlpatterns = [
       path('callback', views.callback, name='callback'),
   ]
   ```

   echobot/settings.py (增加以下兩行)
   
   這兩個環境變數需要從 [LINE Developers](https://developers.line.biz/zh-hant/) 裡取得

   ```python
   # Line
   LINE_CHANNEL_ACCESS_TOKEN = 'Your channel access token'
   LINE_CHANNEL_SECRET = 'Your channel secret'
   ```

   * Messaging API -> Channel access token (如果沒有的話可以按 Issue 按紐來產生)

   ![LineBot_ChannelAccessToken](https://github.com/quien697/linebot-echobot/blob/main/images/LineBot_ChannelAccessToken.png)

   * Basic settings -> Channel secret 

   ![LineBot_ChannelAccessToken](https://github.com/quien697/linebot-echobot/blob/main/images/LineBot_ChannelSecret.png)

   echobot_app/view.py

   ```python
   from django.conf import settings
   from django.http import HttpResponse, HttpResponseBadRequest, HttpResponseForbidden
   from django.views.decorators.csrf import csrf_exempt
    
   from linebot import LineBotApi, WebhookHandler
   from linebot.exceptions import InvalidSignatureError, LineBotApiError
   from linebot.models import MessageEvent, TextMessage, TextSendMessage
    
   line_bot_api = LineBotApi(settings.LINE_CHANNEL_ACCESS_TOKEN)
   handler = WebhookHandler(settings.LINE_CHANNEL_SECRET)
    
   @csrf_exempt
   def callback(request):
       if request.method == 'POST':
           # get X-Line-Signature header value
           signature = request.META['HTTP_X_LINE_SIGNATURE']
           # get request body as text
           body = request.body.decode('utf-8')
    
           # handle webhook body
           try:
               handler.handle(body, signature)
           except InvalidSignatureError:
               # 如果 request 不是從 Line Server 來的，就會丟出 InvalidSignatureError
               return HttpResponseForbidden()
           except LineBotApiError:
               # 其他使用錯誤或 Line Server 的問題都會是丟出 LineBotApiError
               return HttpResponseBadRequest()
           return HttpResponse()
       else:
           return HttpResponseBadRequest()
    
   @handler.add(MessageEvent, message=TextMessage)
   def message_text(event: MessageEvent):
       line_bot_api.reply_message(
           event.reply_token,
           TextSendMessage(text=event.message.text)
       )
   ```

   到這裡，實作就完成了，再來就是要測試了～

# 本機測試

1. Run server

   ```
   (venv) linebot-echobot/echobot$ python manage.py runserver
   ```

   執行結果

      ![Localhost_Runserver](https://github.com/quien697/linebot-echobot/blob/main/images/Localhost_Runserver.png)

2. 利用 [Ngrok](https://ngrok.com) 將 localhost URLs 對應到 Public URLs

    ```
      $ ngrok http 8000
   ```
   
   執行結果
   
      ![Localhost_Ngrok](https://github.com/quien697/linebot-echobot/blob/main/images/Localhost_Ngrok.png)
   
3. echobot_app/settings.py

    ```python
    # Line
    ALLOWED_HOSTS = [
        '600fb7f6b167.ngrok.io'
    ]
    ```


4. 設定 Line Bot 的 Webhook URL

    ![LineBot_WebhookURL1](https://github.com/quien697/linebot-echobot/blob/main/images/LineBot_WebhookURL1.png)

- - -

# 部署到 [Heroku](https://dashboard.heroku.com) 做測試

1. 建立新的App，並取名為 linebot-echobot

   Dashboard -> New -> Create new app

   ![Heroku_CreateNewApp1](https://github.com/quien697/linebot-echobot/blob/main/images/Heroku_CreateNewApp1.png)

   Settings -> Buildpacks -> Add buildpack

   ![Heroku_CreateNewApp2](https://github.com/quien697/linebot-echobot/blob/main/images/Heroku_CreateNewApp2.png)

   選擇 python

   ![Heroku_CreateNewApp3](https://github.com/quien697/linebot-echobot/blob/main/images/Heroku_CreateNewApp3.png)

   這樣雲端伺服器就建立好了。

   （這個部分也可以使用指令的方式建立，但這邊我是直接在官網上面建立）

2. 安裝 gunicorn 工具

   ```
   (venv) linebot-echobot$ pip install gunicorn
   ```

3. linebot-echobot/requirements.txt

   在根目錄 linebot-echobot 下執行下列指令後，將會自動建立 requirements.txt 檔案

   ```
   (venv) linebot-echobot$ pip freeze > requirements.txt
   ```

4. linebot-echo/Procfile

   在根目錄 linebot-echobot 下手動建立 Procfile 檔案，並新增以下內容：

   ```python
   web: gunicorn --pythonpath echobot echobot.wsgi
   ```

   注意：此檔案名稱開頭必須是大寫，且沒有副檔名

5. echobot_app/settings.py

   ```python
   DEBUG = False
   
   ALLOWED_HOSTS = [
       'linebot-echobot.herokuapp.com'
   ]
   
   INSTALLED_APPS = [
       'echobot_app',
   ]
   ```

6. 登入 Heroku

       $ heroku login

7. 建立本地端的 Git Repository，並指定 git 到 Heroku 雲端平台的 linebot-echobot 上

   ```
   (venv) linebot-echobot$ git init
   (venv) linebot-echobot$ heroku git:remote -a linebot-echobot
   ```

8. 利用 Git 將程式碼推送到 Heroku 雲端平台

   ```
   (venv) linebot-echobot$ git add .
   (venv) linebot-echobot$ git commit -am "make it better"
   (venv) linebot-echobot$ git push heroku master
   ```

   如果 push 順利，那雲端伺服器就算是建立完成了，但如果有遇到狀況就得在看問題是什麼在處理，這裡就先不記錄如果遇到問題的處理。

9. 設定 Line Bot 的 Webhook URL

   ![LineBot_WebhookURL2](https://github.com/quien697/linebot-echobot/blob/main/images/LineBot_WebhookURL2.png)

- - -

# 設定 django-dotenv

為了避免上傳時，把 django 所產生的一些不相關的文件以及 Channel Access Token 與 Channel Secret 這兩個環境變數也上傳上去。
所以使用 .gitignore 搭配 python-dotenv 工具來設定。而 .gitignore 是使用 github 預設，也可以參考 [gitignore.io](https://www.toptal.com/developers/gitignore)

1. 安裝 django-dotenv

   ```
   (venv) linebot-echobot$ pip install python-dotenv
   ```

2. echobot_app/settings.py

   ```python
   import os
   
   # Dotenv
   from dotenv import load_dotenv
   load_dotenv()
   
   BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
   
   DATABASES = {
       'default': {
           'ENGINE': 'django.db.backends.sqlite3',
           'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
       }
   }
   
   LINE_CHANNEL_ACCESS_TOKEN = os.getenv("LINE_CHANNEL_ACCESS_TOKEN")
   LINE_CHANNEL_SECRET = os.getenv("LINE_CHANNEL_SECRET")
   ```

3. echobot/.env

   在目錄 echobot 下手動建立 .env 檔案

   ```python
   LINE_CHANNEL_ACCESS_TOKEN = 'Your channel access token'
   LINE_CHANNEL_SECRET = 'Your channel secret'
   ```
