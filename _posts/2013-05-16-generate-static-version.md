如何给web的静态文件生成版本号
=============================================
web页面中，静态文件：图片、css、js等会被浏览器缓存，如果我们更新了这些文件，浏览器却没有重新加载，就会出现错乱的情况。

解决这个问题的简单方法是给静态文件的url加一个动态参数，因为url不同，所以浏览器会重新加载。

# 实现方法
---------------------
1) 在my_context_processors.py中, 添加代码: 

```
from django.conf import settings

def common_context_processor(request):
    return {
        'STATIC_URL': settings.STATIC_URL,
        'STATIC_VERSION': settings.STATIC_VERSION
    }

```

2) 在settings.py中添加：

```
 # add for static file                                       
 STATIC_URL = 'http://res_zhangrui.99fang.com'               
 STATIC_VERSION = '201305161510' # 每次静态文件修改后，将该版本号改成当前的时间日期                            
 TEMPLATE_CONTEXT_PROCESSORS = (                             
     'my_context_processors.common_context_processor',    
 )                                                           
```

3) 在views.py中, 使用：

```
from django.shortcuts import render_to_response
from django.template import RequestContext

def my_view(request, ...):  
    response_dict = {} 
    ...
    # you can still still add variables that specific only to this view
    response_dict['some_var_only_in_this_view'] = 42
    ...
    return render_to_response('my_template_dir/my_template.html', response_dict, context_instance=RequestContext(request))
``` 

或者

```
from django.shortcuts import render_to_response
from django.template import RequestContext

def my_view(request, ...):  
    response_dict = RequestContext(request)
    ...
    # you can still still add variables that specific only to this view
    response_dict['some_var_only_in_this_view'] = 42
    ...
    return render_to_response('my_template_dir/my_template.html', response_dict)
```

4) 在my_template_dir/my_template.html中，使用动态版本号:

```
<!doctype html>
<html>
	<head>
		<link rel="stylesheet" rev="stylesheet" href="{{STATIC_URL}}/css/a.css?ver={{STATIC_VERSION}}" type="text/css" />
	</head>
	<body>
		<div class="logo">
			<img src="{{STATIC_URL}}/img/logo.png?ver={{STATIC_VERSION}}" />
		</div>
		<script type="text/javascript" src="{{STATIC_URL}}/js/footer.js?ver={{STATIC_VERSION}}"></script>
	</body>
</html>
```

