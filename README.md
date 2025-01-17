# Creazione Altcoin (fork bitcoin 0.20) + Mining Pool (Yiimp) Ubuntu 18.04

In questo documento è riportata la riproduzione di una rete Bitcoin e la creazione di una pool di mining sulla stessa. 
E' stata utilizzata la versione di BitcoinCore 0.20

Sono necessarie due istanze Unix: 

Creazione della mining pool e server di sviluppo - Creazione nodo secondario e Wallet destinatario

Installa la versione di Ubuntu su entrambi i server (18.04 nell'esempio). 
Per comodità installeremo Mate come interfaccia grafica e xrpd (Windows) per la connessione remota.
Qualora volessimo connetterci da OSX è consigliato utilizzare vncserver.


```bash
sudo apt update
sudo apt install ubuntu-mate-desktop
sudo apt install gnome-panel
```

per collegamento remoto da windows installa 

```bash
sudo apt install xrdp
sudo systemctl enable xrdp
```



Installa le librerie necessarie

```bash
sudo apt-get install build-essential libtool autotools-dev automake pkg-config bsdmainutils python3 libssl-dev libevent-dev 
libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-test-dev libboost-thread-dev libminiupnpc-dev libzmq3-dev 
libqt5gui5 libqt5core5a libqt5dbus5 qttools5-dev qttools5-dev-tools libprotobuf-dev protobuf-compiler libsqlite3-dev libdb++-dev
```
Installa le lib boost per non avere problemi in fase di compilazione

```bash
sudo apt-get install libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev libboost-iostreams-dev
```

Esegui un fork di BitcoinCore (0.20 nell'esempio)

```bash
sudo apt install git
git clone -b 0.20 https://github.com/bitcoin/bitcoin.git 
```

Modifica il codice sorgente di bitcoin all'interno della cartella src/

```bash
cd bitcoin/src/
```

Modifica la porta di default all'interno del file chainparams.cpp

```bash
DefaultPort = 2333;
```
Fai lo stesso per le altre porte in chainparamsbase.cpp

```bash
if (chain == CBaseChainParams::MAIN) {
        return MakeUnique<CBaseChainParams>("", 2332, 2334);
    } else if (chain == CBaseChainParams::TESTNET) {
        return MakeUnique<CBaseChainParams>("testnet3", 12332, 12334);
    } else if (chain == CBaseChainParams::SIGNET) {
        return MakeUnique<CBaseChainParams>("signet", 32332, 32334);
    } else if (chain == CBaseChainParams::REGTEST) {
        return MakeUnique<CBaseChainParams>("regtest", 12443, 12445);
    }
```


Scegliamo un messaggio di firma diverso in pchMessageStart di chainparams.cpp. 
NB: se non viene modificato e ti colleghi a un nodo bitcoin considererà la blockchain bitcoin come quella corretta. 
Fai cosi:

```bash
pchMessageStart[0] = 0xf0;
pchMessageStart[1] = 0xb0;
pchMessageStart[2] = 0xb0;
pchMessageStart[3] = 0xd0;
```

Modifichiamo anche la difficoltà di mining siccome all'inizio saremo gli unici miner della pool!

```bash
consensus.nMinimumChainWork = uint256S("0x0000000000000000000000000000000000000000000000000000000100010001");
```

Commenta tutti i vSeeds presenti, ci torneremo più tardi una volta che verrà creato il blocco genesis.
//vSeeds.emplace_back("XXX.XXX.XXX.XXX"); 
//vSeeds.emplace_back("XXX.XXX.XXX.XXX"); 


All'interno di validation.cpp puoi modificare la ricompensa per ogni blocco minato. Nel nostro caso 50

```bash
CAmount GetBlockSubsidy(int nHeight, const Consensus::Params& consensusParams)
{
    int halvings = nHeight / consensusParams.nSubsidyHalvingInterval;
    if (halvings >= 64)
        return 0;

    CAmount nSubsidy = 50 * COIN;
    //la ricompensa viene divisa a metà ogni 210,000 blocks nel caso di bitcoin ogni circa 4 anni (halving).
    nSubsidy >>= halvings;
    return nSubsidy;
}
```

# Mining del primo blocco (blocco Genesis)

git clone https://github.com/liveblockchain/genesisgen.git

Genera una chiave privata e copia la sua pubblica.

```bash
openssl ecparam -genkey -name secp256k1 -text -noout -outform DER | xxd -p -c 1000 | sed 's/41534e31204f49443a20736563703235366b310a30740201010420/PrivKey: /' | sed 's/a00706052b8104000aa144034200/\'$'\nPubKey: /'

pubkey: xxxxxxxxx
```

Scegli un messaggio di genesis esempio:

"NY Times 07/Apr/2018 More Jobs, Faster Growth and Now, the Threat of a Trade War"

Lancia il comando di genesi all'interno della repository scaricata sopra:

```bash
gcc genesis.c -o genesis -lcrypto
./genesis  <pubkey> "<timestamp>" <nBits> 0 0

$ ./genesis 048E794284AD7E4D776919BDA05CDD38447D89B436BDAF5F65EBE9D7AD3A0B084908B88162BB60B1AA5ED6542063A30FC9584A335F656A54CD9F66D6C742B67F55 "NY Times 07/Apr/2018 More Jobs, Faster Growth and Now, the Threat of a Trade War" 486604799
nBits: 0x1d00ffff
startNonce: 0
unixtime: 0

Coinbase: 04ffff001d01044c504e592054696d65732030372f4170722f32303138204d6f7265204a6f62732c204661737465722047726f77746820616e64204e6f772c2074686520546872656174206f66206120547261646520576172

PubkeyScript: 41048e794284ad7e4d776919bda05cdd38447d89b436bdaf5f65ebe9d7ad3a0b084908b88162bb60b1aa5ed6542063a30fc9584a335f656a54cd9f66d6c742b67f55ac

Merkle Hash: 63f73f6e72c8355d21b5c198406fde2480acf0263fec63dcbe7f6165d410c2c8
Byteswapped: c8c210d465617fbedc63ec3f26f0ac8024de6f4098c1b5215d35c8726e3ff763
Generating block...
939453 Hashes/s, Nonce 23980758224
Block found!
Hash: 00000000ad913538b8764573d00c3eb4a87723e11d8bd008f9125246c58e0252
Nonce: 2398108787
Unix time: 1524021159
```

Quando hai fatto inserisci dati trovati all'interno di chainparams.cpp
In CMainParams all'interno del costruttore, setta the nonce and unittime, block hash and merkle root assertions:


```bash
genesis = CreateGenesisBlock(1524021159, 2398108787, 0x1d00ffff, 1, 50 * COIN);
assert(consensus.hashGenesisBlock == uint256S("0x00000000ad913538b8764573d00c3eb4a87723e11d8bd008f9125246c58e0252"));
assert(genesis.hashMerkleRoot == uint256S("0xc8c210d465617fbedc63ec3f26f0ac8024de6f4098c1b5215d35c8726e3ff763"));
```

nella funzione static CBlock CreateGenesisBlock(uint32_t nTime, uint32_t nNonce, uint32_t nBits, int32_t nVersion, const CAmount& genesisReward), setta il timestamp creato:

```bash
const char* pszTimestamp = "NY Times 07/Apr/2018 More Jobs, Faster Growth and Now, the Threat of a Trade War";
const CScript genesisOutputScript = CScript() << ParseHex("048e794284ad7e4d776919bda05cdd38447d89b436bdaf5f65ebe9d7ad3a0b084908b88162bb60b1aa5ed6542063a30fc9584a335f656a54cd9f66d6c742b67f55") << OP_CHECKSIG;
```


# Compila Bitcoincore

Entra all'interno della copia locale del repository ed installa Berkeley DB (BDB) v.4.8 necessaria per il completo funzionamento del wallet

```bash
cd bitcoin
./contrib/install_db4.sh `pwd`
```

Prendi nota delle informazioni ottenute come risultato e lancia i comandi riportati

```bash
db4 build complete.
When compiling bitcoind, run `./configure` in the following way:
export BDB_PREFIX='<PATH-TO>/db4'
./configure BDB_LIBS="-L${BDB_PREFIX}/lib -ldb_cxx-4.8" BDB_CFLAGS="-I${BDB_PREFIX}/include"
```

```bash
./autogen.sh
./configure BDB_LIBS="-L${BDB_PREFIX}/lib -ldb_cxx-4.8" BDB_CFLAGS="-I${BDB_PREFIX}/include" 
```

Se desideri aprire bitcoin attraverso GUI il comando è il seguente:

```bash
./autogen.sh
./configure BDB_LIBS="-L${BDB_PREFIX}/lib -ldb_cxx-4.8" BDB_CFLAGS="-I${BDB_PREFIX}/include" -with-gui
```

se hai a disposizione una singola CPU

```bash
make
```

Altrimenti (farai molto più in fretta ;) )

```bash
make -j "$(($(nproc)+1))"
```

Per avviare bitcoin senza gui

```bash
./bitcoind -daemon
```

con GUI all'interno di /src/qt

```bash
./bitcoin-qt
```

# Configurazioni Aggiuntive

```bash
// modifica i BIP in modo che partano dall'1

consensus.BIP34Height = 1; 
consensus.BIP65Height = 1;
consensus.BIP65Height = 1;
```

cambia l'hash dei BIPs inserendo il tuo genesis Hash

```bash
consensus.BIP16Exception = uint256S("0x???");
consensus.BIP34Hash = uint256S("0x???");
```

Cambia la tua checkpoint data inserendo il genesis hash del blocco zero

```bash
checkpointData = {{{0, uint256S("0x???")},}};
```

Inserisci gli indirizzi dei tuoi nodi 

```bash
vSeeds.emplace_back("XXX.XXX.XXX.XXX"); Nodo 1
vSeeds.emplace_back("XXX.XXX.XXX.XXX"); Nodo 2
```


In Validation.cpp abbiamo un problema con SEGWIT. Non può funzionare al blocco 1. Quindi, all'inizio, dobbiamo disabilitare alcuni controlli, che dovranno essere riabilitati quando la blockchain avrà minato almeno 2000 blocchi.

aggiungi true; all'inizio delle seguenti funzioni

```bash
bool IsWitnessEnabled 
IsNullDummyEnabled
```
commenta 
```bash
“return state.DoS(100, false, REJECT_INVALID, "bad-cb-height", false, "block height mismatch in coinbase");”
```

nel file validation.h  :
-   cambia DEFAULT_MAX_TIP_AGE in qualcosa del tipo 60*60*24*365
-   DEFAULT_CHECKPOINTS_ENABLED = true; modificato in false o la tua coin proverà a verificare di essere sulla blockchain bitcoin.

in 

\rpc\mining.cpp :

commenta questa riga

```bash
throw JSONRPCError(RPC_INVALID_PARAMETER, "getblocktemplate must be called with the segwit rule set (call with {\"rules\": [\"segwit\"]})");
```

net_processing.cpp :
-   STALE_CHECK_INTERVAL need to be changed to a large value to avoid having a STALE error, this is again temporary, once the blockchain is mined by many people, it can be back to the original values. (3600*24*365 for example)
-   STALE_RELAY_AGE_LIMIT same for this static variable.

in \qt\bitcoingui.cpp

modifica
```bash
qint64 secs = blockDate.secsTo(currentDate);
in
qint64 secs=0;
```

# Installazione della pool (Yiimp)

Installa Yiimp sul server mining dove andremo a creare la Pool
I pool basati su YiiMP non richiedono la registrazione dell'utente.
L'indirizzo del portafoglio viene utilizzato come nome utente e tutti i pagamenti vengono effettuati in una valuta che estrai.
Il tuo saldo di mining non verrà visualizzato fino a quando non viene trovato il blocco e il tuo saldo non verrà pagato fino a quando i blocchi non saranno maturi.
Non devi prelevare manualmente i tuoi pagamenti: i pagamenti sui pool YiiMP vengono effettuati automaticamente ogni 1 ora, ogni 2 ore o ogni 3 ore a seconda delle impostazioni del pool.
I pool YiiMP supportano più monete e più algoritmi. Ma tieni presente che questo tipo di pool non supporta alcuni degli algoritmi popolari, ad esempio: Ethash, CryptoNight ed Equihash.

```bash
adduser bitPool(o quello che preferisci)
adduser bitPool sudo
su - bitPool
sudo apt -y install git
git clone https://github.com/xavatar/yiimp_install_scrypt.git
cd yiimp_install_scrypt/
bash install.sh (Non lanciare come utente root)
```

Riavvia il sistema

Installa memcache7.3


```bash
sudo apt-get install -y php7.3-memcache
```

http://xxx.xxx.xxx.xxx or /site/adminpanel.
https://xxx.xxx.xxx.xxx/site/adminpanel per accedere al pannello admin della pool.



