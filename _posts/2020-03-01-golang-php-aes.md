---
title: golang,php aes加密互转
description: golang,php aes加密互转
categories:
 - php
 - golang
tags:
 - aes
 - 加密
---


# golang重写PHP版AES加密(aes-cfb-256)


## 背景

公司底层服务(PHP版)计划通过golang进行重写， 结束golang提升性能。

目前遇到PHP版本AES加解密，在golang环境下解密不出来的情况。 暂时还没找到解决方案。 先记录。 

### PHP版本 加解密方式

```
<?php
function ssl_encrypt($item, $key, $cipher_method = 'AES-256-CFB8', $options = []) {
    if (empty($key)) {
        return false;
    }
    $iv = md5($key, true);
    $item = trim($item);
    $item = openssl_encrypt($item, $cipher_method, $key, OPENSSL_RAW_DATA, $iv);
    //echo bin2hex($item).PHP_EOL;
    return base64_encode($item);
}

//echo openssl_cipher_iv_length("AES-192-CFB8").PHP_EOL;
echo ssl_encrypt("18765901111", "111111111111111111111111).PHP_EOL;
```

### golang版本 加解密方式

```
package main

import (
	"bytes"
	"crypto/aes"
	"crypto/cipher"
	"crypto/md5"
	b64 "encoding/base64"
	"fmt"
)

// Encrypts text with the passphrase
func Encrypt(text string, passphrase string) (string) {

	// 密钥md5 iv
	key := passphrase
	data := []byte(passphrase)
	iv := md5.Sum(data)
	iva := iv[:]
	block, err := aes.NewCipher([]byte(key))
	if err != nil {
		panic(err)
	}
	pad := __PKCS7Padding([]byte(text), block.BlockSize())
	cfb := cipher.NewCFBDecrypter(block, iva)
	encrypted := make([]byte, len(pad))
	cfb.XORKeyStream(encrypted, pad)

	return b64.StdEncoding.EncodeToString([]byte(string(encrypted)))
}

// Decrypts encrypted text with the passphrase
func Decrypt(encrypted string, passphrase string) (string) {
	ct, _ := b64.StdEncoding.DecodeString(encrypted)

	key := passphrase
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
	return string(__PKCS7Trimming(dst))
}

func __PKCS7Padding(ciphertext []byte, blockSize int) []byte {
	padding := blockSize - len(ciphertext)%blockSize
	padtext := bytes.Repeat([]byte{byte(padding)}, padding)
	return append(ciphertext, padtext...)
}

func __PKCS7Trimming(encrypt []byte) []byte {
	padding := encrypt[len(encrypt)-1]
	return encrypt[:len(encrypt)-int(padding)]
}

func __DeriveKeyAndIv(passphrase string, salt string) (string, string) {
	salted := ""
	dI := ""

	for len(salted) < 48 {
		md := md5.New()
		md.Write([]byte(dI + passphrase + salt))
		dM := md.Sum(nil)
		dI = string(dM[:16])
		salted = salted + dI
	}

	key := salted[0:32]
	iv := salted[32:48]

	return key, iv
}

func main()  {

	passphrase := "111111111111111111111111"
	fmt.Println(Encrypt("hello world", passphrase))
	fmt.Println(Decrypt("xmdH+4Iv8BE4Gl0kdZNbaA==", passphrase))
}
```


## 分析思路

这里主要涉及`aes-cfb-256`加密， golang版有补齐块长度，即pcsk7补齐,比如差4位，则补齐4个4。


 PHP版本号称也是pcsk7补齐， 但实际加密效果未看到补齐， 这是目前最大的疑点， 排查远吗，PHP是字节调用的openssl加密库， 那就更诡异。
 
 
 
 ## 参考文章
 
 1. [aes加密相关](https://juejin.im/entry/59eea418f265da4320026b1f)
 2. [aes图解](https://www.cnblogs.com/luop/p/4334160.html)
 3. [golang, python 等其他语言互换解密](https://github.com/mervick/aes-everywhere) 但是PHP版本和golang还是不能互转


