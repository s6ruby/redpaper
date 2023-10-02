_The Ruby Programming Language for Contract / Transaction Scripts on the Blockchain World Computer - Yes, It's Just Ruby_

# sruby - Small, Smart, Secure, Safe, Solid & Sound (S6) Ruby

sruby is a subset of mruby [1] that is a subset of "classic" ruby [2].


## What's missing and why?


Less is more. The golden rule of secure code is keep it simple, stupid.

- NO inheritance
- NO recursion
- NO re-entrance - auto-magic protection on function calls
- NO floating point numbers or arithmetic
- NO overflow & underflow in numbers - auto-magic "safe-math" protection
- NO null (`nil`) - all variables, structs and hash mappings have default (zero) values 
- and much much more



## What's the upside?

You can cross-compile (transpile) contract scripts (*) to:

- Solidity [3] - JavaScript-like contract scripts
- Vyper [4] - Python-like contract scripts
- Yul [5] - EVM (Ethereum Virtual Machine) Assembly-like intermediate contract scripts 
- Liquidity [6] - OCaml-like (or ReasonML-like) contract scripts
- and much much more


(*) in the future.


## Yes, yes, yes - It's just "plain-vanilla" ruby

Remember - the code is and always will be just "plain-vanilla" ruby
that runs with "classic" ruby or mruby "out-of-the-box".



**NEW IN 2023** 

Let's update the syntax / style for compatibility with the big brother / sister - Rubidity!  Bonus - Yes, you can. It's just Ruby. [**Run the sample contracts from the Red Paper with Rubidity and Simulacrum »**](https://github.com/s6ruby/rubidity/tree/master/redpaper) 

New to Rubidity?  See [**Rubidity - Ruby for Layer 1 (L1) Contracts / Protocols with "Off-Chain" Indexer**  »](https://github.com/s6ruby/rubidity)




## Contract Samples

### Hello, World! - Greeter


``` ruby
############################
# Greeter Contract 

storage owner:    Address,
        greeting: String

# @sig (String)
def setup( greeting: )
  @owner    = msg.sender
  @greeting = greeting
end

# @sig () => String
def greet
  @greeting
end

# @sig ()
def kill
  selfdestruct( msg.sender )  if msg.sender == @owner
end
```


### Mint Your Own Money - Minimal Viable Token

``` ruby
#######################
# Token Contract

storage balance_of:  Mapping( Address, UInt )
                     
# @sig (UInt)
def setup( initial_supply: )
  @balance_of[ msg.sender] = initial_supply
end

# @sig (Address, UInt) => Bool
def transfer( to:, value: )
  assert @balance_of[ msg.sender ] >= value, 'insufficient funds'
  assert @balance_of[ to ] + value >= @balance_of[ to ], 'overflow - transfer value too big'

  @balance_of[ msg.sender ] -= value
  @balance_of[ to ]         += value

  true
end
```



### Win x65 000 - Roll the (Satoshi) Dice

``` ruby
################################
# Satoshi Dice Contract

event :BetPlaced, id: UInt, user: Address, cap: UInt, amount: UInt
event :Roll,      id: UInt, rolled: UInt

struct :Bet,
         user:   Address, 
         block:  UInt,  
         cap:    UInt, 
         amount: UInt  

storage   owner:    Address,
          counter:  UInt,
          bets:     Mapping( UInt, Bet ) 

## Fee (Casino House Edge) is 1.9%, that is, 19 / 1000
FEE_NUMERATOR   = 19
FEE_DENOMINATOR = 1000

MAXIMUM_CAP = 2**16   # 65_536 = 2^16 = 2 byte/16 bit
MAXIMUM_BET = 100_000_000
MINIMUM_BET = 100
                      
# @sig ()
def setup
  @owner  = msg.sender
end

# @sig (UInt) 
def bet( cap: )
  assert cap >= 1 && cap <= MAXIMUM_CAP
  assert msg.value >= MINIMUM_BET && msg.value <= MAXIMUM_BET

  @counter += 1
  @bets[@counter] = Bet.new( msg.sender, block.number+3, cap, msg.value )
  
  log BetPlaced, id: @counter, 
                 user: msg.sender, 
                 cap: cap, 
                 amount: msg.value
end

# @sig (UInt)
def roll( id: )
  bet = @bets[id]

  assert msg.sender == bet.user,  'only better can roll'
  assert block.number >= bet.block, 'block before bet block; cannot roll'
  assert block.number <= bet.block + 255,  'block beyond 255 blocks; cannot roll'

  ## "provable" fair - random number depends on
  ##  - blockhash (of block in the future - t+3)
  ##  - nonce (that is, bet counter id)
  hex = sha256( "#{blockhash( bet.block )} #{id}" )
  ## get first 2 bytes (4 chars in hex string) and convert to integer number
  ##   results in a number between 0 and 65_535
  rolled = hex_to_i( hex[0,4] )

  if rolled < bet.cap
     payout = bet.amount * MAXIMUM_CAP / bet.cap
     fee = payout * FEE_NUMERATOR / FEE_DENOMINATOR
     payout -= fee

     msg.sender.transfer( payout )
  end

  log Roll, id: id, rolled: rolled
  @bets.delete( id )
end

# @sig ()
def fund
end

# @sig ()
def kill
  assert msg.sender == @owner
  selfdestruct( @owner )
end
```




### Kick Start Your Project with a Crowd Funder

``` ruby
##############################
# Crowd Funder Contract

event :FundingReceived, address: Address, 
                        amount: UInt, 
                        current_total: UInt
event :WinnerPaid, winner_address: Address


enum :State, :fundraising, :expired_refund, :successful

struct :Contribution,
         amount:      UInt, 
         contributor: Address

storage   creator:          Address,
          fund_recipient:   Address,
          campaign_url:     String,
          minimum_to_raise: UInt,
          raise_by:         Timestamp,
          state:            State,
          total_raised:     UInt,
          complete_at:      Timestamp,
          contributions:    Array( Contribution ) 

# @sig (Timedelta, String, Address, UInt)
def setup(
      time_in_hours_for_fundraising:,
      campaign_url:,
      fund_recipient:,
      minimum_to_raise: )

  @creator          = msg.sender
  @fund_recipient   = fund_recipient   # note: creator may be different than recipient
  @campaign_url     = campaign_url
  @minimum_to_raise = minimum_to_raise # required to tip, else everyone gets refund
  @raise_by         = block.timestamp + (time_in_hours_for_fundraising * 1.hour )
  @state            = State.fundraising
end


# @sig ()
def pay_out
  assert @state.successful?,  'state must be set to successful for pay out'

  @fund_recipient.transfer( this.balance )
  log WinnerPaid, winner_address: @fund_recipient
end

# @sig ()
def check_if_funding_complete_or_expired
  if @total_raised > @minimum_to_raise
    @state = State.successful
    pay_out()
  elsif block.timestamp > @raise_by
    # note: backers can now collect refunds by calling refund(id)
    @state = State.expired_refund
    @complete_at = block.timestamp
  end
end

# @sig ()
def contribute
  assert @state.fundraising?, 'state must be set to fundraising to contribute'

  @contributions.push( Contribution.new( msg.value, msg.sender ))
  @total_raised += msg.value

  log FundingReceived, address: msg.sender, 
                       amount: msg.value,
                       current_total: @total_raised 

  check_if_funding_complete_or_expired()

  @contributions.size - 1   # return (contribution) id
end


# @sig (UInt)
def refund( id: )
  assert @state.expired_refund?, 'state must be set to expired_refund to refund'
  assert @contributions.size > id && id >= 0 && @contributions[id].amount != 0,  'contribution id out-of-range'

  amount_to_refund = @contributions[id].amount
  @contributions[id].amount = 0

  @contributions[id].contributor.transfer( amount_to_refund )

  true
end

# @sig
def kill
  assert msg.sender == @creator,  'only creator can kill contract'
  # wait 24 weeks after final contract state before allowing contract destruction
  assert @state.expired_refund? || @state.successful?, 'state must be set to expired_refund'  
  aasert @complete_at + 24.weeks < block.timestamp,  'complete_at time must be beyond 24 weeks'

  # note: creator gets all money that hasn't be claimed
  selfdestruct( msg.sender )
end
```



### Liquid / Delegative Democracy - Let's Vote (or Delegate Your Vote) on a Proposal

``` ruby
#########################
# Ballot Contract

struct :Voter,
          weight:   UInt,     # weight is accumulated by delegation
          voted:    Bool,     # if true, that person already voted
          vote:     UInt,     # index of the voted proposal
          delegate: Address   # person delegated to

struct :Proposal, 
          vote_count: UInt    # number of accumulated votes

storage  chairperson: Address,
         voters:      Mapping( Address, Voter ), 
         proposals:   Array( Proposal )    

## Create a new ballot with $(num_proposals) different proposals.
##   @sig [UInt]
def constructor( num_proposals: )
  @chairperson = msg.sender
  @proposals.length = num_proposals
  
  @voters[ @chairperson ].weight = 1
end

## Give $(to_voter) the right to vote on this ballot.
## May only be called by $(chairperson).
##  @sig [Address]
def give_right_to_vote( to_voter: ) 
   assert msg.sender == @chairperson, "only chairperson"
   aasert @voters[to_voter].voted? == false, "voter already voted"
     
   @voters[to_voter].weight = 1
end

## Delegate your vote to the voter $(to).
##   @sig [Address]
def delegate( to: )
  sender = @voters[msg.sender]  # assigns reference
  assert sender.voted? == false

  while @voters[to].delegate != address(0) && @voters[to].delegate != msg.sender do
    to = @voters[to].delegate
  end
  assert to != msg.sender

  sender.voted    = true
  sender.delegate = to
  delegate_to = @voters[to]
  if delegate_to.voted
    @proposals[delegate_to.vote].vote_count += sender.weight
  else
    delegate_to.weight += sender.weight
  end
end

## Give a single vote to proposal $(to_proposal).
##  @sig [UInt]
def vote( to_proposal: )
  sender = @voters[msg.sender]
  assert sender.voted? == false && to_proposal < @proposals.length
  sender.voted = true
  sender.vote  = to_proposal
  @proposals[to_proposal].vote_count += sender.weight
end

##  @sig [], :view, returns: UInt
def winning_proposal
  winning_vote_count = 0 
  winning_proposal   = 0
  @proposals.each_with_index do |proposal,i|
    if proposal.vote_count > winning_vote_count
      winning_vote_count = proposal.vote_count
      winning_proposal   = i
    end
  end
  winning_proposal
end
```






## Types  (Work-In-Progress)

[Value Types](#value-types) •
[Reference Types](#reference-types)



### Value Types

Bool •
Integer (Money • Timestamp • Timedelta • Enum) •
Address •
String •
Byte Array / Bytes


#### Bool

Class: `Bool`

Values: true | false

Zero: false

``` ruby
Bool.zero   #=> false
```

#### Integer

Class: `Int`

Zero: 0

``` ruby
Int.zero   #=> false
```

**Integer Types**

Money • Timestamp • Timedelta • Enum

#### Money (Integer)

Money Units

#### Timestamp (Integer)

Time Units

``` ruby
Timestamp.now   #=> 1551122309
```

#### Timedelta (Integer)


#### Enum (Integer)




#### Address

Class: `Address`

Zero: `0x0000` or `0x0` or `Address(0)`


Example:

``` ruby
owner = '0x0000'  # or
owner = address(0)
```


#### String

#### Byte Array / Bytes





### Reference Types

[Array](#array)  •
[Struct](#struct)  •
[Mapping](#mapping)



#### Array


Example:

``` ruby
Array()
```

#### Struct


#### Mapping



## Event Logging

...



## Storage

The contract's state gets stored in contract storage.

...






## Request for Comments (RFC)

Send your questions and comments to the ruby-talk mailing list. Thanks!



## References

1. mruby programming language, see <https://mruby.org>
2. ruby programming language, see <https://www.ruby-lang.org> 
3. solidity programming language, see <https://solidity.readthedocs.io>
4. vyper programming language, see <https://vyper.readthedocs.io>
5. yul intermediate language for the EVM (ethereum virtual machine) assembly code, see the Yul section in the solidity programming language reference
6. liquidity programing language, see <http://www.liquidity-lang.org>

