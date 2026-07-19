# Recovery firmware da CFE — Zyxel DX5401-B0

Piattaforma: Broadcom **BCM63138**, bootloader **CFE** con set di comandi **AT-style** (non la CFE Broadcom standard: `help` non funziona, si usa `ATHE`). Flash del firmware via **seriale + TFTP**.

---

## Situazione iniziale e causa

**Il dispositivo.** Zyxel DX5401-B0 brandizzato WindTre (variante ABXA), preso usato e arrivato in **bootloop**: all'accensione si riavviava all'infinito senza completare l'avvio, tanto da sembrare guasto.

**Contesto hardware.** SoC Broadcom BCM63138, radio WiFi Broadcom BCM43684 attaccata via PCIe, storage NAND con schema **multi-immagine** (due partizioni di boot, "latest"/"previous"). Su queste unità la **corruzione della NAND** è un problema noto — lo cita esplicitamente anche il README del firmware — e può rendere difficile persino il reset di fabbrica.

**Cos'era davvero il bootloop.** Non una board morta né un'immagine OS illeggibile. Catturando il log **oltre** il countdown di autoboot si vede che l'OS bootava fino a userspace: caricava il driver WiFi `dhd` e provava a inizializzare la radio. Il fallimento era lì — la **BCM43684 non enumerava sul PCIe**:

```
dhdpcie_bus_read_pcie_ipc: PCIe IPC LOCATION FAILURE ... error -25
dhd_bus_init: PCIe IPC INITIALIZATION FAILURE
dhdpcie_pci_probe: PCIe Enumeration failed
dhdpcie_init: can't find adapter info for this chip
```

Il driver ritentava 3 volte e, non riuscendoci, **la board si resettava** → da qui il loop infinito.

**Causa di fondo.** Non il file firmware (la stessa immagine `ras-w3-config` gira perfettamente adesso), ma lo **stato low-level incoerente** del dispositivo: i parametri board/wl + nvram che servono alla radio per enumerare non erano corretti. Concorrono, verosimilmente:

- **NAND corrotta / dati di calibrazione radio danneggiati** (condizione nota su questi modelli);
- un firmware custom con componenti rimescolati (bootloader sostituito con quello dell'ABXA.1 b5, board-detect via GPIO patchato per riconoscere il DX5401-B0 Wind);
- e soprattutto il fatto che `ATUR` scrive **solo il rootFS** ("without bootloader"): flashare la sola immagine su un substrato con bootloader/parametri non coerenti **non ripara** lo stato che impediva alla radio di partire — ecco perché il loop restava anche dopo il flash.

**In sintesi:** il modem non era rotto. Aveva uno stato firmware/parametri incoerente che faceva fallire l'init del WiFi e, di conseguenza, lo mandava in reboot loop. Tutto il recovery che segue serve a **ripristinare un set low-level coerente**.

---

## Prerequisiti (lato PC)

| Cosa | Comando / valore |
|---|---|
| Console seriale | `picocom -b 115200 /dev/ttyUSB0 --logfile boot.log` |
| IP statico sulla NIC cablata | `sudo ip addr add 192.168.1.33/24 dev <iface>`  →  `sudo ip link set <iface> up` |
| Server TFTP con il firmware | vedi one-liner sotto |
| Firmware | immagine `ras*.bin` **completa** (~80 MB), messa **nella root del TFTP**, nome senza path |

> L'IP `192.168.1.33` non è arbitrario: è l'`Host IP address` che il bootloader si aspetta (verificabile con `ATBL`). Il modem è `192.168.1.1`.

Server TFTP con dnsmasq (foreground, solo TFTP, niente DNS):

```bash
sudo dnsmasq -d --port=0 --enable-tftp \
  --tftp-root=/percorso/cartella/firmware \
  --interface=<iface> --bind-interfaces
```

---

## Comandi CFE (AT) — riferimento completo

| Comando | Descrizione | Usato nel recovery |
|---|---|---|
| `ATHE` | Stampa l'elenco dei comandi disponibili (l'`help` reale). | sì — per scoprire il set comandi |
| `ATBL` | Legge bootline e parametri di board (IP host/board, MAC base, Board Id, dimensioni partizioni). | sì — verifica IP prima del flash |
| `ATSH` | Dump dei dati di produzione dalla NVRAM. | diagnostica (opzionale) |
| `ATSE` | Restituisce il **seed** del generatore di password (serve per calcolare la password di `ATEN`). | non necessario |
| `ATEN` | Imposta il **BootExtension Debug Flag** (sblocco modalità debug). Sintassi `ATEN 1` (talvolta `ATEN 1,<password>` derivata dal seed di `ATSE`). | provato — **non** sblocca comandi extra su questa build |
| `ATUR [hostip:]filename` | **Comando di flash.** Scarica via **TFTP** l'immagine e la scrive in flash, **senza toccare il bootloader**. Il modem fa da **client** TFTP: il file deve stare sul server PC. Se ometti `hostip` usa l'`Host IP` del bootline. | **sì — è il flash vero e proprio** |
| `ATGO` | Avvia il programma da flash o da host, a seconda del flag `[f/h]`. | no |
| `ATSR` | Riavvia il sistema (reboot). Non è un reset di fabbrica. | opzionale (post-flash) |
| `ATPH` | Legge/scrive i registri delle PHY. | no |
| `ATMB` | Imposta un tempo di attesa per l'ExpressTool. | no |

> **Nota importante:** su questa CFE **non esiste** un comando di "reset alle impostazioni di fabbrica". `ATSR` è solo reboot. Il reset config si fa dall'OS (tasto reset / web UI), non dal bootloader.

---

## Sequenza che ha funzionato

1. **Console seriale su** e interrompi l'autoboot premendo un tasto durante il countdown, per restare al prompt `CFE>`.
2. Verifica i parametri di rete:
   ```
   CFE> ATBL
   ```
   (conferma Host IP `192.168.1.33`, board `192.168.1.1`).
3. **Avvia il server TFTP** sul PC (one-liner dnsmasq sopra), col firmware nella `--tftp-root`.
4. **Flash** (il modem scarica e scrive):
   ```
   CFE> ATUR ras-w3-config.bin
   ```
   equivalente esplicito: `ATUR 192.168.1.33:ras-w3-config.bin`
5. Output atteso sul modem:
   ```
   Loading 192.168.1.33:ras-w3-config.bin ...
   Finished loading 84149248 bytes ...
   Correct model ID!!!
   Setting pureUBI image sequence number to 3 and committing image
   Flashing root file system ...
   Succeed to flash the rootFS.
   OK
   Resetting board ...
   ```
   Lato PC, nel log di dnsmasq deve comparire `sent .../ras-w3-config.bin to 192.168.1.1`.
6. **Non staccare la corrente** durante il flash. A fine scrittura riavvia da solo.

---

## Gotcha / lezioni

- **`ATUR` è un PULL, non un PUSH.** È il modem a scaricare via TFTP: firewall UDP/69 aperto, file nella root del server, nome **senza path**.
- **`ATUR` = "without bootloader":** scrive solo l'immagine (kernel+rootfs), **non** il bootloader. Se serve rimettere anche il bootloader coerente, serve un'immagine/procedura completa (vedi sotto).
- **Dopo il flash serve un reset di fabbrica** perché questo firmware funzioni. Su NAND corrotta il reset può richiedere molti tentativi (tasto reset premuto all'accensione, ripetuto).
- **Se resta in bootloop** anche dopo il flash: cattura il log **oltre** il countdown per vedere dove crasha. Nel nostro caso l'OS bootava ma la radio WiFi (BCM43684) non enumerava sul PCIe (`dhd` → `PCIe IPC FAILURE -25 / can't find adapter info for this chip`) e la board si resettava → loop.
- **Recovery definitivo per stato low-level incoerente / NAND ballerina:** caricare via seriale (stesso `ATUR`) il **firmware WindTre ufficiale** completo — nel nostro caso `5.17(ABXA.1)B5` — che ripristina il set coerente bootloader + parametri board/wl. Poi riflashare la config desiderata (`ras-w3-config`) **dalla web UI** con l'opzione **ripristino impostazioni di fabbrica** spuntata.
- Password root/supervisor di default di questo firmware = **numero di serie** del modem.
- **Non** ripristinare backup di config presi da firmware W3 ufficiali → mandano in bootloop.

## Link utili

1. Link al firmware di skyscreaper flashato per primo per recuperare il router: https://mega.nz/file/fdwUHIoJ#8NPjSFTabxIrYfM7B1EirfRNILbBZ4uoROfpYNbUYzE
2. Link al firmware di @handymenny che ho voluto tenerci sopra: https://mega.nz/file/dtVm3aTZ#VaJa1pEUxC-9apydHMUpQkUiAfOkRi-WJsAYAAYV1jk
3. Forum correlato: https://www.hwupgrade.it/forum/showthread.php?p=48319416#post48319416
