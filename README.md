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
diff /tmp/eosio.contracts-v1.1.0-patch3/eosio.system/src/ /tmp/eosio.contracts-v1.2.1/eosio.system/src/delegate_bandwidth.cpp
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
