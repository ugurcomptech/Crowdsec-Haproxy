# CrowdSec + HAProxy Entegrasyonu

Bu repo, CrowdSec’in ürettiği güvenlik kararlarının (ban, throttle, captcha vb.) HAProxy üzerinde gerçek zamanlı olarak uygulanmasını sağlayan Lua tabanlı bouncer entegrasyonunu içerir. Amaç; HAProxy üzerinden gelen trafiği CrowdSec kararlarına göre dinamik olarak filtrelemek ve kötü amaçlı IP’leri uygulama katmanına ulaşmadan engellemektir.

Yapı, Ubuntu 22.04 / 24.04 üzerinde test edilmiştir ve CrowdSec’in resmi crowdsec-haproxy-bouncer paketini temel alır. HAProxy üzerindeki tüm HTTP istekleri, Lua bouncer aracılığıyla CrowdSec’e doğrulanır ve banlanan/IP reputation’ı düşük olan istemciler için HAProxy seviyesinde 403 Forbidden döner. Bu sayede hem performans kaybı minimuma iner hem de saldırı yüzeyi azalır. Yapı IPv4/IPv6 desteklidir ve sistem yeniden başlasa bile kararlar otomatik olarak geri yüklenir.

## 1. Kurulum
```
sudo apt update
sudo apt install crowdsec crowdsec-haproxy-bouncer haproxy -y
```

## 2. HAProxy Bouncer Key Oluşturma
```
sudo cscli bouncers add haproxy-bouncer
# API key otomatik olarak kaydedilir
```
## 3. HAProxy Servisini Aktif Et
```
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

## 4. HAProxy Yapılandırması (Lua Bouncer Entegrasyonu)
```
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<EOF
global
    log /dev/log local0
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    daemon

    lua-prepend-path /usr/lib/crowdsec/lua/haproxy/?.lua
    lua-load /usr/lib/crowdsec/lua/haproxy/crowdsec.lua
    setenv CROWDSEC_CONFIG /etc/crowdsec/bouncers/crowdsec-haproxy-bouncer.conf

defaults
    log global
    mode http
    timeout connect 5000
    timeout client  60000
    timeout server  60000

frontend http-in
    bind *:80
    http-request lua.crowdsec_allow
    default_backend webservers

backend webservers
    balance roundrobin
    server web1 203.0.113.24:80 check accept proxy
EOF
```

Konfigürasyonu doğrula:
```
sudo haproxy -f /etc/haproxy/haproxy.cfg -c
sudo systemctl restart haproxy
```

## 5. Çalışma Durumunu Kontrol Et
```
# Bouncer aktif mi?
sudo systemctl status crowdsec-haproxy-bouncer --no-pager
# HAProxy logları
sudo tail -f /var/log/haproxy.log | grep lua
```

## 6. Manuel Test (Kendi IP’ni 2 Dakikalığına Banla)
```
# Dış IP'n
curl -s ifconfig.me

# Kendini banla
sudo cscli decisions add --ip $(curl -s ifconfig.me) --duration 2m --type ban

# HAProxy erişimini test et
curl -I http://203.0.113.157   # 403 Forbidden dönecek

# Ban’ı kaldır
sudo cscli decisions delete --ip $(curl -s ifconfig.me)
```

## 7. Günlük Yönetim Komutları
```
# Aktif banlar
sudo cscli decisions list | wc -l

# En çok banlanan IP'ler
sudo cscli decisions list | head -20
```


**Not:** *Bu yapılandırma CrowdSec’in ürettiği tehdit verilerini doğrudan HAProxy üzerinden uygulamanıza olanak tanır. Reverse proxy katmanında IP bazlı engelleme yapmak hem backend uygulamalarını korur hem de saldırıların yoğun olduğu durumlarda sistem kaynaklarının daha verimli kullanılmasını sağlar. Lua bouncer’ın mimarisi oldukça hafif olduğu için yüksek trafikli ortamlarda bile performans maliyeti yok denecek kadar azdır.*
