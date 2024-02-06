# Rola `ba_srv`

Rola serwera repozytorium pakietów IBM spectrum Protect Backp Archive Client.
Przetestowana na Fedorze, ale powinna działać na każdym w miarę nowoczesnym Red Hacie.
Playbook jest przystosowany do systemów z SELinuxem w trybie *enforcing*.

Żeby móc użyć tego playbooka musisz:

- ustawić lub przekazać zmienne:
     - `latest_ba_url` - wskazuje skąd pobrać tara
     - `latest_ba_tar` - lokalna nazwa tara z paczkami rpm klienta BA

## Opis zmiennych

Wszyskie zmienne są w pliku `vars/main.yaml`.


- **www_root** - *Document root* serwera Lighttpd. Domyślnie podkatalog `/var/www/lighttpd`.
- **repo_dir** - podkatalog `{{ www_root }}`, do którego zostaną skopiowane pakiety RPM i gdzie zostianie utorzone repo.

     `repo_dir: "{{ www_root }}/ba"`

- **arch_dir** - podkatalog `{{ www_root }}`, do którego zostanie pobrana paczka z IBM

     `arch_dir: "{{ www_root }}/ba_arch"`


- **se_lnx** - kontroluje czy SELinux jest w trybie *enforcing*. Serwer w trybie *permissive* albo *disabled* jest zepsuty, więc nie ruszać!

     `se_lnx: True`


## Zadania w playbooku `ba_srv_inst.yaml`

### Pakiety serwera repozytorium

Instaluje paiety podane w na zmiennej `{{ srv_pkgs }}`. Wrzucałem tam "co wyszło w praniu" względem instalacji "Fedora minimal".

### VG dla repozytorium

Zakłada volume grupę `{{ vg }}` na PV: `{{ lun }}`. W razie potrzeby założy także PV.

### LV dla repozytorium

Zakłada LV `{{ lv }}` w grupie `{{ vg }}` z założeniem `-l 100%FEE`.

### Punkt mntowania repozytorium

Zakłada katalog `{{ www_mnt }}`. To będzie punkt montowania tworzonego repozytorium i jednocześnie `var.server_root` dla Lighttpd. Domyślnie: `"/var/www"`.

### Formatowanie filesystemu XFS

Formatuje `{{ lv }}` na XFSowo.

### Montowanie filesystemu

Montuje filesystem `{{ www_mnt }}` i dodaje do `/etc/fstab`.

### Podkatalog repozytorium

Na zmontowanym filesytemie zakłada `{{ ba_root }}` czyli katalog, do którego zostaną rozpakowane RPMy z paczkami i zostanie w nim utworzone repozytorium.

### Otwieranie portów firewalla

Dodaje do `firewalld` porty z listy `{{ fw_services }}`: teraz http i https.

### Ustawianie bulionów dla Lighttpd

Lighttpd przy włączonym *SELinux* potrzebuje ustawiania *sebool* `httpd_setrlimit = on`.

### Wrzucanie pliku index.html

Durnostojka `index.html` generowana z szablonu `templates/index.html.j2`. Podstawia tylko nazę hosta, link do katalogu z paczkami i link do pliku repo.

### Odpakowanie TARa z klientem BA do {{ ba_root }}

Rozpakowuje paczkę `{{ lates_ba_tar }}` do `{{ ba_root }}`. 

### "Zezwolenie na directory browsing w {{ ba_root }}"

Nadpisuje plik `/etc/lighttpd/conf.d/dirlisting.conf` serwera Lighttpd wersją z dodanym kawałkiem konfiuracji zezwalającym na *directory browsing* na `{{ ba_root }}`:

```
# Zezwolenie na przeszukiwanie katalogu {{ ba_root }} na serwerze lighttpd
$HTTP["url"] =~ "^/ba($|/)" {
     dir-listing.activate = "enable" 
   }
```

**Uwaga:** ścieżka zahardkodowana w regexpa. Poprawić na template!

### Dodawanie polityk SELinux dla katalogu {{ doc_root}}

Dla serwera z uruchomionym *SELinuxem*: Lighttpd nie dodaje polityki dla swojego standardowego katalogu na html'e!! Bez tego selinux nie pozwala serwerowi serwować czegokolwiek. Kontekst:

```
     target: '{{ doc_root }}(/.*)?'
     seuser: "system_u"
     setype: "httpd_sys_rw_content_t"
```

### Tworzenie repozytorium RPM w {{ ba_root }}

Wywołanie `createrepo {{ ba_root }}`. Tworzy repozytorium yum/dnf. 

### Generowanie pliku repozytorium {{ ba_root }}/ba.repo

Dla leniwych. Tworzy plik z definicją repozytorium. Można go sobie siągnąć i wrzucić do `/etc/yum.repos.d` i skorzystać z tego serwera do instalacji paczek klienta Spectrum Protect.

### Ustawianie kontekstu SELinux na {{ doc_root }}

Po skończonym dodawaniu plików do `{{ doc_root }}` trzeba ustawić kontekst przypisany wcześniej, bo inaczej nic nie będzie widać. 

To zadanie nie jest idempotentne, ale to nie szkodzi. `restorecon` jest zawsze zdrowy ;-)

### Włączanie Lighttpd

W końcu można używać.

## Zadania w tasku `ba_srv_refresh.yaml`

Ten task ma za zadanie rozpakować archiwum wskazane przez zminną `latest_ba_tar` do już istniejącego repozytorium wskazanego zmienną `ba_root` i przebudować `repodata`.

**To do:** 
1. Być może warto dodać sprawdzanie co jest nadpisywane. Na razie uruchmienie playbooka zawsze nadpisze repo.
1. Dodać kod sprawdzający 

### Odpakowanie TARa z klientem BA do {{ ba_root }}

Rozpakowuje paczkę `{{ lates_ba_tar }}` do `{{ ba_root }}`. 

**Uwaga:** Tutaj usunięto parametr `creates`, co powoduje, że to zdanie się zawsze wykona!

### Dodawanie polityk SELinux dla katalogu {{ doc_root}}

Dla serwera z uruchomionym *SELinuxem*: Lighttpd nie dodaje polityki dla swojego standardowego katalogu na html'e!! Bez tego selinux nie pozwala serwerowi serwować czegokolwiek. Kontekst:

```
     target: '{{ doc_root }}(/.*)?'
     seuser: "system_u"
     setype: "httpd_sys_rw_content_t"
```

### Tworzenie repozytorium RPM w {{ ba_root }}

Wywołanie `createrepo {{ ba_root }}`. Tworzy repozytorium yum/dnf. 
**Uwaga:** Tutaj usunięto parametr `creates`, co powoduje, że to zdanie się zawsze wykona!

### Generowanie pliku repozytorium {{ ba_root }}/ba.repo

Dla leniwych. Tworzy plik z definicją repozytorium. Można go sobie siągnąć i wrzucić do `/etc/yum.repos.d` i skorzystać z tego serwera do instalacji paczek klienta Spectrum Protect.

### Ustawianie kontekstu SELinux na {{ doc_root }}

Po skończonym dodawaniu plików do `{{ doc_root }}` trzeba ustawić kontekst przypisany wcześniej, bo inaczej nic nie będzie widać. 

To zadanie nie jest idempotentne, ale to nie szkodzi. `restorecon` jest zawsze zdrowy ;-)