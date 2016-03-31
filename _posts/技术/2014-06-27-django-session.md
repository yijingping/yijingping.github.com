---
layout: post
title: Django 源码学习（7）—— session是如何存储的
category: 技术
tags: Django
keywords:
---

django 的session可以存在redis cache database file中，最方便的是使用database存储。

# 存储
django 的session数据存储在数据库表django_session中, 客户端则用key为sessionid的cookie标记。

    mysql> desc django_session;
    +--------------+-------------+------+-----+---------+-------+
    | Field        | Type        | Null | Key | Default | Extra |
    +--------------+-------------+------+-----+---------+-------+
    | session_key  | varchar(40) | NO   | PRI | NULL    |       |
    | session_data | longtext    | NO   |     | NULL    |       |
    | expire_date  | datetime    | NO   | MUL | NULL    |       |
    +--------------+-------------+------+-----+---------+-------+
    3 rows in set (0.00 sec)

取出一条数据看一下：

    *************************** 80. row ***************************
    session_key: pga1ykmzyba4fmv6b34l8c3kbqqynwbt
    session_data: M2E2ZjcyNGZmYzA2ZjcyNjViY2NjMDI0YTkzNjNmNDgyMTZlYmRmNzqAAn1xAShVEl9hdXRoX3VzZXJfYmFja2VuZHECVSlkamFuZ28uY29udHJpYi5hdXRoLmJhY2tlbmRzLk1vZGVsQmFja2VuZHEDVQ1fYXV0aF91c2VyX2lkcQSKAQJ1Lg==
    expire_date: 2014-07-11 02:58:21
    80 rows in set (0.00 sec)

其中session_key是客户端cookie中sessionid的值:

![django sessionid](/assets/images/django/sessionid.png)

session_data是加密的json数据，存储了user_id和其它自定义session, 用下面的方法可以根据一个sessionid来解密session_data。

    from django.contrib.sessions.models import Session
    from django.contrib.auth.models import User
    
    session_key = 'pga1ykmzyba4fmv6b34l8c3kbqqynwbt'
    
    session = Session.objects.get(session_key=session_key)
    data = session.get_decoded()
    print data

    uid = data.get('_auth_user_id')
    user = User.objects.get(pk=uid)

    print user.username, user.get_full_name(), user.email

    # 输出
    In [10]: print data
    {'_auth_user_id': 2L, '_auth_user_backend': 'django.contrib.auth.backends.ModelBackend'}

    In [11]: print user.username, user.get_full_name(), user.email
    test  

session_data数据加解密的方式也很简单(base64和序列化)：

    def _hash(self, value):
        key_salt = "django.contrib.sessions" + self.__class__.__name__
        return salted_hmac(key_salt, value).hexdigest()

    def encode(self, session_dict):
        # 将session dict 序列化
        serialized = self.serializer().dumps(session_dict)
        # 加入随机值
        hash = self._hash(serialized)
        # 拼接字符串"随机值:序列化数据",然后用base64加密
        return base64.b64encode(hash.encode() + b":" + serialized).decode('ascii')

    def decode(self, session_data):
        encoded_data = base64.b64decode(force_bytes(session_data))
        try:
            # could produce ValueError if there is no ':'
            hash, serialized = encoded_data.split(b':', 1)
            expected_hash = self._hash(serialized)
            if not constant_time_compare(hash.decode(), expected_hash):
                raise SuspiciousSession("Session data corrupted")
            else:
                return self.serializer().loads(serialized)
        except Exception as e:
            # ValueError, SuspiciousOperation, unpickling exceptions. If any of
            # these happen, just return an empty dictionary (an empty session).
            if isinstance(e, SuspiciousOperation):
                logger = logging.getLogger('django.security.%s' %
                        e.__class__.__name__)
                logger.warning(force_text(e))
            return {}

如果只是想看看解密的数据，也可用下面这段简单的代码:

    import pickle,base64
    session_data = "M2E2ZjcyNGZmYzA2ZjcyNjViY2NjMDI0YTkzNjNmNDgyMTZlYmRmNzqAAn1xAShVEl9hdXRoX3VzZXJfYmFja2VuZHECVSlkamFuZ28uY29udHJpYi5hdXRoLmJhY2tlbmRzLk1vZGVsQmFja2VuZHEDVQ1fYXV0aF91c2VyX2lkcQSKAQJ1Lg=="
    encoded_data = base64.b64decode(session_data)
    hash, serialized = encoded_data.split(b':', 1)
    pickle.loads(serialized)
    # 输出：
    {'_auth_user_backend': 'django.contrib.auth.backends.ModelBackend',
    '_auth_user_id': 2L}
