---
layout: post
title: Django 源码学习（8）—— password的make和check
category: 技术
tags: Django
keywords:
---

django 的password是加密存储的。 存储在数据库表auth_user中。

# 加密方式 

django < 1.4 加密后的密码是：

    <hashtype>$<salt>$<hash>


    mysql> select * from auth_user limit 1\G;
    *************************** 1. row ***************************
              id: 119
        username: 99fang
      first_name: 
       last_name: 
           email: admin@99fang.com
        password: sha1$8bfb1$54f40454247b0c41c692f5297f8c201904971d35
        is_staff: 1
       is_active: 1
    is_superuser: 1
      last_login: 2014-03-17 10:48:00
     date_joined: 2012-10-26 15:35:25
    1 row in set (0.00 sec)
    
django >= 1.4, 增加了复杂度，具有更高的安全性，加密后的密码是：

    <algorithm>$<iterations>$<salt>$<hash>

    mysql> select * from auth_user limit 2\G;
    *************************** 2. row ***************************
              id: 2
        username: test
      first_name: 
       last_name: 
           email: 
        password: pbkdf2_sha256$10000$jHGIyJOc5mgS$0WLuX+qV+vey3fk56Bfzh/URCPM/y0TQ+8ETLuOXIYo=
        is_staff: 1
       is_active: 1
    is_superuser: 0
      last_login: 2014-06-27 02:58:21
     date_joined: 2014-03-15 03:36:05
    2 rows in set (0.00 sec)

下面只分析django >= 1.4 的情况。

# 生成密码

生成密码的方法叫做make_password，在django/contrib/auth/hashers.py中。

    def make_password(password, salt=None, hasher='default'):
        """
        Turn a plain-text password into a hash for database storage
    
        Same as encode() but generates a new random salt.  If
        password is None or blank then UNUSABLE_PASSWORD will be
        returned which disallows logins.
        """
        if not password:
            return UNUSABLE_PASSWORD
        # 默认使用的是PBKDF2PasswordHasher 
        hasher = get_hasher(hasher)
    
        if not salt:
            salt = hasher.salt()
    
        return hasher.encode(password, salt)

使用的hasher在： django/contrib/auth/hashers.py

    class PBKDF2PasswordHasher(object):
        """
        Secure password hashing using the PBKDF2 algorithm (recommended)
    
        Configured to use PBKDF2 + HMAC + SHA256 with 10000 iterations.
        The result is a 64 byte binary string.  Iterations may be changed
        safely but you must rename the algorithm if you change SHA256.
        """
        algorithm = "pbkdf2_sha256"
        iterations = 10000
        digest = hashlib.sha256
    
        def salt(self):
            """
            Generates a cryptographically secure nonce salt in ascii
            """
            return get_random_string()
    
        def encode(self, password, salt, iterations=None):
            assert password
            assert salt and '$' not in salt
            if not iterations:
                iterations = self.iterations
            hash = pbkdf2(password, salt, iterations, digest=self.digest)
            hash = base64.b64encode(hash).decode('ascii').strip()
            return "%s$%d$%s$%s" % (self.algorithm, iterations, salt, hash)

加密方法pbkdf2在：django/utils/crypto.py

    def pbkdf2(password, salt, iterations, dklen=0, digest=None):
        """
        Implements PBKDF2 as defined in RFC 2898, section 5.2
    
        HMAC+SHA256 is used as the default pseudo random function.
    
        Right now 10,000 iterations is the recommended default which takes
        100ms on a 2.2Ghz Core 2 Duo.  This is probably the bare minimum
        for security given 1000 iterations was recommended in 2001. This
        code is very well optimized for CPython and is only four times
        slower than openssl's implementation.
        """
        assert iterations > 0
        if not digest:
            digest = hashlib.sha256
        password = force_bytes(password)
        salt = force_bytes(salt)
        hlen = digest().digest_size
        if not dklen:
            dklen = hlen
        if dklen > (2 ** 32 - 1) * hlen:
            raise OverflowError('dklen too big')
        l = -(-dklen // hlen)
        r = dklen - (l - 1) * hlen
    
        hex_format_string = "%%0%ix" % (hlen * 2)
    
        def F(i):
            def U():
                u = salt + struct.pack(b'>I', i)
                for j in xrange(int(iterations)):
                    u = _fast_hmac(password, u, digest).digest()
                    yield _bin_to_long(u)
            return _long_to_bin(reduce(operator.xor, U()), hex_format_string)
    
        T = [F(x) for x in range(1, l + 1)]
        return b''.join(T[:-1]) + T[-1][:r]

# 校验密码

校验密码的方法叫做check_password，在django/contrib/auth/hashers.py中。

    def check_password(password, encoded, setter=None, preferred='default'):
        """
        Returns a boolean of whether the raw password matches the three
        part encoded digest.
    
        If setter is specified, it'll be called when you need to
        regenerate the password.
        """
        if not password or not is_password_usable(encoded):
            return False
    
        # 默认使用的是PBKDF2PasswordHasher 
        preferred = get_hasher(preferred)
        hasher = identify_hasher(encoded)
    
        # 如果数据库中存储的加密方式和代码中的不一致(可能是django版本升级)，重新生成加密密码
        must_update = hasher.algorithm != preferred.algorithm
        is_correct = hasher.verify(password, encoded)
        if setter and is_correct and must_update:
            setter(password)
        return is_correct

使用的hasher在： django/contrib/auth/hashers.py

    class PBKDF2PasswordHasher(object):
        def verify(self, password, encoded):
            """ 使用数据库中的algorithm salt iterations 正向加密一下"""
            algorithm, iterations, salt, hash = encoded.split('$', 3)
            assert algorithm == self.algorithm
            encoded_2 = self.encode(password, salt, int(iterations))
            return constant_time_compare(encoded, encoded_2)

Tips: 比较两个加密后的字符串有个小技巧：

    def constant_time_compare(val1, val2):
        """
        Returns True if the two strings are equal, False otherwise.
    
        The time taken is independent of the number of characters that match.
    
        For the sake of simplicity, this function executes in constant time only
        when the two strings have the same length. It short-circuits when they
        have different lengths. Since Django only uses it to compare hashes of
        known expected length, this is acceptable.
        """
        if len(val1) != len(val2):
            return False
        result = 0
        if six.PY3 and isinstance(val1, bytes) and isinstance(val2, bytes):
            for x, y in zip(val1, val2):
                result |= x ^ y
        else:
            for x, y in zip(val1, val2):
                result |= ord(x) ^ ord(y)
        return result == 0

这个方法比val1 == val2 慢，但是每次执行的时间都是固定的，因此可以防计时攻击。有关这个问题的讨论：https://groups.google.com/forum/#!topic/django-developers/iAaq0pvHXuA
