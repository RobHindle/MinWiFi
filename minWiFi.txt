<# WIFI Minimizer
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
#>
<#
Author :  Robert Hindle
Created:  V1 2021-04-17
Limited Liability Statement: Use at your own discretion and risk.  Your counters and activity levels and 
adapters my be different and act differently.
#>
#Constants and environment checks
#Start-Sleep -Seconds 5 # for manual testing 

$regexcnts = [regex]"\d+"
$SecPerHour = 3600
$Min30   = 2100     # 35 min lead time for every 30 minutes test 
                                 # (FAH does load of next WU just prior to end of current WU)
$WiFiLvl = 1000     # level of Wifi activity that indicates it is in active use

$GNA = Get-NetAdapter | Where-Object -Property Name -like "*Wi*Fi*"
$WiFiAdptrM = $GNA.Name

#Other counters found like...
#$GC = Get-Counter -ListSet * | Where-Object -Property PathsWithInstances -Like -Value $("*802.11*")
$CounterNm = "\Network Interface(Broadcom 802.11n Network Adapter)\Bytes Total/sec" 

$ProfileNm = Find-WiFi

$WiFiLOGM = "$PSScriptRoot\WIFIlog.txt"

<# Get-FAHeta
.Synopsis
   Test the Folding at Home Client to determine the number of seconds ETA to next exchange.
   Thanks for direction how to do this To:
   https://pifoldingguide.wordpress.com/
   https://forums.anandtech.com/threads/folding-home-fahclient-config-control-manual-page.2574018/
.Description
   Prepares an API request to the FAH client to receive the client's 
   Estimated Time of Arriving at a Completion (ETA) in seconds.
   Outputs the number of seconds to expected activity.
.Example
   Get-FAHeta
#>
Function Get-FAHeta {
#Build the command line and Parms
$fahpath ='C:\Program Files (x86)\FAHClient'

$cmd = $fahpath+"\fahclient.exe" 

$inforqst = '"simulation-info 0"'
$FAHcmd = " --send-command $inforqst "
$FAHparms = $Fahcmd.Split(" ")

#Execute the command Line with parameters
$out = & $cmd $FAHparms

#Process key information out of return string
$outs = $out.split(",")
foreach ($info in $outs) {
   if ($info -like "*eta*" ) { $eta = $regexCnts.Match($info).value }
   if ($info -like "*done*") { $don = $regexCnts.match($info).value }
}
$eta
} # Get-FAHeta

<# Get-Upcoming Activity
.Synopsis
   This checks any number of activity sources and determines the nearest number of seconds to activity.
   Currently just the FAH test being performed.
.Description
   Checks potentially a number of activity sources for an event needing wifi and picks the shortest.
   Outputs number of seconds till activity expected.
.Example
   Get-UpcomingActivity
#>
Function Get-UpcomingActivity {
   $reta = 99999
   # Decrease ETA for FAH
   $eta = Get-FAHeta
   if ($eta -lt $reta) {
      $reta = $eta
   }
   # Decrease ETA for Other Considerations
   $reta
} # Get-UpcomingActivity

<# Get-ScreenLocked
.Synopsis
   Determines if the users screen is locked.  Trying not to close Wi-Fi while in use by a person.
   Thanks for ideas how to do this To:
   https://social.technet.microsoft.com/Forums/windowsserver/en-US/a6cfc1ba-c2f4-4a65-b3a3-7d44b8f5cae4/powershell-to-tell-if-a-computer-is-locked
.Description
   Checks that there is a signed-in user and the number of LogonUI processes in place.
   Seems that W10 Apr 2021 has always 1 and 2 if the session is locked but still running.
   Outputs flag of either ScreenLocked or ScreenNotLocked.
.Example
   Get-ScreenLocked
#>
Function Get-ScreenLocked {
$currentuser = get-wmiobject -class win32_computersystem | select-object -expandproperty Username
$processLU = Get-Process -Name logonUI -ErrorAction SilentlyContinue
if ($currentuser -and $processLU) 
    {"ScreenLocked"} 
else 
    {"ScreenNotLocked"}
} # Get-ScreenLocked

<# Get-WifiActivityLevel
.Synopsis
   Checks for other Wi-Fi traffic before doing any disconnect to determine if there is other traffic.
.Description
   Checks for other Wi-Fi traffic by checking adapter counters for activity.
   Output is a Cooked value of the scope of activity that is on the wifi connection.
   Threshold of 1000 units is in the Constants section at top of program.
.Parameter
   CounterNm - Name of PowerShell Counter for you wifi adapter   
.Example
   Get-WifiActivityLevel
#>
Function Get-WiFiActivityLevel {
   Param ($CounterNm = "\Network Interface(Broadcom 802.11n Network Adapter)\Bytes Total/sec" )
   $GCWF = Get-Counter -counter $CounterNm -MaxSamples 1 -SampleInterval 1
   $GCWF.CounterSamples.CookedValue
} # Get-WiFiActivityLevel

<# Get-WifiState
.Synopsis
   Determines if your WiFi is Up or Down/Disconnected by adapter state.
.Description
   Detmines simply up or down.  Enabling worries about difference between Down and Disconnected.
.Parameter
   WiFiAdptr - Adapter being used by your Computer. Test for in Constants
.Example
   Get-WifiState -WifiAdptr "WiFi" 
#>
Function Get-WIFIState {
   Param ( $WiFiAdptr = "Wi-Fi")
   if((Get-NetAdapter -Name $WiFiAdptr).Status -eq "Up") {
      $wificon = & "netsh" wlan show interface
      if ($wificon -like "*connected*") { "UP" }
      else { "DOWN" }  
   }
   else {
     "DOWN"
   }
} # Get-WifiState

<# Connect-WiFi
.Synopsis
   Uses WIFI profile name to trigger through netsh commands to reconnect Wifi
   Reference:
   https://docs.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-contexts
.Description
   Temporary push of command into netshcmd.txt filefor netsh in wlan contect to operate.
   Netsh only accepts UTF8 file encoding for the process
.Parameter
   Wifi -Name of wifi profile to use for connection
.Example
   Connect-Wifi -Wifi MyHomeWifi 
#>
Function Connect-Wifi {
   Param ($WiFi)
   $Cmdpath = "$PSScriptRoot\netshcmd.txt"       
   $PSDefaultParametervalues['Out-File:Encoding']='utf8'
   $Context = "wlan"
   $NScmd = " connect name=$wifi"
   if (Test-Path -Path $Cmdpath) {
      Clear-Content -Path $Cmdpath
      }
   Add-Content -Path $Cmdpath -Value $NScmd
   & "netsh" -c $Context -f $Cmdpath
} # Connect-Wifi

<# Find-WiFi
.Synopsis
   Find a broadcasting WIFI SSID that is specified as one of your core managed accounts
   See Baseline-WiFiProfiles Function in WiFIcoreTools.ps1 for help with this.
       Filename is WiFiCore.txt and lists baselined accepted profiles 
.Description
    Given a name of a profile it returns the SSID and the access Key.
    If none it tells you.  If 1 if names it.  If 2+ then it names the first one found.
.Parameter
    BasePath = File path to your baselined WIFI connections
.Example
   Find-WiFI
#>
Function Find-WiFi {
    Param ( $BasePath = "$PSScriptRoot\WiFiCore.txt")
          $BaseWiFis = Get-Content -Path $BasePath
          $WiFilist = "\s("
          Foreach ($BaseWiFi in $BaseWiFis) {
             $WiFilist += $BaseWiFi
             $WiFilist += "|"
          }
          $WiFilist += ")\s"
          $WiFilist = $WiFilist.replace("|)",")")
          $regex0 = [regex]$Wifilist

          $AVLwifis = & "netsh" wlan show networks

          $Match0 = $regex0.match($AVLwifis) 
          if ($Match0.count -eq 1) {
           $Match0.Value
          }
          else {
            if ($Match0.count -lt 1) { "No Core Found" }
            else { 
               $($Match0[0].Value)
            }
          }
} # Find-WiFi

<# Alter-WIFIState
.Synopsis
   Called to Connect or Disconnect the WiFi of this machine based upon connection state and 
   Adapter state. 
.Description
   Determines Adapter state and does connection steps appropriate to the state Up, Down or Disconnected
.Parameter
    EnableDis - "Enable" to Connect anything else to disconnect
.Parameter
    WifiAdptr - Name of the Wifi adapter of running machine
.Parameter
    ProfileName - WiFi Profile to be used to make connection
.Parameter
    WiFiLog to track actions taken and when.
.Example
   Alter-WIFIState -EnableDis "Disable" -WifiAdptr "Wi-Fi" -ProfileName "MyHomeProfile" -WifiLog $wifiLogPath
#>
Function Alter-WIFIState {
   Param ( $EnableDis = "Enable",
           $WiFiAdptr,
           $ProfileName,
           $WifiLog)
           $TDY = get-date
    if((Get-NetAdapter -Name $WiFiAdptrM).Status -eq "Up") {          #### Adapter Up
       if ($EnableDis -eq "Enable") {
         $wificon = & "netsh" wlan show interface
         if ($wificon -like "*connected*") { } #All is good at two levels
         else {
           Connect-Wifi -WiFi $ProfileName
           "Connected $TDY Enable-UP" >> $WifiLog
         } # Adapter Up and Connection made
       } # Enable requested
       else {
           $wificonstate = & "netsh" wlan disconnect
           "Disconnected $TDY Disable-UP" >> $WifiLog

       } # Disable Requested
    } # Adapter UP
    elseif ((Get-NetAdapter -Name $WiFiAdptrM).Status -eq "Disconnected") { #### Disconnected
         if ($EnableDis -eq "Enable") {
         Connect-Wifi -WiFi $ProfileName 
         "Connected $TDY Enable-DISCONNECTED" >> $WifiLog
       }
       else {} # Adapter is not up so connection also disconnected/disabled
    } #Adapter Disconnected
    else {                                                          #### Adapter Down
       if ($EnableDis -eq "Enable") {
         Enable-NetAdapter -Name $WiFiAdptr -Confirm:$false -ErrorAction SilentlyContinue
         Connect-Wifi -WiFi $ProfileName 
         "Connected $TDY Enable-DOWN" >> $WifiLog
       }
       else {} # Adapter is not up so connection also disconnected/disabled
    } #Adapter Down

} # Alter-WIFIState

####### Main ######
# Some expected activity within next 35 minutes (30 minutes + 5)
$GUA = Get-UpcomingActivity
if ([int]$GUA -le [int]$Min30) { # Within 30 of FAH eta
      #"inside Time"
      $GWFS = Get-WIFIState -WiFiAdptr $WifiAdptrM
      if($GWFS -eq "UP") {
        #"Do nothing - Enabled and exchange coming up"
       }
       else { #wifi not enabled so enable it
       #"Enable"
       Alter-WIFIState -EnableDis "Enable" -WiFiAdptr $WiFiAdptrM -ProfileName $ProfileNm -WifiLog $WiFiLOGM
       } #wifi not enabled so enable it
   }
   else { # Long time to next exchange
       $GWFS = Get-WIFIState -WiFiAdptr $WifiAdptrM
       if($GWFS -eq "DOWN") {
        #Do nothing - Disabled and no exchange immenent
       }
       else { # wifistate is Up so Think of disabling it
          $ScrnState = Get-ScreenLocked
          $WiFiActvty = Get-WiFiActivityLevel
          # check if currently in use or if something else is still using wifi traffic.
          If (($ScrnState -eq "ScreenLocked") -and ($WiFiActvty -lt $WiFiLvl)) # Checks on activity says OK to disable
            {          
            Alter-WIFIState -EnableDis "Disable" -WiFiAdptr $WiFiAdptrM -ProfileName $ProfileNm -WifiLog $WiFiLOGM
            }
          else { # Screen not locked Thr4 in use
            #"Do Nothing-Activity Detected"
            }
         }# wifi is enabled so Think of disabling it   
   } # Greater than 30 minutes