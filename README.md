# haproxy
```


                                servapp1
                              +-----------------+
                              |  172.23.0.11    |		
        LB1                   +-----------------+     
    +-------------------+                             
    | 172.23.0.9        |       servapp2                   
    +-------------------+     +-----------------+    
                              |  172.23.0.12    |     
                              +-----------------+

```

## fichier de configuration Haproxy
-> HAPROXY configuration : Global et Default <-
=========

```
global
    # global settings here

defaults
    # defaults here

frontend
    # a frontend that accepts requests from clients

backend
    # servers that fulfill the requests
```

<br>
* section global : s'applique à haproxy lui-même

<br>
```
global
		log /dev/log local5 debug
		chroot /var/lib/haproxy
		stats socket /run/haproxy/admin.sock mode 660 level admin
		stats timeout 30s
		user haproxy
		group haproxy
		daemon
```

<br>
* log : attention log binaire format syslog

* chroot : manière de sécuriser haproxy en chrootant dans /var/lib/haproxy

* stats : activation de la socket (interaction hatop

* timeout : sur la socket stat

* user/group : qui lance le process haproxy

* daemon : faire tourner en background

* éventuellement le ssl

------------------------------------------------------------------------------------



-> HAPROXY configuration : Global et Default <-


<br>
* section default : s'applique si aucune autre configuration n'est précisée (sur front/back)

<br>
```
defaults
		log global
		mode http
		option httplog
		option dontlognull
		timeout connect 5000
		timeout client 50000
		timeout server 50000
		errorfile 400 /etc/haproxy/errors/400.http
```

<br>
* log global : log tout vers syslog

* mode http : type de LB (TCP ou HTTP)

* option : log des requêtes http

* dontlognull : ne log pas les requêtes null

* timeout connect : connexion au serveur cible

* timeout client : connexion cliente (browser)

* timeout server : la réposne du serveur cible


-> Listen <-

* autre format réunissant frontend/backend

```
listen monapp
		bind: *.80
		server server1 127.0.0.1 5000
```



## 1)exemple de configuration avec plusieurs backend et acl:
```bash
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
frontend http_proxy
    bind :80
    mode http

    # Monitor uri used by Keepalived and OVH Load Balancer IP
    monitor-uri /

    # For logging purpose
    capture request header X-Forwarded-For len 15
#---------------------------------------------------------------------
 # Backend ACL 
    acl is_app1 path_beg /app1
    acl is_app2 path_beg /app2
    acl is_app3 path_beg /app3

# Backend rules
    use_backend app1 if is_app1
    use_backend app2 if is_app2
    use_backend app3 if is_app3
# DEFAULT
    default_backend app1
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app1
    balance     roundrobin
    server  kmaster 172.42.42.100:30891 check
backend app2    
    server  kmaster 172.42.42.100:31953 check
backend app3    
    server  kmaster 172.42.42.100:32558 check
```
 


## 3)fetch method (Matching d'url (path) ou routing)


-> HAPROXY : fetch method <-
=========

* en mode http

* à partir d'une url orienter les requêtes

<br>

exemple : 

```
acl is_api path i /premniveau/deuxniveau/index.html
```

<br>
* différentes captures :

		* path : match exactement
		* path_beg : match le début du chemin
		* path_dir : match un des directory
		* path_end : match la fin d'url
		* path_len : match sur la longueur (gt, lt...)
		* path_reg : match par regex
		* path_sub : match un morceau

-------------------------------------------------------------------------

-> Exemples <-

* path :

```
frontend myapp_front
    bind *:80
    mode http

    acl is_match path -i /premniveau/deuxniveau/index.html
    use_backend load1 if is_match

    default_backend load2
```

* path_beg :

```
acl is_match path_beg -i /premniveau
```


-------------------------------------------------------------------------

-> Exemples <-


* path_dir :

```
acl is_match path_dir -i /deuxniveau
```

* path_end :

```
acl is_match path_end -i index.html
```

* path_len :

```
acl is_match path_len 11
```


* path_reg :

```
acl is_match path_reg -i .(css|js)$
```

* path_string :

```
acl is_match path_string -i page
```
# haproxy avec keepalived
* haproxy : load-balancer = SPOF


* doublonner le service mais comment ? haproxy ? non


* keepalived : load-balancer qui va nous permettre de gérer une VIP


* si perte de Haproxy1 la VIP est basculée sur Haproxy2 etc

```
                            +-----------------+
                        LB1 |  172.23.0.9     |		 Pool servers
      VIP                   +-----------------+     +-----------------+
  +-------------------+                             | 172.23.0.11     |
  | 172.23.0.14       |                             |------------------
  +-------------------+     +-----------------+     | 172.23.0.12     |
                        LB2 |  172.23.0.10    |     +-----------------+
                            +-----------------+

```
## fichier de configuration pour le premier LB1

```
vrrp_script reload_haproxy {
		script "killall -0 haproxy"
		interval 1
}

vrrp_instance VI_1 {
   virtual_router_id 100
   state MASTER
   priority 100

   # interval de check
   advert_int 1

   # interface de synchro entre les LB 
   lvs_sync_daemon_interface enp0s9
   interface enp0s9

   # authentification entre les 2 machines LB
   authentication {
		auth_type PASS
		auth_pass secret
   }

   # vip
   virtual_ipaddress {
		192.168.99.110/32 brd 192.168.99.255 scope global
   }

   track_script {
	   reload_haproxy
   }

}

```

## fichier de configuration pour le deuxième LB2

```
vrrp_script reload_haproxy {
		script "killall -0 haproxy"
		interval 1
}

vrrp_instance VI_1 {
   virtual_router_id 100
   state MASTER
   priority 100

   # interval de check
   advert_int 1

   # interface de synchro entre les LB 
   lvs_sync_daemon_interface enp0s9
   interface enp0s9

   # authentification entre les 2 machines LB
   authentication {
		auth_type PASS
		auth_pass secret
   }

   # vip
   virtual_ipaddress {
		192.168.99.110/32 brd 192.168.99.255 scope global
   }

   track_script {
	   reload_haproxy
   }

}
```
## Sticky table : maintien de session
-> HAPROXY : Sticky Table <-
=========



<br>
* maintenir les sessions sur les mêmes serveurs :

  - base de données

  - certificats

  - cache





-----------------------------------------------------------------

-> HAPROXY : mise en place <-

* install de 2 serveurs web sur le port 80

* configuration classique

```
frontend myapp_front
    bind *:80
    mode http
    default_backend pool_load 

backend pool_load
    server serv1 172.17.0.3:80
    server serv1 172.17.0.3:80
```

-------------------------------------------------------------------


-> HAPROXY : Sticky table <-


* en mode sticky table

```
frontend myapp_front
    bind *:80
    mode http
    default_backend pool_load

backend pool_load
    mode http
    server serv1 172.17.0.3:80
    server serv1 172.17.0.4:80
    stick-table type ip size 1m expire 30m
    stick match src
    stick store-request src
```
explication:
stick-table type ip size 1m expire 30m : type du table sticky sera basé sur l'ip source et la taille maximale de la table est 1m (il ya un système de nettoyage qui réinitialise la table), la durée d'expiration de la session est de 30 minutes.
# la gestion des certificats avec haproxy 

-> HAPROXY : SSL ou plutôt TLS <-
=========

<br>
* 3 cas :

  - terminaison SSL : arrivée HTTPS après HTTP

  - passthrough : transparent (TCP)

  - réencryption : entrée cert1 après cert2

-----------------------------------------------------------------

## -> HAPROXY : Terminaison <-


* install de 2 serveurs web sur le port 80

* install haproxy

```
apt-get install haproxy
```

* configuration classique

```
frontend myapp_front
    bind *:80
    mode http
    default_backend pool_load 

backend pool_load
    server serv1 172.17.0.3:80
    server serv1 172.17.0.4:80
```
-------------------------------------------------------------------

-> HAPROXY : Terminaison <-


* génération du certificat auto-signé

```
openssl req -x509 -newkey rsa:2048 -keyout digi.key -out digi.crt -days 365 -nodes
```

* fusion en .pem

```
mkdir /etc/ssl/digi/
cat digi.key digi.crt > /etc/ssl/digi/digi.pem
```

* changement de configuration

```
frontend myapp_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/digi/digi.pem no-sslv3
    redirect scheme https if !{ ssl_fc }
    mode http
    default_backend pool_load

backend pool_load
    server serv1 172.17.0.3:80
```

Rq 1 : ssl_fc retourne True si ssl
Rq 2 : nosslv3 pour pas de ssl3 (idem no-tlsv10)

-----------------------------------------------------------------


## -> HAPROXY : Passthrough <-



* machine 2 : install d'un serveur nginx

```
apt-get install nginx
```

* machine 1 : install haproxy

```
apt-get install haproxy
```

* création d'un certificat pour le nginx

```
openssl req -x509 -newkey rsa:2048 -keyout digimind.key -out digimind.crt -days 365 -nodes
``` 

* configuration du nginx

```
        listen 443;
        ssl    on;
        ssl_certificate    /etc/ssl/nginx/digimind.crt; 
        ssl_certificate_key    /etc/ssl/nginx/digimind.key;

```

-------------------------------------------------------------------

* configuration de haproxy avec tcp


```
frontend myapp_front
    bind *:443
    mode tcp
    default_backend pool_load

backend pool_load
    mode tcp
    server serv1 172.17.0.3:443
```


----------------------------------------------------------------------


## -> HAPROXY : réencryption <-

* client > certificat haproxy > serveur > certificat nginx

* génération du certificat auto-signé pour haproxy

```
openssl req -x509 -newkey rsa:2048 -keyout digimind.key -out digimind.crt -days 365 -nodes
```

* fusion en .pem

```
mkdir /etc/ssl/digimind/
cat digimind.key digimind.crt > /etc/ssl/digimind/digimind.pem
```

* changement de configuration

```
frontend myapp_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/digimind/digimind.pem 
    redirect scheme https if !{ ssl_fc }
    mode http
    default_backend pool_load

backend pool_load
    mode http
    server serv1 172.17.0.3:443 check ssl verify none

```

Rq 1 : ssl_fc retourne True si ssl
Rq 2 : verify none (car cert auto-signé)
