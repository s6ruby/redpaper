_Ruby for Contract / Transaction Scripts on the Blockchain (World Computer)_

# sruby - Simple, Smart, Secure, Safe, Solid & Sound Ruby

sruby is a subset of mruby that is a subset of "classic" ruby.


## What's missing and why?


Less is more. The golden rule of secure code is keep it simple, stupid.

- NO inheritance
- NO recursion
- NO re-entrance - auto-magic protection on function calls
- NO floating point numbers or arithmetic
- NO overflow & underflow in numbers - auto-magic "safe-math" protection
- NO null (`nil`) - all variables, structs and mappings have default (zero) values 
- and much much more



## What's the upside?

You can cross-compile (transpile) contract scripts (*) to:

- Solidity - JavaScript-like contract scripts
- Vyper - Python-like contract scripts
- EVM (Ethereum Virtual Machine) Assembly
- and much much more


(*) in the future.


## Yes, yes, yes - It's still "plain-vanilla" ruby

Remember - the code is still and always will be "plain-vanilla" ruby
that runs with "classic" ruby or mruby "out-of-the-box".


## Contract Samples


**Hello, World! - Greeter**

``` ruby
############################
# Greeter Contract 

def initialize( greeting )
  @owner    = msg.sender
  @greeting = greeting
end

def greet
  @greeting
end

def kill
  selfdestruct( msg.sender )  if msg.sender == @owner
end
```


**Mint Your Own Money - Minimal Viable Token**

``` ruby
#######################
# Token Contract

def initialize( initial_supply )
  @balance_of = Mapping.of( Address => Money )
  @balance_of[ msg.sender] = initial_supply
end

def transfer( to, value )
  assert @balance_of[ msg.sender ] >= value
  assert @balance_of[ to ] + value >= @balance_of[ to ]

  @balance_of[ msg.sender ] -= value
  @balance_of[ to ]         += value

  true
end
```


**Win x65 000 - Roll the (Satoshi) Dice**

``` ruby
################################
# Satoshi Dice Contract

Bet = Struct.new( :user, :block, :cap, :amount )

## Fee (Casino House Edge) is 1.9%, that is, 19 / 1000
FEE_NUMERATOR   = 19
FEE_DENOMINATOR = 1000

MAXIMUM_CAP = 2**16   # 65_536 = 2^16 = 2 byte/16 bit
MAXIMUM_BET = 100_000_000
MINIMUM_BET = 100

BetPlaced = Event.new( :id, :user, :cap, :amount )
Roll      = Event.new( :id, :rolled )

def initialize
  @owner   = msg.sender
  @counter = 0
  @bets    = Mapping.of( Integer => Bet )
end

def bet( cap )
  assert cap >= 1 && cap <= MAXIMUM_CAP
  assert msg.value >= MINIMUM_BET && msg.value <= MAXIMUM_BET

  @counter += 1
  @bets[@counter] = Bet.new( msg.sender, block.number+3, cap, msg.value )
  log BetPlaced.new( @counter, msg.sender, cap, msg.value )
end

def roll( id )
  bet = @bets[id]

  assert msg.sender == bet.user
  assert block.number >= bet.block
  assert block.number <= bet.block + 255

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

  log Roll.new( id, rolled )
  @bets.delete( id )
end

def fund
end

def kill
  assert msg.sender == @owner
  selfdestruct( @owner )
end
```

**Kick Start Your Project with a Crowd Funder**

``` ruby
##############################
# Crowd Funder Contract

State = Enum.new( :fundraising, :expired_refund, :successful )

Contribution = Struct.new( :amount, :contributor )

FundingReceived = Event.new( :address, :amount, :current_total )
WinnerPaid      = Event.new( :winner_address )


def initialize(
      time_in_hours_for_fundraising,
      campaign_url,
      fund_recipient,
      minimum_to_raise )

  @creator          = msg.sender
  @fund_recipient   = fund_recipient   # note: creator may be different than recipient
  @campaign_url     = campaign_url
  @minimum_to_raise = minimum_to_raise # required to tip, else everyone gets refund
  @raise_by         = block.timestamp + (time_in_hours_for_fundraising * 1.hour )

  @state            = State.fundraising
  @total_raised     = 0
  @complete_at      = 0
  @contributions    = Array.of( Contribution )
end



def pay_out
  assert @state.successful?

  @fund_recipient.transfer( this.balance )
  log WinnerPaid.new( @fund_recipient )
end

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


def contribute
  assert @state.fundraising?

  @contributions.push( Contribution.new( msg.value, msg.sender ))
  @total_raised += msg.value

  log FundingReceived.new( msg.sender, msg.value, @total_raised )

  check_if_funding_complete_or_expired()

  @contributions.size - 1   # return (contribution) id
end


def refund( id )
  assert @state.expired_refund?
  assert @contributions.size > id && id >= 0 && @contributions[id].amount != 0

  amount_to_refund = @contributions[id].amount
  @contributions[id].amount = 0

  @contributions[id].contributor.transfer( amount_to_refund )

  true
end

def kill
  assert msg.sender == @creator
  # wait 24 weeks after final contract state before allowing contract destruction
  assert (state.expired_refund? || state.successful?) && @complete_at + 24.weeks < block.timestamp

  # note: creator gets all money that hasn't be claimed
  selfdestruct( msg.sender )
end
```



## Request for Comments (RFC)

Send your questions and comments to the ruby-talk mailing list. Thanks!
