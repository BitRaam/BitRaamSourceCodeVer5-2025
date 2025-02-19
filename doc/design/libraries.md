# Libraries

| Name                     | Description |
|--------------------------|-------------|
| *libbitraam_cli*         | RPC client functionality used by *bitraam-cli* executable |
| *libbitraam_common*      | Home for common functionality shared by different executables and libraries. Similar to *libbitraam_util*, but higher-level (see [Dependencies](#dependencies)). |
| *libbitraam_consensus*   | Consensus functionality used by *libbitraam_node* and *libbitraam_wallet*. |
| *libbitraam_crypto*      | Hardware-optimized functions for data encryption, hashing, message authentication, and key derivation. |
| *libbitraam_kernel*      | Consensus engine and support library used for validation by *libbitraam_node*. |
| *libbitraamqt*           | GUI functionality used by *bitraam-qt* and *bitraam-gui* executables. |
| *libbitraam_ipc*         | IPC functionality used by *bitraam-node*, *bitraam-wallet*, *bitraam-gui* executables to communicate when [`-DWITH_MULTIPROCESS=ON`](multiprocess.md) is used. |
| *libbitraam_node*        | P2P and RPC server functionality used by *bitraamd* and *bitraam-qt* executables. |
| *libbitraam_util*        | Home for common functionality shared by different executables and libraries. Similar to *libbitraam_common*, but lower-level (see [Dependencies](#dependencies)). |
| *libbitraam_wallet*      | Wallet functionality used by *bitraamd* and *bitraam-wallet* executables. |
| *libbitraam_wallet_tool* | Lower-level wallet functionality used by *bitraam-wallet* executable. |
| *libbitraam_zmq*         | [ZeroMQ](../zmq.md) functionality used by *bitraamd* and *bitraam-qt* executables. |

## Conventions

- Most libraries are internal libraries and have APIs which are completely unstable! There are few or no restrictions on backwards compatibility or rules about external dependencies. An exception is *libbitraam_kernel*, which, at some future point, will have a documented external interface.

- Generally each library should have a corresponding source directory and namespace. Source code organization is a work in progress, so it is true that some namespaces are applied inconsistently, and if you look at [`add_library(bitraam_* ...)`](../../src/CMakeLists.txt) lists you can see that many libraries pull in files from outside their source directory. But when working with libraries, it is good to follow a consistent pattern like:

  - *libbitraam_node* code lives in `src/node/` in the `node::` namespace
  - *libbitraam_wallet* code lives in `src/wallet/` in the `wallet::` namespace
  - *libbitraam_ipc* code lives in `src/ipc/` in the `ipc::` namespace
  - *libbitraam_util* code lives in `src/util/` in the `util::` namespace
  - *libbitraam_consensus* code lives in `src/consensus/` in the `Consensus::` namespace

## Dependencies

- Libraries should minimize what other libraries they depend on, and only reference symbols following the arrows shown in the dependency graph below:

<table><tr><td>

```mermaid

%%{ init : { "flowchart" : { "curve" : "basis" }}}%%

graph TD;

bitraam-cli[bitraam-cli]-->libbitraam_cli;

bitraamd[bitraamd]-->libbitraam_node;
bitraamd[bitraamd]-->libbitraam_wallet;

bitraam-qt[bitraam-qt]-->libbitraam_node;
bitraam-qt[bitraam-qt]-->libbitraamqt;
bitraam-qt[bitraam-qt]-->libbitraam_wallet;

bitraam-wallet[bitraam-wallet]-->libbitraam_wallet;
bitraam-wallet[bitraam-wallet]-->libbitraam_wallet_tool;

libbitraam_cli-->libbitraam_util;
libbitraam_cli-->libbitraam_common;

libbitraam_consensus-->libbitraam_crypto;

libbitraam_common-->libbitraam_consensus;
libbitraam_common-->libbitraam_crypto;
libbitraam_common-->libbitraam_util;

libbitraam_kernel-->libbitraam_consensus;
libbitraam_kernel-->libbitraam_crypto;
libbitraam_kernel-->libbitraam_util;

libbitraam_node-->libbitraam_consensus;
libbitraam_node-->libbitraam_crypto;
libbitraam_node-->libbitraam_kernel;
libbitraam_node-->libbitraam_common;
libbitraam_node-->libbitraam_util;

libbitraamqt-->libbitraam_common;
libbitraamqt-->libbitraam_util;

libbitraam_util-->libbitraam_crypto;

libbitraam_wallet-->libbitraam_common;
libbitraam_wallet-->libbitraam_crypto;
libbitraam_wallet-->libbitraam_util;

libbitraam_wallet_tool-->libbitraam_wallet;
libbitraam_wallet_tool-->libbitraam_util;

classDef bold stroke-width:2px, font-weight:bold, font-size: smaller;
class bitraam-qt,bitraamd,bitraam-cli,bitraam-wallet bold
```
</td></tr><tr><td>

**Dependency graph**. Arrows show linker symbol dependencies. *Crypto* lib depends on nothing. *Util* lib is depended on by everything. *Kernel* lib depends only on consensus, crypto, and util.

</td></tr></table>

- The graph shows what _linker symbols_ (functions and variables) from each library other libraries can call and reference directly, but it is not a call graph. For example, there is no arrow connecting *libbitraam_wallet* and *libbitraam_node* libraries, because these libraries are intended to be modular and not depend on each other's internal implementation details. But wallet code is still able to call node code indirectly through the `interfaces::Chain` abstract class in [`interfaces/chain.h`](../../src/interfaces/chain.h) and node code calls wallet code through the `interfaces::ChainClient` and `interfaces::Chain::Notifications` abstract classes in the same file. In general, defining abstract classes in [`src/interfaces/`](../../src/interfaces/) can be a convenient way of avoiding unwanted direct dependencies or circular dependencies between libraries.

- *libbitraam_crypto* should be a standalone dependency that any library can depend on, and it should not depend on any other libraries itself.

- *libbitraam_consensus* should only depend on *libbitraam_crypto*, and all other libraries besides *libbitraam_crypto* should be allowed to depend on it.

- *libbitraam_util* should be a standalone dependency that any library can depend on, and it should not depend on other libraries except *libbitraam_crypto*. It provides basic utilities that fill in gaps in the C++ standard library and provide lightweight abstractions over platform-specific features. Since the util library is distributed with the kernel and is usable by kernel applications, it shouldn't contain functions that external code shouldn't call, like higher level code targeted at the node or wallet. (*libbitraam_common* is a better place for higher level code, or code that is meant to be used by internal applications only.)

- *libbitraam_common* is a home for miscellaneous shared code used by different Bitcoin Core applications. It should not depend on anything other than *libbitraam_util*, *libbitraam_consensus*, and *libbitraam_crypto*.

- *libbitraam_kernel* should only depend on *libbitraam_util*, *libbitraam_consensus*, and *libbitraam_crypto*.

- The only thing that should depend on *libbitraam_kernel* internally should be *libbitraam_node*. GUI and wallet libraries *libbitraamqt* and *libbitraam_wallet* in particular should not depend on *libbitraam_kernel* and the unneeded functionality it would pull in, like block validation. To the extent that GUI and wallet code need scripting and signing functionality, they should be get able it from *libbitraam_consensus*, *libbitraam_common*, *libbitraam_crypto*, and *libbitraam_util*, instead of *libbitraam_kernel*.

- GUI, node, and wallet code internal implementations should all be independent of each other, and the *libbitraamqt*, *libbitraam_node*, *libbitraam_wallet* libraries should never reference each other's symbols. They should only call each other through [`src/interfaces/`](../../src/interfaces/) abstract interfaces.

## Work in progress

- Validation code is moving from *libbitraam_node* to *libbitraam_kernel* as part of [The libbitraamkernel Project #27587](https://github.com/bitraam/bitraam/issues/27587)
