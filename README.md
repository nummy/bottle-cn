### 简介
Bottle框架，它被设计为仅仅只有一个文件的Python模块，并且除Python标准库外，它不依赖于任何第三方模块。

- 路由（Routing）：将请求映射到函数，可以创建十分优雅的 URL
- 模板（Templates）：Pythonic 并且快速的 Python 内置模板引擎，同时还支持 mako, jinja2, cheetah 等第三方模板引擎
- 工具集（Utilites）：快速的读取 form 数据，上传文件，访问 cookies，headers 或者其它 HTTP 相关的 metadata
- 服务器（Server）：内置HTTP开发服务器，并且支持 paste, fapws3, bjoern, Google App Engine, Cherrypy 或者其它任何 WSGI HTTP 服务器

### 安装
正如上面所说的， Bottle 被设计为仅仅只有一个文件，我们甚至可以不安装它，直接将 bottle.py 文件下载即可。
```
$ wget http://bottlepy.org/bottle.py
```
或者使用pip安装：
```
$ pip install bottle
```

如果我们直接将 bottle.py 下载到自己的应用中的话，我们可以建立下面这样的目录结构：
```
+ application
+----bottle.py
+----app.py
```

### Hello World
下面编写一个hello world实例：
```
from bottle import route, run

@route('/hello')
def hello():
    return "Hello World!"

run(host='localhost', port=8080, debug=True)
```
运行上述代码，然后访问 http://localhost:8080/hello 即可在浏览器输出Hello World。

`route()`修饰器用于映射路由，`run()`用于运行一个内置的web服务器。

我们也可以先创建一个 Bottle 对象 app，然后将所有的函数都映射到 app 的 URL 地址上，如上示例我们可以用下面这种办法来实现：
```
from bottle import Bottle, run

app = Bottle()

@app.route('/hello')
def hello():
    return "Hello World!"

run(app, host='localhost', port=8080)
```

### 路由分发

`route()`修饰器将URL路径映射到视图函数，可以给应用添加多个路由:
```python
from bottle import template

@route('/')
@route('/hello/<name>')
def greet(name='Stranger'):
    return template('Hello {{name}}, how are you?', name=name)
```
视图函数可以绑定多个路由，可以在URL中视同通配符来获取查询参数。

#### 动态路由

包含通配符的路由称为动态路由，一个简单的动态路由使用<>将动态参数名字括起来<name>。例如，/hello/<name>匹配/hello/jim，也匹配/hello/kate。

每个通配符将匹配的URL参数作为关键字参数传递给视图函数。
```
@route('/wiki/<pagename>')            # matches /wiki/Learning_Python
def show_wiki_page(pagename):
    ...

@route('/<action>/<user>')            # matches /follow/defnull
def user_api(action, user):
```
路由中还可以使用过滤器限定URL中参数的类型，并将URL参数转换为指定类型，它的语法如下：
```
<name:filter> or <name:filter:config>
```
bottle默认提供一下过滤器：
- :int 只匹配数字，并将其转换为整数
- :float 匹配浮点数
- :path 匹配路径
- :re 匹配符合config中指定的正则表达式

示例：
```
@route('/object/<id:int>')
def callback(id):
    assert isinstance(id, int)

@route('/show/<name:re:[a-z]+>')
def callback(name):
    assert name.isalpha()

@route('/static/<path:path>')
def callback(path):
    return static_file(path, ...)
```

还可以自定义过滤器。

此外，路由还可以显示声明：
```
def setup_routing(app):
    app.route('/new', ['GET', 'POST'], form_new)
    app.route('/edit', ['GET', 'POST'], form_edit)

app = Bottle()
setup_routing(app)
```

### 请求方法

默认请求方法为GET，如果需要使用其它方法，可以在`route()`中指定method参数，或者使用修饰器：`get()`、`post()`、`put()`、`delete()`或者`patch()`等。

例如：
```python
from bottle import get, post, request # or route

@get('/login') # or @route('/login')
def login():
    return '''
        <form action="/login" method="post">
            Username: <input name="username" type="text" />
            Password: <input name="password" type="password" />
            <input value="Login" type="submit" />
        </form>
    '''

@post('/login') # or @route('/login', method='POST')
def do_login():
    username = request.forms.get('username')
    password = request.forms.get('password')
    if check_login(username, password):
        return "<p>Your login information was correct.</p>"
    else:
        return "<p>Login failed.</p>"
```

#### 特殊方法HEAD和ANY

HEAD 方法，经常被用来处理一些仅仅只需要返回请求头部信息而不需要返回整个请求结果的事务，这些HEAD方法十分有用，可以让我们仅仅只获得我们需要的数据，而不必要返回整个文档。

Bottle 可以帮助我们很简单的实现这些功能，它将HEAD请求转换为GET请求，然后自动截取请求需要的数据，这样一来，你不再需要定义任何特殊的 HEAD 路由了。

非标准方法ANY用于会匹配所有的请求，而不去过滤指定的方法。如果没有指定任何HTTP方法，才会使用ANY方法。


#### 静态文件路由
对于静态文件， Bottle 内置的服务器并不会自动的进行处理，这需要你自己定义一个路由，告诉服务器在哪些文件是需要服务的，并且在哪里可以找到它们，我们可以写如下面这样的一个路由器：
```
from bottle import static_file
@route('/static/:filename')
def server_static(filename):
    return static_file(filename, root='/path/to/your/static/files')
```
static_file() 函数可以安全且方便的用来返回静态文件，上面的示例中，我们只返回”/path/to/your/static/files” 路径下的文件，因为 :filename 通配符并不接受任何 “/” 的字符，如果我们想要“/path/to/your/static/files” 目录的子目录下的文件也被处理，那么我们可以使用一个格式化的通配符：
```
@route('/static/:path#.+#')
def server_static(path):
    return static_file(path, root='/path/to/your/static/files')
```

#### 错误页面
如果运行异常，Bottle提供了`error()`修饰器用于指定错误码的返回页面。
```python
from bottle import error
@error(404)
def error404(error):
    return 'Nothing here, sorry'
```
定义上面的函数后，当HTTP返回码为404时，将返回一个错误页面。传递给异常处理函数的参数是一个HTTPError的实例。除此之外异常处理函数与视图函数一样。

### 生成内容
在WSGI中，你的应用能返回的数据类型是十分有限的，你必须返回可迭代的字节字符串，你能返回字符串是因为字符串是可迭代的，但是这导致服务器将你的内容按一字符一字符的传送。Unicode字符串被禁止返回，这很不实用。

Bottle则支持了更多的数据类型，它甚至添加了一个 Content-Length 头部信息，并且自动编码 Unicode 数据，下面列举了 Bottle 应用中，你可以返回的数据类型，并且简单的介绍了一下这些数据类型的数据都是怎么被 Bottle 处理的：
数据类型|	介绍
-------------------|-------------------------
字典	|Python 内置的字典类型数据将被自动被转换为JSON 字符串，并且添加头部信息Content-Type为 'application/json' 的头信息返回至浏览器，这让我们可以很方便的建立基于JSON的API
空字符串，False，None或者任何非真的数据 |	Bottle返回空，Content-Length设置为0
Unicode 字符串 |Unicode 字符串将自动的按 Content-Type 头中定义的编码格式进行编码（默认为UTF8），然后按普通的字符串进行处理
字节串（Byte strings）|	Bottle 返回整个字符串（而不是按字节一个一个返回），同 时设置Content-Length 头为字节串长度，如果是通过yeild返回的字节字符串，则不设置该头部信息。
HTTPError 与HTTPResponse 实例|	返回这些实例就像抛出异常一样，对于 HTTPError，错误将被相关函数处理
文件对象|	任何具有`.read()` 方法的对象都被看作文件或者类似文件的对象进行处理，并传送给 WSGI 服务器框架定义的wsgi.file_wrapper 回调函数，某些WSGI服务器会使用系统优化的请求方式（Sendfile）来发送文件。
迭代器与生成器|	你可以在你的回调函数使用 yield 或者 返回一个迭代器，只要yield的对象是字符串，Unicode 字符串，HTTPError 或者 HTTPResponse 对象就行，但是不允许使用嵌套的迭代器，需要注意的是，当`yield` 的值第一次为非空时， HTTP 的状态 和 头文件将被发送到 浏览器

上面的顺序非常重要，如果你返回一个继承自str的类实例，并且带有 read() 方法，那它还是将按字符串进行处理，因为字符串有更高一级的优先处理权。

#### 改变默认编码

Bottle根据请求头中的Content-Type字段来对字符串进行编码，该字段取值默认为 text/html; charset=UTF8 ，但是可以被Response.content_type 属性修改，或者直接被 Response.charset 属性修改：

```python
from bottle import response
@route('/iso')
def get_iso():
    response.charset = 'ISO-8859-15'
    return u'This will be sent with ISO-8859-15 encoding.'
@route('/latin9')
def get_latin():
    response.content_type = 'text/html; charset=latin9'
    return u'ISO-8859-15 is also known as latin9.'
```
由于某些罕见的原因，Python 编码的名称可能与 HTTP 编码的名称不一致，这时你需要做两方法的工作首先设置Response.content_type 头文件，然后还需要设置 Response.charset 。

#### 静态文件

使用`static_file()`返回静态文件。

#### 强制下载

有些浏览器会使用指定应用打开相应格式的文件，我们可以使用download参数来限制该文件只能下载。
```
@route('/download/<filename:path>')
def download(filename):
    return static_file(filename, root='/path/to/static/files', download=filename)
```
如果download为True，则使用原始文件名。

#### HTTP 错误与重定向

abort() 函数是创建 HTTP 错误页面的快捷方式：
```
from bottle import route, abort
@route('/restricted')
def restricted():
    abort(401, 'Sorry, access denied.')
```
如果要将浏览器请求的地址重定向其它的地址，你可以向浏览器发送一个 303 see other 响应， redirect() 可以实现这个功能：
```
from bottle import redirect
@route('/wrong/url')
def wrong():
    redirect('/right/url')
```
这两个方法都是通过抛出HTTPErro异常来实现。

除了HTTPResponse和HTTPError外，其它异常将返回500 Internal Server Error。

#### 响应对象

响应数据如HTTP状态码，响应头，或者Cookies都被保存在一个叫做response 的对象中，并传送给浏览器，你可以直接操作这些元数据或者写一些预定义的方法来处理它们。

##### 状态码

HTTP状态码控制浏览器的行为，默认为200，大多数情况下，我们都不需要手动设置状态码，但是在使用abort()时可以指定响应码。

##### 响应头

响应头字段，如Cache-Control、Location，通过Respons.set_header()进行设置。
该函数接受两个参数：头部字段名和对应的取值，字段名区分大小写：
```
@route('/wiki/page')
def wiki(page):
    response.set_header('Content-Language', 'en')
    ...
```
绝大多数头部字段都只能定义一次，但是有一些特别的头文件却可以多次定义，这个时候我们在第一次定义时使用Response.set_header() ，但是第二次定义时，就需要使用 Response.add_header() 了：
```
response.set_header('Set-Cookie','name=value')
response.add_header('Set-Cookie','name1=value1')
```

#### COOKIES

通过Request.get_cookie()获取cookie，Response.set_cookie()设置cookie信息。
```python
@route('/hello')
def hello_again():
    if request.get_cookie("visited"):
        return "Welcome back! Nice to see you again"
    else:
        response.set_cookie("visited", "yes")
        return "Hello there! Nice to meet you"
```
Response.set_cookie()还提供了其它选项来控制cookie，常见参数如下:
- max_age ： 该 Cookie 最大的生命期（按秒计算，默认为 None）
- expires ： 上个 datetime 对象或者一个 UNIX timestamp（默认为 None）
- domain ： 允许访问该 Cookie 的域名（默认为当前应用的域名）
- path ： 按照路径限制当前 Cookie（默认为 “/“）
- secure ： 限制当前Cookie仅仅允许通过 HTTPS 连接访问（默认为 off）
- httponly ： 阻止浏览器端 Javascript 读取当前 Cookie（默认为 off，需要 Python 2.7 以上）

如果expires或者max_age都没有设置的话，Cookie将在浏览器的会话结束后或者当浏览器关闭时失效，下面一些问题在使用 Cookie时也需要考虑到的：
- 大多数浏览器都限制 Cookie 的大小不能超过 4Kb
- 有一些用户设置了他们的浏览器不接受任何 Cookie，绝大多数搜索引擎也直接忽略 Cookie，你应该保证你的应用在没有 Cookie 时也是可用的
- Cookie 保存在客户端，并且没有任何加密措施，你存放在 Cookie 中的任何内容，用户都是可访问的，如果有必要的话，攻击者能通过 XSS 漏洞窃取用户的 Cookie，所以，尽可能在不要在 Cookie 中保存机密信息
- Cookie 是很容易被伪造的，所以，尽可能不要想信 Cookie

##### 签名COOKIE
由于Cookie容易被恶意软件盗取，所以Bottle 为 Cookie 提供了加密方法，你所需要做的仅仅只是提供了一个密钥，只要能确保该密钥的安全即可，而其导致的结果是，对于未加密的 Cookie，Request.get_cookie()将返回 None。
```
@route('/login')
def login():
    username = request.forms.get('username')
    password = request.forms.get('password')
    if check_user_credentials(username, password):
        response.set_cookie('account', username, secret='some-sceret-key')
        return 'Welcome {}'.format(username)

@route('/restricted')
def restricted_area(self):
    username = request.get_cookie('account', secret='some-secret-key')
    if username:
        return 'Hello {}'.format(username)
    else:
        return 'You are not logged in.'
```
另外，Bottle 会自动 pickle 与 unpickle 你存储到已签名的 Cookie 上的数据，这表示你可以向 Cookie 中存储任何可以 pickle 的数据对象，只要其大小不超过 4Kb即可。

### 请求数据

Cookie、HTTP头、HTML表单以及其它请求数据都可以通过全局对象request获取。这个对象总是指向当前的请求，即使是在多线程环境且同时存在多个连接的情况下。
```
from bottle import request, route, template

@route('/hello')
def hello():
    name = request.cookies.username or 'Guest'
    return template('Hello {{name}}', name=name)
```

request对象是BaseRequest的子类。

#### FormsDict
Bottle使用一种特殊的字典--FormsDict来存储表单数据和cookie信息。FormsDict与字典类似，但是还提供了其它便利方法。

- 属性访问 所有的值都可以像访问对象属性一样进行访问，这些虚拟属性以unicode的形式返回，即使对应的值为空，或者解码错误都会返回unicode字符串，只不过为空字符串。
```
name = request.cookies.name

# is a shortcut for:

name = request.cookies.getunicode('name') # encoding='utf-8' (default)

# which basically does this:

try:
    name = request.cookies.get('name', '').decode('utf-8')
except UnicodeError:
    name = u''
```
- 每个键可以对应多个值 FormsDict是MultiDict的子类，每个键都可以存储多个值，使用标准的get方法还是只返回一个值，但是使用getall()可以返回所有的取值：
```
for choice in request.forms.getall('multiple_choice'):
    do_something(choice)
```
#### COOKIE

所有客户端的cookie信息都可以通过BaseRequest.cookies来获取。
```
from bottle import route, request, response
@route('/counter')
def counter():
    count = int( request.cookies.get('counter', '0') )
    count += 1
    response.set_cookie('counter', str(count))
    return 'You visited this page %d times' % count
```

#### HTTP 请求头

所有的HTTP头部信息都存储在WSGIHeaderDict中，WSGIHeaderDict是一个字典，但是key对大小写不敏感。通过BaseRequest.headers可以获取头部信息。
```
from bottle import route, request
@route('/is_ajax')
def is_ajax():
    if request.headers.get('X-Requested-With') == 'XMLHttpRequest':
        return 'This is an AJAX request'
    else:
        return 'This is a normal request'
```

#### 查询字符串

查询字符串常常被用来传递一些小数目的键值对参数到服务器，你可以使用 Request.query对其进行访问，使用 Request.query_string来获得整个字符串：
```
from bottle import route, request, response, template
@route('/forum')
def display_forum():
    forum_id = request.query.id
    page = request.query.page or '1'
    return template('Forum ID: {{id}} (page {{page}})', id=forum_id, page=page)
```

#### 表单数据
表单数据存储在BaseRequest.forms中，以FormsDict的形式保存。
```
from bottle import route, request

@route('/login')
def login():
    return '''
        <form action="/login" method="post">
            Username: <input name="username" type="text" />
            Password: <input name="password" type="password" />
            <input value="Login" type="submit" />
        </form>
    '''

@route('/login', method='POST')
def do_login():
    username = request.forms.get('username')
    password = request.forms.get('password')
    if check_login(username, password):
        return "<p>Your login information was correct.</p>"
    else:
        return "<p>Login failed.</p>"
```

Bottle还提供了其他一些方式来获取数据：
Attribute | GET Form fields	| POST Form fields |	File Uploads
:----------------|----------------|------|-----
BaseRequest.query	|yes	|no|	no
BaseRequest.forms|	no|	yes	|no
BaseRequest.files|	no	|no|	yes
BaseRequest.params	|yes	|yes|	no
BaseRequest.GET	|yes	|no|	no
BaseRequest.POST|	no|	yes	|yes


#### 文件上传

Bottle将上传的文件保存在BaseRequest.files中，以FileUploads实例的方式保存。

上传表单定义如下：
```
<form action="/upload" method="post" enctype="multipart/form-data">
  Category:      <input type="text" name="category" />
  Select a file: <input type="file" name="upload" />
  <input type="submit" value="Start upload" />
</form>
```
后端代码如下：
```
@route('/upload', method='POST')
def do_upload():
    category   = request.forms.get('category')
    upload     = request.files.get('upload')
    name, ext = os.path.splitext(upload.filename)
    if ext not in ('.png','.jpg','.jpeg'):
        return 'File extension not allowed.'

    save_path = get_save_path_for_category(category)
    upload.save(save_path) # appends upload.filename automatically
    return 'OK'
```

FileUpload.filename保存了客户端上传文件的名字，但是它经过了一些处理，如果需要获取原始的文件名，应该使用FileUpload.raw_filename。

推荐使用FileUpload.save方法将文件保存到服务器，因为它更加高效，还可以通过FileUpload.file访问文件。

#### JSON数据
如果客户端以JSON的格式传递过来，可以使用BaseRequest.json 获取数据。

#### 原始请求数据
如果想要获取原始的请求数据，可以访问 BaseRequest.body。

#### WSGI环境
每个BaseRequest实例包含一个WSGI环境字典，它保存在BaseRequest.environ中，如果你需要直接访问这些数据，可以这样做：
```
@route('/my_ip')
def show_ip():
    ip = request.environ.get('REMOTE_ADDR')
    # or ip = request.get('REMOTE_ADDR')
    # or ip = request['REMOTE_ADDR']
    return template("Your IP is: {{ip}}", ip=ip)
```
