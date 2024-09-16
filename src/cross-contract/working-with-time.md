# Working with time

The concept of time in the blockchain is tricky - as in
every distributed system, it is not easy to synchronize the
clocks of all the nodes.

However, there is the notion of a time that is
monotonic - which means that it should never go "backward"
between executions. Also, what is important is - time is
always unique throughout the whole transaction - and even
the entire block, which is made up of multiple transactions.

The time is encoded in the
[`Env`](https://docs.rs/cosmwasm-std/1.2.4/cosmwasm_std/struct.Env.html)
type in its
[`block`](https://docs.rs/cosmwasm-std/1.2.4/cosmwasm_std/struct.BlockInfo.html)
field, which looks like this:

```rust
pub struct BlockInfo {
    pub height: u64,
    pub time: Timestamp,
    pub chain_id: String,
}
```

You can see the `time` field, which is the timestamp of the
processed block. The `height` field is also worth
mentioning - it contains a sequence number of the processed
block. It is sometimes more useful than time, as it is
guaranteed that the `height` field increases
between blocks, while two blocks may be executed within the
same `time` (even though it is rather improbable).

Also, many transactions might be executed in a single block.
That means that if we need a unique id for the execution of
a particular message, we should look for something more.
This thing is a
[`transaction`](https://docs.rs/cosmwasm-std/1.2.4/cosmwasm_std/struct.TransactionInfo.html)
field of the `Env` type:

```rust
pub struct TransactionInfo {
    pub index: u32,
}
```

The `index` here contains a unique index of the transaction
in the block. That means that to get the unique identifier
of a transaction through the whole block, we can use the
`(height, transaction_index)` pair.

## Join time

We want to use the time in our system to keep track of the
join time of admins. We don't yet add new members to the
group, but we can already set the join time of initial
admins. Let's start updating our state:

```rust
use cosmwasm_std::{Addr, Timestamp};
use cw_storage_plus::Map;
# use cw_storage_plus::Item;

pub const ADMINS: Map<&Addr, Timestamp> = Map::new("admins");
# pub const DONATION_DENOM: Item<String> = Item::new("donation_denom");
```

As you can see, our admins set became a proper map - we will assign the join time to every admin.

You might argue that we should create a separate structure for the value of this map, so that in the future, if we need to add something there, we can. However, in my opinion, this would be premature optimization. We can also change the entire value type in the future, as it would be the same breaking change.

Now we need to update how we initialize the map. We previously stored Empty data, but it no longer matches our value type. Let's examine the updated instantiation function:

```rust
use crate::state::{ADMINS, DONATION_DENOM};
use cosmwasm_std::{
    DepsMut, Env, MessageInfo, Response, StdResult,
};

pub fn instantiate(
    deps: DepsMut,
    env: Env,
    _info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    for addr in msg.admins {
        let admin = deps.api.addr_validate(&addr)?;
        ADMINS.save(deps.storage, &admin, &env.block.time)?;
    }
    DONATION_DENOM.save(deps.storage, &msg.donation_denom)?;

    Ok(Response::new())
}
```

Instead of storing `&Empty {}` as an admin value, we store
the join time, which we read from `&env.block.time`. Also,
note that I removed the underscore from the name of the
`env` block - it was there to tell the Rust compiler
the variable is purposely unused and not some kind of a bug.

Similarly, we need to update the add_members function, and the way we are calling the function:

```rust
pub fn add_members(
        deps: DepsMut,
        env: Env,
        info: MessageInfo,
        admins: Vec<String>,
    ) -> Result<Response, ContractError> {
    
        for addr in admins {
            let admin = deps.api.addr_validate(&addr)?;
            ADMINS.save(deps.storage, &admin, &env.block.time)?;
        }

        Ok(resp)
    }

    pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    use ExecuteMsg::*;

    match msg {
        AddMembers { admins } => execute::add_members(deps, env, info, admins),
        Leave {} => execute::leave(deps, info).map_err(Into::into),
        Donate {} => execute::donate(deps, info),
    }
}
```
Note that we now need to use and pass in Env to the add_members function as we need to access the block timestamp. 

Finally, remember to remove any obsolete `Empty` imports
through the project - the compiler should help you point out
unused imports.

## Query and tests

The last thing to add regarding join time is the new query
asking for the join time of a particular admin. Everything
you need to do that was already discussed, I'll leave it for
you as an exercise. The query variant should look like:

```rust
#[returns(JoinTimeResp)]
JoinTime { admin: String },
```

And the example response type:

```rust
#[cw_serde]
pub struct JoinTimeResp {
    pub joined: Timestamp,
}
```

You may ask why I suggest always returning a joined value in the response, and what to do when no such admin is added. Well, in such a case, I would rely on the fact that the load function returns a descriptive error of a missing value in storage. However, feel free to define your own error for such a case or even make the joined field optional, to be returned only if the requested admin exists.

Finally, it would be a good idea to create a test for the new functionality. Call the new query right after instantiation to verify that initial admins have the proper join time (possibly by extending the existing instantiation test).

One thing you might need help with in tests is how to get the time of execution. Using any OS-time would be doomed to fail. Instead, you can call the [`block_info`](https://docs.rs/cw-multi-test/0.16.4/cw_multi_test/struct.App.html#method.block_infohttps://docs.rs/cw-multi-test/0.16.4/cw_multi_test/struct.App.html#method.block_info) function to access the [`BlockInfo`](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/struct.BlockInfo.html) structure containing the block state at a particular moment in the app. Calling it just before instantiation would ensure you are working with the same state that would be simulated on the call.
