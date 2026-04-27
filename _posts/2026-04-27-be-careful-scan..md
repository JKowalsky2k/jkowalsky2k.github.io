## Be careful with scan function

Scan is powerfull tool which helps you detects chains/patters in series of events. But recently I discovered that not is easy to skip valuable data using it. Lets consider below query (taken from documentation and modified a little bit):
| Variable | Description      |
|----------|------------------|
| m_id     | Shows Pattern Id |
| Step     | Shows results thats corresponds and maching particular step condition |

# Exmaple 1:
Results as expected - two patters with all step corelated once. So all events covered.
```kql
let Events = datatable (Ts: timespan, Event: string, Data: string) [
    0m, "A", "a",
    1m, "Start", "start a",
    3m, "C", "c",
    4m, "Stop", "stop a",
    6m, "C", "c",
    8m, "Start", "start c",
    11m, "E", "e",
    12m, "Stop", "stop b"
]
;
Events
| sort by Ts asc
| scan with_match_id=m_id declare (Step: int) with 
(
    step s1: Event == "Start" => Step = 1;
    step s2: Event != "Start" and Event != "Stop" and Ts - s1.Ts <= 5m => Step = 2;
    step s3: Event == "Stop" and Ts - s1.Ts <= 5m => Step = 3;
)
```
Results:
| Ts       | Event | Data    | Step | m_id |
|----------|-------|---------|------|------|
| 00:01:00 | Start | start a | 1    | 0    |
| 00:03:00 | C     | c       | 2    | 0    |
| 00:04:00 | Stop  | stop a  | 3    | 0    |
| 00:08:00 | Start | start c | 1    | 1    |
| 00:11:00 | E     | e       | 2    | 1    |
| 00:12:00 | Stop  | stop b  | 3    | 1    |


# Example 2:
In results we can see that both ("start a" and "start b") ware matched as m_id = 0. This might be a problem because If we wanted to use some values from Step 1 in Step 2 and both events in Step 1 are equaly valuable, Step 2 see last event (if we would use eg. s1.data, s1.data will be equal to "start b").  
```kql
let Events = datatable (Ts: timespan, Event: string, Data: string) [
    0m, "A", "a",
    1m, "Start", "start a",
    2m, "Start", "start b", // Added new event 
    3m, "C", "c",
    4m, "Stop", "stop a",
    6m, "C", "c",
    8m, "Start", "start c",
    11m, "E", "e",
    12m, "Stop", "stop b"
]
;
Events
| sort by Ts asc
| scan with_match_id=m_id declare (Step: int) with 
(
    step s1: Event == "Start" => Step = 1;
    step s2: Event != "Start" and Event != "Stop" and Ts - s1.Ts <= 5m => Step = 2;
    step s3: Event == "Stop" and Ts - s1.Ts <= 5m => Step = 3;
)
```
Results:
| Ts       | Event | Data    | Step | m_id |
|----------|-------|---------|------|------|
| 00:01:00 | Start | start a | 1    | 0    |
| 00:02:00 | Start | start b | 1    | 0    |
| 00:03:00 | C     | c       | 2    | 0    |
| 00:04:00 | Stop  | stop a  | 3    | 0    |
| 00:08:00 | Start | start c | 1    | 1    |
| 00:11:00 | E     | e       | 2    | 1    |
| 00:12:00 | Stop  | stop b  | 3    | 1    |

# Why I am writing about it?
Recently I discovered this issue while I was working with sliding_window_counts plugin. I wanted to detects a lot of files created in folder in small time window. Then I was trying to limit date with threshold but in result I had several events above threshold realted with same folder but slightly different files count. It looked like Gauss Distribution (it was expected) but as I decribed above scan mached all those events in Step 1 but Step 2 saw only last event which not was with largest file count in folder. Copilot devoploed below code snippett but finally I used join instead of scan.
```kql
| scan with_match_id=FID declare (
    Step: int,
    BestCapturedScriptFiles: dynamic,
    BestCapturedCount: long = 0,
    BestS1Timestamp: datetime
) with
(
    step s1 output=last: // Returns last event which matches condition but due to logic, this event contains largest CapturedScriptFiles array length
        isnotempty(CapturedScriptFiles)
    =>  Step = 1,
        // All belows iff-s statements are needed to find in step1 row with most files in CapturedScriptFiles array (to fix sliding window "issue")
        BestCapturedCount = iff(
            array_length(CapturedScriptFiles) > s1.BestCapturedCount,
            array_length(CapturedScriptFiles),
            s1.BestCapturedCount
        ),
        BestCapturedScriptFiles = iff(
            array_length(CapturedScriptFiles) > s1.BestCapturedCount,
            CapturedScriptFiles,
            s1.BestCapturedScriptFiles
        ),
        BestS1Timestamp = iff(
            array_length(CapturedScriptFiles) > s1.BestCapturedCount,
            Timestamp,
            s1.BestS1Timestamp
        ),
        CapturedScriptFiles = iff(
            array_length(CapturedScriptFiles) > s1.BestCapturedCount,
            CapturedScriptFiles,
            s1.BestCapturedScriptFiles
        );
```
