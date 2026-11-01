# Linux User Management — Scheda di ripasso

Ultimo aggiornamento: 25 ottobre 2026

## Mappa mentale

Il sistema ragiona sui numeri (UID, GID), non sui nomi: i nomi sono solo una traduzione letta da `/etc/passwd` e `/etc/group`.

|  | sudo | su |
| --- | --- | --- |
| Cosa fa | Esegue **un comando** come un altro utente | Ti fa **diventare** un altro utente (apre una shell) |
| Password chiesta | La **tua** | Quella dell'**utente di destinazione** |
| Chi decide | `/etc/sudoers` | Conoscere la password |
| Tracciato | Sì, nei log di sistema | Solo il login |
| Uso tipico | `sudo -u alex comando` | `su - alex` oppure `su alex -c comando` |

| File | Contiene | Chi lo legge |
| --- | --- | --- |
| `/etc/passwd` | Utenti: nome, UID, GID primario, home, shell | Tutti |
| `/etc/shadow` | Hash delle password e scadenze | Solo root |
| `/etc/group` | Gruppi, GID e membri supplementari | Tutti |

| Comando | Serve a |
| --- | --- |
| `useradd` | Creare un utente |
| `usermod` | Modificare un utente (gruppi, blocco, scadenza) |
| `userdel` | Eliminare un utente |
| `passwd` | Gestire la password |
| `groupadd` / `addgroup` | Creare un gruppo (addgroup è il wrapper Debian) |
| `groupdel` / `delgroup` | Eliminare un gruppo |

## Ricognizione

`id` è il primo comando: dice chi sei, il tuo gruppo primario e i gruppi supplementari.

```
uid=1001(alex) gid=1001(alex) groups=1001(alex),1002(devteam)
```

- **uid**: identificativo dell'utente
- **gid**: gruppo primario, assegnato ai file che crei
- **groups**: gruppi supplementari, i permessi extra

Altri modi di verificare: `groups alex` (solo la lista), `getent passwd alex` (riga utente), `getent group devteam` (chi è membro del gruppo).

**Riga di `/etc/passwd`**: `root:x:0:0:root:/root:/bin/bash` = nome, `x` (password in shadow), UID, GID, descrizione, home, shell.

**Riga di `/etc/shadow`**: `root:<hash>:18395:0:99999:7:::`

| Campo | Significato |
| --- | --- |
| 18395 | Ultimo cambio password, in giorni dal 1° gennaio 1970 |
| 0 | Giorni minimi prima di poterla ricambiare |
| 99999 | Giorni dopo cui scade (99999 = mai) |
| 7 | Giorni di preavviso prima della scadenza |
| 8° campo | Scadenza dell'account (vuoto = mai) |

In forma leggibile: `sudo chage -l utente`.

**Hash, non cifratura.** L'hash è a senso unico: al login il sistema ri-hasha la password digitata e confronta. Formato `$6$sale$hash`, dove `$6$` è l'algoritmo (SHA-512) e il sale rende diversi gli hash di password uguali. Gli hash stanno in shadow perché passwd deve essere leggibile da tutti, e chi avesse gli hash potrebbe craccarli offline.

**`*` contro `!`**

- `*`: non c'è mai stata una password. Tipico degli account di sistema (`daemon`, `www-data`), che esistono per far girare servizi con il minimo privilegio e hanno shell `/usr/sbin/nologin`.
- `!`: account bloccato. Messo davanti a un hash vero, quindi reversibile.

## Creare utenti e gruppi

Regola d'oro: con `-G` usa sempre `-a`.

```
sudo useradd -m -s /bin/bash alex && sudo passwd alex
sudo groupadd devteam
sudo usermod -aG devteam alex
```

- `-m` crea la home, `-s /bin/bash` imposta la shell. Senza, su Debian niente home e shell `/bin/sh`.
- `&&` esegue il secondo comando solo se il primo riesce; `;` lo esegue comunque.
- `usermod -G devteam alex` (senza `-a`) **sostituisce** tutta la lista dei gruppi supplementari: alex perderebbe `docker`, `sudo` e gli altri.
- `-g` minuscola cambia il gruppo **primario**; `-G` maiuscola gestisce i **supplementari**.
- Il gruppo primario non compare tra i membri in `/etc/group`: sta solo nel campo GID di `/etc/passwd`.
- I nuovi gruppi vengono letti al login: una sessione già aperta non li vede finché non rifai login. `sudo -u` e `su -` partono da una sessione nuova, quindi li vedono subito.
- Di default `useradd` assegna l'UID/GID successivo al più alto già in uso (sopra `UID_MIN`), guardando solo passwd e group, non i file sul disco.

## Cartella condivisa

Una cartella del team si fa con gruppo dedicato, `770` e setgid.

```
sudo mkdir /srv/progetto
sudo chgrp devteam /srv/progetto
sudo chmod 2770 /srv/progetto
```

- `r=4`, `w=2`, `x=1`: `770` = proprietario tutto, gruppo tutto, altri niente.
- Su una directory `x` significa poterci **entrare**: senza, la `r` non basta.
- Meglio `chgrp devteam` che `chgrp 1002`: un nome sbagliato dà errore, un numero sbagliato no.
- **Setgid** (`chmod g+s` o la cifra `2` davanti): i nuovi file ereditano il gruppo della cartella invece del gruppo primario di chi li crea. Nel `ls -l` compare una `s`: `drwxrws---`. Vale solo per i file creati dopo.
- I permessi valgono per tutti: anche l'admin, senza `sudo`, è un utente come un altro.
- **Cancellare o rinominare un file dipende dalla cartella**, non dal file: serve `w` sulla directory. Per questo un report di root non va messo nella home di un utente, che potrebbe cancellarlo o sostituirlo anche senza poterlo leggere.
- Ogni file ha un utente proprietario **e** un gruppo proprietario, due cose separate.
- L'`umask` decide i permessi di partenza dei nuovi file (es. `rw-r--r--`): argomento successivo.

## Eseguire come un altro utente

`sudo -u` per un comando puntuale, `su -` per vedere il sistema con gli occhi di un altro.

```
sudo -u alex whoami      # chiede la TUA password
su alex -c whoami        # chiede la password di alex
su - alex                # shell completa di alex, con il suo ambiente
sudo -l                  # cosa posso fare con sudo?
```

- `su alex whoami` fallisce: bash prende `whoami` come uno script da leggere e trova un binario ("cannot execute binary file"). Con `su` il comando si passa con `-c`.
- `sudo -l` è ricognizione gratuita: `(ALL : ALL) ALL` = puoi fare tutto; "may not run sudo" = la porta principale è chiusa, serve un'altra strada.
- **History**: la tiene la shell interattiva. `sudo -u alex ...` digitato da ema finisce nella history di **ema**; i comandi scritti dentro `su - alex` finiscono in quella di alex.
- **Log**: ogni uso di sudo è registrato (`sudo grep sudo /var/log/auth.log` o `sudo journalctl _COMM=sudo`). La history l'utente può cancellarla, i log li protegge root.

## Pipe e redirect con privilegi

La `>` la esegue la tua shell, non `sudo`: per scrivere con privilegi si usa `sudo tee`.

```
# Utenti umani, file di proprietà di alex
awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd | sudo -u alex tee /home/alex/utenti.txt > /dev/null

# Account senza password utilizzabile, report solo per root
sudo awk -F: '$2 ~ /^[!*]/ {print $1}' /etc/shadow | sudo tee /root/report_account.txt > /dev/null && sudo chmod 600 /root/report_account.txt
```

- `sudo comando > file`: sudo protegge solo il comando, la scrittura del file la fa ema. Lo stesso vale per `sudo -u alex comando > file`.
- Dai privilegi **solo dove servono**: `/etc/passwd` lo legge chiunque, quindi `sudo` va solo su `tee`. `/etc/shadow` invece richiede `sudo` anche in lettura.
- Chi scrive il file ne diventa proprietario: `sudo tee` → root, `sudo -u alex tee` → alex. Altrimenti `chown` dopo.
- `tee` scrive su file **e** a schermo; `> /dev/null` zittisce lo schermo.
- `awk -F:` = separatore `:`; `$3` = terzo campo; `{print $1}` = stampa il primo. Fa filtro e taglio insieme.
- `~` in awk = "corrisponde alla regex"; `!~` = non corrisponde; `==` = uguale esatto. `^[!*]` = inizia con `!` o `*`.
- `cut -d: -f1` = delimitatore `:`, campo 1. Il delimitatore va scritto subito dopo `-d`.
- `grep -v nobody` funziona ma è fragile: meglio filtrare sul numero (nobody = UID 65534).

## Offboarding

Prima cerca i file dell'utente, poi lo elimini: dopo, il suo UID può essere riassegnato.

1. **Sospendere** subito: `sudo usermod -L -e 1 bob`
   - `-L` (come `passwd -l`) mette `!` davanti all'hash, ma le chiavi SSH funzionano ancora.
   - `-e 1` fa scadere l'account il 2 gennaio 1970: bloccato per qualsiasi metodo. Lo dice il `man passwd`.
   - Verifica: `sudo grep bob /etc/shadow` e `sudo chage -l bob`.
2. **Cercare** i suoi file in tutto il sistema: `sudo find / -user bob 2> errori.txt`
3. **Decidere** cosa farne: cancellarli o riassegnarli con `chown`.
4. **Eliminare** l'utente con la home: `sudo userdel -r bob`. Toglie solo home e posta, non i file sparsi altrove.
5. **Controllare** gli orfani: `sudo find / \( -nouser -o -nogroup \) 2> errori.txt`

Note:

- Dopo `userdel`, `ls -l` mostra numeri al posto dei nomi: i file contengono sempre UID e GID, la traduzione non trova più nulla.
- Condizioni di `find` in fila = AND; per l'OR serve `-o` tra parentesi con `\(` `\)`.
- `useradd` non controlla i file sul disco: se l'UID liberato torna disponibile (per esempio perché era il più alto), il nuovo utente può riceverlo e **ereditare** i file orfani. A quel punto `-nouser` non li trova più.
- `2>/dev/null` nasconde tutti gli errori, anche quelli che significano "non ho controllato questa cartella". In un controllo di sicurezza salva gli errori su file e leggili, oppure escludi solo il rumore noto: `sudo find / -path /proc -prune -o -nouser -print`.
- `/proc` e `/run` contengono dati di runtime, non file su disco. `/run/user/1000/gvfs` e `doc` sono montaggi dell'ambiente grafico dove neanche root entra.
- Un `[✗]` dopo `find` indica un codice di uscita d'errore, anche se gli errori erano nascosti.

## Errori fatti e lezioni

| Errore | Cosa è successo | Lezione |
| --- | --- | --- |
| `cut -d -f1` | `cut` ha preso `-f1` come delimitatore | Il delimitatore va subito dopo `-d`: `cut -d: -f1` |
| `sudo awk ... > /home/alex/...` | Permission denied | La `>` è della tua shell: usa `sudo tee` |
| `su alex whoami` | "cannot execute binary file" | Con `su` il comando va passato con `-c` |
| `usermod -L -e 2026-11-25` | Account ancora valido via chiavi SSH fino a quella data | Per disattivare subito: `-e 1` |
| `find / -nouser -nogroup` | Trova solo file con entrambi orfani | Per "uno o l'altro": `\( -nouser -o -nogroup \)` |
| `comando > sudo tee ...` | Creato un file vuoto chiamato `sudo` nella home | Tra comandi si usa `\|`, non `>` |

Abitudine da tenere: guardare, agire, verificare. E quando un risultato è "silenzioso", chiedersi cosa potrebbe nascondere.
