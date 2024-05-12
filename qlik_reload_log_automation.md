## Script for Reload Log Automation

Below is the script used to automate reload logs in Qlik Sense. 
- You can copy this into your Qlik environment to set up similar functionality.
- **** indicates redacted information / where you would add in your server-specific information

## Script

```qlik
Let vServerPath = '[lib://ServerLogFolder/Scheduler/Trace]';
//////////////////////// map reload status & create time interval batches for reload //////////////////////////////

Map_Reload_Status:
Mapping Load * Inline [
ReloadStatus, NewStatus
'Execution State Change to FinishedFail', 'Failed'
'Execution State Change to FinishedSuccess', 'Success'
];

Time_Batch_Map:
LOAD * INLINE [
StartTime, EndTime, Batch
12:00:00 AM, 06:59:59 AM, 1
07:00:00 AM, 10:30:59 AM, 2
10:31:00 AM, 12:59:59 PM, 3
01:00:00 PM, 11:59:59 PM, 4
];

//////////////////////// load in reload data ///////////////////////////////////
SET TimeFormat='h:mm:ss TT';

[Server_System_Scheduler]:
LOAD
    Timestamp(Timestamp#(replace(left("Timestamp", 19), 'T', ' '), 'YYYYMMDD hhmmss.fff'),'MM-DD-YYYY hh:mm:ss') as [Timestamp],    
    Thread,
    Id,
    ApplyMap('Map_Reload_Status', [Message], [Message]) as [Reload Status],
    TaskName,
    TaskId,
    ExecutionId,
    AppName,
    AppId,
    RowNo() as RecordKey
FROM [$(vServerPath)/****_System_Scheduler.txt]
(txt, utf8, embedded labels, delimiter is '\t', msq)
WHERE WildMatch(Message, '*FinishedSuccess*', '*FinishedFail*')
AND NOT Match(AppName, 'Operations Monitor', 'License Monitor');

[Temp_Server_System_Scheduler]:
NoConcatenate
Load
*, 
time(frac(Timestamp),'$(TimeFormat)') as TimeOnly 
RESIDENT [Server_System_Scheduler];

DROP TABLE [Server_System_Scheduler];
RENAME TABLE [Temp_Server_System_Scheduler] TO [Server_System_Scheduler];


IntervalMatch:
IntervalMatch (TimeOnly)
LOAD DISTINCT StartTime, EndTime
RESIDENT Time_Batch_Map;

LEFT JOIN (IntervalMatch)
LOAD StartTime, EndTime, Batch
RESIDENT Time_Batch_Map;

LEFT JOIN (IntervalMatch)
LOAD TimeOnly, RecordKey
RESIDENT [Server_System_Scheduler];

LEFT JOIN ([Server_System_Scheduler])
LOAD RecordKey, Batch
RESIDENT IntervalMatch;


DROP TABLE IntervalMatch;
DROP TABLE Time_Batch_Map;


///////////////////////// create inline table to store data //////////////////////

Total_Reload_Status:
LOAD * INLINE [
AppName, TaskName, Reload Status, Timestamp, Reload Batch
];

/////////////////////////// join to inline table //////////////////
Concatenate (Total_Reload_Status)
LOAD AppName,
TaskName,
[Reload Status],
[Timestamp],
Batch as [Reload Batch]
RESIDENT [Server_System_Scheduler]
;
Drop Table [Server_System_Scheduler];


//////////////// HTML Content //////////////////////////
// let vEmailFile = '[lib://ServerLogFolder/Scheduler/Trace/EmailOutput.html]';

sub Encode(vEncodeMe, vEncoded)
let vEncoded = replace(replace(replace(replace(replace(replace(replace(vEncodeMe, ':', '%3a'), '/', '%2f'), '?', '%3f'), '=', '%3d'), '\', '%5c'), '@', '%40'), ' ', '+');
end sub


EMailOutput:
LOAD
[<!--EMailOutput-->]
INLINE [
<!--EMailOutput-->
<html>
<head>
<style>
tr:first-child td {font-weight: bold; background-color: #dddddd;}
h3 {font-family: Arial; font-size: 12pt;}
td {border-left:1px solid #555555; border-top:1px solid #555555; font-family: Arial; font-size:9pt; text-align:left; padding: 2px 10px 2px 10px;}
table {border-right:1px solid #555555; border-bottom:1px solid #555555; border-collapse:collapse;}
.success { color: green; }
.failed { color: red; font-weight: bold; }
</style>
</head>
<body>
<h3>The following applications were reloaded today on the **** server:</h3>
<table>
<tr><td>AppName</td><td>TaskName</td><td>Reload Status</td><td>Timestamp</td><td>Reload Batch</td></tr>
];

CONCATENATE(EMailOutput)
LOAD
'<tr><td>' & AppName &
'</td><td>' & TaskName &
'</td><td>' & [Reload Status] &
'</td><td>' & Timestamp &
'</td><td>' & [Reload Batch] & '</td></tr>' as [<!--EMailOutput-->]
RESIDENT Total_Reload_Status
ORDER BY [Reload Status], [Timestamp] DESC;


CONCATENATE(EMailOutput)
LOAD
[<!--EMailOutput-->]
INLINE [
<!--EMailOutput-->
</table>
</body>
</html>
];

//////////////////////////////// Count the successful and failed tasks /////////////////////
Let vFileDate = Date(Today(),'MM-DD-YYYY');
TaskCount:
LOAD 
    sum(if([Reload Status] = 'Success', 1, 0)) as SuccessCount,
    sum(if([Reload Status] = 'Failed', 1, 0)) as FailedCount,
    Max([Reload Batch]) as BatchNo
RESIDENT Total_Reload_Status
WHERE Date(floor(Timestamp), 'MM-DD-YYYY') = '$(vFileDate)';
Let vSuccessCount = Peek('SuccessCount', 0, 'TaskCount');
Let vFailedCount = Peek('FailedCount', 0, 'TaskCount');
Let vBatchNo = Peek('BatchNo', 0, 'TaskCount'); 
DROP TABLE TaskCount;
/////////////////////////////////////////////////////////////////////

STORE EMailOutput into [$(vServerPath)/Reload_Status_Logs/$(vFileDate)_****_Batch_$(vBatchNo).html] (txt);

////////////////// preparing SMTP Settings ///////////////////////
Let vDate = Date(Today(),'MM/DD/YYYY');
Let vSMTPserver = '****';
Let vFromName = '****';
Let vSMTPPort = '25';
Let vToEmail = '****';
Let vFromEmail = '****';
Let vCCEmail = ****' ;
Let vSubject = '$(vDate)+****+Reload+Status+:+Task+Failed+-+$(vFailedCount)';  
Let vMessage = '%40file%3dC%3a%5cProgramData%5cQlik%5cSense%5cLog%5cScheduler%5cTrace%5cReload_Status_Logs%5c$(vFileDate)_****_Batch_$(vBatchNo).html';

///////////////////////////////////////// Sending Email/////////////////////////////////////////

SMTPConnector_SendEmail:
LOAD
    status as SendEmail_status,
    result as SendEmail_result,
    filesattached as SendEmail_filesattached
FROM [lib://QlikWebConnectors]
(URL IS [http://****/data?connectorID=SMTPConnector&table=SendEmail&SMTPServer=$(vSMTPserver)&Port=$(vSMTPPort)&SSLmode=None&to=$(vToEmail)&subject=$(vSubject)&message=$(vMessage)&html=True&fromEmail=$(vFromEmail)&cc=$(vCCEmail)&fileAttachment1=C%3a%5cProgramData%5cQlik%5cSense%5cLog%5cScheduler%5cTrace%5cReload_Status_Logs%5c$(vFileDate)_****_Batch_$(vBatchNo).html&delayInSeconds=0&ignoreProxy=False&appID=&loadAccessToken=****], qvx);  
exit script;
