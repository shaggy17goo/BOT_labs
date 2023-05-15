# Laboratorium 1 BOT 
### Skanowanie i przełamywanie zabezpieczeń

## Autorzy: 
* Wawrzyńczak Michał
* Gryka Paweł
-----------------------



## Zadanie 1 - Weryfikacja reguł firewalla 
- Przeprowadziliśmy skanowanie typu ping, aby poznać dostępne hosty w sieci.
![](https://i.imgur.com/63H1a0i.png)
- Ustaliliśmy, że w sieci znajduję się jeden host o adresie ip `192.168.241.132`


- W pierwszej kolejności wykonaliśmy skanowanie TCP Connect oraz Skanowanie Stealth.
![](https://i.imgur.com/7hzZQ4V.png)

- W trakcie skanowania obserwowaliśmy wymieniany ruch w Wiresharku Zaobserwowaliśmy wymianę pakietów, zgodną z poniższymi diagramami, świadczącą o tym, że skanowany port jest otwarty. Połączenie TCP między atakującym a ofiarą zostało zestawione.
![](https://i.imgur.com/QAgaZ9M.png)
![](https://i.imgur.com/9Xjb9Pc.png)
![](https://i.imgur.com/tXrSHAY.png)

- Wykonaliśmy także skanowania modyfikując takie parametry jak MTU, TTL czy dokonując dodatkowej fragmentacji pakietu. Wszystkie przeprowadzone skanowania dawały jednoznaczną odpowiedź, że skanowany port jest otwarty. 
![](https://i.imgur.com/Z2jTfgh.png)

- Podjęliśmy także próbę połączenie ssh do skanowanej maszyny, próba ta zakończyła się niepowodzeniem. Połączenie zostało zresetowane.
![](https://i.imgur.com/1ar7Fw1.png)

**Wniosek:** Port jest otwarty, natomiast próba nawiązanie połączenia jest blokowana na firewallu.



## Zadanie 2 - Przełamywanie zabezpieczeń z wykorzystaniem metasploita

### 2.1 Zbieranie informacji o usługach uruchomionych na hoście BOT_Lab1b

Na początku za pomocą ping scan'a ustaliliśmy adres maszyny do exploitowania:
![](https://i.imgur.com/mECUhKi.png)

Następnie wykonaliśmy skanowanie za pomocą Nikto oraz namp, ich wyniki przedstawiamy poniżej:
![](https://i.imgur.com/sIb3GdK.png)

```
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-25 07:01 EDT
Nmap scan report for 192.168.241.130
Host is up (0.0017s latency).
Not shown: 994 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 2.9p2 (protocol 1.99)
|_sshv1: Server supports SSHv1
| ssh-hostkey: 
|   1024 b8:74:6c:db:fd:8b:e6:66:e9:2a:2b:df:5e:6f:64:86 (RSA1)
|   1024 8f:8e:5b:81:ed:21:ab:c1:80:e1:57:a3:3c:85:c4:71 (DSA)
|_  1024 ed:4e:a9:4a:06:14:ff:15:14:ce:da:3a:80:db:e2:81 (RSA)
80/tcp   open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
| http-methods: 
|_  Potentially risky methods: TRACE
111/tcp  open  rpcbind     2 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100000  2            111/udp   rpcbind
|   100024  1           1024/tcp   status
|_  100024  1           1026/udp   status
139/tcp  open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp  open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-09-26T09:32:06
|_Not valid after:  2010-09-26T09:32:06
|_ssl-date: 2022-03-25T12:04:25+00:00; +1h01m50s from scanner time.
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_RC4_64_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|_    SSL2_RC4_128_WITH_MD5
|_http-title: 400 Bad Request
1024/tcp open  status      1 (RPC #100024)
MAC Address: 00:0C:29:EE:0C:96 (VMware)
Device type: general purpose
Running: Linux 2.4.X
OS CPE: cpe:/o:linux:linux_kernel:2.4
OS details: Linux 2.4.9 - 2.4.18 (likely embedded)
Network Distance: 1 hop

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: 1h01m49s
|_nbstat: NetBIOS name: KIOPTRIX, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

TRACEROUTE
HOP RTT     ADDRESS
1   1.74 ms 192.168.241.130

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.79 seconds

```

Po analizie otrzymanych wyników, ustaliliśmy, host posiada system opperacyjny Linux 2.4.9 - 2.4.18. Na hoście jest wiele podatnych serwisów. Między innymi są to:
- Apache 1.3.20
- mod_ssl 2.8.4

Host ma także otwarte porty, co sugeruje możliwe punkty wejścia:
- 22/tcp   open  ssh
- 80/tcp   open  http
- 111/tcp  open  rpcbind
- 139/tcp  open  netbios-ssn Samba smbd
- 443/tcp  open  ssl/https   Apache/1.3.20 (Unix)
- 1024/tcp open

### 2.2 Szukanie dostępnych eksploitów w metasploicie

W celu exploitowania hosta ```BOT_Lab1b``` sprawdziliśmy usługi znalezione w poprzednim punkcie. Testowaliśmy podatności z Apache ale Metasploit twierdził, że mimo tego, że exploitacja powiodła się to nie udało się nawiązać reverse shella. Drogą, która okazała się "tą dobrą" była exploitacja Samby. W Metasploit wyszukaliśmy frazę ```exploit/linux/samba``` i wybraliśmy ```trans2open```.

### 2.3 i 2.4 Przygotowanie i uruchomienie eksploita w metasploicie oraz dowody przełamania zabezpieczeń

Samo ustawienie opcji do exploita nie było skomplikowane, bardziej czasochłonne okazało się znalezienie odpowiedniego, działającego payloadu, po paru nieudanych próbach (głównie różnych wariantach meterpreter'a) udało się otrzymać powłokę atakowanego hosta za pomocą payloadu ```linux/x86/shell/reverse_tcp```. Poniżej przedstawiamy użyte opcje oraz samą exploitację wraz z wykonaniem odpowiednich poleceń identyfikujących hosta:

![](https://i.imgur.com/a2RDdpJ.png)

![](https://i.imgur.com/9Y49oz6.png)


## Zadanie 3
- Przeprowadziliśmy skanowanie typu ping, aby poznać dostępne hosty w sieci.
![](https://i.imgur.com/UFzF8yE.png)
- Ustaliliśmy, że w sieci znajduję się host o adresie ip `192.168.241.131`

- Następnie wykonaliśmy skanowanie hosta o adresie `192.168.241.131` przy wykorzystaniu narzędzia `nmap`, `nikto` oraz `dirb`, aby poznać uruchomione usługi i otwarte porty na tym hoscie.

![](https://i.imgur.com/dcgkx2R.png)


![](https://i.imgur.com/lyscXYt.png)

![](https://i.imgur.com/yWrwjRk.png)




- Wykorzystując przeglądarkę weszliśmy na aplikację działającą na skanowanym hoście i sprawdziliśmy dokładną wersję **WordPress'a - 1.5.1.1** 

![](https://i.imgur.com/C2He1Wj.png)

- Wyszukaliśmy dostępne exploity na tą konkretną wersję WordPress'a, udało nam się znaleźć natępując eksploit [WordPress Core 1.5.1.1 - SQL Injection](https://www.exploit-db.com/exploits/1033)

- Przeanalizowaliśmy kod exploita i po konsultacji z prowadzącym udało nam się spreparować zapytanie POST wykonujące SQL Injection, w wyniku którego otrzymaliśmy nazwę użytkownika i hash hasła.
![](https://i.imgur.com/j5LqNFz.png)
- Następnie zgodnie z instrukcją udało nam się wylistować sześciu użytkowników wraz z hashami ich haseł.
![](https://i.imgur.com/rRFiB3m.png)

- Następnie otrzymane hashe sprawdziliśmy w dostępnym online słowniku hashy, wszystkie hashe udało się bez problemy "złamać".
![](https://i.imgur.com/4OvdAEK.png)


- W aplikacji webowej zalogowaliśmy się na otrzymane konta, jeden z użytkowników ("GeorgeMiller") okazał się posiadać prawa administratora. Użytkownik ten posiadał uprawnienia do modyfikacji kodu źródłowego stron. Dokonaliśmy modyfikacji stronu `wp-content/themes/starburst/404.php` umieszczając w niej kod reverse-shell'a [https://github.com/pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell), zmodyfikowany o adres ip atakującego i wybrany port.
![](https://i.imgur.com/VJg4taf.png)
- Następnie uruchomiliśmy nasłuchiwanie na wybranym wcześniej porcie i uruchomiliśmy zmodyfikowaną stronę w celu uzyskania reverse-shell'a.

![](https://i.imgur.com/RlhxNSZ.png)

Mając dostęp do powłoki użytkownika `apache`, pierw sprawdziliśmy jego identyfikator w celu sprawdzenia jego roli w systemie.

- Naszym kolejnym zadaniem było uzyskanie uprawnień administratora w systemie, w tym celu sprawdziliśmy wersję jądra sytemu. Wyszukaliśmy dostępne eksploity na tą wersję - [https://www.exploit-db.com/exploits/15285](https://www.exploit-db.com/exploits/15285). Znaleziony eksploit wydawał się obiecujący niestety zabrakło nam czasu aby przetransferować i uruchomić go na atakowanej maszynie
![](https://i.imgur.com/oXq67YZ.png)

