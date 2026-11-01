# Servizi e processi Linux

## Processi

Ogni processo ha un PID, un genitore (PPID) e un proprietario; tutti discendono da systemd, PID 1.

```
ps -p 1 -o pid,comm                 # chi è il PID 1
ps -p $$ -o pid,ppid,user,comm      # la mia shell e il suo genitore
pstree -s -p $$                     # tutta la catena in un colpo
```

La catena vista sulla mia macchina:

```
systemd(1) ─ systemd(1389) ─ konsole(2038) ─ bash(2056) ─ pstree(2411)
```

- **systemd (1)** gira come root e gestisce il sistema. Il suo PPID è 0: lo avvia il kernel.
- **systemd (1389)** è l'istanza per utente: gira come ema e gestisce la mia sessione.
- Ogni comando lanciato diventa figlio della shell, vive il tempo del suo lavoro e sparisce (`pstree` compare nel suo stesso output).
- Se un genitore muore, i figli vengono **adottati**: la catena non è sempre la storia completa di chi ha lanciato chi.
- TTY `?` = nessun terminale collegato: è il segno di un demone.
- Ogni processo gira **come un utente**, con i suoi gruppi: gli stessi permessi del modulo utenti valgono anche qui.

## Servizi con systemctl

`start` decide se il servizio gira **adesso**, `enable` se parte **al boot**: sono due interruttori indipendenti.

|  | Gira adesso | Parte al boot |
| --- | --- | --- |
| solo `start` | Sì | No |
| solo `enable` | No | Sì |
| `enable --now` | Sì | Sì |

```
systemctl status ssh
sudo systemctl start ssh
sudo systemctl stop ssh && sudo systemctl disable ssh
systemctl is-active ssh
systemctl is-enabled ssh
systemctl list-units --type=service
sudo ss -tlnp                # porte in ascolto e chi le tiene aperte
```

- `ss -tlnp` = TCP, solo in ascolto, porte numeriche, con il processo. Serve `sudo` per vedere i processi di root.
- `0.0.0.0` = in ascolto su tutte le interfacce, raggiungibile dalla rete. `127.0.0.1` = solo dalla macchina stessa.
- **Ogni servizio in ascolto è una porta aperta.** Con ssh acceso, chiunque sulla stessa rete può provare password, e gli utenti di prova hanno password deboli. Apri solo quello che usi, solo quando lo usi.
- Sui server che devono restare raggiungibili si mettono più strati: chiavi invece di password, niente login diretto di root, accesso solo a certi utenti o gruppi, firewall, blocco di chi insiste (fail2ban), VPN, aggiornamenti.

## Leggere i log

Ogni riga dice quando, dove, quale processo e cosa è successo; il PID tiene insieme le righe della stessa connessione.

```
ott 31 18:34:15 parrot sshd-session[3443]: Failed password for alex from ::1 port 51010 ssh2
└─── quando ──┘ └host┘ └ processo[PID] ─┘ └──────────── cosa è successo ──────────────┘
```

```
journalctl -u ssh --no-pager
journalctl -u ssh --no-pager | grep -E "Accepted|Failed"
```

- `sshd` resta in ascolto e crea un processo figlio (`sshd-session`) per **ogni connessione**: tre tentativi con lo stesso PID = una sola connessione.
- `Accepted password` / `Failed password`, seguiti da **chi** (`for alex`) e **da dove** (`from ::1`). Su un server esposto qui compaiono IP sconosciuti e nomi inventati.
- `::1` è localhost in IPv6, l'equivalente di `127.0.0.1`.
- La porta nel log è quella del **client**, scelta a caso per ogni connessione; la 22 è quella del server.
- `unix_chkpwd` verifica la password contro `/etc/shadow`: esiste perché shadow è leggibile solo da root.
- "Connection closed **by** authenticating user ... \[preauth\]": a chiudere è stato il client, prima di completare l'autenticazione.

## Segnali

Un utente può mandare segnali solo ai propri processi; root a tutti.

| N. | Segnale | Come si genera | Effetto |
| --- | --- | --- | --- |
| 1 | SIGHUP | Chiusura del terminale | Termina ("hang up", riattaccare) |
| 2 | SIGINT | Ctrl+C | Interrompe |
| 3 | SIGQUIT | Ctrl+\\ | Termina con core dump |
| 9 | SIGKILL | `kill -9` | Il kernel elimina il processo, senza pulizia |
| 15 | SIGTERM | `kill` (default) | "Chiuditi": il programma può fare pulizia |
| 19 | SIGSTOP | `kill -19` | Ferma, non si può ignorare |
| 20 | SIGTSTP | Ctrl+Z | Sospende |
| 21 | SIGTTIN | Lettura da terminale in background | Sospende |

```
sleep 1000 &
pgrep sleep
kill -SIGTERM <PID>        # Terminated
kill -9 <PID>              # Killed
kill %1                    # job numero 1 invece del PID
ps -ef --forest | grep sleep
```

- Ctrl+D **non è un segnale**: è la fine dell'input (EOF). `cat` termina normalmente.
- "Core dumped" = il processo ha salvato una copia della sua memoria per il debug.
- `kill` da ema su un processo di alex → "Operation not permitted". Ogni processo ha un proprietario, come i file.
- `sudo -u alex comando &`: il PID che dà la shell è quello di **sudo**, non del comando. sudo ha come utente reale chi l'ha lanciato, quindi posso ucciderlo; il figlio, nato come alex, no
- `nohup comando &` ignora SIGHUP; `disown` stacca un job dalla shell. I servizi non hanno questo problema perché li avvia systemd, non un terminale.

## Job control

Background non vuol dire silenzioso: cambia chi ha la tastiera, non dove va l'output.

```
comando &          # parte subito in background
Ctrl+Z             # sospende il job in primo piano
jobs               # elenca i job
bg                 # riprende il job in background
fg 1               # riporta il job 1 in primo piano
comando > out.txt &   # background davvero background
```

- **Primo piano**: la shell aspetta; la tastiera e Ctrl+C vanno al job.
- **Background**: la shell non aspetta; posso lanciare altri comandi, Ctrl+C non tocca il job.
- Un job in background può **scrivere** sul terminale ma non **leggere**: se ci prova riceve SIGTTIN e si sospende.

## Concatenare comandi

`;` esegue sempre, `&&` si ferma al primo errore, `|` fa partire tutti i comandi insieme collegando solo stdout.

| Esempio | Output | Perché |
| --- | --- | --- |
| `echo 1; ls NO; echo 3` | 1, errore, 3 | `;` ignora l'esito |
| `echo 1 && ls NO && echo 3` | 1, errore | `&&` si ferma |
| `ls NO \| echo ciao` | ciao + errore | La pipe non dipende dal successo |
| `ls NO \| wc -l` | errore + 0 | L'errore va su stderr, non nella pipe |
| `ls NO 2>&1 \| wc -l` | 1 | Ora anche stderr passa nella pipe |

- I comandi di una pipe partono **in parallelo**. `echo` non aspetta niente e può stampare prima dell'errore; `wc -l` deve aspettare la fine dell'input, quindi stampa per forza dopo.
- Lo stato di uscita di una pipe è quello dell'**ultimo** comando: `ls NO | echo ciao; echo $?` dà 0. Un `&&` dopo una pipe guarda solo l'ultimo.

## Imprecisioni del materiale ed errori fatti

Tre affermazioni del materiale non reggono alla prova; gli errori fatti hanno insegnato più dei comandi giusti.

| Cosa | Cosa succede davvero | Lezione |
| --- | --- | --- |
| Mio: `sudo -u alex sleep 1000 &` | sudo si sospende chiedendo la password, quella che digito va alla bash | `sudo -v` prima, controllare la history |
| Mio: `kill` sul PID dato dalla shell | Ho ucciso sudo, non lo sleep di alex | Verificare il PID giusto con `ps --forest` |
| Mio: aspettarmi che `bg` zittisse il job | Il job continua a scrivere sul terminale | Background = tastiera libera, non silenzio |

