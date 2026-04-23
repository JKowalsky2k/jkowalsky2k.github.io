## Calculation Shannon entropy of string value

```kusto
DeviceFileEvents
| extend FileNameOnly = tostring(split(FileName, '.')[0])                  // Extract string value from file name by removing extension
| serialize RowId = row_number()                                           // Assigning row number to each row
| extend Length = strlen(FileNameOnly)                                     // Calculating string length
| mv-expand idx = range(0, Length - 1, 1) to typeof(int)                   // Generating indexes for extracting chars from string
| extend Char = substring(FileNameOnly, idx, 1)                            // Extracting single chars from string 
| summarize CharCount = count() by RowId, FileNameOnly, Char, Length       // Counting char occurances in string
| extend p = todouble(CharCount) / todouble(Length)                        // Calulating probability for char count 
| extend EntropyComponent = -1 * p * log2(p)                               // Calculating Entropy component for char count
| summarize Entropy = sum(EntropyComponent) by Rowid, FileNameOnly         // Calculating Entropy
```
