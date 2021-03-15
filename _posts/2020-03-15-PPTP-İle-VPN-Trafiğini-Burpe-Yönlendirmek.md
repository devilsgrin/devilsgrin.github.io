---
layout: default
---

# [](#header-1)PPTP (Point-to-Point Tunneling Protocol) İle VPN Trafiğini Burp'e Yönlendirmek

Bugün karşıma çıkan bir case'de, uzaktaki bir android cihaza bulaşmış olan malwareyi
analiz etmem gerekti. Neler yapabileceğime dair araştırırken, **PPTP VPN** ile uzaktaki
telefonu önce kendi serverime bağlayıp, sonrasında serverdeki **PPTP** trafiğini Burp'e
yönlendirebileceğimi öğrendim. Her şeyi başarıyla yaptıktan sonra da, bu yazıyı yazma
gereği duydum, herkese iyi okumalar :)


## [](#header-2)PPTP Nedir ? 
Türkçe açılımıyla **Noktadan noktaya Tünelleme Protokolü**; TCP 1723 portunda çalışarak,
noktadan noktaya iletişim kuran **VPN** Protokollerinden sadece birisidir. 
Kısaca; IP bazında kullanım için bu protokol, **IP** paketlerini **'PPP'** çerçevesi ile
**MS-CHAPv2** veya **EAP-TLS** gibi şifreleme anahtarlarıyla **MPPE** ile şifreleyerek, IP paketlerini
kapsüllere döndürüp noktadan noktaya iletişim sağlar. 

#### [](#header4)Neden İletişim Esnasında PPTP Kullanacağız Peki ? 
> Çünkü çalışma mantığı **Openvpn**'in server-client tünelleme mantığının aksine,
> _noktadan noktaya_ olduğu için bu protokolü kullanacağız.

## [](#header-2)Başlayalım ...
İlk öncelikle android trafiğini yönlendirmek amacıyla bir adet PPTP serverine ihtiyacımız var.
Ben bunun için **Digitalocean**'dan bir tane droplet ayağa kaldırdım. 
Sonrasında [bu link'den](https://rdmfiles.s3.amazonaws.com/scripts/pptp.txt), bizim içim sunucumuza
otomatik olarak PPTP server kurulumu yapacak olan bash scriptini indiriyor ve çalıştırıyoruz. 
Her şey yolunda giderse, en son çıktı olarak; 
```bash 
iptables -t nat -A PREROUTING -i ppp0 -p tcp --dport 80 -j REDIRECT --to-ports 8080
iptables -t nat -A PREROUTING -i ppp0 -p tcp --dport 443 -j REDIRECT --to-ports 8080
```
gibi bir çıktı alıyorsak işlem tamamdır.

Bütün işlemler sırasında belki de en fazla dikkat etmemiz gereken noktası burası. Çünkü **PPP** ile
gelen _HTTP_ ve _HTTPS_ trafiklerini, Burp Proxy üzerinden dinleyeceğimiz porta yönlendirmemiz gerekiyor.
Unutarsanız hiç bir trafiği göremezsiniz :)

 
 
## [](#header-2)Burp Konfigürasyonu
Bu casemizde biz, PPTP Serverimiz üzerinden gelen trafiği dinlediğimiz durumu konuşuyoruz. PPTP Servere
bağlı olan başka bir cihaz üzerinden, android trafiğini dinlemeyi değil. Dolayısıyla, Digitalocean'dan 
ayağa kaldırdığımız dropletimize [Burp Suite Community Editon'u kuruyoruz](https://portswigger.net/burp/communitydownload).
> Firefox tarayıcı ve foxy proxy eklentisini kurmayı unutmayın

Burp'u açtıktan sonra, **Proxy** sekmesinden **Options** kısmına geliyor ve **Proxy Listener** yazan yerde 
**Add**'ye tıklayarak yeni Proxy Listenerimizi ekliyoruz. Burada dikkat etmeniz gereken nokta; 
**Bind to port** kısmına **8080** portunu, **Bind to address** kısmına ise **All interfaces** seçeneklerini vermek. 
Bu sayede 8080 portu üzerinden, bütün interfacelerimize gelen trafiği dinlemiş oluyoruz.

> Otomatik olarak 1723 portunda çalışan, PPP'yi de dinlemiş oluyoruz. Detayına bakmak isterseniz ``` ifconfig ``` komutuyla PPP'nin çalıştığı
interfaceyi, ``` netstat -tunlp ``` komutuylada açık olan portları görebilirsiniz. 

Durun ! Daha Proxy Listener sekmesiyle işimiz bitmedi, aman ekleyeyim demeyin. **Request Handling** kısmına gelin ve buradan 
**Support invisible proxying (enable only if needed )** kutucuğunu işaretleyin. 
Bunu yapmamızın sebebi, android telefonumuza her hangi bir proxy tanımı yapmayacağız. Dolayısıyla, proxy tanımıyan istemcimiz, doğrudan
Proxy Listenere bağlanacak ve aradaki trafiği dinlememize izin verecektir.

Tüm bu işlemleri bitirdikten sonra, yeni proxy konfigürasyonunuzu ekleyin çünkü sırada sertfika var.
Droplet üzerindeki Firefox tarayıcında Foxy Proxy ile proxy ayarlarınızı yaptıktan sonra, sertfikanızı indirin. İndirdiğiniz **cacert.der** adlı
sertfikayı, cacert.**cer** olarak değiştirin ve android cihazınıza gönderin. Çünkü android cihazlar **.der** uzantılı sertfikalar yerine, **.cer**
uzantılı sertfikaları tanırlar.
> Sertfikayı nasıl android cihazınıza tanıtacağınızı anlatmıyorum, [bu linkten](https://portswigger.net/support/installing-burp-suites-ca-certificate-in-an-android-device) nasıl yapıldığını öğrenebilirsiniz. 

## [](#header-2)Android Telefondan PPTP Servere Bağlanmak
Artık bütün her şeyimiz hazır olduğuna göre, Android cihazımızla PPP Serverimize bağlanabilir ve serverimiz üzerinden tüm trafiği dinleyebiliriz.
Telefonunuzun ayarlar bölümünden **VPN** kısmını açın ve bağlantı türü olarak **PPP** seçin. Hostname kısmına server ip'inizi, username ve password
kısmına ise, PPP'yi kuran scriptin bize sorduğu ve belirlediğimiz username ve password'ü yazın. 

Her şeyi doğru yaptıysanız, artık Burp'e gidebilir ve uzaktakan PPP ile bize bağlanmış olan Android cihazdan çıkan tüm trafiği dinleyebiliriz.

Yapmanız gereken sadece Intercept'i açmak ve gelen giden trafiği görmek :)

Bir başka yazıda görüşmek üzere, hoşçakalın ...
