运行exe时报错如下：

```java
Traceback (most recent call last):
  File "create_release_jira.py", line 47, in <module>
  File "<frozen importlib._bootstrap>", line 1176, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1147, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 690, in _load_unlocked
  File "PyInstaller\loader\pyimod02_importers.py", line 391, in exec_module
  File "submit_gerrit.py", line 48, in <module>
  File "<frozen importlib._bootstrap>", line 1176, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1147, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 690, in _load_unlocked
  File "PyInstaller\loader\pyimod02_importers.py", line 391, in exec_module
  File "mandroid.py", line 1, in <module>
  File "<frozen importlib._bootstrap>", line 1176, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1147, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 690, in _load_unlocked
  File "PyInstaller\loader\pyimod02_importers.py", line 391, in exec_module
  File "apkutils\__init__.py", line 1, in <module>
  File "<frozen importlib._bootstrap>", line 1176, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1147, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 690, in _load_unlocked
  File "PyInstaller\loader\pyimod02_importers.py", line 391, in exec_module
  File "apkutils\apk.py", line 12, in <module>
  File "<frozen importlib._bootstrap>", line 1176, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1147, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 690, in _load_unlocked
  File "PyInstaller\loader\pyimod02_importers.py", line 391, in exec_module
  File "apkutils\axml\__init__.py", line 12, in <module>
  File "<frozen importlib._bootstrap>", line 1176, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1147, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 690, in _load_unlocked
  File "PyInstaller\loader\pyimod02_importers.py", line 391, in exec_module
  File "apkutils\axml\public.py", line 22, in <module>
      
Exception: need to copy the sdk/platforms/android-?/data/res/values/public.xml here
[6428] Failed to execute script 'create_release_jira' due to unhandled exception!
```



用最后这一条报错`Exception: need to copy the sdk/platforms/android-?/data/res/values/public.xml here`

在网上搜到的信息：https://github.com/Nuitka/Nuitka/issues/2289

人家是通过添加yaml配置解决的：

```
Fixed with a custom package config

- module-name: "apkutils.axml"
  data-files:
    patterns:
      - "public.xml"
```

说明问题在于apkutils找不到public.xml。

可以找到本机的site-packages目录，这个public.xml是apkutils自身的依赖数据

![image-20241127151515088](pyinstaller打包依赖apkutils的程序无法运行_imgs\5mxZUNIpKAF.png)

报错也是public.py抛出的，即public.py希望同路径下有public.xml文件

![image-20241127151645215](pyinstaller打包依赖apkutils的程序无法运行_imgs\itukhqrWkqD.png)

**虽然我们使用pyinstaller无法配置yaml，但利用--add-data命令也可以实现一样的效果**

在pyinstaller命令中增加配置就可以了，如下：

```
pyinstaller --add-data="F:\.dev\PYTHON\Lib\site-packages\apkutils\axml\public.xml;apkutils/axml" py源文件.py
```





