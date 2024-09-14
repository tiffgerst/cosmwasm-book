# Fixing admin contract

Now that we know what we want to achieve, we can start by aligning the
contract we already have to become an admin contract. It is mostly
fine at this point, but we want to do a cleanup.

## Cleaning up queries

The first thing to do is to get rid of the `Greet` query - it was good as a
starter query example, but it has no practical purpose and only generates noise.

We want to remove the unnecessary variant from the query enum:

```rust
# use cosmwasm_schema::{cw_serde, QueryResponses};
# use cosmwasm_std::Addr;
# 
# #[cw_serde]
# pub struct InstantiateMsg {
#     pub admins: Vec<String>,
#     pub donation_denom: String,
# }
# 
# #[cw_serde]
# pub enum ExecuteMsg {
#     Leave {},
#     Donate {},
# }
# 
# #[cw_serde]
# pub struct AdminsListResp {
#     pub admins: Vec<Addr>,
# }
#
#[cw_serde]
#[derive(QueryResponses)]
pub enum QueryMsg {
    #[returns(AdminsListResp)]
    AdminsList {},
}
```

Then we also remove the invalid path in the query dispatcher:

```rust
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    use QueryMsg::*;

    match msg {
        AdminsList {} => to_json_binary(&query::admins_list(deps)?),
    }
}
```

Finally, we remove the irrelevant handler from the `contract::query` module.
We also need to make sure all references to it are gone (eg. if there are any
in the tests).

## Change contract name
"contract" is not very descriptive, so lets update the name in the Cargo.toml to "admin". You may need to update the import in schema.rs as well.

```toml
[package]
name = "admin"
version = "0.1.0"
edition = "2021"
```

## Project structure

Last but not least - we want to better structure our project. So far, we have
only one contract, so we just worked on it as a whole project. Now we want some
directory tree that reflects relations between contracts we create.

First, create a directory for the project. Then we want to create a "contracts"
subdirectory in it. It is not technically required from Rust's POV, but there
are tools in our environment, like the workspace optimizer, which would assume
it is where it should look for a contract. It is a common pattern you will see
in CosmWasm contracts repos.

Then we copy the whole project directory from the previous chapter into the
`contracts`, renaming it to `admin`.

Finally, we want to couple all our projects (for now, it is just one, but we know
there will be more there). To do so, we create the workspace-level `Cargo.toml`
file in the top-level project directory:

```toml
[workspace]
members = ["contracts/*"]
resolver = "2"
```

This `Cargo.toml` differs slightly from the typical project-level one - it
defines the workspace. The most important field here is the `members` - it
defines projects being part of the workspace.

The other field is the `resolver`. It is something to remember to add - it
instructs cargo to use version 2 of the dependency resolver. This has been the
default for non-workspaces since Rust 2021, but because of compatibility reasons,
the default couldn't be changed for workspaces - but it is advised to add it to
every single newly created workspace.

The last field which might be useful for workspaces is exclude - it allows to
create projects in the workspace directory tree, which is not a part of this
workspace - we will not use it, but it is good to know about it.

Now just for clarity, let's see the top-level directory structure:

```none
.
├── Cargo.lock
├── Cargo.toml
├── contracts
│  └── admin
└── target
   ├── CACHEDIR.TAG
   └── debug
```

You can see the target directory and `Cargo.lock` files existing in the tree - it is
because I already built and ran tests for the `admin` contract - in Rust workspaces,
`cargo`` knows to build everything in the top level, even if it would be built from
the inner directory.
