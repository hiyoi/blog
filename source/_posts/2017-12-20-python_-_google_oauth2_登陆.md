---
layout: post
cid: 25
title: python - google oauth2 登陆
slug: google-oauth2-login
date: 2017/12/20 08:59:00
updated: 2019/10/02 13:27:43
status: publish
author: alex
categories: 
  - 默认分类
  - 技术
tags: 
  - python
  - flask
---


最近业务用上了google oauth2登陆,[google文档][1]写的很详细,在python flask很容易就能实现oauth2登陆.
oauth2的原理一张图就能概括:
![webflow.png][2]

需要用到的库:
The Google APIs Client Library for Python:
```
pip install --upgrade google-api-python-client
```
The google-auth, google-auth-oauthlib, and google-auth-httplib2 for user authorization.
```
pip install --upgrade google-auth google-auth-oauthlib google-auth-httplib2
```
下面贴一下关键代码(省略了很多无关的):

```
#登陆界面
@app.route('/index')
def login_index():
    return render_template('login.html')

#登陆
@app.route('/login', methods=['GET', 'POST'])
def login():
    if 'credentials' not in session:
        #首先要获取凭据，调用授权方法
        return redirect(url_for('authorize'))

    credentials = google.oauth2.credentials.Credentials(
        **session['credentials'])

    #到这里就授权成功，可以通过authed_session来调用google api
    authed_session = AuthorizedSession(credentials)
  
    #调用google api 获取授权用户的信息
    response = authed_session.get('https://www.googleapis.com/userinfo/v2/me')
    user = json.loads(response.text)
    #用户账号存入session
    session['user'] = user['email']
    #返回登陆成功页面   
    return render_template('success.html')

#注销
@app.route('/logout')
def logout():
    if 'credentials' not in session:
        return redirect(url_for('login_index'))
    credentials = google.oauth2.credentials.Credentials(
        **session['credentials'])
    try:
        # 注销凭据需要传入一个凭据token参数然后post到相应的api地址
        revoke = requests.post('https://accounts.google.com/o/oauth2/revoke',
                               params={'token': credentials.token},
                               headers={'content-type': 'application/x-www-form-urlencoded'})

        status_code = getattr(revoke, 'status_code')
        #注销成功后删除用户登陆session
        del session['user']
        if status_code == 200:
            #删除session中的凭据
            del session['credentials']
            flash('Logout successful!', 'success')
            return render_template('login.html')
    except Exception as e:
        app.logger.info(e)

#授权
@app.route('/authorize')
def authorize():
    #这里需要一个在google cloud 平台申请的一个凭据密钥，client_secret.json
    #需要到https://console.cloud.google.com/apis/credentials申请
    CLIENT_SECRETS_FILE = app.config['CLIENT_SECRETS_FILE']
    #需要申请的权限范围
    SCOPES = app.config['SCOPES']
    #回调地址
    REDIRECT_URI = app.config['REDIRECT_URI']
    flow = google_auth_oauthlib.flow.Flow.from_client_secrets_file(
        CLIENT_SECRETS_FILE, scopes=SCOPES)

    flow.redirect_uri = REDIRECT_URI
    authorization_url, state = flow.authorization_url(
        access_type='offline',
        include_granted_scopes='true')
    session['state'] = state
    return redirect(authorization_url)

#授权成功后的回调函数
@app.route('/oauth2callback')
def oauth2callback():
    try:
        state = session['state']
        CLIENT_SECRETS_FILE = app.config['CLIENT_SECRETS_FILE']
        SCOPES = app.config['SCOPES']
        REDIRECT_URI = app.config['REDIRECT_URI']
        flow = google_auth_oauthlib.flow.Flow.from_client_secrets_file(
            CLIENT_SECRETS_FILE, scopes=SCOPES, state=state)
        flow.redirect_uri = REDIRECT_URI
        authorization_response = request.url
        #获取授权token
        flow.fetch_token(authorization_response=authorization_response)
        credentials = flow.credentials
        #存储凭据到session
        session['credentials'] = credentials_to_dict(credentials)
    except Exception as e:
        if 'credentials' in session:
            del session['credentials']
        app.logger.info(e)
        return redirect(url_for('login_index'))
    return redirect(url_for('login'))


def credentials_to_dict(credentials):
    return {'token': credentials.token,
            'refresh_token': credentials.refresh_token,
            'token_uri': credentials.token_uri,
            'client_id': credentials.client_id,
            'client_secret': credentials.client_secret,
            'scopes': credentials.scopes}
```


  [1]: https://developers.google.com/api-client-library/python/auth/web-app
  [2]: https://i.loli.net/2019/10/02/t7Cm8rdiDKfq5FA.jpg