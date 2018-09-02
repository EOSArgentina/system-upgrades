# System Upgrade to v1.2.1

## Details

EOS Argentina is proposing upgrading system[0] from version v1.1.0-patch3 to version v1.2.1[1], this proposal will expire on "2018-09-07T05:28:42", so please verify and validate the proposal before that time.

[0]https://eosauthority.com/approval/view?scope=argentinaeos&name=systemupv121

[1] https://github.com/EOSIO/eosio.contracts/tree/v1.2.1


* version v1.2.1 HASH: b721d706e5ea6dbdfd6fc601bbc3e52adca5c83138e3326290451ef888707b6d
* version v1.1.0-patch3  HASH: ec02a22b7f15064f3ff86564ef9e34e2c68ac9061023b11b47e1adfceedf1368

## Requested permissions
```
eoslaomaocom
eosnewyorkio
eos42freedom
starteosiobp
zbeosbp11111
bitfinexeos1
jedaaaaaaaaa
eosswedenorg
eoscanadacom
libertyblock
eosfishrocks
eoshuobipool
cypherglasss
eosasia11111
eosauthority
teamgreymass
eosflytomars
eosriobrazil
argentinaeos
eoscannonchn
eosdacserver
eosyskoreabp
helloeoscnbp
eosbixinboot
eosbeijingbp
atticlabeosb
eosisgravity
eosamsterdam
cryptolions1
eoscafeblock
```


## Tests

**Testnet: CryptoKylin**  
**Account: argentinaeos**

* regproducer ✓
* unreg producer  ✓
* voteproducer ✓
* sellram ✓
* buyram  ✓
* delegatebw ✓
* undelegatebw ✓
* bidname ✓
* regproxy ✓
* updatevote ✓
* newaccount ✓
* transfer ✓
* claimrewards ✓



## Diff  v1.1.0 v1.1.0-patch3


```diff
diff --git a/eosio.system/abi/eosio.system.abi b/eosio.system/abi/eosio.system.abi
index 4e30369..95aee25 100644
--- a/eosio.system/abi/eosio.system.abi
+++ b/eosio.system/abi/eosio.system.abi
@@ -40,6 +40,13 @@
         {"name":"bid", "type":"asset"}
       ]
     },{
+      "name": "bidrefund",
+      "base": "",
+      "fields": [
+        {"name":"bidder",  "type":"account_name"},
+        {"name":"newname", "type":"account_name"}
+      ]
+    },{
       "name": "permission_level_weight",
       "base": "",
       "fields": [
@@ -497,6 +504,10 @@
       "type": "bidname",
       "ricardian_contract": ""
    },{
+      "name": "bidrefund",
+      "type": "bidrefund",
+      "ricardian_contract": ""
+   },{
       "name": "unregprod",
       "type": "unregprod",
       "ricardian_contract": ""
diff --git a/eosio.system/include/eosio.system/eosio.system.hpp b/eosio.system/include/eosio.system/eosio.system.hpp
index 276a61e..6b5c454 100644
--- a/eosio.system/include/eosio.system/eosio.system.hpp
+++ b/eosio.system/include/eosio.system/eosio.system.hpp
@@ -30,10 +30,18 @@ namespace eosiosystem {
      uint64_t by_high_bid()const { return static_cast<uint64_t>(-high_bid); }
    };
 
+   struct bid_refund {
+      account_name bidder;
+      asset        amount;
+
+     auto primary_key() const { return bidder; }
+   };
+
    typedef eosio::multi_index< N(namebids), name_bid,
                                indexed_by<N(highbid), const_mem_fun<name_bid, uint64_t, &name_bid::by_high_bid>  >
                                >  name_bid_table;
 
+   typedef eosio::multi_index< N(bidrefunds), bid_refund> bid_refund_table;
 
    struct eosio_global_state : eosio::blockchain_parameters {
       uint64_t free_ram()const { return max_ram_size - total_ram_bytes_reserved; }
@@ -161,6 +169,7 @@ namespace eosiosystem {
          void onblock( block_timestamp timestamp, account_name producer );
                       // const block_header& header ); /// only parse first 3 fields of block header
 
+         void setalimits( account_name act, int64_t ram, int64_t net, int64_t cpu );
          // functions defined in delegate_bandwidth.cpp
 
          /**
@@ -235,6 +244,9 @@ namespace eosiosystem {
          void rmvproducer( account_name producer );
 
          void bidname( account_name bidder, account_name newname, asset bid );
+
+         void bidrefund( account_name bidder, account_name newname );
+
       private:
          void update_elected_producers( block_timestamp timestamp );
          void update_ram_supply();
diff --git a/eosio.system/src/delegate_bandwidth.cpp b/eosio.system/src/delegate_bandwidth.cpp
index 3b1599b..6b6a315 100644
--- a/eosio.system/src/delegate_bandwidth.cpp
+++ b/eosio.system/src/delegate_bandwidth.cpp
@@ -155,7 +155,6 @@ namespace eosiosystem {
       set_resource_limits( res_itr->owner, res_itr->ram_bytes, res_itr->net_weight.amount, res_itr->cpu_weight.amount );
    }
 
-
   /**
     *  The system contract now buys and sells RAM allocations at prevailing market prices.
     *  This may result in traders buying RAM today in anticipation of potential shortages
@@ -418,7 +417,7 @@ namespace eosiosystem {
       // allow people to get their tokens earlier than the 3 day delay if the unstake happened immediately after many
       // consecutive missed blocks.
 
-      INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {N(eosio.stake),N(active)},
+      INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {{N(eosio.stake),N(active)},{req->owner,N(active)}},
                                                     { N(eosio.stake), req->owner, req->net_amount + req->cpu_amount, std::string("unstake") } );
 
       refunds_tbl.erase( req );
diff --git a/eosio.system/src/eosio.system.cpp b/eosio.system/src/eosio.system.cpp
index 1f0b6a1..dc41d58 100644
--- a/eosio.system/src/eosio.system.cpp
+++ b/eosio.system/src/eosio.system.cpp
@@ -117,6 +117,14 @@ namespace eosiosystem {
       require_auth( _self );
       set_privileged( account, ispriv );
    }
+   
+   void system_contract::setalimits( account_name account, int64_t ram, int64_t net, int64_t cpu ) {
+      require_auth( N(eosio) );
+      user_resources_table userres( _self, account );
+      auto ritr = userres.find( account );
+      eosio_assert( ritr == userres.end(), "only supports unlimited accounts" );
+      set_resource_limits(account, ram, net, cpu);
+   }
 
    void system_contract::rmvproducer( account_name producer ) {
       require_auth( _self );
@@ -156,9 +164,27 @@ namespace eosiosystem {
          eosio_assert( bid.amount - current->high_bid > (current->high_bid / 10), "must increase bid by 10%" );
          eosio_assert( current->high_bidder != bidder, "account is already highest bidder" );
 
-         INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {N(eosio.names),N(active)},
-                                                       { N(eosio.names), current->high_bidder, asset(current->high_bid),
-                                                       std::string("refund bid on name ")+(name{newname}).to_string()  } );
+         bid_refund_table refunds_table(_self, newname);
+
+         auto it = refunds_table.find( current->high_bidder );
+         if ( it != refunds_table.end() ) {
+            refunds_table.modify( it, 0, [&](auto& r) {
+                  r.amount += asset( current->high_bid, system_token_symbol );
+               });
+         } else {
+            refunds_table.emplace( bidder, [&](auto& r) {
+                  r.bidder = current->high_bidder;
+                  r.amount = asset( current->high_bid, system_token_symbol );
+               });
+         }
+
+         action a( {N(eosio),N(active)}, N(eosio), N(bidrefund), std::make_tuple( current->high_bidder, newname ) );
+         transaction t;
+         t.actions.push_back( std::move(a) );
+         t.delay_sec = 0;
+         uint128_t deferred_id = (uint128_t(newname) << 64) | current->high_bidder;
+         cancel_deferred( deferred_id );
+         t.send( deferred_id, bidder );
 
          bids.modify( current, bidder, [&]( auto& b ) {
             b.high_bidder = bidder;
@@ -168,6 +194,16 @@ namespace eosiosystem {
       }
    }
 
+   void system_contract::bidrefund( account_name bidder, account_name newname ) {
+      bid_refund_table refunds_table(_self, newname);
+      auto it = refunds_table.find( bidder );
+      eosio_assert( it != refunds_table.end(), "refund not found" );
+      INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {{N(eosio.names),N(active)},{bidder,N(active)}},
+                                                    { N(eosio.names), bidder, asset(it->amount),
+                                                       std::string("refund bid on name ")+(name{newname}).to_string()  } );
+      refunds_table.erase( it );
+   }
+
    /**
     *  Called after a new account is created. This code enforces resource-limits rules
     *  for new accounts as well as new account naming conventions.
@@ -222,7 +258,7 @@ EOSIO_ABI( eosiosystem::system_contract,
      // native.hpp (newaccount definition is actually in eosio.system.cpp)
      (newaccount)(updateauth)(deleteauth)(linkauth)(unlinkauth)(canceldelay)(onerror)
      // eosio.system.cpp
-     (setram)(setramrate)(setparams)(setpriv)(rmvproducer)(bidname)
+     (setram)(setramrate)(setparams)(setpriv)(setalimits)(rmvproducer)(bidname)(bidrefund)
      // delegate_bandwidth.cpp
      (buyrambytes)(buyram)(sellram)(delegatebw)(undelegatebw)(refund)
      // voting.cpp
diff --git a/eosio.system/src/producer_pay.cpp b/eosio.system/src/producer_pay.cpp
index 39fe64e..27e20d1 100644
--- a/eosio.system/src/producer_pay.cpp
+++ b/eosio.system/src/producer_pay.cpp
@@ -129,11 +129,11 @@ namespace eosiosystem {
       });
 
       if( producer_per_block_pay > 0 ) {
-         INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {N(eosio.bpay),N(active)},
+         INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {{N(eosio.bpay),N(active)},{owner,N(active)}},
                                                        { N(eosio.bpay), owner, asset(producer_per_block_pay), std::string("producer block pay") } );
       }
       if( producer_per_vote_pay > 0 ) {
-         INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {N(eosio.vpay),N(active)},
+         INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {{N(eosio.vpay),N(active)},{owner,N(active)}},
                                                        { N(eosio.vpay), owner, asset(producer_per_vote_pay), std::string("producer vote pay") } );
       }
    }
```


## Diff v1.1.0-patch3  v1.2.1

```diff
diff /tmp/eosio.contracts-v1.1.0-patch3/eosio.system/src/delegate_bandwidth.cpp /tmp/eosio.contracts-v1.2.1/eosio.system/src/delegate_bandwidth.cpp
32a33
>    static constexpr int64_t ram_gift_bytes = 1400;
89c90
<       
---
>
155c156
<       set_resource_limits( res_itr->owner, res_itr->ram_bytes, res_itr->net_weight.amount, res_itr->cpu_weight.amount );
---
>       set_resource_limits( res_itr->owner, res_itr->ram_bytes + ram_gift_bytes, res_itr->net_weight.amount, res_itr->cpu_weight.amount );
181c182
<       
---
>
193c194
<       set_resource_limits( res_itr->owner, res_itr->ram_bytes, res_itr->net_weight.amount, res_itr->cpu_weight.amount );
---
>       set_resource_limits( res_itr->owner, res_itr->ram_bytes + ram_gift_bytes, res_itr->net_weight.amount, res_itr->cpu_weight.amount );
272c273,276
<          set_resource_limits( receiver, tot_itr->ram_bytes, tot_itr->net_weight.amount, tot_itr->cpu_weight.amount );
---
>          int64_t ram_bytes, net, cpu;
>          get_resource_limits( receiver, &ram_bytes, &net, &cpu );
>
>          set_resource_limits( receiver, std::max( tot_itr->ram_bytes + ram_gift_bytes, ram_bytes ), tot_itr->net_weight.amount, tot_itr->cpu_weight.amount );
288,289c292,293
<          
<          
---
>
>
diff /tmp/eosio.contracts-v1.1.0-patch3/eosio.system/src/eosio.system.cpp /tmp/eosio.contracts-v1.2.1/eosio.system/src/eosio.system.cpp
48a49,52
>    block_timestamp system_contract::current_block_time() {
>       const static block_timestamp cbt{ time_point{ microseconds{ static_cast<int64_t>( current_time() ) } } };
>       return cbt;
>    }
76c80,82
<       if( _gstate2.last_block_num <= _gstate2.last_ram_increase ) return;
---
>       auto cbt = current_block_time();
>
>       if( cbt <= _gstate2.last_ram_increase ) return;
79c85
<       auto new_ram = (_gstate2.last_block_num.slot - _gstate2.last_ram_increase.slot)*_gstate2.new_ram_per_block;
---
>       auto new_ram = (cbt.slot - _gstate2.last_ram_increase.slot)*_gstate2.new_ram_per_block;
88c94
<       _gstate2.last_ram_increase = _gstate2.last_block_num;
---
>       _gstate2.last_ram_increase = cbt;
92,93c98,99
<     *  Sets the rate of increase of RAM in bytes per block. It is capped by the uint16_t to
<     *  a maximum rate of 3 TB per year.
---
>     *  Sets the rate of increase of RAM in bytes per block. It is capped by the uint16_t to
>     *  a maximum rate of 3 TB per year.
100a107
>       update_ram_supply();
102,106d108
<       if( _gstate2.last_ram_increase == block_timestamp() ) {
<          _gstate2.last_ram_increase = _gstate2.last_block_num;
<       } else {
<          update_ram_supply();
<       }
diff /tmp/eosio.contracts-v1.1.0-patch3/eosio.system/src/producer_pay.cpp /tmp/eosio.contracts-v1.2.1/eosio.system/src/producer_pay.cpp
23a24,27
>
>       // _gstate2.last_block_num is not used anywhere in the system contract code anymore.
>       // Although this field is deprecated, we will continue updating it for now until the last_block_num field
>       // is eventually completely removed, at which point this line can be removed.
53c57
<             auto highest = idx.begin();
---
>             auto highest = idx.lower_bound( std::numeric_limits<uint64_t>::max()/2 );
```

## ABI Diff v1.1.0-patch3 v1.2.1

```diff
738c738
<       "name": "namebid_info",
---
>       "name": "name_bid",
753a754,764
>     },{
>       "name": "bid_refund",
>       "base": "",
>       "fields": [{
>           "name": "bidder",
>           "type": "account_name"
>         },{
>           "name": "amount",
>           "type": "asset"
>         }
>       ]
963c974,984
<       "type": "namebid_info"
---
>       "type": "name_bid"
>     },{
>       "name": "bidrefunds",
>       "index_type": "i64",
>       "key_names": [
>         "bidder"
>       ],
>       "key_types": [
>         "account_name"
>       ],
>       "type": "bid_refund"
```
