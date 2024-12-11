When running the `exe`, the following error occurs:

```shell
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

The error message to focus on is:

```
Exception: need to copy the sdk/platforms/android-?/data/res/values/public.xml here
```

Information found online: https://github.com/Nuitka/Nuitka/issues/2289

The issue was resolved by adding a custom YAML configuration:

```
- module-name: "apkutils.axml"
  data-files:
    patterns:
      - "public.xml"
```

This indicates that the issue is due to `apkutils` being unable to find the `public.xml`.

You can find the `public.xml` file in the local `site-packages` directory, as this file is a dependency of `apkutils`.

![image-20241127151515088](The program packaged with PyInstaller that depends on apkutils cannot run_imgs\TYcT2xYAz2V.png)

The error is raised by `public.py`, which expects the `public.xml` file to be in the same path.

![image-20241127151645215](The program packaged with PyInstaller that depends on apkutils cannot run_imgs\55HkfA4FNw1.png)

**Although PyInstaller does not support YAML configuration, the same effect can be achieved using the `--add-data` command.**

You can add the configuration in the PyInstaller command as follows:

```
pyinstaller --add-data="F:\.dev\PYTHON\Lib\site-packages\apkutils\axml\public.xml;apkutils/axml" py_source_file.py
```