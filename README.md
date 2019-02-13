# 引入字典处理模块、系统指令模块、xml解析模块
import json
import os
# coding=utf-8
from xml.etree import ElementTree as ET
import config
from multiprocessing import Process, Pool
import json

# 解析xml
tree = ET.parse('broker.xml')
tree_pl = []
if config.ping_num == 0:
    tree_path_0 = tree.findall('./broker')[:5]
    tree_path_1 = tree.findall('./broker')[5:10]
    tree_path_2 = tree.findall('./broker')[10:15]
    tree_path_3 = tree.findall('./broker')[15:20]
    tree_path_4 = tree.findall('./broker')[20:25]
    tree_path_5 = tree.findall('./broker')[25:30]
    tree_path_6 = tree.findall('./broker')[30:35]
    tree_path_7 = tree.findall('./broker')[35:40]
    tree_path_8 = tree.findall('./broker')[40:45]
    tree_path_9 = tree.findall('./broker')[45:50]
    tree_path_10 = tree.findall('./broker')[50:]
    tree_pl = [tree_path_0,
               tree_path_1,
               tree_path_2,
               tree_path_3,
               tree_path_4,
               tree_path_5,
               tree_path_6,
               tree_path_7,
               tree_path_8,
               tree_path_9,
               tree_path_10]
else:
    tree_path = tree.findall('./broker')[:config.ping_num]
    tree_pl = [tree_path]
    # 不想测那么多，切片可以用1
ip_dict = {}    # 这个字典是用在最后输出期货商和电信商


nu = 0
num = 0
for tree in tree_pl:
    # 解析的结果是给tcping的一个‘ip（域名） 端口’文件
    with open('./jumbled/tlist%s.txt' % num, 'w') as f:
        # print(tree_path)
        for ma in tree:
            # print('\n' + ma.attrib['BrokerName'])
            for nat in ma:
                for zeroper in nat:
                    for oneper in zeroper:
                        telecom = 'none'
                        if oneper.tag == 'Name':
                            telecom = oneper.text
                            # print(oneper.text, ':')
                        if oneper.tag == 'MarketData':
                            i = 1
                            for child_one in list(oneper):
                                ip_dict[ma.attrib['BrokerName'] + ' - ' + telecom + str(i)] = child_one.text
                                i += 1
                                content = child_one.text.replace(':', ' ')
                                # print(content)
                                f.write(content + '\n')
                                nu += 1
    num += 1
    # print(ip_dict)
# print(nu)


# 储存字典以备最后解析
js = json.dumps(ip_dict)
with open('./jumbled/ip_dict.txt', 'w') as dict_file:
    dict_file.write(js)

tcli = []
for i in range(11):
    tcli.append(['result%i.txt' % i, 'tlist%i.txt' % i])


# 调用tcping命令
def tcpinger(tu):
    result = str(tu[0])
    tlist = str(tu[1])
    comd = 'tcping -w 0.5 --tee ./jumbled/%s --fqdn --file ./jumbled/%s'% (result, tlist)
    os.system(comd)


if __name__ == '__main__':
    poll = Pool(11)
    ret = poll.map(tcpinger, tcli)
    poll.close()
    poll.join()


txt_read = []
for i in range(11):
    file = open("./jumbled/result%i.txt" % i, "r")
    for line in file:  # 设置文件对象并读取每一行文件
        linez = line.strip()
        if len(linez) == 0:
            pass
        else:
            if linez[0] == 'P':
                txt_read.append(linez)  # 将每一行文件加入到list中


# 把tcping的结果重新统计，小于20ms保存到1个txt格式文件中，名字叫fast_id
ping = []
for lineo in txt_read:

    # 一些字段的读取
    linet = lineo.strip('Probing')
    linef = linet.split('/')[0].strip()
    linefi = linet.split('/')[1].split('time=')[1]

    # 构建元组列表
    key = float(linefi.strip('ms'))
    ping.append((key, linef))

# 排序在这里！
ping = sorted(ping)

# 从字典中读期货公司和电信商
file = open('./jumbled/ip_dict.txt', 'r')
js = file.read()
ip_dict = json.loads(js)
# print(ip_dict)

# 小于某个值的构建一个列表，用于最后输出成txt
fast_result = []
fast_delay = config.fast_delay
fast_result.append('ping小于' + str(fast_delay) + 'ms的服务器是:\n')
for ms_ip in ping:
    ip_address = str(ms_ip[1])
    jet_leg = ms_ip[0]
    if ms_ip[0] < fast_delay:
        for key, value in ip_dict.items():
            if ip_address == value:
                fast_result.append('\n%-13s' % key)
        fast_result.append(ip_address + ', 用时: ' + str(jet_leg) + '(ms)')
filename_less = str.format('fast_delay_%s.txt' % fast_delay)

with open(filename_less, 'w') as file_fast:
    for text in fast_result:
        file_fast.write(text + '\n')

# 排序
fast_rank = config.fast_rank
fast_ranks = ['最快的' + str(fast_rank) + '个服务器是:\n']
for ms_ip in ping[0:fast_rank]:
    ip_address = str(ms_ip[1])
    jet_leg = ms_ip[0]
    for key, value in ip_dict.items():
        if ip_address == value:
            fast_ranks.append('\n%-13s' % key)
    fast_ranks.append(ip_address + ', 用时: ' + str(jet_leg) + '(ms)')

filename = str.format('fast_rank_%s.txt' % fast_rank)
with open(filename, 'w') as file_rank:
    for text in fast_ranks:
        file_rank.write(text + '\n')
