开发智能合约的区别
======================================

读写合约状态
----------------

这是一个简单的存储并读取一个数字的例子。

.. code-block:: solidity

  // SPDX-License-Identifier: GPL-3.0
  pragma solidity >=0.4.16 <0.9.0;

  contract SimpleStorage {
    uint storedUint;

    function setUint(uint x) public {
      storedUint = x;
    }

    function getUint() public view returns (uint) {
      return storedUint;
    }
  }

使用Mud框架，你可以这样写：

.. code-block:: solidity

  // SPDX-License-Identifier: MIT
  pragma solidity >=0.8.24;

  import { System } from "@latticexyz/world/src/System.sol";
  import { StoredUint } from "../codegen/index.sol";

  contract SimpleStorageSystem is System {
    function setUint(uint x) public {
      StoredUint.set(x);
    }

    function getUint() public view returns (uint) {
      return StoredUint.get();
    }
  }

你可能会好奇 ``StoredUint`` 是在哪里被声明的，又是如何被存储和读取的?

这就是Mud框架的重要特性之一，它把合约状态封装成了人们熟知的数据库，你可以像操作库表一样声明、读取和更新合约状态。
为了达到上面的效果，你只需要搭配一个这样的“库表”配置文件即可：

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";

  export default defineWorld({
    namespace: "app",
    tables: {
      StoredUint: {
        schema:{
          value: "uint256",
        },
        key: [],
      },
    },
  });

看上去如果只是存储和读取一个 ``uint`` ，似乎用原生语言更简单和直接。
是的，即使是 ``Array`` 或 ``Mapping`` 类型的变量，在单个合约的使用场景下，Mud只能画蛇添足。
**这也是我们不推荐在简单合约中使用Mud的原因之一。但我们仍鼓励大家尝试使用Mud。**
因为我们经常会需要复杂的数据结构和频繁的合约交互，那就需要为数据结构定义易用的方法，为其他合约定义数据接口。所有的这些都要自己手动去写，还要做好管理，真的是一件耗时伤神的事情。

那么Mud，它到底能为数据读写带来什么好处？

* 自动化的生成数据接口, 以 ``library`` 形式存在，随用随取。
* 相比于原生 ``Solidity`` 更紧致的数据打包，相比于 ``Diamon`` 更方便的管理和维护内部状态。
* 通过一组通用的 ``event`` 集合实现链下标准化、自动化的数据索引，不再需要创建数据更新的 ``event`` 和相应的监听器。

.. note::

  目前，合约无法像查看自己的 ``slot`` 一般查看其他合约的 ``slot`` ，只能依赖开放的数据接口。然而任何链上数据对于任何链下程序都是透明的。

* 开放式、标准化的数据读取接口，方便了任何外部合约读取合约的任意数据。

跨合约调用
------------

这是一个简单的调用 ``SimpleStorage`` 合约，获取存储的数字的例子。

.. code-block:: solidity

  // SPDX-License-Identifier: GPL-3.0
  pragma solidity >=0.4.16 <0.9.0;

  contract SimpleStorageCaller {
    SimpleStorage simpleStorage;

    constructor(address _simpleStorage) {
      simpleStorage = SimpleStorage(_simpleStorage);
    }

    function setUintToSimpleStorage(uint x) public {
      simpleStorage.setUint(x);
    }

    function getUintFromSimpleStorage() public view returns (uint) {
      return simpleStorage.getUint();
    }
  }

使用Mud框架，你应该这样写：

.. note::

  ``SimpleStorageSystem`` 和 ``SimpleStorageCallerSystem`` 是两个不同的合约，
  代表同一个自主世界中的两个不同系统。最重要的是，它们不在 ``root`` 命名空间中。
  因为我们使用不同的方式来调用 ``root`` 命名空间中的系统方法。后面会详细介绍。

.. code-block:: solidity

  // SPDX-License-Identifier: MIT
  pragma solidity >=0.8.24;

  import { System } from "@latticexyz/world/src/System.sol";
  import { IWorld } from "../codegen/world/IWorld.sol";

  contract SimpleStorageCallerSystem is System {
    function setUintToSimpleStorageSystem(uint x) public {
      IWorld(_world()).app__setUint(x);
    }

    function getUintFromSimpleStorageSystem() public view returns (uint) {
      return IWorld(_world()).app__getUint();
    }
  }

这里的 ``IWorld`` 是一个自动生成的接口，它包含了整个项目中所有的 ``System`` 合约对外接口。
当我们把 ``SimpleStorageSystem`` 和 ``SimpleStorageCallerSystem`` 放在一个项目内，就会自动获得一个包含了两个合约对外方法的接口集合，我们也就自然可以通过这个接口集合调用 ``SimpleStorageSystem`` 的 ``setUint`` 和 ``getUint`` 方法。

.. note::

  ``_world()`` 是一个由 ``System`` 引入的内部函数，用于获取项目 ``World`` 地址，你可以暂时把它理解成项目的主合约，所有的方法入口都建立在主合约上。更多的信息请参考后面的章节。

从范例中可以发现，我们使用Mud框架后并没有手动建立两个合约的关联关系，
而去调用了一个被称为 ``World`` 的主合约，就完成了对 ``StoredUint`` 的读取。
而这个主合约实际上也并非  ``SimpleStorageSystem`` 合约本身。

.. note::

  ``World`` 合约跟 ``Diamond`` 合约实际上还有很多区别，但我们后面再细讲。

如果你了解Diamond协议，就不难发现 ``World`` 合约十分类似 ``Diamond`` 合约。
其实它们都有一个集中的数据存储合约，所有的业务逻辑 ``System`` 或 ``Facet`` 合约都以链上代码库的形式存在，
它们不实际存储数据，而是通过 ``delegateCall`` 或 ``call`` 的形式与管理数据的合约进行连接，以此完成对数据的操作。

有人可能会问，如果 ``World`` 合约类似 ``Diamond`` ，所有的数据都是集中存储的，
为什么不让 ``SimpleStorageCallerSystem`` 直接读写 ``StoredUint`` 呢，反而要走合约交互的方式？

确实，如果跨合约交互的需求只是读写一个具体的数据的话，也可以这么写：

.. code-block:: solidity

  // SPDX-License-Identifier: MIT
  pragma solidity >=0.8.24;

  import { System } from "@latticexyz/world/src/System.sol";
  import { StoredUint } from "../codegen/index.sol";

  contract SimpleStorageCallerSystem is System {
    function setUint2(uint x) public {
      StoredUint.set(x);
    }

    function getUint2() public view returns (uint) {
      return StoredUint.get();
    }
  }

.. important::

  即使合约交互逻辑简单到只是修改一个状态，也不一定能直接对状态所在的表进行操作。Mud有一套严格的权限控制机制。
  如果你的项目只有一个自定义的命名空间，比如 ``app``，且所有的表和系统都隶属它，那么上面的转换就是可行的。否则，仍需具体情况具体分析。

.. note::

  这里我们只是希望通过这个极简的例子，来表现库表资源是在一定范围内共享的，不需要专门写相关的交互方法。在真实应用场景中，每一个系统方法都应该是认真设计的，并尽可能地复用。
