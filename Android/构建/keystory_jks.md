### key story


通过`cmd` 或 `Terminal` 查看jks信息:
- 获取 debug.keystore 的信息：
`keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android`
- 查看其他jks信息:
`keytool -list -v -keystore 文件路径 -alias 别名 -storepass 密码 -keypass 密码`

```
Last login: Fri Jul 16 17:45:17 on ttys001
shaoshuai@shaoshuaideMacBook-Pro ~ % keytool -list -v -keystore /Users/work/ishare.jks -alias ishare -storepass mapleshao -keypass mapleshao
别名: ishare
创建日期: 2021-7-22
条目类型: PrivateKeyEntry
证书链长度: 1
证书[1]:
所有者: CN=mapleshao, OU=maple, O=maple, L=beijing, ST=bejing, C=CN
发布者: CN=mapleshao, OU=maple, O=maple, L=beijing, ST=bejing, C=CN
序列号: 3304ebe2
有效期为 Thu Jul 22 19:35:22 CST 2021 至 Mon Jul 16 19:35:22 CST 2046
证书指纹:
	 MD5:  3C:E4:91:B3:E2:10:DD:61:37:02:81:09:4E:00:D2:71:68:F9:7E:A8
	 SHA1: 99:50:F3:6F:A7:78:1D:E0:B5:C7:A1:80:41:A6:AC:CC:B8:D8:79:33:61:0B:23:4C:B8:AA:52:11:85:17:73:91
	 SHA256: SHA256withRSA
签名算法名称: 2048 位 RSA 密钥
主体公共密钥算法: 3
版本: {10}

shaoshuai@shaoshuaideMacBook-Pro ~ %
```
md5: 4a17dfb8aeb07c76c3d70a70efb0d4b4
md5: 3543c5f9eb5570120c1a6600bfd01c0a

```
(Signer: Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 855960546 (0x3304ebe2)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=bejing, L=beijing, O=maple, OU=maple, CN=mapleshao
        Validity
            Not Before: Jul 22 11:35:22 2021 GMT
            Not After : Jul 16 11:35:22 2046 GMT
        Subject: C=CN, ST=bejing, L=beijing, O=maple, OU=maple, CN=mapleshao
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:93:dc:0c:9f:57:1f:96:ba:f7:bb:67:a0:6d:52:
                    a4:9c:19:59:ce:ec:00:a9:16:cf:85:44:13:a6:5c:
                    9d:d7:64:c6:5d:f0:14:b0:56:db:e7:e0:ad:35:d3:
                    f5:ba:f2:90:a1:0a:8f:49:c3:8b:e6:13:f8:77:ae:
                    79:5a:3f:8d:1f:93:c2:4e:9b:31:ff:3a:88:18:f5:
                    c2:b8:13:0b:57:c8:9f:96:28:05:b9:dd:e8:e3:c0:
                    74:41:9d:06:8e:88:81:32:ba:16:9a:c0:dd:dc:a9:
                    ae:77:4b:9b:60:c1:34:29:e1:0b:cc:69:68:c1:d3:
                    bc:bd:7b:5b:f8:0a:69:59:db:1b:22:cb:fc:40:e5:
                    c7:48:6c:77:16:eb:3c:3f:3c:8d:b4:d0:b6:ee:75:
                    da:d0:e3:08:1e:92:9a:c0:ee:7f:93:01:4f:5a:ca:
                    25:ad:e4:7c:18:06:39:6b:0f:4b:0d:dc:b0:1a:25:
                    02:cb:34:6c:d9:2d:84:b9:06:84:03:3a:4f:ac:ff:
                    33:03:72:96:29:bd:18:e5:3d:d8:c4:b9:c6:88:f8:
                    3d:47:49:a3:1e:1c:82:9e:6a:f1:c8:a4:86:3f:d6:
                    1f:e1:6b:2d:ac:07:3f:9f:be:7c:b2:b9:39:25:28:
                    6c:89:69:c7:fe:2d:9e:33:c9:b8:79:e7:0b:dd:3e:
                    fa:ed
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                3E:CE:02:8F:B3:C8:D2:A1:55:89:57:A6:4B:44:DD:F8:CC:8C:ED:86
    Signature Algorithm: sha256WithRSAEncryption
         2b:55:c4:f5:b5:99:40:66:16:77:99:06:ac:ac:68:e0:dc:c0:
         54:a3:1c:cc:80:d2:a2:b1:8b:90:3a:e2:99:8b:59:95:93:09:
         e8:fb:d0:d7:a4:fd:00:94:30:8e:f1:27:d5:ea:30:7e:7f:a7:
         db:48:30:a0:c7:0b:0d:85:12:26:4d:b4:bf:d9:2b:0f:8d:57:
         75:c4:d1:e4:16:2a:bc:1b:70:83:b1:89:ab:c0:9f:0c:3d:d4:
         ec:d0:e8:a6:98:8b:47:c0:e5:bf:51:70:22:f8:96:f4:43:0c:
         d8:60:0d:8f:b3:28:33:a7:6f:9e:96:2c:77:17:dd:46:a0:cb:
         3c:10:79:6e:86:cf:2f:25:09:57:35:10:bf:6e:dd:9e:52:b5:
         d4:2c:5d:b3:fe:31:8b:25:0c:a8:2f:81:5a:b8:2c:bf:b5:62:
         14:fa:e4:6c:f1:03:33:8c:b3:bc:42:dc:9f:43:38:cc:21:20:
         c1:bf:cc:e0:09:3b:f8:f2:77:ce:e7:17:d6:8a:42:46:70:71:
         15:6c:b0:d2:94:4c:da:ac:23:3f:75:15:0c:31:52:e0:49:64:
         90:d7:58:f7:fa:07:7b:c7:1b:f2:e9:fe:43:c7:79:8d:ca:7a:
         7d:62:bf:d2:35:98:1f:81:70:99:05:ae:7b:7d:09:e4:67:ae:
         af:15:42:67
)
```
