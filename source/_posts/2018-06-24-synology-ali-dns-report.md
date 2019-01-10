---
title: synology-ali-dns-report
date: 2018-06-24 12:01:21
tags: 
- synology
- dns
---
# Synology使用aliyun DDNS
常见python和php两个版本；个人比较喜欢php版本，不需要ssh登录盒子，系统升级的时候也不需要做额外的配置；
## php版本
### 安装php运行环境
需要安装以下应用：
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2018-06-24-synology-ali-dns-report/synology-aliyunddns-env-php.png)
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2018-06-24-synology-ali-dns-report/synology-aliyunddns-env-apache.png)

### 配置 Web Station
> 需要使用 PHP 7.0

![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2018-06-24-synology-ali-dns-report/synology-aliyunddns-env-web-station.png)

### web目录上传PHP脚本
创建`aliyunddns`文件夹

#### ddns.php
```php
<?php  
#########################################################
#                  阿里云DDNS                            #
#     阿里云DNS解析API封装PHP版(支持群晖DDNS接口)           #
#            作者 XOEO QQ 6308532                       #
#########################################################
  include "function.php";
  $info = explode("&", $_SERVER["QUERY_STRING"]);
  if ($info == '')
  {
    exit("badauth");
  }
  $AccessKeyId = $info[0];
  $Secret = $info[1];
  $DomainName = $info[2];
  $UpdateIP = $info[3];
  $HostName = '';
  $arr = explode(".",$DomainName);
  $count = count($arr);
  
  switch($count)
  {
    case 2:
      $HostName = 'www';
      break;
    case 1:
      exit("badsys");
      break;
    case 3:
      $HostName = $arr[0];
      $DomainName = $arr[1].'.'.$arr[2];
      break;    
  }

  $Data = AliYunDDNS::SendAPI(AliYunDDNS::DescribeDomainRecords($AccessKeyId, $DomainName), $Secret);

  $xml = simplexml_load_string($Data);  
  $PageNumber = $xml->PageNumber;
  $PageSize = $xml->PageSize;
  $TotalCount = $xml->TotalCount;
  $DomainRecords = $xml->DomainRecords;
  
  if (($PageNumber == '') and ($DomainRecords == ''))
  {
    exit("badauth");
  }
  $boExist = false;
  foreach ($DomainRecords->children() as $key => $value) 
  {   
    $RR = $value->RR;
    $Type = $value->Type;
    $RecordId = $value->RecordId;
    $RecordValue = $value->Value;
    if ((strcasecmp($Type, 'A') == 0) and (strcasecmp($RR, $HostName) == 0))
    {
      $boExist = true;
      if (strcasecmp($RecordValue, $UpdateIP) <> 0)
      {
        $Data = AliYunDDNS::SendAPI(AliYunDDNS::UpdateDomainRecord($AccessKeyId, $UpdateIP, $RR, $RecordId), $Secret);
        $xml = simplexml_load_string($Data);  
        $retRecordId = $xml->RecordId;
        if (strcasecmp($retRecordId,$RecordId) == 0)
        {
          echo "good";
        }
        else
        {
          echo "nochg ".$UpdateIP;  
        }
      }
      else
      {
        echo "nochg ".$UpdateIP;    
      }
    }
  }  
  if (!$boExist)
  {
    echo "nohost";
  }
?>
```

#### function.php
```php
<?php
#########################################################
#                 阿里云DDNS                             #
#     阿里云DNS解析API封装PHP版(支持群晖DDNS接口)           #
#          作者:XOEO QQ 6308532                         #
#########################################################
  class AliYunDDNS {
    function GetUTCTime()
    {
      date_default_timezone_set('UTC');
      return date('Y-m-d\TH:i:s\Z', time());   
    }  
    function Create_Guid()
    {
      if (function_exists ( 'com_create_guid' ))
      {
        return com_create_guid ();
      }
      else 
      {
        $charid = strtoupper ( md5 ( uniqid ( rand (), true ) ) );
        $hyphen = chr ( 45 );
        $uuid = '' .
        substr ( $charid, 0, 8 ) . $hyphen . substr ( $charid, 8, 4 ) . $hyphen . substr ( $charid, 12, 4 ) . $hyphen . substr ( $charid, 16, 4 ) . $hyphen . substr ( $charid, 20, 12 );
        return $uuid;
      }
    }
    
    function GetSignature($str, $key)
    {
      $signature = "";
      if (function_exists('hash_hmac'))
      {
        $signature = base64_encode(hash_hmac("sha1", $str, $key, true));
      }
      return $signature;
    }
    
    function ProcessEncodeStr($str)
    {
      $tmp = str_replace('+','%20',$str);
      $tmp = str_replace('*','%2A',$tmp);
      $tmp = str_replace('@','%40',$tmp);
      return str_replace('%7E','~',$tmp);  
    }
    
    function percentEncode($str)
    {
  
      return Self::ProcessEncodeStr(urlencode(mb_convert_encoding($str, "UTF-8"))); 
    }    
    
    function SendAPI($str, $Secret)
    {
      $Signature = 'GET&'.Self::percentEncode('/').'&'.Self::percentEncode($str);
      $Signature = Self::getSignature($Signature, $Secret.'&');
      $GetString = "http://dns.aliyuncs.com/?";
      $GetString = $GetString.$str.'&Signature='.Self::percentEncode($Signature);
      return GetURL($GetString);      
    }
    
    function DescribeDomainRecords($AccessKeyId, $DomainName)
    {
      $str = 'AccessKeyId='.$AccessKeyId.'&';  
      $str = $str.'Action=DescribeDomainRecords&';  
      $str = $str.'DomainName='.$DomainName.'&';
      $str = $str.'Format=xml&';
      $str = $str.'SignatureMethod=HMAC-SHA1&';
      $str = $str.'SignatureNonce='.strtolower(Self::Create_Guid()).'&';  
      $str = $str.'SignatureVersion=1.0&';
      $str = $str.'Timestamp='.Self::percentEncode(Self::GetUTCTime()).'&';
      $str = $str.'Version=2015-01-09';
      return $str;      
    }
    
    function UpdateDomainRecord($AccessKeyId, $Value, $HostName, $RecordId)
    {
      $str = 'AccessKeyId='.$AccessKeyId.'&';  
      $str = $str.'Action=UpdateDomainRecord&';  
      $str = $str.'Format=xml&';
      $str = $str.'Line=default&';
      $str = $str.'Priority=1&';
      $str = $str.'RR='.Self::percentEncode($HostName).'&';
      $str = $str.'RecordId='.$RecordId.'&';
      $str = $str.'SignatureMethod=HMAC-SHA1&';
      $str = $str.'SignatureNonce='.strtolower(Self::Create_Guid()).'&';  
      $str = $str.'SignatureVersion=1.0&';
      $str = $str.'TTL=600&';
      $str = $str.'Timestamp='.Self::percentEncode(Self::GetUTCTime()).'&';
      $str = $str.'Type=A&';
      $str = $str.'Value='.Self::percentEncode($Value).'&';
      $str = $str.'Version=2015-01-09';
      return $str;
    }
  }
  
  function GetURL($url)
  {
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    $ret = curl_exec($ch);
    curl_close($ch);
    return $ret;
  }  
  
  function getClientIp(){  
    $socket = socket_create(AF_INET, SOCK_STREAM, 6);  
    $ret = socket_connect($socket,'ns1.dnspod.net',6666);  
    $buf = socket_read($socket, 16);  
    socket_close($socket);  
    return $buf;      
  }   
?>
```
#### update.php
```php
<?php
#########################################################
#                阿里云DDNS                              #
#     阿里云DNS解析API封装PHP版(支持群晖DDNS接口)           #
#          作者:XOEO QQ 6308532                         #
#########################################################
  include "function.php";
  $AccessKeyId = "${yourAcessKeyId}";
  $Secret = "${yourSecret}";
  $info = explode("&", $_SERVER["QUERY_STRING"]);
  $HostName = explode("=",$info[0])[1];
  $MYIP = explode("=",$info[1])[1];
  echo GetURL("http://".$_SERVER['SERVER_NAME']."/aliyunddns/ddns.php?".$AccessKeyId."&".$Secret."&".$HostName."&".$MYIP /*getClientIp()*/);
?>
```

### 配置DDNS
> 控制面板 -> 外部访问 -> DDNS

服务提供商选择已配置的aliyun；
主机名称配置自己的域名，比如我自己的`demo.anocelot.cn`；
其余项目（用户名/电子邮件，密码/秘钥）可以随意填写；

![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2018-06-24-synology-ali-dns-report/synology-ddns-config.png)

## python版本