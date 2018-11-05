目录
1，新建项目
2，用户系统开发
3，书籍商品模块
如图4所示，用户中心实现
5，购物车功能
6，订单页面的开发
7，使用缓存
8，评论功能
9，发送邮件功能实现
10，登陆验证码功能实现
11，全文检索的实现
12，用户激活功能实现
13，用户中心最近浏览功能
14，过滤器功能实现
15，使用的nginx + gunicorn + django的进行部署
16，django的日志模块的使用
17，中间件的编写
1，新建项目
1，新建项目
新建项目之前需要先看2.，把包都安装了！

$ django-admin startproject bookstore
2，将需要用的包添加进来
# wq保存
$ vim requirements.txt
安装包文件如下：

＃ requirements.txt 
amqp == 2.2 .2
billiard == 3.5 .0.3
芹菜== 4.1 .0
Django == 1.8 .2
django - haystack == 2.6 .1
django - redis == 4.8 .0
django - tinymce == 2.6 .0
itsdangerous == 0.24 
解霸== 0.39 
海带== 4.1 0.0
olefile == 0.44 
Pillow == 4.3 .0
pycryptodome == 3.4 .7
PyMySQL == 0.7 .11
python -支付宝- sdk == 1.7 .0
pytz == 2017.2 
redis == 2.10 .6
 ＃ uWSGI == 2.0.15 
vine == 1.1 .4
飞快移动== 2.7 .4
gunicorn
安装环境（在虚拟环境中）

$ pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple/
3，修改项目配置文件，将默认的SQLite改为MySQL的
＃书店/ settings.py 
DATABASES  = {
     '缺省'：{
         ' ENGINE '： ' django.db.backends.mysql '，
         ' NAME '： '书店'，
         '用户'： '根'，
         ' PASSWORD '： ' '，
         '主持人'： ' 127.0.0.1 '，
        “港口'：3306，
    }
}
为了使用的MySQL的驱动，在主应用程序的__init__.py文件中加入以下两行代码：

导入 pymysql
pymysql.install_as_MySQLdb（）
2，用户系统开发
如图1所示，用户系统的开发
新建用户这个应用程序，也就是用户的应用程序，先从注册页做起。

$ python manage.py startapp users
我们建好用户app后，需要将它添加到配置文件中去。

bookstore / settings.py
 INSTALLED_APPS  =（
     ... 
    ' users '，＃用户模块 
）
然后我们需要设计表结构，我们要思考一下，这个用户数据表结构应该包含哪些字段？我们需要先抽象出一个BaseModel，一个基本模型，什么意思呢？因为数据表有共同的字段，我们可以把它抽象出来，比如create_at（创建时间），update_at（更新时间），is_delete（软删除）。注意！这个base_model.py要保存在根目录下面的数据库文件夹中，别忘了DB文件夹中的__init__的.py。

＃书店/ DB / base_model.py 
从 django.db进口车型

类 BaseModel（机型。型号）：
     '' '模型抽象基类''' 
    is_delete = models.BooleanField（默认值= 假，verbose_name = '删除标记'）
    create_time = models.DateTimeField（auto_now_add = True，verbose_name = '创建时间'）
    update_time = models.DateTimeField（auto_now = True，verbose_name = '更新时间'）

    class  Meta：
        abstract =  True
然后我们针对用户设计一张表出来。代码以下写入users/models.py文件中。

从 db.base_model 导入 BaseModel

class  Passport（BaseModel）：
     '''用户模型类''' 
    username = models.CharField（max_length = 20，unique = True，verbose_name = '用户名称'）
    password = models.CharField（max_length = 40，verbose_name = '用户密码'）
    email = models.EmailField（verbose_name = '用户邮箱'）
    is_active = models.BooleanField（default = False，verbose_name = '激活状态'）

    ＃用户表的管理器 
    objects = PassportManager（）

    class  Meta：
        db_table =  ' s_user_account '
接下来我们在PassportManager（）中实现添加和查找账户信息的功能，这样抽象性更好。这个下面类需要写在Passport的上面。

来自 utils.get_hash import get_hash

＃在这里创建你的模型。
类 PassportManager（车型。经理）：
    高清 add_one_passport（自我，用户名，密码，电子邮件）：
         '' '添加一个账户信息''' 
        护照 =  自我 .create（用户名=用户名，密码= get_hash（密码），电子邮件=电子邮件）
        ＃ 3。返回护照
        返还护照

    def  get_one_passport（self，username，password）：
         '''根据用户名密码查找账户的信息'''
        尝试：
            passport =  self .get（用户名=用户名，密码= get_hash（密码））
         除了 self .model.DoesNotExist：
             ＃账户不存在 
            护照=  无
        回复护照
我们这里有一个get_hash函数，这个函数用来避免存储明文密码。所以我们来编写这个函数。在根目录新建一个文件夹utils的，用来存放功能函数，比如get_hash，别忘了__init__.py文件。

＃书店/ utils的/ get_hash.py 
从 hashlib进口 SHA1

def  get_hash（str）：
     '''取一个字符串的哈希值''' 
    sh = sha1（）
    sh.update（str .encode（' utf8 '））
     返回 sh.hexdigest（）
接下来我们将用户的表映射到数据库中去。

mysql> create database bookstore charset=utf8;
$ python manage.py makemigrations users
$ python manage.py migrate
2，用户系统前端模板编写
接下来我们要将前端模板写一下，先建一个模板文件夹。

$ mkdir templates
我们第一个实现的功能是渲染注册页。将register.html拷贝到templates / users。将js，css，images文件夹拷贝到静态文件夹下。作为静态文件。接下来我们将程序跑起来。看到了Django的的欢迎页面。

$ python manage.py runserver 9000
然后呢？我们想把register.html渲染出来。我们先来看views.py这个视图文件。

＃用户/ views.py中
DEF  寄存器（请求）：
     '' '显示用户注册页面'''
    返回渲染（请求， '用户/ register.html '）
然后我们将URL映射做好。主应用的urls.py为

＃书店/ urls.py 
urlpatterns的 = [
    URL（[R ' ^管理员/ '，包括（admin.site.urls）），
    URL（[R ' ^用户/ '，包括（' users.urls '，命名空间= '用户'））， ＃用户模块 
]
'^'表示只匹配user为开头的url。然后在用户app里面的urls配置url映射。

＃用户/ urls.py 
从 django.conf.urls导入 URL
从用户输入的意见

urlpatterns = [
    url（r ' ^ register / $ '，views.register，name = ' register '），＃用户注册 
]
将模板的路径写入配置文件中。

＃ settings.py 
TEMPLATES  = [
    {
        ...... 
        ' DIRS '：[os.path.join（BASE_DIR，' templates '）]，＃这里别忘记配置！
        ...
    }，
]
我们可以看到静态文件没有加载出来，所以我们要改一下html文件中的路径。将静态文件的路径前面加上'/ static /'然后在配置文件中加入调试时使用的静态文件目录。

＃ settings.py 
STATICFILES_DIRS  = [
    os.path.join（BASE_DIR，' static '）
] ＃调试时使用的静态文件目录
好。我们渲染注册页的任务就完成了。

3，注册页面表单提交功能
接下来我们要编写注册页面的提交表单功能.1，接受前端传过来的表单数据.2，校验数据.3，返回注册页（因为还没做首页）。

＃用户/ views.py 
DEF  register_handle（请求）：
     '' '进行用户注册处理''' 
    ＃接收数据 
    的用户名 =请求。POST .get（ ' user_name '）
    密码=请求。POST .get（' pwd '）
    电子邮件=请求。POST .get（' email '）

    ＃进行数据校验
    如果 不是 全部（[用户名，密码，电子邮件]）：
        ＃有数据为空
        返回渲染（请求， ' users / register.html '，{ ' errmsg '： '参数不能为空！' }）

    ＃判断邮箱是否合法
    如果 不是 re.match（ r ' ^ [ a-z0-9 ] [ \ w \。\  - ] * @ [ a-z0-9 \  - ] + （\。 [ az ] {2， 5} ）{1,2} $ '，电子邮件）：
        ＃邮箱不合法
        return render（request， ' users / register.html '，{ ' errmsg '： '邮箱不合法！' }）

    ＃进行业务处理：注册，向账户系统中添加账户
    ＃ Passport.objects.create（用户名=用户名，密码=密码，电子邮件=电子邮件）
    尝试：
        Passport.objects.add_one_passport（用户名=用户名，密码=密码，电子邮箱=电子邮件），
     除了：
         return render（request，' users / register.html '，{ ' errmsg '：'用户名已存在！' }）

    ＃。注册完，还是返回注册页
    返回重定向（反向（ '用户：寄存器'））
配置urls.py

＃用户/ urls.py 
URL（ [R ' ^ register_handle / $ '，views.register_handle，名称= ' register_handle '）， ＃用户注册处理
前端使用表格来发送POST请求。

<form method="post" action="/user/register_handle/">
注意添加csrf_token以及错误信息

{% csrf_token %}
{{errmsg}}
然后就完成注册功能了。之后需要实现发送激活邮件。

3，书籍商品模块
1，渲染首页功能（书籍表结构设计）
完成注册功能以后，点击注册按钮应该跳转到首页。所以我们把首页index.html完成。先新建一个书籍app。

$ python manage.py startapp books
在配置文件中添加书籍应用程序

# bookstore/settings.py
INSTALLED_APPS = (
    ...
    'users', # 用户模块
    'books', # 商品模块
)
然后设计模型，也就是书模块的表结构。这里我们先介绍一下富文本编辑器，因为编辑商品详情页需要使用富文本编辑器。

富文本编辑器应用到项目中
在settings.py中为INSTALLED_APPS添加编辑器应用
＃ settings.py 
INSTALLED_APPS  =（
     ... 
    ' tinymce '，
）
在settings.py中添加编辑配置项
TINYMCE_DEFAULT_CONFIG  = {
     '主题'：'高级'，
     '宽度'：600，
     '高度'：400，
}
在根urls.py中配置
urlpatterns的= [
     ... 
    URL（[R ' ^ TinyMCE的/ '，包括（' tinymce.urls '）），
]
下面在我们的应用中添加富文本编辑器。然后我们就可以设计我们的商品表结构了，我们可以通过观察detail.html来设计表结构。我们先把一些常用的常量存储到books / enums.py文件中

PYTHON  =  1 
JAVASCRIPT  =  2 
算法 =  3 
MACHINELEARNING  =  4 
OPERATINGSYSTEM  =  5 
DATABASE  =  6

BOOKS_TYPE  = {
     PYTHON：' Python '，
     JAVASCRIPT：' Javascript '，
     ALGORITHMS：'数据结构与算法'，
     MACHINELEARNING：'机器学习'，
     OPERATINGSYSTEM：'操作系统'，
     数据库：'数据库'，
}

OFFLINE  =  0 
ONLINE  =  1

STATUS_CHOICE  = {
     OFFLINE：'下线'，
     在线：'上线' 
}
然后再来设计表结构：

从 db.base_model 进口 BaseModel
 从 tinymce.models 导入 HTMLField
 从 books.enums 输入 * 
＃这里创建您的模型。
类 图书（BaseModel）：
     '' '商品模型类''' 
    books_type_choices =（（K，V）为 K，V 在 BOOKS_TYPE .items（））
    status_choices =（（k，v）for k，v in  STATUS_CHOICE .items（））
    type_id = models.SmallIntegerField（默认= PYTHON，choices = books_type_choices，verbose_name = '商品种类'）
    name = models.CharField（max_length = 20，verbose_name = '商品名称'）
    desc = models.CharField（max_length = 128，verbose_name = '商品简介'）
    价格= models.DecimalField（max_digits = 10，decimal_places = 2，verbose_name = '商品价格'）
    unit = models.CharField（max_length = 20，verbose_name = '商品单位'）
    stock = models.IntegerField（默认= 1，verbose_name = '商品库存'）
    sales = models.IntegerField（默认值= 0，verbose_name = '商品销量'）
    detail = HTMLField（verbose_name = '商品详情'）
    image = models.ImageField（upload_to = ' books '，verbose_name = '商品图片'）
    status = models.SmallIntegerField（默认= ONLINE，choices = status_choices，verbose_name = '商品状态'）

    objects = BooksManager（）

    ＃管理员显示书籍的名字
    高清 __str__（个体经营）：
        回归 自我。名称

    class  Meta：
        db_table =  ' s_books ' 
        verbose_name =  '书籍' 
        verbose_name_plural =  '书籍'
同样，我们这里再写一下BooksManager（），有一些基本功能在这里抽象出来。

类 BooksManager（车型。经理）：
     '' '商品模型管理器类''' 
    ＃排序= '新'按照创建时间进行排序
    ＃排序= '热'按照商品销量进行排序
    ＃排序= '价格'按照商品的价格进行排序
    ＃排序=按照默认顺序排序
    高清 get_books_by_type（自我，TYPE_ID，上限= 无，排序= '默认'）：
         '' '根据商品类型ID查询商品信息'''
        如果排序==  '新'：
            order_by =（' - create_time '，）
         elif sort ==  ' hot '：
            order_by =（' - sale '，）
         elif sort ==  ' price '：
            order_by =（' price '，）
         else：
            order_by =（' - pk '，）＃按照主键降序排列。

        ＃查询数据 
        books_li =  自 .filter（ TYPE_ID = TYPE_ID）.order_by（ * ORDER_BY）

        ＃查询结果集的限制
        if limit：
            books_li = books_li [：limit]
         return books_li

    def  get_books_by_id（self，books_id）：
         '''根据商品的id获取商品信息'''
        尝试：
            books =  self .get（id = books_id）
         除了 self .model.DoesNotExist：
             ＃不存在商品信息 
            books =  无
        归还书籍
做数据库迁移。

$ python manage.py makemigrations books
$ python manage.py migrate
好，接下来我们就可以将首页的index.html渲染出来了。现在根urls.py中配置URL。

url（r ' ^ '，include（' books.urls '，namespace = ' books '）），＃商品模块
然后在books app中的urls.py中配置url。

来自 django.conf.urls 从 books 导入视图导入 url


urlpatterns = [
    URL（[R ' ^ $ '，views.index，名字= '索引'）， ＃首页 
]
2，编写书籍表视图函数。
然后编写视图文件views.py。

＃书籍/ views.py 
从 django.shortcuts导入已呈现
从 books.models进口图书
从 books.enums导入已 * 
从 django.core.urlresolvers导入逆向
从 django.core.paginator进口分页程序
＃在这里创建你的意见。


def  index（request）：
     '''显示首页''' 
    ＃查询每个种类的3个新品信息和4个销量最好的商品信息 
    python_new = Books.objects.get_books_by_type（PYTHON，limit = 3，sort = '新的'）
    python_hot = Books.objects.get_books_by_type（PYTHON，limit = 4，sort = ' hot '）
    javascript_new = Books.objects.get_books_by_type（JAVASCRIPT，limit =  3，sort = ' new '）
    javascript_hot = Books.objects.get_books_by_type（JAVASCRIPT，limit = 4，sort = ' hot '）
    algorithms_new = Books.objects.get_books_by_type（ALGORITHMS，3，sort = ' new '）
    algorithms_hot = Books.objects.get_books_by_type（ALGORITHMS，4，sort = ' hot '）
    machinelearning_new = Books.objects.get_books_by_type（机器学习，3，排序= '新'）
    machinelearning_hot = Books.objects.get_books_by_type（机器学习，4，排序= '热'）
    operatingsystem_new = Books.objects.get_books_by_type（OPERATINGSYSTEM，3，sort = ' new '）
    operatingsystem_hot = Books.objects.get_books_by_type（OPERATINGSYSTEM，4，sort = ' hot '）
    database_new = Books.objects.get_books_by_type（DATABASE，3，sort = ' new '）
    database_hot = Books.objects.get_books_by_type（DATABASE，4，sort = ' hot '）

    ＃定义模板 
    上下文context = {
         ' python_new '：python_new，
         ' python_hot '：python_hot，
         ' javascript_new '：javascript_new，
         ' javascript_hot '：javascript_hot，
         ' algorithms_new '：algorithms_new，
         ' algorithms_hot '：algorithms_hot，
         ' machinelearning_new '：machinelearning_new，
         ' machinelearning_hot '：machinelearning_hot，
        ' operatingsystem_new '：operatingsystem_new，
         ' operatingsystem_hot '：operatingsystem_hot，
         ' database_new '：database_new，
         ' database_hot '：database_hot，
    }
    ＃使用模板
    返回渲染（请求， '书籍/ index.html的'，上下文）
然后将index.html的拷贝到模板/书籍。

3，将书籍注册到后台管理系统管理
再将书籍这个模型注册到管理里面，方便管理，可以用来在后台编辑商品信息。

＃书籍/ admin.py 
从 django.contrib中输入管理员
从 books.models进口图书
＃注册您的模型在这里。

admin.site.register（Books）＃在admin中添加有关商品的编辑功能。
然后我们创建超级管理员账户。

$ python manage.py createsuperuser
这样我们就可以登陆管理商品信息了，管理功能是django的杀手锏之一。接下来我们要把index.html中的文件路径改一下，要不然显示不出来。我们通过分析模板来改写成后端渲染出来的模板比如：

{ ％ 为书在 python_new ％ }
     < A HREF = “＃” > {{book.name}} < /一> 
{ ％ ENDFOR ％ }
替换用来掉index.html中的：

< 一个 HREF = “＃” >的Python核心编程</ 一 >
< 一个 HREF = “＃” >笨办法学的Python </ 一 >
< 一个 HREF = “＃” > Python的学习手册</ 一 >
用代码：

{％for book in python_hot％}
    < li >
        < H4 > < 一个 HREF = “＃” > {{book.name}} </ 一 > </ H4 >
        < 一个 HREF = “＃” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
        < div  class = “ prize ” >¥{{book.price}} </ div >
    </ li >
{％endfor％}
掉替换index.html中的：

< li >
    < H4 > < 一个 HREF = “＃” >的Python核心编程</ 一 > </ H4 >
    < 一个 HREF = “＃” > < IMG  SRC = “图像/书/ book001.jpg ” > </ 一 >
    < div  class = “ prize ” >¥30.00 </ div >
</ li >
< li >
    < H4 > < 一个 HREF = “＃” > Python的学习手册</ 一 > </ H4 >
    < 一个 HREF = “＃” > < IMG  SRC = “图像/书/ book002.jpg ” > </ 一 >
    < div  class = “ prize ” >¥5.50 </ div >
</ li >
< li >
    < H4 > < 一个 HREF = “＃” > Python的食谱</ 一 > </ H4 >
    < 一个 HREF = “＃” > < IMG  SRC = “图像/书/ book003.jpg ” > </ 一 >
    < div  class = “ prize ” >¥3.90 </ div >
</ li >
< li >
    < H4 > < 一个 HREF = “＃” > Python的高性能编程</ 一 > </ H4 >
    < 一个 HREF = “＃” > < IMG  SRC = “图像/书/ book004.jpg ” > </ 一 >
    < div  class = “ prize ” >¥25.80 </ div >
</ li >
由于我们在编辑商品信息时，需要上传书籍的图片，所以在配置文件中设置图片存放目录。

＃ settings.py 
MEDIA_ROOT  = os.path.join（ BASE_DIR， “ static ”）
我们将'/ static / images / ...'的格式改成django里面推荐的用法。在index.html开头添加

{% load staticfiles %}
并改写静态文件URL格式为：

<img src="{% static book.image %}">
好，那我们首页的渲染工作也就完成了。

如图4所示，从注册页跳转到首页
接下来我们将register.html注册完以后，跳转到首页去。

# user/views.py
return redirect(reverse('books:index'))
做完注册和首页，我们可以来做登陆页面了。将login.html拷贝到templates / users /下。注意修改静态文件路径。然后编写/users/views.py文件中的登录功能。

def  login（request）：
     '''显示登录页面'''
    如果请求。COOKIES .get（“用户名”）：
        用户名=请求。COOKIES .get（“用户名”）
        检查=  “检查”
    其他：
        username =  ' ' 
        checked =  ' ' 
    context = {
         ' username '：username，
         ' checked '：选中，
    }

    return render（request，' users / login.html '，context）
将URL配置到urls.py

url（r ' ^ login / $ '，views.login，name = ' login '）＃显示登陆页面
好，我们显示登陆页面也做完了。

5，登陆功能的实现。
我们先来实现登录数据校验的功能。还有记住用户名的功能。

＃用户/ views.py 
DEF  login_check（请求）：
     '' '进行用户登录校验''' 
    ＃ 1，获取数据 
    的用户名 =请求。POST .get（ ' username '）
    密码=请求。POST .get（' password '）
    记住=请求。POST .get（'记住'）

    ＃ 2.数据校验
    如果 不是 全部（[用户名，密码，记住]）：
        ＃有数据为空
        返回 JsonResponse（{ ' res '： 2 }）

    ＃ 3.进行处理：根据用户名和密码查看账户信息 
    passport = Passport.objects.get_one_passport（用户名=用户名，密码=密码）

    如果护照：
        next_url = reverse（' books：index '）＃ / user / 
        jres = JsonResponse（{ ' res '：1，' next_url '：next_url}）

        ＃判断是否需要
        记住用户名if remember ==  ' true '：
            ＃记住用户名 
            jres.set_cookie（ ' username '，username， max_age = 7 * 24 * 3600）
        否则：
            ＃不要 
            记住用户名 jres.delete_cookie （ '用户名'）

        ＃记住用户的登录状态 
        request.session [ ' islogin ' ] =  真的 
        request.session [ '用户名' ] =用户名
        request.session [ ' passport_id ' ] = passport.id
         return jres
     else：
         ＃用户名或密码错误
        返回 JsonResponse（{ ' res '：0 }）
我们在前端编写一段发送ajax post请求的html代码和jquery代码。将下面这段代码替换掉表单代码

< form >
  ...
</ form >
    {％csrf_token％}
    < input  type = “ text ”  id = “ username ”  class = “ name_input ”  value = “ {{username}} ”  placeholder = “请输入用户名” >
    < div  class = “ user_error ” >输入错误</ div >
    < input  type = “ password ”  id = “ pwd ”  class = “ pass_input ”  placeholder = “请输入密码” >
    < div  class = “ pwd_error ” >输入错误</ div >
    < div  class = “ more_input clearfix ” >
        < input  type = “ checkbox ”  name = “ remember ”  {{  checked  }} >
        < label >记住用户名</ label >
        < 一个 HREF = “＃” >忘记密码</ 一 >
    </ div >
    < input  type = “ button ”  id = “ btnLogin ”  value = “登录”  class = “ input_submit ” >
    < script  src = “ {％static'js / jquery-1.12.4.min.js'％} ” > </ script >
    < script >
        $（function（）{ 
$（'＃btnLogin '）。click（function（）{ var username = $（“＃username ”）。val（）var password = $（“＃pwd ”）。val（）var remember = $（' input [name =“remember”] '）。prop（' checked '）var            
                 
                 
                 
                csrfmiddlewaretoken =  $（' input [name =“csrfmiddlewaretoken”] '）。val（）

                var params = { 
                    username ： username，
                    password ： password，
                    remember ： remember，
                    csrfmiddlewaretoken ： csrfmiddlewaretoken 
                } 
$。交（' /用户/ login_check / '，则params，功能（数据）{ //用户名密码错误{ 'RES'：0} //登录成功{ 'RES'：1} 如果（数据。RES == 1） {                
                    
                    
                      
                        //跳转页面
位置。href = 数据。next_url ;                     }否则如果（数据。 RES == 2）{警报（ “数据不完整”）;                     }否则如果（数据。 RES == 0）{警报（ “用户名或者密码错误”）;                     }                }）            }）        }）</script                          
   
                        
   
                        




    >
配置urls.py

url（r ' ^ login_check / $ '，views.login_check，name = ' login_check '），＃用户登录校验
6，首页上的登陆和注册按钮重定向到相关页面。
< 一个 HREF = “ {％URL '用户：登录' ％} ” >登录</ 一 >
< span > | </ span >
< 一个 HREF = “ {％URL '用户：寄存器' ％} ” >注册</ 一 >
现在登陆以后还是显示登陆和注册，应该显示用户名才对。我们来修改一下index.html的。这里就会用到会话这个概念了。顺便把注销功能也实现了。

＃ / user / logout 
def  logout（ request）：
     '''用户退出登录''' 
    ＃清空用户的会话信息
    request.session.flush（）
    ＃跳转到首页
    返回重定向（反向（ ' books：index '））
然后改写的index.html

{ ％ if request.session.islogin ％ }
 < div class = “ login_btn fl ” > 
    欢迎您：< em > {{request.session.username}} < / em > 
    < span > | < /跨度> 
    < A HREF = “ {％URL '用户：注销' ％} ” >退出< /一> 
< / DIV >  < DIV 类= “ login_btn FL ” > 
    < A HREF = “ {％URL '用户：登录' ％} ” >登录< /一> 
    <跨度> | < /跨度> 
    < A HREF = “ {％URL '用户：寄存器' ％} ” >注册< /一> 
< / DIV > 
{ ％ ENDIF ％ }
配置urls.py

url（r ' ^ logout / $ '，views.logout，name = ' logout '），＃退出用户登录
7，实现商品详情页的功能
＃书籍/ views.py 
DEF  细节（请求， books_id）：
     '' '显示商品的详情页面''' 
    ＃获取商品的详情信息 
    书籍 = Books.objects.get_books_by_id（ books_id = books_id）

    如果书籍是 无：
         ＃商品不存在，跳转到首页
        返回重定向（反向（' books：index '））

    ＃新品推荐 
    books_li = Books.objects.get_books_by_type（ TYPE_ID = books.type_id，极限= 2，排序= '新'）
    
    ＃当前商品类型 
    type_title =  BOOKS_TYPE [books.type_id]
    
    ＃定义 
    上下文context = { ' books '：books， ' books_li '：books_li， ' type_title '：type_title}

    ＃使用模板
    返回渲染（请求， '书籍/ detail.html '，上下文）
配置urls.py

URL（[R '书籍/ （？P <books_id> \ d + ） / $ '，views.detail，名称= '细节'）， ＃详情页
将detail.html页面拷贝到templates / books下。然后将detail.html页面改写成django可以渲染的模板。

＃动态添加详情页商品的标签（全部商品下的）
{{type_title}}
< h3 > {{books.name}} </ h3 >
< p > {{books.desc}} </ p >
< div  class = “ price_bar ” >
    < span  class = “ show_pirze ” >¥< em > {{books.price}} </ em > </ span >
    < span  class = “ show_unit ” >单位：{{books.unit}} </ span >
</ div >
{％for books in books_li％}
< li >
    < 一个 HREF = “ {％URL '的书：详细' books_id = book.id％} ” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
    < H4 > < 一个 HREF = “ {％URL '的书：详细' books_id = book.id％} ” > {{book.name}} </ 一 > </ H4 >
    < div  class = “ price ” >¥{{book.price}} </ div >
</ li >
{％endfor％}
然后将静态文件的路径修改。比如：

<img src="{% static books.image %}">
然后将登陆后的用户名等显示出来。那商品详情页就开发的差不多了。

8，抽象出一个通用的模板，供别的模板继承。
{＃首页登录注册的父模板＃}
<！DOCTYPE html PUBLIC “ -  // W3C // DTD XHTML 1.0 Transitional // EN”  “http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd” >
< html  xmlns = “ http://www.w3.org/1999/xhtml ”  xml：lang = “ en ” >
{％load staticfiles％}
< 头 >
    < meta  http-equiv = “ Content-Type ”  content = “ text / html; charset = UTF-8 ” >
    {＃网页顶部标题块＃}
    < title > {％block title％} {％endblock title％} </ title >
    < link  rel = “ stylesheet ”  type = “ text / css ”  href = “ {％static'css / reset.css'％} ” >
    < link  rel = “ stylesheet ”  type = “ text / css ”  href = “ {％static'css / main.css'％} ” >
    < script  type = “ text / javascript ”  src = “ {％static'js / jquery-1.12.4.min.js'％} ” > </ script >
    < script  type = “ text / javascript ”  src = “ {％static'js / jquery-ui.min.js'％} ” > </ script >
    < script  type = “ text / javascript ”  src = “ {％static'js / slide.js'％} ” > </ script >
    {＃网页顶部引入文件块＃}
    {％block topfiles％}
    {％endblock topfiles％}
</ head >
< body >
{＃网页顶部欢迎信息块＃}
{％block header_con％}
    < div  class = “ header_con ” >
        < div  class = “ header ” >
            < div  class = “ welcome fl ” >欢迎来到尚硅谷书店！</ div >
            < div  class = “ fr ” >
                {％if request.session.islogin％}
                < div  class = “ login_btn fl ” >
                    欢迎您：< em > {{request.session.username}} </ em >
                    < span > | </ span >
                    < 一个 HREF = “ {％URL '用户：注销' ％} ” >退出</ 一 >
                </ div >
                {％else％}
                < div  class = “ login_btn fl ” >
                    < 一个 HREF = “ {％URL '用户：登录' ％} ” >登录</ 一 >
                    < span > | </ span >
                    < 一个 HREF = “ {％URL '用户：寄存器' ％} ” >注册</ 一 >
                </ div >
                {％ 万一 ％}
                < div  class = “ user_link fl ” >
                    < span > | </ span >
                    < 一个 HREF = “＃” >用户中心</ 一 >
                    < span > | </ span >
                    < 一个 HREF = “＃” >我的购物车</ 一 >
                    < span > | </ span >
                    < 一个 HREF = “＃” >我的订单</ 一 >
                </ div >
            </ div >
        </ div >      
    </ div >
{％endblock header_con％}
{＃网页顶部搜索框块＃}
{％block search_bar％}
    < div  class = “ search_bar clearfix ” >
        < 一个 HREF = “ {％URL '的书：索引' ％} ” 类 = “标志FL ” > < IMG  SRC = “ {％静态'图像/ logo.png' ％} ” 风格 = “ 宽度：160 像素 ; 高度：53 px ; “ > </ a >
        < div  class = “ search_con fl ” >
            < input  type = “ text ”  class = “ input_text fl ”  name = “ ”  placeholder = “搜索商品” >
            < input  type = “ button ”  class = “ input_btn fr ”  name = “ ”  value = “搜索” >
        </ div >
        < div  class = “ guest_cart fr ” >
            < 一个 HREF = “＃” 类 = “ cart_name FL ” >我的购物车</ 一 >
            < div  class = “ book_count fl ”  id = “ show_count ” > 0 </ div >
        </ div >
    </ div >
{％endblock search_bar％}
{＃网页主体内容块＃}
{％block body％} {％endblock body％}

    < div  class = “ footer ” >
        < div  class = “ foot_link ” >
            < 一个 HREF = “＃” >关于我们</ 一 >
            < span > | </ span >
            < 一个 HREF = “＃” >联系我们</ 一 >
            < span > | </ span >
            < 一个 HREF = “＃” >招聘人才</ 一 >
            < span > | </ span >
            < 一个 HREF = “＃” >友情链接</ 一 >
        </ div >
        < p > CopyRight©2016北京尚硅谷信息技术有限公司保留所有权利</ p >
        < p >电话：010  -  **** 888京ICP备******* 8号</ p >
    </ div >
    {＃网页底部html元素块＃}
    {％block bottom％} {％endblock bottom％}
    {＃网页底部引入文件块＃}
    {％block bottomfiles％} {％endblock bottomfiles％}
</ body >
</ html >
然后在别的模板中继承base.html，这就是抽象的好处。现在的组件化思路也是这样的，松耦合，紧内聚，复用的思路。比如改写register.html

{％extends'base.html'％}
{％load staticfiles％}
{％block title％}尚硅谷书城 - 注册{％endblock title％}
{％block topfiles％}
{％endblock topfiles％}
{％block header_con％} {％endblock header_con％}
{％block search_bar％} {％endblock search_bar％}
{％block body％}
    < div  class = “ register_con ” >
        < div  class = “ l_con fl ” >
            < 一 类 = “ reg_logo ” > < IMG  SRC = “ /static/images/logo.png ” 风格 = “ 宽度：160 像素 ; 高度：53 像素 ; ” > </ 一 >
            < div  class = “ reg_slogan ” >学计算机·来尚硅谷</ div >
            < div  class = “ reg_banner ” > </ div >
        </ div >

        < div  class = “ r_con fr ” >
            < div  class = “ reg_title clearfix ” >
                < h1 >用户注册</ h1 >
                < 一个 HREF = “＃” >登录</ 一 >
            </ div >
            < div  class = “ reg_form clearfix ” >
                < form  method = “ post ”  action = “ / user / register_handle / ” >
                    {％csrf_token％}
                < ul >
                    < li >
                        < label >用户名：</ label >
                        < input  type = “ text ”  name = “ user_name ”  id = “ user_name ” >
                        < span  class = “ error_tip ” >提示信息</ span >
                    </ li >
                    < li >
                        < label >密码：</ label >
                        < input  type = “ password ”  name = “ pwd ”  id = “ pwd ” >
                        < span  class = “ error_tip ” >提示信息</ span >
                    </ li >
                    < li >
                        < label >确认密码：</ label >
                        < input  type = “ password ”  name = “ cpwd ”  id = “ cpwd ” >
                        < span  class = “ error_tip ” >提示信息</ span >
                    </ li >
                    < li >
                        < label >邮箱：</ label >
                        < input  type = “ text ”  name = “ email ”  id = “ email ” >
                        < span  class = “ error_tip ” >提示信息</ span >
                    </ li >
                    < li  class = “ agreement ” >
                        < input  type = “ checkbox ”  name = “ allow ”  id = “ allow ”  checked = “ checked ” >
                        < label >同意“尚硅谷书城用户使用协议”</ label >
                        < span  class = “ error_tip2 ” >提示信息</ span >
                    </ li >
                    < li  class = “ reg_sub ” >
                        < 输入 类型 = “提交” 值 = “注册” 名称 = “ ” >
                        {{ERRMSG}}
                    </ li >
                </ ul >
                </ form >
            </ div >

        </ div >

    </ div >
{％endblock body％}
然后改写的login.html

{％extends'base.html'％}
{％load staticfiles％}
{％block title％}尚硅谷书城 - 登录{％endblock title％}
{％block topfiles％}
< script >
    $（function（）{ 
$（'＃btnLogin '）。click（function（）{ //获取用户名和密码var username = $（'＃username '）。val（）var password = $（'＃pwd '） 。VAL（）VAR CSRF = $（'输入[名称= “csrfmiddlewaretoken”] '）。VAL（）VAR记住        
            
             
             
             
            =  $（' input [name =“remember”] '）。prop（' checked '）
//发起ajax请求var params = { ' username '： username，' password '： password，' csrfmiddlewaretoken '： csrf，' remember '： remember             } $。post（' / user / login_check / '，params，            
            
                
                
                
                

            （数据）{ 
//用户名密码错误{ 'RES'：0} //登录成功{ 'RES'：1} 如果（数据。RES == 0）{ $（' #username '）。下一个（）。html（'用户名或密码错误'）。show（）                } else                 { //跳转页面位置。href = 数据。next_url // / user /                 }             }）        }）                
                
                  
                    

                

                    
                       



    }）
</ script >
{％endblock topfiles％}
{％block header_con％} {％endblock header_con％}
{％block search_bar％} {％endblock search_bar％}
{％block body％}
    < div  class = “ login_top clearfix ” >
        < 一个 HREF = “ index.html的” 类 = “ login_logo ” > < IMG  SRC = “ {％静态'图像/ logo.png' ％} ” 风格 = “ 宽度：160 像素 ; 高度：53 像素 ; ” > </ a >
    </ div >

    < div  class = “ login_form_bg ” >
        < div  class = “ login_form_wrap clearfix ” >
            < div  class = “ login_banner fl ” > </ div >
            < div  class = “ slogan fl ” >学计算机·来尚硅谷</ div >
            < div  class = “ login_form fr ” >
                < div  class = “ login_title clearfix ” >
                    < h1 >用户登录</ h1 >
                    < 一个 HREF = “＃” >立即注册</ 一 >
                </ div >
                < div  class = “ form_input ” >
                    {％csrf_token％}
                    < input  type = “ text ”  id = “ username ”  class = “ name_input ”  value = “ {{username}} ”  placeholder = “请输入用户名” >
                    < div  class = “ user_error ” >输入错误</ div >
                    < input  type = “ password ”  id = “ pwd ”  class = “ pass_input ”  placeholder = “请输入密码” >
                    < div  class = “ pwd_error ” >输入错误</ div >
                    < div  class = “ more_input clearfix ” >
                        < input  type = “ checkbox ”  name = “ remember ”  {{  checked  }} >
                        < label >记住用户名</ label >
                        < 一个 HREF = “＃” >忘记密码</ 一 >
                    </ div >
                    < input  type = “ button ”  id = “ btnLogin ”  value = “登录”  class = “ input_submit ” >
                </ div >
            </ div >
        </ div >
    </ div >
{％endblock body％}
改写的index.html

{％extends'base.html'％}
{％load staticfiles％}
{％block title％}尚硅谷书店 - 首页{％endblock title％}
{％block topfiles％}
{％endblock topfiles％}
{％block body％}

    < div  class = “ navbar_con ” >
        < div  class = “ navbar ” >
            < h1  class = “ fl ” >全部商品分类</ h1 >
            < ul  class = “ navlist fl ” >
                < 锂 > < 一个 HREF = “ ” >首页</ 一 > </ 李 >
                < li  class = “ interval ” > | </ li >
                < 锂 > < 一个 HREF = “ ” >移动端书城</ 一 > </ 李 >
                < li  class = “ interval ” > | </ li >
                < 锂 > < 一个 HREF = “ ” >秒杀</ 一 > </ 李 >
            </ ul >
        </ div >
    </ div >

    < div  class = “ center_con clearfix ” >
        < ul  class = “ subnav fl ” >
            < 锂 > < 一个 HREF = “＃model01 ” 类 = “蟒” > Python的</ 一 > </ 李 >
            < 锂 > < 一个 HREF = “＃model02 ” 类 = “ JavaScript的” >使用Javascript </ 一 > </ 李 >
            < 锂 > < 一个 HREF = “＃model03 ” 类 = “算法” >数据结构与算法</ 一 > </ 李 >
            < 锂 > < 一个 HREF = “＃model04 ” 类 = “机器学习” >机器学习</ 一 > </ 李 >
            < 锂 > < 一个 HREF = “＃model05 ” 类 = “ OperatingSystem的” >操作系统</ 一 > </ 李 >
            < 锂 > < 一个 HREF = “＃model06 ” 类 = “数据库” >数据库</ 一 > </ 李 >
        </ ul >
        < div  class = “ slide fl ” >
            < ul  class = “ slide_pics ” >
                < li > < img  src = “ {％static'images / slide.jpg'％} ”  alt = “幻灯片”  style = “ width：760 px ; height：270 px ; ” > </ li >
                < li > < img  src = “ {％static'images / slide02.jpg'％} ”  alt = “幻灯片”  style = “ width：760 px ; 身高：270 px ; ” > </ li >
                < li > < img  src = “ {％static'images / slide03.jpg'％} ”  alt = “幻灯片”  style = “ width：760 px ; height：270 px ; ” > </ li >
                < li > < img  src = “ {％static'images / slide04.jpg'％} ”  alt = “幻灯片”  style = “ width：760 px ; height：270 px ; ” > </ li >
            </ ul >
            < div  class = “ prev ” > </ div >
            < div  class = “ next ” > </ div >
            < ul  class = “ points ” > </ ul >
        </ div >
        < div  class = “ adv fl ” >
            < 一个 HREF = “＃” > < IMG  SRC = “ {％静态'图像/ adv01.jpg' ％} ” 风格 = “ 宽度：240 像素 ; 高度：135 像素 ; ” > </ 一 >
            < 一个 HREF = “＃” > < IMG  SRC = “ {％静态'图像/ adv02.jpg' ％} ” 风格 = “ 宽度：240 像素 ; 高度：135 像素 ; ” > </ 一 >
        </ div >
    </ div >

    < div  class = “ list_model ” >
        < div  class = “ list_title clearfix ” >
            < h3  class = “ fl ”  id = “ model01 ” > Python </ h3 >
            < div  class = “ subtitle fl ” >
                < span > | </ span >
                {％for book in python_new％}
                    < 一个 HREF = “ {％URL '的书：详细' books_id = book.id％} ” > {{book.name}} </ 一 >
                {％endfor％}
            </ div >
            < 一个 HREF = “＃” 类 = “ book_more FR ” >查看更多> </ 一 >
        </ div >

        < div  class = “ book_con clearfix ” >
            < div  class = “ book_banner fl ” > < img  src = “ {％static'images / banner01.jpg'％} ” > </ div >
            < ul  class = “ book_list fl ” >
                {％for book in python_hot％}
                < li >
                    < H4 > < 一个 HREF = “ {％URL '的书：详细' books_id = book.id％} ” > {{book.name}} </ 一 > </ H4 >
                    < 一个 HREF = “ {％URL '的书：详细' books_id = book.id％} ” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
                    < div  class = “ price ” >¥{{book.price}} </ div >
                </ li >
                {％endfor％}
            </ ul >
        </ div >
    </ div >

    < div  class = “ list_model ” >
        < div  class = “ list_title clearfix ” >
            < h3  class = “ fl ”  id = “ model02 ” > Javascript </ h3 >
            < div  class = “ subtitle fl ” >
                < span > | </ span >
                {％for book in javascript_new％}
                    < 一个 HREF = “＃” > {{book.name}} </ 一 >
                {％endfor％}
            </ div >
            < 一个 HREF = “＃” 类 = “ book_more FR ” >查看更多> </ 一 >
        </ div >

        < div  class = “ book_con clearfix ” >
            < div  class = “ book_banner fl ” > < img  src = “ {％static'images / banner02.jpg'％} ” > </ div >
            < ul  class = “ book_list fl ” >
                {％for book in javascript_hot％}
                < li >
                    < H4 > < 一个 HREF = “＃” > {{book.name}} </ 一 > </ H4 >
                    < 一个 HREF = “＃” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
                    < div  class = “ price ” >¥{{book.price}} </ div >
                </ li >
                {％endfor％}
            </ ul >
        </ div >
    </ div >

    < div  class = “ list_model ” >
        < div  class = “ list_title clearfix ” >
            < h3  class = “ fl ”  id = “ model03 ” >数据结构与算法</ h3 >
            < div  class = “ subtitle fl ” >
                < span > | </ span >
                算法_新百分之书的{％
                    < 一个 HREF = “＃” > {{book.name}} </ 一 >
                {％endfor％}
            </ div >
            < 一个 HREF = “＃” 类 = “ book_more FR ” >查看更多> </ 一 >
        </ div >

        < div  class = “ book_con clearfix ” >
            < div  class = “ book_banner fl ” > < img  src = “ {％static'images / banner03.jpg'％} ” > </ div >
            < ul  class = “ book_list fl ” >
                {％for algorithm in algorithms_hot％}
                < li >
                    < H4 > < 一个 HREF = “＃” > {{book.name}} </ 一 > </ H4 >
                    < 一个 HREF = “＃” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
                    < div  class = “ price ” >¥{{book.price}} </ div >
                </ li >
                {％endfor％}
            </ ul >
        </ div >
    </ div >

    < div  class = “ list_model ” >
        < div  class = “ list_title clearfix ” >
            < h3  class = “ fl ”  id = “ model04 ” >机器学习</ h3 >
            < div  class = “ subtitle fl ” >
                < span > | </ span >
                {％for machine in machinelearning_new％}
                    < 一个 HREF = “＃” > {{book.name}} </ 一 >
                {％endfor％}
            </ div >
            < 一个 HREF = “＃” 类 = “ book_more FR ” >查看更多> </ 一 >
        </ div >

        < div  class = “ book_con clearfix ” >
            < div  class = “ book_banner fl ” > < img  src = “ {％static'images / banner04.jpg'％} ” > </ div >
            < ul  class = “ book_list fl ” >
                {％for machine in machinelearning_hot％}
                < li >
                    < H4 > < 一个 HREF = “＃” > {{book.name}} </ 一 > </ H4 >
                    < 一个 HREF = “＃” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
                    < div  class = “ price ” >¥{{book.price}} </ div >
                </ li >
                {％endfor％}
            </ ul >
        </ div >
    </ div >

    < div  class = “ list_model ” >
        < div  class = “ list_title clearfix ” >
            < h3  class = “ fl ”  id = “ model05 ” >操作系统</ h3 >
            < div  class = “ subtitle fl ” >
                < span > | </ span >
                {％for operatingsystem_new％}
                    < 一个 HREF = “＃” > {{book.name}} </ 一 >
                {％endfor％}
            </ div >
            < 一个 HREF = “＃” 类 = “ book_more FR ” >查看更多> </ 一 >
        </ div >

        < div  class = “ book_con clearfix ” >
            < div  class = “ book_banner fl ” > < img  src = “ {％static'images / banner05.jpg'％} ” > </ div >
            < ul  class = “ book_list fl ” >
                {％for books operatingsystem_hot％}
                < li >
                    < H4 > < 一个 HREF = “＃” > {{book.name}} </ 一 > </ H4 >
                    < 一个 HREF = “＃” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
                    < div  class = “ price ” >¥{{book.price}} </ div >
                </ li >
                {％endfor％}
            </ ul >
        </ div >
    </ div >

    < div  class = “ list_model ” >
        < div  class = “ list_title clearfix ” >
            < h3  class = “ fl ”  id = “ model06 ” >数据库</ h3 >
            < div  class = “ subtitle fl ” >
                < span > | </ span >
                {％for book in database_new％}
                    < 一个 HREF = “＃” > {{book.name}} </ 一 >
                {％endfor％}
            </ div >
            < 一个 HREF = “＃” 类 = “ book_more FR ” >查看更多> </ 一 >
        </ div >

        < div  class = “ book_con clearfix ” >
            < div  class = “ book_banner fl ” > < img  src = “ {％static'images / banner06.jpg'％} ” > </ div >
            < ul  class = “ book_list fl ” >
                {％for database in database_hot％}
                < li >
                    < H4 > < 一个 HREF = “＃” > {{book.name}} </ 一 > </ H4 >
                    < 一个 HREF = “＃” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
                    < div  class = “ price ” >¥{{book.price}} </ div >
                </ li >
                {％endfor％}
            </ ul >
        </ div >
    </ div >

{％endblock body％}
改写detail.html

{％extends'base.html'％}
{％load staticfiles％}
{％block title％}尚硅谷书店 - 首页{％endblock title％}
{％block body％}
    < div  class = “ navbar_con ” >
        < div  class = “ navbar clearfix ” >
            < div  class = “ subnav_con fl ” >
                < h1 >全部商品分类</ h1 >
                < span > </ span >           
                < ul  class = “ subnav ” >
                    < 锂 > < 一个 HREF = “＃” 类 = “蟒” > Python的</ 一 > </ 李 >
                    < 锂 > < 一个 HREF = “＃” 类 = “ JavaScript的” >使用Javascript </ 一 > </ 李 >
                    < 锂 > < 一个 HREF = “＃” 类 = “算法” >数据结构与算法</ 一 > </ 李 >
                    < 锂 > < 一个 HREF = “＃” 类 = “机器学习” >机器学习</ 一 > </ 李 >
                    < 锂 > < 一个 HREF = “＃” 类 = “ OperatingSystem的” >操作系统</ 一 > </ 李 >
                    < 锂 > < 一个 HREF = “＃” 类 = “数据库” >数据库</ 一 > </ 李 >
                </ ul >
            </ div >
            < ul  class = “ navlist fl ” >
                < 锂 > < 一个 HREF = “ ” >首页</ 一 > </ 李 >
                < li  class = “ interval ” > | </ li >
                < 锂 > < 一个 HREF = “ ” >移动端书城</ 一 > </ 李 >
                < li  class = “ interval ” > | </ li >
                < 锂 > < 一个 HREF = “ ” >秒杀</ 一 > </ 李 >
            </ ul >
        </ div >
    </ div >

    < div  class = “ breadcrumb ” >
        < 一个 HREF = “＃” >全部分类</ 一 >
        < span >> </ span >
        < 一个 HREF = “＃” > Python的</ 一 >
        < span >> </ span >
        < 一个 HREF = “＃” >商品详情</ 一 >
    </ div >

    < div  class = “ book_detail_con clearfix ” >
        < div  class = “ book_detail_pic fl ” > < img  src = “ {％static books.image％} ” > </ div >

        < div  class = “ book_detail_list fr ” >
            < h3 > {{books.name}} </ h3 >
            < p > {{books.desc}} </ p >
            < div  class = “ price_bar ” >
                < span  class = “ show_pirze ” >¥< em > {{books.price}} </ em > </ span >
                < span  class = “ show_unit ” >单位：{{books.unit}} </ span >
            </ div >
            < div  class = “ book_num clearfix ” >
                < div  class = “ num_name fl ” >数量：</ div >
                < div  class = “ num_add fl ” >
                    < input  type = “ text ”  class = “ num_show fl ”  value = “ 1 ” >
                    < 一个 HREF = “ JavaScript的:; ” 类 = “添加FR ” > + </ 一 >
                    < 一个 HREF = “ JavaScript的:; ” 类 = “减去FR ” > - </ 一 >   
                </ div >
            </ div >
            < div  class = “ total ” >总价：< em > 100元</ em > </ div >
            < div  class = “ operate_btn ” >
                < 一个 HREF = “ JavaScript的:; ” 类 = “ buy_btn ” >立即购买</ 一 >
                < 一个 HREF = “ JavaScript的:; ” 类 = “ add_cart ”  ID = “ add_cart ” >加入购物车</ 一 >             
            </ div >
        </ div >
    </ div >

    < div  class = “ main_wrap clearfix ” >
        < div  class = “ l_wrap fl clearfix ” >
            < div  class = “ new_book ” >
                < h3 >新品推荐</ h3 >
                < ul >
                    {％for books in books_li％}
                    < li >
                        < 一个 HREF = “ {％URL '的书：详细' books_id = books.id％} ” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
                        < H4 > < 一个 HREF = “ {％URL '的书：详细' books_id = books.id％} ” > {{book.name}} </ 一 > </ H4 >
                        < div  class = “ price ” >¥{{book.price}} </ div >
                    </ li >
                    {％endfor％}
                </ ul >
            </ div >
        </ div >

        < div  class = “ r_wrap fr clearfix ” >
            < ul  class = “ detail_tab clearfix ” >
                < li  class = “ active ” >商品介绍</ li >
                < li >评论</ li >
            </ ul >

            < div  class = “ tab_content ” >
                < dl >
                    < dt >商品详情：</ dt >
                    < dd > {{books.detail | 安全}} </ dd >
                </ dl >
            </ div >

        </ div >
    </ div >
    < div  class = “ add_jump ” > </ div >
{％endblock body％}
这里要注意的是重写也就是复写的问题，其实是面向对象的思想。

9，列表页的开发
接下来我们来实现列表页。通过观察list.html我们可以发现，这里有分页的功能，以及按照不同特征来进行排序，比如价格，比如人气这样的特征。这里要注意分页功能的实现，以及为什么要分页，一下读取所有数据，数据库的压力很大。


来自 django.core.paginator import Paginator的＃商品种类页码排序方式＃ / list /（种类id）/（页码）/？sort =排序方式


def  list（request，type_id，page）：
     '''商品列表页面''' 
    ＃获取排序方式 
    sort = request。GET .get（' sort '，' default '）

    ＃判断TYPE_ID是否合法
    ，如果 INT（TYPE_ID）未 在 BOOKS_TYPE .keys（）：
        返回重定向（逆转（ '书：指数'））

    ＃根据商品种类id和排序方式查询数据 
    books_li = Books.objects.get_books_by_type（ type_id = type_id， sort = sort）

    ＃分页 
    分页程序 =分页程序（books_li， 1）

    ＃获取分页之后的总页数 
    num_pages = paginator.num_pages

    ＃取第页数据页
    如果页 ==  ' ' 或 INT（页） > NUM_PAGES：
        page =  1 
    else：
        page =  int（页面）

    ＃返回值是一个Page类的实例对象 
    books_li = paginator.page（页）

    ＃进行页码控制
    ＃ 1.总页数<5，显示所有页码
    ＃ 2。当前页是前3页，显示1-5页
    ＃ 3.当前页是后3页，显示后5页10 9 8 7 
    ＃ 4.其他情况，显示当前页前2页，后2页，当前页
    如果 num_pages <  5：
        pages =  range（1，num_pages + 1）
     elif page <=  3：
        页=  范围（1，6）
     的elif NUM_PAGES -页面<=  2：
        pages =  range（num_pages - 4，num_pages + 1）
     else：
        页=  范围（页面- 2，页+ 3）

    ＃新品推荐 
    books_new = Books.objects.get_books_by_type（ TYPE_ID = TYPE_ID，极限= 2，排序= '新'）

    ＃定义上下文 
    type_title =  BOOKS_TYPE [ int（type_id）]
    context = {
         ' books_li '：books_li，
         ' books_new '：books_new，
         ' type_id '：type_id，
         ' sort '：sort，
         ' type_title '：type_title，
         ' pages '：pages
    }

    ＃使用模板
    返回渲染（请求， '书籍/ list.html '，上下文）
然后配置urls.py

＃书籍/ urls.py 
URL（ [R ' ^名单/ （？P <TYPE_ID> \ d + ） / （？P <页> \ d + ） / $ '，views.list，名称= '名单'）， ＃列表页
将list.html拷贝到templates / books先继承base.html。将index.html首页里面的查看更多，修改url定向。

{ ％ URL '的书：名单' TYPE_ID = 1页= 1  ％ }
修改名称。

{{ type_title }}
新品推荐。

{％for books in books_new％}
< li >
    < 一个 HREF = “ {％URL '的书：详细' books_id = book.id％} ” > < IMG  SRC = “ {％静态book.image％} ” > </ 一 >
    < H4 > < 一个 HREF = “ {％URL '的书：详细' books_id = book.id％} ” > {{book.name}} </ 一 > </ H4 >
    < div  class = “ price ” >¥{{book.price}} </ div >
</ li >
{％endfor％}
按不同的特征排序。

< 一个 HREF = “ /列表/ {{TYPE_ID}} / 1 / ”  {％ 如果 排序 == '缺省' ％}类别 = “活性” {％ ENDIF  ％} >默认</ 一 >
< 一个 HREF = “ /列表/ {{TYPE_ID}} / 1 /？排序=价格”  {％ 如果 排序 == '价格' ％}类别 = “活性” {％ ENDIF  ％} >价格</ 一 >
< 一个 HREF = “ /列表/ {{TYPE_ID}} / 1 /？排序=热”  {％ 如果 排序 == '热' ％}类别 = “活性” {％ ENDIF  ％} >人气</ 一 >
商品列表

{％for books in books_li％}
    < li >
        < 一个 HREF = “ {％URL '的书：详细' books_id = books.id％} ” > < IMG  SRC = “ {％静态books.image％} ” > </ 一 >
        < H4 > < 一个 HREF = “ {％URL '的书：详细' books_id = books.id％} ” > {{books.name}} </ 一 > </ H4 >
        < div  class = “ operations ” >
            < span  class = “ price ” >¥{{books.price}} </ span >
            < span  class = “ unit ” > {{books.unit}} </ span >
            < 一个 HREF = “＃” 类 = “ add_books ” 标题 = “加入购物车” > </ 一 >
        </ div >
    </ li >
{％endfor％}
前端分页功能的实现。

{％if books_li.has_previous％}
    < 一个 HREF = “ /列表/ {{TYPE_ID}} / {{books_li.previous_page_number}} /？排序= {{排序}} ” > <上一页</ 一 >
{％ 万一 ％}
{％for pindex in pages％}
    {％if pindex == books_li.number％}
        < 一个 HREF = “ /列表/ {{TYPE_ID}} / {{p食指}} /？排序= {{排序}} ” 类 = “活性” > {{p食指}} </ 一 >
    {％else％}
        < 一个 HREF = “ /列表/ {{TYPE_ID}} / {{p食指}} /？排序= {{排序}} ” > {{p食指}} </ 一 >
    {％ 万一 ％}
{％endfor％}
{％if books_li.has_next％}
    < 一个 HREF = “ /列表/ {{TYPE_ID}} / {{books_li.next_page_number}} /？排序= {{排序}} ” >下一页> </ 一 >
{％ 万一 ％}
如图4所示，用户中心的实现
接下来我们来实现用户中心的功能，先不实现最近浏览这个功能。首先来看一下这个前端页面，那我们知道我们还得给用户这个模型添加地址表。那我们先来建模型

＃用户/ models.py

class  Address（BaseModel）：
     '''地址模型类''' 
    recipient_name = models.CharField（max_length = 20，verbose_name = '收件人'）
    recipient_addr = models.CharField（max_length = 256，verbose_name = '收件地址'）
    zip_code = models.CharField （max_length = 6，verbose_name = '邮政编码'）
    recipient_phone = models.CharField（max_length = 11，verbose_name = '联系电话'）
    is_default = models.BooleanField（default = False，verbose_name = '是否默认'）
    passport = models.ForeignKey（' Passport '，verbose_name = '账户'）

    objects = AddressManager（）

    class  Meta：
        db_table =  ' s_user_address '
然后实现AddressManager（），对一些常用函数做抽象。

类 AddressManager（车型。经理）：
     '' '地址模型管理器类'''
    高清 get_default_address（自我，passport_id）：
         '' '查询指定用户的默认收货地址'''
        尝试：
            地址=  自获得（passport_id = passport_id，is_default = 真）
         除了 自我 .model.DoesNotExist：
            ＃没有默认收货地址 
            地址=  无
        返回地址

    def  add_one_address（self，passport_id，recipient_name，recipient_addr，zip_code，recipient_phone）：
         '''添加收货地址''' 
        ＃判断用户是否有默认收货地址 
        addr =  self .get_default_address（passport_id = passport_id）

        如果 addr：
             ＃存在默认地址 
            is_default =  False 
        否则：
             ＃不存在默认地址 
            is_default =  True

        ＃添加一个地址 
        addr =  self .create（ passport_id = passport_id，
                            recipient_name = recipient_name，
                            recipient_addr = recipient_addr，
                            zip_code = zip_code，
                            recipient_phone = recipient_phone，
                            is_default = is_default）
         return addr
进行数据库迁移

$ python manage.py makemigrations users
$ python manage.py migrate
然后编写视图函数views.py

def  user（request）：
     '''用户中心 - 信息页''' 
    passport_id = request.session.get（' passport_id '）
     ＃获取用户的基本信息 
    addr = Address.objects.get_default_address（passport_id = passport_id）

    books_li = []

    context = {
         ' addr '：addr，
         ' page '：' user '，
         ' books_li '：books_li
    }

    return render（request，' users / user_center_info.html '，context）
配置urls.py

url(r'^$', views.user, name='user'), # 用户中心-信息页
然后将user_center_info.html拷贝到templates / users文件夹下。还是继承base.html，然后对模板进行渲染。

< li > < span >用户名：</ span > {{request.session.username}} </ li >
{％if addr％}
    < li > < span >联系方式：</ span > {{addr.recipient_phone}} </ li >
    < li > < span >联系地址：</ span > {{addr.recipient_addr}} </ li >
{％else％}
    < li > < span >联系方式：</ span >无</ li >
    < li > < span >联系地址：</ span >无</ li >
{％ 万一 ％}
我们这里思考一下，用户中心必须登录以后才可以使用，我们应该怎么实现这个功能呢？在这里，python的装饰器功能就派上用场了。新建一个utils文件夹，用来存放自己编写的一些常用功能函数。

＃ utils的/ decorators.py 
从 django.shortcuts进口重定向
从 django.core.urlresolvers导入反向


def  login_required（view_func）：
     '''登录判断装饰器'''
     def  包装器（request，* view_args，** view_kwargs）：
         if request.session.has_key（' islogin '）：
             ＃用户已登录
            return view_func（request，* view_args，** view_kwargs）
         else：
             ＃跳转到登录页面
            return redirect（reverse（' user：login '））
    返回包装器
然后把这个装饰器打在views.py里的用户函数上面。

@login_required
装饰器有很多作用，比如可以限定具有某些权限的用户才能登陆，等等。

5，购物车功能
1，创建模型
这里我们要引进redis的使用。大家可以参考“redis实战”这本书，有很多redis的妙用，网上有电子版。我们使用redis实现购物车的功能。因为购物车里的数据相对不是那么重要，而且更新频繁。当然如果要是考虑到安全性，还是要持久化到数据库中比较好.redis在内存中不是很安全，比如之前redis就被攻击过。使用shell命令

ps -ef | grep redis
来检查redis server是否启动。我们现在配置文件中配置缓存有关的东西。

＃ settings.py 
＃ PIP安装Django-redis的
CACHES  = {
     “默认”：{
         “ BACKEND ”： “ django_redis.cache.RedisCache ”，
         “ LOCATION ”： “ redis的：//127.0.0.1：2分之6379 ”，
         “ OPTIONS “：{
             ” CLIENT_CLASS “： ” django_redis.client.DefaultClient “，
             ” PASSWORD “： ““
        }
    }
}

SESSION_ENGINE  =  “ django.contrib.sessions.backends.cache ”
 SESSION_CACHE_ALIAS  =  “默认”
然后新建一个购物车购物车app。

$ python manage.py startapp cart
然后开始编写视图函数。先来编写向购物车中添加商品的功能。

＃车/ views.py 
从 django.shortcuts进口呈现
从 django.http进口 JsonResponse
从 books.models进口图书
从 utils.decorators导入已 login_required
从 django_redis进口 get_redis_connection
＃这里创建你的意见。


＃前端发过来的数据：商品id商品数目books_id books_count 
＃涉及到数据的修改，使用post方式

@login_required 
def  cart_add（request）：
     '''向购物车中添加数据'''

    ＃接收数据 
    books_id = request。POST .get（ ' books_id '）
    books_count =请求。POST .get（' books_count '）

    ＃进行数据校验
    如果 不是 全部（[books_id，books_count]）：
         return JsonResponse（{ ' res '： 1， ' errmsg '： '数据不完整' }）

    books = Books.objects.get_books_by_id（books_id = books_id）
     如果书籍为 无：
         ＃商品不存在
        返回 JsonResponse（{ ' res '：2，' errmsg '：'商品不存在' }）

    尝试：
        count =  int（books_count）
     除了 Exception  为 e：
         ＃商品数目不合法
        return JsonResponse（{ ' res '：3，' errmsg '：'商品数量必须为数字' }）

    ＃添加商品到购物车
    ＃每个用户的购物车记录用一条哈希数据保存，格式：cart_用户id：商品id商品数量 
    conn = get_redis_connection（ ' default '）
    cart_key =  ' cart_ ％d ' ％ request.session.get（' passport_id '）

    res = conn.hget（cart_key，books_id）
     如果 res 为 None：
         ＃如果用户的购车中没有添加过该商品，则添加数据 
        res = count
     else：
         ＃如果用户的购车中已经添加过该商品，则累计商品数目 
        res =  int（res）+ count

    ＃判断商品的库存
    如果 res > books.stock：＃库存
        不足
        返回 JsonResponse（{ ' res '： 4， ' errmsg '： '商品库存不足' }）
    否则：
        conn.hset（cart_key，books_id，res）

    ＃返回结果
    返回 JsonResponse（{ ' res '： 5 }）
将商品添加到购物车的功能就实现了。

2，渲染购物车页面
在登陆以后，我们应该能够看到购物车里的商品数量，现在我们就来实现这个功能。

＃车/ views.py

@login_required 
def  cart_count（request）：
     '''获取用户购物车中商品的数目'''

    ＃计算用户购物车商品的数量 
    conn = get_redis_connection（ ' default '）
    cart_key =  ' cart_ ％d ' ％ request.session.get（' passport_id '）
     ＃ res = conn.hlen（cart_key）显示商品的条目数 
    res =  0 
    res_list = conn.hvals（cart_key）

    对于我在 res_list：
        res + =  int（i）

    ＃返回结果
    返回 JsonResponse（{ ' res '：res}）
然后在前端页面调用这个接口，并渲染出来。在base.html中添加：

＃base.html
    {＃获取用户购物车中商品的数目＃}
    {％block cart_count％}
        < script >
            $。得到（' /车/计数/ '，函数（数据）{ 
// { '水库'：商品的总数} $（' #show_count '。）HTML（数据。RES）            }） </脚本 >                
                

        
    {％endblock cart_count％}
而在登陆和注册页面，不需要显示这个，所以我们覆盖掉这个块。

{％block cart_count％} {％endblock cart_count％}
然后配置urls.py

    url(r'^add/$', views.cart_add, name='add'), # 添加购物车数据
    url(r'^count/$', views.cart_count, name='count'), # 获取用户购物车中商品的数量
然后在根目录urls.py中配置URL

    url(r'^cart/', include('cart.urls', namespace='cart')), # 购物车模块
在然后详情前端页detail.html关系编写添加到购物车的jQuery的代码。

< script  type = “ text / javascript ” >
    $（'＃add_cart '）。点击（函数（）{ 
//获取商品的ID和商品数量VAR books_id = $（本）。ATTR（' books_id '）; VAR books_count = $（' .nu​​m_show '）。VAL（）; VAR CSRF = $（' input [name =“csrfmiddlewaretoken”] '）。val（）; //发起请求，访问/ cart / add /，进行购物车数据的添加        
         
         
         
        
        var params = { 
' books_id '： books_id，' books_count '： books_count，' csrfmiddlewaretoken '： csrf         }            
            
            


        $。交（' /购物车/添加/ '，则params，功能（数据）{ 
如果（数据。RES == 5）{ //添加成功变种计数= $（' #show_count '）。HTML（）; VAR计数= parseInt函数（count）+ parseInt（books_count）; $（'＃show_count '）。html（count）;             }              
                
                 
                  
                
否则 { 
//添加失败警报（数据。ERRMSG）            }         }）    }）                
                



</ script >
发送ajax post请求来更新购物车数量。再改写html标签。

{% csrf_token %}
<a href="javascript:;" class="buy_btn">立即购买</a>
<a href="javascript:;" books_id="{{ books.id }}" class="add_cart" id="add_cart">加入购物车</a>
现在我们可以将商品添加到购物车中去了。然后我们再编写一段jquery代码，实现+/-商品数量的功能。并且可以自动更新总价格。

// detail.html
{％block topfiles％}
< script >
$（function（）{ 
update_total_price（）//计算总价函数update_total_price（）{ //获取商品的价格和数量         books_price = $（'。 show_pirze '）。children（' em '）。text（）        books_count = $（' .nu​​m_show '）。val（）//计算商品的总价         书籍_price = parseFloat（books_price）        books_count    
    
     
        
 
 
        
 
=  parseInt（books_count）
        total_price = books_price * books_count 
//设置商品总价$（'。 total '）。孩子们（' em '）。文本（TOTAL_PRICE。toFixed（2）+ '元'）    }        
         


    //商品增加
$（ '。 add '）。click（ function（）{ //获取商品的数量         books_count = $（ '。 num_show '）。 val（） //加1         books_count = parseInt（books_count） + 1 //重新设置值$（ '。 num_show '）。 val（books_count） //计算总价update_total_price（）    }）    
        
 
        
  
        
        
        
        


    //商品减少
$（ '。 minus '）。click（ function（）{ //获取商品的数量         books_count = $（ '。 num_show '）。 val（） //加1         books_count = parseInt（books_count） - 1 if（books_count == 0）{            books_count = 1         } / /重新设置值$（ '。 num_show '）。    
        
 
        
  
         
 

        
        val（books_count）
//计算总价update_total_price（）    }）        
        


    //手动输入
$（ '。 num_show '）。模糊（函数（）{ //获取商品的数量         books_count = $（本）。 VAL（） //数据校验，如果（ isNaN（books_count） || books_count。修剪（）。长度== 0 || parseInt函数（books_count ） <= 0）{            books_count = 1         } //重新设置值$    
        
 
        
              
 

        
        （' .nu​​m_show '）。val（parseInt（books_count））
//计算总价update_total_price（）    }）}）        
        


</ script >
{％endblock topfiles％}
好，添加商品到购物车的功能我们就实现好了。

3，购物车页面的开发
接下来我们来实现展示购物车页面的功能。编写views.py。

＃车/ views.py

@login_required 
def  cart_show（request）：
     '''显示用户购物车页面''' 
    passport_id = request.session.get（' passport_id '）
     ＃获取用户购物车的记录 
    conn = get_redis_connection（' default '）
    cart_key =  ' cart_ ％d ' ％ passport_id
    res_dict = conn.hgetall（cart_key）

    books_li = []
     ＃保存所有商品的总数 
    total_count =  0 
    ＃保存所有商品的总价格 
    total_price =  0

    ＃遍历res_dict获取商品的数据
    为 ID，算上在 res_dict.items（）：
        ＃根据ID获取商品的信息 
        书籍 = Books.objects.get_books_by_id（ books_id = ID）
        ＃保存商品的数目 
        books.count =计数
        ＃保存商品的小计 
        books.amount =  int（count） * books.price
        ＃ books_li.append（（books，count））
        books_li.append（书）

        total_count + =  int（count）
        total_price + =  int（count）* books.price

    ＃定义模板 
    上下文context = {
         ' books_li '：books_li，
         ' total_count '：total_count，
         ' total_price '：total_price，
    }

    return render（request，' cart / cart.html '，context）
然后配置urls.py

    url(r'^$', views.cart_show, name='show'), # 显示用户的购物车页面
然后将cart.html拷贝到templates / cart文件夹下面。还是继承base.html。然后在base.html中改写url定向。

{% url 'cart:show' %}
接下来，我们把订单列表渲染出来。

    {％for books in books_li％}
    < ul  class = “ cart_list_td clearfix ” >
        {＃提交表单时，如果复选框没有被选中，它的值不会被提交＃}
        < li  class = “ col01 ” > < input  type = “ checkbox ”  name = “ books_ids ”  value = “ {{book.id}} ”已 检查 > </ li >
        < li  class = “ col02 ” > < img  src = “ {％static book.image％} ” > </ li >
        < 锂 类 = “ col03 ” > {{book.name}} < BR > < EM > {{book.price}}元/ {{book.unit}} </ EM > </ 李 >
        < li  class = “ col04 ” > {{book.unit}} </ li >
        < li  class = “ col05 ” > {{book.price}} </ li >
        < li  class = “ col06 ” >
            < div  class = “ num_add ” >
                < 一个 HREF = “ JavaScript的:; ” 类 = “添加FL ” > + </ 一 >
                < input  type = “ text ”  books_id = “ {{book.id}} ”  class = “ num_show fl ”  value = “ {{book.count}} ” >
                < 一个 HREF = “ JavaScript的:; ” 类 = “减去FL ” > - </ 一 >   
            </ div >
        </ li >
        < li  class = “ col07 ” > {{book.amount}}元</ li >
        < 锂 类 = “ col08 ” > < 一个 HREF = “ JavaScript的:; ” >删除</ 一 > </ 李 >
    </ ul >
    {％endfor％}
4，购物车中删除商品的功能
先来编写views.py函数。

＃车/ views.py 
＃前端传过来的参数：商品编号books_id 
＃职位
＃ /车/删除/

@login_required 
def  cart_del（request）：
     '''删除用户购物车中商品的信息'''

    ＃接收数据 
    books_id = request。POST .get（ ' books_id '）

    ＃校验商品是否存放
    如果 不是 全部（[books_id]）：
         return JsonResponse（{ ' res '： 1， ' errmsg '： '数据不完整' }）

    books = Books.objects.get_books_by_id（books_id = books_id）
     如果书籍为 无：
         返回 JsonResponse（{ ' res '：2，' errmsg '：'商品不存在' }）

    ＃删除购物车商品信息 
    conn = get_redis_connection（ ' default '）
    cart_key =  ' cart_ ％d ' ％ request.session.get（' passport_id '）
    conn.hdel（cart_key，books_id）

    ＃返回信息
    返回 JsonResponse（{ ' res '： 3 }）
配置urls.py

url(r'^del/$', views.cart_del, name='delete'), # 购物车商品记录删除
然后在购物车页面cart.html编写的jQuery代码来调用德尔接口。

{ ％ block topfiles ％ }
     < script > 
    $（function（）{
         update_cart_count（）
         //计算所有被选中商品的总价，总数目和商品的小计
        函数 update_total_price（）{
            total_count =  0 
            total_price =  0 
            //获取所有被选中的商品所在的ul元素
            $（'。。cart_list_td '）。找（'：已检查'）。父母（' ul '）。each（function（）{
                 //计算商品的小计 
                res_dict =  update_books_price（$（this））

                total_count + =  res_dict。books_count 
                total_price + =  res_dict。books_amount
            }）

            //设置商品的总价和总数目
            $（ '。 settlements '）。找到（ ' em '）。文本（ TOTAL_PRICE。 toFixed（ 2））
             $（ ' .settlements '）。找（ ' b '）。文字（total_count）
        }

        //计算商品的小计
        函数 update_books_price（ books_ul）{
             //获取每一个商品的价格和数量 
                books_price =  books_ul。孩子们（ ' .col05 '）。文字（）
                books_count =  books_ul。找（' .nu​​m_show '）。val（）

                //计算商品的小计 
                books_price =  parseFloat（books_price）
                books_count =  parseInt（books_count）
                books_amount = books_price * books_count

                //设置商品的小计
                books_ul。孩子们（ ' .col07 '）。文本（ books_amount。 toFixed（ 2） +  '元'）

                返回 {
                     ' books_count '： books_count，
                     ' books_amount '： books_amount
                }
        }


        //更新页面上购物车商品的总数
        函数 update_cart_count（）{
            ＃更新列表上方商品总数
            $。得到（' /车/计数/ '，函数（数据）{
                 $（' .total_count '）。子女（' EM '）。文本（数据。RES）
            ＃更新页面右上方购物车商品总数
            $。得到（' /车/计数/ '，函数（数据）{
                 $（' #show_count '）。HTML（数据。RES）
            }）
        }

        //购物车商品信息的删除
        $（ '。 cart_list_td '）。孩子们（ ' .col08 '）。孩子（ ' a '）。点击（函数（）{
             //获取删除的商品的ID 
            books_ul =  $（本）。家长（ ' UL '）
            books_id =  books_ul。找（' .nu​​m_show '）。attr（' books_id '）
            csrf =  $（' input [name =“csrfmiddlewaretoken”] '）。val（）
            params = {
                 “ books_id ”： books_id，
                 “ csrfmiddlewaretoken ”： csrf
            }
            //发起ajax请求，访问/ cart / del / 
            $。交（ ' /购物车/ DEL / '，则params，功能（数据）{
                如果（数据。 RES  ==  3）{
                     //删除成功
                    //移除商品对应的UL元素
                    books_ul。除去（） // books.empty （）
                    //判断商品对应的复选框是否选中 
                    is_checked =  books_ul。找到（ '：复选框'。）丙（' checked '）
                     if（is_checked）{
                         update_total_price（）
                    }
                    //更新页面购物车商品总数
                    update_cart_count（）
                     //更新选择框状态
                    $（ '。 settlements '）。找（ “：复选框”）。道具（ '已检查'，假）
                }
            }）
        }）
    }）
    < / script > 
{ ％ endblock topfiles ％ }
为避免403，我们添加一行{％csrf_token％}

5，实现购物车页面编辑商品数量的功能。
我们先来编写更新购物车的接口。

＃车/ views.py 
＃前端传过来的参数：商品编号books_id更新数目books_count 
＃职位
＃ /车/更新/

@login_required 
def  cart_update（request）：
     '''更新购物车商品数目'''

    ＃接收数据 
    books_id = request。POST .get（ ' books_id '）
    books_count =请求。POST .get（' books_count '）

    ＃数据的校验
    如果 不是 全部（[books_id，books_count]）：
         return JsonResponse（{ ' res '： 1， ' errmsg '： '数据不完整' }）

    books = Books.objects.get_books_by_id（books_id = books_id）
     如果书籍为 无：
         返回 JsonResponse（{ ' res '：2，' errmsg '：'商品不存在' }）

    尝试：
        books_count =  int（books_count）
     除了 Exception  为 e：
         return JsonResponse（{ ' res '：3，' errmsg '：'商品数目必须为数字' }）

    ＃更新操作 
    conn = get_redis_connection（ ' default '）
    cart_key =  ' cart_ ％d ' ％ request.session.get（' passport_id '）

    ＃判断商品库存
    if books_count > books.stock：
         return JsonResponse（{ ' res '： 4， ' errmsg '： '商品库存不足' }）

    conn.hset（cart_key，books_id，books_count）

    返回 JsonResponse（{ ' res '：5 }）
注意别忘了配置cart app的urls.py.

url（r ' ^ update / $ '，views.cart_update，name = “ update ”）
然后编写jQuery的代码来实现前端更新数量以及全选这样的功能。

        //全选和全不选
        $（ '。 settlements '）。找（ '：复选框'）。变化（函数（）{
             //获取全选复选框的选中状态 
            is_checked =  $（本）。丙（ '检查'）

            //遍历所有商品对应的复选框，设置检查属性和全选复选框一致
            $（ '。。cart_list_td '）。找（ '：复选框'）。每个（函数（）{
                 $（本）。支撑（ '检查'，is_checked）
            }）

            //更新商品的信息
            update_total_price（）
        }）

        //商品对应的复选框状态发生改变时，全选checkbox的改变
        $（ '。。cart_list_td '）。找（ '：复选框'）。change（ function（）{
             //获取所有商品对应的复选框的数目 
            all_len =  $（ '。 cart_list_td '）。 find（ '：checkbox '）。 length 
            //获取所有被选中商品的复选框的数目 
            checked_len   =  $（ ' .cart_list_td '）。找（'：已检查'）。长度

            if（checked_len < all_len）{
                 $（'。 settlements '）。找（'：复选框'）。道具（'已检查'，假）
            }
            否则 {
                  $（'。 settlements '）。找（'：复选框'）。道具（'已检查'，真实）
            }

            //更新商品的信息
            update_total_price（）
        }）
        //购物车商品数目的增加
        $（ '。 add '）。点击（函数（）{
             //获取商品的数目和商品的ID 
            books_count =  $（本）。下一个（）。 VAL（）
            books_id =  $（this）。下一个（）。attr（' books_id '）

            //更新购物车信息 
            books_count =  parseInt（books_count） +  1 
            update_remote_cart_info（books_id，books_count）

            //根据更新的结果进行操作
            if（error_update ==  false）{
                 //更新成功
                $（ this）。下一个（）。val（books_count）
                 //获取商品对应的复选框的选中状态 
                is_checked =  $（ this）。父母（ ' ul '）。找（ '：复选框'）。prop（ ' checked '）
                 if（is_checked）{
                    //更新商品的总数目，总价格和小计
                    update_total_price（）
                }
                否则 {
                     //更新商品的小计
                    update_books_price（$（本）。家长（' UL '））
                }
                //更新页面购物车商品总数
                update_cart_count（）
            }
        }）

        //购物车商品数目的减少
        $（ '。 minus '）。点击（函数（）{
             //获取商品的数目和商品的ID 
            books_count =  $（本）。上一个（）。 VAL（）
            books_id =  $（this）。prev（）。attr（' books_id '）

            //更新购物车信息 
            books_count =  parseInt（books_count） -  1 
            if（books_count <=  0）{
                books_count =  1

            }

            update_remote_cart_info（books_id，books_count）

            //根据更新的结果进行操作
            if（error_update ==  false）{
                 //更新成功
                $（ this）。prev（）。val（books_count）
                 //获取商品对应的复选框的选中状态 
                is_checked =  $（ this）。父母（ ' ul '）。找（ '：复选框'）。prop（ ' checked '）
                 if（is_checked）{
                    //更新商品的总数目，总价格和小计
                    update_total_price（）
                }
                否则 {
                     //更新商品的小计
                    update_books_price（$（本）。家长（' UL '））
                }
                //更新页面购物车商品总数
                update_cart_count（）
            }
        }）

        pre_books_count =  0 
        $（'。 num_show '）。focus（function（）{
            pre_books_count =  $（this）。val（）
        }）

         //购物车商品数目的手动输入
        $（ '。 num_show '）。模糊（函数（）{
             //获取商品的数目和商品的ID 
            books_count =  $（本）。 VAL（）
            books_id =  $（this）。attr（' books_id '）

            //校验用户输入的商品数目
            如果（ isNaN（books_count） ||  books_count。修剪（）。长度 ==  0  ||  parseInt函数（books_count） <= 0）{
                 //设置回输入之前的值
                $（这） 。val（pre_books_count）
                返回
            }

            //更新购物车信息 
            books_count =  parseInt（books_count）

            update_remote_cart_info（books_id，books_count）

            //根据更新的结果进行操作
            if（error_update ==  false）{
                 //更新成功
                $（ this）。val（books_count）
                 //获取商品对应的复选框的选中状态 
                is_checked =  $（ this）。父母（ ' ul '）。找（ '：复选框'）。prop（ ' checked '）
                 if（is_checked）{
                     //更新商品的总数目，总价格和小计
                    update_total_price（）
                }
                否则 {
                     //更新商品的小计
                    update_books_price（$（本）。家长（' UL '））
                }
                //更新页面购物车商品总数
                update_cart_count（）
            }
            else {
                 //设置回输入之前的值
                $（this）。val（pre_books_count）
            }
        }）
这里要注意看看jquery是怎么发送ajax post请求的。

        //更新redis中购物车商品数目 
        error_update =  false 
        function  update_remote_cart_info（ books_id， books_count）{
            csrf =  $（' input [name =“csrfmiddlewaretoken”] '）。val（）
            params = {
                 ' books_id '： books_id，
                 ' books_count '： books_count，
                 ' csrfmiddlewaretoken '： csrf
            }
            //设置同步
            $。ajaxSettings。async  =  false 
            //发起请求，访问/ cart / update / 
            $。交（ ' /购物车/更新/ '，则params，功能（数据）{
                如果（数据。 RES  ==  5）{
                     //警报（ '更新成功'） 
                    error_update =  假
                }
                其他 {
                    error_update =  真
                    警报（数据。ERRMSG）
                }
            }）
            //设置异步
            $。ajaxSettings。async  =  true 
        }
这里设置了同步，因为我们需要马上更新购物车数据，即使阻塞了其他操作也在所不惜。好，购物车页面基本就开发完了。

6，订单页面的开发
1，创建模型
先来设计数据库表结构，记住表结构的设计是最重要的，因为一般情况下设计好表结构，就不再变更了。我们先安装订单应用。

$ python manage.py startapp order
然后订单信息模式的设计如下：

来自 django.db 从 db.base_model 导入模型
 导入 BaseModel
 ＃在这里创建模型。

class  OrderInfo（BaseModel）：
     '''订单信息模型类'''

    PAY_METHOD_CHOICES  =（
        （1，“货到付款”），
        （2，“微信支付”），
        （3，“支付宝”），
        （4，“银联支付”）
    ）

    PAY_METHODS_ENUM  = {
         “ CASH ”：1，
         “ WEIXIN ”：2，
         “ ALIPAY ”：3，
         “ UNIONPAY ”：4，
    }

    ORDER_STATUS_CHOICES  =（
        （1，“待支付”），
        （2，“待发货”），
        （3，“待收货”），
        （4，“待评价”），
        （5，“已完成”），
    ）

    order_id = models.CharField（max_length = 64，primary_key = True，verbose_name = '订单编号'）
    passport = models.ForeignKey（' users.Passport '，verbose_name = '下单账户'）
    addr = models.ForeignKey（' users.Address '，verbose_name = '收货地址'）
    total_count = models.IntegerField（默认值= 1，verbose_name = '商品总数'）
    TOTAL_PRICE = models.DecimalField（max_digits = 10，decimal_places = 2，verbose_name = '商品总价'）
    transit_price = models.DecimalField（max_digits = 10，decimal_places = 2，verbose_name = '订单运费'）
    pay_method = models.SmallIntegerField（choices = PAY_METHOD_CHOICES，default = 1，verbose_name = '支付方式'）
    status = models.SmallIntegerField（choices = ORDER_STATUS_CHOICES，default = 1，verbose_name = '订单状态'）
    trade_id = models.CharField （max_length = 100，unique = True，null = True，blank = True，verbose_name = '支付编号'）

    class  Meta：
        db_table =  ' s_order_info '
由于每一笔订单都是由不同的商品组成，所以我们需要把一笔订单拆分开，来建立一个订单中每种商品的信息数据表。关系数据库的一个好处就是强约束，冗余也很少，这点比MongoDB的好。

class  OrderGoods（BaseModel）：
     '''订单商品模型类''' 
    order = models.ForeignKey（' OrderInfo '，verbose_name = '所属订单'）
    books = models.ForeignKey（' books.Books '，verbose_name = '订单商品'）
    count = models.IntegerField（默认值= 1，verbose_name = '商品数量'）
    价格= models.DecimalField（max_digits = 10，decimal_places = 2，verbose_name = '商品价格'）
     ＃注释= models.CharField（MAX_LENGTH = 128，空=真，空白=真，verbose_name = '商品评论'）

    class  Meta：
        db_table =  ' s_order_books '
接下来我们来做数据库迁移。别忘了在settings.py中添加订购应用。

$ python manage.py makemigrations order
$ python manage.py migrate
2，开发有关订单的接口。
我们先把订单显示页面来渲染出来。先来开发后台接口。

从 django.shortcuts 导入已渲染，重定向
 从 django.core.urlresolvers 导入逆向
 从 utils.decorators 导入已 login_required
 从 django.http 进口的HttpResponse，JsonResponse
 从 users.models 导入地址
 从 books.models 进口图书
 从 order.models 进口订单信息，OrderGoods
 从 django_redis 进口 get_redis_connection
 从日期时间进口日期时间
 从django.conf 导入设置
 导入 os
 导入时间
 ＃在这里创建您的视图。


@login_required 
def  order_place（request）：
     '''显示提交订单页面''' 
    ＃接收数据 
    books_ids = request。POST .getlist（' books_ids '）

    ＃校验数据
    如果 不是 全部（books_ids）：
        ＃跳转会购物车页面
        返回重定向（反向（ '购物车：展示'））

    ＃用户收货地址 
    passport_id = request.session.get（ ' passport_id '）
    addr = Address.objects.get_default_address（passport_id = passport_id）

    ＃用户要购买商品的信息 
    books_li = []
    ＃商品的总数目和总金额 
    total_count =  0 
    total_price =  0

    conn = get_redis_connection（' default '）
    cart_key =  ' cart_ ％d ' ％ passport_id

    为 ID  在 books_ids：
        ＃根据ID获取商品的信息 
        书籍= Books.objects.get_books_by_id（books_id = ID）
         ＃从Redis的中获取用户要购买的商品的数目
        计数= conn.hget（cart_key，ID）
        books.count = count
         ＃计算商品的小计 
        金额=  int（count）* books.price
        books.amount =金额
        books_li.append（书）

        ＃累计计算商品的总数目和总金额 
        total_count + =  int（count）
        total_price + = books.amount

    ＃商品运费和实付款 
    transit_price =  10 
    total_pay = total_price + transit_price

    ＃ 1,2,3 
    books_ids =  '，'。 join（books_ids）
    ＃组织模板 
    上下文context = {
         ' addr '：addr，
         ' books_li '：books_li，
         ' total_count '：total_count，
         ' total_price '：total_price，
         ' transit_price '：transit_price，
         ' total_pay '：total_pay，
         ' books_ids '：books_ids，
    }

    ＃使用模板
    返回渲染（请求， '订单/ place_order.html '，上下文）
然后配置urls.py.

＃ urls.py 
    URL（ [R ' ^顺序/ '，包括（ ' order.urls '，命名空间= '顺序'））， ＃订单模块
＃订单/ urls.py 
从 django.conf.urls输入网址
从订单输入的意见

urlpatterns = [
    url（r ' ^ place / $ '，views.order_place，name = ' place '），＃订单提交页面 
]
然后将place_order.html拷贝到模板/订单文件夹下。并用继承base.html文件的方式来改写。

{％extends'base.html'％}
{％load staticfiles％}
{％block title％}尚硅谷书店 - 首页{％endblock title％}
{％block topfiles％}
{％endblock topfiles％}
{％block body％}
    
    < h3  class = “ common_title ” >确认收货地址</ h3 >

    < div  class = “ common_list_con clearfix ” >
        < dl >
            < dt >寄送到：</ dt >
            < dd > < input  type = “ radio ”  name = “ ”  checked = “ ” >北京市海淀区东北旺西路8号中关村软件园（李思收）182 **** 7528 </ dd >
        </ dl >
        < 一个 HREF = “ user_center_site.html ” 类 = “ edit_site ” >编辑收货地址</ 一 >

    </ div >
    
    < h3  class = “ common_title ” >支付方式</ h3 >  
    < div  class = “ common_list_con clearfix ” >
        < div  class = “ pay_style_con clearfix ” >
            < input  type = “ radio ”  name = “ pay_style ”  checked >
            < label  class = “ cash ” >货到付款</ label >
            < input  type = “ radio ”  name = “ pay_style ” >
            < label  class = “ weixin ” >微信支付</ label >
            < input  type = “ radio ”  name = “ pay_style ” >
            < label  class = “ zhifubao ” > </ label >
            < input  type = “ radio ”  name = “ pay_style ” >
            < label  class = “ bank ” >银行卡支付</ label >
        </ div >
    </ div >

    < h3  class = “ common_title ” >商品列表</ h3 >
    
    < div  class = “ common_list_con clearfix ” >
        < ul  class = “ book_list_th clearfix ” >
            < li  class = “ col01 ” >商品名称</ li >
            < li  class = “ col02 ” >商品单位</ li >
            < li  class = “ col03 ” >商品价格</ li >
            < li  class = “ col04 ” >数量</ li >
            < li  class = “ col05 ” >小计</ li >       
        </ ul >
        < ul  class = “ book_list_td clearfix ” >
            < li  class = “ col01 ” > 1 </ li >            
            < li  class = “ col02 ” > < img  src = “ images / book / book012.jpg ” > </ li >
            < li  class = “ col03 ” >计算机程序设计艺术</ li >
            < li  class = “ col04 ” >册</ li >
            < li  class = “ col05 ” > 25.80元</ li >
            < li  class = “ col06 ” > 1 </ li >
            < li  class = “ col07 ” > 25.80元</ li >   
        </ ul >
        < ul  class = “ book_list_td clearfix ” >
            < li  class = “ col01 ” > 2 </ li >
            < li  class = “ col02 ” > < img  src = “ images / book / book003.jpg ” > </ li >
            < li  class = “ col03 ” > Python Cookbook </ li >
            < li  class = “ col04 ” >册</ li >
            < li  class = “ col05 ” > 16.80元</ li >
            < li  class = “ col06 ” > 1 </ li >
            < li  class = “ col07 ” > 16.80元</ li >
        </ ul >
    </ div >

    < h3  class = “ common_title ” >总金额结算</ h3 >

    < div  class = “ common_list_con clearfix ” >
        < div  class = “ settle_con ” >
            < div  class = “ total_book_count ” >共< em > 2 </ em >件商品，总金额< b > 42.60元</ b > </ div >
            < div  class = “ transit ” >运费：< b > 10元</ b > </ div >
            < div  class = “ total_pay ” >实付款：< b > 52.60元</ b > </ div >
        </ div >
    </ div >

    < div  class = “ order_submit clearfix ” >
        < 一个 HREF = “ JavaScript的:; ”  ID = “ order_btn ”  books_ids = “ {{books_ids}} ” >提交订单</ 一 >
    </ div >
{％endblock body％}
{％block bottom％}
    < div  class = “ popup_con ” >
        < div  class = “ popup ” >
            < p >订单提交成功！</ p >
        </ div >
        < div  class = “ mask ” > </ div >
    </ div >
{％endblock bottom％}
然后将模板中的对应元素修改为后端渲染的代码。

{％extends'base.html'％}
{％load staticfiles％}
{％block title％}尚硅谷书店 - 我的订单{％endblock title％}
{％block topfiles％}
{％endblock topfiles％}
{％block body％}

    < h3  class = “ common_title ” >确认收货地址</ h3 >

    < div  class = “ common_list_con clearfix ” >
        < dl >
            < dt >寄送到：</ dt >
            < dd > < input  type = “ radio ”  name = “ addr_id ”  value = “ {{addr.id}} ”  checked = “ ” > {{addr.recipient_addr}}（{{addr.recipient_name}}收）{{ addr.recipient_phone}} </ dd >
        </ dl >
        < 一个 HREF = “ user_center_site.html ” 类 = “ edit_site ” >编辑收货地址</ 一 >

    </ div >

    < h3  class = “ common_title ” >支付方式</ h3 >
    < div  class = “ common_list_con clearfix ” >
        < div  class = “ pay_style_con clearfix ” >
            < input  type = “ radio ”  name = “ pay_style ”  checked >
            < label  class = “ cash ” >货到付款</ label >
            < input  type = “ radio ”  name = “ pay_style ” >
            < label  class = “ weixin ” >微信支付</ label >
            < input  type = “ radio ”  name = “ pay_style ” >
            < label  class = “ zhifubao ” > </ label >
            < input  type = “ radio ”  name = “ pay_style ” >
            < label  class = “ bank ” >银行卡支付</ label >
        </ div >
    </ div >

    < h3  class = “ common_title ” >商品列表</ h3 >
    < div  class = “ common_list_con clearfix ” >
        < ul  class = “ book_list_th clearfix ” >
            < li  class = “ col01 ” >商品名称</ li >
            < li  class = “ col02 ” >商品单位</ li >
            < li  class = “ col03 ” >商品价格</ li >
            < li  class = “ col04 ” >数量</ li >
            < li  class = “ col05 ” >小计</ li >
        </ ul >
        {％for books in books_li％}
        < ul  class = “ books_list_td clearfix ” >
            < li  class = “ col01 ” > {{forloop.counter}} </ li >
            < li  class = “ col02 ” > < img  src = “ {％static book.image％} ” > </ li >
            < li  class = “ col03 ” > {{book.name}} </ li >
            < li  class = “ col04 ” > {{book.unit}} </ li >
            < li  class = “ col05 ” > {{book.price}}元</ li >
            < li  class = “ col06 ” > {{book.count}} </ li >
            < li  class = “ col07 ” > {{book.amount}}元</ li >
        </ ul >
        {％endfor％}
    </ div >

    < h3  class = “ common_title ” >总金额结算</ h3 >

    < div  class = “ common_list_con clearfix ” >
        < div  class = “ settle_con ” >
            < div  class = “ total_book_count ” >共< em > {{total_count}} </ em >件商品，总金额< b > {{total_price}}元</ b > </ div >
            < div  class = “ transit ” >运费：< b > {{transit_price}}元</ b > </ div >
            < div  class = “ total_pay ” >实付款：< b > {{total_pay}}元</ b > </ div >
        </ div >
    </ div >

    < div  class = “ order_submit clearfix ” >
        < 一个 HREF = “ JavaScript的:; ”  ID = “ order_btn ”  books_ids = “ {{books_ids}} ” >提交订单</ 一 >
    </ div >
{％endblock body％}
{％block bottom％}
    < div  class = “ popup_con ” >
        < div  class = “ popup ” >
            < p >订单提交成功！</ p >
        </ div >
        < div  class = “ mask ” > </ div >
    </ div >
{％endblock bottom％}
那么订单显示页面就初步开发完了。

3，订单提交功能
接下来我们来开发提交订单的功能。先开发后端接口，这里要用到事务，交易，原子操作的概念。

＃顺序/ views.py 
＃提交订单，需要向两张表中添加信息
＃ s_order_info：订单信息表添加一条
＃ s_order_books：订单商品表，订单中买了几件商品，添加几条记录
＃前端需要提交过来的数据：地址支付方式购买的商品id

＃ 1向订单表中添加一条信息
＃ 2遍历向订单商品表中添加信息
    ＃ 2.1添加订单商品信息之后，增加商品销量，减少库存
    ＃ 2.2累计计算订单商品的总数目和总金额
＃ 3。更新订单商品的总数目和总金额
＃ 4. 清除购物车对应信息

＃。事务：原子性：一组SQL操作，要么都成功，要么都失败
＃开启事务：开始; 
＃事务回滚：rollback; 
＃事务提交：提交; 
＃设置保存点：savepoint保存点; 
＃回滚到保存点：rollback to保存点; 
来自 django.db导入事务

@ transaction.atomic 
def  order_commit（request）：
     '''生成订单''' 
    ＃验证用户是否登录
    如果 不是 request.session.has_key（' islogin '）：
         return JsonResponse（{ ' res '：0，' errmsg '：'用户未登录' }）

    ＃接收数据 
    addr_id = request。POST .get（ ' addr_id '）
    pay_method =请求。POST .get（' pay_method '）
    books_ids =请求。POST .get（' books_ids '）

    ＃进行数据校验
    如果 不是 全部（[addr_id，pay_method，books_ids]）：
         return JsonResponse（{ ' res '： 1， ' errmsg '： '数据不完整' }）

    尝试：
        addr = Address.objects.get（id = addr_id），
     除了 Exception  为 e：
         ＃地址信息出错
        return JsonResponse（{ ' res '：2，' errmsg '：'地址信息错误' }）

    如果 INT（pay_method）未 在订单信息。PAY_METHODS_ENUM .values（）：
         返回 JsonResponse（{ ' res '：3，' errmsg '：'不支持的支付方式' }）

    ＃订单创建
    ＃组织订单信息 
    passport_id = request.session.get（ ' passport_id '）
    ＃订单id：20171029110830+用户的id 
    order_id = datetime.now（）。strftime（ '％Y％m ％d％H％M％ S '） +  str（passport_id）
    ＃运费 
    transit_price =  10 
    ＃订单商品总数和总金额 
    total_count =  0 
    total_price =  0

    ＃创建一个保存点 
    sid = transaction.savepoint（）
    尝试：
        ＃向订单信息表中添加一条记录 
        order = OrderInfo.objects.create（ order_id = order_id，
                                  passport_id = passport_id，
                                  addr_id = addr_id，
                                  total_count = total_count，
                                  total_price = total_price ，
                                  transit_price = transit_price，
                                 pay_method = pay_method）

        ＃向订单商品表中添加订单商品的记录 
        books_ids = books_ids.split（ '，'）
        conn = get_redis_connection（' default '）
        cart_key =  ' cart_ ％d ' ％ passport_id

        ＃遍历获取用户购买的商品信息
        的 ID  在 books_ids：
            books = Books.objects.get_books_by_id（books_id = id）
             如果书籍为 无：
                transaction.savepoint_rollback（SID）
                return JsonResponse（{ ' res '：4，' errmsg '：'商品信息错误' }）

            ＃获取用户购买的商品数目 
            count = conn.hget（cart_key， id）

            ＃判断商品的库存
            if  int（count） > books.stock：
                transaction.savepoint_rollback（SID）
                return JsonResponse（{ ' res '：5，' errmsg '：'商品库存不足' }）

            ＃创建一条订单商品记录 
            OrderGoods.objects.create（ order_id = order_id，
                                       books_id = id，
                                       count = count，
                                       price = books.price）

            ＃增加商品的销量，减少商品库存 
            books.sales + =  int（count）
            books.stock -  =  int（count）
            books.save（）

            ＃累计计算商品的总数目和总额 
            total_count + =  int（count）
            total_price + =  int（count）* books.price

        ＃更新订单的商品总数目和总金额 
        order.total_count = total_count
        order.total_price = total_price
        order.save（）
    除 例外 为 E：
        ＃操作数据库出错，进行回滚操作
        transaction.savepoint_rollback（SID）
        return JsonResponse（{ ' res '：7，' errmsg '：'服务器错误' }）

    ＃ 
    清除购物车对应记录 conn.hdel（cart_key， * books_ids）

    ＃事务提交
    transaction.savepoint_commit（SID）
    ＃返回应答
    返回 JsonResponse（{ ' RES '： 6 }）
然后配置urls.py

    url（r ' ^ commit / $ '，views.order_commit，name = ' commit '），＃生成订单
然后改写前端页面place_order.html，来调用后端提交订单的接口。

{ ％ block bottomfiles ％ }
     < script type = “ text / javascript ” > 
        $（'＃order_btn '）。click（function（）{
             //获取收货地址的id，支付方式，用户购买的商品id 
            var addr_id =  $（' input [name =“addr_id”] '）。val（）
             var pay_method =  $（' input [名称= “pay_style”]：检查'）。VAL（）
             var books_ids =  $（this）。attr（' books_ids '）
             var csrf =  $（' input [name =“csrfmiddlewaretoken”] '）。val（）
             // alert（addr_id +'：'+ pay_method +'：'+ books_ids）
            //发起post请求，访问/ order / commit / 
            var params = {
                 ' addr_id '： addr_id，
                 ' pay_method '：pay_method，
                 ' books_ids '： books_ids，
                 ' csrfmiddlewaretoken '： csrf
            }
            $。交（' /订单/提交/ '，则params，功能（数据）{
                 //根据JSON进行处理
                ，如果（数据。RES  ==  6）{
                     $（' .popup_con '）。淡入（'快'，函数（） {
                         setTimeout（function（）{
                             $（'。 popup_con '）。淡出（'快'，函数（）{
                                 窗口。位置。HREF  =  ' /用户/订单/ ' ;
                            }）;
                        }，3000）

                    }）;
                }
                其他 {
                     警报（数据。ERRMSG）
                }
            }）

        }）;
    < / script > 
{ ％ endblock bottomfiles ％ }
那么，我们提交订单的功能也就开发好了。

如图4所示，接下来我们回过头去把购物车中的提交功能给做了，然后就能做支付功能了。
将cart.html中的去结算功能给出如下的实现。

    < form  method = “ post ”  action = “ / order / place / ” >
    {％for books in books_li％}
    < ul  class = “ cart_list_td clearfix ” >
        {＃提交表单时，如果复选框没有被选中，它的值不会被提交＃}
        < li  class = “ col01 ” > < input  type = “ checkbox ”  name = “ books_ids ”  value = “ {{book.id}} ”已 检查 > </ li >
        < li  class = “ col02 ” > < img  src = “ {％static book.image％} ” > </ li >
        < 锂 类 = “ col03 ” > {{book.name}} < BR > < EM > {{book.price}}元/ {{book.unit}} </ EM > </ 李 >
        < li  class = “ col04 ” > {{book.unit}} </ li >
        < li  class = “ col05 ” > {{book.price}} </ li >
        < li  class = “ col06 ” >
            < div  class = “ num_add ” >
                < 一个 HREF = “ JavaScript的:; ” 类 = “添加FL ” > + </ 一 >
                < input  type = “ text ”  books_id = {{  book.id  }}  class = “ num_show fl ”  value = “ {{book.count}} ” >
                < 一个 HREF = “ JavaScript的:; ” 类 = “减去FL ” > - </ 一 >   
            </ div >
        </ li >
        < li  class = “ col07 ” > {{book.amount}}元</ li >
        < 锂 类 = “ col08 ” > < 一个 HREF = “ JavaScript的:; ” >删除</ 一 > </ 李 >
    </ ul >
    {％endfor％}


    < ul  class = “ settlements ” >
        {％csrf_token％}
        < li  class = “ col01 ” > < input  type = “ checkbox ”  name = “ ”  checked = “ ” > </ li >
        < li  class = “ col02 ” >全选</ li >
        < 锂 类 = “ col03 ” >合计（不含运费）：< 跨度 >¥</ 跨度 > < EM > {{TOTAL_PRICE}} </ EM > < BR >共计< b > {{TOTAL_COUNT}} </ b >件商品</ li >
        < li  class = “ col04 ” > < input  type = “ submit ”  value = “去结算” > </ li >
    </ ul >
    </ form >
好，我们去结算功能也实现了。

5，接下来我们将提交订单页面完善一下，完成去支付功能。将支付方式绑定值值，供提交。
< input  type = “ radio ”  name = “ pay_style ”  value = “ 1 ”  checked >
< label  class = “ cash ” >货到付款</ label >
< input  type = “ radio ”  name = “ pay_style ”  value = “ 2 ” >
< label  class = “ weixin ” >微信支付</ label >
< input  type = “ radio ”  name = “ pay_style ”  value = “ 3 ” >
< label  class = “ zhifubao ” > </ label >
< input  type = “ radio ”  name = “ pay_style ”  value = “ 4 ” >
< label  class = “ bank ” >银行卡支付</ label >
查缺补漏，发现编辑收货地址功能还没有实现。我们先来编写用户中心地址页的接口。注意，在要写users/views.py中。

@login_required 
def  地址（请求）：
     '''用户中心 - 地址页''' 
    ＃获取登录用户的id 
    passport_id = request.session.get（' passport_id '）

    if request.method ==  ' GET '：
         ＃显示地址页面
        ＃查询用户的默认地址 
        addr = Address.objects.get_default_address（passport_id = passport_id）
         return render（request，' users / user_center_site.html '，{ ' addr '： addr，' page '：' address ' }）
     else：
         ＃添加收货地址
        ＃ 1.接收数据 
        recipient_name =请求。POST .get（' username '）
        recipient_addr =请求。POST .get（' addr '）
        zip_code =请求。POST .get（' zip_code '）
        recipient_phone =请求。POST .get（' phone '）

        ＃ 2.进行校验
        如果 不是 全部（[recipient_name，recipient_addr，zip_code，recipient_phone]）：
             return render（request， ' users / user_center_site.html '，{ ' errmsg '： '参数不能为空！' }）

        ＃ 3.添加收货地址 
        Address.objects.add_one_address（ passport_id = passport_id，
                                         recipient_name = recipient_name，
                                         recipient_addr = recipient_addr，
                                         zip_code = zip_code，
                                         recipient_phone = recipient_phone）

        ＃ 4。返回应答
        return redirect（reverse（ ' user：address '））
然后配置urls.py

    url（r ' ^ address / $ '，views.address，name = ' address '），＃用户中心 - 地址页
然后将user_center_site.html拷贝到templates / users文件夹下。并继承base.html。然后改写模板。

< div  class = “ site_con ” >
    < dl >
        < dt >当前地址：</ dt >
        {％if addr％}
            < dd > {{addr.recipient_addr}}（{{addr.recipient_name}}收）{{addr.recipient_phone}} </ dd >
        {％else％}
            < dd >无</ dd >
        {％ 万一 ％}
    </ dl >
</ div >
改写形式提交表单。

< form  method = “ post ”  action = “ / user / address / ” >
    {％csrf_token％}
    < div  class = “ form_group ” >
        < label >收件人：</ label >
        < input  type = “ text ”  name = “ username ” >
    </ div >
    < div  class = “ form_group form_group2 ” >
        < label >详细地址：</ label >
        < textarea  class = “ site_area ”  name = “ addr ” > </ textarea >
    </ div >
    < div  class = “ form_group ” >
        < label >邮编：</ label >
        < input  type = “ text ”  name = “ zip_code ” >
    </ div >
    < div  class = “ form_group ” >
        < label >手机：</ label >
        < input  type = “ text ”  name = “ phone ” >
    </ div >
    < input  type = “ submit ”  value = “提交”  class = “ info_submit ” >
</ form >
好，我们编辑地址的功能已经编写好了。

6，完善用户中心
接下来我们进一步完善一下用户中心，把用户中心的订单显示页面给做了。先来实现订单显示的后台接口。

＃用户/ views.py 
从 django.core.paginator进口分页程序

@login_required 
def  order（request，page）：
     '''用户中心 - 订单页''' 
    ＃查询用户的订单信息 
    passport_id = request.session.get（' passport_id '）

    ＃获取订单信息 
    order_li = OrderInfo.objects.filter（ passport_id = passport_id）

    ＃遍历获取订单的商品信息
    ＃命令- > OrderInfo的实例对象
    为顺序在 order_li：
        ＃根据订单ID查询订单商品信息 
        的order_id = order.order_id
        order_books_li = OrderGoods.objects.filter（order_id = order_id）

        ＃计算商品的小计
        ＃ order_books - > OrderGoods实例对象
        为 order_books在 order_books_li：
            count = order_books.count
            price = order_books.price
            amount = count * price
             ＃保存订单中每一个商品的小计 
            order_books.amount =金额

        ＃给订对象动态增加一个属性order_books_li，保存订单中商品的信息 
        order.order_books_li = order_books_li
    
    paginator = Paginator（order_li，3）       ＃每页显示3个订单
    
    num_pages = paginator.num_pages
    
    如果 不是页面：         ＃首次进入时默认进入第一页 
        page =  1 
    if page ==  ' ' 或 int（page）> num_pages：
        page =  1 
    else：
        page =  int（页面）
        
    order_li = paginator.page（页面）
    
    如果 num_pages <  5：
        pages =  range（1，num_pages +  1）
     elif page <=  3：
        页=  范围（1，6）
     的elif NUM_PAGES -页面<=  2：
        pages =  range（num_pages -  4，num_pages +  1）
     else：
        页=  范围（页面-  2，页+  3）

    context = {
         ' order_li '：order_li，
         ' pages '：pages，
    }

    return render（request，' users / user_center_order.html '，context）
然后配置urls.py.

    url(r'^order/(?P<page>\d+)?/?$', views.order, name='order'), # 用户中心-订单页  增加分页功能
然后将user_center_order.html拷贝到templates / users文件夹下，并继承base.html。然后改写模板中的元素，使得后端可以渲染。

{％extends'base.html'％}
{％load staticfiles％}
{％block title％}尚硅谷书店 - 首页{％endblock title％}
{％block topfiles％}
{％endblock topfiles％}
{％block body％}

    < div  class = “ main_con clearfix ” >
        < div  class = “ left_menu_con clearfix ” >
            < h3 >用户中心</ h3 >
            < ul >
                < 锂 > < 一个 HREF = “ {％URL '用户：用户' ％} ” >·个人信息</ 一 > </ 李 >
                < 锂 > < 一个 HREF = “ {％URL '用户：为了' ％} ” 类 = “活性” >·全部订单</ 一 > </ 李 >
                < 锂 > < 一个 HREF = “ {％URL '用户：地址' ％} ” >·收货地址</ 一 > </ 李 >
            </ ul >
        </ div >
        < div  class = “ right_content clearfix ” >
                {％csrf_token％}
                < h3  class = “ common_title2 ” >全部订单</ h3 >
                {＃OrderInfo＃}
                {％for order in order_li％}
                < ul  class = “ order_list_th w978 clearfix ” >
                    < li  class = “ col01 ” > {{order.create_time}} </ li >
                    < li  class = “ col02 ” >订单号：{{order.order_id}} </ li >
                    < li  class = “ col02 stress ” > {{order.status}} </ li >
                </ ul >

                < table  class = “ order_list_table w980 ” >
                    < tbody >
                        < tr >
                            < td  width = “ 55％” >
                                {＃遍历出来的order_books是一个OrderGoods对象＃}
                                order.order_books_li％中的{％for order_books}
                                < ul  class = “ order_book_list clearfix ” >                   
                                    < li  class = “ col01 ” > < img  src = “ {％static order_books.books.image％} ” > </ li >
                                    < li  class = “ col02 ” > {{order_books.books.name}} < em > {{order_books.books.price}}元/ {{order_books.books.unit}} </ em > </ li >
                                    < li  class = “ col03 ” > {{order_books.count}} </ li >
                                    < li  class = “ col04 ” > {{order_books.amount}}元</ li >
                                </ ul >
                                {％endfor％}
                            </ td >
                            < td  width = “ 15％” > {{order.total_price}}元</ td >
                            < td  width = “ 15％” > {{order.status}} </ td >
                            < td  width = “ 15％” > < a  href = “＃”  pay_method = “ {{order.pay_method}} ”  order_id = “ {{order.order_id}} ”  order_status = “ {{order.status}} ”  class = “ oper_btn ” >去付款</ a > </ td >
                        </ tr >
                    </ tbody >
                </ table >
                {％endfor％}

                < div  class = “ pagenation ” >
                    {％if order_li.has_previous％}
                        < 一个 HREF = “ {％URL'用户：为了页面= order_li.previous_page_number％} ” >上一页</ 一 >
                    {％ 万一 ％}
                    {％for page in pages％}
                        {％if page == order_li.number％}
                            < 一个 HREF = “ {％URL'用户：为了页面=页％} ” 类 = “活性” > {{页}} </ 一 >
                        {％else％}
                            < 一个 HREF = “ {％URL'用户：为了页面=页％} ” > {{页}} </ 一 >
                        {％ 万一 ％}
                    {％endfor％}
                    {％if order_li.has_next％}
                        < 一个 HREF = “ {％URL'用户：为了页面= order_li.next_page_number％} ” >下一页</ 一 >
                    {％ 万一 ％}
                </ div >
        </ div >
    </ div >
{％endblock body％}
这样我们个人中心的订单的显示页面也就做完了。

如图7所示，“去付款”功能的实现
接下来我们需要实现“去付款”功能。这里需要集成阿里的支付宝sdk。我们先来编写后端代码。

生成秘钥文件

openssl
OpenSSL> genrsa -out app_private_key.pem 2048  # 私钥
OpenSSL> rsa -in app_private_key.pem -pubout -out app_public_key.pem # 导出公钥
OpenSSL> exit
设置支付宝沙箱公司支付宝逐渐转换为RSA2秘钥，可以使用官方工具生成秘钥

支付宝沙箱地址：https://openhome.alipay.com/platform/appDaily.htm?tab=info
生成RSA2教程：https://docs.open.alipay.com/291/106130
测试用秘钥： 链接: https://pan.baidu.com/s/1HpAoD8heei18rXdjRIZdUg 密码: rcip
设置本地公钥和私钥格式

app_private_key_string.pem

-----BEGIN RSA PRIVATE KEY-----
         私钥内容
-----END RSA PRIVATE KEY-----


alipay_public_key_string.pem

-----BEGIN PUBLIC KEY-----
         公钥内容
-----END PUBLIC KEY-----
＃订单/ views.py 
＃前端需要发过来的参数：ORDER_ID 
＃职位
＃接口文档：HTTPS：//github.com/fzlee/alipay/blob/master/README.zh-hans.md 
＃安装的python-alipay- SDK 
＃点子安装python-支付宝-SDK --upgrade

来自支付宝进口支付宝

@login_required 
def  order_pay（request）：
     '''订单支付'''

    ＃接收订单id 
    order_id = request。POST .get（ ' order_id '）

    ＃数据校验
    如果 不是 order_id：
        返回 JsonResponse（{ ' res '： 1， ' errmsg '： '订单不存在' }）

    尝试：
        order = OrderInfo.objects.get（order_id = order_id，
                                       status = 1，
                                       pay_method = 3）
     除 OrderInfo.DoesNotExist：
         return JsonResponse（{ ' res '：2，' errmsg '：'订单信息出错' }）


    ＃将app_private_key.pem和app_public_key.pem拷贝到顺序文件夹下。 
    app_private_key_path = os.path.join（设置。 BASE_DIR， '订单/ app_private_key.pem '）
    alipay_public_key_path = os.path.join（设置。BASE_DIR，'订单/ app_public_key.pem '）

    app_private_key_string =  open（app_private_key_path）.read（）
    alipay_public_key_string =  open（alipay_public_key_path）.read（）

    ＃和支付宝进行交互 
    宝 =支付宝（
         APPID = “ 2016091500515408 ”，＃应用ID 
        app_notify_url = 无，  ＃默认回调URL 
        app_private_key_string = app_private_key_string，
         alipay_public_key_string = alipay_public_key_string，  ＃支付宝的公钥，验证支付宝回传消息使用，不是你自己的公司，
        sign_type  =  “ RSA2 ”，  ＃ RSA或者RSA2 
        debug  =  True，  ＃默认False
    ）

    ＃电脑网站支付，需要跳转到https://openapi.alipaydev.com/gateway.do？+ order_string 
    total_pay = order.total_price + order.transit_price＃小数 
    order_string = alipay.api_alipay_trade_page_pay（
         out_trade_no = ORDER_ID， ＃订单ID 
        TOTAL_AMOUNT = STR（total_pay）， ＃的Json传递，需要将浮点转换为字符串
        对象= '尚硅谷书城％s ' ％ order_id，
         return_url = 无，
         notify_url = 无  ＃可选，不填则使用默认通知网址
    ）
    ＃返回应答 
    pay_url =设置。ALIPAY_URL  +  '？'  + order_string
    返回 JsonResponse（{ ' res '： 3， ' pay_url '：pay_url， ' message '： ' OK ' }）
然后我们把公司和校钥册拷贝到订单文件夹下。然后我们需要获取用户的支付结果。

＃前端需要发过来的参数：ORDER_ID 
＃后
从支付宝进口支付宝

@login_required 
def  check_pay（request）：
     '''获取用户支付的结果'''

    passport_id = request.session.get（' passport_id '）
     ＃接收订单id 
    order_id = request。POST .get（' order_id '）

    ＃数据校验
    如果 不是 order_id：
        返回 JsonResponse（{ ' res '： 1， ' errmsg '： '订单不存在' }）

    尝试：
        order = OrderInfo.objects.get（order_id = order_id，
                                       passport_id = passport_id，
                                       pay_method = 3）
     除 OrderInfo.DoesNotExist：
         return JsonResponse（{ ' res '：2，' errmsg '：'订单信息出错' }）

    app_private_key_path = os.path.join（设置。BASE_DIR，'订单/ app_private_key.pem '）
    alipay_public_key_path = os.path.join（设置。BASE_DIR，'订单/ app_public_key.pem '）

    app_private_key_string =  open（app_private_key_path）.read（）
    alipay_public_key_string =  open（alipay_public_key_path）.read（）

    ＃和支付宝进行交互 
    宝 =支付宝（
         APPID = “ 2016091500515408 ”，＃应用ID 
        app_notify_url = 无，  ＃默认回调URL 
        app_private_key_string = app_private_key_string，
         alipay_public_key_string = alipay_public_key_string，  ＃支付宝的公钥，验证支付宝回传消息使用，不是你自己的公司，
        sign_type  =  “ RSA2 ”，  ＃ RSA或者RSA2 
        debug  =  True，  ＃默认False
    ）

    而 True：
         ＃进行支付结果查询 
        结果= alipay.api_alipay_trade_query（order_id）
        代码= result.get（'代码'）
         如果代码==  ' 10000 ' 和 result.get（' trade_status '）==  ' TRADE_SUCCESS '：
             ＃用户支付成功
            ＃改变订单支付状态 
            order.status =  2  ＃待发货
            ＃填写支付宝交易号 
            order.trade_id = result.get（' trade_no '）
            order.save（）
            ＃返回数据
            返回 JsonResponse（{ ' res '： 3， ' message '： '支付成功' }）
         elif代码 ==  ' 40004 ' 或（code ==  ' 10000 ' 和 result.get（ ' trade_status '） ==  ' WAIT_BUYER_PAY '）：
            ＃支付订单还未生成，继续查询
            ＃用户还未完成支付，继续查询time.sleep 
            （ 5）
            继续
        其他：
             ＃支付出错
            返回 JsonResponse（{ ' res '：4，' errmsg '：'支付出错' }）
配置支付宝网关到配置文件中。

ALIPAY_URL = ' https: //openapi.alipaydev.com/gateway.do '
配置urls.py.

    url（r ' ^ pay / $ '，views.order_pay，name = ' pay '），＃订单支付 
    url（r ' ^ check_pay / $ '，views.check_pay，name = ' check_pay '），＃查询支付结果
然后编写前端的jQuery代码，来处理支付后的结果，比如支付成功以后刷新页面。代码以下写入templates/users/user_center_order.html中。

{ ％ block bottomfiles ％ }
     < script > 
    $（function（）{
         $（'。 oper_btn '）。click（function（）{
             //获取订单id和订单的状态 
            order_id =  $（this）.attr（' order_id '）
            order_status =  $（this）。attr（' order_status '）
            csrf =  $（' input [name =“csrfmiddlewaretoken”] '）。val（）
            params = { ' order_id '： order_id，' csrfmiddlewaretoken '： csrf}
             if（order_status ==  1）{
                 $。交（' /订单/收费/ '，则params，功能（数据）{
                     如果（数据。RES  ==  3）{
                         //把用户引导支付页面
                        窗口。打开（数据。pay_url）
                         //查询用户的支付结果
                        $。交（' /订单/ check_pay / '，则params，功能（数据）{
                             如果（数据。RES  ==  3）{
                                 警报（'支付成功'）
                                 //重新刷新页面
                                位置。重新加载（）
                            }
                            其他 {
                                 警报（数据。ERRMSG）
                            }
                        }）
                    }
                    其他 {
                         警报（数据。ERRMSG）
                    }
                }）
            }
        }）
    }）
    < / script > 
{ ％ endblock bottomfiles ％ }
好，我们支付功能也编写好了。

7，使用缓存
使用Redis的缓存首页的页面。

来自 django.views.decorators.cache import cache_page
 @cache_page（60  *  15）
 def  index（request）：
     '''显示首页'''
     ...
别忘了把redis启动起来。在settings.py中配置redis缓存。

＃ PIP安装Django-redis的
CACHES  = {
     “默认”：{
         “ BACKEND ”： “ django_redis.cache.RedisCache ”，
         “ LOCATION ”： “ redis的：//127.0.0.1：2分之6379 ”，
         “ OPTIONS ”：{
             “ CLIENT_CLASS “： ” django_redis.client.DefaultClient “，
             ” PASSWORD “： ”“
        }
    }
}

SESSION_ENGINE  =  “ django.contrib.sessions.backends.cache ”
 SESSION_CACHE_ALIAS  =  “默认”
8，评论功能的实现
先来新建注释应用。

$ python manage.py startapp comments
然后设计数据库表结构。

从 django.db 进口车型
 从 db.base_model 进口 BaseModel
 从 users.models 进口护照
 从 books.models 进口图书
 ＃创建您的模型在这里。
class  Comments（BaseModel）：
    disabled = models.BooleanField（default = False，verbose_name = “禁用评论”）
    user = models.ForeignKey（' users.Passport '，verbose_name = “ user user ID ”）
    book = models.ForeignKey（' books.Books '，verbose_name = “书籍ID ”）
    content = models.CharField（max_length = 1000，verbose_name = “评论内容”）

    class  Meta：
        db_table =  ' s_comment_table '
这里要注意外键的使用和理解。然后我们要在配置文件里注册app。

＃书店/ settings.py 
INSTALLED_APPS  =（
     ... 
    '意见'，＃评论模块
    ... 
）
然后我们来写评论应用的视图函数。视图函数使用Redis的作为缓存，缓存了GET请求的结果。

＃评论/ views.py 
从 django.shortcuts导入已呈现
从 django.http进口 JsonResponse
从 django.views.decorators.http进口 require_http_methods
从 comments.models进口评论
从 books.models进口图书
从 users.models进口护照
从 django.views .decorators.csrf import csrf_exempt
从 utils.decorators import 导入 json
 import redis
login_required
 ＃在此处创建您的视图。
＃设置过期时间
EXPIRE_TIME  =  60  *  10 
＃连接redis数据库 
pool = redis.ConnectionPool（host = ' localhost '，port = 6379，db = 2）
redis_db = redis.Redis（connection_pool = pool）

@csrf_exempt 
@require_http_methods（[ ' GET '，' POST ' ]）
 @ logininquired 
def  comment（request，books_id）：
    book_id = books_id
     if request.method ==  ' GET '：
         ＃先在redis里面寻找评论 
        c = redis_db.get（' comment_ ％s ' ％ book_id）
         试试：
            c = c.decode（' utf-8 '）
         除外：
             传递
        print（' c：'，c）
         如果 c：
             return JsonResponse（{
                     ' code '：200，
                     ' data '：json.loads（c），
                }）
        否则：
             ＃找不到，就从数据库里面取 
            评论= Comments.objects.filter（book_id = book_id）
            数据= []
             为 Ç 在评论：
                data.append（{
                    ' user_id '：c.user_id，
                     ' content '：c.content，
                }）

            res = {
                 ' code '：200，
                 ' data '：数据，
            }
            尝试：
                redis_db.setex（' comment_ ％s ' ％ book_id，json.dumps（data），EXPIRE_TIME）
             除了 异常 为 e：
                 print（' e：'，e）
             返回 JsonResponse（res）

    否则：
        params = json.loads（request.body.decode（' utf-8 '））

        book_id = params.get（' book_id '）
        user_id = params.get（' user_id '）
        content = params.get（' content '）

        book = Books.objects.get（id = book_id）
        user = Passport.objects.get（id = user_id）

        comment =评论（book = book，user = user，content = content）
        comment.save（）

        返回 JsonResponse（{
                 ' code '：200，
                 ' msg '：'评论成功'，
            }）
然后注册网址。

＃根urls.py：书店：urls.py 
urlpatterns的 = [
     ... 
    URL（ [R ' ^评论/ '，包括（ ' comments.urls '，命名空间= '评论'））， ＃评论模块
    ... 
]
评论app的url注册。

来自 django.conf.urls 从评论导入视图导入 url


urlpatterns = [
    url（r '评论/ （？P <books_id> \ d + ） / $ '，views.comment，name = ' comment '），＃评论内容 
]
注册好URL以后，我们要编写detail.html里面的前端代码了。

            < div  class = “ operate_btn ” >
                {％csrf_token％}
                < 一个 HREF = “ JavaScript的:; ” 类 = “ buy_btn ” >立即购买</ 一 >
                < 一个 HREF = “ JavaScript的:; ”  books_id = “ {{books.id}} ” 类 = “ add_cart ”  ID = “ add_cart ” >加入购物车</ 一 >
                < 一个 HREF = “＃”  ID = “写评论” 类 = “评论” >我要写评论</ 一 >
            </ div >
            < div  style = “ display：flex ; ”  id = “ comment-input ”  data-bookid = “ {{books.id}} ”  data-userid = “ {{request.session.passport_id}} ” >
                < div >
                    < input  type = “ text ”  placeholder = “评论内容” >
                </ div >
                < div  id = “ submit-comment ” >
                    < 按钮 >
                      提交评论
                    </ button >
                </ div >
            </ div >
再增加id-book_detail和id-book_comment增加点击效果

    < div  class = “ r_wrap fr clearfix ” >
        < ul  class = “ detail_tab clearfix ” >
            < li  class = “ active ”  id = “ detail ” >商品介绍</ li >
            < li  id = “ comment ” >评论</ li >
        </ ul >

        < div  class = “ tab_content ” >
            < dl  id = “ book_detail ” >
                < dt >商品详情：</ dt >
                < dd > {{books.detail | 安全}} </ dd >
            </ dl >
            < dl  id = “ book_comment ”  style = “ display：none ; font-size：15 px ; color：＃0a0a0a ” >
                < dt >用户评论：</ dt >
                < dd > </ dd >
            </ dl >
        </ div >
    </ div >
然后写样式。

< style type =“text / css” > 
.comment {
     background-color：＃c40000 ;
    颜色：#fff ;
    margin-left：10 px ;
    位置：相对 ;
    z-index：10 ;
    display：inline-block ;
    宽度：178 像素 ;
    身高：38 像素 ;
    border：1 px  solid  ＃c40000;
    font-size：14 px ;
    行高：38 像素 ;
    text-align：center ;
}
</ style >
以及提交评论的JS代码。

    //获取评论
    $。ajax（{
        url ： ' / comment / comment / '  +  $（'＃comment-input '）。数据（' BOOKID '），
         成功： 函数（RES）{
             如果（RES。代码 ===  200）{
                 VAR数据=  RES。数据 ;
                控制台。记录（数据）;
                var div_head =  '<div> ' ;
                var div_tail =  ' </ div> ' ;
                VAR dom_element =  ' '
                为（I =  0 ;我<  数据。长度 ;我++）{
                     VAR头=  ' <DIV> ' ;
                    var tail =  ' </ div> ' ;
                    var temp = head +  '<span> '  + data [i]。user_id  +  ' </ span> '  +  ' <br> '  +  ' <span> '  + data [i]。内容 +  ' </ span> '  +尾巴;
                    dom_element + = temp;
                }
                dom_element = div_head + dom_element + div_tail;
                $（'＃book_comment '）。追加（dom_element）;
            }
        }
    }）

    $（'＃tail '）。点击（函数（）{
         $（本）。addClass（'主动'）;
         $（'＃注释'）。removeClass（'主动'）;
         $（' #book_comment '）。隐藏（）;
         $（' #book_detail '）。show（）;
    }）
    $（'＃ comment '）。点击（函数（）{
         $（本）。addClass（'主动'）;
         $（' #detail '）。removeClass（'主动'）;
         $（' #book_comment '。）显示（）;
         $（' #book_detail '）。hide（）;
    }）
    $（'＃write-comment '）。click（function（）{
         $（'＃comment-input '）。show（）;
    }）
    $（'＃submit-comment '）。click（function（）{
         var book_id =  $（'＃comment-input '）。data（' bookid '）;
         var user_id =  $（'＃comment-input '）。data（' userid '）;
         var content =  $（'＃comment-input input '）。val（）;
        var data = {
            book_id ： book_id，
            user_id ： user_id，
            内容：内容，
        }
        控制台。log（' content：'，content）;
        $。ajax（{
            输入： ' POST '，
            url ： ' / comment / comment / '  + book_id +  ' / '，
            数据： JSON。字符串化（数据），
             成功： 函数（RES）{
                 如果（RES。代码 ===  200）{
                     //执行console.log（ 'RES：'，RES）
                    $（'＃注释输入'）。hide（）;
                }
            }
        }）
    }）
这样评论功能就做好了。

9，发送邮件功能实现。
1，同步发送邮件
先在配置文件中配置邮件相关参数。

＃ settings.py 
EMAIL_BACKEND  =  ' django.core.mail.backends.smtp.EmailBackend '
 EMAIL_HOST  =  ' smtp.126.com ' 
＃ 126和163邮箱的SMTP端口为25; QQ邮箱使用的SMTP端口为465 
EMAIL_PORT  =  25 
＃如果使用QQ邮箱发送邮件，需要开启SSL加密，如果在阿里云上部署，也需要开启SSL加密，同时修改端口为EMAIL_PORT = 465 
＃ EMAIL_USE_SSL =真
＃发送邮件的邮箱
EMAIL_HOST_USER  =  ' xxxxxxxx@126.com ' 
＃在邮箱中设置的客户端授权密码
EMAIL_HOST_PASSWORD  =  ' xxxxxxxx ' 
＃收件人看到的发件人
EMAIL_FROM =  ' shangguigu <xxxxxxxx@126.com> '
在注册页的视图函数里写发邮件的代码。

＃用户/ views.py 
从 itsdangerous进口 TimedJSONWebSignatureSerializer为串行
从 itsdangerous进口 SignatureExpired
itsdangerous是一个产生令牌的库，有烧瓶的作者编写。

def  register_handle（request）：
     '''进行用户注册处理''' 
    ＃接收数据 
    用户名=请求。POST .get（' user_name '）
    密码=请求。POST .get（' pwd '）
    电子邮件=请求。POST .get（' email '）

    ＃进行数据校验
    如果 不是 全部（[用户名，密码，电子邮件]）：
        ＃有数据为空
        返回渲染（请求， ' users / register.html '，{ ' errmsg '： '参数不能为空！' }）

    ＃判断邮箱是否合法
    如果 不是 re.match（ r ' ^ [ a-z0-9 ] [ \ w \。\  - ] * @ [ a-z0-9 \  - ] + （\。 [ az ] {2， 5} ）{1,2} $ '，电子邮件）：
        ＃邮箱不合法
        return render（request， ' users / register.html '，{ ' errmsg '： '邮箱不合法！' }）

    p = Passport.objects.check_passport（用户名=用户名）

    if p：
         return render（request，' users / register.html '，{ ' errmsg '：'用户名已存在！' }）

    ＃进行业务处理：注册，向账户系统中添加账户
    ＃ Passport.objects.create（用户名=用户名，密码=密码，电子邮件=电子邮件） 
    passport = Passport.objects.add_one_passport（用户名=用户名，密码=密码，电子邮件=电子邮件）

    ＃生成激活的令牌itsdangerous 
    串行 =串行器（设置。 SECRET_KEY， 3600）
    token = serializer.dumps（{ ' confirm '：passport.id}）＃返回bytes 
    token = token.decode（）

    ＃给用户的邮箱发激活邮件 
    send_mail（ '尚硅谷书城用户激活'， ' '，设置 .EMAIL_FROM，[email]， html_message = ' <a href =“http://127.0.0.1:8000/user/active / ％s /" > http:// 127.0.0.1:8000/user/active/ </a> ' ％ token）

    ＃。注册完，还是返回注册页
    返回重定向（逆转（ '书：指数'））
register_handle函数变为以上代码，增加了发送邮件的功能。注意这里我们没有实现check_passport函数。所以在要users/models.py中的PassportManager中实现这个函数。

类 PassportManager（车型。经理）：
     ... 
    DEF  check_passport（个体经营，用户名）：
         尝试：
            passport =  self .get（username = username）
         除了 self .model.DoesNotExist：
            护照=  无
        如果护照：
             回 真正的
        回报 假
2，使用消息队列芹菜来异步发送邮件。
首先配置芹菜。在书店的文件夹下面。

＃书店/ celery.py 
进口 OS
从芹菜进口芹菜

＃设置'celery'程序的默认Django设置模块。
os.environ.setdefault（ ' DJANGO_SETTINGS_MODULE '， ' bookstore.settings '）

app = Celery（' bookstore '，broker = ' redis：//127.0.0.1：6379/6 '）

＃使用这里的字符串意味着工人不必序列化
＃配置对象子进程。
＃ -  namespace ='CELERY'表示所有与celery相关的配置键
＃    应该有一个`CELERY_`前缀。
app.config_from_object（ ' django.conf：settings '， namespace = ' CELERY '）

＃从所有已注册的Django app配置中加载任务模块。
app.autodiscover_tasks（）


@ app.task（bind = True）
 def  debug_task（self）：
     print（' Request：{0 ！r } '。 format（self .request））
然后在用户app中编写异步任务。

＃用户/ tasks.py 
从芹菜进口 shared_task
从 django.conf进口设置
从 django.core.mail进口 send_mail

@shared_task 
def  send_active_email（令牌，用户名，电子邮件）：
     '''发送激活邮件''' 
    subject =  '尚硅谷书城用户激活'  ＃标题 
    message =  ' ' 
    sender = settings。EMAIL_FROM  ＃发件人 
    接收机= [电子邮件] ＃收件人列表 
    html_message =  ' <a href="http://127.0.0.1:8000/user/active/ %s /"> http://127.0.0.1： 8000 / user / active / </a> '代币
    send_mail（主题，消息，发件人，接收者，html_message = html_message）
然后在视图函数中导入异步任务。

来自 users.tasks import send_active_email
 def  register_handle（request）：
     ...
    send_active_email.delay（令牌，用户名，电子邮件）
    ...
然后改写根应用文件夹里的__init__.py，将整个文件改为：

导入 pymysql
pymysql.install_as_MySQLdb（）
＃这将确保当应用程序总是进口
＃ Django的启动，使shared_task将使用这个应用程序。
来自 .celery导入应用程序 as celery_app

__all__  = [ ' celery_app ' ]
然后运行:(在根目录，和manage.py同级）

$ celery -A bookstore worker -l info
10，登陆验证码功能实现
项目将中的Ubuntu-RI.ttf字体文件拷贝产品到你的项目的根目录下面。

＃用户/ views.py 
从 django.http进口的HttpResponse
从 django.conf进口设置
导入口
 DEF  附加码（请求）：
    ＃引入绘图模块
    从 PIL  进口图片，ImageDraw，ImageFont
    ＃引入随机函数模块
    进口随机
    ＃定义变量，用于画面的背景色，宽，高 
    BGCOLOR =（random.randrange（ 20， 100），random.randrange（
         20， 100）， 255）
    width =  100 
    height =  25 
    ＃创建画面对象 
    im = Image.new（' RGB '，（width，height），bgcolor）
     ＃创建画笔对象 
    draw = ImageDraw.Draw（im）
     ＃调用画笔的point（）函数绘制噪点
    对于我在 范围（0，100）：
        xy =（random.randrange（0，width），random.randrange（0，height））
        填=（random.randrange（0，255），255，random.randrange（0，255））
        draw.point（XY，填=填充）
     ＃定义验证码的备选值 
    STR1 =  ' ABCD123EFGHIJK456LMNOPQRS789TUVWXYZ0 ' 
    ＃随机选取4个值作为验证码 
    rand_str =  ' '
    为我在 范围（0，4）：
        rand_str + = STR1 [random.randrange（0，LEN（STR1））]
     ＃构造字体对象 
    字体= ImageFont.truetype（os.path.join（设置。BASE_DIR，“ Ubuntu的RI.ttf ”），15）
     ＃构造字体颜色 
    FONTCOLOR =（255，random.randrange（0，255），random.randrange（0，255））
    ＃绘制4个字 
    draw.text（（5，2），rand_str [ 0 ]，font = font，fill = fontcolor）
    draw.text（（25，2），rand_str [ 1 ]，字体=字体，填= FONTCOLOR）
    draw.text（（50，2），rand_str [ 2 ]，字体=字体，填= FONTCOLOR）
    draw.text（（75，2），rand_str [ 3 ]，字体=字体，填= FONTCOLOR）
    ＃释放画笔
    德尔绘制
     ＃存入会话，用于做进一步验证 
    的request.session [ '附加码' ] = rand_str
     ＃内存文件操作
    import io
    buf = io.BytesIO（）
     ＃将图片保存在内存中，文件类型为png 
    im.save（buf，' png '）
     ＃将内存中的图片数据返回给客户端，MIME类型为图片png 
    return HttpResponse（buf .getvalue（），' image / png '）
然后在urls.py中配置URL。

    url(r'^verifycode/$', views.verifycode, name='verifycode'), # 验证码功能
编写前端代码。在前段代码中的表格里添加以下代码。

// templates / users / login.html
< div  style = “ top：100 px ; position：absolute ; ” >
    < input  type = “ text ”  id = “ vc ”  name = “ vc ” >
    < IMG  ID = '附加码'  SRC = “ /用户/附加码/ ” 的onclick = “ this.src = '/用户/附加码/？' +的Math.random（） ”  ALT = “注册码” />
</ div >
前端需要向后端发布数据。员额以下数据

var username =  $（'＃username '）。val（）
 var password =  $（'＃pwd '）。val（）
 var csrf =  $（' input [name =“csrfmiddlewaretoken”] '）。val（）
 var remember =  $（' input [name =“remember”] '）。prop（' checked '）
 var vc = $（' input [name =“vc”] '）。val（）
 //发起ajax请求
var params = {
     ' username '： username，
     ' password '： password，
     ' csrfmiddlewaretoken '： csrf，
     ' remember '：记住，
     ' verifycode '： vc，
}
然后在后端进行校验.login_check函数改为以下代码实现。

def  login_check（request）：
     '''进行用户登录校验''' 
    ＃ 1.获取数据 
    用户名=请求。POST .get（' username '）
    密码=请求。POST .get（' password '）
    记住=请求。POST .get（'记住'）
    verifycode =请求。POST .get（' verifycode '）

    ＃ 2.数据校验
    如果 不是 全部（[用户名，密码，请记住，验证码]）：
        ＃有数据为空
        返回 JsonResponse（{ ' res '： 2 }）

    if verifycode.upper（）！= request.session [ ' verifycode ' ]：
         return JsonResponse（{ ' res '：2 }）

    ＃ 3.进行处理：根据用户名和密码查看账户信息 
    passport = Passport.objects.get_one_passport（用户名=用户名，密码=密码）

    如果护照：
         ＃用户名密码正确
        ＃获取会话中的url_path 
        ＃如果request.session.has_key（ 'url_path'）： 
        ＃      next_url = request.session.get（ 'url_path'） 
        ＃其他：
        ＃      next_url =反向（'书：index'） 
        next_url = reverse（' books：index '）＃ / user / 
        jres = JsonResponse（{ ' res '：1，' next_url '：next_url}）

        ＃判断是否需要
        记住用户名if remember ==  ' true '：
            ＃记住用户名 
            jres.set_cookie（ ' username '，username， max_age = 7 * 24 * 3600）
        否则：
            ＃不要 
            记住用户名 jres.delete_cookie （ '用户名'）

        ＃记住用户的登录状态 
        request.session [ ' islogin ' ] =  真的 
        request.session [ '用户名' ] =用户名
        request.session [ ' passport_id ' ] = passport.id
         return jres
     else：
         ＃用户名或密码错误
        返回 JsonResponse（{ ' res '：0 }）
那我们的验证码功能就实现了。

11，全文检索的实现
添加全文检索应用，在配置文件中。

INSTALLED_APPS  =（
     ... 
    ' haystack '，
）
在配置文件中写入以下配置。

＃全文
检索 配置HAYSTACK_CONNECTIONS = {
     '默认'：{
        ＃使用whoosh引擎
        ' ENGINE '： ' haystack.backends.whoosh_cn_backend.WhooshEngine '，
        ＃ 'ENGINE'：'haystack.backends.whoosh_backend.WhooshEngine'，
        ＃索引文件路径
        ' PATH '：os.path.join（ BASE_DIR， ' whoosh_index '），
    }
}

＃当添加，修改，删除数据时，自动生成索引
HAYSTACK_SIGNAL_PROCESSOR  =  ' haystack.signals.RealtimeSignalProcessor '

HAYSTACK_SEARCH_RESULTS_PER_PAGE  =  6  ＃指定搜索结果每页的条数
在urls.py中配置。

urlpatterns = [
    ...
    url(r'^search/', include('haystack.urls')),
]
在图书应用目录下建立search_indexes.py文件。

从草垛进口指数
 从 books.models 进口图书


＃指定对于某个类的某些数据建立索引，一般类名：模型类名+指数
类 BooksIndex（索引。 SearchIndex，索引。可转位）：
    ＃指定根据表中的哪些字段建立索引：比如：商品名字商品描述 
    text = indexes.CharField（ document = True， use_template = True）

    def  get_model（self）：
         返回 Books

    def  index_queryset（self，using = None）：
         return  self .get_model（）。objects.all（）
在目录“模板/搜索/索引/书籍/”下创建“books_text.txt”文件

{{ object.name }} # 根据书籍的名称建立索引
{{ object.desc }} # 根据书籍的描述建立索引
{{ object.detail }} # 根据书籍的详情建立索引
在目录“模板/搜索/”下建立search.html。

{％extends'base.html'％}
{％load staticfiles％}
{％block title％}尚硅谷书城 - 书籍搜索结果列表{％endblock title％}
{％block body％}
    < div  class = “ breadcrumb ” >
        < 一个 HREF = “＃” > {{查询}} </ 一 >
        < span >> </ span >
        < 一个 HREF = “＃” >搜索结果如下：</ 一 >
    </ div >

    < div  class = “ main_wrap clearfix ” >
            < ul  class = “ book_type_list clearfix ” >
                {％for page in page％}
                    < li >
                        < 一个 HREF = “ {％URL '的书：详细' books_id = item.object.id％} ” > < IMG  SRC = “ {％静态item.object.image％} ” > </ 一 >
                        < H4 > < 一个 HREF = “ {％URL '的书：详细' books_id = item.object.id％} ” > {{item.object.name}} </ 一 > </ H4 >
                        < div  class = “ operations ” >
                            < span  class = “ price ” >¥{{item.object.price}} </ span >
                            < span  class = “ unit ” > {{item.object.unit}} </ span >
                            < 一个 HREF = “＃” 类 = “ add_books ” 标题 = “加入购物车” > </ 一 >
                        </ div >
                    </ li >
                {％endfor％}
            </ ul >
            < div  class = “ pagenation ” >
                {％if page.has_previous％}
                    < 一个 HREF = “ /搜索/ Q = {{查询}}＆页= {{page.previous_page_number}} ” > <上一页</ 一 >
                {％ 万一 ％}
                {％for pindex in paginator.page_range％}
                    {％if pindex == page.number％}
                        < 一个 HREF = “ /搜索/ Q = {{查询}}＆页= {{p食指}} ” 类 = “活性” > {{p食指}} </ 一 >
                    {％else％}
                        < 一个 HREF = “ /搜索/ Q = {{查询}}＆页= {{p食指}} ” > {{p食指}} </ 一 >
                    {％ 万一 ％}
                {％endfor％}
                {％if page.has_next％}
                    < 一个 HREF = “ /搜索/ Q = {{查询}}＆页= {{page.next_page_number}} ” >下一页> </ 一 >
                {％ 万一 ％}
            </ div >
    </ div >
{％endblock body％}
建立ChineseAnalyzer.py文件。保存在haystack的安装文件夹下，路径如“/home/python/.virtualenvs/django_py2/lib/python3.5/site-packages/haystack/backends”

进口解霸
 从 whoosh.analysis 进口标记生成器，令牌


类 ChineseTokenizer（标记生成器）：
     DEF  __call__（自，值，位置= 假，字符= 假，
                  keeporiginal = 假，removestops = 真，
                  START_POS = 0，start_char = 0，模式= ' '，** kwargs）：
        t =令牌（位置，字符，removestops = removestops，mode = mode，
                   ** kwargs）
        seglist = jieba.cut（值，cut_all = 真）
         为瓦特在 seglist：
            t.original = t.text = w
            t.boost =  1.0 
            如果职位：
                t.pos = start_pos + value.find（w）
             如果是 chars：
                t.startchar = start_char + value.find（w）
                t.endchar = start_char + value.find（w）+  len（w）
             yield t


def  ChineseAnalyzer（）：
     返回 ChineseTokenizer（）
复制whoosh_backend.py文件，改名为whoosh_cn_backend.py注意：复制出来的文件名，末尾会有一个空格，记得要删除这个空格然后将下面这一行代码写入whoosh_cn_backend.py文件中。

来自 .ChineseAnalyzer 进口 ChineseAnalyzer
然后查找下面的这一行代码

analyzer = StemmingAnalyzer（）
改为

analyzer = ChineseAnalyzer（）
生成索引

$ python manage.py rebuild_index
模板在base.html中创建³³搜索栏

<form method='get' action="/search/" target="_blank">
    <input type="text" name="q">
    <input type="submit" value="查询">
</form>
12，用户激活功能的实现
首先编写视图函数：

def  register_active（request，token）：
     '''用户账户激活''' 
    serializer = Serializer（设置.SECRET_KEY，3600）
     试试：
        info = serializer.loads（token）
        passport_id = info [ ' confirm ' ]
         ＃进行用户激活 
        护照= Passport.objects.get（id = passport_id）
        passport.is_active =  True
        passport.save（）
        ＃跳转的登录页
        返回重定向（后退（ '用户：登录'））
    除了 SignatureExpired：
        ＃链接过期
        返回的HttpResponse（ '激活链接已过期'）
然后配置URL就完成了。

    url(r'^active/(?P<token>.*)/$', views.register_active, name='active'), # 用户激活
13，用户中心最近浏览功能的实现
最近浏览使用Redis的实现。重新编写书籍/ views.py中的细节函数，每次点击商品，都将商品信息写入redis的，作为最近浏览的数据。

＃书籍/ views.py 
DEF  细节（请求， books_id）：
     '' '显示商品的详情页面''' 
    ＃获取商品的详情信息 
    书籍 = Books.objects.get_books_by_id（ books_id = books_id）

    如果书籍是 无：
         ＃商品不存在，跳转到首页
        返回重定向（反向（' books：index '））

    ＃新品推荐 
    books_li = Books.objects.get_books_by_type（ TYPE_ID = books.type_id，极限= 2，排序= '新'）

    ＃用户登录之后，才记录浏览记录
    ＃每个用户浏览记录对应redis中的一条信息格式：'history_用户id'：[10,9,2,3,4] 
    ＃ [ 
    9,10,2,3] ，4] if request.session.has_key（ ' islogin '）：
        ＃用户已登录，记录浏览记录 
        con = get_redis_connection（ ' default '）
        key =  ' history_ ％d ' ％ request.session.get（' passport_id '）
         ＃先从redis列表中移除books.id 
        con.lrem（key，0，books.id）
        con.lpush（key，books.id）
        ＃保存用户最近浏览的5个商品 
        con.ltrim（键， 0， 4）

    ＃定义 
    上下文context = { ' books '：books， ' books_li '：books_li}

    ＃使用模板
    返回渲染（请求， '书籍/ detail.html '，上下文）
然后重写用户中心的视图函数代码：用户/ views.py中的用户函数。

@login_required 
def  user（request）：
     '''用户中心 - 信息页''' 
    passport_id = request.session.get（' passport_id '）
     ＃获取用户的基本信息 
    addr = Address.objects.get_default_address（passport_id = passport_id）

    ＃获取用户的最近浏览信息 
    con = get_redis_connection（ ' default '）
    键=  ' history_ ％d ' ％ passport_id
     ＃取出用户最近浏览的5个商品的ID 
    history_li = con.lrange（键，0，4）
     ＃ history_li = [21,20,11] 
    ＃打印（history_li） 
    ＃查询数据库，获取用户最近浏览的商品信息
    ＃ books_li = Books.objects.filter（id__in = history_li） 
    books_li = []
     为 ID  在 history_li：
        books = Books.objects.get_books_by_id（books_id = id）
        books_li.append（书）

    return render（request，' users / user_center_info.html '，{ ' addr '：addr，
                                                            ' page '：' user '，
                                                            ' books_li '：books_li}）
然后编写前端页面。重写user_center_info.html中最近浏览下面的HTML内容。

< h3  class = “ common_title2 ” >最近浏览</ h3 >
< div  class = “ has_view_list ” >
    < ul  class = “ book_type_list clearfix ” >
        {％for books in books_li％}
            < li >
                < 一个 HREF = “ {％URL '的书：详细' books_id = books.id％} ” > < IMG  SRC = “ {％静态books.image％} ” > </ 一 >
                < H4 > < 一个 HREF = “ {％URL '的书：详细' books_id = books.id％} ” > {{books.name}} </ 一 > </ H4 >
                < div  class = “ operations ” >
                    < span  class = “ price ” >¥{{books.price}} </ span >
                    < span  class = “ unit ” > {{books.unit}} </ span >
                    < 一个 HREF = “＃” 类 = “ add_books ” 标题 = “加入购物车” > </ 一 >
                </ div >
            </ li >
        {％endfor％}
    </ ul >
</ div >
最近浏览功能就实现了。

14，前端过滤器实现
在用户文件夹中新建templatetags文件夹。然后新建__init__.py文件，这是空文件。然后新建filters.py文件。

来自 django.template 导入库

＃创建一个Library类的对象 
register = Library（）


＃创建一个过滤器函数
@register.filter 
def  order_status（ status）：
     '''返回订单状态对应的字符串''' 
    status_dict =   {
         1： “待支付”，
         2： “待发货”，
         3： “待收货“，
         4： ”待评价“，
         5： ”已完成“，
    }
    return status_dict [status]
然后在根应用settings.py里面添加应用：

INSTALLED_APPS  =（
     ... 
    ' users.templatetags.filters '，＃过滤器功能 
）
这样我们就能在前端使用这个过滤器了。

< td  width = “ 15％” > {{order.status | order_status}} </ td >
注意要在页面里

{％load filters％}
15，使用gunicorn + nginx的+ django的进行部署
安装nginx的。

sudo apt install nginx
先看nginx配置文件nginx.conf，一般情况下，路径为/etc/nginx/nginx.conf

user root;
worker_processes auto;
pid /run/nginx.pid;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;
        gzip_disable "msie6";
        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        #include /etc/nginx/sites-enabled/*;

        server {
            listen 80;
            server_name localhost;
            location / {
                proxy_pass http://0.0.0.0:8000;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            }
            error_page 500 502 503 504 /50x.html;

            location = /50x.html {
                root html;
            }
            location /media {
                alias /root/bookstore/bookstore/static;
            }
            location /static {
                alias /root/bookstore/bookstore/collect_static;
            }
        }
}
如果nginx没启动，则执行

$ nginx
如果nginx已经启动，则执行以下命令重启

$ nginx -s reload
然后在根目录书店新建文件夹collect_static。注意要在配置文件settings.py中写一行

STATIC_ROOT  = os.path.join（BASE_DIR，' collect_static '）
然后在根目录运行python manage.py collectstatic命令，这个命令用来收集静态文件。并将books / models.py中添加代码：

来自 django.core.files.storage 导入 FileSystemStorage
fs = FileSystemStorage（location = ' / root / bookstore / bookstore / collect_static '）
 类 Books（BaseModel）：
     ... 
    image = models.ImageField（storage = fs，upload_to = ' books '，verbose_name = '商品图片'）
     。 ..
然后在根目录运行gunicorn。安装gunicorn，pip install gunicorn

nohup gunicorn -w 3 -b 0.0.0.0:8000 bookstore.wsgi:application &
16，django的日志模块的使用
首先将下面的代码添加到配置文件settings.py。

LOGGING  = {
     ' version '：1，
     ' disable_existing_loggers '：False，
     ' formatters '：{              ＃日志输出的格式
        '详细'：{
             '格式'：' ％（levelname）s  ％（asctime）s  ％（模块）s  ％（进程）d  ％（线程）d  ％（消息）s '
        }，
        ' simple '：{
             ' format '： ' ％（levelname）s  ％（message）s '
        }，
    }，
    ' handlers '：{               ＃处理日志的函数
        ' file '：{
             ' level '： ' DEBUG '，
             ' class '： ' logging.FileHandler '，
             ' filename '： BASE_DIR  +  '/ log / debug.log '，
             ' formatter '： '简单'，
        }，
    }，
    ' loggers '：{
         ' django '：{
             ' handlers '：[ ' file ' ]，
             ' propagate '：是的，
        }，
        ' django.request '：{     ＃日志的命名空间
            ' handlers '：[ ' file ' ]，
             ' level '： ' DEBUG '，
             ' propagate '：是的，
        }，
    }，
}
然后在根目录新建文件夹log。

mkdir log
然后在代码中添加日志相关的代码例如，在。books/views.py中，添加以下代码：

导入日志
logger = logging.getLogger（' django.request '）
在books/views.py中的index函数中添加一行：

logger.info（request.body）
就会发现当我们访问首页的时候，在log/debug.log中有日志信息。

17，中间件的编写
在utils文件夹数中新建middleware.py文件。

来自 django import http
 ＃中间件示例，打印
中间件执行语句类 BookMiddleware（object）：
     def  process_request（self，request）：
         print（“中间件执行”）


类别 AnotherMiddleware（对象）：
     def  process_request（ self， request）：
         print（ “另一个中间件被执行”）＃分别处理收到的请求和发出去的相应，要理解中间件的原理。

    def  process_response（self，request，response）：
         print（“ AnotherMiddleware process_response execution ”）
         返回响应

＃记录用户访问的url地址
类 UrlPathRecordMiddleware（ object）：
     '''记录用户访问的url地址'''
     EXCLUDE_URLS  = [ ' / user / login / '， ' / user / logout / '， ' / user / register / ' ]
    ＃ 1./user/记录url_path = /用户/ 
    ＃ 2./user/login/ url_path = /用户/ 
    ＃ 3./user/login_check/ url_path = /用户/ 
    DEF  process_view（自，请求， view_func， * view_args，** view_kwargs）：
        ＃当用户请求的地址不在排除的列表中，同时不是阿贾克斯的获取请求
        如果 Request的不是 在 UrlPathRecordMiddleware。EXCLUDE_URLS  而 不是 request.is_ajax（）和 request.method ==  ' GET '：
            request.session [ ' url_path ' ] = request.path

BLOCKED_IPS  = []
 ＃拦截在BLOCKED_IPS中的IP 
类 BlockedIpMiddleware（object）：
     def  process_request（self，request）：
         if request。META [ ' REMOTE_ADDR ' ] 中 BLOCKED_IPS：
             返回 http.HttpResponseForbidden（' <H1>禁止</ H1> '）
在然后配置文件settings.py中，写入中间件类的名字。

MIDDLEWARE_CLASSES  =（
     ... 
    ' utils.middleware.BookMiddleware '，
     ' utils.middleware.AnotherMiddleware '，
     ' utils.middleware.UrlPathRecordMiddleware '，
     ' utils.middleware.BlockedIpMiddleware '，
）
这样就可以使用中间件了。
