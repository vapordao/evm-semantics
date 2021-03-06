Web3 RPC JSON Handler
====================

```k
requires "evm.k"

module WEB3
    imports EVM
    imports EVM-DATA
    imports K-IO

    configuration
      <kevm-client>
        <kevm/>
        <web3socket> $SOCK:Int </web3socket>
        <web3clientsocket> 0:IOInt </web3clientsocket>
        <web3request>
          <jsonrpc> "":JSON </jsonrpc>
          <callid> 0:JSON </callid>
          <method> "":JSON </method>
          <params> [ .JSONList ] </params>
          <batch> undef </batch>
        </web3request>
        <web3response> .List </web3response>
      </kevm-client>

    syntax JSON ::= Int | Bool | "null" | "undef"
                  | #getJSON ( JSONKey , JSON ) [function]
 // ------------------------------------------------------
    rule #getJSON( KEY, { KEY : J, _ } )     => J
    rule #getJSON( _, { .JSONList } )        => undef
    rule #getJSON( KEY, { KEY2 : _, REST } ) => #getJSON( KEY, { REST } )
      requires KEY =/=K KEY2

    syntax Int ::= #getInt ( JSONKey , JSON ) [function]
 // ----------------------------------------------------
    rule #getInt( KEY, J ) => {#getJSON( KEY, J )}:>Int

    syntax String ::= #getString ( JSONKey , JSON ) [function]
 // ----------------------------------------------------------
    rule #getString( KEY, J ) => {#getJSON( KEY, J )}:>String

    syntax IOJSON ::= JSON | IOError

    syntax EthereumSimulation ::= accept() [symbol]
 // -----------------------------------------------
    rule <k> accept() => getRequest() ... </k>
         <web3socket> SOCK </web3socket>
         <web3clientsocket> _ => #accept(SOCK) </web3clientsocket>

    syntax KItem ::= getRequest()
 // -----------------------------
    rule <k> getRequest() => #loadRPCCall(#getRequest(SOCK)) ... </k>
         <web3clientsocket> SOCK </web3clientsocket>
         <batch> _ => undef </batch>

    syntax IOJSON ::= #getRequest(Int) [function, hook(JSON.read)]
 // --------------------------------------------------------------

    syntax K ::= #putResponse(JSON, Int) [function, hook(JSON.write)]
 // -----------------------------------------------------------------

    syntax KItem ::= #loadRPCCall(IOJSON)
 // -------------------------------------
    rule <k> #loadRPCCall({ _ } #as J) => #checkRPCCall ~> #runRPCCall ... </k>
         <jsonrpc> _             => #getJSON("jsonrpc", J) </jsonrpc>
         <callid>  _             => #getJSON("id"     , J) </callid>
         <method>  _             => #getJSON("method" , J) </method>
         <params>  _             => #getJSON("params" , J) </params>

    rule <k> #loadRPCCall(#EOF) => #shutdownWrite(SOCK) ~> #close(SOCK) ~> accept() ... </k>
         <web3clientsocket> SOCK </web3clientsocket>

    rule <k> #loadRPCCall([ _, _ ] #as J) => #loadFromBatch ... </k>
         <batch> _ => J </batch>
         <web3response> _ => .List </web3response>

    rule <k> #loadRPCCall(_:String #Or null #Or _:Int #Or [ .JSONList ]) => #sendResponse("error": {"code": -32600, "message": "Invalid Request"}) ... </k>
         <callid> _ => null </callid>

    syntax KItem ::= "#loadFromBatch"
 // ---------------------------------
    rule <k> #loadFromBatch ~> _ => #loadRPCCall(J) </k>
         <batch> [ J , JS => JS ] </batch>

    rule <k> #loadFromBatch ~> _ => #putResponse(List2JSON(RESPONSE), SOCK) ~> getRequest() </k>
         <batch> [ .JSONList ] </batch>
         <web3clientsocket> SOCK </web3clientsocket>
         <web3response> RESPONSE </web3response>
      requires size(RESPONSE) >Int 0

    rule <k> #loadFromBatch ~> _ => getRequest() </k>
         <batch> [ .JSONList ] </batch>
         <web3response> .List </web3response>

    syntax JSON ::= List2JSON(List)           [function]
                  | List2JSON(List, JSONList) [function, klabel(List2JSONAux)]
 // --------------------------------------------------------------------------
    rule List2JSON(L) => List2JSON(L, .JSONList)

    rule List2JSON(L ListItem(J), JS) => List2JSON(L, (J, JS))
    rule List2JSON(.List, JS) => [ JS ]

    syntax KItem ::= #sendResponse( JSON )
 // --------------------------------------
    rule <k> #sendResponse(J) ~> _ => #putResponse({ "jsonrpc": "2.0", "id": CALLID, J }, SOCK) ~> getRequest() </k>
         <callid> CALLID </callid>
         <web3clientsocket> SOCK </web3clientsocket>
         <batch> undef </batch>
      requires CALLID =/=K undef

    rule <k> #sendResponse(_) ~> _ => getRequest() </k>
         <callid> undef </callid>
         <batch> undef </batch>

    rule <k> #sendResponse(J) ~> _ => #loadFromBatch </k>
         <callid> CALLID </callid>
         <batch> [ _ ] </batch>
         <web3response>... .List => ListItem({ "jsonrpc": "2.0", "id": CALLID, J }) </web3response>

    rule <k> #sendResponse(_) ~> _ => #loadFromBatch </k>
         <callid> undef </callid>
         <batch> [ _ ] </batch>

    syntax KItem ::= "#checkRPCCall"
 // --------------------------------
    rule <k> #checkRPCCall => . ...</k>
         <jsonrpc> "2.0" </jsonrpc>
         <method> _:String </method>
         <params> undef #Or [ _ ] #Or { _ } #Or undef </params>
         <callid> _:String #Or null #Or _:Int #Or undef </callid>

    rule <k> #checkRPCCall => #sendResponse( "error": {"code": -32600, "message": "Invalid Request"} ) ... </k>
         <callid> undef #Or [ _ ] #Or { _ } => null </callid> [owise]

    rule <k> #checkRPCCall => #sendResponse( "error": {"code": -32600, "message": "Invalid Request"} ) ... </k>
         <callid> _:Int </callid> [owise]

    rule <k> #checkRPCCall => #sendResponse( "error": {"code": -32600, "message": "Invalid Request"} ) ... </k>
         <callid> _:String </callid> [owise]

    syntax KItem ::= "#runRPCCall"
 // ------------------------------
    rule <k> #runRPCCall => #net_version ... </k>
         <method> "net_version" </method>
    rule <k> #runRPCCall => #web3_clientVersion ... </k>
         <method> "web3_clientVersion" </method>
    rule <k> #runRPCCall => #eth_gasPrice ... </k>
         <method> "eth_gasPrice" </method>
    rule <k> #runRPCCall => #eth_blockNumber ... </k>
         <method> "eth_blockNumber" </method>
    rule <k> #runRPCCall => #eth_accounts ... </k>
         <method> "eth_accounts" </method>
    rule <k> #runRPCCall => #eth_getBalance ... </k>
         <method> "eth_getBalance" </method>
    rule <k> #runRPCCall => #eth_getStorageAt ... </k>
         <method> "eth_getStorageAt" </method>
    rule <k> #runRPCCall => #eth_getCode ... </k>
         <method> "eth_getCode" </method>

    rule <k> #runRPCCall => #sendResponse( "error": {"code": -32601, "message": "Method not found"} ) ... </k> [owise]

    syntax KItem ::= "#net_version"
 // -------------------------------
    rule <k> #net_version => #sendResponse( "result" : Int2String( CID ) ) ... </k>
         <chainID> CID </chainID>

    syntax KItem ::= "#web3_clientVersion"
 // -------------------------------
    rule <k> #web3_clientVersion => #sendResponse( "result" : "Firefly RPC/v0.0.1/kevm" ) ... </k>

    syntax KItem ::= "#eth_gasPrice"
 // --------------------------------
    rule <k> #eth_gasPrice => #sendResponse( "result" : #unparseQuantity( PRICE ) ) ... </k>
         <gasPrice> PRICE </gasPrice>

    syntax KItem ::= "#eth_blockNumber"
 // -----------------------------------
    rule <k> #eth_blockNumber => #sendResponse( "result" : #unparseQuantity( BLOCKNUM ) ) ... </k>
         <number> BLOCKNUM </number>

    syntax KItem ::= "#eth_accounts"
 // --------------------------------
    rule <k> #eth_accounts => #sendResponse( "result" : [ #acctsToJArray( ACCTS ) ] ) ... </k>
         <activeAccounts> ACCTS </activeAccounts>

    syntax JSONList ::= #acctsToJArray ( Set ) [function]
 // -----------------------------------------------------
    rule #acctsToJArray( .Set                      ) => .JSONList
    rule #acctsToJArray( SetItem( ACCT ) ACCTS:Set ) => #unparseData( ACCT, 20 ), #acctsToJArray( ACCTS )

    syntax KItem ::= "#eth_getBalance"
 // ----------------------------------
    rule <k> #eth_getBalance ... </k>
         <params> [ (DATA => #parseHexWord(DATA)), _ ] </params>

    rule <k> #eth_getBalance => #sendResponse( "result" : #unparseQuantity( ACCTBALANCE ) ) ... </k>
         <params> [ DATA, TAG, .JSONList ] </params>
         <account>
           <acctID> DATA </acctID>
           <balance> ACCTBALANCE </balance>
           ...
         </account>

    syntax KItem ::= "#eth_getStorageAt"
 // ------------------------------------
    rule <k> #eth_getStorageAt ... </k>
         <params> [ (DATA => #parseHexWord(DATA)), QUANTITY:Int, _ ] </params>

    rule <k> #eth_getStorageAt => #sendResponse( "result" : #unparseQuantity( #lookup (STORAGE, QUANTITY) ) ) ... </k>
         <params> [ DATA, QUANTITY, TAG, .JSONList ] </params>
         <account>
           <acctID> DATA </acctID>
           <storage> STORAGE </storage>
           ...
         </account>

    syntax KItem ::= "#eth_getCode"
 // -------------------------------
    rule <k> #eth_getCode ... </k>
         <params> [ (DATA => #parseHexWord(DATA)), _ ] </params>

    rule <k> #eth_getCode => #sendResponse( "result" : #unparseDataByteArray( CODE ) ) ... </k>
         <params> [ DATA, TAG, .JSONList ] </params>
         <account>
           <acctID> DATA </acctID>
           <code> CODE </code>
           ...
         </account>
endmodule
```
