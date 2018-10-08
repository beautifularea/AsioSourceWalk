从这章开始ssl相关的代码分析.

相关文件  
[verify_mode](asio\ssl\verify_mode.hpp)  

ssl中可以设置verify的模式，有下面几种  
```
typedef int verify_mode;

* @li verify_none 不验证证书
* @li verify_peer 验证peer节点的证书
* @li verify_fail_if_no_peer_cert peer节点没有证书即验证失败
* @li verify_client_once  验证client端一次
```
定义如下
```
const int verify_none = SSL_VERIFY_NONE;
const int verify_peer = SSL_VERIFY_PEER;
const int verify_fail_if_no_peer_cert = SSL_VERIFY_FAIL_IF_NO_PEER_CERT;
const int verify_client_once = SSL_VERIFY_CLIENT_ONCE;
```

使用如下：
```
stream_.set_verify_mode (boost::asio::ssl::verify_peer);
```
