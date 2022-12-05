## Mina-Protocol-Testnet

## Referensi

[Dokumen resmi](https://docs.minaprotocol.com/zkapps/tutorials/zkapp-ui-with-react)

[Server Discord ](https://discord.gg/zJkqXjm8)

[Install Wallet](https://www.aurowallet.com/)

[Mina Faucet](https://faucet.minaprotocol.com/)

[Rules Reward](https://minaprotocol.com/blog/zkspark-cohort0?_hsenc=p2ANqtz-8smwqFrO-bZbm3_8-KWLkOJEV5_-yyWKkPzNswcOViTtGGAsJ2Ixg_W6Efo0kaIah9zr_wPl3trIgYeeJwCA40SGbKOQ&_hsmi=234896730)

## Persyaratan hardware & software

### Persyaratan software/OS

| Komponen | Spesifikasi minimal |
|----------|---------------------|
|Sistem Operasi|Ubuntu 16.04|

| Komponen | Spesifikasi rekomendasi |
|----------|---------------------|
|Sistem Operasi|Ubuntu 20.04|

## Bahan Awal Yang Harus di Siapkan

1 . [Install Wallet](https://www.aurowallet.com/) Buat Akun dan Save Pharse Dan Ganti Jaringan ke Berkeley

2 . [Mina Faucet](https://faucet.minaprotocol.com/) Claim Faucet Masukan Address Mina Yang Kalian Dapatkan di Wallet Yang Telah Kalian Buat

## 1. Open Port

```
ufw allow 22 && ufw allow 3000
ufw enable
```

## 2. Instal Node Js 16 & NPM & Git

```
sudo apt update
sudo apt upgrade
sudo apt install git
sudo apt install -y curl
```
```
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
```
```
sudo apt install -y nodejs
```
## 3 . Install Zkapp CLI

```
git clone https://github.com/o1-labs/zkapp-cli
```
```
npm instal -g zkapp-cli@0.5.3
```

## 4 . Create Project

```
zk project 04-zkapp-browser-ui --ui next
```

1 . pilih Yes 3x dan `Enter` Biarkan Hingga Proses Instalisasi Selesai

## 5 . Buat Repository di Github kalian

Beri Nama Samain Kasih Nama `04-zkapp-browser-ui` Setelah Selesai Membuat Repo Anda Akan Mendapatkan Command di Github Diemin, Nanti Akan di Butuhkan

Balik ke VPS Kalian Jalankan : 

```
cd 04-zkapp-browser-ui
```
```
git remote add origin <your-repo-url>
```
```
git push -u origin main
```

`<Your-Repo-Url>` = Check di Repository Yang Sudah Kalian Buat

Lalu Nanti Anda Akan di Mintai Username dan Juga Password Github Anda (Jika Terjadi Error) Anda Harus Membuat Token Acces Sebagian Pengganti Password Nya, Caranya : 

1. Pergi ke `Settingan Profil` Github Anda
2. `Depelover Setting`
3. `Personal Acces Token` > Pilih yang `Token (Classic)`
4. Lalu `Generate` dan Centang Bagian `Repo`
5. Token Acces Sudah Jadi, Itu Untuk Pengganti Password Github Anda

## 6 . Run Contract

```
cd
cd 04-zkapp-browser-ui/contracts/
npm run build
```

##  7. Membuat 2 File Baru 

```
cd
cd 04-zkapp-browser-ui/ui/pages
```

### Buat 2 File Baru Berisi :

##### Folder Pertama : 
```
nano zkappWorker.ts
```
Copy dan Paste Isian Script Ini di Terminal Vps Kalian :

```
import {
    Mina,
    isReady,
    PublicKey,
    PrivateKey,
    Field,
    fetchAccount,
} from 'snarkyjs'

type Transaction = Awaited<ReturnType<typeof Mina.transaction>>;

// ---------------------------------------------------------------------------------------

import type { Add } from '../../contracts/src/Add';

const state = {
    Add: null as null | typeof Add,
    zkapp: null as null | Add,
    transaction: null as null | Transaction,
}

// ---------------------------------------------------------------------------------------

const functions = {
    loadSnarkyJS: async (args: {}) => {
        await isReady;
    },
    setActiveInstanceToBerkeley: async (args: {}) => {
        const Berkeley = Mina.BerkeleyQANet(
            "https://proxy.berkeley.minaexplorer.com/graphql"
        );
        Mina.setActiveInstance(Berkeley);
    },
    loadContract: async (args: {}) => {
        const { Add } = await import('../../contracts/build/src/Add.js');
        state.Add = Add;
    },
    compileContract: async (args: {}) => {
        await state.Add!.compile();
    },
    fetchAccount: async (args: { publicKey58: string }) => {
        const publicKey = PublicKey.fromBase58(args.publicKey58);
        return await fetchAccount({ publicKey });
    },
    initZkappInstance: async (args: { publicKey58: string }) => {
        const publicKey = PublicKey.fromBase58(args.publicKey58);
        state.zkapp = new state.Add!(publicKey);
    },
    getNum: async (args: {}) => {
        const currentNum = await state.zkapp!.num.get();
        return JSON.stringify(currentNum.toJSON());
    },
    createUpdateTransaction: async (args: {}) => {
        const transaction = await Mina.transaction(() => {
            state.zkapp!.update();
        }
        );
        state.transaction = transaction;
    },
    proveUpdateTransaction: async (args: {}) => {
        await state.transaction!.prove();
    },
    getTransactionJSON: async (args: {}) => {
        return state.transaction!.toJSON();
    },
};

// ---------------------------------------------------------------------------------------

export type WorkerFunctions = keyof typeof functions;

export type ZkappWorkerRequest = {
    id: number,
    fn: WorkerFunctions,
    args: any
}

export type ZkappWorkerReponse = {
    id: number,
    data: any
}
if (process.browser) {
    addEventListener('message', async (event: MessageEvent<ZkappWorkerRequest>) => {
        const returnData = await functions[event.data.fn](event.data.args);

        const message: ZkappWorkerReponse = {
            id: event.data.id,
            data: returnData,
        }
        postMessage(message)
    });
}
```

Simpan `CTRL` `X` `Y` dan `Enter`

##### Folder Kedua :

```
nano zkappWorkerClient.ts
```

Copy dan Paste Isian Script Ini di Terminal Vps Kalian :

```
import {
    fetchAccount,
    PublicKey,
    PrivateKey,
    Field,
} from 'snarkyjs'

import type { ZkappWorkerRequest, ZkappWorkerReponse, WorkerFunctions } from './zkappWorker';

export default class ZkappWorkerClient {

    // ---------------------------------------------------------------------------------------

    loadSnarkyJS() {
        return this._call('loadSnarkyJS', {});
    }

    setActiveInstanceToBerkeley() {
        return this._call('setActiveInstanceToBerkeley', {});
    }

    loadContract() {
        return this._call('loadContract', {});
    }

    compileContract() {
        return this._call('compileContract', {});
    }

    fetchAccount({ publicKey }: { publicKey: PublicKey }): ReturnType<typeof fetchAccount> {
        const result = this._call('fetchAccount', { publicKey58: publicKey.toBase58() });
        return (result as ReturnType<typeof fetchAccount>);
    }

    initZkappInstance(publicKey: PublicKey) {
        return this._call('initZkappInstance', { publicKey58: publicKey.toBase58() });
    }

    async getNum(): Promise<Field> {
        const result = await this._call('getNum', {});
        return Field.fromJSON(JSON.parse(result as string));
    }

    createUpdateTransaction() {
        return this._call('createUpdateTransaction', {});
    }

    proveUpdateTransaction() {
        return this._call('proveUpdateTransaction', {});
    }

    async getTransactionJSON() {
        const result = await this._call('getTransactionJSON', {});
        return result;
    }

    // ---------------------------------------------------------------------------------------

    worker: Worker;

    promises: { [id: number]: { resolve: (res: any) => void, reject: (err: any) => void } };

    nextId: number;

    constructor() {
        this.worker = new Worker(new URL('./zkappWorker.ts', import.meta.url))
        this.promises = {};
        this.nextId = 0;

        this.worker.onmessage = (event: MessageEvent<ZkappWorkerReponse>) => {
            this.promises[event.data.id].resolve(event.data.data);
            delete this.promises[event.data.id];
        };
    }

    _call(fn: WorkerFunctions, args: any) {
        return new Promise((resolve, reject) => {
            this.promises[this.nextId] = { resolve, reject }

            const message: ZkappWorkerRequest = {
                id: this.nextId,
                fn,
                args,
            };

            this.worker.postMessage(message);

            this.nextId++;
        });
    }
}
```

Simpan `CTRL` `X` `Y` dan `Enter`

## 8 . Tambahkan Css

```
cd
cd 04-zkapp-browser-ui/ui/styles
```

Lalu

```
nano globals.css
```

Hapus Semu Isi Yang Ada di Sana dan Ganti Dengan Script di Bawah Ini :

```
html,
body {
  padding: 0;
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, Segoe UI, Roboto, Oxygen,
    Ubuntu, Cantarell, Fira Sans, Droid Sans, Helvetica Neue, sans-serif;
}

a {
  color: inherit;
  text-decoration: none;
}

* {
  box-sizing: border-box;
}

@media (prefers-color-scheme: dark) {
  html {
    color-scheme: dark;
  }
  body {
    color: white;
    background: black;
  }
}
```

Simpan `CTRL` `X` `Y` dan `Enter`
## 10 .  Running Web Buka 2 Terminal Baru (1 Perintah Masing Masing 1 Tab)

##### TAB 1
```
cd
cd 04-zkapp-browser-ui/ui/
npm run dev
```

##### TAB 2

```
cd
cd 04-zkapp-browser-ui/ui/
npm run ts-watch
```

1. Lalu Buka Browser Chrome http://ip-vps-kalian:3000
2. Pastikan Snarkjs Terbuka (Walaupun Error Abaikan Saja)
3. Lalu 2 Tab Tersebut Jika Sudah Jalan, Bisa Kalian Close Langsung Atau Kalian Bisa Menjalankan Tab Baru

## 11 . Implementing the react app

```
cd
cd 04-zkapp-browser-ui/ui/pages
nano _app.page.tsx
```

Hapus Command Yang Ada di VPS kalian Ganti Dengan Script di Bawah ini :

```

import Head from 'next/head';
import Image from 'next/image';
import styles from '../styles/Home.module.css';
import '../styles/globals.css'
import { useEffect, useState, useRef} from "react";
import './reactCOIServiceWorker';
import ZkappWorkerClient from './zkappWorkerClient';

import {
  PublicKey,
  PrivateKey,
  Field,
} from 'snarkyjs'

let transactionFee = 0.1;

export default function App() {

  let [state, setState] = useState({
    zkappWorkerClient: null as null | ZkappWorkerClient,
    hasWallet: null as null | boolean,
    hasBeenSetup: false,
    accountExists: false,
    currentNum: null as null | Field,
    publicKey: null as null | PublicKey,
    zkappPublicKey: null as null | PublicKey,
    creatingTransaction: false,
  });

// --------------------------------------------------------
// Status

  const status1 = () => {
    const el = document.getElementById("status1")!
	el.style.display = "block";
	const el2 = document.getElementById("status2")!
	el2.style.display = "none";
	const el3 = document.getElementById("status3")!
	el3.style.display = "none";
	const el4 = document.getElementById("status4")!
	el4.style.display = "none";
	const el5 = document.getElementById("status5")!
	el5.style.display = "none";
	const el6 = document.getElementById("status6")!
	el6.style.display = "none";
  }
  
   const status2 = () => {
    const el = document.getElementById("status1")!
	el.style.display = "none";
	const el2 = document.getElementById("status2")!
	el2.style.display = "block";
	const el3 = document.getElementById("status3")!
	el3.style.display = "none";
	const el4 = document.getElementById("status4")!
	el4.style.display = "none";
	const el5 = document.getElementById("status5")!
	el5.style.display = "none";
	const el6 = document.getElementById("status6")!
	el6.style.display = "none";
  }
  
   const status3 = () => {
    const el = document.getElementById("status1")!
	el.style.display = "none";
	const el2 = document.getElementById("status2")!
	el2.style.display = "none";
	const el3 = document.getElementById("status3")!
	el3.style.display = "block";
	const el4 = document.getElementById("status4")!
	el4.style.display = "none";
	const el5 = document.getElementById("status5")!
	el5.style.display = "none";
	const el6 = document.getElementById("status6")!
	el6.style.display = "none";
  }
  
   const status4 = () => {
    const el = document.getElementById("status1")!
	el.style.display = "none";
	const el2 = document.getElementById("status2")!
	el2.style.display = "none";
	const el3 = document.getElementById("status3")!
	el3.style.display = "none";
	const el4 = document.getElementById("status4")!
	el4.style.display = "block";
	const el5 = document.getElementById("status5")!
	el5.style.display = "none";
	const el6 = document.getElementById("status6")!
	el6.style.display = "none";
  }
  
   const status5 = () => {
    const el = document.getElementById("status1")!
	el.style.display = "none";
	const el2 = document.getElementById("status2")!
	el2.style.display = "none";
	const el3 = document.getElementById("status3")!
	el3.style.display = "none";
	const el4 = document.getElementById("status4")!
	el4.style.display = "none";
	const el5 = document.getElementById("status5")!
	el5.style.display = "block";
	const el6 = document.getElementById("status6")!
	el6.style.display = "none";
  }
  
   const status6 = () => {
    const el = document.getElementById("status1")!
	el.style.display = "none";
	const el2 = document.getElementById("status2")!
	el2.style.display = "none";
	const el3 = document.getElementById("status3")!
	el3.style.display = "none";
	const el4 = document.getElementById("status4")!
	el4.style.display = "none";
	const el5 = document.getElementById("status5")!
	el5.style.display = "none";
	const el6 = document.getElementById("status6")!
	el6.style.display = "block";
  }
  
   const send1 = () => {
    const el = document.getElementById("send1")!
	el.style.display = "block";
	const el2 = document.getElementById("send2")!
	el2.style.display = "none";
	const el3 = document.getElementById("send3")!
	el3.style.display = "none";
	const el4 = document.getElementById("send4")!
	el4.style.display = "none";
  }
  
  const send2 = () => {
    const el = document.getElementById("send1")!
	el.style.display = "none";
	const el2 = document.getElementById("send2")!
	el2.style.display = "block";
	const el3 = document.getElementById("send3")!
	el3.style.display = "none";
	const el4 = document.getElementById("send4")!
	el4.style.display = "none";
  }
  
  const send3 = () => {
    const el = document.getElementById("send1")!
	el.style.display = "none";
	const el2 = document.getElementById("send2")!
	el2.style.display = "none";
	const el3 = document.getElementById("send3")!
	el3.style.display = "block";
	const el4 = document.getElementById("send4")!
	el4.style.display = "none";
  }
  
  const send4 = () => {
    const el = document.getElementById("send1")!
	el.style.display = "none";
	const el2 = document.getElementById("send2")!
	el2.style.display = "none";
	const el3 = document.getElementById("send3")!
	el3.style.display = "none";
	const el4 = document.getElementById("send4")!
	el4.style.display = "block";
  }
  
  const donesend = () => {
    const el = document.getElementById("send1")!
	el.style.display = "none";
	const el2 = document.getElementById("send2")!
	el2.style.display = "none";
	const el3 = document.getElementById("send3")!
	el3.style.display = "none";
	const el4 = document.getElementById("send4")!
	el4.style.display = "none";
	const el5 = document.getElementById("sendLoading")!
	el5.style.display = "none";
	const el6 = document.getElementById("sendDone")!
	el6.style.display = "block";
	const el7 = document.getElementById("sendCheck")!
	el7.style.display = "block";
	const el8 = document.getElementById("closeSend")!
	el8.style.display = "block";
  }
	
  // -------------------------------------------------------
  // Do Setup
  const connectWallet = async () => {
      if (!state.hasBeenSetup) {
        const zkappWorkerClient = new ZkappWorkerClient();
        
        console.log('Loading SnarkyJS...');
		status1();
        await zkappWorkerClient.loadSnarkyJS();
        console.log('done');

        await zkappWorkerClient.setActiveInstanceToBerkeley();

        const mina = (window as any).mina;
		status2();

        if (mina == null) {
          setState({ ...state, hasWallet: false });
		  return;
        }

        const publicKeyBase58 : string = (await mina.requestAccounts())[0];
        const publicKey = PublicKey.fromBase58(publicKeyBase58);

        console.log('using key', publicKey.toBase58());

        console.log('checking if account exists...');
        const res = await zkappWorkerClient.fetchAccount({ publicKey: publicKey! });
        const accountExists = res.error == null;

        await zkappWorkerClient.loadContract();

        console.log('compiling zkApp');
		status3();
        await zkappWorkerClient.compileContract();
        console.log('zkApp compiled');

        const zkappPublicKey = PublicKey.fromBase58('B62qrDe16LotjQhPRMwG12xZ8Yf5ES8ehNzZ25toJV28tE9FmeGq23A');

        await zkappWorkerClient.initZkappInstance(zkappPublicKey);

        console.log('getting zkApp state...');
        await zkappWorkerClient.fetchAccount({ publicKey: zkappPublicKey })
        const currentNum = await zkappWorkerClient.getNum();
        console.log('current state:', currentNum.toString());

        setState({ 
            ...state, 
            zkappWorkerClient, 
            hasWallet: true,
            hasBeenSetup: true, 
            publicKey, 
            zkappPublicKey, 
            accountExists, 
            currentNum
        });
      }
  };

  // -------------------------------------------------------
  // Newwwwww
  const connectBtnclick = () => {
    const el = document.getElementById("loadingBtn")!
	el.style.display = "block";
	const el2 = document.getElementById("connectBtn")!
	el2.style.display = "none";
	const el3 = document.getElementById("loading")!
	el3.style.display = "block";
	const el4 = document.getElementById("banner")!
	el4.style.display = "none";
	const el5 = document.getElementById("banner2")!
	el5.style.display = "none";
  };
  
  const hideloadingBtn = () => {
    const el = document.getElementById("loadingBtn")!
	el.style.display = "none";
	const el2 = document.getElementById("loading")!
	el2.style.display = "none";
	const el3 = document.getElementById("succes")!
	el3.style.display = "block";
  };
  
  const closeGetclick = () => {
    const el = document.getElementById("getscreen")!
	el.style.display = "none";
	const el2 = document.getElementById("getBtnDisable")!
	el2.style.display = "none";
	const el3 = document.getElementById("getBtn")!
	el3.style.display = "block";
	const el4 = document.getElementById("sendBtnDisable")!
	el4.style.display = "none";
	const el5 = document.getElementById("sendBtn")!
	el5.style.display = "block";
  };
  
  const closeSendclick = () => {
    const el = document.getElementById("sendscreen")!
	el.style.display = "none";
	const el2 = document.getElementById("sendBtnDisable")!
	el2.style.display = "none";
	const el3 = document.getElementById("sendBtn")!
	el3.style.display = "block";
	const el4 = document.getElementById("getBtnDisable")!
	el4.style.display = "none";
	const el5 = document.getElementById("getBtn")!
	el5.style.display = "block";
	const el6 = document.getElementById("sendDone")!
	el6.style.display = "none";
    const el7 = document.getElementById("sendCheck")!
	el7.style.display = "none";
    const el8 = document.getElementById("closeSend")!
	el8.style.display = "none";
  };
  
  const getscreenShow = () => {
	const el = document.getElementById("getscreen")!
	el.style.display = "block";
	const el2 = document.getElementById("getBtn")!
	el2.style.display = "none";
	const el3 = document.getElementById("getBtnDisable")!
	el3.style.display = "block";
	const el4 = document.getElementById("sendBtnDisable")!
	el4.style.display = "block";
	const el5 = document.getElementById("sendBtn")!
	el5.style.display = "none";
  };
  
  const sendscreenShow = () => {
	const el = document.getElementById("sendscreen")!
	el.style.display = "block";
	const el2 = document.getElementById("sendBtn")!
	el2.style.display = "none";
	const el3 = document.getElementById("sendBtnDisable")!
	el3.style.display = "block";
	const el4 = document.getElementById("getBtnDisable")!
	el4.style.display = "block";
	const el5 = document.getElementById("getBtn")!
	el5.style.display = "none";
    const el6 = document.getElementById("sendLoading")!
	el6.style.display = "block";
  };
  
  const noAccount = () => {
    const el = document.getElementById("loadingBtn")!
	el.style.display = "none";
    const el2 = document.getElementById("caution")!
	el2.style.display = "block";
    const el3 = document.getElementById("ftxt")!
	el3.style.display = "block";
	const el4 = document.getElementById("backNoAccount")!
	el4.style.display = "block";
    const el5 = document.getElementById("loading")!
	el5.style.display = "none";
  }
  
  const backNoAccountClick = () => {
    location.reload();
  }
  
  const backNoWalletClick = () => {
    location.reload();;
  }
  
  const noWallet = () => {
    const el = document.getElementById("loadingBtn")!
	el.style.display = "none";
    const el2 = document.getElementById("caution")!
	el2.style.display = "block";
    const el3 = document.getElementById("walletTxt")!
	el3.style.display = "block";
	const el4 = document.getElementById("backNoWallet")!
	el4.style.display = "block";
    const el5 = document.getElementById("loading")!
	el5.style.display = "none";
  }
  
  const getload = () => {
    const el = document.getElementById("getLoading")!
	el.style.display = "block";
	const el2 = document.getElementById("gettext")!
	el2.style.display = "none";
	const el3 = document.getElementById("getnumber")!
	el3.style.display = "none";
  }
  
  const getdone = () => {
    const el = document.getElementById("getLoading")!
	el.style.display = "none";
	const el2 = document.getElementById("gettext")!
	el2.style.display = "block";
	const el3 = document.getElementById("getnumber")!
	el3.style.display = "block";
  }
  // -------------------------------------------------------
 
   // -------------------------------------------------------
  // Send a transaction

  const onSendTransaction = async () => {
    setState({ ...state, creatingTransaction: true });
	send1();
    console.log('sending a transaction...');

    await state.zkappWorkerClient!.fetchAccount({ publicKey: state.publicKey! });

    await state.zkappWorkerClient!.createUpdateTransaction();

    console.log('creating proof...');
	send2();
    await state.zkappWorkerClient!.proveUpdateTransaction();

    console.log('getting Transaction JSON...');
	send3();
    const transactionJSON = await state.zkappWorkerClient!.getTransactionJSON()

    console.log('requesting send transaction...');
	send4();
    const { hash } = await (window as any).mina.sendTransaction({
      transaction: transactionJSON,
      feePayer: {
        fee: transactionFee,
        memo: '',
      },
    });

    console.log(
      'See transaction at https://berkeley.minaexplorer.com/transaction/' + hash
    );

	donesend();

    setState({ ...state, creatingTransaction: false });
  }
  
   // -------------------------------------------------------
  // Refresh the current state

  const onRefreshCurrentNum = async () => {
    console.log('getting zkApp state...');
	getload();
    await state.zkappWorkerClient!.fetchAccount({ publicKey: state.zkappPublicKey! })
    const currentNum = await state.zkappWorkerClient!.getNum();
    console.log('current state:', currentNum.toString());
	getdone();

    setState({ ...state, currentNum });
  }
 
  let hasWallet;
  if (state.hasWallet != null && !state.hasWallet) {
    const auroLink = 'https://www.aurowallet.com/';
    hasWallet = <a id="walletLink" style={{display: 'block'}} href={auroLink} target="_blank" rel="noreferrer">
		<h1 className={styles.walletLink}>[[CLICK HERE]]</h1>
	</a>
	status4();
	noWallet();

  }

  let setupText = state.hasBeenSetup ? 'SnarkyJS Ready' : 'Loading...';
  let setup = <div id="setup" style={{display: 'block'}}> { setupText } { hasWallet }</div>
  
  let accountDoesNotExist;
  if (state.hasBeenSetup && !state.accountExists) {
	  const faucetLink = "https://faucet.minaprotocol.com/?address=" + state.publicKey!.toBase58();
	accountDoesNotExist = <a id="flink" style={{display: 'block'}} href={faucetLink} target="_blank" rel="noreferrer">
		<h1 className={styles.faucetHere}>[[CLICK HERE]]</h1>
	</a>
	status5();
	noAccount();
	hasBeenSetup: false;
  }
  
  let mainContent;
  if (state.hasBeenSetup && state.accountExists) {	
    mainContent =
		<div>
			<a id="sendBtn" style={{display: 'block'}} onClick={() => {onSendTransaction(); sendscreenShow(); }}>
					<span className={styles.sendBtn}> </span>
			</a>
			
			<a id="getBtn" style={{display: 'block'}} onClick={() => {onRefreshCurrentNum(); getscreenShow(); }}>
					<span className={styles.getBtn}></span>
			</a>
			
			<span id="getBtnDisable" style={{display: 'none'}} className={styles.getBtnDisable}></span>
			<span id="sendBtnDisable" style={{display: 'none'}} className={styles.sendBtnDisable}></span>
			
			<h1 className={styles.txtAddrs}>{  state.publicKey!.toBase58() } </h1>
			<h1 className={styles.addrs}>Address :</h1>
			
			<div id="getscreen" style={{display: 'none'}} className={styles.getscreen}>
				<span className={styles.getscreenBlack}> </span>
				<span className={styles.getscreenImg}> </span>
				
				<a id="closeGet" style={{display: 'block'}} onClick={() => {closeGetclick(); }}>
					<span className={styles.closeGet}> </span>
				</a>
				
				<span id="getLoading" style={{display: 'none'}} className={styles.getLoading}> </span>
				
				<h1 id="gettext" style={{display: 'none'}} className={styles.txtState}>Current Number in ZkApp :</h1>
				<h1 id="getnumber" style={{display: 'none'}} className={styles.numState}>{ state.currentNum!.toString() } </h1>
			</div>
			
			<div id="sendscreen" style={{display: 'none'}} className={styles.sendscreen}>
				<span className={styles.sendscreenBlack}> </span>
				<span className={styles.sendscreenImg}> </span>
				
				<a id="closeSend" style={{display: 'none'}} onClick={() => {closeSendclick(); }}>
					<span className={styles.closeSend}> </span>
				</a>
				
				<span id="sendLoading" style={{display: 'none'}} className={styles.sendLoading}> </span>
				<span id="sendCheck" style={{display: 'none'}} className={styles.sendCheck}> </span>
				<span id="sendDone" style={{display: 'none'}} className={styles.sendDone}> </span>
				
				<h1 id="send1" style={{display: 'none'}} className={styles.statusSendTxt}>Sending a Transaction...</h1>
				<h1 id="send2" style={{display: 'none'}} className={styles.statusSendTxt}>Creating Proof...</h1>
				<h1 id="send3" style={{display: 'none'}} className={styles.statusSendTxt}>Getting Transaction JSON...</h1>
				<h1 id="send4" style={{display: 'none'}} className={styles.statusSendTxt}>Requesting Send Transaction...</h1>


			</div>

		</div>
	hideloadingBtn();
	status6();
  }
	
  return (
	<div className={styles.container}>	
	  <Head>
        <title>zkApp [<username kalian>]</title>
        <meta name="description" content="ZkApp By <username kalian>" />
		<meta name="viewport" content="width=1280, initial-scale=2.0"/>
        <link href="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAYAAABzenr0AAAABmJLR0QA/wD/AP+gvaeTAAAEvklEQVRYhe2XW2wUZRTHf2d2drfb2gu10HZbYzA1kQfLRYyghEcgJNJYQZ+KJMYE30x8MkYSQ3zyyUBM5EVDeRBDwGJMiDFivGAhhCCmmCpUjZRipbW7FHZ3Lt/xYfYysxfw+qQnmezsfOc7//93zv878w38103+yiSdTHehVhqhHzzF2FdAr8qK6bl/jYBe6h/AZwR0CGUlKCjBVb4351EZAzMqD85e/kcI6NTyblxnD/AcEA+AtDioVQTK9y6GtxD2yspfZv8yAf2+bz1GjoL21AeuR6LkA6DXMTwpa379rBGGdRvwpzCcRLUnAKkAGB88T1ETXJ4XPAv7oApGu0A/0rOdO/4UAf02/Sg+B1GSQaBKUOMpWdmEs2wfmXwPmUIap3s/WWszxtcKeIVMEuSQnunY+IcI6NSybiw5imoyGkwxvuK6SnLpMM0925HUGiS1mube7SSXDmNKREOEMQqqCbAO67m7lt45A469B6PdlcnFtHuQlU2YvgMk2tcCkOrbTapvNwB2ywCLPEQ+r1WlKJPpMa79SjVcRIQ60T+AmIugNWr3PHB7D5Dq3lYvk2XLzRxFp56nOeFRFiagARlHbF0hD2emSv52lI7ZiWq8/h7XiOvCN88ixehiLyHV9wzxtkFSvcNk5j5Fbx5CpAxcipPA0RHg1VKcqhKYoQY1rFyliQtjtLvHaXeO07r4DjcndpXH4ks24vlaBI+WRJWhMGKZgE6muzAMVgsvUsuw1dTZhMac4FlIQ6GFrNJTbZ21JfDcfowVDU5jAqZtGxkJfMTuouWeUgYU59oRmiSUsaieBM/0A/NRAn6st7yKep2uikTHqrepMfX47cILpBY/QRLVnTH0a0gDF6IE1FdU6k8oB6owyM28X3axYs0k7l6PZbfSOvAiN86eImEuBVusbhYqgUIZkBkiaavKQqjEAIWLu2hpkpKwyJh76djwFXbLclIrXseZeIKk3eA9EZOrpTiVXeDmrjRoIDU7AIIGYluKbSnxmNKsP1KYGwcg2bkO1wnNM6GYRpVcbLqGgDxyYw5jzkcVS1nJWlxppWSVoGoU1xdiTelgyM8HPaLeIlTPyZbsfG0JANQaA7MqnDaNvl7LrqZjG1kp/pcUyft3EG97AID8tRMkYqWyRZuawFgYsqoTuqMY6yWURA1wVIMsWXuQeuYsfI373cukbOroSR2MHAr7RzqhrM1cRjmg1RowioVSmH6PW9PH8HNBCZ250zhzpwFwMxMsjD+NM76R1ths3Wamhjdlc/6HhgQA8Ny9qF6rFqQlSlv+Q6zJnRTmA9Dc1H5yU/sDAtlJWhY/oCXhImEhVzrgjOUUXquGs6sfyGOLs/pl67AiJ1GS4fRbosQtyP58GHwHzZ5BFW799C6F6WMkJZLu8NbLi6/D8jjXa/BqMlA0/bx9h2JGoXgqCgU1RjEGYkVA3wTkgnRGRYeSF8OIbHWO1MO5/aH0i7Z16ptjlA6lNFxhbdsNbmfE6LBsdccbYTQ8lALIhuy4CIOqui9QcMPmEhEsiqNG3xDXGbwd+B0zEDb9uOk+hBFFhlBd3eB74JwYM0bMHq1W+98mECFzgk5iiT6M9APg6RVizrRsYf4OU/+3GvsdPD5jAu/D1tEAAAAASUVORK5CYII=" rel="icon" type="image/x-icon" />
	  </Head>
	  
		<main className={styles.main}>
		
			<div id="homepage" className={styles.homepage}>
				<span className={styles.homepageImg}> </span>
				<a id="connectBtn" style={{display: 'block'}} onClick={() => {connectBtnclick(); connectWallet();}}>
					<span className={styles.connectBtn}> </span>
				</a>
				
					<span id="loadingBtn" style={{display: 'none'}} className={styles.loadingBtn}> </span>
					<span id="loading" style={{display: 'none'}} className={styles.loading}> </span>
					
					<h1 id="status1" style={{display: 'none'}} className={styles.statusTxt}>Status : Sync & Checking Wallet ...</h1>
					<h1 id="status2" style={{display: 'none'}} className={styles.statusTxt}>Status : Sync DONE! Connect to Wallet...</h1>
					<h1 id="status3" style={{display: 'none'}} className={styles.statusTxt}>Status : Checking & Validation Address...</h1>
					<h1 id="status4" style={{display: 'none'}} className={styles.statusTxt}>Status : Wallet Extension Not Found!</h1>
					<h1 id="status5" style={{display: 'none'}} className={styles.statusTxt}>Status : Address Not Valid or No Balance!</h1>
					<h1 id="status6" style={{display: 'none'}} className={styles.statusTxt}>Status : READY FOR TRANSACTION!!!</h1>
					
					<span id="caution" style={{display: 'none'}} className={styles.caution}> </span>
					
					<span id="succes" style={{display: 'none'}} className={styles.succes}> </span>

					<h1 id="ftxt" style={{display: 'none'}} className={styles.faucetTxt}>Invalid Account or No Balance!! Please check and fund on this link </h1>
					
					<h1 id="walletTxt" style={{display: 'none'}} className={styles.walletTxt}>Could not find a wallet. Please Install Auro wallet and Re-Connect!! </h1>
					
					<a id="backNoAccount" style={{display: 'none'}} onClick={() => {backNoAccountClick(); }}>
						<span className={styles.backNoAccount}> </span>
					</a>
					
					<a id="backNoWallet" style={{display: 'none'}} onClick={() => {backNoWalletClick(); }}>
						<span className={styles.backNoWallet}> </span>
					</a>
					
					<span id="banner" style={{display: 'block'}} className={styles.banner}> </span>
					<span id="banner2" style={{display: 'block'}} className={styles.banner2}> </span>
					
				{mainContent}
				{accountDoesNotExist}
				{hasWallet}
				
			<div id="footer" style={{display: 'block'}} >	
				<span id="footerbg" style={{display: 'block'}} className={styles.footerbg}> </span>
				<a style={{display: 'block'}} href="<link tele kalian>" target="_blank" rel="noopener noreferrer" >
						<span className={styles.teleIcon}> </span>
					</a>
					
				<a style={{display: 'block'}} href="<link discord kalian>" target="_blank" rel="noopener noreferrer" >
						<span className={styles.dcIcon}> </span>
					</a>
					
				<a style={{display: 'block'}} href="<link github kalian>" target="_blank" rel="noopener noreferrer" >
						<span className={styles.gitIcon}> </span>
					</a>
			</div>
					
			</div>
		</main>
		
	<footer className={styles.footer}>
		</footer>


	</div>
  );
}
```
Username diganti dengan username kalian dan link sosmed diganti dengan link sosmed kalian..

Buat Folder images
```
cd 04-zkapp-browser-ui/ui
mkdir images
```
Masukan beberapa gambar yang akan dibuat untuk web nya. Pastikan rename namanya dan sesuaikan dengan script css nya
Edit Home.module.css
```
cd ui/styles
nano Home.module.css
```
Lalu hapus semua isinya dan ganti dengan ini
```
.container {
  padding: 0 2rem;
}

.main {
  min-height: 100vh;
  padding: 1rem 0;
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;

  
}

.footer {
  display: flex;

  
}

.footerbg {
	display: block;
	background-image: url("../images/footer.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 520px;
	height: 145px;
	position: absolute;
	left: 200px;
	top: 560px;
}

.blank {
	display: block;
	background-image: url("../images/footer.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 520px;
	height: 145px;
	position: absolute;
	left: 200px;
	top: 600px;
}

.title a {
  color: #0070f3;
  text-decoration: none;
}

.title a:hover,
.title a:focus,
.title a:active {
  text-decoration: underline;
}

.title {
  margin: 0;
  line-height: 1.15;
  font-size: 4rem;
}

.title,
.description {
  text-align: center;
}

.description {
  margin: 4rem 0;
  line-height: 1.5;
  font-size: 1.5rem;
}

.code {
  background: #fafafa;
  border-radius: 5px;
  padding: 0.75rem;
  font-size: 1.1rem;
  font-family: Menlo, Monaco, Lucida Console, Liberation Mono, DejaVu Sans Mono,
    Bitstream Vera Sans Mono, Courier New, monospace;
}

.grid {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-wrap: wrap;
  max-width: 800px;
}

.card {
  margin: 1rem;
  padding: 1.5rem;
  text-align: left;
  color: inherit;
  text-decoration: none;
  border: 1px solid #eaeaea;
  border-radius: 10px;
  transition: color 0.15s ease, border-color 0.15s ease;
  max-width: 300px;
}

.card:hover,
.card:focus,
.card:active {
  color: #0070f3;
  border-color: #0070f3;
}

.card h2 {
  margin: 0 0 1rem 0;
  font-size: 1.5rem;
}

.card p {
  margin: 0;
  font-size: 1.25rem;
  line-height: 1.5;
}

.logo {
	display: flex;
	background-image: url("../images/icon.png");
	height: 50px;
	width: 50px;
	background-size: cover;
	background-repeat: no-repeat;
	align-items: center;
	justify-content: center;
	margin-left: 85px;

}

.homepage {
	display: block;
	display: flex;
	align-items: center;
	justify-content: center;
	margin-left: auto;
	margin-right: auto;
	position: relative;	
	width: 900px;
	height: 600px;
	transform: scale(1);
}

.homepageImg {
	display: block;
	display: flex;
	background-size: cover;
	align-items: center;
	justify-content: center;
	margin-left: auto;
	margin-right: auto;
	background-image: url("../images/homepageImg.png");
	background-repeat: no-repeat;
	position: absolute;
	width: 608px;
	height: 500px;
}

.banner {
	display: block;
	background-image: url("../images/banner.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 420px;
	height: 142px;
	position: absolute;
	left: 250px;
	top: 150px;
}

.banner2 {
	display: block;
	background-image: url("../images/banner2.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 250px;
	height: 70px;
	position: absolute;
	left: 335px;
	top: 310px;
}

.connectBtn {
	display: block;
	background-image: url("../images/connectBtn1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 380px;
	top: 450px;
}

.connectBtn:hover {
	display: block;
	background-image: url("../images/connectBtn2.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 380px;
	top: 450px;
}

.connectBtn:active {
	display: block;
	background-image: url("../images/connectBtn3.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 380px;
	top: 450px;
}

.loadingBtn {
	display: block;
	background-image: url("../images/loadingBtn.gif");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 380px;
	top: 450px;
}

.loading {
	display: block;
	background-image: url("../images/loading.gif");
	background-repeat: no-repeat;
	background-size: cover;
	width: 300px;
	height: 300px;
	position: absolute;
	left: 305px;
	top: 50px;
}

.getLoading {
	display: block;
	background-image: url("../images/getLoading.gif");
	background-repeat: no-repeat;
	background-size: cover;
	width: 280px;
	height: 280px;
	position: absolute;
	left: 315px;
	top: 150px;
}

.sendLoading {
	display: block;
	background-image: url("../images/getLoading.gif");
	background-repeat: no-repeat;
	background-size: cover;
	width: 250px;
	height: 250px;
	position: absolute;
	left: 330px;
	top: 110px;
}

.sendCheck {
	display: block;
	background-image: url("../images/check.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 43px;
	height: 35px;
	position: absolute;
	left: 295px;
	top: 325px;
}

.sendDone {
	display: block;
	background-image: url("../images/senddone.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 250px;
	height: 120px;
	position: absolute;
	left: 330px;
	top: 170px;
}

.caution {
	display: block;
	background-image: url("../images/caution.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 180px;
	height: 155px;
	position: absolute;
	left: 370px;
	top: 120px;
}

.succes {
	display: block;
	background-image: url("../images/succes.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 180px;
	height: 180px;
	position: absolute;
	left: 370px;
	top: 120px;
}

.statusTxt {
	display: block;
	position: absolute;
	left: 300px;
	top: 300px;
	color: white;
	font-size: 15px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;

}

.statusSendTxt {
	display: block;
	position: absolute;
	left: 340px;
	top: 320px;
	color: white;
	font-size: 18px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;

}

.walletLink {
	display: block;
	position: absolute;
	left: 390px;
	top: 370px;
	color: white;
	font-size: 20px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;
}

.walletTxt {
	display: block;
	position: absolute;
	left: 210px;
	top: 350px;
	color: white;
	font-size: 15px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;
}

.faucetTxt {
	display: block;
	position: absolute;
	left: 215px;
	top: 350px;
	color: white;
	font-size: 15px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;
}

.faucetHere {
	display: block;
	position: absolute;
	left: 390px;
	top: 370px;
	color: white;
	font-size: 20px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;
}

.backNoAccount {
	display: block;
	background-image: url("../images/backBtn1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 50px;
	height: 52px;
	position: absolute;
	left: 215px;
	top: 110px;
}

.backNoAccount:hover {
	display: block;
	background-image: url("../images/backBtn2.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 50px;
	height: 52px;
	position: absolute;
	left: 215px;
	top: 110px;
}

.backNoWallet {
	display: block;
	background-image: url("../images/backBtn1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 50px;
	height: 52px;
	position: absolute;
	left: 215px;
	top: 110px;
}

.backNoWallet:hover {
	display: block;
	background-image: url("../images/backBtn2.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 50px;
	height: 52px;
	position: absolute;
	left: 215px;
	top: 110px;
}

.sendBtn {
	display: block;
	background-image: url("../images/sendBtn1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 250px;
	top: 400px;
}

.sendBtn:hover {
	display: block;
	background-image: url("../images/sendBtn2.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 250px;
	top: 400px;
}

.sendBtn:active {
	display: block;
	background-image: url("../images/sendBtn3.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 250px;
	top: 400px;
}

.sendBtnDisable {
	display: block;
	background-image: url("../images/sendBtn3.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 250px;
	top: 400px;
}

.getBtn {
	display: block;
	background-image: url("../images/getBtn1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 510px;
	top: 400px;
}

.getBtn:hover {
	display: block;
	background-image: url("../images/getBtn2.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 510px;
	top: 400px;
}

.getBtn:active {
	display: block;
	background-image: url("../images/getBtn3.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 510px;
	top: 400px;
}

.getBtnDisable {
	display: block;
	background-image: url("../images/getBtn3.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 160px;
	height: 73px;
	position: absolute;
	left: 510px;
	top: 400px;
}

.txtState {
	display: block;
	position: absolute;
	left: 330px;
	top: 210px;
	color: white;
	font-size: 20px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;

}

.numState {
	display: block;
	position: absolute;
	left: 360px;
	top: 200px;
	color: white;
	font-size: 80px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;

}

.txtAddrs {
	display: block;
	position: absolute;
	left: 285px;
	top: 350px;
	color: white;
	font-size: 12px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;

}

.addrs {
	display: block;
	position: absolute;
	left: 220px;
	top: 350px;
	color: white;
	font-size: 13px;
	text-align: center;
    text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;

}

.getscreen{
	display: block;
	display: flex;
	align-items: center;
	justify-content: center;
	margin-left: auto;
	margin-right: auto;
	position: relative;	
	width: 900px;
	height: 600px;
}

.getscreenBlack {
	display: block;
	display: flex;
	background-image: url("../images/black.png");
	background-size: repeat;
	width: 100%;
	height: 100%;
	left: 0px;
	top: 0px;
	position: absolute;
	align-items: center;
	justify-content: center;
	margin-left: auto;
	margin-right: auto;
	opacity: 0.5;
	-webkit-transform: scale(1.5);
    transform: scale(1.5); /* you can change the value also */
   -webkit-transform-origin: center;
    transform-origin: center;
}

.getscreenImg {
	display: block;
	display: flex;
	background-image: url("../images/screen.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 380px;
	height: 280px;
	position: absolute;
	left: 260px;
	top: 150px;
}

.sendscreen{
	display: block;
	display: flex;
	align-items: center;
	justify-content: center;
	margin-left: auto;
	margin-right: auto;
	position: relative;	
	width: 900px;
	height: 600px;
}

.sendscreenBlack {
	display: block;
	display: flex;
	background-image: url("../images/black.png");
	background-size: repeat;
	width: 100%;
	height: 100%;
	left: 0px;
	top: 0px;
	position: absolute;
	align-items: center;
	justify-content: center;
	margin-left: auto;
	margin-right: auto;
	opacity: 0.5;
	-webkit-transform: scale(1.5);
    transform: scale(1.5); /* you can change the value also */
   -webkit-transform-origin: center;
    transform-origin: center;
}

.sendscreenImg {
	display: block;
	display: flex;
	background-image: url("../images/send.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 400px;
	height: 330px;
	position: absolute;
	left: 250px;
	top: 100px;
}


.closeGet {
	display: block;
	background-image: url("../images/closeBtn1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 50px;
	height: 52px;
	position: absolute;
	left: 590px;
	top: 150px;
}

.closeGet:hover {
	display: block;
	background-image: url("../images/closeBtn2.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 50px;
	height: 52px;
	position: absolute;
	left: 590px;
	top: 150px;
}

.closeSend {
	display: block;
	background-image: url("../images/closeBtn1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 50px;
	height: 52px;
	position: absolute;
	left: 600px;
	top: 120px;
}

.closeSend:hover {
	display: block;
	background-image: url("../images/closeBtn2.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 50px;
	height: 52px;
	position: absolute;
	left: 600px;
	top: 120px;
}

.teleIcon {
	display: block;
	background-image: url("../images/teleicon1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 90px;
	height: 70px;
	position: absolute;
	left: 280px;
	top: 600px;
}

.dcIcon {
	display: block;
	background-image: url("../images/dcicon1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 90px;
	height: 70px;
	position: absolute;
	left: 380px;
	top: 600px;
}

.fbIcon {
	display: block;
	background-image: url("../images/fbicon1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 90px;
	height: 70px;
	position: absolute;
	left: 480px;
	top: 600px;
}

.gitIcon {
	display: block;
	background-image: url("../images/giticon1.png");
	background-repeat: no-repeat;
	background-size: cover;
	width: 90px;
	height: 70px;
	position: absolute;
	left: 580px;
	top: 600px;
}


@media (max-width: 600px) {
  .grid {
    width: 100%;
    flex-direction: column;
  }
}

@media (prefers-color-scheme: dark) {
  .card,
  .footer {
    border-color: #222;
  }
  .code {
    background: #111;
  }
  .logo img {
  }
   
  
}
```
## PASTIKAN KALIAN EDIT UKURAN GAMBAR ATAU POSISI MAUPUN WARNA NYA AGAR TIDAK TERLALU SAMA DENGAN YANG LAIN.. !!
## 12 . Deploy UI ke Repository Kalian

```
cd 04-zkapp-browser-ui/ui/
```
```
npm run deploy
```

Jika ada Output Error Seperti Ini Abaikan Dan Tunggu Hingga Proses Selesai :

`./pages/_app.page.tsx
95:6  Warning: React Hook useEffect has a missing dependency: 'state'. Either include it or remove the dependency array. You can also do a functional update 'setState(s => ...)' if you only need 'state' in the 'setState' call.  react-hooks/exhaustive-deps
115:6  Warning: React Hook useEffect has a missing dependency: 'state'. Either include it or remove the dependency array. You can also do a functional update 'setState(s => ...)' if you only need 'state' in the 'setState' call.  react-hooks/exhaustive-deps`

1. Nanti Ketika Selesai Anda Akan di Suruh Memasukan Username Github Kalian
2. Masukan Password Github Anda, Ingat di Awal Kita Menggunakan Token Acces. Gunakan Itu Lagi Untuk Password Github (Atau Buat Token Acces Baru Lagi)

## 13 . Untuk Memastikan Apakah Sudah Jalan Dengan Baik

1. Edit ini : `https://<Your-Username>.github.io/04-zkapp-browser-ui/index.html`
2. Your-Username = Ganti Dengan Nama Github Kalian
3. Jika Sudah, Paste ke Google Chrome

## Jika web sudah benar dan tidak ada error apapun bisa lanjut task nya

# Buat Project
```
cd ..
zk project 05-common-types-and-functions
```
```
cd 05-common-types-and-functions
```
hapus beberapa file
```
rm src/Add.ts
rm src/Add.test.ts
rm src/interact.ts
```
Buat file baru
```
zk file src/BasicMerkleTreeContract
```
kemudian jika kalian menggunakan vscode buka file BasicMerkleTreeContract.ts pada folder src/src/BasicMerkleTreeContract.ts dan isi dengan script berikut
```
import {
  Field,
  SmartContract,
  state,
  State,
  method,
  DeployArgs,
  Permissions,
  MerkleWitness,
  Poseidon,
  PublicKey,
  Signature,
  Circuit,
} from 'snarkyjs';

class MerkleWitness20 extends MerkleWitness(20) {}

export class BasicMerkleTreeContract extends SmartContract {
  @state(Field) treeRoot = State<Field>();

  deploy(args: DeployArgs) {
    super.deploy(args);
    this.setPermissions({
      ...Permissions.default(),
      editState: Permissions.proofOrSignature(),
    });
  }

  @method initState(initialRoot: Field) {
    this.treeRoot.set(initialRoot);
  }

  @method update(
    leafWitness: MerkleWitness20, 
    numberBefore: Field,
    incrementAmount: Field,
  ) {
    const initialRoot = this.treeRoot.get();
    this.treeRoot.assertEquals(initialRoot);

    incrementAmount.assertLt(Field(10));

    // check the initial state matches what we expect
    const rootBefore = leafWitness.calculateRoot(numberBefore);
    rootBefore.assertEquals(initialRoot);

    // compute the root after incrementing
    const rootAfter = leafWitness.calculateRoot(numberBefore.add(incrementAmount));

    // set the new root
    this.treeRoot.set(rootAfter);
  }
}

```
Jika sudah silahkan save. Ctrl + s. kembali ke terminal dan buat file baru lagi
```
zk file src/main
```
Buka file main.ts kemudian ganti/isi dengan script berikut.
```

import {
  Mina,
  isReady,
  shutdown,
  Bool,
  UInt32,
  UInt64,
  Int64,
  Character,
  CircuitString,
  PrivateKey,
  PublicKey,
  Signature,
  Poseidon,
  Field,
  CircuitValue,
  prop,
  arrayProp,
  Circuit,
  MerkleWitness,
  MerkleTree,
  AccountUpdate
} from 'snarkyjs'

import { LedgerContract } from './LedgerContract.js'
import { BasicMerkleTreeContract } from './BasicMerkleTreeContract.js'

async function main() {
  await isReady;

  // --------------------------------------

  const num1 = UInt32.from(40);
  const num2 = UInt64.from(40);

  const num1EqualsNum2: Bool = num1.toUInt64().equals(num2);
  
  console.log("// --------------------------------------");
  console.log(`num1 == num2: ${num1EqualsNum2.toString()}`);
  console.log(`Fields in num1: ${num1.toFields().length}`);

  // --------------------------------------

  const signedNum1 = Int64.from(-3);
  const signedNum2 = Int64.from(45);

  const signedNumSum = signedNum1.add(signedNum2);

  console.log(`signedNum1 + signedNum2: ${signedNumSum.toString()}`);
  console.log(`Fields in signedNum1: ${signedNum1.toFields().length}`);

  // --------------------------------------

  const char1 = Character.fromString('c');
  const char2 = Character.fromString('d');

  console.log(`char1: ${char1.toString()}`);
  console.log(`char1 == char2: ${char1.equals(char2).toString()}`);
  console.log(`Fields in char1: ${char1.toFields().length}`);
  console.log("// --------------------------------------");

  // --------------------------------------

  const str1 = CircuitString.fromString('abc..xyz');
  console.log(`str1: ${str1.toString()}`);
  console.log(`Fields in str1: ${str1.toFields().length}`);

  // --------------------------------------

  const privateKey = PrivateKey.random();
  const publicKey = privateKey.toPublicKey();

  const data1 = char2.toFields().concat(signedNumSum.toFields())
  const data2 = char1.toFields().concat(str1.toFields());

  const signature = Signature.create(privateKey, data2);

  const verifiedData1 = signature.verify(publicKey, data1);
  const verifiedData2 = signature.verify(publicKey, data2);

  console.log(`private key: ${privateKey.toBase58()}`);
  console.log(`public key: ${publicKey.toBase58()}`);
  console.log(`Fields in private key: ${privateKey.toFields().length}`);
  console.log(`Fields in public key: ${publicKey.toFields().length}`);

  console.log(`signature verified for data1: ${verifiedData1.toString()}`);
  console.log(`signature verified for data2: ${verifiedData2.toString()}`);

  console.log(`Fields in signature: ${signature.toFields().length}`);
  console.log("// --------------------------------------");

  // --------------------------------------

  class Point extends CircuitValue {
    @prop x: Field;
    @prop y: Field;

    static fromField(x: Field, y: Field) {
      return new Point(x, y);
    }

    static addPoints(a: Point, b: Point) {
      return new Point(a.x.add(b.x), a.y.add(b.y));
    }
  }

  const point1 = Point.fromField(Field(10), Field(4));
  const point2 = Point.fromField(Field(1), Field(2));

  const pointSum = Point.addPoints(point1, point2);

  console.log(`pointSum Fields: ${pointSum.toFields().map((p) => p.toString())}`);

  class Points8 extends CircuitValue {
    @arrayProp(Point, 8) points: Point[];

    constructor(points: Point[]) {
      super();
      this.points = points;
    }
  }

  const pointsArray = new Array(8).fill(null).map((_, i) => Point.fromField(Field(i), Field(i*10)));
  const points8 = new Points8(pointsArray);

  console.log(`points8 Fields: ${JSON.stringify(points8)}`);
  console.log("// --------------------------------------");

  // --------------------------------------

  const input1 = Int64.from(10);
  const input2 = Int64.from(-15);

  const inputSum = input1.add(input2);

  const inputSumAbs = Circuit.if(inputSum.isPositive(), inputSum, inputSum.mul(Int64.minusOne));

  console.log(`inputSum: ${inputSum.toString()}`);
  console.log(`inputSumAbs: ${inputSumAbs.toString()}`);

  const input3 = Int64.from(22);

  const input1largest = input1.sub(input2).isPositive().and(input1.sub(input3).isPositive());
  const input2largest = input2.sub(input1).isPositive().and(input2.sub(input3).isPositive());
  const input3largest = input3.sub(input1).isPositive().and(input3.sub(input2).isPositive());

  const largest = Circuit.switch([ input1largest, input2largest, input3largest ], Int64, [ input1, input2, input3 ])

  console.log(`largest: ${largest.toString()}`);
  console.log("// --------------------------------------");

  // --------------------------------------

  const Local = Mina.LocalBlockchain();
  Mina.setActiveInstance(Local);
  const deployerAccount = Local.testAccounts[0].privateKey;

  // --------------------------------------
  // create a new merkle tree and BasicMerkleTreeContract zkapp account 

  {

    const basicTreeZkAppPrivateKey = PrivateKey.random();
    const basicTreeZkAppAddress = basicTreeZkAppPrivateKey.toPublicKey();

    const height = 20;
    const tree = new MerkleTree(height);
    class MerkleWitness20 extends MerkleWitness(height) {}

    const zkapp = new BasicMerkleTreeContract(basicTreeZkAppAddress);

    const deployTxn = await Mina.transaction(deployerAccount, () => {
      AccountUpdate.fundNewAccount(deployerAccount);
      zkapp.deploy({ zkappKey: basicTreeZkAppPrivateKey });
      zkapp.initState(tree.getRoot());
      zkapp.sign(basicTreeZkAppPrivateKey);
    });
    await deployTxn.send();

    const incrementIndex = 522;
    const incrementAmount = Field(9);

    const witness = new MerkleWitness20(tree.getWitness(BigInt(incrementIndex)));
    tree.setLeaf(BigInt(incrementIndex), incrementAmount);

    const txn1 = await Mina.transaction(deployerAccount, () => {
      zkapp.update(
        witness,
        Field.zero,
        incrementAmount);
      zkapp.sign(basicTreeZkAppPrivateKey);
    });
    await txn1.send();

    console.log(`BasicMerkleTree: local tree root hash after send1: ${tree.getRoot()}`);
    console.log(`BasicMerkleTree: smart contract root hash after send1: ${zkapp.treeRoot.get()}`);

  }

  // --------------------------------------
  // create a new merkle tree and LedgerContract zkapp account 
  
  {

    const ledgerZkAppPrivateKey = PrivateKey.random();
    const ledgerZkAppAddress = ledgerZkAppPrivateKey.toPublicKey();

    const height = 20;
    const tree = new MerkleTree(height);
    class MerkleWitness20 extends MerkleWitness(height) {}

    const senderInitialBalance = Field(100);
    const recipientInitialBalance = Field(7);

    const senderPrivateKey = PrivateKey.random();
    const senderPublicKey = senderPrivateKey.toPublicKey();

    const recipientPrivateKey = PrivateKey.random();
    const recipientPublicKey = recipientPrivateKey.toPublicKey();

    const senderAccount = 10;
    const recipientAccount = 500;

    tree.setLeaf(
      BigInt(senderAccount), 
      Poseidon.hash([ senderInitialBalance, Poseidon.hash(senderPublicKey.toFields()) ])
    );
    tree.setLeaf(
      BigInt(recipientAccount), 
      Poseidon.hash([ recipientInitialBalance, Poseidon.hash(recipientPublicKey.toFields()) ])
    );

    const zkapp = new LedgerContract(ledgerZkAppAddress);

    const deployTxn = await Mina.transaction(deployerAccount, () => {
      AccountUpdate.fundNewAccount(deployerAccount);
      zkapp.deploy({ zkappKey: ledgerZkAppPrivateKey });
      zkapp.initState(tree.getRoot());
      zkapp.sign(ledgerZkAppPrivateKey);
    });
    await deployTxn.send();

    // --------------------------------------
    // send from the sender to the recipient

    const amount = Field(12);

    const newSenderBalance = senderInitialBalance.sub(amount)

    const sendWitness1 = new MerkleWitness20(tree.getWitness(BigInt(senderAccount)));
    tree.setLeaf(
      BigInt(senderAccount), 
      Poseidon.hash([ newSenderBalance, Poseidon.hash(senderPublicKey.toFields()) ])
    );
    const recipientWitness1 = new MerkleWitness20(tree.getWitness(BigInt(recipientAccount)));

    tree.setLeaf(
      BigInt(recipientAccount), 
      Poseidon.hash([ recipientInitialBalance.add(amount), Poseidon.hash(recipientPublicKey.toFields()) ])
    );

    const signature1 = Signature.create(
      senderPrivateKey, 
      [ zkapp.ledgerRoot.get(), amount ].concat(recipientPublicKey.toFields())
    );

    const txn1 = await Mina.transaction(deployerAccount, () => {
      zkapp.sendBalance(
        sendWitness1, 
        recipientWitness1, 
        senderInitialBalance, 
        recipientInitialBalance, 
        senderPublicKey,
        recipientPublicKey,
        signature1,
        amount);
      zkapp.sign(ledgerZkAppPrivateKey);
    });
    await txn1.send();

    console.log(`LedgerContract: local tree root hash after send1: ${tree.getRoot()}`);
    console.log(`LedgerContract: smart contract root hash after send1: ${zkapp.ledgerRoot.get()}`);

    // --------------------------------------
    // send from the sender to a recipient that wasn't in the account before

    const newRecipientPublicKey = PrivateKey.random().toPublicKey();
    const newRecipientAccount = 10000;

    const sendWitness2 = new MerkleWitness20(tree.getWitness(BigInt(senderAccount)));
    tree.setLeaf(BigInt(senderAccount), Poseidon.hash([ newSenderBalance.sub(amount), Poseidon.hash(senderPublicKey.toFields()) ]));
    const recipientWitness2 = new MerkleWitness20(tree.getWitness(BigInt(newRecipientAccount)));

    tree.setLeaf(BigInt(newRecipientAccount), Poseidon.hash([ amount, Poseidon.hash(newRecipientPublicKey.toFields()) ]));

    const signature2 = Signature.create(
      senderPrivateKey, 
      [ zkapp.ledgerRoot.get(), amount ].concat(newRecipientPublicKey.toFields())
    );

    const txn2 = await Mina.transaction(deployerAccount, () => {
      zkapp.sendBalance(
        sendWitness2, 
        recipientWitness2, 
        newSenderBalance, 
        Field.zero, 
        senderPublicKey,
        newRecipientPublicKey,
        signature2,
        amount);
      zkapp.sign(ledgerZkAppPrivateKey);
    });
    await txn2.send();

    console.log(`LedgerContract: local tree root hash after send2: ${tree.getRoot()}`);
    console.log(`LedgerContract: smart contract root hash after send2: ${zkapp.ledgerRoot.get()}`);

  }

  // --------------------------------------

  await shutdown();
}

main();
```
save dan buat file baru lagi
```
zk file src/LedgerContract
```
Buka file LedgerContract.ts dan isi dengan script berikut
```
import {
  Field,
  SmartContract,
  state,
  State,
  method,
  DeployArgs,
  Permissions,
  MerkleWitness,
  Poseidon,
  PublicKey,
  Signature,
  Circuit,
} from 'snarkyjs';

class MerkleWitness20 extends MerkleWitness(20) {}

export class LedgerContract extends SmartContract {
  @state(Field) ledgerRoot = State<Field>();

  deploy(args: DeployArgs) {
    super.deploy(args);
    this.setPermissions({
      ...Permissions.default(),
      editState: Permissions.proofOrSignature(),
    });
  }

  @method initState(initialLedgerRoot: Field) {
    this.ledgerRoot.set(initialLedgerRoot);
  }

  @method sendBalance(
    senderWitness: MerkleWitness20, 
    recipientWitness: MerkleWitness20,
    senderBalanceBefore: Field,
    recipientBalanceBefore: Field,
    senderPublicKey: PublicKey,
    recipientPublicKey: PublicKey,
    senderSignature: Signature,
    sendAmount: Field
  ) {

    const initialLedgerRoot = this.ledgerRoot.get();
    this.ledgerRoot.assertEquals(initialLedgerRoot);

    // check the sender's signature
    senderSignature.verify(
      senderPublicKey, 
      [ initialLedgerRoot, sendAmount ].concat(recipientPublicKey.toFields())
    ).assertTrue();

    // check the initial state matches what we expect
    const rootSenderBefore = senderWitness.calculateRoot(
      Poseidon.hash([ Field(senderBalanceBefore), Poseidon.hash(senderPublicKey.toFields()) ]));
    rootSenderBefore.assertEquals(initialLedgerRoot);

    senderBalanceBefore.assertGte(sendAmount);

    // compute the sender state after sending
    const rootSenderAfter = senderWitness.calculateRoot(
      Poseidon.hash([ Field(senderBalanceBefore).sub(sendAmount), Poseidon.hash(senderPublicKey.toFields()) ]));

    // compute the possible recipient states before receiving
    const rootRecipientBefore = recipientWitness.calculateRoot(
      Poseidon.hash([ Field(recipientBalanceBefore), Poseidon.hash(recipientPublicKey.toFields()) ]));
    const rootRecipientBeforeEmpty = recipientWitness.calculateRoot(Field.zero);

    const recipientAccountNew = rootSenderAfter.equals(rootRecipientBeforeEmpty);

    // check requirements on the recipient state before receiving
    const recipientAccountPassesRequirements = Circuit.if(
      recipientAccountNew, 
      (() => {
        // new account
        // balance before must be zero
        return recipientBalanceBefore.equals(Field.zero)
      })(),
      (() => {
        // existing account
        // check existing account witness
        return rootSenderAfter.equals(rootRecipientBefore);
      })());

    recipientAccountPassesRequirements.assertTrue();

    // compute the recipient state after receiving
    const rootRecipientAfter = recipientWitness.calculateRoot(
      Poseidon.hash([ Field(recipientBalanceBefore).add(sendAmount), Poseidon.hash(recipientPublicKey.toFields()) ]));

    // set the new ledgerRoot
    this.ledgerRoot.set(rootRecipientAfter);
  }
}
```
Jika sudah silahkan pindahkan 3 file yang sudah di edit tadi ke folder build/src dan file dengan akhiran .test.ts dihapus saja. 

## Konfigurasi zk
```
zk config
```
isi dengan ini
Nama:berkeley
URL: https://proxy.berkeley.minaexplorer.com/graphql
Biaya:0.1

Jika sudah salin link faucet dari outputnya dan paste di chrome. Tunggu beberapa menit untuk memastikan faucet sudah masuk.
Kemudian Deploy
```
zk deploy berkeley
```
Enter
kemudian buka file keys/berkeley.json

Silahkan kalian simpan dan import wallet ke auro wallet. setelah itu copy address nya dan scan di mina scan untuk mendapatkan hash verificationKey nya.. 

## Cara Uninstal Semua (Jika Pengen Menghapus)

```
rm -rf 04-zkapp-browser-ui
rm -rf zkapp-cli
rm -rf .npm
npm uninstall -g zkapp-cli
sudo apt-get remove nodejs
```

