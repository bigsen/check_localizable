## 目录
* 背景
* 如何详细定位多语言错误信息
* 全自动定位多语言错误信息

---

### 一、背景
* iOS 多语言文件，是APP在需要做国际化的时候用到的一种文件，例如：**`Localzable.strings`** 文件。

* 做国际化的APP肯定有遇到多语言文件中，字符串少一个分号，多一个引号等情况，这个时候Xcode就会出现编译错误，但是Xcode遇到这种编译错误 是不会报出来具体的行数的，例如：
* ![](https://upload-images.jianshu.io/upload_images/790890-b9a8ac4b915008fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 如果APP中的多语言文件非常多，那么由于Xcode并不会报出具体的行数，那么在查找错误的时候就会非常费事，非常耗费时间和精力去定位到具体的哪一行多语言文件错误了。

* 本篇文章主要说明如何 **`利用 plutil 命令 去定位多语言错误`** 和 **`自动定定位多语言文件格式错误的脚本介绍`**。

#### 二、如何详细定位多语言错误信息
* 主要利用到了 plutil 命令
##### （1）plutil 命令 介绍

* plutil 命令 是 Mac OS 自带的一个命令。
* 利用 plutil 可校验 plist，多语言文件 是否有错误。可进行操作 plist 文件，可进行格式转换等功能。
* 本篇文章主要利用 plutil 语法检测的作用，更多详细用法，例如参考：https://blog.csdn.net/cneducation/article/details/54729106
##### （2）plutil 命令 使用
* 帮助信息：
* ![](https://upload-images.jianshu.io/upload_images/790890-73e779e3254b7942.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 例如 检测多语言文件：plutil 后面跟文件名即可。
* ![](https://upload-images.jianshu.io/upload_images/790890-1d5c068874eb7b7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 从上图可看到，如果多语言文件有误，通过 plutil 是可以定位具体错误，行数 等信息的。

* 但是可以体会到，这样还是会有一些缺点，不方便，你需要手动去指定所有多语言文件，去检测。下面来说一下，如何全自动定位多语言文件错误信息。

 #### 三、全自动定位多语言错误信息
#### 使用方法：
（1）放在项目中，直接执行python脚本。
（2）脚本支持参数，第一个参数为搜索路径，第二个参数为输出日志路径（当有参数时，非必传）。
（3）参数不传，默认当前脚本执行路径。

#### 示例：
* 无参数，默认目录为脚本当前执行目录
```
python check_localizable.py
```

* 指定搜索多语言路径
```
python check_localizable.py 搜索路径
python check_localizable.py /Users/sen/Desktop/项目
```

* 指定搜索路径 和 报告生成路径
```
python check_localizable.py 搜索路径 日志输出路径
python check_localizable.py /Users/sen/Desktop/项目 /Users/sen/Desktop/duoyuy
```

#### 演示：
* 模拟多语言为错误格式 ----> 运行脚本校验多语言 -----> 生成报告 ----> 修复多语言格式 
* ![](https://upload-images.jianshu.io/upload_images/790890-dc019a466284f7e8.gif?imageMogr2/auto-orient/strip)

#### 脚本流程
* ![](https://upload-images.jianshu.io/upload_images/790890-637b7a6cd0d62291.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 脚本源码：
```
# coding=utf-8
import os
import re
import sys
import commands
import linecache
from os import path


# 多語言錯誤信息模型
class StingInfo(object):
    # 多语言文件名
    file_name = ""

    # 多语言文件路径
    file_path = ""

    # 多语言错误信息
    error_message = ""

    # 对应错误行数
    line_number = ""

    # 多语言内容
    line_content = ""
    line_exception = 0


# 多语言自动校验类
class LocalizableManager(object):
    # 初始化
    def __init__(self, file_path, out_path):
        # 存放多语言文件
        self.strings = [StingInfo]

        # 初始化搜索目录为当前
        self.search_dir = file_path

        # 指定生成目录
        self.report_file = out_path

        # 是否自动打开多语言文件
        self.is_open_file = 1

        # 是否自动打开日志
        self.is_open_log = 1

    # 获取文件后缀名
    @staticmethod
    def file_extension(file_path):
        return os.path.splitext(file_path)[1]

    # 取出多语言提示错误
    @staticmethod
    def get_error(msg):
        left = "plist parser: "
        right = "on line"
        list_content = re.findall(left + "(.+?)" + right, msg)
        if len(list_content):
            return list_content[0]
        else:
            return msg

    # 检查是否是异常错误
    @staticmethod
    def exception(msg):
        left = "plist parser: "
        right = "on line"
        list_content = re.findall(left + "(.+?)" + right, msg)
        return len(list_content)

    # 获取错误行数
    @staticmethod
    def get_line(msg):
        left = "on line"
        right = "Parsing will be abandoned"
        line = re.findall(left + "(.+?)" + right, msg)
        if len(line):
            line_number = line[0]
            return int(float(line_number))
        else:
            return 0

    # 执行shell命令
    @staticmethod
    def shell(run):
        os.system(run)

    # 写入文件到本地
    def write_file(self, msg):
        self.f.write(msg + "\n")

    # 写入报告
    def write_analysis(self, info):

        self.write_file("---------------------")
        if not info.line_exception:
            self.write_file("文件行数：" + str(info.line_number))
            self.write_file("错误信息：" + info.error_message)
            self.write_file("文件路径：" + info.file_path)
            self.write_file("错误多语言：" + info.line_content)
        else:
            self.write_file("错误信息：" + info.line_content)

    # 写入到文件
    def generate_analysis(self):
        for str_info in self.strings:
            if str_info.file_name:

                # 打开错误文件
                if self.is_open_file:
                    self.shell("open " + str_info.file_path)

                # 生成错误报告
                self.write_analysis(str_info)

        manager.f.flush()
        manager.f.close()

    # 打开报告
    def open_report(self):
        if self.is_open_log:
            self.shell("open " + self.report_file)

    @staticmethod
    def contain_english(str0):
        import re
        return bool(re.search('[a-z]', str0))

    def get_content(self, line_number, fi_d):

        # 获取对应行内容
        content = linecache.getline(fi_d, line_number)

        if not self.contain_english(content):
            content = ""

        # 直到获取到内容
        while not content.strip():
            line_number -= 1
            content = linecache.getline(fi_d, line_number)
        return content

    # 遍历寻找本地多语言错误
    def find_localizable(self, dir_path):
        dirs = os.listdir(dir_path)
        for f in dirs:
            fi_d = os.path.join(dir_path, f)
            if os.path.isdir(fi_d):
                self.find_localizable(fi_d)
            else:

                # 判断是否是多语言文件
                if self.file_extension(f) == ".strings":
                    (status, output) = commands.getstatusoutput('plutil ' + fi_d)

                    # 判断是否有错误
                    if status:
                        # 创建多语言错误模型
                        info = StingInfo()
                        info.file_name = f
                        info.file_path = fi_d
                        info.error_message = self.get_error(output)
                        info.line_number = self.get_line(output)

                        # 判断是否有异常
                        if self.exception(output):

                            # 取出对应错误行数的多语言，进行上下适配
                            content1 = self.get_content(info.line_number - 1, fi_d)
                            content2 = self.get_content(info.line_number, fi_d)
                            content3 = self.get_content(info.line_number + 1, fi_d)

                            if content2 == content1:
                                content2 = ""

                            if content3 == content2 or content1:
                                content3 = ""

                            info.line_content = "\n" + content1 + content2 + content3
                        else:
                            info.line_content = output
                            info.line_exception = 1

                        # 模型加入list
                        self.strings.append(info)

        return len(self.strings) - 1


# main 函数开始执行
if __name__ == '__main__':

    # 支持参数传递，第一个参数为搜索路径，第二个参数为输出日志路径
    # 例如 python check_localizable.py /users/sen/desktop /users/sen/log
    # 如果路径不存在会走默认路径，从脚本执行当前路径进行遍历

    # 获取用户输入参数
    argv_count = len(sys.argv)

    # 报告文件名称
    report_file = "/localizable_repo.txt"

    # 指定默认参数
    search_dir = os.getcwd()
    output_dir = search_dir + report_file

    # 判断是否传递python 参数
    if argv_count == 3:
        if path.exists(sys.argv[1]):
            search_dir = sys.argv[1]
        if path.exists(sys.argv[2]):
            output_dir = sys.argv[2] + report_file

    # 处理未传输出路径
    if argv_count == 2:
        if path.exists(sys.argv[1]):
            search_dir = sys.argv[1]

    # 多语言管理对象
    manager = LocalizableManager(search_dir, output_dir)

    # 是否自动打开出错多语言文件
    manager.is_open_file = 1

    # 是否自动打开多语言报告日志
    manager.is_open_log = 1

    # 寻找本地多语言错误
    print "开始校验多语言文件。。。。。。"
    if not manager.find_localizable(manager.search_dir):
        print "多语言文件，格式正确。"
    else:
        manager.f = open(manager.report_file, "w+")
        manager.generate_analysis()
        manager.open_report()
        print "已经生成报告：" + manager.report_file
```
####  注意
* 通过plutil 命令和脚本，大部分多语言错误都能定位，小部分特别的错误，还需手动查看。
