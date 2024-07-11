# 服务安全-mTLS

微服务的安全主要包含两种方式：面向用户的[基于token认证机制](https://www.okta.com/identity-101/what-is-token-based-authentication/)
和面向服务的[mTLS](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-mutual-tls/),这里主要介绍mTLS的实现。

mTLS是一种双向认证机制，服务端和客户端都需要验证对方的身份，这样既可以保证通信的安全性，
也可以保证跨集群调用中客户端和服务器的的身份校验。

## 知识点

* [零信任安全模式](https://www.cloudflare.com/zh-cn/learning/security/glossary/what-is-zero-trust/)
* [mTLS](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-mutual-tls/)
* [AWS PCA](https://aws.amazon.com/cn/private-ca/)
* [cert-manager](https://cert-manager.io/docs/)
* [k8s](https://kubernetes.io/zh/docs/concepts/overview/what-is-kubernetes/)

### k8s集群如何配置？
1. mTLS 需要一个私有的CA证书，用于颁发服务端和客户端的证书,这边使用的[AWS PCA](https://aws.amazon.com/cn/private-ca/)
   来颁发证书,你需要创建一个PCA服务。
2. 服务端和客户端都需要颁发证书，在k8s中部署[cert-manager](https://cert-manager.io/docs/)：
    ```shell
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml
   helm repo add jetstack https://charts.jetstack.io --force-update
   helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace
   ```
3. [部署 cert-manager csi-driver](https://cert-manager.io/docs/usage/csi-driver/installation/)
   ```shell
   helm repo add jetstack https://charts.jetstack.io --force-update
   helm upgrade -i -n cert-manager cert-manager-csi-driver jetstack/cert-manager-csi-driver --wait
   ```
4. [部署 aws-privateca-issuer](https://github.com/cert-manager/aws-privateca-issuer)
    ```shell
    # 注意：需要创建新的策略添加到节点组IAM角色中，以便节点可以访问私有CA
     helm repo add awspca https://cert-manager.github.io/aws-privateca-issuer
     helm install awspca/aws-privateca-issuer --generate-name
    ```
5. 创建自己的PCA颁发证书
   ``` yaml
     # aws-pca-issuer.yaml
      apiVersion: awspca.cert-manager.io/v1beta1
      kind: AWSPCAClusterIssuer
      metadata:
      name: <your-root-ca>
      spec:
      arn: <your aws root ca arn>
      region: ap-southeast-1
   ```
   ```shell
    kubectl apply -f ./cert-manager/aws-pca-issuer.yaml
    ```
6. 直接配置部署证书

```yaml
# your-tls.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: <your-cert-name>
  namespace: default
spec:
  commonName: <your domain name>
  dnsNames:
    - "*.<your domain name>"
  duration: 168h0m0s
  issuerRef:
    group: awspca.cert-manager.io
    kind: AWSPCAClusterIssuer
    name: <your-root-ca>
  renewBefore: 1h0m0s
  secretName: <your-cert-name>
  usages:
    - server auth
    - client auth
  privateKey:
    algorithm: "RSA"
    size: 2048
```

```shell
 kubectl apply -f ./cert-manager/your-tls.yaml
```

## 7. 部署证书到服务端

```yaml
 # helm values.yaml
 ingress:
   enabled: true
   hosts:
     - host: <your domain name>
       paths: /
   tls:
     - secretName: <your-cert-name>
       hosts:
         - <your domain name>
```

## 8. 使用[CSI Driver](https://cert-manager.io/docs/usage/csi/#enabling-mTLS-of-pods-using-cert-manager-csi-driver)给服务器自动部署证书

```yaml
volumes:
  - name: tls-server
    csi:
      readOnly: true
      driver: csi.cert-manager.io
      volumeAttributes:
        csi.cert-manager.io/issuer-name: "your-root-ca"
        csi.cert-manager.io/dns-names: "*.${POD_NAMESPACE}.svc.cluster.local"
        csi.cert-manager.io/issuer-kind: "AWSPCAClusterIssuer"
        csi.cert-manager.io/issuer-group: "awspca.cert-manager.io"
        csi.cert-manager.io/common-name: "${SERVICE_ACCOUNT_NAME}.${POD_NAMESPACE}"
        csi.cert-manager.io/duration: "168h"
        csi.cert-manager.io/renew-before: "1h"
volumeMounts:
  - name: tls-server
    mountPath: "/configs/tls-client"
    readOnly: true
```

## 使用mTLS证书
 使用mTLS证书，需要在客户端和服务端都配置证书，这里使用[grpc](https://grpc.io/docs/languages/go/quickstart/)的证书配置，其他协议基本一致。
这儿需要注意的是证书的自动更新，因为上面的配置会自动更新目录下的证书，所以我们需要在代码中监听证书的变化，然后重新加载证书。 
 具体代码实现可以参考：[server-side](https://github.com/GStones/moke-kit/blob/main/server/internal/cmux/cmux.go#L114),[client-side](https://github.com/GStones/moke-kit/blob/main/server/tools/dial.go#L64)

### 服务端代码：
```go
// server side
// NewConnectionMux creates a new connection mux.
// Watch the tls certificate and reload it when it changes.
func makeTLSConfig(logger *zap.Logger, tlsCert, tlsKey string, clientCa string) (*tls.Config, error) {
	if cert, err := tls.LoadX509KeyPair(tlsCert, tlsKey); err != nil {
		return nil, err
	} else if caBytes, err := os.ReadFile(clientCa); err != nil {
		return nil, err
	} else {
		ca := x509.NewCertPool()
		if ok := ca.AppendCertsFromPEM(caBytes); !ok {
			return nil, fmt.Errorf("failed to parse %v ", clientCa)
		}

		tlsCertValue := atomic.Value{}
		tlsCertValue.Store(cert)
		p, _ := path.Split(tlsCert)
		if _, err := tools.Watch(logger, p, time.Second*10, func() {
			logger.Info("service reloading x509 key pair")
			c, err := tls.LoadX509KeyPair(tlsCert, tlsKey)
			if err != nil {
				logger.Error("service failed to load x509 key pair", zap.Error(err))
				return
			}
			tlsCertValue.Store(c)
		}); err != nil {
			return nil, err
		}
		tlsConfig := &tls.Config{
			ClientAuth: tls.RequireAndVerifyClientCert,
			GetCertificate: func(info *tls.ClientHelloInfo) (*tls.Certificate, error) {
				c := tlsCertValue.Load()
				if c == nil {
					return nil, fmt.Errorf("certificate not loaded")
				}
				res := c.(tls.Certificate)
				return &res, nil
			},
			ClientCAs: ca,
		}
		return tlsConfig, nil
	}

}
```

### 客户端代码：
```golang
// client side
// makeTls make tls config
// Watch the client certificate and reload it when it changes
func makeTls(logger *zap.Logger, clientCert, clientKey, serverName, serverCa string) (*tls.Config, error) {
	cert, err := tls.LoadX509KeyPair(clientCert, clientKey)
	if err != nil {
		return nil, err
	}
	tlsCert := atomic.Value{}
	tlsCert.Store(cert)

	p, _ := path.Split(clientCert)
	if _, err := Watch(logger, p, time.Second*10, func() {
		logger.Info("client reload certificate")
		c, err := tls.LoadX509KeyPair(clientCert, clientKey)
		if err != nil {
			return
		}
		tlsCert.Store(c)
	}); err != nil {
		return nil, err
	}

	tlsConfig := &tls.Config{
		GetClientCertificate: func(info *tls.CertificateRequestInfo) (*tls.Certificate, error) {
			c := tlsCert.Load()
			if c == nil {
				return nil, fmt.Errorf("certificate not loaded")
			}
			cert := c.(tls.Certificate)
			return &cert, nil
		},
	}
	if serverName != "" {
		tlsConfig.ServerName = serverName
	}
	if serverCa != "" {
		ca := x509.NewCertPool()
		caBytes, err := os.ReadFile(serverCa)
		if err != nil {
			return nil, err
		}
		if ok := ca.AppendCertsFromPEM(caBytes); !ok {
			return nil, fmt.Errorf("failed to parse %q", serverCa)
		}
		tlsConfig.RootCAs = ca
	}
	return tlsConfig, nil
}
```

    
   




