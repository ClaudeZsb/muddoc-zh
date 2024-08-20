表
======

**自主世界中任何需要基于合约状态实现持久化存储的数据都通过表实现。**

**使用配置文件来定义和创建表，使用自动生成的表代码库来操作表。**

定义一个表
----------

这是一份简单的配置文件，在 ``muddoc`` 命名空间内定义了一个名为 ``Users`` 的表，
它有三个字段 ``addr`` 、 ``data`` 和 ``description`` ，
类型分别为 ``address`` 、 ``uint256`` 和 ``string`` ， 其中 ``addr`` 字段是主键。

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";

  export default defineWorld({
    namespace: "muddoc",
    tables: {
      Users: {
        schema: {
          addr: "address",
          data: "uint256",
          description: "string",
        },
        key: ["addr"],
      },
    }
  });

在配置文件 ``mud.config.ts`` 中，每一个 ``tables`` 中的键值对都将定义一个表。
键值对的键确定了表的名称，值是一个对象，包含了表的其他配置项，主要包含表字段和主键的定义。
所有的表都属于 ``namespace`` 字段定义的命名空间。

在 Mud 定义的自主世界中，表是一种资源，每个资源都由一个唯一的 ``ResourceId`` 标识。
表的 ``ResourceId`` 被称为 ``tableId``。例子中 ``Users`` 的 ``tableId`` 是
``0x74626d7564646f63000000000000000055736572730000000000000000000000`` 。
其中 ``7462`` 是 ``tb`` 的十六进制形式，`在线转换工具 <https://www.rapidtables.com/convert/number/ascii-to-hex.html>`_ 。
``6d7564646f63`` 代表 ``muddoc``， ``5573657273`` 代表 ``Users`` 。

表的配置项如下：

- ``schema``: ``object`` ，表的字段定义，是字段名和类型的键值对。

  .. important::

    每张表除主键字段外，最多拥有 **28** 个字段，其中至多 **5** 个 :ref:`引用类型 <field-supported-types>` 字段。

    **在编写定义时，引用类型字段必须放在最后。**

- ``key``: ``string[]`` ，表的主键，可以是一个或多个字段的数组。也可以是一个空数组，意味着这是一个单例表。

  .. important::

    主键字段的类型必须是 :ref:`数值类型 <field-supported-types>`。主键字段的个数主要受 EVM 的栈深度限制。
    过多的主键字段将导致表的读写方法不可用。

- ``type``: ``string`` （可选），表的类型。``table`` （默认值，存储在链上的表） 或
  ``offchainTable`` （表的数据只能通过 ``event`` 在链下获取）.
- ``codegen``: ``object`` （可选）。

  - ``outputDirectory``: ``string``，默认: ``"tables"`` 。代码生成的输出目录，
    默认放在配置文件目录下的 ``src/codegen/tables`` 。
  - ``tableIdArgument``: ``boolean``，默认: ``false`` 。是否为读写方法生成 ``tableId`` 参数。
  - ``storeArgument``: ``boolean``，默认: ``false`` 。是否为读写方法生成 ``store`` 参数。

  .. note::

    当同一种表（字段和主键定义相同）存在于多个命名空间，或以不同名字存在于同一个命名空间时，
    可以通过调整 ``tableIdArgument`` ，让自动生成的代码库根据传入的 ``tableId`` 来操作不同的表。

    当这种情形扩展到不同的自主世界时，可以进一步调整 ``storeArgument`` 引入 ``store`` 参数。

  - ``dataStruct``: ``boolean``，当存在超过一个不是主键组成部分的字段时，默认为 ``true`` 。
    是否为表的非主键字段生成数据结构，默认 ``struct <表名>Data`` 。
- ``deploy``: ``object`` （可选）。

  - ``disabled``: ``boolean``，默认: ``false`` 。是否创建该表。


.. _field-supported-types:

表字段支持的类型
^^^^^^^^^^^^^^^^^^^^^^

+--------------+-----------------------------------------------------------+
| 类型         |                                                           |
+==============+===========================================================+
|| 数值类型    || ``uint8`` ~ ``uint256``, ``int8`` ~ ``int256``,          |
||             || ``address``, ``bool``, ``bytes1`` ~ ``bytes32``          |
+--------------+-----------------------------------------------------------+
| 引用类型     | 数值类型构成的定长数组或动态数组， ``string`` , ``bytes`` |
+--------------+-----------------------------------------------------------+
| 枚举         | ✅                                                        |
+--------------+-----------------------------------------------------------+
| 自定义类型   | ✅                                                        |
+--------------+-----------------------------------------------------------+
| ``mapping``  | ❌                                                        |
+--------------+-----------------------------------------------------------+
| ``string[]`` | ❌                                                        |
+--------------+-----------------------------------------------------------+
| ``bytes[]``  | ❌                                                        |
+--------------+-----------------------------------------------------------+
| ``struct``   | ❌                                                        |
+--------------+-----------------------------------------------------------+


.. important::

  并不是 Mud 框架不能读写 ``mapping``, ``string[]``, ``bytes[]``, ``struct`` 类型的数据，
  而是这些类型的数据不需要以表字段的形式存在。

  如果我们想要实现 ``mapping(uint256 => address)`` 类型，可以创建一个有两个字段的表，
  两个字段类型分别是 ``uint256`` 和 ``address`` ，并将 ``uint256`` 字段设为主键。

  如果我们想要实现 ``string[], bytes[]`` 类型，可以创建一个有两个字段的表，
  两个字段类型分别是 ``uint256`` , ``string`` 或 ``bytes``, 并将 ``uint256`` 字段设为主键， 意为数组的索引。

  每一个单例表中的唯一一行都可以看作一个类型为 ``struct`` 的数据。

枚举
""""""""""""

在配置文件中我们可以定义枚举，并在表的字段中使用定义的枚举。

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";

  export default defineWorld({
    namespace: "muddoc",
    enums: {
      UserStatus: ["active", "inactive"],
    },
    tables: {
      UserStates: {
        schema: {
          addr: "address",
          status: "UserStatus",
        },
        key: ["addr"],
      },
    }
  });

每一个 ``enums`` 中的键值对都将定义一个枚举。
键值对的键确定了枚举的名称，值是一个包含所有枚举成员名称的字符串数组。

所有枚举类型由 ``CLI: mud tablegen`` 统一生成和存放于 ``src/codegen/common.sol``。

自定义类型
""""""""""""

在配置文件中我们可以通过文件路径引入自定义类型，并在表的字段中使用这些引入的自定义类型。

自定义类型需要事先准备， ``CLI: mud tablegen`` 根据配置文件中的引入路径自动为表代码库生成对应的引入。

这些自定义类型既可以来自本项目也可以来自于三方库。

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";

  export default defineWorld({
    namespace: "muddoc",
    userTypes: {
      MyUint256: {
        type: "uint256",
        filePath: "./src/utils/MyUint256s.sol",
      },
      ShortString: {
        type: "bytes32",
        filePath: "@openzeppelin/contracts/utils/ShortStrings.sol",
      }
    },
    tables: {
      UserStates: {
        schema: {
          addr: "address",
          data: "MyUint256",
          label: "ShortString",
        },
        key: ["addr"],
      },
    }
  });

``./src/utils/MyUint256s.sol`` 是对于配置文件而言的相对路径，其内容大致如下。

.. code-block:: solidity

  // SPDX-License-Identifier: MIT
  pragma solidity >=0.8.24;

  type MyUint256 is uint256;

  library MyUint256s {
    // MyUint256 utils
  }

表定义的简写
^^^^^^^^^^^^^^^^^^^^^^

为方便定义只有一个字段或无需额外配置的表，可以使用如下的几种简写方式，
其中 ``T*`` 是表定义的简写，相应的 ``Table*`` 是与之等价的完整的表定义。

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";

  export default defineWorld({
    namespace: "muddoc",
    tables: {
      T1: "address",
      T2: "uint256[]",
      T3: "uint8[10]",
      T4: {
        id: "address",
        value: "uint256",
        data: "string",
      },
      Table1: {
        schema: {
          id: "bytes32",
          value: "address",
        },
        key: ["id"],
      },
      Table2: {
        schema: {
          id: "bytes32",
          value: "uint256[]",
        },
        key: ["id"],
      },
      Table3: {
        schema: {
          id: "bytes32",
          value: "uint8[10]",
        },
        key: ["id"],
      },
      Table4: {
        schema: {
          id: "address",
          value: "uint256",
          data: "string",
        },
        key: ["id"],
      },
    }
  });


表的使用
----------

表的主要操作包括创建（注册）、读取、更新和删除。
所有的操作依赖于 ``CLI: mud tablegen`` 根据表的定义所生成的代码库。
每张表的代码库都是一个单独的 ``solidity library``，并以表名命名，它包含 ``tableId``，表结构和 CRUD 方法，

只需要将表的代码库引入到合约中，就可以直接调用 CRUD 方法。

.. code-block:: solidity

  // SPDX-License-Identifier: MIT
  pragma solidity >=0.8.24;

  import { System } from "@latticexyz/world/src/System.sol";
  import { Users } from "../codegen/index.sol";

  contract TableOperationSystem is System {
    function CRUD() public {
      Users.register(); // Don't do this. It's just for demonstration purposes.
      (uint256 data, string memory description) = Users.get(address(0));
      Users.set(address(0), 1 /* data */, "address zero" /* description */);
      Users.deleteRecord(address(0));
    }
  }

- ``register()``, 将表注册到自主世界中。一次性操作。需要所属的命名空间所有权。

  .. note::

    通过配置文件定义的表，在部署时会自动完成创建，无需人工操作。

  .. note::

    ``register()`` 一般在模组中使用，将表注册到模组所在的自主世界中。

- ``get()``， ``set``，整行地读写数据，表定义中的 ``codegen.dataStruct`` 配置项将影响
  ``get()`` 的返回结果类型。
- ``get<Fieldname>()``， ``set<Fieldname>``, 读写一条数据的一个字段。
- ``getItem<Fieldname>`` 按索引读取一个引用类型字段的元素。
- ``update<Fieldname>``，按索引更新一个引用类型字段的元素。
- ``length<Fieldname>``，获取一个引用类型字段的长度，不支持定长数组如 ``uint8[4]``。
- ``push<Fieldname>``， ``pop<Fieldname>``，向一个引用类型字段末尾添加或删除一个元素，不支持定长数组。

内部 CRUD 方法
^^^^^^^^^^^^^^^^^^^^^^

当你仔细观察一个表的代码库时，你会发现每一个 CRUD 方法都伴随一个相似的但名字不同的方法。这些方法以 ``_``
开头，如 ``_register()`` ，按照习惯，它们代表了内部方法。但代码库中的所有方法都带有 ``internal`` 修饰词。
**这里内部方法指这些方法相较于上面提及的方法而言，仅能在自主世界主合约的语境下使用。**

.. note::

  这些内部方法可以在 ``root`` 命名空间下的系统中使用。
  如果你的项目使用了自定义的命名空间，请不要使用这些内部方法。
  但你无需担心项目数据的安全，使用这些内部方法只会产生错误或没有产生预期的效果，不会对项目数据造成损害。

带 ``tableId`` 参数的 CRUD 方法
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在某些情况下，我们需要通过 ``tableId`` 参数来区分操作的表。
在配置文件中，为需要的表定义加入 ``codegen.tableIdArgument`` 配置项，可以为所有 CRUD 方法引入
``tableId`` 参数。

带 ``store`` 参数的 CRUD 方法
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在某些情况下，我们需要通过 ``store`` 参数来指定操作的表所处的自主世界。
在配置文件中，为需要的表定义加入 ``codegen.storeArgument`` 配置项，
可以在代码库中额外生成一套引入 ``store`` 参数的 CRUD 方法，这些方法具有相同的命名且不带 ``_`` 前缀。
