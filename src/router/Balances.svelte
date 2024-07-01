<script lang="ts">
  import {
    PhantasmaAPI,
    ScriptBuilder,
    Decoder,
    StakeReward,
    Transaction,
    Address,
    PhantasmaKeys,
    Base16
  } from "phantasma-ts";
  import { BigNumber } from "bignumber.js";
  import { Card, Listgroup, Avatar, GradientButton, Button, Table,
    TableBody,
    TableBodyCell,
    TableBodyRow,
    TableHead,
    TableHeadCell,
    ImagePlaceholder,
    Range, 
    Label, 
    Input, 
    Checkbox,
    Modal,
    Spinner,
    Toast,
    Textarea 
     } from "flowbite-svelte";

     import { slide } from 'svelte/transition';
      import { quintOut } from "svelte/easing";
  let fmt = {
    decimalSeparator: ".",
    groupSeparator: " ",
    groupSize: 3,
    secondaryGroupSize: 0,
    fractionGroupSeparator: " ",
    fractionGroupSize: 0,
  };
  BigNumber.config({ FORMAT: fmt });

  let tokens = [];
  let token = [];

  let stakedSoul;
  let unclaimedKcal;
  
  let isUnclaimedKcalZero = true;
  let isStakeAmountZero = true;
  let kcalGenerationBoost=0;
  let dailyKcalGeneration="0";
  let crownBoost=5;
  let kcalGenerationRate=0.002;
  let soulMasterReward=0;
  let isSoulMaster=false;
  let isInflationTime=false;
  let nextInflationDateInfo:string;
  const soulMasterRewardPool=125000;
  let remainedTimeForNextKcalGenerationInfo:string;
  let timeRemainedForUnstake:string;
  let crownDistiributionInfo:string;
  let isSoulMasterRewardTime:boolean=false;
  let isCrowner:boolean=false;

  let openInfoToast=false;
  let InfoMessage:string[]=[];

  let sendTokenModal = false;  // Send Modal variables common for some other modals too
  let modalTokenTicker = '';
  let modalMaxAmount = 0;

  let stakeSoulModal = false; // Stake Soul Modal variables
  let minStakeAmount=1;

  let unStakeSoulModal=false; // unStake Soul Modal variables
  let isUnstakeTime=false;

  let sendNftModal=false; // Send nft modal variables
  let modalNftIds:string[]=[];
  let modalNftTicker="";
  let dataForShowSendNft:{
    ticker:string,
    id:number,
    properties:{key:string,value:string}[],
    infusion:{key:string,value:string}[]
  }[]=[];

  let selectedNfts:{nftTicker:string,nftId:number}[] = [];
  let NftSendButtonDisable=false;
  
  
  let confirmModal=false;
  let confirmModalMessage:string[]=[];
  let isTransactionConfirmed=false;
  let isUserClickedConfirmModalButtons=false;
  let feeWarning:string="";
  let isFeeBalanceEnough:boolean;

  let toAddress = '';
  let sendAmount = 0;
  let sendTokenTicker="";
  let sendTokenDecimals=0;
  let feeTokenBalance:BigNumber;
  const feeAdjustStep=10000;
  const feeTokenDecimals=10;
  let feeAmount="0.21";
  

  let WIF = import.meta.env.VITE_PRIVATE_KEY_2; //WIF format
  const keys = PhantasmaKeys.fromWIF(WIF);
  let chain = "main";
  let network = "testnet";
  let gasPrice = 100000;
  let gasLimit = 21000;
  let payload = Base16.encode("Yurei"); //Says 'Yurei' in hex
  let address = keys.Address; // 
    let rpcLink = "https://testnet.phantasma.info/rpc"; // https://testnet.phantasma.info/rpc https://pharpc1.phantasma.info/rpc
  const RPC = new PhantasmaAPI(rpcLink, null, network);

  let tokenBalances: {
    Ticker: string;
    Amount: string;
    Decimals: number;
  }[] = [];

  let nftBalances: {
    Ticker: string;
    Amount: string;
    Ids: string[];
  }[] = [];

  
  async function getBalances() {
    let updatedTokenBalances: {
    Ticker: string;
    Amount: string;
    Decimals: number;
  }[] = [];

  let UpdatedNftBalances: {
    Ticker: string;
    Amount: string;
    Ids: string[];
  }[] = [];

    let isSoulInBalances = false;
    let boostedKcalGenerationRate:number=kcalGenerationRate;
    try {
      
    
    const balanceResponse = await RPC.getAccount(address.toString(), true);

    console.log(balanceResponse);

    console.log(tokens);

    balanceResponse.balances.forEach((element) => {
      let isElementFungible = true;
      

      if (element.ids.length > 0) {
        isElementFungible = false;
        UpdatedNftBalances.push({
          Ticker: element.symbol,
          Amount: element.amount,
          Ids: element.ids,
        });
        if(element.symbol=="CROWN"){
          kcalGenerationBoost=BigNumber(element.amount).toNumber()*crownBoost;
          boostedKcalGenerationRate=kcalGenerationRate*(1+kcalGenerationBoost/100);
        }
      } else {
        updatedTokenBalances.push({
          Ticker: element.symbol,
          Amount: BigNumber(element.amount)
            .shiftedBy(-element.decimals)
            .toFormat(),
          Decimals: element.decimals,
        });
        if(element.symbol==="KCAL"){
          feeTokenBalance=BigNumber(element.amount);
        }
      }

      if (element.symbol == "SOUL") {
        isSoulInBalances = true;
      }

      
    });

    stakedSoul = BigNumber(balanceResponse.stake).shiftedBy(-8).toFormat();
    dailyKcalGeneration=BigNumber(balanceResponse.stake).shiftedBy(-8).multipliedBy(boostedKcalGenerationRate).decimalPlaces(4).toFormat(),
    isStakeAmountZero = BigNumber(balanceResponse.stake).isEqualTo(0);
    unclaimedKcal = BigNumber(balanceResponse.unclaimed)
      .shiftedBy(-10)
      .toFormat();
    isUnclaimedKcalZero = BigNumber(balanceResponse.unclaimed).isEqualTo(0);
    

    

    if (isStakeAmountZero){
      minStakeAmount=2;
    }else{minStakeAmount=1};

    
    let sb = new ScriptBuilder();
        sb.CallContract(
          "stake",
            "GetMasterCount",
            [],
        );
        
        sb.CallContract("stake","IsMaster",[address]);
        sb.CallContract("stake","GetStakeTimestamp",[address]);
        sb.CallContract("stake","GetTimeBeforeUnstake",[address]);
        sb.CallContract("gas","GetNextInflationDate",[]); 
    
        const script = sb.EndScript();

        const response = await RPC.invokeRawScript(chain, script);

        let sb2=new ScriptBuilder();
        sb2.CallContract("gas","GetLastInflationDate",[]);
        sb2.CallContract("stake","GetMasterDate",[address]);
        sb2.CallContract("stake","GetMasterClaimDateForAddress",[address])
        const script2 = sb2.EndScript();
        const response2 = await RPC.invokeRawScript(chain, script2);

        const masterCountDecode = new Decoder(response.results[0]).readVmObject();
        const isSoulMasterDecoder = new Decoder(response.results[1]).readVmObject();
        const getStakeTimestampDecoder = new Decoder(response.results[2]).readBigInt();
        const remainedTimeForKcalGeneration = new Decoder(response.results[3]).readVmObject();
        let nextInflationDateDecoder = new Decoder(response.results[4]).readTimestamp(); 

        const lastInflationDateDecoder = new Decoder(response2.results[0]).readBigInt();
        const getFirstMasterClaimDateForAddressDecoder= new Decoder(response2.results[1]).readBigInt();
        const getMasterClaimDateDecoder = new Decoder(response2.results[2]).readBigInt();
        
        isSoulMaster=isSoulMasterDecoder;
        const masterCount = masterCountDecode; 

    soulMasterReward=soulMasterRewardPool/masterCount;

    nextInflationDateDecoder=lastInflationDateDecoder+90*86400;  //it returns wrong soo i calculated it from last

    
    
    let d = new Date(nextInflationDateDecoder*1000);
    nextInflationDateInfo= d.toLocaleDateString();
    let timeForUnstake=getStakeTimestampDecoder*1000+86400*1000;
    let remainedTimeForUnstake:number;
    let remainedTimeForCrown:number;
    let currentTime = new Date().valueOf();
    
    if(nextInflationDateDecoder*1000>currentTime){
      remainedTimeForCrown=nextInflationDateDecoder*1000-currentTime;
    }else{remainedTimeForCrown=0}

    if(timeForUnstake>currentTime){
      remainedTimeForUnstake=timeForUnstake-currentTime;
    }else{remainedTimeForUnstake=0;}

    if(remainedTimeForUnstake>3600000){
      isUnstakeTime=false;
      timeRemainedForUnstake=Math.ceil((remainedTimeForUnstake/3600000)) +" Hours remained for unstake";//+Math.ceil((remainedTimeForUnstake % 3600000)/60000)+ " Minutes remained for unstake";
    }else if (remainedTimeForUnstake>0 && remainedTimeForUnstake<=3600000){
      isUnstakeTime=false;
      timeRemainedForUnstake=Math.ceil(remainedTimeForUnstake/60000).toString()+" Minutes remained for unstake"
    }else {
      timeRemainedForUnstake="You can unstake";
      isUnstakeTime=true;
    };

    if(remainedTimeForKcalGeneration<82801){
      remainedTimeForNextKcalGenerationInfo=Math.ceil((86400-remainedTimeForKcalGeneration)/3600).toString()+" Hours";
    }else{remainedTimeForNextKcalGenerationInfo=Math.ceil((86400-remainedTimeForKcalGeneration)/60).toString()+" Minutes";};

    if(getFirstMasterClaimDateForAddressDecoder>lastInflationDateDecoder && isSoulMaster){
      isInflationTime=false;
      //crownDistiributionInfo="You are not eligible for current Crown distribution";
      isCrowner=false;

      if(remainedTimeForCrown>86400000){
        crownDistiributionInfo="You are eligible for Crown after "+Math.ceil(remainedTimeForCrown/86400000).toString()+" Days";
      }else if(remainedTimeForCrown<=86400000 && remainedTimeForCrown>3600000){
        crownDistiributionInfo="You are eligible for Crown after "+Math.ceil(remainedTimeForCrown/3600000).toString()+" Hours";
      }else if(remainedTimeForCrown>0 && remainedTimeForCrown<=3600000){
        crownDistiributionInfo="You are eligible for Crown after "+Math.ceil(remainedTimeForCrown/60000).toString()+" Minutes";
      } else {crownDistiributionInfo="Waiting for distribution to be eligible for next Crown";
        isInflationTime=true;
      }
      

    }else if (isSoulMaster){
      isInflationTime=false;
      //crownDistiributionInfo="You are eligible for current Crown distribution";
      isCrowner=true;

      if(remainedTimeForCrown>86400000){
        crownDistiributionInfo="\nYou will get a Crown after "+Math.ceil(remainedTimeForCrown/86400000).toString()+" Days";
      }else if(remainedTimeForCrown<=86400000 && remainedTimeForCrown>3600000){
        crownDistiributionInfo="\nYou will get a Crown after "+Math.ceil(remainedTimeForCrown/3600000).toString()+" Hours";
      }else if(remainedTimeForCrown>0 && remainedTimeForCrown<=3600000){
        crownDistiributionInfo="\nYou will get a Crown after"+Math.ceil(remainedTimeForCrown/60000).toString()+" Minutes";
      } else {crownDistiributionInfo="Waiting for Crown distribution";
        isInflationTime=true;
      }
    }

    if(currentTime>getMasterClaimDateDecoder*1000){
      isSoulMasterRewardTime=true;
    }else{isSoulMasterRewardTime=false;}
    
    console.log("Stake Timestamp_"+getStakeTimestampDecoder+
    "\nTime For Unstake_" + timeForUnstake+
    "\nRemained Time For Unstahe_"+remainedTimeForUnstake+
    "\nMessage For remaining timeForUnstake_"+timeRemainedForUnstake+
    "\nEstimated SoulMaster Reward_"+soulMasterReward+
    "\nSoulMaster Count_"+masterCount+
    "\nIs SoulMaster_" +isSoulMaster+
    "\nTokens In account_"+updatedTokenBalances.length+
    "\nNft Tokens In account_"+UpdatedNftBalances.length+
    "\nNext Inflation Timestamp_"+nextInflationDateDecoder+
    "\nNext InflationDistribution is on_"+d.toLocaleDateString()+
    "\nCurrent Time_"+currentTime+
    "\nLast Inflation Timestamp_"+lastInflationDateDecoder+
    "\nRemained Time For Kcal Generation_"+remainedTimeForKcalGeneration+
    "\nCrown distribution info_"+crownDistiributionInfo+
    "\nFirst SoulMaster Reward claim date_"+getFirstMasterClaimDateForAddressDecoder+
    "\nRemained time for crown_"+remainedTimeForCrown+
    "\nMaster Claim Date_"+getMasterClaimDateDecoder+
    "\nIs unclaimed Kcal zero_"+isUnclaimedKcalZero+
    "\nUnclaimed Kcal_" + unclaimedKcal + 
    "\nStaked Soul_" + stakedSoul  
    );

  } catch (error) {
      console.log("error for get balances\n"+error)
    }

    if (!isStakeAmountZero && !isSoulInBalances) {
      updatedTokenBalances.splice(0,0,{ Ticker: "SOUL", Amount: "0", Decimals: 8 });
    }

    nftBalances=UpdatedNftBalances;
    tokenBalances=updatedTokenBalances;
  }

  async function triggerInflation(from:Address){
    let expiration_date = new Date(new Date().valueOf() + 36000000);
    console.log("Trying to trigger inf");
    console.log("Tx expiration time:"+expiration_date);
  let trInfsb = new ScriptBuilder();
  
  trInfsb.AllowGas(from, Address.Null, gasPrice, gasLimit);
      //sbinf.CallContract("gas","FixInflationTiming",[keys.Address,1710205200000])
      trInfsb.CallContract("gas", "ApplyInflation", [from]);
      trInfsb.SpendGas(from);   
  let trInfScrip = trInfsb.EndScript();

    
  //Creating New Transaction Object
  let trInfTransaction = new Transaction(
    network, //Nexus Name - if you're using mainnet change it to mainnet
    chain, //Chain
    trInfScrip, //In string format
    expiration_date, //Date Object
    payload //Extra Info to attach to Transaction in Serialized Hex
  );

  //Sign's Transaction with WIF
  trInfTransaction.signWithKeys(keys);
    
    try {
        const rawTx = Base16.encodeUint8Array(trInfTransaction.ToByteAray(true));
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
  async function claimSoulMasterReward(from:Address) { //waits 5 sec to get tx response
    let expiration_date = new Date(new Date().valueOf() + 36000000);
    let sbsmr = new ScriptBuilder();
  
      sbsmr.AllowGas(from, Address.Null, gasPrice, gasLimit);
      //sbinf.CallContract("gas","FixInflationTiming",[keys.Address,1710205200000])
      sbsmr.CallContract("stake", "MasterClaim", [from]);
      sbsmr.SpendGas(from);   
  let scriptsmr = sbsmr.EndScript();

    
  //Creating New Transaction Object
  let smrTransaction = new Transaction(
    network, //Nexus Name - if you're using mainnet change it to mainnet
    chain, //Chain
    scriptsmr, //In string format
    expiration_date, //Date Object
    payload //Extra Info to attach to Transaction in Serialized Hex
  );

  //Sign's Transaction with keys
  smrTransaction.signWithKeys(keys);
  try {
        const rawTx = Base16.encodeUint8Array(smrTransaction.ToByteAray(true));
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
  

  async function claimKcal(from:Address) {  //waits 5 sec to get tx response
    let expiration_date = new Date(new Date().valueOf() + 36000000);
    let sbkcal = new ScriptBuilder();
  
      sbkcal.AllowGas(from, Address.Null, gasPrice, gasLimit);
      
      sbkcal.CallContract("stake", "Claim", [from,from]);
      sbkcal.SpendGas(from);   
  let scriptkcal = sbkcal.EndScript();

    
  //Creating New Transaction Object
  let claimKcaltransaction = new Transaction( 
    network, //Nexus Name - if you're using mainnet change it to mainnet
    chain, //Chain
    scriptkcal, //In string format
    expiration_date, //Date Object
    payload //Extra Info to attach to Transaction in Serialized Hex
  );

  //Sign's Transaction with keys
  claimKcaltransaction.signWithKeys(keys);
    

    try {
        const rawTx = Base16.encodeUint8Array(claimKcaltransaction.ToByteAray(true));
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
  
  async function handleSendToken() {
    // Check text input length
    confirmModalMessage=[];
    toAddress=toAddress.trim(); //removing whitespaces
    if (toAddress.length < 47) {
      alert('To address must be at least 47 characters long.');
      return;
    }

    if(toAddress.charAt(0)!=="P"){
      alert('Phantasma adresses starts with P please check your input');
      return;
    }

    if(sendAmount<0){
      alert('Amount must be greater than 0');
      return;
    }
    let messageLine1="You will send"
    let messageLine2=sendAmount + " "+sendTokenTicker;
    let messageLine3="To";
    let messageLine4=toAddress;

    checkFeeTokenBalanceIsEnough();

    confirmModalMessage.push(messageLine1,messageLine2,messageLine3,messageLine4);
    confirmModal=true;
    
    const userResponse = await waitUserInput();
     
    if(userResponse){
      sendToken(keys.Address.toString(),toAddress.toString(),sendTokenTicker,sendTokenDecimals,sendAmount,gasPrice,gasLimit).then(()=>{getBalances();});
      console.log("Transaction confirmed")
      sendTokenModal=false;
      isTransactionConfirmed=false;
    } else{
      console.log("Transaction Cancelled") 
    };
    confirmModalMessage=[];
  }

  
const timeout = async ms => new Promise(res => setTimeout(res, ms));

function checkFeeTokenBalanceIsEnough(){
  feeAmount=BigNumber(gasLimit*gasPrice).shiftedBy(-feeTokenDecimals).toFormat();
  if(BigNumber(gasLimit*gasPrice).isLessThan(feeTokenBalance)){
    isFeeBalanceEnough=true;
    feeWarning="You have enough Kcal"
  }else{isFeeBalanceEnough=false;
    feeWarning="Check your fee settings or get some more Kcal"
  }

  

}

async function waitUserInput() {
    while (isUserClickedConfirmModalButtons === false) await timeout(50); // pause script but avoid browser to freeze ;)
    const userInputVal = isTransactionConfirmed;
    isTransactionConfirmed = false; // reset var
    isUserClickedConfirmModalButtons=false;
    return userInputVal;
}

 async function handleStakeSoul() {
    // Check text input length
    confirmModalMessage=[];

    if(sendAmount<0){
      alert('Amount must be greater than 0');
      return;
    }

    
    let messageLine1="You will stake";
    let messageLine2=sendAmount +" Soul";
    confirmModalMessage.push(messageLine1,messageLine2);
    checkFeeTokenBalanceIsEnough();
    confirmModal=true;

    
    
    const userResponse = await waitUserInput();
     
    if(userResponse){
      stakeSoul(keys.Address.toString(),sendAmount,gasLimit,gasPrice,sendTokenDecimals).then(()=>{getBalances()});
      console.log("Transaction confirmed")
      stakeSoulModal=false;
      isTransactionConfirmed=false;
    } else{console.log("Transaction Not Confirmed")
    
    
    };
    // Additional form submission logic goes here
    
  }
  function openSendTokenModal(tokenTicker, maxAmount,decimals) {
    modalTokenTicker = tokenTicker; // Set the modal header dynamically
    modalMaxAmount = maxAmount; // Set the maximum amount dynamically
    sendTokenTicker=tokenTicker;
    sendTokenDecimals=decimals;
    sendTokenModal = true; // Open the modal
  }

  function openStakeSoulModal(tokenTicker, maxAmount,decimals) {
    modalTokenTicker = tokenTicker; // Set the modal header dynamically
    modalMaxAmount = maxAmount; // Set the maximum amount dynamically
    sendTokenTicker = tokenTicker;
    sendTokenDecimals = decimals;
    stakeSoulModal = true; // Open the modal
  }

  function delay(time) {
    return new Promise(resolve => setTimeout(resolve, time));
}

async function sendToken(from,to,token:string,decimals:number,amount,setGasPrice,setGasLimit) { //waits 5sec for getting tx

console.log("Send Token Data\n"+from+" "+to+" "+token+" "+decimals+" "+amount+" "+setGasPrice+" "+setGasLimit);
let expiration_date = new Date(new Date().valueOf() + 36000000);
const sb = new ScriptBuilder();
sb.AllowGas(from, Address.Null, setGasPrice, setGasLimit);

sb.CallInterop("Runtime.TransferTokens", [from, to, token, BigNumber(amount).shiftedBy(decimals).decimalPlaces(0).toFixed()]);
sb.SpendGas(keys.Address);
const script = sb.EndScript();

//console.log("Transaction script: ", script);

let transaction = new Transaction(network, chain, script, expiration_date ,payload);

transaction.signWithKeys(keys);


    console.log("Sending Transaction: ");
    try {
        openInfoToast=false;
        InfoMessage=[];
        const rawTx = Base16.encodeUint8Array(transaction.ToByteAray(true));
        console.log("Raw transaction: ");
        InfoMessage.push("Sending Transaction");
        openInfoToast=true;
        const txHash = await RPC.sendRawTransaction(rawTx);    
        console.log("Transaction hash: ", txHash);
        await delay(5000);
        let result = await RPC.getTransaction(txHash);
        console.log("Transaction result: ", result);
        openInfoToast=false;
        InfoMessage=[]; 

            if(result.state!==undefined && result.state!==null){
              if(result.state.toString()=="Halt"){
              InfoMessage.push("Transaction\n"+txHash.substring(0,15)+"..."+"\nSent successfully");
              }else{
                InfoMessage.push("Transaction "+txHash.substring(0,15)+"..."+" failed");
            }
          }else{InfoMessage.push("Cannot get Tx info check it on explorer\n"+ result.error
          )};
        
        
    } catch (error) {
        InfoMessage.push('Error during transaction:'+ error)
        console.error('Error during transaction:', error);
        throw error;
    }
    openInfoToast=true;
    
}

function copyMaxAmount(event) {
    event.preventDefault();
    sendAmount = modalMaxAmount;
  }

  async function stakeSoul(from:string,amount,setGasLimit,setGasPrice,decimals) { //waits 5 sec for geeting tx response

    let expiration_date = new Date(new Date().valueOf() + 36000000);
    let stakeSb= new ScriptBuilder();
    let stakeScript = stakeSb
    .AllowGas(from, Address.Null, setGasPrice, setGasLimit)
    .CallContract("stake", "Stake", [from, BigNumber(amount).shiftedBy(decimals).decimalPlaces(0).toFixed()])
    .SpendGas(from)
    .EndScript();

    let stakeTransaction = new Transaction(network, chain, stakeScript, expiration_date ,payload);

    stakeTransaction.signWithKeys(keys);

    try {
        const rawTx = Base16.encodeUint8Array(stakeTransaction.ToByteAray(true));
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

  async function unStakeSoul(from:string,amount,setGasLimit,setGasPrice,decimals) {

let expiration_date = new Date(new Date().valueOf() + 36000000);
let unStakeSb= new ScriptBuilder();
let unStakeScript = unStakeSb
.AllowGas(from, Address.Null, setGasPrice, setGasLimit)
.CallContract("stake", "Unstake", [from, BigNumber(amount).shiftedBy(decimals).decimalPlaces(0).toFixed()])
.SpendGas(from)
.EndScript();

let unStakeTransaction = new Transaction(network, chain, unStakeScript, expiration_date ,payload);

unStakeTransaction.signWithKeys(keys);

try {
    const rawTx = Base16.encodeUint8Array(unStakeTransaction.ToByteAray(true));
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
function openUnStakeSoulModal(tokenTicker, maxAmount,decimals) {
    modalTokenTicker = tokenTicker; // Set the modal header dynamically
    modalMaxAmount = maxAmount; // Set the maximum amount dynamically
    sendTokenTicker = tokenTicker;
    sendTokenDecimals = decimals;
    unStakeSoulModal = true; // Open the modal
  }

  async function handleUnStakeSoul() {
    // Check text input length
    
    confirmModalMessage=[];
    if(sendAmount<0){
      alert('Amount must be greater than 0');
      return;
    }

    let messageLine1="You will unstake";
    let messageLine2=sendAmount +" Soul";
    confirmModalMessage.push(messageLine1,messageLine2);
    checkFeeTokenBalanceIsEnough();
    confirmModal=true;
    
    
    const userResponse = await waitUserInput();
     
    if(userResponse){
      unStakeSoul(keys.Address.toString(),sendAmount,gasLimit,gasPrice,sendTokenDecimals).then(()=>getBalances());
      console.log("Transaction confirmed")
      unStakeSoulModal=false;
      isTransactionConfirmed=false;
    } else{
      console.log("Transaction Cancelled") 
    };

    
  }

  async function confirmTriggerInflation() {
    
    confirmModalMessage=[];
    let messageLine1="You will get a crown";

    gasLimit=200000;
    
    confirmModalMessage.push(messageLine1);
    checkFeeTokenBalanceIsEnough();
    confirmModal=true;
  
    const userResponse = await waitUserInput();
     
    if(userResponse){
      triggerInflation(keys.Address).then(()=>{getBalances();});
      console.log("Transaction confirmed")
      
      isTransactionConfirmed=false;
    } else{
      console.log("Transaction Cancelled") 
    };
    
  }

  async function confirmSmReward() {

    confirmModalMessage=[];
    let messageLine1="You will get Sm Reward";
    
    confirmModalMessage.push(messageLine1);
    checkFeeTokenBalanceIsEnough();
    confirmModal=true;
  
    const userResponse = await waitUserInput();
     
    if(userResponse){
      claimSoulMasterReward(keys.Address).then(()=>{getBalances();});
      console.log("Transaction confirmed")
      isTransactionConfirmed=false;
    } else{
      console.log("Transaction Cancelled") 
    };
    
  }

  async function confirmKcalClaim() {
    
    confirmModalMessage=[];
    let messageLine1="You will claim "+unclaimedKcal+" Kcal";
    
    confirmModalMessage.push(messageLine1);
    checkFeeTokenBalanceIsEnough();
    confirmModal=true;
  
    const userResponse = await waitUserInput();
     
    if(userResponse){
      claimKcal(keys.Address).then(()=>{getBalances();})
      console.log("Transaction confirmed")
      
      isTransactionConfirmed=false;
    } else{
      console.log("Transaction Cancelled") 
    };
    
  }

  async function handleSendNft(){

    confirmModalMessage=[];
    sendNftModal=false;

    if(selectedNfts.length<0){
      alert('Amount must be greater than 0');
      return;
    }

    let messageLine1="You will Send "+selectedNfts.length+" "+ selectedNfts[0].nftTicker;
    confirmModalMessage.push(messageLine1);
    checkFeeTokenBalanceIsEnough();
    confirmModal=true;

    
    
    const userResponse = await waitUserInput();
     
    if(userResponse){
      
      console.log("Transaction confirmed")
      sendNft(keys.Address.toString(),toAddress.toString()).then(()=>{getBalances()});
      sendNftModal=false;
      NftSendButtonDisable=false;
      isTransactionConfirmed=false;
      selectedNfts=[];
    } else{
      console.log("Transaction Cancelled") 
      sendNftModal=true;
      selectedNfts=[];
    };



  }

  async function openSendNftModal(NftTicker, NftIds) {
    modalNftIds = NftIds;
    dataForShowSendNft=[];
    selectedNfts=[];
    NftSendButtonDisable=true;

    const NftPromises = NftIds.map(async (nftId) => {
        const nftDataResponse = await RPC.getNFT(NftTicker, nftId);
        
        let infusions : {key:string,value:string}[]= [];
        let properties:{key:string,value:string}[] = [];

        if (nftDataResponse.infusion.length > 0) {
            nftDataResponse.infusion.forEach((infused) => {
                infusions.push({ key: infused.key, value: infused.value });
            });
        }
        if (nftDataResponse.properties.length > 0) {
            nftDataResponse.properties.forEach((property) => {
                properties.push({ key: property.key, value: property.value });
            });
        }
        
        
        dataForShowSendNft.push({
            ticker:NftTicker,
            id: nftDataResponse.id,
            infusion: infusions,
            properties: properties
        });
    });
   

    await Promise.all(NftPromises).then(()=>NftSendButtonDisable=false);
    
    modalNftTicker = NftTicker;
    sendNftModal = true;
    
    

    // Print all data for dataForShowingNft
    dataForShowSendNft.forEach((nftData) => {
        console.log(`NFT ID: ${nftData.id}`);
        console.log('Infusions:');
        nftData.infusion.forEach((infusion) => {
            console.log(`  Key: ${infusion.key}, Value: ${infusion.value}`);
        });
        console.log('Properties:');
        nftData.properties.forEach((property) => {
            console.log(`  Key: ${property.key}, Value: ${property.value}`);
        });
    });
}

// Function to get the NFT name from the properties array
function getNftName(nftData) {
    for (let property of nftData.properties) {
      if (property.key === 'Name') {
        return property.value;
      }
    }
    // Return a placeholder if Name is not found
    return 'Unnamed NFT';
  }

  function getNftDescription(nftData) {
    for (let property of nftData.properties) {
      if (property.key === 'Description') {
        return property.value;
      }
    }
    // Return a placeholder if Name is not found
    return 'No description';
  }

  // Function to navigate to the Ghost Market page for the specific NFT
  function goToMarketPage(ticker, id) {
    // Construct the URL based on the ticker and ID
    const url = `https://ghostmarket.io/asset/pha/${ticker}/${id}`;
    // Open the URL in a new tab
    window.open(url, '_blank');
  }

  function addRemoveFromNftList(nftId:number,nftTicker:string){

    
    console.log("Nft Data for process \n"+"Nft ID\t"+nftId+"\nNFT ticker\t"+nftTicker);

    const index = selectedNfts.findIndex(nft => nft.nftId === nftId);
    if (index === -1) {
      selectedNfts.push({ nftId: nftId, nftTicker:nftTicker });

    } else {
      selectedNfts.splice(index, 1);
    }

    if(selectedNfts.length>0){
      for(let i=0; i<selectedNfts.length ;i++)
      console.log("Selected Nfts\n"+selectedNfts[i].nftId+" "+selectedNfts[i].nftTicker+" "+selectedNfts.length);
    }else{console.log("list empty");}
    
    
  }

async function sendNft(from:string,to:string,){

  
let expiration_date = new Date(new Date().valueOf() + 36000000);
let sendNftSb= new ScriptBuilder();

sendNftSb.AllowGas(from, Address.Null, gasPrice, gasLimit);

selectedNfts.forEach((nft)=>{
  sendNftSb.CallInterop("Runtime.TransferToken", [from, to, nft.nftTicker, nft.nftId ]);
})

sendNftSb.SpendGas(from);

let sendNftScript = sendNftSb.EndScript();

let sendNftTransaction = new Transaction(network, chain, sendNftScript, expiration_date ,payload);

sendNftTransaction.signWithKeys(keys);

try {
    const rawTx = Base16.encodeUint8Array(sendNftTransaction.ToByteAray(true));
    console.log("Raw transaction: ", rawTx);
    const txHash = await RPC.sendRawTransaction(rawTx);
    console.log("Transaction hash: ", txHash);
    await delay(5000);
    let result = await RPC.getTransaction(txHash);
    console.log("Transaction result: ", result.state);
    return result;
} catch (error) {
    console.error('Error during transaction:', error);
    throw error;
}
  }

  getBalances().then(()=>checkFeeTokenBalanceIsEnough());
</script>

<div class="mx-10 text-gray-900 dark:text-white">
  <div class="mb-4 flex items-center justify-center">
    <p class="font-bold text-xl">Balances for {address}</p>
  </div>
  <div class="flex justify-center">
    <div class="grid gap-2 grid-cols-1">
      {#each tokenBalances as tokenBalance}
        <Card padding="md" size="lg" class="w-full">
          <div class="flex items-center space-x-2">
            <div class="grid grid-cols-1">
              {@html `
                <div class="grid grid-cols-1">
                  
                  
                  ${tokenBalance.Ticker!=="SOUL" ?
                  `<img
                    src='../src/assets/${tokenBalance.Ticker}.png'
                    alt=${tokenBalance.Ticker}
                    class="flex-shrink-0"
                    style="width: 40px; height: 40px; object-fit: contain !important; " 
                    onerror="this.onerror=null;this.src='../src/assets/UNKNOWN.png';"
                    
                   />
                   `
                   :` ${tokenBalance.Ticker==="SOUL" && !isCrowner ?
                  `<img
                    src="../src/assets/SOUL.png"
                    alt={tokenBalance.Ticker}
                    class="flex-shrink-0"
                    style="width: 40px; height: 40px; object-fit: contain !important; " 
                    
                   />
                   `
                   :`${isCrowner && tokenBalance.Ticker==="SOUL" ?
                  `<img
                    src="../src/assets/crowner.png"
                    class="flex-shrink-0"
                    style="width: 40px; height: 40px; object-fit: contain !important; " 
                  />`
                  : ''
                }`
                  
                }
                   
                `}
                   

                  ${isSoulMaster && tokenBalance.Ticker==="SOUL" ?
                    `<img
                      src="../src/assets/soul_master.png"
                      alt="SM"
                      class="flex-shrink-0"
                      style="width: 40px; height: 40px; object-fit: contain !important;"
                      
                    />`
                    : ''
                  }
                  
  
                </div>
              `}
            </div>
            
            <div class="grid grid-cols-1 flex-grow">
              <p class="text-lg font-bold text-gray-900 truncate dark:text-white">{tokenBalance.Ticker}</p>
              <p class="text-md font-semibold text-gray-600 truncate dark:text-gray-200">{tokenBalance.Amount}</p>
              
              {#if tokenBalance.Ticker == "SOUL" && stakedSoul != undefined && !isStakeAmountZero}
                <p class="text-sm text-gray-600 truncate dark:text-gray-400">{stakedSoul} Staked</p>
                <p class="text-xs text-blue-700 truncate dark:text-blue-400">{timeRemainedForUnstake}</p>
                {#if isSoulMaster}
                  <p class="text-xs text-blue-700 truncate dark:text-blue-400">{soulMasterReward.toFixed(4)} Soul estimated for SM Reward</p>
                  <p class="text-xs text-blue-700 truncate dark:text-blue-400 text-balance">{crownDistiributionInfo}</p>
                {/if}
              {:else if tokenBalance.Ticker == "KCAL" && unclaimedKcal != undefined }
                <p class="text-sm text-gray-600 truncate dark:text-gray-400">{unclaimedKcal} Unclaimed</p>
                {#if !isStakeAmountZero}
                  <p class="text-xs text-red-900 truncate dark:text-red-500">{dailyKcalGeneration} Kcal will be stacked after {remainedTimeForNextKcalGenerationInfo}</p>
                {/if}
              {/if}
            </div>
            <div class="grid grid-cols-1 justify-end">
              <Button class="w-24" size="xs" on:click={() => openSendTokenModal(tokenBalance.Ticker, tokenBalance.Amount,tokenBalance.Decimals)} outline >Send</Button> 
              {#if tokenBalance.Ticker==="SOUL"}
                 <Button class="w-24" size="xs" outline on:click={() => openStakeSoulModal(tokenBalance.Ticker, tokenBalance.Amount,tokenBalance.Decimals)}>Stake</Button> 
              <Button class="w-24" size="xs" outline disabled={!isUnstakeTime} on:click={() => openUnStakeSoulModal(tokenBalance.Ticker, stakedSoul,tokenBalance.Decimals)}>Unstake</Button>
              {/if}
              
              {#if isSoulMaster && tokenBalance.Ticker==="SOUL"}
                <Button class="w-24" size="xs" disabled={!isInflationTime} outline color="blue" on:click={()=>confirmTriggerInflation()}>Get Crown</Button>
              {/if}
              {#if isSoulMaster && tokenBalance.Ticker==="SOUL"}
                <Button class="w-24" size="xs" disabled={!isSoulMasterRewardTime} outline color="blue" on:click={()=>confirmSmReward()}>Sm Reward</Button>
              {/if}
              {#if unclaimedKcal!=0 && unclaimedKcal != undefined && tokenBalance.Ticker==="KCAL"}
                <Button class="w-24" size="xs" disabled={isUnclaimedKcalZero} outline color="blue" on:click={()=>confirmKcalClaim()}>Claim Kcal</Button>
              {/if}
            </div>
          </div>
          
        </Card>
        
      {/each}
    </div>
  </div>

  <div class="mx-10 my-4">
    <div class="mb-4 flex items-center justify-center">
      <p class="font-bold text-xl">NFT Balances</p>
    </div>
    <div class="flex justify-center">
      <div class="grid gap-2 grid-cols-1">
        {#each nftBalances as nftBalance}
          <Card padding="md" size="lg" class="w-full">
            <div class="flex items-center space-x-2 rtl:space-x-reverse">
              {@html `
                <div class="grid grid-cols-1">
                  <img
                    src="../src/assets/${nftBalance.Ticker}.png"
                    alt={tokenBalance.Ticker}
                    class="flex-shrink-0"
                    style="width: 40px; height: 40px; object-fit: contain !important; " 
                    onerror="this.onerror=null;this.src='../src/assets/UNKNOWN.png';"
                  />
                  `}
              <div class="flex-1 min-w-0">
                <p class="text-lg font-medium text-gray-900 truncate dark:text-white">{nftBalance.Ticker}</p>
                <p class="text-md text-gray-600 truncate dark:text-gray-200">{nftBalance.Amount}</p>
                {#if nftBalance.Ticker == "CROWN"}
                  <p class="text-xs text-gray-400 truncate dark:text-gray-400">Kcal Generation {kcalGenerationBoost}% Boosted</p >
                  <p class="text-xs text-blue-700 truncate dark:text-blue-400">Next Crown Distribution is after {nextInflationDateInfo}</p>
                {/if}
              </div>
              <div class="grid grid-cols-1 justify-end">
                <Button class="w-24" size="xs" id="NftSendButton" outline on:click={()=>openSendNftModal(nftBalance.Ticker,nftBalance.Ids) } disabled={NftSendButtonDisable} >{#if NftSendButtonDisable} 
                  <Spinner size="5" class="text-left dark:fill-white fill-black"  />
                {/if} Send</Button> 
              </div>
            </div>
          </Card>
        {/each}
      </div>
    </div>
  </div>
</div>

<Modal bind:open={sendTokenModal} size="xs" autoclose={false} class="w-full">
  <form on:submit|preventDefault={handleSendToken} class="flex flex-col space-y-6" action="">
    <h3 class="mb-4 text-xl font-medium text-gray-900 dark:text-white">Send {modalTokenTicker}</h3>
    <Label class="space-y-2">
      <span>To</span>
      <Input type="text" name="To" placeholder="P2K..." required bind:value={toAddress}  />
    </Label>
    <Label class="space-y-2">
      <span class="flex justify-between"><p>Amount to send</p> <button id="copyMaxAmountToken" on:click={copyMaxAmount} style="cursor: pointer;">Max {modalMaxAmount}</button></span>
      <Input type="number" name="sendAmount" required bind:value={sendAmount} max={modalMaxAmount} min="0" step="any" pattern="[0-9]+(\.[0-9]+)?" />
    
    </Label>
    <Label>Adjust Fee</Label>
    <Range id="range-steps" min="10000" max="200000" bind:value={gasLimit} on:change={()=>checkFeeTokenBalanceIsEnough()} step="{feeAdjustStep}" />
    <p>{feeAmount} Kcal, {feeWarning}</p>
    <Button  class="w-full1" type="submit">Send</Button>
    
  </form>
</Modal>

<Modal bind:open={stakeSoulModal} size="xs" autoclose={false} class="w-full">
  <form on:submit|preventDefault={handleStakeSoul} class="flex flex-col space-y-6" action="#">
    <h3 class="mb-4 text-xl font-medium text-gray-900 dark:text-white">Stake {modalTokenTicker}</h3>
    
    <Label class="space-y-2">
      <span class="flex justify-between"><p>Amount to stake</p> <button on:click={copyMaxAmount} id="copyMaxAmountStake" style="cursor: pointer;">Max {modalMaxAmount}</button><p>min {minStakeAmount}</p></span>
      <Input type="number" name="stakeAmount" required bind:value={sendAmount} max={modalMaxAmount} min={minStakeAmount} step="any" pattern="[0-9]+(\.[0-9]+)?" />
    </Label>
    
    <Button  class="w-full1" type="submit">Stake</Button>
    
  </form>
</Modal>

<Modal bind:open={unStakeSoulModal} size="xs" autoclose={false} class="w-full">
  <form on:submit|preventDefault={handleUnStakeSoul} class="flex flex-col space-y-6" action="#">
    <h3 class="mb-4 text-xl font-medium text-gray-900 dark:text-white">Unstake {modalTokenTicker}</h3>
    
    <Label class="space-y-2">
      <span class="flex justify-between"><p>Amount to stake</p> <button on:click={copyMaxAmount} style="cursor: pointer;">Max {modalMaxAmount}</button><p>min {minStakeAmount}</p></span>
      <Input type="number" name="unStakeAmount"   required bind:value={sendAmount} max={stakedSoul} min={minStakeAmount} step="any" pattern="[0-9]+(\.[0-9]+)?" />
    </Label>
    
    <Button  class="w-full1" type="submit">Unstake</Button>
    
  </form>
</Modal>

<Modal bind:open={confirmModal} size="md" autoclose={false} class="w-full">
  <div>
      <h3 class="mb-4 text-xl font-bold text-gray-900 dark:text-white">Are you sure ?</h3>
      {#each confirmModalMessage as ModalMessageLine}
         <p class="font-semibold text-lg text-blue-900 dark:text-blue-400">{ModalMessageLine}</p>
      {/each}
        <p class="font-semibold text-sm text-red-900 dark:text-red-400">{feeWarning}</p>
      
      <Button outline color="blue" disabled={!isFeeBalanceEnough} on:click={() => {
          isUserClickedConfirmModalButtons=true;
          isTransactionConfirmed = true;
          confirmModal = false; // Close the modal after confirming
      }} class="w-full">Yes</Button>
      <Button outline color="red" on:click={() => {
          isUserClickedConfirmModalButtons=true;
          isTransactionConfirmed = false;
          confirmModal = false; // Close the modal if canceled
      }} class="w-full">Cancel</Button>
  </div>
</Modal>

<Modal bind:open={sendNftModal} size="md" autoclose={false} class="w-full" id="sendNftModal">
  <form on:submit|preventDefault={handleSendNft} class="flex flex-col space-y-2" action="">
    <h3 class="mb-4 text-xl font-medium text-gray-900 dark:text-white">Send {modalNftTicker} NFT</h3>
    <Label class="space-y-2">
      <span>To</span>
      <Input type="text" name="To" placeholder="P2K..." required bind:value={toAddress} />
    </Label>

    <p class="mb-5 text-lg font-medium text-gray-900 dark:text-white">Select Nft:</p>
    <div class="grid gap-2 w-full md:grid-cols-3">
      {#each dataForShowSendNft as nftData}
        <Checkbox custom on:click={()=>addRemoveFromNftList(nftData.id,nftData.ticker)} class="h-30 mb-2 px-2 py-2">
          <div class="font-normal p-2 w-full text-gray-500 bg-white rounded-lg border-4 border-gray-200 cursor-pointer dark:hover:text-gray-300 dark:border-gray-700 peer-checked:border-primary-600 hover:text-gray-600 dark:peer-checked:text-gray-300 peer-checked:text-gray-600 hover:bg-gray-50 dark:text-gray-400 dark:bg-gray-800 dark:hover:bg-gray-700 overflow-hidden">
            <div class="text-sm font-semibold text-nowrap text-clip flex">{getNftName(nftData)}</div>
          
            <Textarea name="nftDesc" id="nftDesc" readonly value={getNftDescription(nftData)} class="overflow-auto" />

            
           
              <div class="mt-2 text-xs text-gray-500 dark:text-gray-400 overflow-y-auto">
              {#each nftData.infusion as infused}
                <div>{infused.key}: {infused.value}</div>
              {/each}
            </div>
            <Button class="min-w-full" size="xs" on:click={() => goToMarketPage(modalNftTicker, nftData.id)}>View</Button>
          </div>
        </Checkbox>
      {/each}
    </div>

    <Label>Adjust Fee</Label>
    <Range id="range-steps" min="10000" max="200000" bind:value={gasLimit} on:change={()=>checkFeeTokenBalanceIsEnough()} step="{feeAdjustStep}" />
    <p>{feeAmount} Kcal, {feeWarning}</p>
    <Button class="w-full1" type="submit">Send</Button>
  </form>
</Modal>

<Toast  position="bottom-right" bind:open={openInfoToast} class="text-balance">{InfoMessage}</Toast>


  









