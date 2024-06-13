わかりました。Twitterの「いいね」したツイートを取得して表示するWebアプリケーションを作成し、Dockerを使ってデプロイする方法をステップごとに説明します。

## ステップ 1: プロジェクトのディレクトリ構造を設定する

プロジェクトのディレクトリ構造を以下のように設定します。

```
twitter_likes/
├── app/
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
└── .env
```

```
# mkdir.sh
mkdir -p twitter_likes/app \
touch twitter_likes/app/main.py \
touch twitter_likes/app/requirements.txt \
touch twitter_likes/app/Dockerfile \
touch twitter_likes/.env
echo "DONE"
```

## ステップ 2: 必要なファイルを作成する

### `main.py`

`main.py`はFastAPIアプリケーションのメインファイルです。以下の内容を記述します。

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.templating import Jinja2Templates
from fastapi.responses import JSONResponse
from pydantic import BaseModel
import requests
import os
import logging

app = FastAPI()
templates = Jinja2Templates(directory="templates")

# ログ設定
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class UserID(BaseModel):
    screen_name: str

def get_twitter_likes(screen_name: str):
    bearer_token = os.getenv("BEARER_TOKEN")
    auth_token = os.getenv("AUTH_TOKEN")
    ct0 = os.getenv("CT0")
    cookie = os.getenv("COOKIE")

    logger.info(f"Bearer Token: {bearer_token}")
    logger.info(f"Auth Token: {auth_token}")
    logger.info(f"CT0 Token: {ct0}")
    logger.info(f"Cookie: {cookie}")

    if not bearer_token or not auth_token or not ct0 or not cookie:
        raise HTTPException(status_code=500, detail="Necessary tokens are not found")

    url = f"https://api.twitter.com/1.1/favorites/list.json?screen_name={screen_name}"
    headers = {
        "Authorization": f"Bearer {bearer_token}",
        "Content-Type": "application/json",
        "Cookie": cookie,
        "x-csrf-token": ct0
    }

    response = requests.get(url, headers=headers)

    logger.info(f"Response Status Code: {response.status_code}")
    logger.info(f"Response Text: {response.text}")

    if response.status_code != 200:
        raise HTTPException(status_code=response.status_code, detail=response.text)

    return response.json()

@app.post("/likes/")
def read_likes(user: UserID):
    try:
        likes = get_twitter_likes(user.screen_name)
        return JSONResponse(content=likes)
    except Exception as e:
        logger.error(f"Error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/")
def root(request: Request):
    return templates.TemplateResponse("index.html", {"request": request})
```

### `requirements.txt`

`requirements.txt`には、プロジェクトで必要なPythonライブラリを記述します。

```
fastapi
uvicorn
requests
pydantic
jinja2
```

### `Dockerfile`

`Dockerfile`は、アプリケーションをDockerコンテナ内で動作させるための設定ファイルです。

```Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### `.env`

`.env`ファイルには、Twitter APIの認証情報を記述します。

```dotenv
BEARER_TOKEN=your_bearer_token
AUTH_TOKEN=your_auth_token
CT0=your_ct0_token
COOKIE='your_cookie_data'
```

### `index.html`

`templates`ディレクトリを作成し、その中に`index.html`を配置します。

```html
<!DOCTYPE html>
<html>
<head>
    <title>Twitter Likes Visualization</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f8fa;
            margin: 0;
            padding: 0;
        }
        .container {
            width: 600px;
            margin: 0 auto;
            padding: 20px;
        }
        h1 {
            text-align: center;
            color: #1da1f2;
        }
        form {
            display: flex;
            justify-content: center;
            margin-bottom: 20px;
        }
        input[type="text"] {
            width: 300px;
            padding: 10px;
            border: 1px solid #ccc;
            border-radius: 5px;
            margin-right: 10px;
        }
        button {
            padding: 10px 20px;
            background-color: #1da1f2;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        button:hover {
            background-color: #0d95e8;
        }
        .tweet {
            background-color: white;
            border: 1px solid #e1e8ed;
            border-radius: 10px;
            padding: 15px;
            margin-bottom: 10px;
            box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
        }
        .tweet p {
            margin: 0;
        }
        .tweet a {
            color: #1da1f2;
            text-decoration: none;
        }
        .tweet a:hover {
            text-decoration: underline;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Twitter Likes Visualization</h1>
        <form id="userForm">
            <input type="text" id="screen_name" name="screen_name" placeholder="Enter Twitter Screen Name" required>
            <button type="submit">Get Likes</button>
        </form>
        <div id="tweets"></div>
    </div>

    <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
    <script>
        document.getElementById('userForm').addEventListener('submit', function(event) {
            event.preventDefault();
            const screenName = document.getElementById('screen_name').value;
            fetch('/likes/', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ screen_name: screenName })
            })
            .then(response => response.json())
            .then(data => {
                const tweetsDiv = document.getElementById('tweets');
                tweetsDiv.innerHTML = '';
                data.forEach(tweet => {
                    const tweetDiv = document.createElement('div');
                    tweetDiv.className = 'tweet';
                    tweetDiv.innerHTML = `
                        <blockquote class="twitter-tweet">
                            <p lang="en" dir="ltr">${tweet.text}</p>
                            &mdash; ${tweet.user.name} (@${tweet.user.screen_name}) <a href="https://twitter.com/${tweet.user.screen_name}/status/${tweet.id_str}"> ${new Date(tweet.created_at).toLocaleString()}</a>
                        </blockquote>
                    `;
                    tweetsDiv.appendChild(tweetDiv);
                });
                twttr.widgets.load();
            })
            .catch(error => console.error('Error:', error));
        });
    </script>
</body>
</html>
```

## ステップ 3: Dockerコンテナのビルドと実行

### Dockerイメージのビルド

以下のコマンドを実行してDockerイメージをビルドします。

```sh
docker build -t twitter-likes-app ./app
```

### Dockerコンテナの実行

以下のコマンドを実行してDockerコンテナを起動します。`.env`ファイルを環境変数として読み込むために`--env-file`オプションを使用します。

```sh
docker run --env-file .env -p 8000:8000 twitter-likes-app
```

## ステップ 4: アプリケーションの確認

コンテナが起動したら、ブラウザで`http://localhost:8000/`にアクセスします。フォームにTwitterのユーザー名を入力し、「Get Likes」ボタンを押すと、そのユーザーが「いいね」したツイートがTwitter埋め込み形式で表示されます。