<script lang="ts">
  import BigNumber from "bignumber.js";
  import {
    PhantasmaAPI,
    ScriptBuilder,
    Decoder,
    Base16,
    PhantasmaKeys,
    Transaction,
    Address,
  } from "phantasma-ts/html/phantasma";
  import { Input, Label, Checkbox, A } from "flowbite-svelte";

  import { Card, Listgroup, Avatar, Button } from "flowbite-svelte";
  import { FloatingLabelInput, Helper } from "flowbite-svelte";
  import { bigIntToByteArray } from "phantasma-ts";

  //const dotenv = require("dotenv");
  //dotenv.config();

  // Set up Phantasma API and contract name
  const NexusName = "mainnet"; // or 'mainnet'
  const ChainName = "main"; // The chain name
  const RPC = new PhantasmaAPI( // mainnet: https://pharpc1.phantasma.info/rpc Testnet: https://testnet.phantasma.info/rpc
    "https://pharpc1.phantasma.info/rpc",
    null,
    NexusName
  );

  const wif = import.meta.env.VITE_PRIVATE_KEY; // **************Replace with your WIF key**********************
  //console.log(wif);
  const keys = PhantasmaKeys.fromWIF(wif);
  const expiration = new Date(new Date().getTime() + 36000000); // Transaction expiration (1 hour from now)
  const gasPrice = 100000;
  const gasLimit = 75000;
  const payload = Base16.encode("Ebisu");
  const contractAddress = "SATRN"; // contract name
  const ContractName = "SATRN"; // Ensure this is correctly set in your .env file
  let profitAmount = 0;
  let arbAmount = 50;
  let optimizedAmount = 0;
  let tokenForArbitrage="SOUL";
  const minProfit=0.75/100;
  const feeTokenTicker="KCAL";
  let isFeeTokenEnough=false;
  let isHaveFeeToken=false;
  let isHaveArbitrageToken=false;
  let isOptimizationSuccessfull=false;
  let isAuto=false;
  
  let feeTokenBalance: {
    Ticker: string;
    Amount: BigNumber;
    Decimals: number;
  }={Ticker:"",Amount:BigNumber(0),Decimals:0};
  let arbitrageTokenBalance: {
    Ticker: string;
    Amount: BigNumber;
    Decimals: number;
  }={Ticker:"",Amount:BigNumber(0),Decimals:0};


  let PoolsForArbitrage: {
    PoolName: string;
    Token1Ticker: string;
    Token1Amount: string;
    Token1Decimals: number;
    Token2Ticker: string;
    Token2Amount: string;
    Token2Decimals: number;
    IsRelativePricePool: boolean;
    IsDivisorPool: boolean;
  }[] = [];

  let TokenPairs: {
    Token1Ticker: string;
    Token1Decimals: number;
    Token2Ticker: string;
    Token2Decimals: number;
  }[] = [];

  TokenPairs.push(
    {
      Token1Ticker: "SOUL",
      Token1Decimals: 8,
      Token2Ticker: "KCAL",
      Token2Decimals: 10,
    },
    {
      Token1Ticker: "RAA",
      Token1Decimals: 18,
      Token2Ticker: "SOUL",
      Token2Decimals: 8,
    },
    {
      Token1Ticker: "RAA",
      Token1Decimals: 18,
      Token2Ticker: "KCAL",
      Token2Decimals: 10,
    }
  );

  let InPoolForArb: {
    PoolName: string;
    InTicker: string;
    IntickerDecimals: number;
    InAmount: number;
    OutTicker: string;
    OutTickerDecimals: number;
    IsDivisorPool: boolean;
    IsRelativePricePool: boolean;
    InTickerReserve: string;
    OutTickerReserve: string;
  } = {
    PoolName: "",
    InTicker: "",
    InAmount: 0,
    OutTicker: "",
    IntickerDecimals: 0,
    OutTickerDecimals: 0,
    IsDivisorPool: false,
    IsRelativePricePool: false,
    InTickerReserve: "",
    OutTickerReserve: "",
  };
  let OutPoolForArb: {
    PoolName: string;
    InTicker: string;
    IntickerDecimals: number;
    InAmount: number;
    OutTicker: string;
    OutTickerDecimals: number;
    IsDivisorPool: boolean;
    IsRelativePricePool: boolean;
    InTickerReserve: string;
    OutTickerReserve: string;
  } = {
    PoolName: "",
    InTicker: "",
    InAmount: 0,
    OutTicker: "",
    IntickerDecimals: 0,
    OutTickerDecimals: 0,
    IsDivisorPool: false,
    IsRelativePricePool: false,
    InTickerReserve: "",
    OutTickerReserve: "",
  };
  let TransitPoolForArb: {
    PoolName: string;
    InTicker: string;
    IntickerDecimals: number;
    InAmount: number;
    OutTicker: string;
    OutTickerDecimals: number;
    IsDivisorPool: boolean;
    IsRelativePricePool: boolean;
    InTickerReserve: string;
    OutTickerReserve: string;
  } = {
    PoolName: "",
    InTicker: "",
    InAmount: 0,
    OutTicker: "",
    IntickerDecimals: 0,
    OutTickerDecimals: 0,
    IsDivisorPool: false,
    IsRelativePricePool: false,
    InTickerReserve: "",
    OutTickerReserve: "",
  };

  

  async function getPoolReserves() {
    let updatedPoolsForArbitrage: {
    PoolName: string;
    Token1Ticker: string;
    Token1Amount: string;
    Token1Decimals: number;
    Token2Ticker: string;
    Token2Amount: string;
    Token2Decimals: number;
    IsRelativePricePool: boolean;
    IsDivisorPool: boolean;
  }[] = [];
    console.log("Token Pairs\n" + TokenPairs + TokenPairs.length);

    // Map each pair to a Promise
    const promises = TokenPairs.map(async (pairElement): Promise<void> => {
      let sb = new ScriptBuilder();
      sb.CallContract(ContractName, "getTokenPairAndReserveKeysOnListVALUE", [
        pairElement.Token1Ticker +
          "_" +
          pairElement.Token2Ticker +
          "_" +
          pairElement.Token1Ticker,
      ]);
      sb.CallContract(ContractName, "getTokenPairAndReserveKeysOnListVALUE", [
        pairElement.Token1Ticker +
          "_" +
          pairElement.Token2Ticker +
          "_" +
          pairElement.Token2Ticker,
      ]);
      const script = sb.EndScript();

      let amounts = [];

      try {
        const response = await RPC.invokeRawScript("main", script);

        response.results.forEach((element) => {
          const decoder = new Decoder(element);
          amounts.push(decoder.readVmObject());
        });
      } catch (error) {
        console.log("An error happened" + error);
      }

      updatedPoolsForArbitrage.push({
        PoolName: pairElement.Token1Ticker + "_" + pairElement.Token2Ticker,
        Token1Ticker: pairElement.Token1Ticker,
        Token1Amount: BigNumber(amounts[0])
          .shiftedBy(-pairElement.Token1Decimals)
          .toFixed(),
        Token1Decimals: pairElement.Token1Decimals,
        Token2Ticker: pairElement.Token2Ticker,
        Token2Amount: BigNumber(amounts[1])
          .shiftedBy(-pairElement.Token2Decimals)
          .toFixed(),
        Token2Decimals: pairElement.Token2Decimals,
        IsRelativePricePool: false,
        IsDivisorPool: false,
      });
    });

    

    

    // Wait for all promises to resolve
    await Promise.all(promises);

    // updatedPoolsForArbitrage=[];
    // updatedPoolsForArbitrage.push(
    //   {PoolName: "SOUL_KCAL",
    //     Token1Ticker: "SOUL",
    //     Token1Amount: "3012.20230435",
    //     Token1Decimals: 8,
    //     Token2Ticker: "KCAL",
    //     Token2Amount: "542556.413469642",
    //     Token2Decimals: 10,
    //     IsRelativePricePool: false,
    //     IsDivisorPool: false,
    //   },
        
    //    { PoolName: "RAA_SOUL",
    //     Token1Ticker: "RAA",
    //     Token1Amount: "70.1126716334343",
    //     Token1Decimals: 18,
    //     Token2Ticker: "SOUL",
    //     Token2Amount: "6157.88615802196",
    //     Token2Decimals: 8,
    //     IsRelativePricePool: false,
    //     IsDivisorPool: false},

    //     {PoolName: "RAA_KCAL",
    //     Token1Ticker: "RAA",
    //     Token1Amount: "24.5154566825775",
    //     Token1Decimals: 18,
    //     Token2Ticker: "KCAL",
    //     Token2Amount: "441628.920535605",
    //     Token2Decimals: 10,
    //     IsRelativePricePool: false,
    //     IsDivisorPool: false,}

    // );


    PoolsForArbitrage = updatedPoolsForArbitrage;

    // Now all PoolsForArbitrage should be populated
  }

  function calculateSwapOutV2(
    inAmount: number,
    inReserves: number,
    outReserves: number
  ) {
    let outAmount = 0;
    try {
      if (inAmount > 0 && inReserves > 0 && outReserves > 0) {
        outAmount =
          outReserves -
          (inReserves * outReserves) / ((inAmount * 997) / 1000 + inReserves);

        console.log("Calculate Swap Amount V2_" + outAmount);
      } else {
        throw new Error(
          "Cannot Calculate Out Amount With Below Parameters\n" +
            "In Amount_" +
            inAmount +
            "\nIn Reserves_" +
            inReserves +
            "\nOut Reserves_" +
            outReserves
        );
      }
    } catch (error) {
      console.log(error);
    }

    return outAmount;
  }

  function arrangePoolTickersForCorrectRouteCalculation(
    ArrangeTokenTicker: string
  ) {
    /*
   *******************************************************************************************************************************
    selecting relative price pool then arranging tickers for correct pool division to compare relative pool and calculated pool 
    ie if soul_kcal is relative pool and other 2 raa/kcal and raa/soul you need to calculate it (soul/raa)/(kcal/raa) 
    in order to reach calculated soul_kcal Pool price correctly
   *********************************************************************************************************************************
   */
    let selectRelativePricePool = false;
    let counter = 0;
    let relativePricePoolSecondaryTicker = "";
    let relativePricePoolPrimaryTicker = "";

    PoolsForArbitrage.forEach((element) => {
      // checking if relative pool price selected for calculating price correctly against a token

      if (element.IsRelativePricePool == false) {
        counter++;
        console.log(counter);
      }
    });

    if (counter == PoolsForArbitrage.length) {
      selectRelativePricePool = true;
      console.log("need to select relative pool");
    }

    PoolsForArbitrage.forEach((element) => {// checking pool tickers correctly arranged for calculation, wanted to be sure first token is arrangetokenticker
      
      if (element.Token2Ticker === ArrangeTokenTicker) {
        let SwapToken1Ticker = element.Token1Ticker;
        let SwapToken1Amount = element.Token1Amount;
        let SwapToken1Decimals = element.Token1Decimals;

        let SwapToken2Ticker = element.Token2Ticker;
        let SwapToken2Amount = element.Token2Amount;
        let SwapToken2Decimals = element.Token2Decimals;

        element.Token1Ticker = SwapToken2Ticker;
        element.Token1Amount = SwapToken2Amount;
        element.Token1Decimals = SwapToken2Decimals;

        element.Token2Ticker = SwapToken1Ticker;
        element.Token2Amount = SwapToken1Amount;
        element.Token2Decimals = SwapToken1Decimals;
      }

      if (
        selectRelativePricePool == true &&
        (element.Token1Ticker == ArrangeTokenTicker ||
          element.Token2Ticker == ArrangeTokenTicker)
      ) {
        // if reletive price pool not selected pick a pool which have arrangetoken ticker
        element.IsRelativePricePool = true;
        relativePricePoolSecondaryTicker = element.Token2Ticker;
        relativePricePoolPrimaryTicker = element.Token1Ticker;
        selectRelativePricePool = false;
      }
    });

    PoolsForArbitrage.forEach((element) => {
      /*
      ***************************************************************************************************************
      arranging divisor pool tickers for correct calculation for converted pool 
      ie converted pool for selected relative pool RAA/SOUL calculation should be like this 
      converted Pool price= (RAA/KCAL)/(SOUL/KCAL), if divisor pool is KCAL/SOUL we will get wrong calculation
      *********************************************************************************************************************
      */
      if (
        element.Token2Ticker == relativePricePoolSecondaryTicker &&
        element.IsRelativePricePool == false
      ) {
        let SwapToken1Ticker = element.Token1Ticker;
        let SwapToken1Amount = element.Token1Amount;
        let SwapToken1Decimals = element.Token1Decimals;

        let SwapToken2Ticker = element.Token2Ticker;
        let SwapToken2Amount = element.Token2Amount;
        let SwapToken2Decimals = element.Token2Decimals;

        element.Token1Ticker = SwapToken2Ticker;
        element.Token1Amount = SwapToken2Amount;
        element.Token1Decimals = SwapToken2Decimals;

        element.Token2Ticker = SwapToken1Ticker;
        element.Token2Amount = SwapToken1Amount;
        element.Token2Decimals = SwapToken1Decimals;

        element.IsDivisorPool = true;
      } else if (
        element.Token1Ticker == relativePricePoolSecondaryTicker &&
        element.IsRelativePricePool == false
      ) {
        element.IsDivisorPool = true;
      }
    });

    PoolsForArbitrage.forEach((element)=>{

      /*
      ***************************************************************************************************************
      arranging divident pool tickers for correctly calculate converted pool 
      ie converted pool for selected relative pool RAA/SOUL calculation should be like this 
      converted Pool price= (RAA/KCAL)/(SOUL/KCAL), if divisor pool is KCAL/SOUL we will get wrong calculation
      *********************************************************************************************************************
      */

      if(!element.IsDivisorPool && !element.IsRelativePricePool && element.Token2Ticker == relativePricePoolPrimaryTicker ){

        let SwapToken1Ticker = element.Token1Ticker;
        let SwapToken1Amount = element.Token1Amount;
        let SwapToken1Decimals = element.Token1Decimals;

        let SwapToken2Ticker = element.Token2Ticker;
        let SwapToken2Amount = element.Token2Amount;
        let SwapToken2Decimals = element.Token2Decimals;

        element.Token1Ticker = SwapToken2Ticker;
        element.Token1Amount = SwapToken2Amount;
        element.Token1Decimals = SwapToken2Decimals;

        element.Token2Ticker = SwapToken1Ticker;
        element.Token2Amount = SwapToken1Amount;
        element.Token2Decimals = SwapToken1Decimals;
      }

    });
    console.log("\n%c********************_Arrranged pools_**********************\n,",'background:  #55fe01  ; color:   #fe0101 ');
    PoolsForArbitrage.forEach((element) => {
      
      console.log(
        element.PoolName +
          "\n" +
          "Token 1 ticker_" +
          element.Token1Ticker +
          " Amount_" +
          element.Token1Amount +
          " Decimals_" +
          element.Token1Decimals +
          "\n" +
          "Token 2 ticker_" +
          element.Token2Ticker +
          " Amount_" +
          element.Token2Amount +
          " Decimals_" +
          element.Token2Decimals +
          "\n" +
          "Is relative price pool_" +
          element.IsRelativePricePool +"\n"+
          "Is divider pool_" +
          element.IsDivisorPool
      );
    });
  }

  function calculatePoolRatios(token1Amount: number, token2Amount: number) {
    let poolRatios = token1Amount / token2Amount;
    return poolRatios;
  }

  function calculateArbitrageAmounts(inAmount: number) {
    InPoolForArb.InAmount = inAmount;

    // TransitPoolForArb.InAmount = calculateSwapOut(
    //     InPoolForArb.PoolName,
    //     InPoolForArb.InAmount,
    //     InPoolForArb.InTicker,
    // );

    TransitPoolForArb.InAmount = calculateSwapOutV2(
      InPoolForArb.InAmount,
      BigNumber(InPoolForArb.InTickerReserve).toNumber(),
      BigNumber(InPoolForArb.OutTickerReserve).toNumber()
    );

    // OutPoolForArb.InAmount = calculateSwapOut(
    //     TransitPoolForArb.PoolName,
    //     TransitPoolForArb.InAmount,
    //     TransitPoolForArb.InTicker,
    // );

    OutPoolForArb.InAmount = calculateSwapOutV2(
      TransitPoolForArb.InAmount,
      BigNumber(TransitPoolForArb.InTickerReserve).toNumber(),
      BigNumber(TransitPoolForArb.OutTickerReserve).toNumber()
    );

    // let outAmount = calculateSwapOut(
    //     OutPoolForArb.PoolName,
    //     OutPoolForArb.InAmount,
    //     OutPoolForArb.InTicker,
    // );

    let outAmount = calculateSwapOutV2(
      OutPoolForArb.InAmount,
      BigNumber(OutPoolForArb.InTickerReserve).toNumber(),
      BigNumber(OutPoolForArb.OutTickerReserve).toNumber()
    );

    profitAmount = outAmount - inAmount;

    console.log(
      "Profit Amount_" + profitAmount + "_" + OutPoolForArb.OutTicker
    );

    console.log(
      "In Pool_" +
        InPoolForArb.PoolName +
        " " +
        "In Amount_" +
        inAmount +
        "_" +
        InPoolForArb.InTicker +
        " Raw_" +
        BigNumber(inAmount.toString())
          .shiftedBy(InPoolForArb.IntickerDecimals)
          .decimalPlaces(0)
          .toFixed()
    );
    console.log(
      "In Amount for Transit Pool_" +
        TransitPoolForArb.InAmount +
        "_" +
        TransitPoolForArb.InTicker +
        " Raw_" +
        BigNumber(TransitPoolForArb.InAmount.toString())
          .shiftedBy(TransitPoolForArb.IntickerDecimals)
          .decimalPlaces(0)
          .toFixed()
    );
    console.log(
      "In Amount for Out Pool _" +
        OutPoolForArb.InAmount +
        "_" +
        OutPoolForArb.InTicker +
        " Raw_" +
        BigNumber(OutPoolForArb.InAmount.toString())
          .shiftedBy(OutPoolForArb.IntickerDecimals)
          .decimalPlaces(0)
          .toFixed()
    );
    console.log(
      "OutPool_" +
        OutPoolForArb.PoolName +
        " OutAmount_" +
        outAmount +
        "_" +
        OutPoolForArb.OutTicker +
        " Raw_" +
        BigNumber(outAmount.toString())
          .shiftedBy(OutPoolForArb.OutTickerDecimals)
          .decimalPlaces(0)
          .toFixed()
    );
  }

  async function findArbSwapRoute(
    tokenToincrease: string
    
  ) {
    

    arrangePoolTickersForCorrectRouteCalculation(tokenToincrease);
    console.log("Pool For arb have " + PoolsForArbitrage.length + " Pools");

    let relativePricePoolPrice;
    let divisorPoolPrice;
    let dividentPoolPrice;
    let relativePoolInfo: {
      PoolName: string;
      Token1: string;
      Token1Decimals: number;
      Token2: string;
      Token2Decimals: number;
      IsRelativePricePool: boolean;
      IsDivisorPool: boolean;
      Token1Reserve: string;
      Token2Reserve: string;
    } = {
      PoolName: "",
      Token1: "",
      Token2: "",
      Token1Decimals: 0,
      Token2Decimals: 0,
      IsDivisorPool: false,
      IsRelativePricePool: false,
      Token1Reserve: "",
      Token2Reserve: "",
    };
    let divisorPoolInfo: {
      PoolName: string;
      Token1: string;
      Token1Decimals: number;
      Token2: string;
      Token2Decimals: number;
      IsRelativePricePool: boolean;
      IsDivisorPool: boolean;
      Token1Reserve: string;
      Token2Reserve: string;
    } = {
      PoolName: "",
      Token1: "",
      Token2: "",
      Token1Decimals: 0,
      Token2Decimals: 0,
      IsDivisorPool: false,
      IsRelativePricePool: false,
      Token1Reserve: "",
      Token2Reserve: "",
    };
    let transitPoolInfo: {      //this is not in or out pool middle pool
      
      PoolName: string;
      Token1: string;
      Token1Decimals: number;
      Token2: string;
      Token2Decimals: number;
      IsRelativePricePool: boolean;
      IsDivisorPool: boolean;
      Token1Reserve: string;
      Token2Reserve: string;
    } = {
      PoolName: "",
      Token1: "",
      Token2: "",
      Token1Decimals: 0,
      Token2Decimals: 0,
      IsDivisorPool: false,
      IsRelativePricePool: false,
      Token1Reserve: "",
      Token2Reserve: "",
    }; 

    PoolsForArbitrage.forEach((element) => {
      if (element.IsRelativePricePool) {
        relativePricePoolPrice = calculatePoolRatios(
          BigNumber(element.Token1Amount).toNumber(),
          BigNumber(element.Token2Amount).toNumber()
        );
        relativePoolInfo.Token1 = element.Token1Ticker;
        relativePoolInfo.Token2 = element.Token2Ticker;
        relativePoolInfo.PoolName = element.PoolName;
        relativePoolInfo.Token1Decimals = element.Token1Decimals;
        relativePoolInfo.Token2Decimals = element.Token2Decimals;
        relativePoolInfo.IsDivisorPool = element.IsDivisorPool;
        relativePoolInfo.IsRelativePricePool = element.IsRelativePricePool;
        relativePoolInfo.Token1Reserve = element.Token1Amount;
        relativePoolInfo.Token2Reserve = element.Token2Amount;
      } else if (element.Token1Ticker!=tokenToincrease && element.Token2Ticker!=tokenToincrease) {
        divisorPoolPrice = calculatePoolRatios(
          BigNumber(element.Token1Amount).toNumber(),
          BigNumber(element.Token2Amount).toNumber()
        );
        transitPoolInfo.Token1 = element.Token1Ticker;
        transitPoolInfo.Token2 = element.Token2Ticker;
        transitPoolInfo.PoolName = element.PoolName;
        transitPoolInfo.Token1Decimals = element.Token1Decimals;
        transitPoolInfo.Token2Decimals = element.Token2Decimals;
        transitPoolInfo.IsDivisorPool = element.IsDivisorPool;
        transitPoolInfo.IsRelativePricePool = element.IsRelativePricePool;
        transitPoolInfo.Token1Reserve = element.Token1Amount;
        transitPoolInfo.Token2Reserve = element.Token2Amount;
      } else if (
        element.IsDivisorPool == false &&
        element.IsRelativePricePool == false
      ) {
        dividentPoolPrice = calculatePoolRatios(
          BigNumber(element.Token1Amount).toNumber(),
          BigNumber(element.Token2Amount).toNumber()
        );
        divisorPoolInfo.Token1 = element.Token1Ticker;
        divisorPoolInfo.Token2 = element.Token2Ticker;
        divisorPoolInfo.PoolName = element.PoolName;
        divisorPoolInfo.Token1Decimals = element.Token1Decimals;
        divisorPoolInfo.Token2Decimals = element.Token2Decimals;
        divisorPoolInfo.IsRelativePricePool = element.IsRelativePricePool;
        divisorPoolInfo.IsDivisorPool = element.IsDivisorPool;
        divisorPoolInfo.Token1Reserve = element.Token1Amount;
        divisorPoolInfo.Token2Reserve = element.Token2Amount;
      }
    });

    let convertedPoolPrice = dividentPoolPrice / divisorPoolPrice;

    if (relativePricePoolPrice > convertedPoolPrice) {//finding swap route
      

      InPoolForArb.InTicker = divisorPoolInfo.Token1;
      InPoolForArb.IntickerDecimals = divisorPoolInfo.Token1Decimals;
      InPoolForArb.OutTicker = divisorPoolInfo.Token2;
      InPoolForArb.OutTickerDecimals = divisorPoolInfo.Token2Decimals;
      InPoolForArb.PoolName = divisorPoolInfo.PoolName;
      InPoolForArb.IsRelativePricePool = divisorPoolInfo.IsRelativePricePool;
      InPoolForArb.IsDivisorPool = divisorPoolInfo.IsDivisorPool;
      InPoolForArb.InTickerReserve = divisorPoolInfo.Token1Reserve;
      InPoolForArb.OutTickerReserve = divisorPoolInfo.Token2Reserve;

      OutPoolForArb.InTicker = relativePoolInfo.Token2;
      OutPoolForArb.IntickerDecimals = relativePoolInfo.Token2Decimals;
      OutPoolForArb.OutTicker = relativePoolInfo.Token1;
      OutPoolForArb.OutTickerDecimals = relativePoolInfo.Token1Decimals;
      OutPoolForArb.PoolName = relativePoolInfo.PoolName;
      OutPoolForArb.IsRelativePricePool = relativePoolInfo.IsRelativePricePool;
      OutPoolForArb.IsDivisorPool = relativePoolInfo.IsDivisorPool;
      OutPoolForArb.InTickerReserve = relativePoolInfo.Token2Reserve;
      OutPoolForArb.OutTickerReserve = relativePoolInfo.Token1Reserve;

      TransitPoolForArb.InTicker = transitPoolInfo.Token1;
      TransitPoolForArb.IntickerDecimals = transitPoolInfo.Token1Decimals;
      TransitPoolForArb.OutTicker = transitPoolInfo.Token2;
      TransitPoolForArb.OutTickerDecimals = transitPoolInfo.Token2Decimals;
      TransitPoolForArb.PoolName = transitPoolInfo.PoolName;
      TransitPoolForArb.IsDivisorPool = transitPoolInfo.IsDivisorPool;
      TransitPoolForArb.IsRelativePricePool =
        transitPoolInfo.IsRelativePricePool;
      TransitPoolForArb.InTickerReserve = transitPoolInfo.Token1Reserve;
      TransitPoolForArb.OutTickerReserve = transitPoolInfo.Token2Reserve;

    } else if (relativePricePoolPrice < convertedPoolPrice) {
      InPoolForArb.InTicker = relativePoolInfo.Token1;
      InPoolForArb.IntickerDecimals = relativePoolInfo.Token1Decimals;
      InPoolForArb.OutTicker = relativePoolInfo.Token2;
      InPoolForArb.OutTickerDecimals = relativePoolInfo.Token2Decimals;
      InPoolForArb.PoolName = relativePoolInfo.PoolName;
      InPoolForArb.IsRelativePricePool = relativePoolInfo.IsRelativePricePool;
      InPoolForArb.IsDivisorPool = relativePoolInfo.IsDivisorPool;
      InPoolForArb.InTickerReserve = relativePoolInfo.Token1Reserve;
      InPoolForArb.OutTickerReserve = relativePoolInfo.Token2Reserve;

      OutPoolForArb.InTicker = divisorPoolInfo.Token2;
      OutPoolForArb.IntickerDecimals = divisorPoolInfo.Token2Decimals;
      OutPoolForArb.OutTicker = divisorPoolInfo.Token1;
      OutPoolForArb.OutTickerDecimals = divisorPoolInfo.Token1Decimals;
      OutPoolForArb.PoolName = divisorPoolInfo.PoolName;
      OutPoolForArb.IsDivisorPool = divisorPoolInfo.IsDivisorPool;
      OutPoolForArb.IsRelativePricePool = divisorPoolInfo.IsRelativePricePool;
      OutPoolForArb.InTickerReserve = divisorPoolInfo.Token2Reserve;
      OutPoolForArb.OutTickerReserve = divisorPoolInfo.Token1Reserve;

      TransitPoolForArb.InTicker = transitPoolInfo.Token1;
      TransitPoolForArb.IntickerDecimals = transitPoolInfo.Token1Decimals;
      TransitPoolForArb.OutTicker = transitPoolInfo.Token2;
      TransitPoolForArb.OutTickerDecimals = transitPoolInfo.Token2Decimals;
      TransitPoolForArb.PoolName = transitPoolInfo.PoolName;
      TransitPoolForArb.IsRelativePricePool =
        transitPoolInfo.IsRelativePricePool;
      TransitPoolForArb.IsDivisorPool = transitPoolInfo.IsDivisorPool;
      TransitPoolForArb.InTickerReserve = transitPoolInfo.Token1Reserve;
      TransitPoolForArb.OutTickerReserve = transitPoolInfo.Token2Reserve;
    }

    if (InPoolForArb.OutTicker === TransitPoolForArb.OutTicker) {
      //arranging out/in tickers

      let exchangeTicker1 = "";
      let exchangeTicker2 = "";
      let exchangeDecimal1 = 0;
      let exchangeDecimal2 = 0;
      let exchangeReserve1 = "0";
      let exchangeReserve2 = "0";

      exchangeTicker1 = TransitPoolForArb.InTicker;
      exchangeTicker2 = TransitPoolForArb.OutTicker;
      exchangeDecimal1 = TransitPoolForArb.IntickerDecimals;
      exchangeDecimal2 = TransitPoolForArb.OutTickerDecimals;
      exchangeReserve1 = TransitPoolForArb.InTickerReserve;
      exchangeReserve2 = TransitPoolForArb.OutTickerReserve;

      TransitPoolForArb.InTicker = exchangeTicker2;
      TransitPoolForArb.OutTicker = exchangeTicker1;
      TransitPoolForArb.IntickerDecimals = exchangeDecimal2;
      TransitPoolForArb.OutTickerDecimals = exchangeDecimal1;
      TransitPoolForArb.OutTickerReserve = exchangeReserve1;
      TransitPoolForArb.InTickerReserve = exchangeReserve2;
    }

    console.log(
      "Relative pool price_" +
        relativePricePoolPrice +
        "\n" +
        "Converted pool price_" +
        convertedPoolPrice
    );
    console.log("\n%c**************************_In/Out/Transit Pools_********************************\n"+
      
        "In Pool_" +
        InPoolForArb.InTicker +
        "_" +
        InPoolForArb.OutTicker +
        "\n" +
        "Transit Pool_" +
        TransitPoolForArb.InTicker +
        "_" +
        TransitPoolForArb.OutTicker +
        "\n" +
        "Out Pool_" +
        OutPoolForArb.InTicker +
        "_" +
        OutPoolForArb.OutTicker
    ,'background:  #55fe01  ; color:   #fe0101 ');
  }

  function delay(time) {
    return new Promise((resolve) => setTimeout(resolve, time));
  }

  async function sendArbTransaction() {
    if (profitAmount > 0) {
      const tokenIn = InPoolForArb.InTicker; // Symbol for Token In
      const tokenOut = InPoolForArb.OutTicker; // Symbol for Token Out
      const amountIn = new BigNumber(InPoolForArb.InAmount)
        .shiftedBy(InPoolForArb.IntickerDecimals)
        .decimalPlaces(0)
        .toFixed(); // Amount for Token In
      const slippage = 0; // slippage 0-99

      const tokenIn2 = TransitPoolForArb.InTicker; // Symbol for Token In
      const tokenOut2 = TransitPoolForArb.OutTicker; // Symbol for Token Out
      const amountIn2 = new BigNumber(TransitPoolForArb.InAmount)
        .shiftedBy(TransitPoolForArb.IntickerDecimals)
        .decimalPlaces(0)
        .toFixed(); // Amount for Token In
      const slippage2 = 0; // slippage 0-99

      const tokenIn3 = OutPoolForArb.InTicker; // Symbol for Token In
      const tokenOut3 = OutPoolForArb.OutTicker; // Symbol for Token Out
      const amountIn3 = new BigNumber(OutPoolForArb.InAmount)
        .shiftedBy(OutPoolForArb.IntickerDecimals)
        .decimalPlaces(0)
        .toFixed(); // Amount for Token In
      const slippage3 = 0;

      console.log("Address: ", keys.Address);
      console.log("Token In: ", tokenIn);
      console.log("Amount In: ", amountIn);
      console.log("Token Out: ", tokenOut);
      console.log("Slippage: ", slippage);

      console.log("Address: ", keys.Address);
      console.log("Token In2: ", tokenIn2);
      console.log("Amount In2: ", amountIn2);
      console.log("Token Out2: ", tokenOut2);
      console.log("Slippage2: ", slippage2);

      console.log("Address: ", keys.Address);
      console.log("Token In2: ", tokenIn3);
      console.log("Amount In2: ", amountIn3);
      console.log("Token Out2: ", tokenOut3);
      console.log("Slippage2: ", slippage3);

      const sb = new ScriptBuilder();
      sb.AllowGas(keys.Address, Address.Null, gasPrice, gasLimit);
      sb.CallContract(contractAddress, "swap", [
        keys.Address, // The address of the liquidity provider
        amountIn, // The amount of Token0
        tokenIn, // The symbol of Token0 (KCAL)
        tokenOut, // The symbol of Token1 (SOUL)
        slippage,
      ]);
      sb.CallContract(contractAddress, "swap", [
        keys.Address, // The address of the liquidity provider
        amountIn2, // The amount of Token0
        tokenIn2, // The symbol of Token0 (KCAL)
        tokenOut2, // The symbol of Token1 (SOUL)
        slippage2,
      ]);

      sb.CallContract(contractAddress, "swap", [
        keys.Address, // The address of the liquidity provider
        amountIn3, // The amount of Token0
        tokenIn3, // The symbol of Token0 (KCAL)
        tokenOut3, // The symbol of Token1 (SOUL)
        slippage3,
      ]);

      sb.SpendGas(keys.Address);
      const script = sb.EndScript();

      console.log("Transaction script: ", script);

      let transaction = new Transaction(
        NexusName,
        ChainName,
        script,
        expiration,
        payload
      );
      transaction.signWithKeys(keys);

      console.log("Transaction info: ", transaction);

      console.log("Sending Transaction: ");
      try {
        const rawTx = Base16.encodeUint8Array(transaction.ToByteAray(true));
        console.log("Raw transaction: ", rawTx);
        const txHash = await RPC.sendRawTransaction(rawTx);
        console.log("Transaction hash: ", txHash);
        await delay(5000);
        let result = await RPC.getTransaction(txHash);
        console.log("Transaction result: ", result);
        return result;
      } catch (error) {
        console.error("Error during transaction:", error);
        throw error;
      }
    } else {
      console.log(
        "Arbitrage is not profitable, profit amount: " + profitAmount
      );
    }
  }

  function optimizeArbAmountv2() {
    console.log("%c*****************OptimizationStarted***************\n"+InPoolForArb+"\n"+OutPoolForArb+"\n"+TransitPoolForArb+"\n",'background:  #55fe01  ; color:   #fe0101 ');
    
    

    let poolReserveDifference = 0;
    let stepPercent = 0.2; //20% error margin or correction rate
    let precision = 0.0025;

    if (InPoolForArb.IsRelativePricePool) {
      //Converting transit pool reserves to in/out pool reserves for token in/out which is same

      let relativePoolPrice = 0;

      relativePoolPrice =
        BigNumber(InPoolForArb.OutTickerReserve).toNumber() /
        BigNumber(InPoolForArb.InTickerReserve).toNumber();

      console.log("relativepool= in pool");

      console.log(relativePoolPrice);

      let conversionRateForTransitPoolInToken =
        BigNumber(TransitPoolForArb.InTickerReserve).toNumber() /
        BigNumber(InPoolForArb.OutTickerReserve).toNumber();

      let convertedTransitPoolTokenReserveForInPool =
        conversionRateForTransitPoolInToken *
        BigNumber(InPoolForArb.InTickerReserve).toNumber();

      console.log(convertedTransitPoolTokenReserveForInPool);

      let conversionRateForTransitPoolOutToken =
        BigNumber(TransitPoolForArb.OutTickerReserve).toNumber() /
        BigNumber(OutPoolForArb.InTickerReserve).toNumber();

      let convertedTransitPoolTokenReserveForOutPool =
        conversionRateForTransitPoolOutToken *
        BigNumber(OutPoolForArb.OutTickerReserve).toNumber();

      console.log(convertedTransitPoolTokenReserveForOutPool);

      poolReserveDifference = Math.abs(
        convertedTransitPoolTokenReserveForInPool -
          convertedTransitPoolTokenReserveForOutPool
      );
    } else {
      let relativePoolPrice = 0;

      relativePoolPrice =
        BigNumber(OutPoolForArb.OutTickerReserve).toNumber() /
        BigNumber(OutPoolForArb.InTickerReserve).toNumber();

      console.log("relativepool  = out pool");

      console.log(relativePoolPrice);

      let conversionRateForTransitPoolInToken =
        BigNumber(TransitPoolForArb.InTickerReserve).toNumber() /
        BigNumber(InPoolForArb.OutTickerReserve).toNumber();

      let convertedTransitPoolTokenReserveForInPool =
        conversionRateForTransitPoolInToken *
        BigNumber(InPoolForArb.InTickerReserve).toNumber();

      console.log(convertedTransitPoolTokenReserveForInPool);

      let convertedTransitPoolTokenReserveForOutPool =
        relativePoolPrice *
        BigNumber(TransitPoolForArb.OutTickerReserve).toNumber();

      console.log(convertedTransitPoolTokenReserveForOutPool);

      poolReserveDifference = Math.abs(
        convertedTransitPoolTokenReserveForInPool -
          convertedTransitPoolTokenReserveForOutPool
      );
    }

    let amountForZeroProfit =
      poolReserveDifference /
      (BigNumber(TransitPoolForArb.InTickerReserve).toNumber() /
        BigNumber(InPoolForArb.OutTickerReserve).toNumber() /
        (BigNumber(TransitPoolForArb.OutTickerReserve).toNumber() /
          BigNumber(OutPoolForArb.InTickerReserve).toNumber()));
    let shiftAmount = amountForZeroProfit * stepPercent;
    let tries = 0;

    console.log(
      "initial Values \n amount for zero profit_" +
        amountForZeroProfit +
        "\n Initial Shift Amount_" +
        shiftAmount +
        "Step Percent_" +
        stepPercent
    );
    const maxGuess=2*amountForZeroProfit;
    const minGuess=2*minProfit;
    calculateArbitrageAmounts(amountForZeroProfit);

    while (tries < 50) {
      tries++;
      
       
      console.log(
        "StepPercent_" + stepPercent + "\nShift Amount_" + shiftAmount +"\nMin guess "+minGuess + "\nMax Guess "+maxGuess +"\nAmount For Zero Profit_" + amountForZeroProfit
      );
      
      if (BigNumber(profitAmount).isLessThan(0) && BigNumber(amountForZeroProfit).isGreaterThan(shiftAmount)) {
        amountForZeroProfit = amountForZeroProfit - shiftAmount;
        calculateArbitrageAmounts(amountForZeroProfit);

        if (profitAmount > 0) {
          stepPercent = stepPercent / 2;
          shiftAmount = amountForZeroProfit * stepPercent;
        }
        
      } else if (profitAmount>0 && amountForZeroProfit<maxGuess) {
        amountForZeroProfit = amountForZeroProfit + shiftAmount;
        calculateArbitrageAmounts(amountForZeroProfit);

        if (profitAmount < 0 ) {
          stepPercent = stepPercent / 2;
          shiftAmount = amountForZeroProfit * stepPercent;
        }
      }

      if(BigNumber(shiftAmount).isGreaterThanOrEqualTo(amountForZeroProfit)){
        shiftAmount=shiftAmount/2;
      }

      console.log("Shift amount after check"+shiftAmount)

      if(BigNumber(amountForZeroProfit).isLessThanOrEqualTo(minGuess)){
        console.log("amountForZeroProfit less than min guess")
        isOptimizationSuccessfull=false;
        break;
      }

      if (stepPercent < precision) {
        console.log("max precision reached");
        break;
      }

      
    }
    if (profitAmount>minProfit){
      isOptimizationSuccessfull=true;
    }else{isOptimizationSuccessfull=false;
      arbAmount=0;
    }

    console.log("%c*****************************************************************\n"+"*  Optimization ended after trying "+ tries+" times, is success "+isOptimizationSuccessfull +"  *" + "*\n"+"*****************************************************************",'background:  #55fe01  ; color:   #fe0101 ');
    if(isOptimizationSuccessfull){

      optimizedAmount = amountForZeroProfit / 2;
      arbAmount = optimizedAmount;
      calculateArbitrageAmounts(arbAmount);
    }
    
  }


async function checkBalancesIsCorrectForArb() {
  let updatedFeeTokenBalance: {
    Ticker: string;
    Amount: BigNumber;
    Decimals: number;
  };
  let updatedArbitrageTokenBalance: {
    Ticker: string;
    Amount: BigNumber;
    Decimals: number;
  };
  isHaveFeeToken=false;
  isHaveArbitrageToken=false;
  try {

      const balanceResponseForArb = await RPC.getAccount(keys.Address.toString(),true);

      console.log("Balances for arb\n"+balanceResponseForArb);
      const balancePromises = balanceResponseForArb.balances.map(async (element): Promise<void> => {
      
        console.log("balance"+element.amount+element.symbol);

      if(element.symbol===tokenForArbitrage && element.symbol===feeTokenTicker) {
        updatedFeeTokenBalance = {Ticker:element.symbol,Amount:BigNumber(element.amount),Decimals:element.decimals};
        updatedArbitrageTokenBalance={Ticker:element.symbol,Amount:BigNumber(element.amount),Decimals:element.decimals};
        isHaveArbitrageToken=true;
        isHaveFeeToken=true;      
      }else if(element.symbol===tokenForArbitrage){
        updatedArbitrageTokenBalance={Ticker:element.symbol,Amount:BigNumber(element.amount),Decimals:element.decimals};
        isHaveArbitrageToken=true;
      } else if(element.symbol===feeTokenTicker){
        updatedFeeTokenBalance = {Ticker:element.symbol,Amount:BigNumber(element.amount),Decimals:element.decimals};
        isHaveFeeToken=true;
      }

    });
        await Promise.all(balancePromises);
        arbitrageTokenBalance={Ticker:updatedArbitrageTokenBalance.Ticker,Amount:updatedArbitrageTokenBalance.Amount,Decimals:updatedArbitrageTokenBalance.Decimals};
        feeTokenBalance={Ticker:updatedFeeTokenBalance.Ticker,Amount:updatedFeeTokenBalance.Amount,Decimals:updatedFeeTokenBalance.Decimals};
 
    }catch(error){
       console.log("Error happened to get balances",error)
      };

        

        console.log("Arbitrage Token Balance:\n"+arbitrageTokenBalance.Amount+arbitrageTokenBalance.Ticker+arbitrageTokenBalance.Decimals);
        console.log("Fee token BAlance:\n"+feeTokenBalance.Amount+feeTokenBalance.Ticker+feeTokenBalance.Decimals);

        if(isHaveArbitrageToken && isHaveFeeToken){
          console.log("Address have arbitrage and fee Token");
          return true;
        }else{
          console.log("Address dont have "+tokenForArbitrage+" or fee token "+feeTokenTicker);
          return false;};

}

function checkBalancesAndFeeIsEnoughForArbitrage(){
  let isArbPossible=false;
  let bigArbAmount=BigNumber(arbAmount).shiftedBy(arbitrageTokenBalance.Decimals);
  let bigFeeAmount=BigNumber(gasPrice).multipliedBy(gasLimit);
  

  console.log("big Arb Amount: "+bigArbAmount.shiftedBy(-arbitrageTokenBalance.Decimals).toFormat());
  console.log("big fee amount: "+bigFeeAmount.shiftedBy(-feeTokenBalance.Decimals).toFormat());
  
  
try{

  if(BigNumber(feeTokenBalance.Amount).isGreaterThan(bigFeeAmount)){
    isFeeTokenEnough=true;
    isArbPossible=true;
    console.log("Fee token enough");
  }else{
    isFeeTokenEnough=false;
    isArbPossible=false;
    console.log("You need more fee token " + "min "+bigFeeAmount.shiftedBy(-feeTokenBalance.Decimals).toFixed()+ " "+ "Kcal" );
  }

  if(tokenForArbitrage===feeTokenTicker && arbitrageTokenBalance.Amount.isGreaterThan(bigArbAmount) && isFeeTokenEnough && isHaveArbitrageToken){
      console.log("Arb token is fee token");
      
      let isFeeTokenEnoughForArbitrage=arbitrageTokenBalance.Amount.isGreaterThan(bigArbAmount.plus(bigFeeAmount));

    if(isFeeTokenEnoughForArbitrage){
      isArbPossible=true;
      console.log("Arbitrage token enough");
    }else{
      
      bigArbAmount=arbitrageTokenBalance.Amount.minus(bigFeeAmount);
      console.log("Arbitrage token enough but need to deduct fees from arb amount, new arb Amount_"+bigArbAmount.shiftedBy(-arbitrageTokenBalance.Decimals).toFormat());
      isArbPossible=true;
    }
    
       
  }else if (arbitrageTokenBalance.Amount.isGreaterThanOrEqualTo(bigArbAmount) && isFeeTokenEnough && isHaveArbitrageToken){
    isArbPossible=true;
    console.log("Arbitrage token enough");
  }else if (arbitrageTokenBalance.Amount.isLessThan(bigArbAmount) && isFeeTokenEnough && isHaveArbitrageToken ){
    isArbPossible=true;
    arbAmount=arbitrageTokenBalance.Amount.shiftedBy(-arbitrageTokenBalance.Decimals).toNumber();
    console.log("Arbitrage token not enough need to set it to max balance, new arb Amount_"+arbAmount);
  }else{isArbPossible=false;}

  return isArbPossible;
}catch(error){console.log("error happened in checkBalancesAndFeeIsEnoughForArbitrage\n"+error)}

}

function main(){
  
  
  checkBalancesIsCorrectForArb().then(()=>{
    if(isHaveArbitrageToken && isHaveFeeToken){
      getPoolReserves().then(()=>{
        findArbSwapRoute(tokenForArbitrage).then(()=>{
          optimizeArbAmountv2()
        }).then(()=>{
          checkBalancesAndFeeIsEnoughForArbitrage()
        }).then(()=>{
          if(profitAmount>minProfit && isFeeTokenEnough && isHaveArbitrageToken && isAuto && isOptimizationSuccessfull){
            sendArbTransaction().then(()=>checkBalancesIsCorrectForArb().then(()=>getPoolReserves()))
          }
        })










      })
    }
    
  
  
  
  
  
  
  
  }
); 




}

  function runWithInterval() {
    // Initial call
    main();

    // Set interval to call getPoolReserves() every 5 minutes
    setInterval(async () => {
      main();
    }, 300000); // 5 minutes in milliseconds
  }

  // Start running with interval
  runWithInterval();
</script>


<div class="text-gray-900 dark:text-white flex flex-col items-center justify-center">
    <div class="grid grid-cols-1 gap-2 w-full max-w-md">
      {#each PoolsForArbitrage as pool}
        <Card padding="md" size="lg" class="w-full">
          <div class="flex justify-center items-center mb-4">
            <h5 class="text-xl font-bold leading-none">{pool.PoolName}</h5>
          </div>
  
          <div class="flex items-center space-x-2 rtl:space-x-reverse">
            <Avatar
              src="../src/assets/{pool.Token1Ticker}.png"
              alt={pool.Token1Ticker}
              onerror="this.onerror=null;this.src='../src/assets/UNKNOWN.png';"
              class="flex-shrink-0"
              rounded
            />
            <div class="flex-1 min-w-0">
              <p class="text-sm font-medium text-gray-900 truncate dark:text-white">{pool.Token1Ticker}</p>
              <p class="text-sm text-gray-500 truncate dark:text-gray-400">{pool.Token1Amount}</p>
            </div>
            <Avatar
              src="../src/assets/{pool.Token2Ticker}.png"
              alt={pool.Token2Ticker}
              class="flex-shrink-0"
              rounded
            />
            <div class="flex-1 min-w-0">
              <p class="text-sm font-medium text-gray-900 truncate dark:text-white">{pool.Token2Ticker}</p>
              <p class="text-sm text-gray-500 truncate dark:text-gray-400">{pool.Token2Amount}</p>
            </div>
          </div>
        </Card>
      {/each}
    </div>
  
    <div class="grid gap-2 grid-cols-2 mt-4 w-full max-w-md">
      <Button color="light" size="md" class="mr-1 mt-1" on:click={() => getPoolReserves()}>Update Pair Reserves</Button>
      <Button color="light" size="md" class="mr-1 mt-1" on:click={() => findArbSwapRoute("SOUL")}>Find Route</Button>
    </div>
  
    <div class="grid gap-2 grid-cols-2 mt-4 w-full max-w-md">
      <Button color="light" size="md" class="mr-1 mt-1" on:click={() => optimizeArbAmountv2()}>Optimize Amount</Button>
      <FloatingLabelInput style="outlined" id="floating_filled" name="floating_filled" type="number" label="Optimized Amount" readonly value={optimizedAmount} class="mr-1 mt-1"></FloatingLabelInput>
    </div>
  
    <div class="grid gap-2 grid-cols-2 mt-4 w-full max-w-md">
      <FloatingLabelInput style="outlined" id="floating_filled" name="floating_filled" type="number" label="Arbitrage Amount" class="mr-1 mt-1" bind:value={arbAmount} on:keyup={() => calculateArbitrageAmounts(arbAmount)}></FloatingLabelInput>
      <FloatingLabelInput style="outlined" color="green" id="floating_filled" name="floating_filled" type="number" label="Profit Amount" class="mr-1 mt-1" readonly value={profitAmount}></FloatingLabelInput>
    </div>
  
    <div class="grid gap-2 grid-cols-1 mt-4 w-full max-w-md">
      <Button outline color="red" size="md" class="mr-1 mt-1" on:click={() => sendArbTransaction().then((txHash) => { console.log("Transaction sent successfully. Hash: ", txHash); getPoolReserves(); }).catch((error) => { console.error("Error in sendATransaction: ", error); })}>Send Arb Tx</Button>
    </div>
  </div>
  