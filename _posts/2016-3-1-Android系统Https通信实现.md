---
date: 2016-3-1  UTC
title: Android系统Https通信实现
description: HTTPS，是以安全为目标的HTTP通道，简单讲是HTTP的安全版。即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL...
permalink: /posts/learn/
key: 10004
labels: [Android]
---
#Android系统Https通信实现
***
### 一、Https原理
> HTTPS（全称：Hyper Text Transfer Protocol over Secure Socket Layer），是以安全为目标的HTTP通道，简单讲是HTTP的安全版。即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。 它是一个URI scheme（抽象标识符体系），句法类同http:体系。用于安全的HTTP数据传输。https:URL表明它使用了HTTP，但HTTPS存在不同于HTTP的默认端口及一个加密/身份验证层（在HTTP与TCP之间）。这个系统的最初研发由网景公司(Netscape)进行，并内置于其浏览器Netscape Navigator中，提供了身份验证与加密通讯方法。现在它被广泛用于万维网上安全敏感的通讯，例如交易支付方面。

### 二、SSL协议作用
> SSL协议位于TCP/IP协议与各种应用层协议之间，为数据通讯提供安全支持。SSL协议可分为两层： SSL记录协议（SSL Record Protocol）：它建立在可靠的传输协议（如TCP）之上，为高层协议提供数据封装、压缩、加密等基本功能的支持。 SSL握手协议（SSL Handshake Protocol）：它建立在SSL记录协议之上，用于在实际的数据传输开始前，通讯双方进行身份认证、协商加密算法、交换加密密钥等。

### 三、Android实现HTTPS通信
    protected HttpClient getHttpsClient() throws TaskException {
		BasicHttpParams httpParameters = new BasicHttpParams();
		HttpConnectionParams.setConnectionTimeout(httpParameters, connectTimeout);
		HttpConnectionParams.setSoTimeout(httpParameters, soTimeout);

		HttpClient client = null;
		try {
			KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
			keyStore.load(null, null);

			// 如果不设置这里，会报no peer certificate错误
			SSLSocketFactory sf = new MSSLSocketFactory(keyStore);
			sf.setHostnameVerifier(SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);

			SchemeRegistry schemeRegistry = new SchemeRegistry();
			schemeRegistry.register(new Scheme("https", sf, 443));
			schemeRegistry.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
			HttpParams params = new BasicHttpParams();
			HttpProtocolParams.setVersion(params, HttpVersion.HTTP_1_1);
			HttpProtocolParams.setContentCharset(params, HTTP.UTF_8);
			SingleClientConnManager clientManager = new SingleClientConnManager(params, schemeRegistry);
			client = new DefaultHttpClient(clientManager, httpParameters);
		} catch (Exception e) {
			e.printStackTrace();
			throw new TaskException(TaskException.TaskError.timeout.toString());
		}
    }    

加入对HTTPS的支持，就可以有效的建立HTTPS连接了。但是访问自己搭建的HTTPS服务器却不行，因为它使用了不被系统承认的自定义证书，会报出如下问题：No peer certificate。

#### 使用自定义证书并忽略验证的HTTPS连接方式
解决证书不被系统承认的方法，就是跳过系统校验。要跳过系统校验，就不能再使用系统标准的SSL SocketFactory了，需要自定义一个。然后为了在这个自定义SSL SocketFactory里跳过校验，还需要自定义一个TrustManager，在其中忽略所有校验，即TrustAll。
    public class MSSLSocketFactory extends SSLSocketFactory {
	
	SSLContext sslContext = SSLContext.getInstance("TLS");

	public MSSLSocketFactory(KeyStore truststore)
			throws NoSuchAlgorithmException, KeyManagementException,
			KeyStoreException, UnrecoverableKeyException {

		    super(truststore);

		    TrustManager tm = new X509TrustManager() {

			    public void checkClientTrusted(X509Certificate[] chain,
					String authType) throws CertificateException {

			    }

			    public void checkServerTrusted(X509Certificate[] chain,
					String authType) throws CertificateException {

			}

			public X509Certificate[] getAcceptedIssuers() {

				return null;

			}

		};

		sslContext.init(null, new TrustManager[] { tm }, null);

	}

	@Override
	public Socket createSocket(Socket socket, String host, int port,
			boolean autoClose) throws IOException, UnknownHostException {

		return sslContext.getSocketFactory().createSocket(socket, host, port,
				autoClose);

	}

	@Override
	public Socket createSocket() throws IOException {

		return sslContext.getSocketFactory().createSocket();

	}

    }

 *缺陷：虽然这个方案使用了HTTPS，客户端和服务器端的通信内容得到了加密，嗅探程序无法得到传输的内容，但是无法抵挡“中间人攻击”。例如，在内网配置一个DNS，把目标服务器域名解析到本地的一个地址，然后在这个地址上使用一个中间服务器作为代理，它使用一个假的证书与客户端通讯，然后再由这个代理服务器作为客户端连接到实际的服务器，用真的证书与服务器通讯。这样所有的通讯内容都会经过这个代理，而客户端不会感知，这是由于客户端不校验服务器公钥证书导致的。*

#### 使用自定义证书建立HTTPS连接 
为了防止上面方案可能导致的“中间人攻击”，我们可以下载服务器端公钥证书，然后将公钥证书编译到Android应用中，由应用自己来验证证书。<p>

**生成KeyStore**
要验证自定义证书，首先要把证书编译到应用中，这需要使用keytool工具生产KeyStore文件。这里的证书就是指目标服务器的公钥，可以从web服务器配置的.crt文件或.pem文件获得。同时，你需要配置bouncycastle，我下载的是bcprov-jdk16-145.jar。<p>

    keytool -importcert -v -trustcacerts -alias cert12306
     -file srca.cer -keystore cert12306.bks -storetype BKS -providerclass 
    org.bouncycastle.jce.provider.BouncyCastleProvider
    -provider path ./bcprov-jdk15on-146.jar -storepass pw12306

生产KeyStore文件成功后，将其放在app应用的res/raw目录下即可。
使用自定义KeyStore实现连接思路和TrushAll差不多，也是需要一个自定义的SSLSokcetFactory，不过因为还需要验证证书，因此不需要自定义TrustManager了。

    public class CustomerSocketFactory extends SSLSocketFactory { 
       private static final String PASSWD = "pw123456"; 
   
       public CustomerSocketFactory(KeyStore truststore) 
            throws NoSuchAlgorithmException, KeyManagementException, 
       KeyStoreException, UnrecoverableKeyException { 
             super(truststore); 
       } 
   
       public static SSLSocketFactory getSocketFactory(Context context) { 
         InputStream input = null; 
     try { 
       input = context.getResources().openRawResource(R.raw.example); 
       KeyStore trustStore = KeyStore.getInstance(KeyStore 
           .getDefaultType()); 
   
       trustStore.load(input, PASSWD.toCharArray()); 
   
       SSLSocketFactory factory = new CustomerSocketFactory(trustStore); 
   
       return factory; 
     } catch (Exception e) { 
       e.printStackTrace(); 
       return null; 
     } finally { 
       if (input != null) { 
         try { 
           input.close(); 
         } catch (IOException e) { 
           e.printStackTrace(); 
         } 
         input = null; 
       } 
     } 
     } 
   
    } 
