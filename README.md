本專案使用 Python 搭配 Django 框架實作 Line 聊天機器人並部署到 Heroku 雲端平台

開發
-----

1. 更新pip

  ```
  # python3 -m pip install --upgrade pip
  ```

2. 創建資料夾

  ```
  mkdir linebot-echobot
  cd linebot-echobot
  ```

3. 建立虛擬環境

   建立虛擬環境，並命名為 venv

   ```
   linebot-echobot$ python3 -m venv venv
   linebot-echobot$ source ./venv/bin/activate
   ```

   啟動虛擬環境

   	linebot-echobot$ source ./venv/bin/activate


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

  ```python
  # Line
  LINE_CHANNEL_ACCESS_TOKEN = 'Your channel access token'
  LINE_CHANNEL_SECRET = 'Your channel secret'
  ```

以上兩個環境變數需要從 [LINE Developers](https://developers.line.biz/zh-hant/) 裡取得

* Messaging API -> Channel access token (如果沒有的話可以按 Issue 按紐來產生)

  ![LineBot_ChannelAccessToken](/Users/quien/Desktop/images/LineBot_ChannelAccessToken.png)

* Basic settings -> Channel secret 

  ![LineBot_ChannelAccessToken](/Users/quien/Desktop/images/LineBot_ChannelAccessToken.png)

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

7.   本機測試

      Run server

        ```
      (venv) linebot-echobot/echobot$ python manage.py runserver
        ```

      執行結果

     ![Localhost_Runserver](/Users/quien/Desktop/images/Localhost_Runserver.png)

8. 利用 Ngrok

        $ ngrok http 8000
    
     執行結果
    
    ![Localhost_Ngrok](/Users/quien/Desktop/images/Localhost_Ngrok.png)
    
9. echobot_app/settings.py

    ```python
    # Line
    ALLOWED_HOSTS = [
        '600fb7f6b167.ngrok.io'
    ]
    ```


10. 設定 Webhook URL

    ![LineBot_WebhookURL1](/Users/quien/Desktop/images/LineBot_WebhookURL1.png)

11. 結果測試

    <img src="/Users/quien/Desktop/images/Result.jpg" alt="Demo" style="zoom: 33%;" />

    

部署到 [Heroku](https://dashboard.heroku.com)

1. 建立新的App，並取名為 linebot-echobot

   Dashboard -> New -> Create new app

   ![Heroku_CreateNewApp1](/Users/quien/Desktop/images/Heroku_CreateNewApp1.png)

   Settings -> Buildpacks -> Add buildpack

   ![Heroku_CreateNewApp2](/Users/quien/Desktop/images/Heroku_CreateNewApp2.png)

   選擇 python

   ![Heroku_CreateNewApp3](/Users/quien/Desktop/images/Heroku_CreateNewApp3.png)

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

   ![LineBot_WebhookURL2](/Users/quien/Desktop/images/LineBot_WebhookURL2.png)

上傳到 Github 前的設定

因為在上傳程式碼到 Github 的時，Django 框架會有一些文件是不需要 push 上去的，再加上 Line bot 的 Channel Access Token 與 Channel Secret 這兩個環境變數也不想要上傳上去。所以需要使用 python-dotenv 與 .gitignore

設定django-dotenv

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

4. linebot-echobot/.gitignore 

   在根目錄 linebot-echobot 下建立 .gitignore 檔案，並新增以下內容：

   .gitignore 內容來自 [gitignore.io](https://www.toptal.com/developers/gitignore)

   ```python
   # Created by https://www.toptal.com/developers/gitignore/api/python,django
   # Edit at https://www.toptal.com/developers/gitignore?templates=python,django
   
   ### Django ###
   *.log
   *.pot
   *.pyc
   __pycache__/
   local_settings.py
   db.sqlite3
   db.sqlite3-journal
   media
   
   # If your build process includes running collectstatic, then you probably don't need or want to include staticfiles/
   # in your Git repository. Update and uncomment the following line accordingly.
   # <django-project-name>/staticfiles/
   
   ### Django.Python Stack ###
   # Byte-compiled / optimized / DLL files
   *.py[cod]
   *$py.class
   
   # C extensions
   *.so
   
   # Distribution / packaging
   .Python
   build/
   develop-eggs/
   dist/
   downloads/
   eggs/
   .eggs/
   lib/
   lib64/
   parts/
   sdist/
   var/
   wheels/
   pip-wheel-metadata/
   share/python-wheels/
   *.egg-info/
   .installed.cfg
   *.egg
   MANIFEST
   
   # PyInstaller
   #  Usually these files are written by a python script from a template
   #  before PyInstaller builds the exe, so as to inject date/other infos into it.
   *.manifest
   *.spec
   
   # Installer logs
   pip-log.txt
   pip-delete-this-directory.txt
   
   # Unit test / coverage reports
   htmlcov/
   .tox/
   .nox/
   .coverage
   .coverage.*
   .cache
   nosetests.xml
   coverage.xml
   *.cover
   *.py,cover
   .hypothesis/
   .pytest_cache/
   pytestdebug.log
   
   # Translations
   *.mo
   
   # Django stuff:
   
   # Flask stuff:
   instance/
   .webassets-cache
   
   # Scrapy stuff:
   .scrapy
   
   # Sphinx documentation
   docs/_build/
   doc/_build/
   
   # PyBuilder
   target/
   
   # Jupyter Notebook
   .ipynb_checkpoints
   
   # IPython
   profile_default/
   ipython_config.py
   
   # pyenv
   .python-version
   
   # pipenv
   #   According to pypa/pipenv#598, it is recommended to include Pipfile.lock in version control.
   #   However, in case of collaboration, if having platform-specific dependencies or dependencies
   #   having no cross-platform support, pipenv may install dependencies that don't work, or not
   #   install all needed dependencies.
   #Pipfile.lock
   
   # PEP 582; used by e.g. github.com/David-OConnor/pyflow
   __pypackages__/
   
   # Celery stuff
   celerybeat-schedule
   celerybeat.pid
   
   # SageMath parsed files
   *.sage.py
   
   # Environments
   .env
   .venv
   env/
   venv/
   ENV/
   env.bak/
   venv.bak/
   pythonenv*
   
   # Spyder project settings
   .spyderproject
   .spyproject
   
   # Rope project settings
   .ropeproject
   
   # mkdocs documentation
   /site
   
   # mypy
   .mypy_cache/
   .dmypy.json
   dmypy.json
   
   # Pyre type checker
   .pyre/
   
   # pytype static type analyzer
   .pytype/
   
   # profiling data
   .prof
   
   ### Python ###
   # Byte-compiled / optimized / DLL files
   
   # C extensions
   
   # Distribution / packaging
   
   # PyInstaller
   #  Usually these files are written by a python script from a template
   #  before PyInstaller builds the exe, so as to inject date/other infos into it.
   
   # Installer logs
   
   # Unit test / coverage reports
   
   # Translations
   
   # Django stuff:
   
   # Flask stuff:
   
   # Scrapy stuff:
   
   # Sphinx documentation
   
   # PyBuilder
   
   # Jupyter Notebook
   
   # IPython
   
   # pyenv
   
   # pipenv
   #   According to pypa/pipenv#598, it is recommended to include Pipfile.lock in version control.
   #   However, in case of collaboration, if having platform-specific dependencies or dependencies
   #   having no cross-platform support, pipenv may install dependencies that don't work, or not
   #   install all needed dependencies.
   
   # PEP 582; used by e.g. github.com/David-OConnor/pyflow
   
   # Celery stuff
   
   # SageMath parsed files
   
   # Environments
   
   # Spyder project settings
   
   # Rope project settings
   
   # mkdocs documentation
   
   # mypy
   
   # Pyre type checker
   
   # pytype static type analyzer
   
   # profiling data
   
   # End of https://www.toptal.com/developers/gitignore/api/python,django
   
   ```