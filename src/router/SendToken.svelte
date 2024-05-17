<script lang="ts">
  import {Transaction,PhantasmaAPI } from 'phantasma-ts';
  import  {ScriptBuilder, Base16, Address}   from  'phantasma-ts/core';
  import BigNumber from 'bignumber.js';
import {Button} from "flowbite-svelte"
 
// Your WIF key and contract information
const wif = import.meta.env.VITE_PRIVATE_KEY; // **************Replace with your WIF key**********************
const NexusName = 'testnet'; // or 'mainnet'
const ChainName = 'main'; // The chain name
const expiration = new Date(new Date().getTime() + 36000000); // Transaction expiration (1 hour from now)
const RPC = new PhantasmaAPI("https://testnet.phantasma.info/rpc",null, NexusName);

const gasPrice = 100000;
const gasLimit = 21000;
const payload = Base16.encode("Phanta-ts");


let fromAddress = " ";
let toAddress = " ";

const tokenToSend = "KCAL";
const tokenAmount = new BigNumber(1).shiftedBy(10).toFixed();

function delay(time) {
    return new Promise(resolve => setTimeout(resolve, time));
}

const sb = new ScriptBuilder();
sb.AllowGas(fromAddress, Address.Null, gasPrice, gasLimit);   
sb.CallInterop("Runtime.TransferTokens", [fromAddress, toAddress, tokenToSend, tokenAmount]);
sb.SpendGas(fromAddress);
const script = sb.EndScript();

//console.log("Transaction script: ", script);

let transaction = new Transaction(NexusName, ChainName, script, expiration,payload);

transaction.sign(wif);

async function sendATransaction(tx) {
    console.log("Sending Transaction: ");
    try {
        const rawTx = Base16.encodeUint8Array(tx.ToByteAray(true));
        console.log("Raw transaction: ", rawTx);
        const txHash = await RPC.sendRawTransaction(rawTx);
        console.log("Transaction hash: ", txHash);
        await delay(5000);
        let result = await RPC.getTransaction(txHash);
        console.log("Transaction result: ", result);
        return result;
    } catch (error) {
        console.error('Error during transaction:', error);
        throw error;
    }
}

/*sendATransaction(transaction)
    .then(txHash => {
        console.log("Transaction sent successfully. Hash: ", txHash);
    })
    .catch(error => {
        console.error("Error in sendATransaction: ", error);
    });
    */
</script>

<main>

    <Button on:click={()=>sendATransaction(transaction)}>Send Tx</Button>

</main>