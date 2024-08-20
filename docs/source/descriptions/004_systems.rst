系统
=======

**自主世界的功能主要由系统来表达。系统是以合约形式部署的链上公共代码库。
正常情况下，它不持有任何资产，不具备任何状态存储，并且能够被所有自主世界共享。**

开发一个系统
--------------

系统合约需要继承 ``@latticexyz/world/src/System.sol``。这个基础合约提供了系统需要的的基础功能，
并指导我们正确的开发一个系统。

``System.sol`` 提供了三个非常重要的函数：

- ``_world()``: 返回系统合约被调用时关联的 ``World`` 合约地址。
- ``_msgSender()``: 返回关联的 ``World`` 合约的调用者的地址。
- ``_msgValue()``: 返回调用者调用关联的 ``World`` 合约时携带的 ETH 数量。

假设同一个 ``TokenSystem`` 的 ``buy`` 函数在两个自主世界中被用户调用

.. code-block:: ts

  --> call, ==> delegatecall

  User1 (with 0.5 eth) --> WorldAbc --> TokenSystem.buy // TokenSystem in non-root namespace

  User2 (with 1 eth) --> WorldXyz ==> TokenSystem.buy // TokenSystem in root namespace

那么在第一次调用中，对 ``TokenSystem`` 而言

.. code-block:: solidity

  _world() == address(WorldAbc)
  _msgSender() == address(User1)
  msg.sender == address(WorldAbc)
  _msgValue() == 0.5 ether
  msg.value == 0

.. note::

  ``World`` 作为交互入口的同时统一保管所有转入的 ETH。任何 ``World`` 到 ``System`` 的路由调用都不会携带 ETH。

在第二次调用中

.. code-block:: solidity

  _world() == address(WorldXyz)
  _msgSender() == address(User2)
  msg.sender == address(User2)
  _msgValue() == 1 ether
  msg.value == 1 ether

我们可以发现，对于第一种情况，经过 ``World`` 的路由调用， ``msg.sender`` 会变成 ``World`` 合约地址，
``msg.value`` 也会变为 0。而无论哪种情况下， ``_msgSender()`` 和 ``_msgValue()`` 都能返回实际期望的值。
正因此，我们在开发系统时， **应该尽量使用 _msgSender() 和 _msgValue() 来获取逻辑意义上的调用者和携带的 ETH**。

.. tip::

  不要使用 ``this`` 。尤其是不要使用 ``address(this)`` 做任何逻辑判断或者作为参数传递。

此外，我们必须注意到，同一个系统合约是可以被多个自主世界使用的。这表现为，同一个系统合约在不同的调用场景下
``_world()`` 返回的 ``World`` 地址是不同的。而 ``World`` 合约定义和储存了系统要操作的表， **因此在不同
的 World 背景下，系统实际操作的表所在的合约也是不同的。**

调用其他系统
^^^^^^^^^^^^^^

在章节 :ref:`dev-differences` 中，我们通过 ``IWorld(_world()).muddoc__setUint(x)``
的方式实现了 ``SimpleStorageCallerSystem`` 调用 ``SimpleStorageSystem`` 的 ``setUint``
方法。

这是最常用也是最常见的方式，但只有符合特定条件的两个系统才能这样做。

- 最基本的，调用系统需要有被调用系统的访问权限，这对所有的系统调用都适用。
- 其次，要求调用系统属于自定义命名空间。
- 最后要求被调用系统在 ``World`` 合约上注册了对应的函数选择器。

  .. note::

    ``IWorld`` 是一个由 ``Mud CLI: worldgen`` 自动生成的接口，它包含了所有系统注册在 ``World``
    合约上的函数选择器对应的函数借口。

假设没有权限的问题，调用系统所在的命名空间，被调用系统所在的命名空间，以及被调用系统的方法注册情况，都会影响
我们实现系统间调用的方式。

如果调用系统属于 ``root`` 命名空间，推荐使用 ``SystemSwitch``

.. note::

  ``SystemSwitch`` 适用于任何情况下的系统间调用。但手动编码 calldata 极不方便。
  如果你明确知道调用系统属于自定义命名空间，且被调用系统在 ``World`` 合约上注册了对应的函数选择器，
  那么推荐直接使用 ``IWorld`` 接口中自动生成的函数。

.. code-block:: solidity

  // SPDX-License-Identifier: MIT
  pragma solidity >=0.8.24;

  import { WorldResourceIdLib } from "@latticexyz/world/src/WorldResourceId.sol";
  import { System } from "@latticexyz/world/src/System.sol";
  import { ResourceId } from "@latticexyz/store/src/ResourceId.sol";
  import { IWorld } from "../codegen/world/IWorld.sol";
  import { SystemSwitch } from "@latticexyz/world-modules/src/utils/SystemSwitch.sol";
  import { SimpleStorageSystem } from "./SimpleStorageSystem.sol";

  contract SimpleStorageCallerSystem is System {
    function getUintFromSimpleStorageSystem() public view returns (uint) {
      ResourceId simpleStorageSystemId = WorldResourceIdLib.encode("sy", "muddoc", "SimpleStorage");
      return abi.decode(
        SystemSwitch.call(simpleStorageSystemId, abi.encodeWithSelector(SimpleStorageSystem.getUint.selector)),
        (uint256)
      );
    }
  }

如果调用系统属于自定义命名空间，且被调用系统未注册系统方法，推荐使用 ``IWorld.call``。

.. note::

  相比于 ``SystemSwitch``，直接使用 ``IWorld.call`` 可以节省一个 ``if...else...`` 判断。

.. code-block:: solidity

  function getUintFromSimpleStorageSystem() public view returns (uint) {
    ResourceId simpleStorageSystemId = WorldResourceIdLib.encode("sy", "muddoc", "SimpleStorage");
    return abi.decode(
      IWorld(_world()).call(simpleStorageSystemId, abi.encodeWithSelector(SimpleStorageSystem.getUint.selector)),
      (uint256)
    );
  }

如果调用系统属于自定义命名空间，且被调用系统注册了系统方法，推荐像
:ref:`dev-differences_contract_interaction` 中那样，
直接使用 ``IWorld`` 中对应的系统方法接口。

为了更清晰展示系统间调用的实现方式，不同情况下的完整调用链路如下：

.. code-block:: ts

  --> call, ==> delegatecall

  // root 系统 调用 root 系统，无论被调用系统有无注册系统方法
  User --> World ==> SystemFrom ==> SystemTo.foo()
  // root 系统 调用 root 系统，无论被调用系统有无注册系统方法
  User --> World ==> SystemFrom --> SystemTo
  // 非 root 系统 调用 root 系统，被调用系统未注册系统方法
  User --> World --> SystemFrom --> World.call() ==> SystemTo.foo()
  // 非 root 系统 调用非 root 系统，被调用系统未注册系统方法
  User --> World --> SystemFrom --> World.call() --> SystemTo.foo()
  // 非 root 系统 调用 root 系统，被调用系统注册了系统方法
  User --> World --> SystemFrom --> World.fallback() ==> SystemTo.foo()
  // 非 root 系统 调用非 root 系统，被调用系统注册了系统方法
  User --> World --> SystemFrom --> World.fallback() --> SystemTo.foo()

.. note::

  当调用系统属于 ``root`` 命名空间时，不能以 ``call`` 的形式调用 ``World`` 做调用路由。
  虽然可以用 ``delegatecall`` 但是多余的调用浪费了 ``gas``。

  .. code-block::

    User --> World ==> SystemFrom -❌-> World ==> SystemTo.foo()
    User --> World ==> SystemFrom (==> World) ==> SystemTo.foo()


调用外部合约
^^^^^^^^^^^^^^

谨慎以 ``call`` 形式调用不是 ``System`` 的合约，包括其他 ``World`` 合约。
尤其是当被调用的合约使用 ``msg.sender`` 作为参数的情况。

.. important::
  如果发起外部合约调用的系统 ``SystemX`` 属于自定义命名空间，那么这次跨合约交互的调用者将是
  ``SystemX``，不是 ``World``，也不是 ``tx.origin``。这意味着对于被调用的外部合约而言，
  ``msg.sender == address(SystemX)``。
  一旦被调用的外部合约使用 ``msg.sender`` 作为参数，可能造成财产损失。因为通常情况下，
  ``System`` 被认为是公共的可重复使用的代码库资源。

  假如 ``SystemX`` 能够将一部分 USDT 存入一个依赖 ``msg.sender`` 作资金来源的
  链上 Defi 挖矿池， 并且实现了与之对应的从池子中取走存入的 USDT 的方法。那么任何一个人都可以通过复用该系统，
  将这些存入的 USDT 取走。 即使在提取资产的方法实现中加入了权限控制，也无法阻止这种行为。因为按照默认，
  系统合约实现权限控制所依赖的数据存储在 ``World`` 中，而 ``World`` 合约是根据谁在使用 ``SystemX``
  来确定的。当你的自主世界在使用这个系统合约时，就从你的 ``World`` 合约读取数据。
  当攻击者的自主世界在使用 ``SystemX`` 时，就从他的 ``World`` 合约读取数据，那时他就可以根据需要提供任何数据。

.. note::

  如果 ``SystemX`` 是 ``root`` 命名空间的系统，情况要改善许多。此时，对于被调用的外部合约而言，
  ``msg.sender == address(World)``。虽然任何人都可以在你的 ``World`` 合约中
  注册任何命名空间和系统，但是只有 ``root`` 下的系统可以在 ``World`` 语境下发起对外调用。 而只有你
  能在 ``root`` 命名空间下注册系统，只要你没有转让 ``root`` 命名空间给其他人。


系统注册
--------------

系统需要在任意 ``World`` 合约完成注册，才能被使用。
系统注册包含两部分内容，注册系统合约，以及注册系统方法。

注册系统合约的目的在于确定系统所在的命名空间。
注册系统方法的目的是在 ``World`` 合约以后备函数的形式添加一个指定的函数选择器。随后可以使用注册的函数选择器调用
``World`` 合约， ``World`` 合约会自动将调用转发给对应的系统合约。

通过配置文件注册
^^^^^^^^^^^^^^^^^^

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";

  export default defineWorld({
    namespace: "muddoc",
    systems: {
      SimpleStorageSystem: {
        name: "SimpleStorage",
        openAccess: false,
        accessList: ["SimpleStorageCallerSystem", "0x0123456789012345678901234567890123456789"],
        deploy: {
          disabled: false,
          registerWorldFunctions: true,
        },
      },
      // SimpleStorageCallerSystem: {
      //   name: "SimpleStorageCal",
      //   openAccess: true,
      //   accessList: [],
      //   deploy: {
      //     disabled: false,
      //     registerWorldFunctions: true,
      //   },
      // },
    },
    tables: {...},
  });

这是一份适用于 :ref:`dev-differences` 中 ``SimpleStorageCallerSystem`` 和
``SimpleStorageSystem`` 系统的配置文件。两个系统将被注册在 ``muddoc`` 命名空间下。

先来看一下每个系统配置项的意义：

- ``name``: ``string``, 默认：带 ``System`` 后缀的系统名称的前 16 个字符。
  用于确定系统的 ``ResourceId``。系统的 ``ResourceId`` 用于在 ``World`` 中注册系统。
- ``openAccess``: ``bool``, 默认: ``true``。是否开放访问。如果为 ``true``，
  则任何地址都可以通过 ``World`` 合约调用该系统合约。如果为 ``false``，则可以通过 ``accessList`` 进行配置。

  .. note::

    当 ``openAccess`` 为 ``false`` 且 ``accessList`` 为空时，该系统合约只能被同命名空间内的系统
    或命名空间的所有者调用。

- ``accessList``: ``string[]``, 默认: 空数组。访问列表，既可以是项目内系统的全名，也可以是地址。
- ``deploy``: ``object``。部署配置。

  - ``disabled``: ``bool``, 默认: ``false``。 是否部署和注册该系统合约。
  - ``registerWorldFunctions``: ``bool``, 默认: ``true``。是否为所有对外的系统合约函数
    在 ``World`` 中注册相应的函数选择器。

    .. note::

      当系统处于 ``root`` 命名空间时， 注册的函数选择器与系统合约的函数选择器一致。

      当系统处于自定义命名空间时， 注册的函数选择器的函数名会用命名空间的名字做前缀。
      例如 ``IWorld(_world()).muddoc__getUint()``。

``Mud CLI`` 在部署/测试时会根据配置文件自动完成项目内所有系统的部署，并注册到刚刚部署的 ``World`` 合约。
如果一个系统没有需要特殊配置的地方，那么不需要在配置文件中为它做任何配置。**默认的配置项和数值将被自动运用到
未在配置文件中出现但确实存在于项目目录下的系统合约。**

.. note::
  自动化的默认系统配置要求系统合约所在文件名为 ``*System.sol``，置于 ``src`` 文件夹内，通常放置于 ``src/systems``。
  并且系统合约名称需与文件名（除格式后缀 ``.sol``）保持一致。

现在再来看一下上面的配置文件，我们对 ``SimpleStorageSystem`` 进行了重命名，这影响了它的 ``ResourceId``。
``0x73796d7564646f63000000000000000053696d706c6553746f72616765000000``，
其中 ``7379`` 是 ``sy`` 的十六进制编码， ``6d7564646f63`` 是 ``muddoc`` 的十六进制编码，
``53696d706c6553746f72616765`` 是 ``SimpleStorage`` 的十六进制编码。
我们关闭了 ``SimpleStorageSystem`` 的公开访问，只额外允许 ``SimpleStorageCallerSystem`` 和
``0x0123456789012345678901234567890123456789`` 经过 ``World`` 调用它。
我们正常启用了 ``SimpleStorageSystem`` 的部署，并且为所有对外的系统方法在 ``World`` 中注册了对应的函数选择器。
这允许有权限的地址使用 ``IWorld(worldAddress).muddoc__getUint`` 和
``IWorld(worldAddress).muddoc__setUint``。

.. note::

  因为 ``SimpleStorageCallerSystem`` 和 ``SimpleStorageSystem`` 在同一个命名空间 ``muddoc``,
  所以即使没有配置 ``accessList``， ``SimpleStorageCallerSystem`` 也可以调用 ``SimpleStorageSystem``。

对于 ``SimpleStorageCallerSystem``，我们没有在配置文件中配置，这意味着它将使用默认的配置项。
默认配置项与配置文件中被注释的配置项相同。系统的名称截取了 ``SimpleStorageCallerSystem`` 的前 16 个字符。
他的 ``ResourceId`` 是
``0x73796d7564646f63000000000000000053696d706c6553746f7261676543616c``，
不同的是最后 16 个字符， ``53696d706c6553746f7261676543616c`` 代表 ``SimpleStorageCal``。
默认配置开启了公开访问，不再需要额外的访问列表，启用了部署，并注册了所有对外的系统方法。

手动注册
^^^^^^^^^^^^

.. code-block:: solidity

  // SPDX-License-Identifier: MIT
  pragma solidity >=0.8.24;

  import { Script } from "forge-std/Script.sol";
  import { WorldResourceIdLib } from "@latticexyz/world/src/WorldResourceId.sol";
  import { System } from "@latticexyz/world/src/System.sol";
  import { ResourceId } from "@latticexyz/store/src/ResourceId.sol";

  import { IWorld } from "../src/codegen/world/IWorld.sol";

  contract ManuallyRegisterSystem is Script {
    // Load the private key from the `PRIVATE_KEY` environment variable (in .env)
    uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
    // Start broadcasting transactions from the deployer account
    vm.startBroadcast(deployerPrivateKey);

    // 如果注册的系统所在的命名空间不存在，应该先注册命名空间
    IWorld(worldAddress).registerNamespace({namespaceId: WorldResourceIdLib.encodeNamespace("muddoc")});
    // 部署 SimpleStorageSystem
    SimpleStorageSystem simpleStorageSystem = new SimpleStorageSystem();
    // 获取 SimpleStorageSystem的 资源 ID
    ResourceId simpleStorageSystemId = WorldResourceIdLib.encode("sy", "muddoc", "SimpleStorage");
    // 在指定的 world 地址注册 SimpleStorageSystem，并设置关闭公开访问
    IWorld(worldAddress).registerSystem({
      systemId: simpleStorageSystemId,
      system: simpleStorageSystem,
      publicAccess: false
    });
    // 为 setUint 注册函数选择器，对应的 World 函数为 muddoc__setUint(uint256)
    IWorld(worldAddress).registerFunctionSelector({
      systemId: simpleStorageSystemId,
      systemFunctionSignature: "setUint(uint256)"
    });
    // 为 getUint 注册函数选择器，对应的 World 函数为 muddoc__getUint()
    IWorld(worldAddress).registerFunctionSelector({
      systemId: simpleStorageSystemId,
      systemFunctionSignature: "getUint()"
    });
  }

这是一个手动部署和注册 ``SimpleStorageSystem`` 的脚本。 ``SimpleStorageSystem`` 归属于
``muddoc`` 命名空间。

如果我们希望在 ``root`` 命名空间注册 ``SimpleStorageSystem``，可以参照如下示例。不同的是，
``root`` 命名空间内的系统在注册系统方法时，可以自定义函数签名。

.. code-block:: solidity

  SimpleStorageSystem simpleStorageRootSystem = new SimpleStorageSystem();
  // root 命名空间的名称是空字符串
  ResourceId simpleStorageRootSystemId = WorldResourceIdLib.encode("sy", "", "SimpleStorage");
  IWorld(worldAddress).registerSystem({
    systemId: simpleStorageRootSystemId,
    system: simpleStorageRootSystem,
    publicAccess: false
  });
  // 为 root 命名空间的系统方法注册函数选择器时，可以指定函数签名
  IWorld(worldAddress).registerRootFunctionSelector({
    systemId: simpleStorageRootSystemId,
    worldFunctionSignature: "myRootSetUint(uint256)",
    systemFunctionSignature: "setUint(uint256)"
  });
  IWorld(worldAddress).registerRootFunctionSelector({
    systemId: simpleStorageRootSystemId,
    worldFunctionSignature: "myRootGetUint()",
    systemFunctionSignature: "getUint()"
  });

.. important::

  上述代码只是一个在 ``root`` 命名空间注册系统的示例，不代表我们可以如此手动更换系统所在的命名空间。

  我们建议通过配置文件完成命名空间的变更。因为它可以同时变更表和系统。当我们手动注册系统并更换命名空间时，
  很有可能忘记更新自动生成的表代码库，进而造成数据紊乱。

系统使用
--------------

这里系统使用指的是对于 ``World`` 合约以外的 EOA 或合约使用已注册的系统的方法。

.. note::

  与非 ``root`` 命名空间的系统调用其他系统的实现过程一样。

我们仍要重申 **World 是自主世界的统一入口**。对外而言，任何系统方法的调用都要经过 ``World`` 合约。

使用的方法分为两种，一种是通过系统的 ``SystemId`` 也就是 ``ResourceId``，将调用的 ``calldata`` 通过
``World`` 合约转发给系统合约。

.. code-block:: solidity

  ResourceId simpleStorageSystemId = WorldResourceIdLib.encode("sy", "muddoc", "SimpleStorage");
  uint256 res = abi.decode(
    IWorld(worldAddress).call(simpleStorageSystemId, abi.encodeWithSelector(SimpleStorageSystem.getUint.selector)),
    (uint256)
  );

另一种是通过系统注册的函数选择器，直接调用 ``World`` 合约。

.. code-block:: solidity

  uint256 res = IWorld(worldAddress).muddoc__getUint();

核心系统
--------------

todo

- ``AccessManagementSystem``
- ``BalanceTransferSystem``
- ``BatchCallSystem``
- ``ModuleInstallationSystem``
- ``StoreRegistrationSystem``
- ``WorldRegistrationSystem``
