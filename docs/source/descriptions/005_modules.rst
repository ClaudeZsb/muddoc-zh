模组
=======

**模组是一种链上脚本，通过各种资源的注册和配置，以插件的形式为自主世界添加特定的功能。
模组可以在自主世界间共享。**

实现过程
------------

我们以官方提供的 ``uniqueentity`` 模组为例介绍模组如何为自主世界引入特定的功能。
这个模组能够为自主世界添加一个获取唯一实体 ID 的功能。
该功能本质上由一个 ``uint256`` 的单例表和一个操作这个表实现自增的系统组成。
使用该功能获取到的 ID 就是从 1 开始自增的整数。

首先看一下模组文件的目录结构：

.. code-block::

  node_modules/@latticexyz/world-modules/src/modules/uniqueentity
  ├── UniqueEntityModule.sol
  ├── UniqueEntitySystem.sol
  ├── constants.sol
  ├── getUniqueEntity.sol
  └── tables
      └── UniqueEntity.sol

``UniqueEntityModule.sol`` 是模组的安装脚本，通过调用该脚本合约完成模组的安装。
``UniqueEntitySystem.sol`` 是实现单例表自增的系统。
``constants.sol`` 是模组的常量定义， 包含即将引入的功能所在的命名空间，所需的各类资源的 ``ResourceId`` 。
``getUniqueEntity.sol`` 是模组引入的功能也就是获取唯一实体 ID 的使用脚本。
``tables/UniqueEntity.sol`` 是 ``uint256`` 的单例表。

模组的安装方式有两种：

.. code-block:: solidity

  interface IModule is IERC165, IModuleErrors {
    /**
    * @notice 安装根模组。
    * @dev 这个函数会被 World 合约在执行 `installRootModule` 过程中调用。
    * @param encodedArgs 安装过程可能需要的 ABI 编码的数据。
    */
    function installRoot(bytes memory encodedArgs) external;

    /**
    * @notice 安装模组。
    * @dev 这个函数会被 World 合约在执行 `installModule` 过程中调用。
    * @param encodedArgs 安装过程可能需要的 ABI 编码的数据。
    */
    function install(bytes memory encodedArgs) external;
  }

两种安装方法的区别主要体现在安装入口和安装脚本权限的不同。

.. code-block:: ts

  --> call, ==> delegatecall

  // 安装根模组
  User --> IWorld.installRootModule ==> IModule.installRoot
  // 安装模组
  User --> IWorld.installModule --> IModule.install

安装根模组时，我们需要调用 ``World`` 合约的 ``installRootModule`` 方法，传入模组的地址和需要的参数。
而安转普通模组时，我们需要调用 ``installModule`` 方法。另外在根模组安装过程中， ``World`` 合约通过
``delegatecall`` 调用模组的 ``installRoot`` 方法，因此模组的安装脚本在执行时具有最高权限，
相比于普通模组安装脚本能做的事情更多，例如在 ``root`` 命名空间内注册资源和配置访问权限。

``uniqueentity`` 模组的安装脚本主要完成以下几个事情：

1. 注册一个命名空间，用于存放 ``UniqueEntity`` 表和 ``UniqueEntitySystem`` 系统。
2. 注册 ``UniqueEntity`` 表。
3. 注册 ``UniqueEntitySystem`` 系统，注册系统方法的函数选择器 ``getUniqueEntity()``。

官方模组
------------

官方提供了一些常用的模组，源码见 `此处 <https://github.com/latticexyz/mud/tree/main/packages/world-modules/src/modules/uniqueentity>`_ 。

- ``uniqueentity``：提供获取唯一实体 ID 的功能。
- ``keysintable``：为指定表统计存在的键，提供获取所有存在的键的功能。
- ``keyswithvalue``：为指定表的值建立索引，提供通过值查询键的功能。
- ``callwithsignature`` ：提供通过 EIP712 签名调用合约的功能。
- ``std-delegations``：提供几种常见的委托调用模式，包括限定次数、限定系统、限定时间。
- ``puppet``：提供为系统合约创建代理合约的功能，可以通过代理合约调用系统合约。
- ``erc20-puppet``：一种基于 ``puppet`` 模组实现的 ``ERC20`` 代币合约。
- ``erc721-puppet``：一种基于 ``puppet`` 模组实现的 ``ERC721`` 代币合约。

模组安装
------------

通过配置文件安装
^^^^^^^^^^^^^^^^^^^

我们建议通过编辑配置文件 ``mud.config.ts`` 安装模组。

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";
  import { resolveTableId } from "@latticexyz/world/internal";
  import { encodeAbiParameters, parseAbiParameters, toHex } from 'viem';

  export default defineWorld({
    tables: {
      StoredUint: {
        schema:{
          value: "uint256",
        },
        key: [],
      },
    },
    modules: [
      {
        artifactPath: "@latticexyz/world-modules/out/KeysInTableModule.sol/KeysInTableModule.json",
        root: true,
        args: [resolveTableId("StoredUint")],
      },
      {
        artifactPath: "@latticexyz/world-modules/out/PuppetModule.sol/PuppetModule.json",
        root: true,
        args: [],
      },
      {
        artifactPath: "@latticexyz/world-modules/out/ERC20Module.sol/ERC20Module.json",
        root: false,
        args: [
          {type: "bytes", value: encodeAbiParameters(
            parseAbiParameters('bytes14 namespace, (uint8 decimals, string name, string symbol)'),
            [toHex("token", { size: 14 }), {decimals: 18, name: "muddoc", symbol: "MUDOC"}],
          )}
        ],
      },
    ],
  });

上面的配置文件完成了三个模组的安装，分别是 ``keysintable``、 ``puppet`` 和 ``erc20-puppet``。
需要留意的是模组的安装方式，有些模组同时支持以跟模组或普通模组的形式安装，而有的只支持其中的一种。
安装一个模组前，先查看它支持的安装方式。其次我们要提供必要的安装参数，例如安装 ``keysintable`` 模组时，
我们需要指定为哪个表统计存在的键。再比如，安装 ``erc20-puppet`` 模组时，我们需要提供 ``ERC20`` 代币系统所在的命名空间、
代币的名称、符号和精度。

模组的配置项：

- ``artifactPath``：模组合约编译后的 JSON 文件路径。既支持本地相对路径，又支持包引入路径。
- ``root``：是否以根模组形式安装。
- ``args``：安装模组时需要的参数。

手动安装
^^^^^^^^^^^^

手动安装模组需要根据安装方式调用 ``World`` 合约的 ``installModule`` 或 ``installRootModule`` 方法。

与上面的配置文件安装内容等价的手动安装脚本如下：

.. code-block:: solidity

  // SPDX-License-Identifier: MIT
  pragma solidity ^0.8.0;

  import { Script } from "forge-std/Script.sol";
  import { StoredUint } from "../src/codegen/index.sol";
  import { IWorld } from "../src/codegen/world/IWorld.sol";
  import { KeysInTableModule } from "@latticexyz/world-modules/src/modules/keysintable/KeysInTableModule.sol";
  import { PuppetModule } from "@latticexyz/world-modules/src/modules/puppet/PuppetModule.sol";
  import { registerERC20 } from "@latticexyz/world-modules/src/modules/erc20-puppet/registerERC20.sol";
  import { ERC20MetadataData } from "@latticexyz/world-modules/src/modules/erc20-puppet/tables/ERC20Metadata.sol";
  import { IERC20Mintable } from "@latticexyz/world-modules/src/modules/erc20-puppet/IERC20Mintable.sol";

  contract InstallModule is Script {
    function run(address worldAddress) public {
      // 安装根模组 `keysintable` 统计 `StoredUint` 表中的键
      IWorld(worldAddress).installRootModule(new KeysInTableModule(), abi.encode(StoredUint._tableId));
      // 安装根模组 `puppet`
      IWorld(worldAddress).installRootModule(new PuppetModule(), new bytes(0));
      // 安装模组 `erc20-puppet`，创建一个 `ERC20` 代币
      IERC20Mintable token = registerERC20(
        IWorld(worldAddress),
        bytes14("token"),
        ERC20MetadataData({ decimals: 6, name: "muddoc", symbol: "MUDOC" })
      );
    }
  }

编写模组
------------

这里有一份官方的 `模组编写教程 <https://mud.dev/guides/modules>`_ 。

