# -
爬取某菜谱网站的图片
# -*- coding: utf-8 -*-
"""
Created on
爬取网页图片
@author: zhangtianqi5
"""
import os
import re
import urllib
import urllib2
import socket
import sys

FORBIDDEN_CHAR = ['/', '\\', '*', '?', '|', '"', '>', '<', ':']
SAVE_ROOT_PATH = u"D:/组内资料/客流组论文分享/食堂菜品分类/dataset/检索用爬取样本/"

def get_html(url):
    """
    根据给定的网址来获取网页详细信息，得到的html就是网页的源代码
    """
    try:
        page = urllib.urlopen(url)
        hr = page.getcode()
        html = page.read()
    except :
        hr = 404
        html = ""
    return hr, html


def get_links(html, reg):

    html_re = re.compile(reg)
    lis = html_re.findall(html)
    return lis


global myper
def jindu(a, b, c):
    """
    urlretrieve()的回调函数，显示当前的下载进度
    a为已经下载的数据块
    b为数据块大小
    c为远程文件的大小
    """
    if not a:
        print("连接打开")
    if c < 0:
        print("要下载的文件大小为0")
    else:
        global myper
        per = 100 * a * b / c

        if per > 100:
            per = 100
        myper = per
        sys.stdout.write("\r%d%%" % per + ' complete')
        sys.stdout.flush()

    if per == 100:
        return True


def auto_down(url, filename):
    """
    解决urlretrieve下载文件不完全的问题且避免下载时长过长陷入死循环
    """
    try:
        urllib.urlretrieve(url, filename, jindu)
    except socket.timeout:
        count = 1
        while count <= 30:
            try:
                urllib.urlretrieve(url, filename, jindu)
                break
            except socket.timeout:
                err_info = 'Reloading for %d time' % count if count == 1 else 'Reloading for %d times' % count
                print(err_info)
                count += 1
            except IOError:
                print "%s IOError 下载失败!" % url
        if count > 30:
            print("下载失败")
    except IOError:
        print "%s IOError 下载失败!" % url


def get_img(download_img_list, dish_name, dish_class_num):
    """
    根据下载地址列表和菜品名，创建文件夹并下载图片进去
    """
    # 首先检查菜品名是否符合文件夹创立规范
    for char in FORBIDDEN_CHAR:
        if char in dish_name:
            dish_name = dish_name.replace(char, '')

    f = open(SAVE_ROOT_PATH + "img.txt", "a+")
    f.write(dish_name)
    f.write('\n')

    save_dir = os.path.join(SAVE_ROOT_PATH, "%05d_" % dish_class_num + dish_name.decode("utf-8"))

    if os.path.exists(save_dir):
        pass
    else:
        os.makedirs(save_dir)

    x = 0

    nums_per_dish = len(download_img_list)
    if nums_per_dish > 500:
        nums_per_dish = 500
    for idx in range(nums_per_dish):
        img_url = download_img_list[idx]
        f.write(img_url)
        f.write('\n')
        print '\n', img_url
        auto_down(img_url, os.path.join(save_dir, '%05d_%05d.jpg' % (dish_class_num, x)))  # 打开imglist中保存的图片网址，并下载图片保存在本地
        x += 1

    f.close()


if __name__ == "__main__":
    pic_dict = {}
    ori_html = "http://www.douguo.com/caipu/%E5%AE%B6%E5%B8%B8%E8%8F%9C/"
    jc_id = 12
    dish_class_num = 12

    # 最外层循环，在家常菜下，翻页，每页30个链接
    while (1):
        list_1 = []

        # 30个菜品一页
        if jc_id > 0:
            html_1 = ori_html + str((jc_id / 30) * 30)
        else:
            html_1 = ori_html

        # 最外层家常菜网址每页尝试100次
        for i in range(500):
            hr, html = get_html(html_1)

            # 匹配“家常菜”链接下，各个菜品的链接，每页有30个
            list_1 = get_links(html, reg='href="(.+?cookbook.+)" class=')

            # 若抓取到下级链接，则退出尝试循环
            if len(list_1) > 0:
                # jc_id += 30
                break

        # 若该网址尝试100次都未有菜品，则表示翻页到最后，退出外层循环
        if len(list_1) == 0:
            print "家常菜网页下链接打开失败！"
            break

        # 遍历当前页抓取的30个菜品链接
        for link_id in range(30):
            if link_id < (jc_id % 30):
                continue
            link = list_1[link_id]
            jc_id += 1
            dish_name_list = []
            dish_name = ""
            # 菜品类别首页网址每个尝试50次
            for j in range(100):
                hr, html = get_html(link)
                # 先抓取菜品名称
                dish_name_list = get_links(html, reg='<h2>(.+)的做法步骤</h2>')
                # 再根据菜品名称抓取首张图片地址
                if len(dish_name_list) > 0:
                    dish_name = dish_name_list[0]
                    dish_pic_link_list = get_links(html, reg='img src="(.+)" alt=' + '"' + dish_name + '的做法"')
                    if len(dish_pic_link_list) <= 0:
                        print "获取首张图片失败！"
                        continue
                    dish_pic_link = dish_pic_link_list[0]
                    pic_dict[dish_name] = [dish_pic_link]
                    # print pic_dict.keys()
                    break
            if len(dish_name_list) == 0:
                print "菜品类别首页网址打开失败！"
                break

            # 开始抓取alldish
            ori_alldish_link = link[:-5] + '/alldish/'
            hr, alldish_html = get_html(ori_alldish_link)
            if 404 == hr:
                print "%s has no alldish!" % dish_name
                continue

            # 若该类别菜品有alldish
            print "\ndownload: %s..." % dish_name
            alldish_id = 0
            # alldish下面，每12个菜翻页一次
            while (1):
                print alldish_id
                if alldish_id > 600:
                    break

                if alldish_id > 0:
                    alldish_link = ori_alldish_link + str(alldish_id)
                else:
                    alldish_link = ori_alldish_link
                # alldish中每页12个菜的页面打开也尝试50次
                for k in range(100):
                    hr, alldish_html = get_html(alldish_link)
                    all_dish_link_list = get_links(alldish_html, reg='a href="(.+)" target="_blank" title="%s"' % dish_name)
                    if len(all_dish_link_list) > 0:
                        alldish_id += 12
                        break
                if len(all_dish_link_list) == 0 and alldish_id > 800:
                    print "alldish网页打开失败！"
                    break
                elif len(all_dish_link_list) == 0:
                    alldish_id += 12
                    continue

                for one_dish_link in all_dish_link_list:
                    # 每个人上传的菜页面也尝试50次
                    for m in range(100):
                        hr, one_dish_html = get_html(one_dish_link)
                        dish_pic_link_list = get_links(one_dish_html, reg='img src="(.+)" alt=' + '"' + dish_name + '的做法"')
                        if len(dish_pic_link_list) > 0:
                            pic_dict[dish_name].append(dish_pic_link_list[0])
                            break

            # 保存该类别alldish的图片
            if dish_name in pic_dict.keys():
                get_img(pic_dict[dish_name], dish_name, dish_class_num)
                dish_class_num += 1






