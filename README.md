# traefik-reverse-proxy-for-docker
Using Traefik as a reverse proxy for Docker Containers.


### Network

Öncelikle Docker üzerinde Traefik ve uygulama container'larının ortak çalışacağı bir network oluşturuyoruz.

```
docker network create web
```

Yeni oluşturduğumuz Docker Network'ümüzü güvenlik duvarı üzerinden yapılandırmamız gerekiyor. Bunun için aşağıdaki adımları takip ediyoruz.

- Yeni oluşturduğumuz Docker Network'ümüzün VPS üzerindeki Connection DEVICE bilgisine erişmemiz gerekiyor. Docker Network'lerini listeliyoruz.

> ```
docker network ls
```
[![](http://barisates.com/git/traefik/network-ls.png)](http://barisates.com/git/traefik/network-ls.png)
Buradaki NETWORK_ID bilgisi ile VPS üzerindeki network bağlantılarımızı karşılaştırarak doğru bağlantıya yetkilendirme yapacağız.

- Daha sonra VPS üzerindeki network bağlantıları listeliyoruz.

>  ```
nmcli connection show
```
[![](http://barisates.com/git/traefik/network-connection.png)](http://barisates.com/git/traefik/network-connection.png)
DEVICE alanındaki değer ile bağlantımıza güvenlik duvarı üzerinden yetkilendirme yapacağız.

- Firewall üzerinden yeni oluşturduğumuz Docker Network'üne yetki yanımlıyoruz.

 ```
firewall-cmd --permanent --zone=trusted --change-interface=br-ff48477ab8a4
firewall-cmd --reload
```

### Traefik

Öncelikle Traefik Dashboard erişim için bir yönetici parolası oluşturacağız. Traefik yapılandırma dosyasında yönetici giriş bilgileri şifreli bir şekilde tutuluyor. Bu bilgileri iki şekilde şifreleyebilirsiniz.

##### htpasswd kullanarak bilgileri şifreleyebilirsiniz;

Önce `htpasswd` için gerekli paketleri yüklüyoruz.

```bash
sudo apt-get install apache2-utils
```

Daha sonra `htpasswd` ile Dashboard için kullanacağımız şifreyi oluşturuyoruz. `SECURE_PASSWROD` alanına kullanmak istediğini şifrenizi girin.

```bash
htpasswd -nb admin SECURE_PASSWROD
```

Programdan çıktı şu şekilde görünecektir:

```bash
admin:$apr1$9r5rUTis$nqG/J4R7365QtB7JBlc2N0
```

##### Htpasswd Generator kullanarak bilgileri şifreleyebilirsiniz;

https://hostingcanada.org/htpasswd-generator/ adresi üzerinden `Username` ve `Password` alanlarını doldurduktan sonra `Mode` alanını **Apache specific salted MD5 (insecure but common)* seçerek şifrenizi oluşturabilirsiniz.

#### Traefik Yapılandırma Dosyası

Traefik Dashboard şifremizi hazırlıladıktan sonra `traefik.toml` dosyamızı hazırlıyoruz. Ben repo içerisine örnek bir dosya bıraktım.

Yapılandırma dosyamız aşağıdaki gibi görünecek;

```toml
defaultEntryPoints = ["http", "https"]

[entryPoints]
  [entryPoints.dashboard]
    address = ":8080"
    [entryPoints.dashboard.auth]
      [entryPoints.dashboard.auth.basic]
        users = ["admin:YOUR_SECURE_PASSWORD"]
  [entryPoints.http]
    address = ":80"
      [entryPoints.http.redirect]
        entryPoint = "https"
  [entryPoints.https]
    address = ":443"
      [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      certFile = "certs/yourdomain.com.crt"
      keyFile = "certs/yourdomain.com.key"

[api]
entrypoint="dashboard"

[docker]
domain = "yourdomain.com"
watch = true
network = "web"
```
- Traefik'i bütün istekleri `https` üzerine yönlendireck şekilde yapılandırıyoruz. 
- Yayın yapacağımız adresimizin sertifika dosyasını örnekteki gibi `.key` ve `.crt` olarak tanımlıyoruz. (Sertifika dosyamızı `.key` ve `.crt` olarak nasıl ayıracağımızı bu repoda anlattım; https://github.com/barisates/docker-registry-server#ssl-certificate)
- Bir önceki adımda oluşturduğumu şifremiz `YOUR_SECURE_PASSWORD` kısmına gelecek.

#### Traefik Konteynırımızı Çalıştırıyoruz

Öncelikle Traefik için bir `docker-compose` dosyası hazırlıyoruz. Ben repo içerisine örnek bir dosya bıraktım, içerik aşağıdaki gibi görünüyor;

```yml
version: "3"
services:
  traefik:
    network_mode: web
    container_name: traefik
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.toml:/traefik.toml
      - ./certs:/certs
    ports:
      - 80:80
      - 443:443
    labels:
      - traefik.frontend.rule=Host:monitor.yourdomain.com
      - traefik.port=8080
    image: traefik:1.7.2-alpine
```

Daha sonra `root` klasörüne gidiyoruz. Yapılandırma dosyamızı (traefik.toml), docker-compose ve sertifika dosyalarımızı bu dizine kopyalıyoruz. `root` klasörümüz aşağıdaki gibi görünecektir.

```
root
│   traefik.toml
│   docker-compose.yml 
└───certs
     │   yourdomain.com.crt
     │   yourdomain.com.key
```

Şimdi Treafik konteynırımızı çalıştırabiliriz;

```bash
docker-compose up -d
```

Docker komutları ile Traefik konteynırınızı başlatmak için;

```bash
docker run -d \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD/traefik.toml:/traefik.toml \
  -v $PWD/certs:/certs \
  -p 80:80 \
  -p 443:443 \
  -l traefik.frontend.rule=Host:monitor.yourdomain.com \
  -l traefik.port=8080 \
  --network web \
  --name traefik \
  traefik
```

Artık aynı VPS üzerinde çalışan Docker konteynırlarınızı, SSL desteği ile birlikte barındırabilirsiniz. Örnek olarak bir uygulamayı yayına alalım;



https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-18-04
https://medium.com/@xavier.priour/secure-traefik-dashboard-with-https-and-password-in-docker-5b657e2aa15f
https://hostingcanada.org/htpasswd-generator/
