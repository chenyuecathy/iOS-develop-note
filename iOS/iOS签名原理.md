
#iOS App签名原理

关于iOS app的签名机制，实质上就是RSA和DES加密。下面我们来看一张图，这张图来源于[WeRead团队的博客](https://wereadteam.github.io/2017/03/13/Signature/)，我重新绘制了这张图，并在博客图片的基础上细化了一些内容。

![avatar](https://raw.githubusercontent.com/chenyuecathy/iOS-develop-note/master/pic/iOS%E7%AD%BE%E5%90%8D%E5%8E%9F%E7%90%86.png)


上面的步骤对应到我们平常具体的操作和概念是这样的：
<br/><br/>1. 第 1 步对应的是 keychain 里的 “从证书颁发机构请求证书”，这里就本地生成了一堆公私钥，保存的 CertificateSigningRequest 就是公钥，私钥保存在本地电脑里。
<br/><br/>2. 第 2 步苹果处理，不用管。
<br/><br/>3. 第 3 步对应把 CertificateSigningRequest 传到苹果后台生成证书，并下载到本地。这时本地有两个证书，一个是第 1 步生成的，一个是这里下载回来的，keychain 会把这两个证书关联起来，因为他们公私钥是对应的，在XCode选择下载回来的证书时，实际上会找到 keychain 里对应的私钥去签名。这里私钥只有生成它的这台 Mac 有，如果别的 Mac 也要编译签名这个 App 怎么办？答案是把私钥导出给其他 Mac 用，在 keychain 里导出私钥，就会存成 .p12 文件，其他 Mac 打开后就导入了这个私钥。
<br/><br/>4. 第 4 步都是在苹果网站上操作，配置 AppID / 权限 / 设备等，最后下载 Provisioning Profile 文件。
<br/><br/>5. 第 5 步 XCode 会通过第 3 步下载回来的证书（存着公钥），在本地找到对应的私钥（第一步生成的），用本地私钥去签名 App，并把 Provisioning Profile 文件命名为 embedded.mobileprovision 一起打包进去。这里对 App 的签名数据保存分两部分，Mach-O 可执行文件会把签名直接写入这个文件里，其他资源文件则会保存在 _CodeSignature 目录下。
<br/><br/>第 6 - 7 步的打包和验证都是 Xcode 和 iOS 系统自动做的事。

<br/>这里再总结一下这些概念：
<br/>1. 证书：内容是公钥或私钥，由其他机构对其签名组成的数据包。
<br/>2. Entitlements：包含了 App 权限开关列表。
<br/>3. CertificateSigningRequest：本地公钥。
<br/>4. p12：本地私钥，可以导入到其他电脑。
<br/>5. Provisioning Profile：包含了 证书 / Entitlements 等数据，并由苹果后台私钥签名的数据包。

#加密和解密（非对称RSA）
<http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html>
<http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html>  
<br/>有了公钥和密钥，就能进行加密和解密了。
<br/>**（1）加密要用公钥 (n,e)**  
假设鲍勃要向爱丽丝发送加密信息m，他就要用爱丽丝的公钥 (n,e) 对m进行加密。这里需要注意，m必须是整数（字符串可以取ascii值或unicode值），且m必须小于n。    
    所谓"加密"，就是算出下式的c：  
    me ≡ c (mod n)  
　　爱丽丝的公钥是 (3233, 17)，鲍勃的m假设是65，那么可以算出下面的等式：  
　　6517 ≡ 2790 (mod 3233)  
于是，c等于2790，鲍勃就把2790发给了爱丽丝。  

**（2）解密要用私钥(n,d)**  
爱丽丝拿到鲍勃发来的2790以后，就用自己的私钥(3233, 2753) 进行解密。可以证明，下面的等式一定成立：  
　　cd ≡ m (mod n)  
也就是说，c的d次方除以n的余数为m。现在，c等于2790，私钥是(3233, 2753)，那么，爱丽丝算出  
　　27902753 ≡ 65 (mod 3233)  
因此，爱丽丝知道了鲍勃加密前的原文就是65。  
至此，"加密--解密"的整个过程全部完成。  
我们可以看到，如果不知道d，就没有办法从c求出m。而前面已经说过，要知道d就必须分解n，这是极难做到的，所以RSA算法保证了通信安全。  
你可能会问，公钥(n,e) 只能加密小于n的整数m，那么如果要加密大于n的整数，该怎么办？有两种解决方法：一种是把长信息分割成若干段短消息，每段分别加密；另一种是先选择一种"对称性加密算法"（比如DES），用这种算法的密钥加密信息，再用RSA公钥加密DES密钥。
