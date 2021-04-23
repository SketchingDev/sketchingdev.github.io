---
layout: post
title:  "PowerShell: Extract the summary table from RoboCopy's log file"
date:   2015-10-25 00:00:00
categories: powershell robocopy
---

When automating the copying of files with [RoboCopy](https://en.wikipedia.org/wiki/Robocopy) I wanted to be able to interrogate the summary table at the bottom of its log file - shown below.

```
            Total    Copied   Skipped  Mismatch    FAILED    Extras
 Dirs :         1         0         1         0         0         0
Files :         2         2         0         0         0         0
Bytes :  328.45 m  328.45 m         0         0         0         0
```

To allow me to analyse the summary table I wrote a [PowerShell Cmdlet](https://technet.microsoft.com/en-us/library/ms714395%28v=vs.85%29.aspx) that extracts it out into PowerShell objects, which I can then feed into other processes/scripts for analysis.

This means tasks like ordering the summary table or finding any failures are now really simple:

```
> Get-Content "C:\robocopy.log" -Raw | Select-RoboSummary | Sort-Object Type | Format-Table

Type       Total       Copied       Skipped       Mismatch       Failed       Extras                          
----       -----       ------       -------       --------       ------       ------                          
Bytes      328.45 m    328.45 m     0             0              0            0                               
Dirs       1           0            1             0              0            0                               
Files      2           2            0             0              1            0                     
```

```powershell
Get-Content "C:\robocopy.log" -Raw | Select-RoboSummary | Where{ $_.Failed -gt 0 }
```

```powershell
function Select-RoboSummary {
  [CmdletBinding()]
  param (
    [parameter(Mandatory=$true,ValueFromPipeline=$true)]
    [string]$log,
    [parameter(Mandatory=$false,ValueFromPipeline=$false)]
    [switch]$separateUnits
  )
  PROCESS
  {
    $cellHeaders = @("Total", "Copied", "Skipped", "Mismatch", "Failed", "Extras")
    $rowTypes  = @("Dirs", "Files", "Bytes")

    # Extract rows
    $rows = $log | Select-String -Pattern "(Dirs|Files|Bytes)\s*:(\s*([0-9]+(\.[0-9]+)?( [a-zA-Z]+)?)+)+" -AllMatches
    if ($rows.Count -eq 0)
    {
      throw "Summary table not found"
    }

    if ($rows.Matches.Count -ne $rowTypes.Count)
    {
      throw "Unexpected number of rows/ Expected {0}, found {1}" -f $rowTypes.Count, $rowsMatch.Count
    }

    # Merge each row with its corresponding row type, with property names of the cell headers
    for($x = 0; $x -lt $rows.Matches.Count; $x++)
    {
      $rowType  = $rowTypes[$x]
      $rowCells = $rows.Matches[$x].Groups[2].Captures | foreach{ $_.ToString().Trim() }

      if ($cellHeaders.Length -ne $rowCells.Count)
      {
        throw "Unexpected amount of cells in a row. Expected {0} cells (the amount of headers) but found {1}" -f $cellHeaders.Length,$rowCells.Count
      }

      $row = New-Object -TypeName PSObject
      $row | Add-Member -Type NoteProperty Type($rowType)

      for($i = 0; $i -lt $rowCells.Count; $i++)
      {
        $header = $cellHeaders[$i]
        $cell   = $rowCells[$i]

        if ($separateUnits -and ($cell -match " "))
        {
          $cell = $cell -split " "
        }

        $row | Add-Member -Type NoteProperty -Name $header -Value $cell
      }

      $row
    }
  }
}
```
