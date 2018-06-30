---
title: django开发之项目部署
tags:
categories:
---
# 数据导入
```
import os, django
# 导入django环境
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'simplesite.settings')
# django设置
if 'setup' in dir(django):
    django.setup()

def main():
    from app01.models import Author
    with open(r'name.txt') as f:
        namelist = [Author(name=line) for line in f]
        Author.objects.bulk_create(namelist)
if __name__ == '__main__':
    main()
    print('done')
```
