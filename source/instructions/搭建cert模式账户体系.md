# 搭建Cert模式账户体系
作者：长安链团队 韩学洋

## 使用CA搭建长安链网络
长安链支持PermissionWithCert、PermissionWithKey、Public等三种不同账户模式的链，本章节我们将详细介绍如何通过CA服务搭建完整的长安链PermissionWithCert模式的账户体系，以及如果管理该模式下的链账户，包括组织账户、节点账户、用户账户的增删等。

本文示例说明：
- 创建新链：4组织，1共识节点/每组织，1管理员/每组织
- 链管理：组织管理、节点管理、用户管理

### 启动CA服务
长安链Ca服务可用于签发证书，部署PermissionWithCert模式的长安链前，我们需要通过长安链CA签发出相关的证书文件。
长安链CA服务支持多种方式部署，环境准备请参考[CA证书服务-安装部署](../dev/CA证书服务.html#安装部署)，本文我们使用docker快速部署CA服务。

**CA服务的配置文件config.yml（4组织示例)**

  ```yaml
  # log config
  log_config:
    level: info                                   # The log level                               
    filename: ../log/ca.log                       # The path to the log file            
    max_size: 1                                   # The maximum size of the log file before cutting (MB)
    max_age: 30                                   # The maximum number of days to retain old log files
    max_backups: 5                                # Maximum number of old log files to keep
  
  # db config
  db_config:
    user: root
    password: 123456
    ip: 127.0.0.1
    port: 13306
    dbname: chainmaker_ca
  
  # Base config
  base_config:
    server_port: 8090                                  # Server port configuration
    ca_type: single_root                               # Ca server type : double_root/single_root/tls/sign
  #  expire_year: 2                                    # The expiration time of the certificate (year)
    expire_month: 6                                    # The expiration time of the certificate (month)(high level)
  #  cert_valid_time : 2m                              # cert valid time (for testing use only)
    hash_type: SHA256                                  # SHA256/SHA3_256/SM3
    key_type: ECC_NISTP256                             # ECC_NISTP256/SM2
    can_issue_ca: false                                # Whether can continue to issue CA cert
  #  provide_service_for: [wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org,wx-org4.chainmaker.org]      
                                                       # A list of organizations that provide services
    key_encrypt: false                                  # Whether the key is stored in encryption
    access_control: false                              # Whether to enable permission control
  #  default_domain: chainmaker.org                    # the default value for sans in the certificate
  
  pkcs11_config:
    enabled: false
    library: /usr/local/lib64/pkcs11/libupkcs11.so
    label: HSM
    password: 11111111
    session_cache_size: 10
    hash: "SHA256"
  
  # Root CA config
  root_config:
    cert:
      - cert_type: sign                                                  # Certificate path type : tls/sign (if ca_type is 'single_root',should be sign)
        cert_path: ../crypto-config/rootCA/root.crt                      # Certificate file path
        private_key_path: ../crypto-config/rootCA/root.key               # private key file path    
        key_id: SM2SignKey261                                            # pkcs11 key id
    csr:
      CN: root                
      O: org-root                         
      OU: root                         
      country: CN                      
      locality: Beijing                
      province: Beijing             
  
  # intermediate config
  intermediate_config: 
    - csr:
        CN: ca-wx-org1.chainmaker.org                        
        O: wx-org1.chainmaker.org                        
        OU: ca                         
        country: CN                       
        locality: Beijing                
        province: Beijing            
      key_id: SM2SignKey6
  
    - csr:
        CN: ca-wx-org2.chainmaker.org                       
        O: wx-org2.chainmaker.org                     
        OU: ca                         
        country: CN                       
        locality: Beijing                
        province: Beijing            
      key_id: SM2SignKey249
      
    - csr:
        CN: ca-wx-org3.chainmaker.org                       
        O: wx-org3.chainmaker.org                    
        OU: ca                         
        country: CN                       
        locality: Beijing                
        province: Beijing            
      key_id: SM2SignKey257
  
    - csr:
        CN: ca-wx-org4.chainmaker.org                    
        O: wx-org4.chainmaker.org                    
        OU: ca                         
        country: CN                       
        locality: Beijing                
        province: Beijing            
      key_id: SM2SignKey260
  
  # access control config
  access_control_config:
    - app_role: admin
      app_id: admin1
      app_key: passw0rd
    - app_role: user
      app_id: user1
      app_key: passw0rd
  ```

**docker方式启动服务**  

将以上config.yml配置文件拷贝到`chainmaker-ca/src/conf/`目录后，在项目主目录下运行`./deploy.sh`脚本。 

- 启动成功后，终端显示
```shell
2022-08-24T03:00:12.529492Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.30'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
start ca services...
9a4fa1c7f3dcb37fdfe6ae2a434ebc91a4d21ec94364c929e0ed1199eef4903d
[env]:
env is empty, set default
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /test                     --> main.main.func1 (3 handlers)
[GIN-debug] POST   /api/ca/login             --> chainmaker.org/chainmaker-ca/src/handlers.Login.func1 (3 handlers)
[GIN-debug] POST   /api/ca/gencertbycsr      --> chainmaker.org/chainmaker-ca/src/handlers.GenCertByCsr.func1 (3 handlers)
[GIN-debug] POST   /api/ca/gencert           --> chainmaker.org/chainmaker-ca/src/handlers.GenCert.func1 (3 handlers)
[GIN-debug] POST   /api/ca/querycerts        --> chainmaker.org/chainmaker-ca/src/handlers.QueryCerts.func1 (3 handlers)
[GIN-debug] POST   /api/ca/renewcert         --> chainmaker.org/chainmaker-ca/src/handlers.RenewCert.func1 (3 handlers)
[GIN-debug] POST   /api/ca/revokecert        --> chainmaker.org/chainmaker-ca/src/handlers.RevokeCert.func1 (3 handlers)
[GIN-debug] POST   /api/ca/gencrl            --> chainmaker.org/chainmaker-ca/src/handlers.GenCrl.func1 (3 handlers)
[GIN-debug] POST   /api/ca/gencsr            --> chainmaker.org/chainmaker-ca/src/handlers.GenCsr.func1 (3 handlers)
[GIN-debug] POST   /api/ca/getnodeid         --> chainmaker.org/chainmaker-ca/src/handlers.GetCertNodeId.func1 (3 handlers)
[GIN-debug] Listening and serving HTTP on :8090
chainmaker-ca server start!
```
- 通过curl方式调用测试接口，显示成功
```shell
curl --location --request GET 'http://localhost:8096/test'
{"data":"Hello,World!","msg":"The request service returned successfully"}%
```

### 基于CA服务生成证书账户
#### 生成组织证书  

由于在CA配置文件中，我们配置了`intermediate_config`中间CA（每个中间CA代表一个组织），所以CA服务在启动时，会生成相应的组织CA证书。 以org1为例：

- 获取组织org1的CA证书。  
```shell
➜  chainmaker-ca-backend git:(develop) ✗ curl --location --request POST 'http://localhost:8096/api/ca/querycerts' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org1.chainmaker.org",
    "userType": "ca",
    "certUsage": "sign"
}' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1299  100  1211  100    88  71235   5176 --:--:-- --:--:-- --:--:-- 76411
{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": [
    {
      "userId": "ca-wx-org1.chainmaker.org",
      "orgId": "wx-org1.chainmaker.org",
      "userType": "ca",
      "certUsage": "sign",
      "certSn": 1696861425238169000,
      "issuerSn": 3682789430329868000,
      "certContent": "-----BEGIN CERTIFICATE-----\nMIICaDCCAg6gAwIBAgIIF4x2edRiVb8wCgYIKoZIzj0EAwIwYjELMAkGA1UEBhMC\nQ04xEDAOBgNVBAgTB0JlaWppbmcxEDAOBgNVBAcTB0JlaWppbmcxETAPBgNVBAoT\nCG9yZy1yb290MQ0wCwYDVQQLEwRyb290MQ0wCwYDVQQDEwRyb290MB4XDTIyMDgy\nNDAyNTAyMVoXDTIzMDIyMDAyNTAyMVowgYMxCzAJBgNVBAYTAkNOMRAwDgYDVQQI\nEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNo\nYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZY2Etd3gtb3JnMS5j\naGFpbm1ha2VyLm9yZzBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABJm8UUrxkzvL\nzB6u++UjdTvFhy9Fay1h+X7xVijPp8LeyAX70P5O4VGvik7j/yCHoJ0GE89pxH4B\nUIq6WnVTtDWjgYswgYgwDgYDVR0PAQH/BAQDAgEGMA8GA1UdEwEB/wQFMAMBAf8w\nKQYDVR0OBCIEIFQUtaMGHEYWLDGcfWBR0SQZjuWdI31bW7tCAyrplWpuMCsGA1Ud\nIwQkMCKAID2QQznj2uOhDBY0zQDZs0PsixvqNqTx5WLTT7JErCJMMA0GA1UdEQQG\nMASCAIIAMAoGCCqGSM49BAMCA0gAMEUCIFHIdvzax2/g183R6RU1S0jqRZNiKvrL\nLPzccecq0N/1AiEApW3sgl93X7iKYQmThST97ha9W06au4wCh74CNRzCVKc=\n-----END CERTIFICATE-----\n",
      "expirationDate": 1676861421,
      "isRevoked": false
    }
  ]
}

# 将返回的certContent保存为ca.key文件
echo -n "-----BEGIN CERTIFICATE-----\nMIICaDCCAg6gAwIBAgIIF4x2edRiVb8wCgYIKoZIzj0EAwIwYjELMAkGA1UEBhMC\nQ04xEDAOBgNVBAgTB0JlaWppbmcxEDAOBgNVBAcTB0JlaWppbmcxETAPBgNVBAoT\nCG9yZy1yb290MQ0wCwYDVQQLEwRyb290MQ0wCwYDVQQDEwRyb290MB4XDTIyMDgy\nNDAyNTAyMVoXDTIzMDIyMDAyNTAyMVowgYMxCzAJBgNVBAYTAkNOMRAwDgYDVQQI\nEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNo\nYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZY2Etd3gtb3JnMS5j\naGFpbm1ha2VyLm9yZzBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABJm8UUrxkzvL\nzB6u++UjdTvFhy9Fay1h+X7xVijPp8LeyAX70P5O4VGvik7j/yCHoJ0GE89pxH4B\nUIq6WnVTtDWjgYswgYgwDgYDVR0PAQH/BAQDAgEGMA8GA1UdEwEB/wQFMAMBAf8w\nKQYDVR0OBCIEIFQUtaMGHEYWLDGcfWBR0SQZjuWdI31bW7tCAyrplWpuMCsGA1Ud\nIwQkMCKAID2QQznj2uOhDBY0zQDZs0PsixvqNqTx5WLTT7JErCJMMA0GA1UdEQQG\nMASCAIIAMAoGCCqGSM49BAMCA0gAMEUCIFHIdvzax2/g183R6RU1S0jqRZNiKvrL\nLPzccecq0N/1AiEApW3sgl93X7iKYQmThST97ha9W06au4wCh74CNRzCVKc=\n-----END CERTIFICATE-----\n" > ca.crt
```

- 查看组织org1的CA证书
```shell
$ openssl x509 -in ca.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1696861425238169023 (0x178c7679d46255bf)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: C = CN, ST = Beijing, L = Beijing, O = org-root, OU = root, CN = root
        Validity
            Not Before: Aug 24 02:50:21 2022 GMT
            Not After : Feb 20 02:50:21 2023 GMT
        Subject: C = CN, ST = Beijing, L = Beijing, O = wx-org1.chainmaker.org, OU = ca, CN = ca-wx-org1.chainmaker.org
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:99:bc:51:4a:f1:93:3b:cb:cc:1e:ae:fb:e5:23:
                    75:3b:c5:87:2f:45:6b:2d:61:f9:7e:f1:56:28:cf:
                    a7:c2:de:c8:05:fb:d0:fe:4e:e1:51:af:8a:4e:e3:
                    ff:20:87:a0:9d:06:13:cf:69:c4:7e:01:50:8a:ba:
                    5a:75:53:b4:35
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier:
                54:14:B5:A3:06:1C:46:16:2C:31:9C:7D:60:51:D1:24:19:8E:E5:9D:23:7D:5B:5B:BB:42:03:2A:E9:95:6A:6E
            X509v3 Authority Key Identifier:
                keyid:3D:90:43:39:E3:DA:E3:A1:0C:16:34:CD:00:D9:B3:43:EC:8B:1B:EA:36:A4:F1:E5:62:D3:4F:B2:44:AC:22:4C

            X509v3 Subject Alternative Name:
                DNS:, DNS:
    Signature Algorithm: ecdsa-with-SHA256
         30:45:02:20:51:c8:76:fc:da:c7:6f:e0:d7:cd:d1:e9:15:35:
         4b:48:ea:45:93:62:2a:fa:cb:2c:fc:dc:71:e7:2a:d0:df:f5:
         02:21:00:a5:6d:ec:82:5f:77:5f:b8:8a:61:09:93:85:24:fd:
         ee:16:bd:5b:4e:9a:bb:8c:02:87:be:02:35:1c:c2:54:a7
```
从证书Subject字段内容上我们可以看出，该证书代表组织`wx-org1.chainmaker.org`

分别指定不同的`orgId`获取`wx-org2.chainmaker.org`，`wx-org3.chainmaker.org`和`wx-org4.chainmaker.org`的根证书。

#### 生成节点证书

通过组织根CA签发共识节点证书，以org1为例：

- 共识节点（consensus node）Sign证书

*注：生成共识节点证书时，userId需要保证链上唯一；同一节点的Sign和Tls证书，userId需要保持一致。*

```shell
curl --location --request POST 'http://localhost:8096/api/ca/gencert' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org1.chainmaker.org",
    "userId": "org1.consensus1.com",
    "userType": "consensus",
    "certUsage": "sign",
    "country": "CN",
    "locality": "BeiJing",
    "province": "BeiJing"
}'  | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1559  100  1352  100   207  26000   3980 --:--:-- --:--:-- --:--:-- 29980
{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": {
    "certSn": 2694363940699227000,
    "issueCertSn": 1696861425238169000,
    "cert": "-----BEGIN CERTIFICATE-----\nMIICjTCCAjOgAwIBAgIIJWRNuoz/xNYwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNDAzNTZaFw0yMzAy\nMjAwNDAzNTZaMIGEMQswCQYDVQQGEwJDTjEQMA4GA1UECBMHQmVpSmluZzEQMA4G\nA1UEBxMHQmVpSmluZzEfMB0GA1UEChMWd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzES\nMBAGA1UECxMJY29uc2Vuc3VzMRwwGgYDVQQDExNvcmcxLmNvbnNlbnN1czEuY29t\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE63g+GfDmTKtuKn/Kv+0Z/gdwo7PB\nVioKfe3Om9NuQPklmn5fQju8KHBdEbCxvlsSsw1HjfLEuPyCUfsdCEVnuaOBjTCB\nijAOBgNVHQ8BAf8EBAMCBsAwKQYDVR0OBCIEICijhLeuc7ogfjdCOsVtypWmwWIn\n4pCUfuWuhIjFk3jAMCsGA1UdIwQkMCKAIFQUtaMGHEYWLDGcfWBR0SQZjuWdI31b\nW7tCAyrplWpuMCAGA1UdEQQZMBeCE29yZzEuY29uc2Vuc3VzMS5jb22CADAKBggq\nhkjOPQQDAgNIADBFAiEAwvcrJHmxaU27wolEIMsYpYJVbbbDepRWN3OVWuWLg4MC\nIDTIoD0gMTJekIkNJbGSdXspotWHAQp6BA0jV277LvkU\n-----END CERTIFICATE-----\n",
    "privateKey": "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIG7UtWyFE55mrAn9yUAx5EtrMD5eqAb619NZBjT7y/iVoAoGCCqGSM49\nAwEHoUQDQgAE63g+GfDmTKtuKn/Kv+0Z/gdwo7PBVioKfe3Om9NuQPklmn5fQju8\nKHBdEbCxvlsSsw1HjfLEuPyCUfsdCEVnuQ==\n-----END EC PRIVATE KEY-----\n"
  }
}

# 保存节点sign证书和sign私钥到文件中
$ echo -n "-----BEGIN CERTIFICATE-----\nMIICjTCCAjOgAwIBAgIIJWRNuoz/xNYwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNDAzNTZaFw0yMzAy\nMjAwNDAzNTZaMIGEMQswCQYDVQQGEwJDTjEQMA4GA1UECBMHQmVpSmluZzEQMA4G\nA1UEBxMHQmVpSmluZzEfMB0GA1UEChMWd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzES\nMBAGA1UECxMJY29uc2Vuc3VzMRwwGgYDVQQDExNvcmcxLmNvbnNlbnN1czEuY29t\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE63g+GfDmTKtuKn/Kv+0Z/gdwo7PB\nVioKfe3Om9NuQPklmn5fQju8KHBdEbCxvlsSsw1HjfLEuPyCUfsdCEVnuaOBjTCB\nijAOBgNVHQ8BAf8EBAMCBsAwKQYDVR0OBCIEICijhLeuc7ogfjdCOsVtypWmwWIn\n4pCUfuWuhIjFk3jAMCsGA1UdIwQkMCKAIFQUtaMGHEYWLDGcfWBR0SQZjuWdI31b\nW7tCAyrplWpuMCAGA1UdEQQZMBeCE29yZzEuY29uc2Vuc3VzMS5jb22CADAKBggq\nhkjOPQQDAgNIADBFAiEAwvcrJHmxaU27wolEIMsYpYJVbbbDepRWN3OVWuWLg4MC\nIDTIoD0gMTJekIkNJbGSdXspotWHAQp6BA0jV277LvkU\n-----END CERTIFICATE-----\n" > consensus1.sign.crt
$ echo -n "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIG7UtWyFE55mrAn9yUAx5EtrMD5eqAb619NZBjT7y/iVoAoGCCqGSM49\nAwEHoUQDQgAE63g+GfDmTKtuKn/Kv+0Z/gdwo7PBVioKfe3Om9NuQPklmn5fQju8\nKHBdEbCxvlsSsw1HjfLEuPyCUfsdCEVnuQ==\n-----END EC PRIVATE KEY-----\n" > consensus1.sign.key

# 查看节点sign证书
 openssl x509 -in consensus1.sign.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2694363940699227350 (0x25644dba8cffc4d6)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: C = CN, ST = Beijing, L = Beijing, O = wx-org1.chainmaker.org, OU = ca, CN = ca-wx-org1.chainmaker.org
        Validity
            Not Before: Aug 24 04:03:56 2022 GMT
            Not After : Feb 20 04:03:56 2023 GMT
        Subject: C = CN, ST = BeiJing, L = BeiJing, O = wx-org1.chainmaker.org, OU = consensus, CN = org1.consensus1.com
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:eb:78:3e:19:f0:e6:4c:ab:6e:2a:7f:ca:bf:ed:
                    19:fe:07:70:a3:b3:c1:56:2a:0a:7d:ed:ce:9b:d3:
                    6e:40:f9:25:9a:7e:5f:42:3b:bc:28:70:5d:11:b0:
                    b1:be:5b:12:b3:0d:47:8d:f2:c4:b8:fc:82:51:fb:
                    1d:08:45:67:b9
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation
            X509v3 Subject Key Identifier:
                28:A3:84:B7:AE:73:BA:20:7E:37:42:3A:C5:6D:CA:95:A6:C1:62:27:E2:90:94:7E:E5:AE:84:88:C5:93:78:C0
            X509v3 Authority Key Identifier:
                keyid:54:14:B5:A3:06:1C:46:16:2C:31:9C:7D:60:51:D1:24:19:8E:E5:9D:23:7D:5B:5B:BB:42:03:2A:E9:95:6A:6E

            X509v3 Subject Alternative Name:
                DNS:org1.consensus1.com, DNS:
    Signature Algorithm: ecdsa-with-SHA256
         30:45:02:21:00:c2:f7:2b:24:79:b1:69:4d:bb:c2:89:44:20:
         cb:18:a5:82:55:6d:b6:c3:7a:94:56:37:73:95:5a:e5:8b:83:
         83:02:20:34:c8:a0:3d:20:31:32:5e:90:89:0d:25:b1:92:75:
         7b:29:a2:d5:87:01:0a:7a:04:0d:23:57:6e:fb:2e:f9:14
```
从证书Subject字段内容上我们可以看到，该证书账户属于`wx-org1.chainmaker.org`组织，角色为`consensus`

- 共识节点（consensus node）Tls证书

*注：生成共识节点证书时，userId需要保证链上唯一；同一节点的Sign和Tls证书，userId需要保持一致。*

```shell
 curl --location --request POST 'http://localhost:8096/api/ca/gencert' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org1.chainmaker.org",
    "userId": "org1.consensus1.com",
    "userType": "consensus",
    "certUsage": "tls",
    "country": "CN",
    "locality": "BeiJing",
    "province": "BeiJing"
}'  | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1600  100  1394  100   206  26807   3961 --:--:-- --:--:-- --:--:-- 30769
{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": {
    "certSn": 1344498995994953700,
    "issueCertSn": 1696861425238169000,
    "cert": "-----BEGIN CERTIFICATE-----\nMIICqzCCAlKgAwIBAgIIEqietqEmB/IwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNDEyNTVaFw0yMzAy\nMjAwNDEyNTVaMIGEMQswCQYDVQQGEwJDTjEQMA4GA1UECBMHQmVpSmluZzEQMA4G\nA1UEBxMHQmVpSmluZzEfMB0GA1UEChMWd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzES\nMBAGA1UECxMJY29uc2Vuc3VzMRwwGgYDVQQDExNvcmcxLmNvbnNlbnN1czEuY29t\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEwcTZ0srq9NqDaXKbgyBA5YlSrqZQ\nTOchtoMQLOxpjfsCNLanNGpqFDgg2Aa8rEv77/GW8NAl2RQEuRQES//YwaOBrDCB\nqTAOBgNVHQ8BAf8EBAMCA/gwHQYDVR0lBBYwFAYIKwYBBQUHAwIGCCsGAQUFBwMB\nMCkGA1UdDgQiBCCVdXEP1II8b1lCnDbVT6nokYwOShyxV4NbP42nayY1xDArBgNV\nHSMEJDAigCBUFLWjBhxGFiwxnH1gUdEkGY7lnSN9W1u7QgMq6ZVqbjAgBgNVHREE\nGTAXghNvcmcxLmNvbnNlbnN1czEuY29tggAwCgYIKoZIzj0EAwIDRwAwRAIgeGaY\nTebfVu8sCrwZZrJDg0D4n2qTQoZcBy6LWOCSjRACIC5X+H+1NeMW3gPQ6fSSwd/x\n5YDXog2pUocoUuUhiphJ\n-----END CERTIFICATE-----\n",
    "privateKey": "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEICXRU0a1Jnz9VReDVLA7W5AQUUg6f/22uFuaJgg7aZQtoAoGCCqGSM49\nAwEHoUQDQgAEwcTZ0srq9NqDaXKbgyBA5YlSrqZQTOchtoMQLOxpjfsCNLanNGpq\nFDgg2Aa8rEv77/GW8NAl2RQEuRQES//YwQ==\n-----END EC PRIVATE KEY-----\n"
  }
}

# 将返回的节点tls私钥和节点tls证书保存到文件中：
$ echo -n "-----BEGIN CERTIFICATE-----\nMIICqzCCAlKgAwIBAgIIEqietqEmB/IwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNDEyNTVaFw0yMzAy\nMjAwNDEyNTVaMIGEMQswCQYDVQQGEwJDTjEQMA4GA1UECBMHQmVpSmluZzEQMA4G\nA1UEBxMHQmVpSmluZzEfMB0GA1UEChMWd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzES\nMBAGA1UECxMJY29uc2Vuc3VzMRwwGgYDVQQDExNvcmcxLmNvbnNlbnN1czEuY29t\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEwcTZ0srq9NqDaXKbgyBA5YlSrqZQ\nTOchtoMQLOxpjfsCNLanNGpqFDgg2Aa8rEv77/GW8NAl2RQEuRQES//YwaOBrDCB\nqTAOBgNVHQ8BAf8EBAMCA/gwHQYDVR0lBBYwFAYIKwYBBQUHAwIGCCsGAQUFBwMB\nMCkGA1UdDgQiBCCVdXEP1II8b1lCnDbVT6nokYwOShyxV4NbP42nayY1xDArBgNV\nHSMEJDAigCBUFLWjBhxGFiwxnH1gUdEkGY7lnSN9W1u7QgMq6ZVqbjAgBgNVHREE\nGTAXghNvcmcxLmNvbnNlbnN1czEuY29tggAwCgYIKoZIzj0EAwIDRwAwRAIgeGaY\nTebfVu8sCrwZZrJDg0D4n2qTQoZcBy6LWOCSjRACIC5X+H+1NeMW3gPQ6fSSwd/x\n5YDXog2pUocoUuUhiphJ\n-----END CERTIFICATE-----\n" > consensus1.tls.crt
$ echo -n "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEICXRU0a1Jnz9VReDVLA7W5AQUUg6f/22uFuaJgg7aZQtoAoGCCqGSM49\nAwEHoUQDQgAEwcTZ0srq9NqDaXKbgyBA5YlSrqZQTOchtoMQLOxpjfsCNLanNGpq\nFDgg2Aa8rEv77/GW8NAl2RQEuRQES//YwQ==\n-----END EC PRIVATE KEY-----\n" > consensus1.tls.key

# 查看节点tls证书
$ openssl x509 -in consensus1.tls.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2694363940699227350 (0x25644dba8cffc4d6)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: C = CN, ST = Beijing, L = Beijing, O = wx-org1.chainmaker.org, OU = ca, CN = ca-wx-org1.chainmaker.org
        Validity
            Not Before: Aug 24 04:03:56 2022 GMT
            Not After : Feb 20 04:03:56 2023 GMT
        Subject: C = CN, ST = BeiJing, L = BeiJing, O = wx-org1.chainmaker.org, OU = consensus, CN = org1.consensus1.com
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:eb:78:3e:19:f0:e6:4c:ab:6e:2a:7f:ca:bf:ed:
                    19:fe:07:70:a3:b3:c1:56:2a:0a:7d:ed:ce:9b:d3:
                    6e:40:f9:25:9a:7e:5f:42:3b:bc:28:70:5d:11:b0:
                    b1:be:5b:12:b3:0d:47:8d:f2:c4:b8:fc:82:51:fb:
                    1d:08:45:67:b9
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation
            X509v3 Subject Key Identifier:
                28:A3:84:B7:AE:73:BA:20:7E:37:42:3A:C5:6D:CA:95:A6:C1:62:27:E2:90:94:7E:E5:AE:84:88:C5:93:78:C0
            X509v3 Authority Key Identifier:
                keyid:54:14:B5:A3:06:1C:46:16:2C:31:9C:7D:60:51:D1:24:19:8E:E5:9D:23:7D:5B:5B:BB:42:03:2A:E9:95:6A:6E

            X509v3 Subject Alternative Name:
                DNS:org1.consensus1.com, DNS:
    Signature Algorithm: ecdsa-with-SHA256
         30:45:02:21:00:c2:f7:2b:24:79:b1:69:4d:bb:c2:89:44:20:
         cb:18:a5:82:55:6d:b6:c3:7a:94:56:37:73:95:5a:e5:8b:83:
         83:02:20:34:c8:a0:3d:20:31:32:5e:90:89:0d:25:b1:92:75:
         7b:29:a2:d5:87:01:0a:7a:04:0d:23:57:6e:fb:2e:f9:14
```
从证书Subject字段内容上我们可以看到，该证书账户属于`wx-org1.chainmaker.org`组织，角色为`consensus`

- 获取共识节点的NodeId

获取共识节点（consensus node）Tls证书的NodeId

```shell
 curl --location --request POST 'http://localhost:8096/api/ca/getnodeid' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org1.chainmaker.org",
    "userId": "org1.consensus1.com",
    "userType": "consensus",
    "certUsage": "tls"
}' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   249  100   118  100   131   6555   7277 --:--:-- --:--:-- --:--:-- 13833
{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": "QmNgWQTRn8ijLThpu7sknFqEmGo2jELT32Wuwt3ZsnqKAP" #节点nodeId
}
```

注：重复以上步骤，依次生成组织org2，org3，org4下共识节点sign私钥/证书、tls私钥/证书以及节点的NodeId

### 基于生成的组织和节点证书创建链
获取的组织CA证书，需要在启动链时，将他们配置到链配置文件`bc1.yml`的`trust_roots`里，并将`bc1.yml`和`chainmaker.yml`中的`nodes.node_id`替换为以上获取的节点nodeId。

**配置文件修改位置如下**

- bc1.yml（链配置文件）

```yaml
#共识配置
consensus:
  # 共识类型(0-SOLO,1-TBFT,2-MBFT,3-MAXBFT,4-RAFT,10-POW)
  type: 1
  # 共识节点列表，组织必须出现在trust_roots的org_id中，每个组织可配置多个共识节点，节点地址采用libp2p格式
  nodes:
    - org_id: "wx-org1.chainmaker.org"
      node_id:
        - "QmNgWQTRn8ijLThpu7sknFqEmGo2jELT32Wuwt3ZsnqKAP"
    - org_id: "wx-org2.chainmaker.org"
      node_id:
        - "QmT5hVtAABN69BPAiUjdXddC4ZY9qJf2HgNuRDLSPT8V7R"
    - org_id: "wx-org3.chainmaker.org"
      node_id:
        - "QmeWW4h7XFADkcabEmzEdWzMNeWzHZa494W2pVprv61a4R"
    - org_id: "wx-org4.chainmaker.org"
      node_id:
        - "QmcXaixSAteMZWi4MbqrPijoG8koR6igx3Mrjfe7iHF5iw"

# Trust roots is used to specify the organizations' root certificates in permessionedWithCert mode.
# When in permessionedWithKey mode or public mode, it represents the admin users.
trust_roots:
  - org_id: "wx-org4.chainmaker.org"
    root:
      - "../config/wx-org1.chainmaker.org/certs/ca/wx-org4.chainmaker.org/ca.crt"
  - org_id: "wx-org3.chainmaker.org"
    root:
      - "../config/wx-org1.chainmaker.org/certs/ca/wx-org3.chainmaker.org/ca.crt"
  - org_id: "wx-org2.chainmaker.org"
    root:
      - "../config/wx-org1.chainmaker.org/certs/ca/wx-org2.chainmaker.org/ca.crt"
  - org_id: "wx-org1.chainmaker.org"
    root:
      - "../config/wx-org1.chainmaker.org/certs/ca/wx-org1.chainmaker.org/ca.crt"
```

- chainmaker.yml （节点配置文件）

```yaml
# Network Settings
net:
  # Network provider, can be libp2p or liquid.
  # libp2p: using libp2p components to build the p2p module.
  # liquid: a new p2p module we build from 0 to 1.
  # This item must be consistent across the blockchain network.
  provider: LibP2P

  # The address and port the node listens on.
  # By default, it uses 0.0.0.0 to listen on all network interfaces.
  listen_addr: /ip4/0.0.0.0/tcp/11301

  # The seeds peer list used to join in the network when starting.
  # The connection supervisor will try to dial seed peer whenever the connection is broken.
  # Example ip format: "/ip4/127.0.0.1/tcp/11301/p2p/"+nodeid
  # Example dns format："/dns/cm-node1.org/tcp/11301/p2p/"+nodeid
  seeds:
    - "/ip4/127.0.0.1/tcp/11301/p2p/QmNgWQTRn8ijLThpu7sknFqEmGo2jELT32Wuwt3ZsnqKAP"
    - "/ip4/127.0.0.1/tcp/11302/p2p/QmT5hVtAABN69BPAiUjdXddC4ZY9qJf2HgNuRDLSPT8V7R"
    - "/ip4/127.0.0.1/tcp/11303/p2p/QmeWW4h7XFADkcabEmzEdWzMNeWzHZa494W2pVprv61a4R"
    - "/ip4/127.0.0.1/tcp/11304/p2p/QmcXaixSAteMZWi4MbqrPijoG8koR6igx3Mrjfe7iHF5iw"
```


节点部署目录如下图所示：
```shell
.
├── bin
│   ├── chainmaker
│   ├── chainmaker.service
│   ├── docker-vm-standalone-start.sh
│   ├── docker-vm-standalone-stop.sh
│   ├── init.sh
│   ├── panic.log
│   ├── restart.sh
│   ├── run.sh
│   ├── start.sh
│   └── stop.sh
├── config
│   └── wx-org1.chainmaker.org
├── lib
│   ├── libwasmer.dylib
│   └── wxdec
└── log
```

在启动节点之前，需要将以上部署过程中用到的各个组织的根证书、共识节点证书和私钥、管理员证书和私钥替换为以上用CA生成的私钥/证书。
同时各个节点部署目录下的`bc.yml`和`chainmaker.yml`配置文件按照以上说明进行修改。

**配置更新说明**  
```shell
# 以组织org1为例说明
config
└── wx-org1.chainmaker.org
    ├── certs
    │   ├── ca
    │   │   ├── wx-org1.chainmaker.org #组织1根证书
    │   │   ├── wx-org2.chainmaker.org #组织2根证书
    │   │   ├── wx-org3.chainmaker.org #组织3根证书
    │   │   └── wx-org4.chainmaker.org #组织4根证书
    │   ├── node
    │   │   ├── common1 # optional
    │   │   └── consensus1 # 组织1签发的共识节点sign证书、tls证书以及nodeId
    │   └── user
    │       ├── admin1 # 组织1签发的管理员sign证书、tls证书（包含私钥）
    │       ├── client1 #optional
    │       └── light1 #optional
    ├── chainconfig
    │   └── bc1.yml # 更新后的链配置文件，参考上节说明
    ├── chainmaker.yml #更新后的节点配置文件，参考上节说明
    └── log.yml
```

**启动节点**  

节点启动方式参考：
- docker部署方式，参考[docker方式-启动节点](./通过Docker部署链.html#启动节点)
- 命令行部署方式，参考[通过命令行部署链](../quickstart/通过命令行体验链.html#节点启动)

请自行将上述两文章内的证书文件替换成上述流程新生成的证书文件，然后按步骤执行，直至链启动成功。

按照以上方式启动org1、org2、org3和org4下的共识节点，等到所有节点建立连接，表明区块链网络部署成功，并可以对外提供区块链服务。  
可通过遗下命令查看节点日志，若看到all necessary peers connected则表示节点已经准备就绪。

```shell
[INFO]  [Net]   libp2pnet/libp2p_connection_supervisor.go:116   [ConnSupervisor] all necessary peers connected.
```

### 部署/调用智能合约
链部署后，在进行部署/调用合约测试，以验证链是否正常运行，首先我们需要先生成链的用户证书。
#### 生成admin测试账户 
客户端账户包含client和admin两种类型，admin一般具有更高的链上权限，我们这里以admin为例进行测试
- 用户管理员（admin）Sign证书
```shell
# 生成私钥和证书
 curl --location --request POST 'http://localhost:8096/api/ca/gencert' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org1.chainmaker.org",
    "userId": "admin1",
    "userType": "admin",
    "certUsage": "sign",
    "country": "CN",
    "locality": "BeiJing",
    "province": "BeiJing"
}' | jq
{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": {
    "certSn": 1856171032692815600,
    "issueCertSn": 1696861425238169000,
    "cert": "-----BEGIN CERTIFICATE-----\nMIICZTCCAgygAwIBAgIIGcJxuEYGd3gwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNTQ1MjNaFw0yMzAy\nMjAwNTQ1MjNaMHMxCzAJBgNVBAYTAkNOMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYD\nVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQ4w\nDAYDVQQLEwVhZG1pbjEPMA0GA1UEAxMGYWRtaW4xMFkwEwYHKoZIzj0CAQYIKoZI\nzj0DAQcDQgAEr2NLaz88XzSwFHLwbFgwGo80CjjRwVeRvstB6uzsT6wd0RPy7R5+\nSUP4sEBnEgsLCGFmNZ0wwdTkesfZLXLFdKN5MHcwDgYDVR0PAQH/BAQDAgbAMCkG\nA1UdDgQiBCBMvi1OVMM1nZHL1OCFjsg8cgAjiYT7TRHt062OozHarTArBgNVHSME\nJDAigCBUFLWjBhxGFiwxnH1gUdEkGY7lnSN9W1u7QgMq6ZVqbjANBgNVHREEBjAE\nggCCADAKBggqhkjOPQQDAgNHADBEAiB3IaMiuGeLQCDVGpSFJUjXcdEJ7MjizZj/\nOu08VrWniQIgeH5jyUMG3Izu7dh064iSRidzgR7sLt48CWtXpCuI4nE=\n-----END CERTIFICATE-----\n",
    "privateKey": "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIBQy4VFzQ+TH3i23VzuWLNPzhS0k3F2IMC/gT2suJfXDoAoGCCqGSM49\nAwEHoUQDQgAEr2NLaz88XzSwFHLwbFgwGo80CjjRwVeRvstB6uzsT6wd0RPy7R5+\nSUP4sEBnEgsLCGFmNZ0wwdTkesfZLXLFdA==\n-----END EC PRIVATE KEY-----\n"
  }
}

# 保存管理员sign证书和私钥
$ echo -n "-----BEGIN CERTIFICATE-----\nMIICZTCCAgygAwIBAgIIGcJxuEYGd3gwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNTQ1MjNaFw0yMzAy\nMjAwNTQ1MjNaMHMxCzAJBgNVBAYTAkNOMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYD\nVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQ4w\nDAYDVQQLEwVhZG1pbjEPMA0GA1UEAxMGYWRtaW4xMFkwEwYHKoZIzj0CAQYIKoZI\nzj0DAQcDQgAEr2NLaz88XzSwFHLwbFgwGo80CjjRwVeRvstB6uzsT6wd0RPy7R5+\nSUP4sEBnEgsLCGFmNZ0wwdTkesfZLXLFdKN5MHcwDgYDVR0PAQH/BAQDAgbAMCkG\nA1UdDgQiBCBMvi1OVMM1nZHL1OCFjsg8cgAjiYT7TRHt062OozHarTArBgNVHSME\nJDAigCBUFLWjBhxGFiwxnH1gUdEkGY7lnSN9W1u7QgMq6ZVqbjANBgNVHREEBjAE\nggCCADAKBggqhkjOPQQDAgNHADBEAiB3IaMiuGeLQCDVGpSFJUjXcdEJ7MjizZj/\nOu08VrWniQIgeH5jyUMG3Izu7dh064iSRidzgR7sLt48CWtXpCuI4nE=\n-----END CERTIFICATE-----\n" > admin1.sign.crt
$ echo -n "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIBQy4VFzQ+TH3i23VzuWLNPzhS0k3F2IMC/gT2suJfXDoAoGCCqGSM49\nAwEHoUQDQgAEr2NLaz88XzSwFHLwbFgwGo80CjjRwVeRvstB6uzsT6wd0RPy7R5+\nSUP4sEBnEgsLCGFmNZ0wwdTkesfZLXLFdA==\n-----END EC PRIVATE KEY-----\n" > admin1.sign.key

# 查看管理员sign证书
$ openssl x509 -in admin1.sign.crt -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1856171032692815736 (0x19c271b846067778)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: C = CN, ST = Beijing, L = Beijing, O = wx-org1.chainmaker.org, OU = ca, CN = ca-wx-org1.chainmaker.org
        Validity
            Not Before: Aug 24 05:45:23 2022 GMT
            Not After : Feb 20 05:45:23 2023 GMT
        Subject: C = CN, ST = BeiJing, L = BeiJing, O = wx-org1.chainmaker.org, OU = admin, CN = admin1
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:af:63:4b:6b:3f:3c:5f:34:b0:14:72:f0:6c:58:
                    30:1a:8f:34:0a:38:d1:c1:57:91:be:cb:41:ea:ec:
                    ec:4f:ac:1d:d1:13:f2:ed:1e:7e:49:43:f8:b0:40:
                    67:12:0b:0b:08:61:66:35:9d:30:c1:d4:e4:7a:c7:
                    d9:2d:72:c5:74
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation
            X509v3 Subject Key Identifier:
                4C:BE:2D:4E:54:C3:35:9D:91:CB:D4:E0:85:8E:C8:3C:72:00:23:89:84:FB:4D:11:ED:D3:AD:8E:A3:31:DA:AD
            X509v3 Authority Key Identifier:
                keyid:54:14:B5:A3:06:1C:46:16:2C:31:9C:7D:60:51:D1:24:19:8E:E5:9D:23:7D:5B:5B:BB:42:03:2A:E9:95:6A:6E

            X509v3 Subject Alternative Name:
                DNS:, DNS:
    Signature Algorithm: ecdsa-with-SHA256
         30:44:02:20:77:21:a3:22:b8:67:8b:40:20:d5:1a:94:85:25:
         48:d7:71:d1:09:ec:c8:e2:cd:98:ff:3a:ed:3c:56:b5:a7:89:
         02:20:78:7e:63:c9:43:06:dc:8c:ee:ed:d8:74:eb:88:92:46:
         27:73:81:1e:ec:2e:de:3c:09:6b:57:a4:2b:88:e2:71
```
从证书Subject字段内容上我们可以看到，该管理员证书账户属于`wx-org1.chainmaker.org`组织，角色为`admin`

- 用户管理员（admin）Tls证书

```shell
# 生成管理员tls证书
curl --location --request POST 'http://localhost:8096/api/ca/gencert' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org1.chainmaker.org",
    "userId": "admin1",
    "userType": "admin",
    "certUsage": "tls",
    "country": "CN",
    "locality": "BeiJing",
    "province": "BeiJing"
}' | jq

{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": {
    "certSn": 3793576108139400000,
    "issueCertSn": 1696861425238169000,
    "cert": "-----BEGIN CERTIFICATE-----\nMIICfDCCAiOgAwIBAgIINKV9XwIOzHEwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNTUzMTRaFw0yMzAy\nMjAwNTUzMTRaMHMxCzAJBgNVBAYTAkNOMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYD\nVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQ4w\nDAYDVQQLEwVhZG1pbjEPMA0GA1UEAxMGYWRtaW4xMFkwEwYHKoZIzj0CAQYIKoZI\nzj0DAQcDQgAE4lJgyz1A/GledacE3HUCwT2MrWHqDzHHpy81RTY22Ap6uj/ahO1h\n1+o0GSGS8xdF0ygw07BhtBOe9dkuYvptQ6OBjzCBjDAOBgNVHQ8BAf8EBAMCA/gw\nEwYDVR0lBAwwCgYIKwYBBQUHAwIwKQYDVR0OBCIEIJ9OT40+LKdBATiWmfeIXmXO\n2RkxhbG1+Ai5vt+l5LxsMCsGA1UdIwQkMCKAIFQUtaMGHEYWLDGcfWBR0SQZjuWd\nI31bW7tCAyrplWpuMA0GA1UdEQQGMASCAIIAMAoGCCqGSM49BAMCA0cAMEQCIDbU\nvsm/C2OPw5HbhZCBzZJNbJ5x3QW+I8kOKNZd4X7jAiAq12qTcqZcNSBKsqM9HRTj\nqt/2rzh38uEgu/2oUvpIUw==\n-----END CERTIFICATE-----\n",
    "privateKey": "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIIA9k6+tvHrCa0Ee3P+IlGO0AQFW3GR6VSYtD33lK+PDoAoGCCqGSM49\nAwEHoUQDQgAE4lJgyz1A/GledacE3HUCwT2MrWHqDzHHpy81RTY22Ap6uj/ahO1h\n1+o0GSGS8xdF0ygw07BhtBOe9dkuYvptQw==\n-----END EC PRIVATE KEY-----\n"
  }
}

# 保存管理员tls证书和私钥
$ echo -n "-----BEGIN CERTIFICATE-----\nMIICfDCCAiOgAwIBAgIINKV9XwIOzHEwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNTUzMTRaFw0yMzAy\nMjAwNTUzMTRaMHMxCzAJBgNVBAYTAkNOMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYD\nVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQ4w\nDAYDVQQLEwVhZG1pbjEPMA0GA1UEAxMGYWRtaW4xMFkwEwYHKoZIzj0CAQYIKoZI\nzj0DAQcDQgAE4lJgyz1A/GledacE3HUCwT2MrWHqDzHHpy81RTY22Ap6uj/ahO1h\n1+o0GSGS8xdF0ygw07BhtBOe9dkuYvptQ6OBjzCBjDAOBgNVHQ8BAf8EBAMCA/gw\nEwYDVR0lBAwwCgYIKwYBBQUHAwIwKQYDVR0OBCIEIJ9OT40+LKdBATiWmfeIXmXO\n2RkxhbG1+Ai5vt+l5LxsMCsGA1UdIwQkMCKAIFQUtaMGHEYWLDGcfWBR0SQZjuWd\nI31bW7tCAyrplWpuMA0GA1UdEQQGMASCAIIAMAoGCCqGSM49BAMCA0cAMEQCIDbU\nvsm/C2OPw5HbhZCBzZJNbJ5x3QW+I8kOKNZd4X7jAiAq12qTcqZcNSBKsqM9HRTj\nqt/2rzh38uEgu/2oUvpIUw==\n-----END CERTIFICATE-----\n" > admin1.tls.crt
$ echo -n "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIIA9k6+tvHrCa0Ee3P+IlGO0AQFW3GR6VSYtD33lK+PDoAoGCCqGSM49\nAwEHoUQDQgAE4lJgyz1A/GledacE3HUCwT2MrWHqDzHHpy81RTY22Ap6uj/ahO1h\n1+o0GSGS8xdF0ygw07BhtBOe9dkuYvptQw==\n-----END EC PRIVATE KEY-----\n" > admin1.tls.key

# 查看管理员tls证书
$ openssl x509 -in admin1.tls.crt -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 3793576108139400305 (0x34a57d5f020ecc71)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: C = CN, ST = Beijing, L = Beijing, O = wx-org1.chainmaker.org, OU = ca, CN = ca-wx-org1.chainmaker.org
        Validity
            Not Before: Aug 24 05:53:14 2022 GMT
            Not After : Feb 20 05:53:14 2023 GMT
        Subject: C = CN, ST = BeiJing, L = BeiJing, O = wx-org1.chainmaker.org, OU = admin, CN = admin1
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:e2:52:60:cb:3d:40:fc:69:5e:75:a7:04:dc:75:
                    02:c1:3d:8c:ad:61:ea:0f:31:c7:a7:2f:35:45:36:
                    36:d8:0a:7a:ba:3f:da:84:ed:61:d7:ea:34:19:21:
                    92:f3:17:45:d3:28:30:d3:b0:61:b4:13:9e:f5:d9:
                    2e:62:fa:6d:43
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation, Key Encipherment, Data Encipherment, Key Agreement
            X509v3 Extended Key Usage:
                TLS Web Client Authentication
            X509v3 Subject Key Identifier:
                9F:4E:4F:8D:3E:2C:A7:41:01:38:96:99:F7:88:5E:65:CE:D9:19:31:85:B1:B5:F8:08:B9:BE:DF:A5:E4:BC:6C
            X509v3 Authority Key Identifier:
                keyid:54:14:B5:A3:06:1C:46:16:2C:31:9C:7D:60:51:D1:24:19:8E:E5:9D:23:7D:5B:5B:BB:42:03:2A:E9:95:6A:6E

            X509v3 Subject Alternative Name:
                DNS:, DNS:
    Signature Algorithm: ecdsa-with-SHA256
         30:44:02:20:36:d4:be:c9:bf:0b:63:8f:c3:91:db:85:90:81:
         cd:92:4d:6c:9e:71:dd:05:be:23:c9:0e:28:d6:5d:e1:7e:e3:
         02:20:2a:d7:6a:93:72:a6:5c:35:20:4a:b2:a3:3d:1d:14:e3:
         aa:df:f6:af:38:77:f2:e1:20:bb:fd:a8:52:fa:48:53
```
从证书Subject字段内容上我们可以看到，该管理员证书账户属于`wx-org1.chainmaker.org`组织

#### 使用管理员账户部署/调用合约
admin的用户证书生成后，我们在进行合约的部署和调用，目前支持三种方式进行合约的部署和调用。示例合约可从[此处获得](../instructions/部署示例合约.md)
- 通过长安链CMC命令行工具进行调用，详细教程请见命令行[交易功能](../dev/命令行工具.html#交易功能)
  - 使用长安链命令行工具cmc进行测试，cmc工具配置文件sdk_config.yaml需要进行相应修改，主要修改以下几个配置:
    - 客户端tls私钥和证书：user_key_file_path、user_crt_file_path
    - 客户端sign私钥和证书：user_sign_key_file_path、user_sign_crt_file_path
    - 可信根目录：trust_root_paths (注：该目录下需要包含该管理员账户组织ca根证书)
    - 节点tls证书CN或SAN：tls_host_name（该字段用于tls握手）
  - 需要将CMC工具所需的相关证书替换成，上文所生成的证书，然后进行操作。
- 通过长安链管理台进行部署/调用，详细教程请见[长安链管理台订阅外部链，以及部署合约章节](../dev/长安链管理台.md)
  - 需要注意，通过长安链管理台订阅上述创建的链时，需要确保链与管理台间网络通畅。
- 通过长安链SDK进行部署/调用，详情[SDK使用说明章节](../instructions/GoSDK使用说明.md)
  - 需要将SDK所需的相关证书替换成，上文所生成的证书，然后进行操作。


## 链管理

在长安链中，不同的账户一般绑定不同的角色，具有不同的权限。 为了提高安全性，长安链默认设置了许多权限，部分操作需要多个管理员多签才能完成。 不同组织的管理员证书账户生成，参考[生成admin测试账户](搭建cert模式账户体系.html#生成admin测试账户)
### 组织管理
组织管理包含共识节点组织管理和组织根证书管理, 以org5组织管理为例：
#### 新增一个组织到区块链网络
```shell
./cmc client chainconfig consensusnodeorg add \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--node-ids=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
--node-org-id=wx-org5.chainmaker.org
```
注：新增组织时，需要为组织至少指定一个共识节点

#### 将组织移除区块链网络
```shell
./cmc client chainconfig consensusnodeorg remove \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--node-org-id=wx-org5.chainmaker.org
```
注：组织被移除后，该组织下的**所有共识节点**会自动降级为**同步节点**，只能同步账本数据，不能参与共识。

#### 更新区块链网络到组织信息
```shell
./cmc client chainconfig consensusnodeorg update \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--node-ids=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4,QmaWrR72CbT51nFVpNDS8NaqUZjVuD4Ezf8xcHcFW9SJWF \
--node-org-id=wx-org5.chainmaker.org
```
注：可以通过更新组织的方式，可以批量更新共识节点，已有节点会被覆盖，请谨慎操作。单个共识节点的增删，请参考[节点管理](搭建cert模式账户体系.html#节点管理)

#### 组织根证书管理
- 申请组织根证书
```shell
# 申请组织org5根证书
$ curl --location --request POST 'http://localhost:8096/api/ca/gencert' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org5.chainmaker.org",
    "userId": "",
    "userType": "ca",
    "certUsage": "sign",
    "country": "CN",
    "locality": "BeiJing",
    "province": "BeiJing"
}' | jq
#response:
{
    "code": 200,
    "msg": "The request service returned successfully",
    "data": {
        "certSn": 4526784566558937862,
        "issueCertSn": 3682789430329867930,
        "cert": "-----BEGIN CERTIFICATE-----\nMIICUDCCAfagAwIBAgIIPtJelFB3UwYwCgYIKoZIzj0EAwIwYjELMAkGA1UEBhMC\nQ04xEDAOBgNVBAgTB0JlaWppbmcxEDAOBgNVBAcTB0JlaWppbmcxETAPBgNVBAoT\nCG9yZy1yb290MQ0wCwYDVQQLEwRyb290MQ0wCwYDVQQDEwRyb290MB4XDTIyMDgy\nNDA3MjMzMVoXDTIzMDIyMDA3MjMzMVowbDELMAkGA1UEBhMCQ04xEDAOBgNVBAgT\nB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxHzAdBgNVBAoTFnd4LW9yZzUuY2hh\naW5tYWtlci5vcmcxCzAJBgNVBAsTAmNhMQswCQYDVQQDEwJjYTBZMBMGByqGSM49\nAgEGCCqGSM49AwEHA0IABHQI7/AWTxbh9nU/x33I/WhU/6YxZm6nvASZQmAycx8q\nT2kQjLXUFYjl3i4Ku/B0X0IK7OP6NrqgWlDz82VPNoejgYswgYgwDgYDVR0PAQH/\nBAQDAgEGMA8GA1UdEwEB/wQFMAMBAf8wKQYDVR0OBCIEIKsQDmt6hb17p46/Mybw\nS+afVxm8Gk/SHdkglefR/XYIMCsGA1UdIwQkMCKAID2QQznj2uOhDBY0zQDZs0Ps\nixvqNqTx5WLTT7JErCJMMA0GA1UdEQQGMASCAIIAMAoGCCqGSM49BAMCA0gAMEUC\nIDIzqzySsueRMYzgOpvI1SkqzwPtLSgmt2yiEV6QwNOxAiEA2yhlEmr07t6YezwQ\nRBTkHdZEOZjygjUmieU77XpWgTI=\n-----END CERTIFICATE-----\n",
        "privateKey": "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIGsZnFxjdglimqodGaaQu+nLbFnzr0KO/gjomPwCaoyloAoGCCqGSM49\nAwEHoUQDQgAEdAjv8BZPFuH2dT/Hfcj9aFT/pjFmbqe8BJlCYDJzHypPaRCMtdQV\niOXeLgq78HRfQgrs4/o2uqBaUPPzZU82hw==\n-----END EC PRIVATE KEY-----\n"
    }
}

# 保存根证书和私钥
$ echo -n "-----BEGIN CERTIFICATE-----\nMIICUDCCAfagAwIBAgIIPtJelFB3UwYwCgYIKoZIzj0EAwIwYjELMAkGA1UEBhMC\nQ04xEDAOBgNVBAgTB0JlaWppbmcxEDAOBgNVBAcTB0JlaWppbmcxETAPBgNVBAoT\nCG9yZy1yb290MQ0wCwYDVQQLEwRyb290MQ0wCwYDVQQDEwRyb290MB4XDTIyMDgy\nNDA3MjMzMVoXDTIzMDIyMDA3MjMzMVowbDELMAkGA1UEBhMCQ04xEDAOBgNVBAgT\nB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxHzAdBgNVBAoTFnd4LW9yZzUuY2hh\naW5tYWtlci5vcmcxCzAJBgNVBAsTAmNhMQswCQYDVQQDEwJjYTBZMBMGByqGSM49\nAgEGCCqGSM49AwEHA0IABHQI7/AWTxbh9nU/x33I/WhU/6YxZm6nvASZQmAycx8q\nT2kQjLXUFYjl3i4Ku/B0X0IK7OP6NrqgWlDz82VPNoejgYswgYgwDgYDVR0PAQH/\nBAQDAgEGMA8GA1UdEwEB/wQFMAMBAf8wKQYDVR0OBCIEIKsQDmt6hb17p46/Mybw\nS+afVxm8Gk/SHdkglefR/XYIMCsGA1UdIwQkMCKAID2QQznj2uOhDBY0zQDZs0Ps\nixvqNqTx5WLTT7JErCJMMA0GA1UdEQQGMASCAIIAMAoGCCqGSM49BAMCA0gAMEUC\nIDIzqzySsueRMYzgOpvI1SkqzwPtLSgmt2yiEV6QwNOxAiEA2yhlEmr07t6YezwQ\nRBTkHdZEOZjygjUmieU77XpWgTI=\n-----END CERTIFICATE-----\n" > ca.crt
$ echo -n "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIGsZnFxjdglimqodGaaQu+nLbFnzr0KO/gjomPwCaoyloAoGCCqGSM49\nAwEHoUQDQgAEdAjv8BZPFuH2dT/Hfcj9aFT/pjFmbqe8BJlCYDJzHypPaRCMtdQV\niOXeLgq78HRfQgrs4/o2uqBaUPPzZU82hw==\n-----END EC PRIVATE KEY-----\n" > ca.key

# 查看org5组织根证书
$ openssl x509 -in ca.crt -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4526784566558937862 (0x3ed25e9450775306)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: C = CN, ST = Beijing, L = Beijing, O = org-root, OU = root, CN = root
        Validity
            Not Before: Aug 24 07:23:31 2022 GMT
            Not After : Feb 20 07:23:31 2023 GMT
        Subject: C = CN, ST = BeiJing, L = BeiJing, O = wx-org5.chainmaker.org, OU = ca, CN = ca
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:74:08:ef:f0:16:4f:16:e1:f6:75:3f:c7:7d:c8:
                    fd:68:54:ff:a6:31:66:6e:a7:bc:04:99:42:60:32:
                    73:1f:2a:4f:69:10:8c:b5:d4:15:88:e5:de:2e:0a:
                    bb:f0:74:5f:42:0a:ec:e3:fa:36:ba:a0:5a:50:f3:
                    f3:65:4f:36:87
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier:
                AB:10:0E:6B:7A:85:BD:7B:A7:8E:BF:33:26:F0:4B:E6:9F:57:19:BC:1A:4F:D2:1D:D9:20:95:E7:D1:FD:76:08
            X509v3 Authority Key Identifier:
                keyid:3D:90:43:39:E3:DA:E3:A1:0C:16:34:CD:00:D9:B3:43:EC:8B:1B:EA:36:A4:F1:E5:62:D3:4F:B2:44:AC:22:4C

            X509v3 Subject Alternative Name:
                DNS:, DNS:
    Signature Algorithm: ecdsa-with-SHA256
         30:45:02:20:32:33:ab:3c:92:b2:e7:91:31:8c:e0:3a:9b:c8:
         d5:29:2a:cf:03:ed:2d:28:26:b7:6c:a2:11:5e:90:c0:d3:b1:
         02:21:00:db:28:65:12:6a:f4:ee:de:98:7b:3c:10:44:14:e4:
         1d:d6:44:39:98:f2:82:35:26:89:e5:3b:ed:7a:56:81:32
```
从证书Subject字段内容上我们可以看出，该证书代表组织`wx-org5.chainmaker.org`, 证书类型为`CA`，表示为根证书

- 增加组织根证书
```shell
./cmc client chainconfig trustroot add \
   --sdk-conf-path=./testdata/sdk_config.yml \
   --org-id=wx-org1.chainmaker.org \
   --user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
   --user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key \
   --user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.crt \
   --user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.key \
   --admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
   --admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
   --trust-root-org-id=wx-org5.chainmaker.org \  # 组织id
   --trust-root-path=./testdata/crypto-config/wx-org5.chainmaker.org/ca/ca.crt # 组织根证书
```

- 删除组织根证书  
```shell
./cmc client chainconfig trustroot remove \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--trust-root-org-id=wx-org5.chainmaker.org #组织id
```

- 更新组织根证书  
```shell
./cmc client chainconfig trustroot update \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org5.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org5.chainmaker.org/user/admin1/admin1.tls.key \
--trust-root-org-id=wx-org5.chainmaker.org \ #组织id
--trust-root-path=./testdata/crypto-config/wx-org5.chainmaker.org/ca/ca.crt # 组织根证书
```

#### 组织信息查询
组织信息可以通过链配置查询，在`trust_roots`字段下会返回当前区块链网络中的组织列表以及各个组织下的组织根证书，同时
在`consensus`下，会返回各个组织下的共识节点列表，命令如下：
```shell
./cmc client chainconfig query \
--sdk-conf-path=./testdata/sdk_config.yml
```

### 节点管理
以组织org5增加共识节点为例, 组织的节点证书生成，参考[生成节点证书](搭建cert模式账户体系.html#生成节点证书)
#### 共识节点管理
- 增加共识节点
```shell
./cmc client chainconfig consensusnodeid add \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--node-id=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \ #这里为新添加节点的nodeId
--node-org-id=wx-org5.chainmaker.org
```

- 更新共识节点Id
```shell
./cmc client chainconfig consensusnodeid update \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--node-id=QmXxeLkNTcvySPKMkv3FUqQgpVZ3t85KMo5E4cmcmrexrC \
--node-id-old=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
--node-org-id=wx-org5.chainmaker.org
```

- 删除共识节点
```shell
./cmc client chainconfig consensusnodeid remove \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--node-id=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
--node-org-id=wx-org5.chainmaker.org
```
- 查询共识节点  
共识节点可以通过链配置查询，在`consensus`字段下会返回当前区块链网络中的共识节点列表，命令如下：
```shell
./cmc client chainconfig query \
--sdk-conf-path=./testdata/sdk_config.yml

#...
  "consensus": {
    "nodes": [
      {
        "node_id": [
          "QmSx7PhNBPtSN9R8t5DRgKM2iTZNbpyKtBXWGMH6V2CK53"
        ],
        "org_id": "wx-org1.chainmaker.org"
      },
      {
        "node_id": [
          "QmVviBSVY4xK2161hFWh2v4Wh5ThGAgBiPtXV8XjzVbzPW"
        ],
        "org_id": "wx-org2.chainmaker.org"
      },
      {
        "node_id": [
          "QmbBhed1jeFMkFazYnvVJiqp9RAxnnjA6wxiPVkgAdbeDT"
        ],
        "org_id": "wx-org3.chainmaker.org"
      },
      {
        "node_id": [
          "QmdyLwr6ahCQeSixfw72E17rqn8s4vLewzJgYR64eEeQvD"
        ],
        "org_id": "wx-org4.chainmaker.org"
      }
    ],
    "type": 1
  },
```


#### 同步节点管理
- 添加同步节点  
通过CA服务生成同步节点的账户与共识节点类似，请参考[生成节点证书](搭建cert模式账户体系.html#生成节点证书)，主要不同点在证书生成过程中`userType`字段
使用`common`  
通过CA服务生成同步节点sign和tls证书如下（示例）：
```shell
# 生成同步节点sign证书
$ curl --location --request POST 'http://localhost:8096/api/ca/gencert' \
--header 'Content-Type: application/json' \
--data-raw '{
"orgId": "wx-org1.chainmaker.org",
"userId": "org1.common1.com",
"userType": "common",
"certUsage": "sign",
"country": "CN",
"locality": "BeiJing",
"province": "BeiJing"
}'

# 生成同步节点tls证书
$ curl --location --request POST 'http://localhost:8096/api/ca/gencert' \
--header 'Content-Type: application/json' \
--data-raw '{
"orgId": "wx-org1.chainmaker.org",
"userId": "org1.common1.com",
"userType": "common",
"certUsage": "tls",
"country": "CN",
"locality": "BeiJing",
"province": "BeiJing"
}'
```
使用上面生成的同步节点sign私钥/证书和tls私钥/证书启动同步节点后，启动方式请参考[基于生成的组织和节点证书创建链](搭建cert模式账户体系.html#基于生成的组织和节点证书创建链)。
同步节点启动后，会自动同步区块链账本数据；同时同步节点可以通过[节点管理-增加共识节点](搭建cert模式账户体系.html#节点管理)的方式切换为共识节点，并参与共识。

- 更新同步节点  
  暂不支持同步节点更新相关操作

- 删除同步节点  
  同步节点的删除可以通过删除同步节点组织根证书（注：会影响组织下的所有同步节点和共识节点，请谨慎操作）； 或者通过[证书管理-冻结证书](搭建cert模式账户体系.html#证书的管理)方式冻结同步节点证书。
  同步节点删除后，将不能够继续同步账本数据


### 用户管理
#### 管理员用户管理
- 新增管理员  
 新增管理员方式，请参考 [测试新创建的链是否正常](搭建cert模式账户体系.html#测试新创建的链是否正常)章节中管理员账户的生成， 管理员证书生成后，
 管理员可以使用CA颁发的admin证书直接访问链上信息。

- 删除用户证书

暂不支持直接删除链上管理员账户，可以通过**证书管理功能**来冻结或吊销链上证书的方式，进而限制用户对链的访问，详情请参考[证书管理](搭建cert模式账户体系.html#证书管理)

- 查询管理员列表  
可以通过CA服务提供的证书列表查询方式，间接查询管理员列表。 以查询组织org5下查询管理员为例，`userType`需要指定为`admin`, 如下：

```shell
curl --location --request POST 'http://localhost:8096/api/ca/querycerts' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org5.chainmaker.org",
    "userType": "admin",
    "certUsage": "sign"
}

{
    "code": 200,
    "msg": "The request service returned successfully",
    "data": [
        {
            "userId": "admin1",
            "orgId": "wx-org5.chainmaker.org",
            "userType": "admin",
            "certUsage": "sign",
            "certSn": 476309406125711642,
            "issuerSn": 4526784566558937862,
            "certContent": "-----BEGIN CERTIFICATE-----\nMIICTTCCAfSgAwIBAgIIBpww4ZtM4RowCgYIKoZIzj0EAwIwbDELMAkGA1UEBhMC\nQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxHzAdBgNVBAoT\nFnd4LW9yZzUuY2hhaW5tYWtlci5vcmcxCzAJBgNVBAsTAmNhMQswCQYDVQQDEwJj\nYTAeFw0yMjA4MzAwMjE4MjhaFw0yMzAyMjYwMjE4MjhaMHMxCzAJBgNVBAYTAkNO\nMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYDVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3\neC1vcmc1LmNoYWlubWFrZXIub3JnMQ4wDAYDVQQLEwVhZG1pbjEPMA0GA1UEAxMG\nYWRtaW4xMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEG+e33zUe7tYz10yVXrTK\n+Vk9cHPRjz65J2q1fWitFdRp63vYlF2hGOGPwohHXMQ6mofNiR9UZqyFGOAjuZKF\n7KN5MHcwDgYDVR0PAQH/BAQDAgbAMCkGA1UdDgQiBCDbNi9Kuq2YskAOwJDIfTFa\n6xdsaD8CV366t1vEtEKWODArBgNVHSMEJDAigCCrEA5reoW9e6eOvzMm8Evmn1cZ\nvBpP0h3ZIJXn0f12CDANBgNVHREEBjAEggCCADAKBggqhkjOPQQDAgNHADBEAiA9\nVoR9iRV1rYwbNRjewMs/NZPrQp6l6gZsAOKch/W0uAIgSuRdqYPikrC+4WgX5HQg\nM6nwJPUxSq2FTUd6SvVtJUY=\n-----END CERTIFICATE-----\n",
            "expirationDate": 1677377908,
            "isRevoked": false
        },
        {
            "userId": "admin2",
            "orgId": "wx-org5.chainmaker.org",
            "userType": "admin",
            "certUsage": "sign",
            "certSn": 2655863384365393409,
            "issuerSn": 4526784566558937862,
            "certContent": "-----BEGIN CERTIFICATE-----\nMIICTzCCAfSgAwIBAgIIJNuFrT0Q5gEwCgYIKoZIzj0EAwIwbDELMAkGA1UEBhMC\nQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxHzAdBgNVBAoT\nFnd4LW9yZzUuY2hhaW5tYWtlci5vcmcxCzAJBgNVBAsTAmNhMQswCQYDVQQDEwJj\nYTAeFw0yMjA4MzAwMjE4MzFaFw0yMzAyMjYwMjE4MzFaMHMxCzAJBgNVBAYTAkNO\nMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYDVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3\neC1vcmc1LmNoYWlubWFrZXIub3JnMQ4wDAYDVQQLEwVhZG1pbjEPMA0GA1UEAxMG\nYWRtaW4yMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEiS86NLQero79bxS3Lse7\n46Rp+oZndblFqPGRkuiztcFwEuUyVYwVk/syEHU6yTIY8kaJelhHrq2/KJ1eNp0Q\npaN5MHcwDgYDVR0PAQH/BAQDAgbAMCkGA1UdDgQiBCDA1fzr67GDcwuiaOXzUzuW\np+8ux6DrgAfy5UHzIZyM5jArBgNVHSMEJDAigCCrEA5reoW9e6eOvzMm8Evmn1cZ\nvBpP0h3ZIJXn0f12CDANBgNVHREEBjAEggCCADAKBggqhkjOPQQDAgNJADBGAiEA\nlX9uFrgJlUHk8DwsPzhaZ3E9xOa/H8SugruS153qG3YCIQDUBpMv7HD9H3wVKfU1\nOVpBq2bCdHFSO5yPrYIBy3zF2A==\n-----END CERTIFICATE-----\n",
            "expirationDate": 1677377911,
            "isRevoked": false
        }
    ]
}
````
从查询结果看，组织org5下有两个管理员admin1和admin2

#### 普通用户管理
- 生成用户证书（client）
```shell
$ curl --location --request POST 'http://localhost:8096/api/ca/gencert' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org1.chainmaker.org",
    "userId": "client1",
    "userType": "client",
    "certUsage": "sign",
    "country": "CN",
    "locality": "BeiJing",
    "province": "BeiJing"
}' | jq

{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": {
    "certSn": 1877624678000075300,
    "issueCertSn": 1696861425238169000,
    "cert": "-----BEGIN CERTIFICATE-----\nMIICaDCCAg6gAwIBAgIIGg6pslGzoaYwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNzA2NThaFw0yMzAy\nMjAwNzA2NThaMHUxCzAJBgNVBAYTAkNOMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYD\nVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQ8w\nDQYDVQQLEwZjbGllbnQxEDAOBgNVBAMTB2NsaWVudDEwWTATBgcqhkjOPQIBBggq\nhkjOPQMBBwNCAASzGFb7efLCJcHbk1SQ9iYWr5gH70O/v5j53mAi6OdYLqgiNzaE\nSudb343MWUPMPKDJcFv6WWvgvuwKulPwt4ljo3kwdzAOBgNVHQ8BAf8EBAMCBsAw\nKQYDVR0OBCIEIPlN3wdwbv+AmN3BPybsylivb4c/UgTeZ//dFTnoejuhMCsGA1Ud\nIwQkMCKAIFQUtaMGHEYWLDGcfWBR0SQZjuWdI31bW7tCAyrplWpuMA0GA1UdEQQG\nMASCAIIAMAoGCCqGSM49BAMCA0gAMEUCIQCdMobpCCJX4hx3f8uNpAsK8EUrmzgU\no1Mup+v9WbpLwAIgFT/Tf4nlhchXMFIZ4W4t9s6Z2NHDDdt5fz+i2CyVOQ0=\n-----END CERTIFICATE-----\n",
    "privateKey": "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIAkp0yFG1amB2SSuhdFMslVGOj8BH5eymvvdZOEdEivXoAoGCCqGSM49\nAwEHoUQDQgAEsxhW+3nywiXB25NUkPYmFq+YB+9Dv7+Y+d5gIujnWC6oIjc2hErn\nW9+NzFlDzDygyXBb+llr4L7sCrpT8LeJYw==\n-----END EC PRIVATE KEY-----\n"
  }
}

# 保存用户sign证书和私钥
$ echo -n "-----BEGIN CERTIFICATE-----\nMIICfDCCAiOgAwIBAgIINKV9XwIOzHEwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYT\nAkNOMRAwDgYDVQQIEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQK\nExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZ\nY2Etd3gtb3JnMS5jaGFpbm1ha2VyLm9yZzAeFw0yMjA4MjQwNTUzMTRaFw0yMzAy\nMjAwNTUzMTRaMHMxCzAJBgNVBAYTAkNOMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYD\nVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNoYWlubWFrZXIub3JnMQ4w\nDAYDVQQLEwVhZG1pbjEPMA0GA1UEAxMGYWRtaW4xMFkwEwYHKoZIzj0CAQYIKoZI\nzj0DAQcDQgAE4lJgyz1A/GledacE3HUCwT2MrWHqDzHHpy81RTY22Ap6uj/ahO1h\n1+o0GSGS8xdF0ygw07BhtBOe9dkuYvptQ6OBjzCBjDAOBgNVHQ8BAf8EBAMCA/gw\nEwYDVR0lBAwwCgYIKwYBBQUHAwIwKQYDVR0OBCIEIJ9OT40+LKdBATiWmfeIXmXO\n2RkxhbG1+Ai5vt+l5LxsMCsGA1UdIwQkMCKAIFQUtaMGHEYWLDGcfWBR0SQZjuWd\nI31bW7tCAyrplWpuMA0GA1UdEQQGMASCAIIAMAoGCCqGSM49BAMCA0cAMEQCIDbU\nvsm/C2OPw5HbhZCBzZJNbJ5x3QW+I8kOKNZd4X7jAiAq12qTcqZcNSBKsqM9HRTj\nqt/2rzh38uEgu/2oUvpIUw==\n-----END CERTIFICATE-----\n" > admin1.tls.crt
$ echo -n "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIIA9k6+tvHrCa0Ee3P+IlGO0AQFW3GR6VSYtD33lK+PDoAoGCCqGSM49\nAwEHoUQDQgAE4lJgyz1A/GledacE3HUCwT2MrWHqDzHHpy81RTY22Ap6uj/ahO1h\n1+o0GSGS8xdF0ygw07BhtBOe9dkuYvptQw==\n-----END EC PRIVATE KEY-----\n" > admin1.tls.key
```

client用户tls证书和私钥的生成、测试与[测试新创建的链是否正常](搭建cert模式账户体系.html#测试新创建的链是否正常)章节中管理员账户的生成和测试类似，这里不在赘述。主要区别：
生成证书接口参数userType为client；client账户链上权限与admin相比权限较低。权限相关介绍参考[权限管理](搭建cert模式账户体系.html#权限管理)

- 删除用户证书

暂不支持直接删除链上账户，可以通过**证书管理功能**来冻结或吊销证书的方式，进而限制用户对链的访问，详情请参考[证书管理](搭建cert模式账户体系.html#证书管理)

- 查询用户证书  
可以通过CA服务提供的证书列表查询方式，间接查询普通用户列表。 以查询组织org5下查询普通账户为例，`userType`需要指定为`client`，如下：

```shell
$ curl --location --request POST 'http://localhost:8096/api/ca/querycerts' \
--header 'Content-Type: application/json' \
--data-raw '{
    "orgId": "wx-org5.chainmaker.org",
    "userType": "client",
    "certUsage": "sign"
}

{
    "code": 200,
    "msg": "The request service returned successfully",
    "data": [
        {
            "userId": "client1",
            "orgId": "wx-org5.chainmaker.org",
            "userType": "client",
            "certUsage": "sign",
            "certSn": 3649421597944817067,
            "issuerSn": 4526784566558937862,
            "certContent": "-----BEGIN CERTIFICATE-----\nMIICUDCCAfagAwIBAgIIMqVZm50jpaswCgYIKoZIzj0EAwIwbDELMAkGA1UEBhMC\nQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxHzAdBgNVBAoT\nFnd4LW9yZzUuY2hhaW5tYWtlci5vcmcxCzAJBgNVBAsTAmNhMQswCQYDVQQDEwJj\nYTAeFw0yMjA4MzAwMjI0MjBaFw0yMzAyMjYwMjI0MjBaMHUxCzAJBgNVBAYTAkNO\nMRAwDgYDVQQIEwdCZWlKaW5nMRAwDgYDVQQHEwdCZWlKaW5nMR8wHQYDVQQKExZ3\neC1vcmc1LmNoYWlubWFrZXIub3JnMQ8wDQYDVQQLEwZjbGllbnQxEDAOBgNVBAMT\nB2NsaWVudDEwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAAT8FzELiNwtT4xavmP6\nS3iTNYAKtMIXbEKDglELwGImTz01l3Lc8tZsos550VwKgjmLPAvBgMyMCIrsbIT2\n8bZMo3kwdzAOBgNVHQ8BAf8EBAMCBsAwKQYDVR0OBCIEIK6x0q+mj9DLfYT7uIxl\nowVLNx6xiuPDvG3JGe4jjEzIMCsGA1UdIwQkMCKAIKsQDmt6hb17p46/MybwS+af\nVxm8Gk/SHdkglefR/XYIMA0GA1UdEQQGMASCAIIAMAoGCCqGSM49BAMCA0gAMEUC\nIQCiAU3LEGJcr1pbdlpn+dENvzkqwuwq0BI5PeWfKn/07wIgKGpqzzJLb8qf8IWp\nJPEjn7a1TMWgAPIa6jktnRMf4gc=\n-----END CERTIFICATE-----\n",
            "expirationDate": 1677378260,
            "isRevoked": false
        }
    ]
}
```
从查询结果看，组织org5下有一个普通用户client1


### 权限管理
长安链目前支持资源级别的权限管理，可以通过cmc命令行工具或者sdk来查询、新增、修改以及删除资源权限。  
- 权限定义介绍请参考[权限定义](长安链账户整体介绍.html#权限定义)  
- 资源定义介绍请参考[资源定义](长安链账户整体介绍.html#资源定义)

长安链证书模式下账户权限与绑定的角色相关，资源的访问权限也与角色相关。 一般情况下修改账户权限需要对资源权限进行修改。

注：长安链支持资源级别的细粒度修改，暂不支持账户级别的细粒度修改。


- 权限列表查询
```shell
  ./cmc client chainconfig permission list \
  --sdk-conf-path=./testdata/sdk_config.yml
```

证书模式下默认权限列表，请参考[证书模式权限定义](../tech/身份权限管理.html#PermissionedWithCert)。

- 设置账户权限   
权限修改相关的操作一般需要**多数管理员多签**授权，假如我们有个资源名叫：TEST_SUM, 需要设置为"任一用户可以访问"， 使用cmc命令设置权限权限如下：
```shell
  ./cmc client chainconfig permission add \
  --sdk-conf-path=./testdata/sdk_config.yml \
  --admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
  --admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
  --admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
  --sync-result=true \
  --permission-resource-name="TEST_SUM" \
  --permission-resource-policy-rule=ANY \
  --permission-resource-policy-roleList=CLIENT
```
- 修改账户权限   
使用cmc命令修改TEST_SUM资源权限为多数管理员多签操作
```shell
./cmc client chainconfig permission update \
  --sdk-conf-path=./testdata/sdk_config.yml \
  --admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
  --admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
  --admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
  --sync-result=true \
  --permission-resource-name="TEST_SUM" \
  --permission-resource-policy-rule=MAJORITY \
  --permission-resource-policy-roleList=ADMIN
```

- 删除账户权限    
使用cmc命令删除TEST_SUM资源的权限限制，删除后节点权限校验模块不会对该资源进行权限检查  
```shell
./cmc client chainconfig permission delete \
  --sdk-conf-path=./testdata/sdk_config.yml \
  --admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
  --admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
  --admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
  --sync-result=true \
  --permission-resource-name="TEST_SUM"
```

注：除了自定义资源的权限设置外，长安链也支持默认权限的修改，比如升级合约需要**MAJORITY多签**操作，可以通过`修改账户权限`改为**ANY单签**权限。此外，对于默认权限虽然可以进行修改，但是不能删除，删除操作仅删除到系统默认权限。

### 证书的管理
#### 冻结/解冻证书
长安链证书模式下，支持证书的冻结和解冻操作，在节点或者用户账户不再安全的情况下，比如说密钥丢失或被盗用，可以通过冻结证书的方式禁止该账户继续发起交易。
- 冻结证书  
```shell
$ ./cmc client certmanage freeze  \
--sdk-conf-path=./testdata/sdk_config.yml \
--cert-file-paths=./client1.sign.crt,./client2.sign.crt \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key
```
通过参数`cert-file-paths`指定要冻结的证书文件路径，如果需要冻结多个证书账户，证书文件名之间用`，`隔开。 

- 解冻证书  
证书冻结后，需要通过证书解冻才能继续发起交易，证书解冻方式如下:
```shell
$ ./cmc client certmanage unfreeze  \
--sdk-conf-path=./testdata/sdk_config.yml \
--cert-file-paths=./client1.sign.crt,./client2.sign.crt \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key
```
通过参数`cert-file-paths`指定要解冻的证书文件路径，如果需要解冻多个证书账户，证书文件名之间用`，`隔开。

注：根证书不支持冻结和解冻操作

#### 撤销证书
当用户账户不安全时，除了通过证书冻结方式保护链上信息，还可以通过撤销证书的方式。 与证书冻结的方式不同，证书一旦撤销不可恢复，请谨慎操作。证书撤销需要用到`crl证书撤销列表`，
crl一般来自CA服务。
- 生成证书撤销列表crl
```shell
# 撤销证书，用证书sn指定
curl --location --request POST 'http://localhost:8096/api/ca/revokecert' \
--header 'Content-Type: application/json' \
--data-raw '{
    "revokedCertSn": 1877624678000075174,
    "issuerCertSn": 1696861425238169023,
    "reason":"密钥丢失，无法找回"
}' | jq

{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": "-----BEGIN X509 CRL-----\nMIIBXDCCAQECAQEwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYTAkNOMRAwDgYDVQQI\nEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNo\nYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZY2Etd3gtb3JnMS5j\naGFpbm1ha2VyLm9yZxcNMjIwODI1MDcxNjA5WhcNMjIwODI2MDcxNjA5WjAbMBkC\nCBoOqbJRs6GmFw0yMzAyMjAwNzA2NThaoC8wLTArBgNVHSMEJDAigCBUFLWjBhxG\nFiwxnH1gUdEkGY7lnSN9W1u7QgMq6ZVqbjAKBggqhkjOPQQDAgNJADBGAiEA+y1U\ntVHVH1vW1L/N9lGSz5RVXs89irzN44RFoXP0VwACIQDrllN4otO4qtPidEBEJZKv\ngkWAUWBVxn91TCM2OxeIKg==\n-----END X509 CRL-----\n"
}

# 将返回的data保存为crl文件
echo -n "-----BEGIN X509 CRL-----\nMIIBWzCCAQECAQEwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYTAkNOMRAwDgYDVQQI\nEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNo\nYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZY2Etd3gtb3JnMS5j\naGFpbm1ha2VyLm9yZxcNMjIwODI1MDcxOTEwWhcNMjIwODI2MDcxOTEwWjAbMBkC\nCBoOqbJRs6GmFw0yMzAyMjAwNzA2NThaoC8wLTArBgNVHSMEJDAigCBUFLWjBhxG\nFiwxnH1gUdEkGY7lnSN9W1u7QgMq6ZVqbjAKBggqhkjOPQQDAgNIADBFAiBg6Mdn\nNa2aXRPDJC6/ukDWQnnDGtrDTp09oVVaGV0b4wIhALtTXy9AAlsn7UL6HMP+UHk4\nlH2sGzq3lsYslqhHKXwG\n-----END X509 CRL-----\n" > ./client1.crl


# 如果要查询某CA下的所有撤销证书，可以通过以下方式查询。
 curl --location --request POST 'http://localhost:8096/api/ca/gencrl' \
--header 'Content-Type: application/json' \
--data-raw '{
    "issuerCertSn":1696861425238169023,
    "token":""
}' | jq

{
  "code": 200,
  "msg": "The request service returned successfully",
  "data": "-----BEGIN X509 CRL-----\nMIIBWzCCAQECAQEwCgYIKoZIzj0EAwIwgYMxCzAJBgNVBAYTAkNOMRAwDgYDVQQI\nEwdCZWlqaW5nMRAwDgYDVQQHEwdCZWlqaW5nMR8wHQYDVQQKExZ3eC1vcmcxLmNo\nYWlubWFrZXIub3JnMQswCQYDVQQLEwJjYTEiMCAGA1UEAxMZY2Etd3gtb3JnMS5j\naGFpbm1ha2VyLm9yZxcNMjIwODI1MDcxOTEwWhcNMjIwODI2MDcxOTEwWjAbMBkC\nCBoOqbJRs6GmFw0yMzAyMjAwNzA2NThaoC8wLTArBgNVHSMEJDAigCBUFLWjBhxG\nFiwxnH1gUdEkGY7lnSN9W1u7QgMq6ZVqbjAKBggqhkjOPQQDAgNIADBFAiBg6Mdn\nNa2aXRPDJC6/ukDWQnnDGtrDTp09oVVaGV0b4wIhALtTXy9AAlsn7UL6HMP+UHk4\nlH2sGzq3lsYslqhHKXwG\n-----END X509 CRL-----\n"
}
```

- 链上证书撤销  
```shell
 ./cmc client certmanage revoke \
--sdk-conf-path=./testdata/sdk_config.yml \
--cert-crl-path=./client1.crl \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key
```
通过参数`cert-crl-path`指定crl，并发起交易，交易执行成功后，该crl中包含的证书将不能够再发起交易或访问链上信息。