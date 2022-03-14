---
layout: post
title:  "某网站每日自动登录并签到的三种python脚本及云服务器上的部署"
date:   2017-09-13 20:31:06
---

基于本人每日要在某网站签到的需求，但苦于总是忘记签到（尤其是周末），遂即决定自己写一个python小脚本来实现自动登录网站并自动签到的功能，然后再将其部署到一个免费的云服务平台设定每日自动执行。

由于是python新手，所以前前后后参考了很多代码，最终先后写了三种不同的脚本来实现，个人觉得各有其优劣点。三种脚本详述如下：

 

<strong>脚本一</strong>：

主要使用模块Selenium + geckodriver

Selenium模块本是设计来用于网络自动化测试的，但它也完全可以被用来做一些自动化任务，比如这里的登录和签到。这里用到的是它的webdriver然后配合火狐浏览器驱动geckodriver（也有Chrome的驱动，这里可以自己选，据说火狐的稳定，所以选了火狐）。此脚本运行时你会看到火狐浏览器启动→加载登录页面→输入网站用户名和密码并登入→触发签到的整个自动化过程。代码如下（此处特意隐去网站信息和本人账户密码信息）：
{% highlight ruby %}
#-*- coding: utf-8 -*-
from selenium import webdriver
import time
import logging
import sys
reload(sys)
sys.setdefaultencoding('utf-8')

def sign(b):

    #登录并签到
    b.find_element_by_id("Phone").send_keys("*******")
    b.find_element_by_id("Password").send_keys("*******")
    b.find_element_by_xpath("//*[@onclick='login()']").click()
    b.find_element_by_xpath("//*[@class='btn-sign']").click()

b=webdriver.Firefox()
b.get("http://www.***.com")
sign(b)

# 构建日志文件
logging.basicConfig(level=logging.DEBUG, 
    format='%(asctime)s %(levelname)s %(message)s', 
    datefmt='%a, %d %b %Y %H:%M:%S', 
    filename='./log.log', 
    filemode='a')
logging.info('Successfully signed!')

print "Successfully signed!"
{% endhighlight %}

总结：
优点：适合在windows系统运行；可以看到整个自动化过程；不需要提供headers等信息
缺点：运行过程较慢；依赖浏览器；不适合在云服务平台运行。


<strong>脚本二</strong>：
主要使用模块Selenium +phantomjs

这个思路其实和脚本1一致，只是整个自动化过程在后台运行，后台模仿浏览器但不会启动浏览器。
代码如下：
{% highlight ruby %}
#-*- coding: utf-8 -*-
from selenium import webdriver
import time
import logging
import sys
reload(sys)
sys.setdefaultencoding('utf-8')

b=webdriver.PhantomJS('phantomjs')
b.set_window_size(1600, 900)
b.get("http://www.***.com")

def sign(b):
    b.find_element_by_id("Phone").send_keys("*******")
    b.find_element_by_id("Password").send_keys("*******")
    b.find_element_by_xpath("//*[@onclick='login()']").click()
    b.find_element_by_xpath("//*[@class='btn-sign']").click()
    s = b.find_elements("css selector", ".btn-sign")
    c = b.find_element_by_class_name("p2")
    log = '%s, %s!'%(s[0].text, c.text)
    print log

    #日志文件的建立
    logging.basicConfig(level=logging.DEBUG, 
    format='%(asctime)s %(levelname)s %(message)s', 
    datefmt='%a, %d %b %Y %H:%M:%S', 
    filename='./log.log', 
    filemode='a')   
   logging.info(log) 
sign(b)
{% endhighlight %}

总结：
优点：windows系统和linux系统都适合运行；自动化过程在后台；不必启动浏览器；不需要提供headers等信息
缺点：运行过程较慢；不大适合在云服务平台运行


<strong>脚本三</strong>：
主要使用模块Requests

前两种脚本都是用selenium模仿浏览器来执行登录和签到操作，此脚本则是仅利用requests模块来执行操作，不涉及模仿或驱动浏览器的过程。脚本3的编写过程中要检测用户向服务器post的内容，尽管Chrome和火狐都有自带的网络监测工具，但我还是推荐Fiddler，它要更专业一些，而且能很清晰地检测到你签到时向服务器post的信息。
代码如下：

{% highlight ruby %}
#-*- coding: utf-8 -*-
import requests
from bs4 import BeautifulSoup 
def sign():

    #登录及签到post数据准备
    s = requests.session()
    log_data = {'Phone':'*******','Password':'*******'}
    sgn_data = {'id':'********'}

    #登录操作
    log = s.post('http://www.***.com', log_data)
    html = s.get(‘www.***.com/***')

    #签到操作
    s.post('www.***.com/sign', sgn_data)

    #提取签到结果并打印
    page = s.get('h‘www.***.com/***').content
    soup = BeautifulSoup(page,"html.parser")
    txt1 = soup.find_all('a', attrs={"class":"btn-sign"})[0].get_text()
    txt2 = soup.find_all('p', attrs={"class":"p2"})[0].get_text()

   print (txt1+', '+txt2) 

sign()
{% endhighlight %}

总结：
优点：windows系统和linux系统都适合运行；后台自动化；不必启动浏览器；只需要python模块，依赖少；较前两种脚本稳定；运行速度快
缺点：暂无


云服务平台（PaaS）上脚本的部署：
本人先后尝试了国内外多家云服务平台，包括Pythonanywhere，新浪sae和Heroku，最终安家Heroku。Pythonanywhere的局限是它不接受访问除了它的whitelist之外的任何网站，除非你要访问的网站有API才可以申请，所以果断放弃。新浪sae其实并不是免费的，但它超便宜，也基本上就是免费了，但新浪sae的最大缺点是限制比较多而且使用不灵活，整个系统感觉就是模仿国外类似平台的，操作不灵活而且部署说明啰嗦，整个平台有点邯郸学步的感觉，为了珍惜时间我转战Heroku。Heroku首先整个界面就很有品很友好，即使整个平台都是英文但也都要比新浪sae的中文部署说明可读性好得多。由于是第一次部署，折腾了一个晚上终于也搞好了。建议把代码放到Github上，然后在Heroku调用，这样会方便的多。你的Github项目文件根目录下需要这两个文件：yourapp.py，requirements.txt。yourapp.py即你要运行的脚本（这里用的上述脚本3），文件名随你起，注意heroku中默认的python版本是3.X，所以这里的print语句要加括号。requirements.txt中需要列出你的脚本运行需要的模块极其版本信息。例如我的requirements.txt内容如下：
{% highlight ruby %}
Flask==0.12.2
certifi==2017.7.27.1
beautifulsoup4==4.6.0
idna==2.6
chardet==3.0.4
requests==2.18.4
urllib3==1.22
{% endhighlight %}

如果既想让你的脚本做后台运行同时又想让它能在网页上显示运行结果。那么你的根目录下还需要有另外两个文件，一个是名称为Procfile的文件，除此之外还要把你的yourapp.py复制一份在根目录并加上一个最简单的flask的web框架然后重新命名（如yourappweb.py）, 这个脚本将作为网页调用的脚本。Procfile内容如下：

{% highlight ruby %}
web: gunicorn yourappweb:app --log-file –
{% endhighlight %}

加有flask框架的yourappweb.py如下：
{% highlight ruby %}
import flask
import requests
from bs4 import BeautifulSoup

app = flask.Flask(__name__)
@app.route("/")

def sign():
 
    s = requests.session()
    log_data = {'Phone':'*******','Password':'*******'}
    sgn_data = {'id':'*******'}
 
    log = s.post('http://www.***.com', log_data)
    html = s.get(www.***.com/***)
    page = html.content
    s.post('http:/www.***.com/sign', sgn_data)

    soup = BeautifulSoup(page,"html.parser")
    txt1 = soup.find_all('a', attrs={"class":"btn-sign"})[0].get_text()
    txt2 = soup.find_all('p', attrs={"class":"p2"})[0].get_text()
    return txt1+', '+txt2

if __name__ == "__main__":
    app.run()
{% endhighlight %}

注意这个脚本是用于网页，所以签到结果要用return而非print，否则无法在网页显示签到结果。

最后你需要的是在Heroku上安装名叫Heroku Scheduler的add-on。虽然安装它的时候需要提供下你的信用卡信息，但这个add-on标明是免费的，不会扣你钱。Heroku Scheduler上你就可以设置你的脚本自动执行的频率了，上面要填的执行命令就是$python yourapp.py。注意Heroku Scheduler用的是UTC时间，所以如果要推算对应的北京时间就是用UTC时间+8小时。所以最终设定每日UTC时间0：00（也就是北京时间8：00）自动执行。

至此某网站的每日自动登录并签到功能就全部实现了。
