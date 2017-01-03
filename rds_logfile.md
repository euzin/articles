# RDS LogFile Downloader

### rds logfile 
> rds log file 을 확인하기 위해서 매번 aws console 을 이용하기는 매우 불편함.
> log file 을 특정 위치에 백업 할 필요가 있거나 여러 account 의 aws 를 관리 하는 경우 log 관리

- RDS 관리용 서버에서 cli 형태로 원하는 rds instance 의 최근 로그를 확인할 수 있는 스크립트를 작성한다.
- python boto 를 이용 한다. 
	- rds.download_db_logfile_portion()
		- 로그 파일을 다운로드 받을때 사용
		- [ boto 문서 ](http://boto3.readthedocs.io/en/latest/reference/services/rds.html#RDS.Client.download_db_log_file_portion)
	- rds.describe_db_logfile_list()
		- 로그 파일 리스트 확인 
		- [ boto 문서 ](http://boto3.readthedocs.io/en/latest/reference/services/rds.html#RDS.Client.describe_db_log_files)

### code:

```
#!/usr/bin/env python
#-*- coding:utf-8 -*-

import os, sys
import boto3
import datetime
import pytz
import subprocess

# boto3 rds init
rds = boto3.client('rds')

def todate(unixtime):
    kst=pytz.timezone('Asia/Seoul')
    utc = pytz.utc
    return datetime.datetime.utcfromtimestamp(unixtime).replace(tzinfo=utc).astimezone(kst).strftime('%Y-%m-%d %H:%M:%S')

def header():
    print '='*90

def footer():
    print '='*90

def list_db_logfile(dbid):
    res = rds.describe_db_log_files(
              DBInstanceIdentifier = dbid
              )
    loglistres = res['DescribeDBLogFiles']
    for i in range (len(loglistres)):
        log_file_name  = loglistres[i]['LogFileName']
        log_file_time  = loglistres[i]['LastWritten']
        log_file_size  = loglistres[i]['Size']
        print todate(log_file_time/1000) , ' : ' +  log_file_name + ' ' , log_file_size/1024, '(kbytes)'

def write_db_logfile(dbid, logfilename):
    cwd = os.getcwd()
    ymd = datetime.datetime.now(pytz.timezone('Asia/Seoul')).strftime('%Y%m%d%H%M%S')
    localfile = cwd + '/rdslog.' + ymd
    token = '0'
    f = open(localfile, 'w')
    try:
        res = rds.download_db_log_file_portion(
                DBInstanceIdentifier = dbid,
                LogFileName = logfilename,
                Marker = token
                )
        while res['AdditionalDataPending']:
            f.write(res['LogFileData'])
            print "> write log file"
            token = res['Marker']
            res = rds.download_db_log_file_portion(
                    DBInstanceIdentifier = dbid,
                    LogFileName = logfilename,
                    Marker = token
                    )
        f.write(res['LogFileData'])
        print "> write log file"
        print "> done "
    except error as e :
        print e
        sys.exit(2)
    f.close()
    return localfile

def list_db_instances():
    res = rds.describe_db_instances()

    if len(res['DBInstances']) > 0:
        header()
        print ' : DB Instances : '
        footer()

    dbres = res['DBInstances']

    for i in range(len(dbres)):
            db_id                   = dbres[i]['DBInstanceIdentifier']
            db_status               = dbres[i]['DBInstanceStatus']
            print '- InstanceIdentifier : ' + db_id + ' (' + db_status + ')'

def main():
    list_db_instances()
    footer()
    dbid = raw_input("Enter db-instance-identifier : ")
    header()
    list_db_logfile(dbid)
    logfilename = raw_input("Enter logfile name : ")
    localfilename = write_db_logfile(dbid,logfilename)
    cmd = "vi " + localfilename
    subprocess.call(cmd, shell=True)

if __name__ == '__main__':
    sys.exit(main())
```


