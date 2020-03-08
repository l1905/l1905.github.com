---
title: golang,php aes-cfb加密互转
description: golang,php,python aes-cfb加密互转
categories:
 - php
 - golang
 -python
tags:
 - aes
 - 加密
---


# golang重写PHP版AES加密(aes-256-cfb)


## 背景

公司底层服务(PHP版)计划通过golang进行重写， 结束golang提升性能。

目前遇到PHP版本AES加解密，在golang环境下解密不出来的情况。 暂时还没找到解决方案。 先记录。 

### PHP版本 加解密方式

```
<?php
function ssl_encrypt($item, $key, $cipher_method = 'aes-256-cfb', $options = []) {
    $iv = md5($key, true);
    $item = trim($item);
    $item = openssl_encrypt($item, $cipher_method, $key, OPENSSL_RAW_DATA, $iv);
    return base64_encode($item);
}

function ssl_decrypt($cipertext, $key, $cipher_method = 'aes-256-cfb', $options = []) {
    $iv = md5($key, true);
    $string = base64_decode($cipertext);
    $p_t = openssl_decrypt($string, $cipher_method, $key, OPENSSL_RAW_DATA, $iv);
    return $p_t;
}

echo ssl_encrypt("11111121", "111111111111111111111111\x00\x00\x00").PHP_EOL;
echo ssl_decrypt("4oqSP6yIT1w=", "111111111111111111111111\x00\x00\x00").PHP_EOL;

```

### golang版本 加解密方式

```
package main

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/md5"
	b64 "encoding/base64"
	"fmt"
)

// Encrypts text with the passphrase
func Encrypt(text string, passphrase string) (string) {
	key := passphrase
	data := []byte(passphrase)
	iv := md5.Sum(data)
	iva := iv[:]
	key = key + "\x00\x00\x00\x00\x00\x00\x00\x00"
	block, err := aes.NewCipher([]byte(key))
	if err != nil {
		panic(err)
	}
	//pad := __PKCS7Padding([]byte(text), block.BlockSize())
	byteText := []byte(text)
	cfb := cipher.NewCFBEncrypter(block, iva)
	encrypted := make([]byte, len(byteText))
	cfb.XORKeyStream(encrypted, byteText)

	return b64.StdEncoding.EncodeToString([]byte(string(encrypted)))
}

// Decrypts encrypted text with the passphrase
func Decrypt(encrypted string, passphrase string) (string) {
	ct, _ := b64.StdEncoding.DecodeString(encrypted)
	key := passphrase+"\x00\x00\x00\x00\x00\x00\x00\x00"
	data := []byte(passphrase)
	iv := md5.Sum(data)
	iva := iv[:]

	block, err := aes.NewCipher([]byte(key))
	if err != nil {
		panic(err)
	}

	cfb := cipher.NewCFBDecrypter(block, iva)
	dst := make([]byte, len(ct))

	cfb.XORKeyStream(dst, ct)
	return string(dst)
}


func main()  {
	// 加密密钥
	passphrase := "111111111111111111111111"
	fmt.Println(Encrypt("11111121", passphrase))
	fmt.Println(Decrypt("r7Jz3WfqvIY=", passphrase))
}
```

### python版本 加解密方式

```
#!/usr/bin/python3
# -*- coding:utf-8 -*-
#

import base64
from Crypto.Cipher import AES
import hashlib
import binascii

def aes_encrypt(data, encryption_key):
    try:
        iv_key = encryption_key
        iv = hashlib.md5(iv_key).digest()

        encryption_key = encryption_key + b"\x00\x00\x00\x00\x00\x00\x00\x00"
        cipher = AES.new(encryption_key, AES.MODE_CFB, iv, segment_size=128)

        encrypted = cipher.encrypt(str.encode(data))
        return base64.b64encode(encrypted).decode()

    except Exception as e:
        raise Exception(str(e))

def aes_decrypt(data, encryption_key):
    try:
        iv_key = encryption_key
        iv = hashlib.md5(iv_key).digest()

        encryption_key = encryption_key + b"\x00\x00\x00\x00\x00\x00\x00\x00"
        cipher = AES.new(encryption_key, AES.MODE_CFB, iv, segment_size=128)

        encrypted_data = base64.b64decode(data)
        # print(binascii.hexlify(encrypted_data))
        show_data = cipher.decrypt(encrypted_data)
        return show_data

    except Exception as e:
        raise Exception(str(e))


if __name__ == '__main__':

    #       12345678123456781234567812345678
    data = '11111121'
    encryption_key = b"111111111111111111111111"

    data = aes_encrypt(data, encryption_key)
    print("encrypt_data = %s" % data)

    data = "r7Jz3WfqvIY="
    data = aes_decrypt(data, encryption_key)
    print("data = %s" % data)
```


## 分析思路

先分析我们选的加密算法`aes-256-cfb8`

`aes`：顾名思义， 这里即是选择的加密算法

`256`:  iv向量位数。 一般是 128， 192， 256， 即对应的iv为 16字节， 24字节，32字节。通过查看PHP源码确认，如果自定义的iv长度不满足给定的长度， 比如使用256，需要设置32位字节的IV， 而我们只有24位的IV长度， 剩余的8位需要补0

`cfb8`： 具体到aes下加密的模式，最后数字代表 加密算法的复杂度，越大复杂度越高， 有1， 8， 128位，三个选项。php， python都支持这三个选项。 但是golang官网只支持128位。

因此`aes-256-cfb8`,  没办法直接转成golang模式。

cfb加密模式不需要内容字段补齐， 比如待加密的内容有两位， 加密结果也是仅有2位。 这个和其他加密模式不一样， 比如`cbc`模式，就需要补齐内容。



因此目前能想到的解决方案是
1. 调整PHP的加密方式， 改成aes-256-cfb(128位)加密， 适配golang 加密
2. golang选择三方加密， 但是存在安全问题， 因为不是官方的，需要确认其方案是否靠谱 。

我这里更倾向于模式1。 我上面实现的例子， 即通过golang, python, php互解密的方式来呈现这种加密结果


 
 
 ## 参考文章
 
 1. [aes加密相关](https://juejin.im/entry/59eea418f265da4320026b1f)
 2. [aes图解](https://www.cnblogs.com/luop/p/4334160.html)
 3. [golang, python 等其他语言互换解密](https://github.com/mervick/aes-everywhere) 但是PHP版本和golang还是不能互转
 4. https://stackoverflow.com/questions/23897809/different-results-in-go-and-pycrypto-when-using-aes-cfb/23899604
 5. https://stackoverflow.com/questions/46346371/convert-openssl-aes-in-php-to-python-aes
 6. https://ekyu.moe/article/aes-cfb-in-golang-and-nodejs/
 7. https://takeai.silverpigeon.jp/tried-crypt-aes-mode-cfb-with-python-javascript-and-php-data-cross-each-other/
 8. https://stackoverflow.com/questions/23897809/different-results-in-go-and-pycrypto-when-using-aes-cfb


