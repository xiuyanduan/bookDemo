2016年 03月 15日 星期二 10:48:16 CST
```python
#!/usr/bin/env python
#coding:utf8
import urllib,urllib2,sys,requests
from bs4 import BeautifulSoup
reload(sys)
sys.setdefaultencoding( "utf-8" )

# url of each market,and method of its beautifulsoup
anzhuoshichang=["http://apk.hiapk.com/search?key=%E8%AE%B0%E5%81%A5%E5%BA%B7&pid=0",".select('.list_version')[0].text"]
#anzhuoshichang=["http://apk.hiapk.com/appinfo/com.ciji.jjk",".select('#appSoftName')[0].text"]
anzhishichang=["http://www.anzhi.com/soft_2434545.html",".select('.app_detail_version')[0].text"]
yingyonghui=["http://www.appchina.com/app/com.ciji.jjk",".select('.app-setup')[0]['meta-versionname']"]
jifengshichang=["http://apk.gfan.com/Product/App1055191.html",".select('.app-info')[0].p.text"]
youyishichang=["http://www.eoemarket.com/soft/751145.html",".select('.info_appintr')[0].em.next_sibling.text"]
mumayi=["http://www.mumayi.com/android-1012276.html",".select('.iappname')[0].text"]
nduoshichang=["http://www.nduoa.com/search?sk=f2a6f7c1e13e9321f70ad32c839c5c23&q=%E8%AE%B0%E5%81%A5%E5%BA%B7",".select('.version')[0].text"]
#nduoshichang=["http://www.nduoa.com/apk/detail/922139",".select('.version')[0].text"]
zhushou91=["http://apk.91.com/soft/android/search/1_5_0_0_%E8%AE%B0%E5%81%A5%E5%BA%B7",".select('.zoom')[0].span.em.text"]
#zhushou91=["http://apk.91.com/Soft/Android/com.ciji.jjk-9.html",".select('#Version_btn')[0].previous_sibling"]
wandoujia=["http://www.wandoujia.com/apps/com.ciji.jjk",".find_all('dd')[2].next_sibling.next_sibling.next_sibling.next_sibling.text"]
zhushou360=["http://zhushou.360.cn/detail/index/soft_id/3067141",".select('.base-info')[0].tr.next_sibling.next_sibling.td.text"]
shougou=["http://zhushou.sogou.com/apps/detail/561940.html",".select('.info')[0].tr.td.next_sibling.next_sibling.text"]
tengxun=["http://android.myapp.com/myapp/detail.htm?apkName=com.ciji.jjk",".select('.det-othinfo-data')[0].text"]
baidu=["http://shouji.baidu.com/s?wd=%E8%AE%B0%E5%81%A5%E5%BA%B7&data_type=app&f=header_app%40input%40btn_search&from=as",".select('.little-install')[1].a['data_versionname']"]
#baidu=["http://shouji.baidu.com/soft/item?docid=8011339",".select('.version')[0].text"]
taobao=["http://android.25pp.com/detail_6631124.html",".select('.gradeStar2')[0].next_sibling.next_sibling.li.text"]
wo=["",""]
samlung=["",""]
xiaomi=["http://app.mi.com/detail/108005",".select('.details')[0].ul.li.next_sibling.next_sibling.next_sibling.text"]
lenovo=["",""]
meizu=["http://app.meizu.com/apps/public/detail?package_name=com.ciji.jjk",".select('.app_content')[3].text"]
huawei=["http://appstore.huawei.com/app/C10333234",".select('.ul-li-detail')[3].text"]
oppo=["",""]
jinli=["http://www.anzhuoapk.com/apps-83-1/143411/",".select('.content1_top_txt')[0].h1.text"]

def getHTML(url):
    user_agent = 'Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)'
    values = {'name' : 'Michael Foord',
              'location' : 'Northampton',
              'language' : 'Python' }
    headers = { 'User-Agent' : user_agent }
    data = urllib.urlencode(values)
    req = urllib2.Request(url, data, headers)
    response = urllib2.urlopen(req)
    html_page = response.read()
    return html_page

# some markets must add headers to use urllib2
def getVersionAddheaders(appurl,beautifulsoupMethod):
    html=getHTML(appurl)
    return eval("BeautifulSoup(html, 'html.parser')"+beautifulsoupMethod)

# use requests.get
def getVersionRequests(appurl,beautifulsoupMethod):
    html=requests.get(appurl).text
    return eval("BeautifulSoup(html, 'html.parser')"+beautifulsoupMethod)

# come markets could get the html by urllib2.urlopen
def getVersion(appurl,beautifulsoupMethod):
    html=urllib2.urlopen(appurl)
    return eval("BeautifulSoup(html, 'html.parser')"+beautifulsoupMethod)


if __name__=="__main__":
    marketName=sys.argv[1]
    if marketName=='mumayi' or marketName=='zhushou91':
        print getVersionAddheaders(eval(marketName)[0],eval(marketName)[1])
    elif marketName=='yingyonghui':
        print getVersionRequests(eval(marketName)[0],eval(marketName)[1])
    else:
        print getVersion(eval(marketName)[0],eval(marketName)[1])

```
