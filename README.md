New to (Secure) Ruby? See the [Red Paper](https://github.com/s6ruby/redpaper)!


# (Secure) Ruby to Liquidity / Michelson (Source-to-Source) Cross-Compiler Cheat Sheet / White Paper


## By Example

**Let's Vote**

``` ruby
def setup( myname )
  @votes = Mapping.of( String, Integer )
  @votes[ "ocaml"  ] = 0
  @votes[ "reason" ] = 0
  @votes[ myname   ] = 0
end

def main( choice )
  assert msg.value >= 5.tz, "Not enough money, at least 5tz to vote"
  assert @votes.has_key?( choice ), "Bad vote"

  @votes[choice] += 1
end
```

gets cross-compiled to:

``` reason
type storage = map(string, int);

let%init initial_votes = (myname: string) =>
  Map.add(myname, 0, Map([("ocaml", 0), ("reason", 0)]));

let%entry main = (choice: string, votes) => {
  let amount = Current.amount();
  if (amount < 5.00tz) {
    Current.failwith("Not enough money, at least 5tz to vote");
  } else {
    switch (Map.find(choice, votes)) {
    | None => Current.failwith("Bad vote")
    | Some(x) =>
      let votes = Map.add(choice, x + 1, votes);
      ([], votes);
    };
  };
};
```



**Minimum Viable Token**

``` ruby
struct :Account,
  balance:    Money(0),
  allowances: Mapping.of( Address, Money )

def setup( owner, total_supply, decimals, name, symbol )
  @accounts = Mapping.of( Address, Account )
  @accounts[ owner ].total_supply = total_supply

  @version      = 1
  @total_supply = total_supply
  @decimals     = decimals
  @name         = name
  @symbol       = symbol
  @owner        = owner
end  

def perform_transfer( from, dest, tokens )
  account_sender = @accounts[ from ]
  assert account_sender.balance - tokens > 0, "Not enough tokens for transfer: #{account_sender.balance}"  

  account_sender = @accounts[ dest ].balance -= token
  account_dest   = @accounts[ dest ].balance += token
end

def transfer( dest, tokens )
 perform_transfer( msg.sender, dest, tokens )
end

def approve( spender, tokens )
  account_sender = @accounts[ msg.sender ]
  if tokens == 0
    account_sender.allowances[ spender ].delete
  else
    account_sender.allowances[ spender ] = tokens
  end
end  

def transfer_from( from, dest, tokens )
  account_from = @accounts[ from ]

  assert account_from.allowances.has_key?( msg.sender ), "Not allowed to spend from: #{from}"

  allowed     = account_from.allowances[ msg.sender ]
  new_allowed = allowed - tokens
  assert new_allowed > 0, "Not enough allowance for transfer: #{allowed}"

  if new_allowed == 0
    account_from.allowances[ msg.sender ].delete
  else
    account_from.allowances[ msg.sender ] = new_allowed
  end

  perform_transfer( from, dest, tokens )
end

def create_account( dest, tokens )
  assert msg.sender == @owner, "Only owner can create accounts"
  perform_transfer( @owner, dest, tokens )
end

def create_accounts( new_accounts )
  assert msg.sender == @owner, "Only owner can create accounts"

  new_accounts.each do |rec|
    dest, tokens = rec
    perform_transfer( @owner, dest, tokens )
  end
end
```

gets cross-compiled to:

``` reason
type account = {
  balance: nat,
  allowances: map(address, nat),
};

type storage = {
  accounts: big_map(address, account),
  version: nat /* version of token standard */,
  totalSupply: nat,
  decimals: nat,
  name: string,
  symbol: string,
  owner: address,
};

let%init storage = (owner, totalSupply, decimals, name, symbol) => {
  let owner_account = {balance: totalSupply, allowances: Map};
  let accounts = Map.add(owner, owner_account, BigMap);
  {accounts, version: 1p, totalSupply, decimals, name, symbol, owner};
};

let get_account = ((a, accounts: big_map(address, account))) =>
  switch (Map.find(a, accounts)) {
  | None => {balance: 0p, allowances: Map}
  | Some(account) => account
  };

let perform_transfer = ((from, dest, tokens, storage)) => {
  let accounts = storage.accounts;
  let account_sender = get_account((from, accounts));
  let new_account_sender =
    switch (is_nat(account_sender.balance - tokens)) {
    | None =>
      failwith(("Not enough tokens for transfer", account_sender.balance))
    | Some(b) => account_sender.balance = b
    };
  let accounts = Map.add(from, new_account_sender, accounts);
  let account_dest = get_account((dest, accounts));
  let new_account_dest = account_dest.balance = account_dest.balance + tokens;
  let accounts = Map.add(dest, new_account_dest, accounts);
  ([], storage.accounts = accounts);
};

let%entry transfer = ((dest, tokens), storage) =>
  perform_transfer((Current.sender(), dest, tokens, storage));

let%entry approve = ((spender, tokens), storage) => {
  let account_sender = get_account((Current.sender(), storage.accounts));
  let account_sender =
    account_sender.allowances = (
      if (tokens == 0p) {
        Map.remove(spender, account_sender.allowances);
      } else {
        Map.add(spender, tokens, account_sender.allowances);
      }
    );
  let storage =
    storage.accounts =
      Map.add(Current.sender(), account_sender, storage.accounts);
  ([], storage);
};

let%entry transferFrom = ((from, dest, tokens), storage) => {
  let account_from = get_account((from, storage.accounts));
  let new_allowances_from =
    switch (Map.find(Current.sender(), account_from.allowances)) {
    | None => failwith(("Not allowed to spend from", from))
    | Some(allowed) =>
      switch (is_nat(allowed - tokens)) {
      | None => failwith(("Not enough allowance for transfer", allowed))
      | Some(allowed) =>
        if (allowed == 0p) {
          Map.remove(Current.sender(), account_from.allowances);
        } else {
          Map.add(Current.sender(), allowed, account_from.allowances);
        }
      }
    };
  let account_from = account_from.allowances = new_allowances_from;
  let storage =
    storage.accounts = Map.add(from, account_from, storage.accounts);
  perform_transfer((from, dest, tokens, storage));
}

let%entry createAccount = ((dest, tokens), storage) => {
  if (Current.sender() != storage.owner) {
    failwith("Only owner can create accounts");
  };
  perform_transfer((storage.owner, dest, tokens, storage));
};

let%entry createAccounts = (new_accounts, storage) => {
  if (Current.sender() != storage.owner) {
    failwith("Only owner can create accounts");
  };
  List.fold(
    (((dest, tokens), (_ops, storage))) =>
      perform_transfer((storage.owner, dest, tokens, storage)),
    new_accounts,
    ([], storage),
  );
};
```



## License

![](https://publicdomainworks.github.io/buttons/zero88x31.png)

The (secure) ruby cross-compiler scripts are dedicated to the public domain.
Use it as you please with no restrictions whatsoever.


## Request for Comments (RFC)

Send your questions and comments to the ruby-talk mailing list. Thanks!

