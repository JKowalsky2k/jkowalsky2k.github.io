## Custom function to calculate Shannon entropy of string value

```kusto
DeviceFileEvents
| extend FileNameOnly = tostring(split(FileName, '.')[0])
| serialize RowId = row_number()
| extend Length = strlen(FileNameOnly)
| extend idxs = range(0, Length - 1, 1)
| mv-expand idx = idxs to typeof(int)
| extend Char = substring(FileNameOnly, idx, 1)
| summarize CharCount = count() by RowId, FileNameOnly, Char, Length
| extend p = todouble(CharCount) / todouble(Length)
| extend EntropyComponent = -1 * p * log2(p)
| summarize Entropy = sum(EntropyComponent) by Rowid, FileNameOnly
| where Entropy > 5.0
```
