# Post Office Model

#### Abstract 

This module is used for send socket request with multi threads. 

#### Main Module

##### EasySocketBatchSend

A packaged module to send and receive request list, with three methods open for usage

```python

EasySocketBatchSend(ip, port, worker=50)
"""
        Initializes a new EasySocketBatchSend instance.
        Args:
            ip: socket ip
            port: socket port
            worker: number of threads in threadpool, default 50 
	
"""
```

```python
run_cmd_list(cmd_list)
        """
        Run requests list until all results returned (received or time out)

        Args:
        	cmd_list: list of request, according to current protocol, the full request should be like XXXX, sid\n, yet here only the XXXX part is legal and required, for system will add ,sid\n part automatically.

        Returns:
            result_list : list of result for each request, each result are packed up like [full request, result json], thus the the result list returned looks like [[request1,json1],[requset2,json2],...]

        Raises:
            RuntimeError: If the length of receiving list fails to match the requesting list, or if the func is called after the receiving unit stopped.Error message will be saved in log/socket_log_$datetime$.log

        """
```

```python
quit_receiving()
"""
        Stop receiving to end the whole progress of current socket work.
        This func should be called when no more requests are going to be sent to through current socket connection, or the main progress won't exit
        Once quit_receiving is called, no more message will be sent successfully.

"""
```

##### EasySocketBatch

Core module in realizing the EasySocketBatchSend, including three classes

| Class Name      | Usage                                                        |
| --------------- | ------------------------------------------------------------ |
| EasySocketBatch | A singleton class for control socket connections             |
| NetConnection   | Class used for sending through multiple threads and receiving responds simultaneously |
| ConnSem         | Message carrier unit, used for monitor return(release)       |

#### Example

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @File  : SocketExample.py
# @Author: MoonKuma
# @Date  : 2018/12/13
# @Desc  : This is an example on how to send message with EasySocketBatchSend.py
# Note the func works only in python3

# Step0: import and check if user are running with python 3
from util.valid_python_version import valid_python_major
valid_python_major(3)
from conf import ConfParameters
from util import EasySocketBatchSend
import time
import random
# Step1: get IP address and port for the sever location
sk_conf = ConfParameters.ConfParameters().socket_conf_xf
ip = sk_conf['ip']
port = sk_conf['port']

# Step2: create a work session
easy_batch_send = EasySocketBatchSend.EasySocketBatchSend(ip, port)

# Step3: generate a cmd list
# ***Caution*** : sid for each cmd is added by the system, user should not add sid, nor \n at the end of each cmd
cmd_list = list()
for i in range(0,500):
    cmd_list.append('playerinfo,u72339077604631508')

# Step4: get result list
# result_list contains the cmd and respond for each request in way like [[cmd1,respond1], [cmd2,respond2]...]
time_start = time.time()
pool_result_list = easy_batch_send.run_cmd_list(cmd_list)
time_finish = time.time() - time_start

# Step5: do something to the result
def report_result(result_list, time_cost):
    count = 0
    valid = 0
    match = 0
    for result_pair in result_list:
        count += 1
        key = result_pair[0]
        if result_pair[1] is not None:
            valid += 1
            key_s = key.strip()
            key_list = key_s.split(',')
            if 'uid' in result_pair[1] and len(key_list) > 2 and key_list[1][key_list[1].find('u') + 1:] == (
                    result_pair[1]['uid']):
                match += 1
    msg = '[Result report]Count:' + str(count) + ', valid:' + str(valid) + ', matched:' + str(match) + ', with time cost:' + str(time_cost)
    print(msg)

report_result(pool_result_list, time_finish)

# Step6: quit receiving (***system won't quit without this***)
easy_batch_send.quit_receiving()

# This is the example of result

# [root@VM_141_39_centos statistic_p3]# python3 SocketExample.py
# You are using: Python3
# Log file for EasySocketBatchSend maintained in: /data/tmpStatistic/statistic_p3/log/socket_log_2018-12-14.log
# Log file for NetConnection maintained in: /data/tmpStatistic/statistic_p3/log/socket_log_2018-12-14.log
# 2018-12-14 10:18:35,540,EasySocketBatchSend,INFO,Start committing cmd to socket local 000.000.0.0:00000 with 50workers.
# 2018-12-14 10:18:36,725,EasySocketBatchSend,INFO,Run success! 500 cmd message is sent,500 was returned
# [Result report]Count:500, valid:500, matched:500, with time cost:1.2304916381835938
# Quit receiving as requested, total receiving time:1.2419137954711914
```