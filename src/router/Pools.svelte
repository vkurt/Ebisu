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
  } from "phantasma-ts";

  import {
    Card,
    Avatar,
    Button,
    FloatingLabelInput,
    Toast,
  } from "flowbite-svelte";

  // Set up Phantasma API and contract name
  const NexusName = "mainnet"; // or 'mainnet'
  const ChainName = "main"; // The chain name
  let RpcLink = "https://pharpc1.phantasma.info/rpc"; // mainnet: https://pharpc1.phantasma.info/rpc Testnet: https://testnet.phantasma.info/rpc

  const RPC = new PhantasmaAPI(RpcLink, null, NexusName);

  const wif = import.meta.env.VITE_PRIVATE_KEY; // **************Replace with your WIF key********************** set up your .env.local data
  //console.log(wif);
  const keys = PhantasmaKeys.fromWIF(wif);
  const expiration = 36000000; // Transaction expiration (1 hour from now)
  const gasPrice = 110000;
  const gasLimit = 75000;
  const payload = Base16.encode("Ebisu");
  const contractAddress = "SATRN"; // contract name
  const feeTokenTicker = "KCAL";

  let profitAmount: BigNumber = BigNumber(0);
  let arbAmount = BigNumber(0);
  let optimizedAmount = BigNumber(0);
  let tokenForArbitrage = "SOUL";
  const minProfit = 0.75 / 100;
  const minArbAmount: BigNumber = BigNumber(1);
  let isFeeTokenEnough = false;
  let isHaveFeeToken = false;
  let isHaveArbitrageToken = false;
  let isOptimizationSuccessfull = false;
  let isAuto = true;
  let checkInterval = 60000; //1 min in milisec

  let feeTokenBalance: {
    Ticker: string;
    Amount: BigNumber;
    Decimals: number;
  } = { Ticker: "", Amount: BigNumber(0), Decimals: 0 };
  let arbitrageTokenBalance: {
    Ticker: string;
    Amount: BigNumber;
    Decimals: number;
  } = { Ticker: "", Amount: BigNumber(0), Decimals: 0 };

  let PoolsForArbitrage: {
    //pushing amount here with decimals not raw
    PoolName: string;
    Token1Ticker: string;
    Token1Amount: BigNumber;
    Token1Decimals: number;
    Token2Ticker: string;
    Token2Amount: BigNumber;
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
    //pushing amount here with decimals not raw
    PoolName: string;
    InTicker: string;
    IntickerDecimals: number;
    InAmount: BigNumber;
    OutTicker: string;
    OutTickerDecimals: number;
    IsDivisorPool: boolean;
    IsRelativePricePool: boolean;
    InTickerReserve: BigNumber;
    OutTickerReserve: BigNumber;
  } = {
    PoolName: "",
    InTicker: "",
    InAmount: BigNumber(0),
    OutTicker: "",
    IntickerDecimals: 0,
    OutTickerDecimals: 0,
    IsDivisorPool: false,
    IsRelativePricePool: false,
    InTickerReserve: BigNumber(0),
    OutTickerReserve: BigNumber(0),
  };
  let OutPoolForArb: {
    //pushing amount here with decimals not raw
    PoolName: string;
    InTicker: string;
    IntickerDecimals: number;
    InAmount: BigNumber;
    OutTicker: string;
    OutTickerDecimals: number;
    IsDivisorPool: boolean;
    IsRelativePricePool: boolean;
    InTickerReserve: BigNumber;
    OutTickerReserve: BigNumber;
  } = {
    PoolName: "",
    InTicker: "",
    InAmount: BigNumber(0),
    OutTicker: "",
    IntickerDecimals: 0,
    OutTickerDecimals: 0,
    IsDivisorPool: false,
    IsRelativePricePool: false,
    InTickerReserve: BigNumber(0),
    OutTickerReserve: BigNumber(0),
  };
  let TransitPoolForArb: {
    PoolName: string;
    InTicker: string;
    IntickerDecimals: number;
    InAmount: BigNumber;
    OutTicker: string;
    OutTickerDecimals: number;
    IsDivisorPool: boolean;
    IsRelativePricePool: boolean;
    InTickerReserve: BigNumber;
    OutTickerReserve: BigNumber;
  } = {
    PoolName: "",
    InTicker: "",
    InAmount: BigNumber(0),
    OutTicker: "",
    IntickerDecimals: 0,
    OutTickerDecimals: 0,
    IsDivisorPool: false,
    IsRelativePricePool: false,
    InTickerReserve: BigNumber(0),
    OutTickerReserve: BigNumber(0),
  };

  async function getPoolReserves() {
    //pushing amount here with decimals not raw
    let updatedPoolsForArbitrage: {
      PoolName: string;
      Token1Ticker: string;
      Token1Amount: BigNumber;
      Token1Decimals: number;
      Token2Ticker: string;
      Token2Amount: BigNumber;
      Token2Decimals: number;
      IsRelativePricePool: boolean;
      IsDivisorPool: boolean;
    }[] = [];
    console.log("Token Pairs\n" + TokenPairs + TokenPairs.length);

    // Map each pair to a Promise
    const promises = TokenPairs.map(async (pairElement): Promise<void> => {
      let sb = new ScriptBuilder();
      sb.CallContract(
        contractAddress,
        "getTokenPairAndReserveKeysOnListVALUE",
        [
          pairElement.Token1Ticker +
            "_" +
            pairElement.Token2Ticker +
            "_" +
            pairElement.Token1Ticker,
        ]
      );
      sb.CallContract(
        contractAddress,
        "getTokenPairAndReserveKeysOnListVALUE",
        [
          pairElement.Token1Ticker +
            "_" +
            pairElement.Token2Ticker +
            "_" +
            pairElement.Token2Ticker,
        ]
      );
      const script = sb.EndScript();

      let amounts = [];

      try {
        const response = await RPC.invokeRawScript("main", script);

        response.results.forEach((element) => {
          const decoder = new Decoder(element);
          amounts.push(decoder.readVmObject());
        });
      } catch (error) {
        console.log("An error happened during getting pool reserves" + error);
      }

      updatedPoolsForArbitrage.push({
        PoolName: pairElement.Token1Ticker + "_" + pairElement.Token2Ticker,
        Token1Ticker: pairElement.Token1Ticker,
        Token1Amount: BigNumber(amounts[0]).shiftedBy(
          -pairElement.Token1Decimals
        ),

        Token1Decimals: pairElement.Token1Decimals,
        Token2Ticker: pairElement.Token2Ticker,
        Token2Amount: BigNumber(amounts[1]).shiftedBy(
          -pairElement.Token2Decimals
        ),
        Token2Decimals: pairElement.Token2Decimals,
        IsRelativePricePool: false,
        IsDivisorPool: false,
      });
    });

    // Wait for all promises to resolve
    await Promise.all(promises);

    // updatedPoolsForArbitrage=[];       ////fake data for testing
    // updatedPoolsForArbitrage.push(
    //    {PoolName: "SOUL_KCAL",
    //     Token1Ticker: "SOUL",
    //     Token1Amount: BigNumber("50000.20230435"),
    //     Token1Decimals: 8,
    //     Token2Ticker: "KCAL",
    //     Token2Amount: BigNumber("10700000.413469642"),
    //     Token2Decimals: 10,
    //     IsRelativePricePool: false,
    //     IsDivisorPool: false,
    //   },{

    //     PoolName: "RAA_SOUL",
    //     Token1Ticker: "RAA",
    //     Token1Amount: BigNumber("400.1126716334343"),
    //     Token1Decimals: 18,
    //     Token2Ticker: "SOUL",
    //     Token2Amount: BigNumber("100.88615802196"),
    //     Token2Decimals: 8,
    //     IsRelativePricePool: false,
    //     IsDivisorPool: false},

    //     {PoolName: "RAA_KCAL",
    //     Token1Ticker: "RAA",
    //     Token1Amount: BigNumber("24.5154566825775"),
    //     Token1Decimals: 18,
    //     Token2Ticker: "KCAL",
    //     Token2Amount: BigNumber("441628.920535605"),
    //     Token2Decimals: 10,
    //     IsRelativePricePool: false,
    //     IsDivisorPool: false,}

    // );

    PoolsForArbitrage = updatedPoolsForArbitrage;

    // Now all PoolsForArbitrage should be populated
  }

  function calculateSwapOutV2(
    inAmount: BigNumber,
    inReserves: BigNumber,
    outReserves: BigNumber
  ) {
    let outAmount: BigNumber = BigNumber(0);
    try {
      if (
        inAmount.isGreaterThan(0) &&
        inReserves.isGreaterThan(0) &&
        outReserves.isGreaterThan(0)
      ) {
        outAmount = outReserves.minus(
          inReserves
            .multipliedBy(outReserves)
            .dividedBy(
              inAmount.multipliedBy(997).dividedBy(1000).plus(inReserves)
            )
        );

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

    PoolsForArbitrage.forEach((element) => {
      // checking pool tickers correctly arranged for calculation, wanted to be sure first token is arrangetokenticker

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

    PoolsForArbitrage.forEach((element) => {
      /*
      ***************************************************************************************************************
      arranging divident pool tickers for correctly calculate converted pool 
      ie converted pool for selected relative pool RAA/SOUL calculation should be like this 
      converted Pool price= (RAA/KCAL)/(SOUL/KCAL), if divisor pool is KCAL/SOUL we will get wrong calculation
      *********************************************************************************************************************
      */

      if (
        !element.IsDivisorPool &&
        !element.IsRelativePricePool &&
        element.Token2Ticker == relativePricePoolPrimaryTicker
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
      }
    });
    console.log(
      "\n%c********************_Arrranged pools_**********************\n,",
      "background:  #55fe01  ; color:   #fe0101 "
    );
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
          element.IsRelativePricePool +
          "\n" +
          "Is divider pool_" +
          element.IsDivisorPool
      );
    });
  }

  function calculatePoolRatio(
    token1Amount: BigNumber,
    token2Amount: BigNumber
  ) {
    let poolRatios = token1Amount.dividedBy(token2Amount);
    return poolRatios;
  }

  function calculateArbitrageAmounts(inAmount: BigNumber) {
    InPoolForArb.InAmount = inAmount;

    // TransitPoolForArb.InAmount = calculateSwapOut(
    //     InPoolForArb.PoolName,
    //     InPoolForArb.InAmount,
    //     InPoolForArb.InTicker,
    // );

    TransitPoolForArb.InAmount = calculateSwapOutV2(
      InPoolForArb.InAmount,
      BigNumber(InPoolForArb.InTickerReserve),
      BigNumber(InPoolForArb.OutTickerReserve)
    );

    // OutPoolForArb.InAmount = calculateSwapOut(
    //     TransitPoolForArb.PoolName,
    //     TransitPoolForArb.InAmount,
    //     TransitPoolForArb.InTicker,
    // );

    OutPoolForArb.InAmount = calculateSwapOutV2(
      TransitPoolForArb.InAmount,
      BigNumber(TransitPoolForArb.InTickerReserve),
      BigNumber(TransitPoolForArb.OutTickerReserve)
    );

    // let outAmount = calculateSwapOut(
    //     OutPoolForArb.PoolName,
    //     OutPoolForArb.InAmount,
    //     OutPoolForArb.InTicker,
    // );

    let outAmount = calculateSwapOutV2(
      OutPoolForArb.InAmount,
      BigNumber(OutPoolForArb.InTickerReserve),
      BigNumber(OutPoolForArb.OutTickerReserve)
    );

    profitAmount = outAmount.minus(inAmount);

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

  async function findArbSwapRoute(tokenToincrease: string) {
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
      Token1Reserve: BigNumber;
      Token2Reserve: BigNumber;
    } = {
      PoolName: "",
      Token1: "",
      Token2: "",
      Token1Decimals: 0,
      Token2Decimals: 0,
      IsDivisorPool: false,
      IsRelativePricePool: false,
      Token1Reserve: BigNumber(0),
      Token2Reserve: BigNumber(0),
    };
    let divisorPoolInfo: {
      PoolName: string;
      Token1: string;
      Token1Decimals: number;
      Token2: string;
      Token2Decimals: number;
      IsRelativePricePool: boolean;
      IsDivisorPool: boolean;
      Token1Reserve: BigNumber;
      Token2Reserve: BigNumber;
    } = {
      PoolName: "",
      Token1: "",
      Token2: "",
      Token1Decimals: 0,
      Token2Decimals: 0,
      IsDivisorPool: false,
      IsRelativePricePool: false,
      Token1Reserve: BigNumber(0),
      Token2Reserve: BigNumber(0),
    };
    let transitPoolInfo: {
      //this is not in or out pool middle pool

      PoolName: string;
      Token1: string;
      Token1Decimals: number;
      Token2: string;
      Token2Decimals: number;
      IsRelativePricePool: boolean;
      IsDivisorPool: boolean;
      Token1Reserve: BigNumber;
      Token2Reserve: BigNumber;
    } = {
      PoolName: "",
      Token1: "",
      Token2: "",
      Token1Decimals: 0,
      Token2Decimals: 0,
      IsDivisorPool: false,
      IsRelativePricePool: false,
      Token1Reserve: BigNumber(0),
      Token2Reserve: BigNumber(0),
    };

    PoolsForArbitrage.forEach((element) => {
      if (element.IsRelativePricePool) {
        relativePricePoolPrice = calculatePoolRatio(
          BigNumber(element.Token1Amount),
          BigNumber(element.Token2Amount)
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
      } else if (
        element.Token1Ticker != tokenToincrease &&
        element.Token2Ticker != tokenToincrease
      ) {
        divisorPoolPrice = calculatePoolRatio(
          BigNumber(element.Token1Amount),
          BigNumber(element.Token2Amount)
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
        dividentPoolPrice = calculatePoolRatio(
          BigNumber(element.Token1Amount),
          BigNumber(element.Token2Amount)
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

    if (relativePricePoolPrice > convertedPoolPrice) {
      //finding swap route

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
      let exchangeReserve1 = BigNumber(0);
      let exchangeReserve2 = BigNumber(0);

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
    console.log(
      "\n%c**************************_In/Out/Transit Pools_********************************\n" +
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
        OutPoolForArb.OutTicker,
      "background:  #55fe01  ; color:   #fe0101 "
    );
  }

  function delay(time) {
    return new Promise((resolve) => setTimeout(resolve, time));
  }

  async function sendArbTransaction() {
    let txExpiration = new Date(new Date().getTime() + expiration);

    if (profitAmount.isGreaterThan(0) && arbAmount.isGreaterThan(0)) {
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
        txExpiration,
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
    console.log(
      "%c*****************OptimizationStarted***************\n" +
        InPoolForArb +
        "\n" +
        OutPoolForArb +
        "\n" +
        TransitPoolForArb +
        "\n",
      "background:  #55fe01  ; color:   #fe0101 "
    );

    let amountForZeroProfit = BigNumber(0);
    let poolReserveDifference = BigNumber(0);
    let stepPercent = 0.2; //20% error margin or correction rate
    let precision = 0.001;
    let relativePoolPrice = BigNumber(0);
    let conversionRateForTransitPoolInToken = BigNumber(0);
    let convertedTransitPoolTokenReserveForInPool = BigNumber(0);
    let conversionRateForTransitPoolOutToken = BigNumber(0);
    let convertedTransitPoolTokenReserveForOutPool = BigNumber(0);

    if (InPoolForArb.IsRelativePricePool) {
      //Converting transit pool reserves to in/out pool reserves for token in/out which is same

      relativePoolPrice = BigNumber(InPoolForArb.InTickerReserve).dividedBy(
        InPoolForArb.OutTickerReserve
      );

      console.log("relativepool= in pool");

      console.log("Relative Pool Price_" + relativePoolPrice.toFormat());

      conversionRateForTransitPoolInToken = BigNumber(
        TransitPoolForArb.InTickerReserve
      ).dividedBy(InPoolForArb.OutTickerReserve);

      convertedTransitPoolTokenReserveForInPool =
        conversionRateForTransitPoolInToken.multipliedBy(
          InPoolForArb.InTickerReserve
        );

      console.log(
        "Converted token reserve for in pool_" +
          convertedTransitPoolTokenReserveForInPool.toFormat()
      );

      conversionRateForTransitPoolOutToken = BigNumber(
        TransitPoolForArb.OutTickerReserve
      ).dividedBy(OutPoolForArb.InTickerReserve);

      convertedTransitPoolTokenReserveForOutPool =
        conversionRateForTransitPoolOutToken.multipliedBy(
          OutPoolForArb.OutTickerReserve
        );

      console.log(
        "Converted token reserve for out pool_" +
          convertedTransitPoolTokenReserveForOutPool.toFormat()
      );
    } else {
      relativePoolPrice = calculatePoolRatio(
        OutPoolForArb.OutTickerReserve,
        OutPoolForArb.InTickerReserve
      );

      console.log("relativepool  = out pool");

      console.log("Relative Pool Price_" + relativePoolPrice.toFormat());

      conversionRateForTransitPoolInToken = BigNumber(
        TransitPoolForArb.InTickerReserve
      ).dividedBy(InPoolForArb.OutTickerReserve);

      convertedTransitPoolTokenReserveForInPool =
        conversionRateForTransitPoolInToken.multipliedBy(
          InPoolForArb.InTickerReserve
        );

      console.log(
        "Converted token reserve for in pool_" +
          convertedTransitPoolTokenReserveForInPool.toFormat()
      );

      convertedTransitPoolTokenReserveForOutPool =
        relativePoolPrice.multipliedBy(TransitPoolForArb.OutTickerReserve);

      console.log(
        "Converted token reserve for out pool_" +
          convertedTransitPoolTokenReserveForOutPool.toFormat()
      );
    }

    poolReserveDifference = convertedTransitPoolTokenReserveForInPool.minus(
      convertedTransitPoolTokenReserveForOutPool
    );

    amountForZeroProfit = poolReserveDifference.abs();
    // if(poolReserveDifference.isGreaterThan(0)){
    //   amountForZeroProfit =
    // poolReserveDifference.dividedBy(TransitPoolForArb.InTickerReserve.dividedBy(InPoolForArb.OutTickerReserve).dividedBy(TransitPoolForArb.OutTickerReserve.dividedBy(OutPoolForArb.InTickerReserve))).abs();
    // }else{
    //   amountForZeroProfit =
    // poolReserveDifference.multipliedBy(TransitPoolForArb.InTickerReserve.dividedBy(InPoolForArb.OutTickerReserve).dividedBy(TransitPoolForArb.OutTickerReserve.dividedBy(OutPoolForArb.InTickerReserve))).abs();

    // }

    let shiftAmount = amountForZeroProfit.multipliedBy(stepPercent);
    let tries = 0;

    console.log(
      "initial Values \n amount for zero profit_" +
        amountForZeroProfit +
        "\n Initial Shift Amount_" +
        shiftAmount +
        "Step Percent_" +
        stepPercent
    );
    const maxGuess = amountForZeroProfit.multipliedBy(2);
    const minGuess = minArbAmount.multipliedBy(2);
    calculateArbitrageAmounts(amountForZeroProfit);

    while (tries < 50) {
      tries++;

      console.log(
        "%cStepPercent_" +
          stepPercent +
          "\nShift Amount_" +
          shiftAmount +
          "\nMin guess " +
          BigNumber(minGuess).toNumber() +
          "\nMax Guess " +
          maxGuess +
          "\nAmount For Zero Profit_" +
          BigNumber(amountForZeroProfit).toNumber() +
          "\nPrecision " +
          precision,
        " color:   #55fe01 "
      );

      if (
        profitAmount.isLessThan(0) &&
        amountForZeroProfit.isGreaterThan(shiftAmount)
      ) {
        amountForZeroProfit = amountForZeroProfit.minus(shiftAmount);
        // if (shiftAmount.isGreaterThanOrEqualTo(amountForZeroProfit)) {
        //   amountForZeroProfit = shiftAmount.dividedBy(2);
        // }
        calculateArbitrageAmounts(amountForZeroProfit);

        if (profitAmount.isGreaterThan(0)) {
          stepPercent = stepPercent / 2;
          shiftAmount = amountForZeroProfit.multipliedBy(stepPercent);
        }
      } else if (
        profitAmount.isGreaterThan(0) &&
        amountForZeroProfit.isLessThan(maxGuess)
      ) {
        amountForZeroProfit = amountForZeroProfit.plus(shiftAmount);
        calculateArbitrageAmounts(amountForZeroProfit);

        if (profitAmount.isLessThan(0)) {
          stepPercent = stepPercent / 2;
          shiftAmount = amountForZeroProfit.multipliedBy(stepPercent);
        }
      }

      if (BigNumber(amountForZeroProfit).isLessThanOrEqualTo(shiftAmount.multipliedBy(2))) {
        shiftAmount = amountForZeroProfit.multipliedBy(stepPercent);
      }

      console.log("Shift amount after check" + shiftAmount.toFormat());

      if (BigNumber(amountForZeroProfit).isLessThanOrEqualTo(minGuess)) {
        console.log("amountForZeroProfit less than min guess");
        isOptimizationSuccessfull = false;
        break;
      }

      if (stepPercent < precision) {
        console.log("max precision reached");
        break;
      }
    }
    optimizedAmount = amountForZeroProfit.dividedBy(2);
    arbAmount = optimizedAmount;
    calculateArbitrageAmounts(arbAmount);
    if (profitAmount.isGreaterThan(minProfit)) {
      isOptimizationSuccessfull = true;
    } else {
      isOptimizationSuccessfull = false;
      arbAmount = BigNumber(0);
      optimizedAmount = BigNumber(0);
      profitAmount = BigNumber(0);
    }

    console.log(
      "%c*****************************************************************\n" +
        "*  Optimization ended after trying " +
        tries +
        " times, is success " +
        isOptimizationSuccessfull +
        "  *" +
        "*\n" +
        "*****************************************************************",
      "background:  #55fe01  ; color:   #fe0101 "
    );
    if (isOptimizationSuccessfull) {
      playNotification();
    }
  }

  function playNotification() {
    let notificationSound = document.getElementById("successNotificationSound");
    if (notificationSound instanceof HTMLAudioElement) {

      if(notificationSound.paused || notificationSound.ended){
        notificationSound.play();
      }
      
    } else {
      console.error("Element is not an HTMLAudioElement");
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
    isHaveFeeToken = false;
    isHaveArbitrageToken = false;
    try {
      const balanceResponseForArb = await RPC.getAccount(
        keys.Address.toString(),
        true
      );

      console.log("Balances for arb\n" + balanceResponseForArb);
      const balancePromises = balanceResponseForArb.balances.map(
        async (element): Promise<void> => {
          console.log("balance" + element.amount + element.symbol);

          if (
            element.symbol === tokenForArbitrage &&
            element.symbol === feeTokenTicker
          ) {
            updatedFeeTokenBalance = {
              Ticker: element.symbol,
              Amount: BigNumber(element.amount).shiftedBy(-element.decimals),
              Decimals: element.decimals,
            };
            updatedArbitrageTokenBalance = {
              Ticker: element.symbol,
              Amount: BigNumber(element.amount).shiftedBy(-element.decimals),
              Decimals: element.decimals,
            };
            isHaveArbitrageToken = true;
            isHaveFeeToken = true;
          } else if (element.symbol === tokenForArbitrage) {
            updatedArbitrageTokenBalance = {
              Ticker: element.symbol,
              Amount: BigNumber(element.amount).shiftedBy(-element.decimals),
              Decimals: element.decimals,
            };
            isHaveArbitrageToken = true;
          } else if (element.symbol === feeTokenTicker) {
            updatedFeeTokenBalance = {
              Ticker: element.symbol,
              Amount: BigNumber(element.amount).shiftedBy(-element.decimals),
              Decimals: element.decimals,
            };
            isHaveFeeToken = true;
          }
        }
      );
      await Promise.all(balancePromises);
      arbitrageTokenBalance = {
        Ticker: updatedArbitrageTokenBalance.Ticker,
        Amount: updatedArbitrageTokenBalance.Amount,
        Decimals: updatedArbitrageTokenBalance.Decimals,
      };
      feeTokenBalance = {
        Ticker: updatedFeeTokenBalance.Ticker,
        Amount: updatedFeeTokenBalance.Amount,
        Decimals: updatedFeeTokenBalance.Decimals,
      };
    } catch (error) {
      console.log("Error happened to get balances", error);
    }

    console.log(
      "Arbitrage Token Balance:\n" +
        arbitrageTokenBalance.Amount +
        arbitrageTokenBalance.Ticker +
        arbitrageTokenBalance.Decimals
    );
    console.log(
      "Fee token Balance:\n" +
        feeTokenBalance.Amount +
        feeTokenBalance.Ticker +
        feeTokenBalance.Decimals
    );

    if (isHaveArbitrageToken && isHaveFeeToken) {
      console.log("Address have arbitrage and fee Token");
      return true;
    } else {
      console.log(
        "Address dont have " +
          tokenForArbitrage +
          " or fee token " +
          feeTokenTicker
      );
      return false;
    }
  }

  function checkBalancesAndFeeIsEnoughForArbitrage() {
    let isArbPossible = false;
    let feeAmount = BigNumber(gasPrice)
      .multipliedBy(gasLimit)
      .shiftedBy(-feeTokenBalance.Decimals);

    console.log(
      "required Arb token Amount:\n" +
        arbAmount.toFormat() +
        " " +
        arbitrageTokenBalance.Ticker +
        "\nrequired Fee token amount:\n" +
        feeAmount.toFormat() +
        " " +
        feeTokenBalance.Ticker
    );

    try {
      if (BigNumber(feeTokenBalance.Amount).isGreaterThan(feeAmount)) {
        isFeeTokenEnough = true;
        isArbPossible = true;
        console.log(
          "Fee token enough\n" +
            "Address have\t" +
            feeTokenBalance.Amount.toFormat() +
            " " +
            feeTokenTicker
        );
      } else {
        isFeeTokenEnough = false;
        isArbPossible = false;
        console.log(
          "You need more fee token " +
            "min " +
            feeAmount.toFixed() +
            " " +
            "Kcal"
        );
      }

      if (
        tokenForArbitrage === feeTokenTicker &&
        arbitrageTokenBalance.Amount.isGreaterThan(arbAmount) &&
        isFeeTokenEnough &&
        isHaveArbitrageToken
      ) {
        console.log("Arb token is fee token");

        let isFeeTokenEnoughForArbitrage =
          arbitrageTokenBalance.Amount.isGreaterThan(arbAmount.plus(feeAmount));

        if (isFeeTokenEnoughForArbitrage) {
          isArbPossible = true;
          console.log("Arbitrage token enough");
        } else {
          arbAmount = arbitrageTokenBalance.Amount.minus(feeAmount);
          calculateArbitrageAmounts(arbAmount);
          console.log(
            "Arbitrage token enough but need to deduct fees from arb amount, new arb Amount_" +
              arbAmount.toFormat()
          );
          isArbPossible = true;
        }
      } else if (
        arbitrageTokenBalance.Amount.isGreaterThanOrEqualTo(arbAmount) &&
        isFeeTokenEnough &&
        isHaveArbitrageToken
      ) {
        isArbPossible = true;
        console.log(
          "Arbitrage token enough\n" +
            "Address have\t" +
            arbitrageTokenBalance.Amount.toFormat() +
            " " +
            arbitrageTokenBalance.Ticker
        );
      } else if (
        arbitrageTokenBalance.Amount.isLessThan(arbAmount) &&
        isFeeTokenEnough &&
        isHaveArbitrageToken
      ) {
        isArbPossible = true;
        arbAmount = arbitrageTokenBalance.Amount;
        calculateArbitrageAmounts(arbAmount);
        console.log(
          "Arbitrage token not enough need to set it to max balance, new arb Amount_" +
            arbAmount.toFormat()
        );
      } else {
        isArbPossible = false;
      }

      return isArbPossible;
    } catch (error) {
      console.log(
        "error happened in checkBalancesAndFeeIsEnoughForArbitrage\n" + error
      );
    }
  }

  function main() {
    checkBalancesIsCorrectForArb().then(() => {
      if (isHaveArbitrageToken && isHaveFeeToken) {
        getPoolReserves().then(() => {
          findArbSwapRoute(tokenForArbitrage)
            .then(() => {
              optimizeArbAmountv2();
            })
            .then(() => {
              checkBalancesAndFeeIsEnoughForArbitrage();
            })
            .then(() => {
              if (
                profitAmount.isGreaterThan(minProfit) &&
                isFeeTokenEnough &&
                isHaveArbitrageToken &&
                isAuto &&
                isOptimizationSuccessfull
              ) {
                sendArbTransaction().then(() => main());
              }
            });
        });
      }
    });
  }

  let intervalId;
  function runWithInterval() {
    main();

    intervalId = setInterval(async () => {
      main();
    }, checkInterval);
  }

  runWithInterval();

  console.log("Interval ID for Main_" + intervalId);
</script>

<div
  class="text-gray-900 dark:text-white flex flex-col items-center justify-center"
>
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
            <p
              class="text-sm font-medium text-gray-900 truncate dark:text-white"
            >
              {pool.Token1Ticker}
            </p>
            <p class="text-sm text-gray-500 truncate dark:text-gray-400">
              {pool.Token1Amount}
            </p>
          </div>
          <Avatar
            src="../src/assets/{pool.Token2Ticker}.png"
            alt={pool.Token2Ticker}
            class="flex-shrink-0"
            rounded
          />
          <div class="flex-1 min-w-0">
            <p
              class="text-sm font-medium text-gray-900 truncate dark:text-white"
            >
              {pool.Token2Ticker}
            </p>
            <p class="text-sm text-gray-500 truncate dark:text-gray-400">
              {pool.Token2Amount}
            </p>
          </div>
        </div>
      </Card>
    {/each}
  </div>

  <div class="grid gap-2 grid-cols-2 mt-4 w-full max-w-md">
    <Button
      color="light"
      size="md"
      class="mr-1 mt-1"
      on:click={() => getPoolReserves()}>Update Pair Reserves</Button
    >
    <Button
      color="light"
      size="md"
      class="mr-1 mt-1"
      on:click={() => findArbSwapRoute("SOUL")}>Find Route</Button
    >
  </div>

  <div class="grid gap-2 grid-cols-2 mt-4 w-full max-w-md">
    <Button
      color="light"
      size="md"
      class="mr-1 mt-1"
      on:click={() => optimizeArbAmountv2()}>Optimize Amount</Button
    >
    <FloatingLabelInput
      style="outlined"
      id="floating_filled"
      name="floating_filled"
      type="number"
      label="Optimized Amount"
      readonly
      value={optimizedAmount}
      class="mr-1 mt-1"
    ></FloatingLabelInput>
  </div>

  <div class="grid gap-2 grid-cols-2 mt-4 w-full max-w-md">
    <FloatingLabelInput
      style="outlined"
      id="floating_filled"
      name="floating_filled"
      type="number"
      label="Arbitrage Amount"
      class="mr-1 mt-1"
      bind:value={arbAmount.toNumber}
      on:change={() => calculateArbitrageAmounts(arbAmount)}
    ></FloatingLabelInput>
    <FloatingLabelInput
      style="outlined"
      color="green"
      id="floating_filled"
      name="floating_filled"
      type="number"
      label="Profit Amount"
      class="mr-1 mt-1"
      readonly
      value={profitAmount}
    ></FloatingLabelInput>
  </div>

  <div class="grid gap-2 grid-cols-1 mt-4 w-full max-w-md">
    <Button
      outline
      color="red"
      size="md"
      class="mr-1 mt-1"
      on:click={() =>
        sendArbTransaction()
          .then((txHash) => {
            console.log("Transaction sent successfully. Hash: ", txHash);
            getPoolReserves();
          })
          .catch((error) => {
            console.error("Error in sendATransaction: ", error);
          })}>Send Arb Tx</Button
    >
  </div>
</div>

<audio src="../src/assets/sounds/success.mp3" id="successNotificationSound"
></audio>

