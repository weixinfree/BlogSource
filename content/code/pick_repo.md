---
title: "Python+Github搭建免费，无限容量的markdown图床"
date: 2018-11-27T16:39:18+08:00
draft: true
---

# Details makes perfect

经常写markdown，图片的管理是一个不那么方便的事情。网络上的图片说不定那天就不能访问了；本地的图片显示更麻烦一些，和各个markdown编辑器的支持有关。

之前都是本地使用python启动一个静态文件服务器`python3 -m http.server`。优势是非常简单，但是劣势也非常明显，只能局域网内访问；每次打开markdown，还要记得启动下文件服务器，用起来颇为不爽。

这对于追求效率，简洁，优美的程序员来说无法忍受。所以花了一点时间，使用python + github 搭建了一个免费的，无限容量的图床，而且还不用自己运维，需求方只有自己，需求变化不大。可谓理想中的完美的系统了。


核心代码如下:

```python
class PicRepo:

    def add(self, name: str, url: str = None):
        file: Path = None
        if url:
            _file = _download(url)
            file = Path(_file)
        elif Path(name).exists():
            file = Path(name)

        dst = BASE / name

        shutil.copyfile(file, dst)
        _clear()

        os.chdir(BASE.parent)
        sh = partial(check_call, shell=True)
        sh(f'git add {BASE}/ 2>/dev/null')
        sh(f'git commit -m "add new images" 2>/dev/null')
        sh(f'git push 2>/dev/null')

        print()
        print(f'![{name}](https://raw.githubusercontent.com/{your account}/{repo}/{branch}/images/{name})')

```

> `functools.partial`非常nice，结合`subprocess.check_call`调用shell，看起来就有一种美感

使用requests进行下载网络图片：

```python
def _download(url: str) -> str:
    file_name = 'tmp_' + md5(url.encode('utf-8')).hexdigest() + '.png'
    try:
        rsp = requests.get(url).content
        with open(file_name, mode='wb') as f:
            f.write(rsp)

        return file_name
    except Exception as e:
        raise SystemExit(-1, e)

def _clear():
    for f in Path('.').glob('tmp_*'):
        print('remove tmpfile: ', f)
        os.remove(f)
```

默认建立了一个顶层文件夹

```python
BASE = Path(__file__).parent / 'images'
```

import 如下：

```python
import requests
from hashlib import md5
import shutil
from pathlib import Path
import os
from subprocess import check_call
from functools import partial
```

> for human 的requests 和 Path

还有非常nice的google Fire:
```python
if __name__ == "__main__":
    import fire
    fire.Fire(PicRepo)
```