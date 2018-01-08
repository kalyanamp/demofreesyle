# demo freesyle
#!/usr/bin/python
import sys, os, json, time
import argparse
import traceback
import boto3
from botocore.exceptions import ClientError
import math
import numpy

logsclient = boto3.client('logs')
lambdaclient = boto3.client('lambda')
region='us-east-1'
MONTHLY = "MONTHLY"
MS_MAP = {MONTHLY: (3.6E6) * 720}

#function = 'LogS3DataEvents'
minutes = 1000000  # in minutes
mem_used_array = []
duration_array = []
prov_mem_size = 0
firstEventTs = 0
lastEventTs = 0
ts_format = "%Y-%m-%d %H:%M:%S UTC"
outputs= []
#log_group_name = '/aws/lambda/' + function
#compute_time = 0
#number_of_requests = 0
i = 0
compute_charge = 0
windowStartTime = (int(time.time()) - minutes * 60) * 1000
firstEventTs = windowStartTime  # temporary value, it will be updated once (if) we get results from the CW Logs get_log_events API
lastEventTs = int(time.time() * 1000)  # this will also be updated once (if) we get results from the CW Logs get_log_events API
nextLogstreamToken = True

log_groups = logsclient.describe_log_groups(logGroupNamePrefix='/aws/lambda/')['logGroups']

for log_group in log_groups:
    log_group_name=log_group['logGroupName']
    billed_duration = []
    compute_time = 0
    number_of_requests = 0
    all_bills = []
    billed_times_freq = {}
    memory_list = {}
    memory_duration = {}
    mem_used_list = {}
    logstreamsargs = {'logGroupName': log_group_name, 'orderBy': 'LastEventTime', 'descending': True}
    logstreams = logsclient.describe_log_streams(**logstreamsargs)
    #print(log_group_name)
    print(log_group['logGroupName'])
    if 'logStreams' in logstreams:
         #print("Number of logstreams found:[{}]".format(len(logstreams['logStreams'])))

         # Go through all logstreams in descending order
         for ls in logstreams['logStreams']:
             nextEventsForwardToken = True
             logeventsargs = {'logGroupName': log_group_name, 'logStreamName': ls['logStreamName'],
                              'startFromHead': True, 'startTime': windowStartTime}
             while nextEventsForwardToken:
                 logevents = logsclient.get_log_events(**logeventsargs)
                 if 'events' in logevents:
                     if len(logevents['events']):
                         #print("\nEvents for logGroup:[{}] - logstream:[{}] - nextForwardToken:[{}]".format(log_group_name, ls['logStreamName'], nextEventsForwardToken))
                         for e in logevents['events']:
                             # Extract lambda execution duration and memory utilization from "REPORT" log events
                             if 'REPORT RequestId:' in e['message']:
                                 mem_used = e['message'].split('Max Memory Used: ')[1].split()[0]
                                 mem_used_array.append(int(mem_used))
                                 duration = e['message'].split('Billed Duration: ')[1].split()[0]
                                 billed_duration.append(int(duration))
                                 duration_array.append(int(duration))
                                 mem_size = int(e['message'].split('Memory Size: ')[1].split()[0])
                                 if i == 0:
                                     prov_mem_size = int(e['message'].split('Memory Size: ')[1].split()[0])
                                     firstEventTs = e['timestamp']
                                     lastEventTs = e['timestamp']
                                 else:
                                     if e['timestamp'] < firstEventTs: firstEventTs = e['timestamp']
                                     if e['timestamp'] > lastEventTs: lastEventTs = e['timestamp']
                                 #print (e['timestamp'])
                                 #print(e['message'])
                                 i += 1
                                 memory_list[mem_size] = memory_list.get(mem_size, 0) + 1
                                 memory_duration[mem_size] = memory_duration.get(mem_size, 0) + int(duration)
                                 mem_used_list[mem_size] = mem_used_list.get(mem_size,0) + int(mem_used)
                         if (prov_mem_size != mem_size):
                             prov_mem_size = mem_size
                     else:
                         break

                 nextEventsForwardToken = logevents.get('nextForwardToken', False)
                 if nextEventsForwardToken:
                     logeventsargs['nextToken'] = nextEventsForwardToken
                 else:
                     logeventsargs.pop('nextToken', False)
    #print(memory_list)
    #print(memory_duration)
    #print(mem_used_list)
    for k, v in memory_list.items():

        compute_time = float(memory_duration[k])/1000
        number_of_requests = v
        mem_used_avrg = round(int(mem_used_list[k])/int(v),2)
        mem_percet = round(float(100 * mem_used_avrg / int(k)), 2)
        #request_charge = number_of_requests * 0.0000002 * 10000000
        request_charge = float(number_of_requests * 0.2)
        compute_charge = float(compute_time * 0.00001667 * int(k) / 1024)
        Total_charge = float(compute_charge + request_charge)
        output = {"Memory Size": k, "total Billed Duration": memory_duration[k], "number_of_requests": v, "Avege Memory Used":mem_used_avrg, "memory in percent":mem_percet,"Toatal charge":int(Total_charge)}
        print(json.dumps(output,sort_keys=False,indent=4))
