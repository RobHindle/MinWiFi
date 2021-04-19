WIFI Minimizer
.Synopsis
   Minimize your PC exposure to WiFi interfaces while still allowing processes to run and, as they require
   wifi access, trigger WiFi connects for a period long enough to do business and close once more.
.Description
    Set for a 30 minute granularity of a Scheduled Task running this function.
    Currently triggers on the Folding at Home ETA determinant.
    Calls upon netsh tools to detect and manipulate a PC's WiFi connection.
    Wifi connections must be manual rather than automatic.
    Will automatically connect to a WiFi in your baselined list that is currently transmitting.
    Log of disconnects and Connects provide in that same location.
.Parameter
    BasePath = File path to your baselined WIFI connections
.Example
   In Scheduled Task running every 30 minutes...
   Execute Program    %systemroot%\System32\windowsPowerShell\v1.0\powershell.exe
   Attributes         -executionpolicy Bypass -Windowstyle Hidden -file "<install location>\minWiFi.ps1"

Hash Check Values for this Tool and its files available at http://web.ncf.ca/bv178/HashChecks.html
