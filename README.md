django session
今天注册功能遇到个问题。验证码不正确，但是我他妈输入的是正确的验证码啊，我本地测试是好的啊，为什么显示不行呢？更蛋疼的是，有的时候是好的，有的时候不行，nmb。

跟源码
验证码的实现过程

 
    def get(self, request, *args, **kwargs):
        """
        获取验证码
        ---
        """
        uid = str(uuid.uuid4())
        mp_src = hashlib.md5(uid.encode("UTF-8")).hexdigest()
        text = mp_src[0:4]
        request.session["captcha_code"] = text
        request.session.save()
        font_path = current_path + "/www/static/www/fonts/Vera.ttf"
        logger.debug("======> font path " + str(font_path))
        font = ImageFont.truetype(font_path, 22)
​
        size = self.getsize(font, text)
        size = (size[0] * 2, int(size[1] * 1.4))
​
        image = Image.new('RGBA', size)
​
        try:
            PIL_VERSION = int(NON_DIGITS_RX.sub('', Image.VERSION))
        except:
            PIL_VERSION = 116
        xpos = 2
​
        charlist = []
        for char in text:
            charlist.append(char)
​
        for char in charlist:
            fgimage = Image.new('RGB', size, '#001100')
            charimage = Image.new('L', self.getsize(font, ' %s ' % char), '#000000')
            chardraw = ImageDraw.Draw(charimage)
            chardraw.text((0, 0), ' %s ' % char, font=font, fill='#ffffff')
            if PIL_VERSION >= 116:
                charimage = charimage.rotate(random.randrange(*(-35, 35)), expand=0, resample=Image.BICUBIC)
            else:
                charimage = charimage.rotate(random.randrange(*(-35, 35)), resample=Image.BICUBIC)
            charimage = charimage.crop(charimage.getbbox())
            maskimage = Image.new('L', size)
​
            maskimage.paste(charimage, (xpos, from_top, xpos + charimage.size[0], from_top + charimage.size[1]))
            size = maskimage.size
            image = Image.composite(fgimage, image, maskimage)
            xpos = xpos + 2 + charimage.size[0]
​
        image = image.crop((0, 0, xpos + 1, size[1]))
​
        ImageDraw.Draw(image)
​
        out = StringIO()
        image.save(out, "PNG")
        out.seek(0)
​
        response = HttpResponse(content_type='image/png')
        response.write(out.read())
        response['Content-length'] = out.tell()
​
        return response
整个实现过程比较简单，跟常见的验证码实现方式一样。这里主要用到了python的 PIL 用于生成图片。生成随机验证码后将验证码保存在session里 request.session["captcha_code"] = text。

验证码的校验过程

 
real_captcha_code = request.session.get("captcha_code")
​
if real_captcha_code is None or captcha_code is None or real_captcha_code.lower() != captcha_code.lower():
    raise forms.ValidationError(
        self.error_messages['captcha_code_error'],
        code='captcha_code_error'
    )
整个过程没有什么问题，本地运行也666。但是打包到容器运行失败（公司采用docker方式运行应用）。

 

填坑
session和cookie
session和cookie，session是保存在服务器端，cookie存储在浏览器上，我们称为客户端，客户端向服务端发送请求时，会将cookie一起发送给服务端。服务端接收到请求后，会去检查是否已经有该客户端的session信息，如果没有，则创建一个新的session对象，用于保存客户端的一些必要信息，如果从服务器上找到了该客户端的信息，则会将该信息加载到session里

django session相关配置
 
# Cache to store session data if using the cache session backend.
SESSION_CACHE_ALIAS = 'default'  # 这个值对应CACHES里面的key
# Cookie name. This can be whatever you want.
SESSION_COOKIE_NAME = 'sessionid'
# The module to store session data
SESSION_ENGINE = 'django.contrib.sessions.backends.db'
# class to serialize session data
SESSION_SERIALIZER = 'django.contrib.sessions.serializers.JSONSerializer'
​
#########
# CACHE #
#########
​
# The cache backends to use.
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    }
}
使用django的session机制需要在settings文件中添加如下配置

中间件配置

 
MIDDLEWARE_CLASSES = (
    ......
    'django.contrib.sessions.middleware.SessionMiddleware',
    ...... 
)
安装app配置

 
INSTALLED_APPS = ('django.contrib.sessions',)
添加配置后使用python manage.py migrate 会在数据库生成对应的数据表。

我发现我们项目中处理的时候自定义了配置里的SESSION_ENGINE属性。

 
SESSION_ENGINE = "www.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = 'session'
SESSION_COOKIE_DOMAIN = '.goodrain.com'
SESSION_COOKIE_AGE = 3600
关于django session的SESSION_ENGINE

SESSION_ENGINE的主要配置有5个。

缓存类配置django.contrib.sessions.backends.cache

Set SESSION_ENGINE to "django.contrib.sessions.backends.cache" for a simple caching session store. Session data will be stored directly in your cache. However, session data may not be persistent: cached data can be evicted if the cache fills up or if the cache server is restarted.

缓存类加持久化配置 django.contrib.sessions.backends.cached_db

For persistent, cached data, set SESSION_ENGINE to "django.contrib.sessions.backends.cached_db". This uses a write-through cache -- every write to the cache will also be written to the database. Session reads only use the database if the data is not already in the cache.

数据库持久化配置 django.contrib.sessions.backends.db 默认方式

Once you have configured your installation, run manage.py migrate to install the single database table that stores session data.

文件存放session配置 django.contrib.sessions.backends.file

To use file-based sessions, set the SESSION_ENGINE setting to"django.contrib.sessions.backends.file".

You might also want to set the SESSION_FILE_PATH setting (which defaults to output from tempfile.gettempdir(), most likely /tmp) to control where Django stores session files. Be sure to check that your Web server has permissions to read and write to this location.

基于cookie的session配置 django.contrib.sessions.backends.signed_cookies

To use cookies-based sessions, set the SESSION_ENGINE setting to"django.contrib.sessions.backends.signed_cookies". The session data will be stored using Django's tools for cryptographic signing and the SECRET_KEY setting.

详细可以参看这里

我发现我们的SESSION_ENGINE被自定义过。代码如下

 
from django.contrib.sessions.backends.cache import SessionStore as BaseSessionStore
​
import logging
logger = logging.getLogger('default')
​
​
class SessionStore(BaseSessionStore):
​
    def load(self):
        try:
            logger.debug('session', 'start load key {}'.format(self.cache_key))
            session_data = self._cache.get(self.cache_key, None)
        except Exception, e:
            logger.error('session', 'load key {} error.'.format(self.cache_key))
            logger.exception('session', e)
            # Some backends (e.g. memcache) raise an exception on invalid
            # cache keys. If this happens, reset the session. See #17810.
            session_data = None
        if session_data is not None:
            logger.debug('session', 'got data for key {}: {}'.format(self.cache_key, session_data))
            return session_data
        self.create()
        return {}
​
这段代码其实没有什么好讲的。只是继承了原有的 django.contrib.sessions.backends.cache 的 SessionStore 类，然后copy原有的load方法加上了logger。

填坑记

在生成验证码的时候加上一句代码

 
request.session.save()
结果不管用

将配置里 SESSION_ENGINE 去掉，则会使用默认的 django.contrib.sessions.backends.db 。结果成功。

小记
原因没找到，乱改很折腾。
