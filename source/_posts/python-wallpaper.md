title: 爱壁纸爬虫
date: 2017-08-11 10:21:18
tags:
  - python
  - 爬虫
---

声明：
> 如有侵权，邮件`627014295@qq.com`告知，四十八小时内删除
> 涉及图片格式，需要安装PIL

<!-- more -->

```python
import os
import json
import uuid
from PIL import Image
from urllib import urlopen
from urllib import urlretrieve
import shutil

def generateUUID():
  return str(uuid.uuid1()).replace('-', '')

def concatParams(url, params): 
  first = True
  strParams = ''
  for k, v in params.items(): 
    if first :
      first = False
    else :
      strParams += '&'
    strParams += k + '=' + str(v)
  return url + '?'+ strParams

def getCategories():
  url = 'http://api.lovebizhi.com/android_v3.php'
  params = { 
    'client_id' : 1001, 
    'model_id' : 100,
    'screen_width' : 1080,
    'screen_height' : 1920,
    'bizhi_width' : 1080,
    'bizhi_height' : 1920,
    'uuid' : generateUUID(),
    'a' : 'browse',
    'id' : 'category'
  }
  request = concatParams(url, params)
  response = urlopen(request).read()
  return response

def touch(name):  
  if not os.path.exists(name):  
    file = open(name,'a+')  
    file.close()

def mkdir(name): 
  if os.path.exists(name):
    shutil.rmtree(name)
  os.mkdir(name)  

def download(url, file):
  touch(file)
  urlretrieve(url, file)

def webp2png(file):
  image = Image.open(file)
  image.save(file.replace('.webp', '.png'), "PNG")
  os.remove(file)

def fetchCategory(category):
  url = category['url']
  name = category['name']
  icon = category['icon']
  file = name+'/icon.webp'
  mkdir(name)
  download(icon, file)
  webp2png(file)
  response = json.loads(urlopen(url).read())
  data = response['data']
  for item in data:
    vip = item['image']['vip_original']
    image = name + vip[vip.rfind("/"):]
    download(vip, image)

def main():
  categories = json.loads(getCategories())
  for category in categories :
    fetchCategory(category)

main()

```
