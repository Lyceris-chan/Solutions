## Instructions on how to fix Windows Terminal (Preview) 0xc0000022

Check if there is a registry key over at:
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\wt.exe
HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\wt.exe

There should be a (Default) and Path key (string) containing the dir of the WT.exe which can be found in:
C:\Program Files\WindowsApps\Microsoft.WindowsTerminal_[version]
C:\Program Files\WindowsApps\Microsoft.WindowsTerminalPreview_[version]

Example: C:\Program Files\WindowsApps\Microsoft.WindowsTerminalPreview_1.11.2421.0_x64__8wekyb3d8bbwe

Create a new Registry key if the wt.exe folder is absent from the registry as mentioned above
Create a new Registry string if the (Default) and Path strings are absent as mentioned above

A restart should not be nessecary and the issue should be fixed.
