New to (Secure) Ruby? See the [Red Paper](https://github.com/s6ruby/redpaper)!


# (Secure) Ruby to Liquidity (w/ ReasonML Syntax) / Michelson (Source-to-Source) Cross-Compiler Cheat Sheet / White Paper


## By Example


**Let's Count - 0, 1, 2, 3**

``` ruby
def setup
  @counter = 0
end

sig [Integer],
def inc( by )
  @counter += by
end
```

gets cross-compiled to:

``` reason
type storage = int;

let%init setup = () => {
  0;
};

let%entry inc = (by: int, storage) => {
  ([], storage + by);
};
```



**Let's Vote**

``` ruby
def setup
  @votes = Mapping.of( String => Integer )
  @votes[ "ocaml"  ] = 0
  @votes[ "reason" ] = 0
  @votes[ "ruby"   ] = 0
end

sig [String],
def vote( choice )
  assert msg.value >= 5.tz, "Not enough money, at least 5tz to vote"
  assert @votes.has_key?( choice ), "Bad vote"

  @votes[choice] += 1
end
```

gets cross-compiled to:

``` reason
type storage = map(string, int);

let%init setup = () => {
  Map([("ocaml", 0), ("reason", 0), ("ruby", 0)]);
};

let%entry vote = (choice: string, votes) => {
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
  allowances: Mapping.of( Address => Money )

sig [Address, Money, Integer, String, String],
def setup( owner, total_supply, decimals, name, symbol )
  @accounts = Mapping.of( Address => Account )
  @accounts[ owner ].total_supply = total_supply

  @version      = 1
  @total_supply = total_supply
  @decimals     = decimals
  @name         = name
  @symbol       = symbol
  @owner        = owner
end  

sig [Address, Money],
def transfer( dest, tokens )
 perform_transfer( msg.sender, dest, tokens )
end

sig [Address, Money],
def approve( spender, tokens )
  account_sender = @accounts[ msg.sender ]
  if tokens == 0
    account_sender.allowances[ spender ].delete
  else
    account_sender.allowances[ spender ] = tokens
  end
end  

sig [Address, Address, Money],
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

sig [Address, Money],
def create_account( dest, tokens )
  assert msg.sender == @owner, "Only owner can create accounts"
  perform_transfer( @owner, dest, tokens )
end

private

sig [Address, Address, Money],
def perform_transfer( from, dest, tokens )
  account_sender = @accounts[ from ]
  assert account_sender.balance - tokens > 0, "Not enough tokens for transfer: #{account_sender.balance}"  

  account_sender.balance -= token
  account_dest = @accounts[ dest ]
  account_dest.balance   += token
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

let%init setup = (owner, totalSupply, decimals, name, symbol) => {
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
```


**Roll the Dice**

``` ruby
# to be done
```

gets cross-compiled to:

``` reason
type game = {
  number: nat,
  bet: tez,
  player: key_hash,
};

type storage = {
  game: option(game),
  oracle_id: address,
};

let%init setup = (oracle_id: address) => {game: None, oracle_id} /* Start a new game */;

let%entry play = ((number: nat, player: key_hash), storage) => {
  if (number > 100p) {
    failwith("number must be <= 100");
  };
  if (Current.amount() == 0tz) {
    failwith("bet cannot be 0tz");
  };
  if (2p * Current.amount() > Current.balance()) {
    failwith("I don't have enough money for this bet");
  };
  switch (storage.game) {
  | Some(g) => failwith(("Game already started with", g))
  | None =>
    let bet = Current.amount();
    let storage = storage.game = Some({number, bet, player});
    ([], storage);
  };
} 

/* Receive a random number from the oracle and compute outcome of the
   game */

let%entry finish = (random_number: nat, storage) => {
  let random_number =
    switch (random_number / 101p) {
    | None => failwith()
    | Some((_, r)) => r
    };
  if (Current.sender() != storage.oracle_id) {
    failwith("Random numbers cannot be generated");
  };
  switch (storage.game) {
  | None => failwith("No game already started")
  | Some(game) =>
    let ops =
      if (random_number < game.number) {
        /* Lose */
        [];
      } else {
        /* Win */
        let gain =
          switch (game.bet * game.number / 100p) {
          | None => 0tz
          | Some((g, _)) => g
          };
        let reimbursed = game.bet + gain;
        [Account.transfer(~dest=game.player, ~amount=reimbursed)];
      };

    let storage = storage.game = None;
    (ops, storage);
  };
} 

/* accept funds */;
let%entry fund = ((), storage) => ([], storage);
```

**Let's Vote (Again)**

``` ruby
# to be done
```

gets cross-compiled to:

``` reason
/* Smart contract for voting. Winners of vote split the contract
   balance at the end of the voting period. */

type storage = {
  voters: big_map(address, unit)   /*** Used to register voters */,
  votes: map(string, nat)          /*** Keep track of vote counts */,
  addresses: map(string, key_hash) /*** Addresses for payout */,
  deadline: timestamp              /*** Deadline after which vote closes */,
} 

let%init setup = addresses => {
  /* Initialize vote counts to zero */
  votes:
    Map.fold(
      (((name, _kh), votes)) => Map.add(name, 0p, votes),
      addresses,
      Map,
    ),
  addresses,
  voters: BigMap /* No voters */,
  deadline: Current.time() + 3600 * 24 /* 1 day from now */,
}

/** Entry point for voting.
    @param choice A string corresponding to the candidate */

let%entry vote = (choice, storage) => {
  /* Only allowed while voting period is ongoing */
  if (Current.time() > storage.deadline) {
    failwith("Voting closed");
  } /* Voter must send at least 5tz to vote */;
  if (Current.amount() < 5.00tz) {
    failwith("Not enough money, at least 5tz to vote");
  } /* Voter cannot vote twice */;
  if (Map.mem(Current.sender(), storage.voters)) {
    failwith(("Has already voted", Current.sender()));
  };
  let votes = storage.votes;
  switch (Map.find(choice, votes)) {
  | None =>
    /* Vote must be for an existing candidate */
    failwith(("Bad vote", choice))
  | Some(x) =>
    /* Increase vote count for candidate */
    let storage = storage.votes = Map.add(choice, x + 1p, votes) /* Register voter */;
    let storage =
      storage.voters = Map.add(Current.sender(), (), storage.voters) /* Return updated storage */;
    ([], storage);
  };
}

/* Auxiliary function : returns the list of candidates with the
   maximum number of votes (there can be more than one in case of
   draw). */;

let find_winners = votes => {
  let (winners, _max) =
    Map.fold(
      (((name, nb), (winners, max))) =>
        if (nb == max) {
          ([name, ...winners], max);
        } else if (nb > max) {
          ([name], nb);
        } else {
          (winners, max);
        },
      votes,
      ([], 0p),
    );
  winners;
};

/** Entry point for paying winning candidates. */

let%entry payout = ((), storage) => {
  /* Only allowed once voting period is over */
  if (Current.time() <= storage.deadline) {
    failwith("Voting ongoing");
  } /* Indentify winners of vote */;
  let winners = find_winners(storage.votes) /* Balance of contract is split equally between winners */;
  let amount =
    switch (Current.balance() / List.length(winners)) {
    | None => failwith("No winners")
    | Some((v, _rem)) => v
    } /* Generate transfer operations */;
  let operations =
    List.map(
      name => {
        let dest =
          switch (Map.find(name, storage.addresses)) {
          | None => failwith() /* This cannot happen */
          | Some(d) => d
          };
        Account.transfer(~amount, ~dest);
      },
      winners,
    ) /* Return list of operations. Storage is unchanged */;
  (operations, storage);
};
```


## Notes

The Liquidity Language for programming contracts with OCaml or ReasonML syntax (see <http://www.liquidity-lang.org>)
compiles to Michelson bytecode (see <https://www.michelson-lang.com>).


## License

![](https://publicdomainworks.github.io/buttons/zero88x31.png)

The (secure) ruby cross-compiler scripts are dedicated to the public domain.
Use it as you please with no restrictions whatsoever.


## Request for Comments (RFC)

Send your questions and comments to the ruby-talk mailing list. Thanks!

