.. _best-practices:

最佳实践
========

节约 gas
---------

选择更小的数据类型
^^^^^^^^^^^^^^^^^^

虽然Mud 能够帮我们在存储多个数据时，通过紧致编码来节省总体存储空间。
但当一个表字段的类型被定义为 ``uint256`` 时，它仍然至少需要一个 ``slot`` 的空间来存储。 
所以我们可以在保留未来扩展性的同时，尽量使用更小的数据类型来节省存储空间。

.. code-block:: ts

  Users: {
    schema: {
      id: "uint256",
      age: "uint256",
      weight: "uint256",
      height: "uint256",
    },
    key: ["id"],
  },

上面的 ``Users`` 表中，我们定义了三个 ``uint256`` 类型的字段，也就是每个用户需要占用 3 个 ``slot`` 的空间。
如果我们认为 ``age``, ``weight`` 和 ``height`` 这三个字段在绝大多数情况下，都不会超过 ``uint32`` 的范围:

.. code-block:: ts

  Users: {
    schema: {
      id: "uint256",
      age: "uint32",
      weight: "uint32",
      height: "uint32",
    },
    key: ["id"],
  },

我们可以将它们的类型定义为 ``uint32`` ，这样每个用户就只需要占用 1 个 ``slot`` 的空间。并且这一个 ``slot`` 的空间，
还没有被填满，我们还可以在未来，向 ``Users`` 表中添加更多的字段，且不会显著影响原有字段的读写成本。

尽可能地一次性读写表的全部字段
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果一个表包含至少两个字段，那么自动生成的代码库中，不仅会生成 ``get()`` 方法，还会生成 ``get<Fieldname>()`` 方法。
同理， ``set()`` 方法也会生成 ``set<Fieldname>()`` 方法。

在一次系统调用中，如果需要读写同一张表的多个字段，尽可能使用 ``get()`` 和 ``set()`` 方法，相比于连续使用
``get<Fieldname>()`` 和 ``set<Fieldname>()`` 方法，更节省 gas。

.. code-block:: ts

  T: {
    schema: {
      id: "uint256",
      F1: "uint32",
      F2: "uint32",
    },
    key: ["id"],
  },

  // gasCostOf(T.getF1() + T.getF2()) > gasCostOf(T.get())
  // gasCostOf(T.setF1() + T.setF2()) > gasCostOf(T.set())

.. note::

  同一个表中需要读写的字段越多，使用 ``get()`` 和 ``set()`` 一次性读写所有字段，节省 gas 越多。

.. important::

  如果只需要读写表的一个字段，使用 ``get<Fieldname>()`` 和 ``set<Fieldname>()`` 方法更节约 gas。

根据字段的使用频率设计表结构
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果你有一个对象，它有非常多的属性， 例如 10 个，选择把每个属性都定义成一个表还是把它们合并成一个表将是一个需要权衡的问题。

显然，每个属性都定义成一个表，会使得表结构更加清晰，易于理解。但每个表都至少占用一个 ``slot`` 的空间，也会显著增加读写成本。

而一概而论的将所有字段都合并成一个表，虽然节省了存储空间，但不一定节省读写成本，反而让表结构变得臃肿，难以理解和维护。

如果从节省 gas 的角度出发，我们可以根据字段的使用频率将它们分散到不同的表中，并通过尽可能多地使用 ``get()`` 和 ``set()`` 方法，
来减少每次系统调用中的表读写成本。此外，这种分类方式往往与按照字段内在含义进行分类的结果相契合，不会额外增加表的理解成本。
例如一个房子的长、宽、高属性，都属于房子的固有属性，且在绝大多数情况下，它们都会同时被使用。

.. note::

  如何归纳整理字段并设计表结构，需要结合具体的业务场景，没有统一的标准。按照字段的使用频率分类，是一种更契合
  节省 gas 的设计思路。

.. important::

  如果一个字段的类型是 :ref:`引用类型<field-supported-types>` ，它更适合定义成一个单独的表。

  如果有其他字段，无论是数值类型还是引用类型，总是跟它一起被使用，那么它们适合定义在同一个表中。

使用 ``IWorld.call()`` 调用系统合约
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们通常习惯于使用 CLI 生成的自主世界接口，来调用系统合约。例如：

.. code-block:: solidity

  // SpawnSystem 是一个 root 命名空间下的系统合约，并提供 spawn() 方法
  //   调用来自 world 外部，或者 non-root system 合约
  IWorld(worldAddress).spawn();
  //   调用来自 world 内部，例如 root system 合约
  worldAddress.delegatecall(abi.encodeCall(IWorld(worldAddress).spawn, ()));

  // ListSystem 是一个 muddoc 命名空间下的系统合约，并提供 list() 方法
  IWorld(worldAddress).mudddoc_list();

这种调用方式简单、清晰。

如果你希望更多地节省 gas，也可以使用 ``IWorld.call()`` 方法来调用系统合约。
这种方式通过显式地指定系统资源 ID 和方法调用参数，来避免通过已注册的自主世界函数选择器查找对应的系统资源和系统函数选择器，
从而节省 gas。例如：

.. code-block:: ts

  // SpawnSystem 是一个 root 命名空间下的系统合约，并提供 spawn() 方法
  //   调用来自 world 外部，或者 non-root system 合约
  IWorld(worldAddress).call(
    WorldResourceIdLib.encode("sy", "", "SpawnSystem"),
    abi.encodeCall(SpawnSystem.spawn, ())
  );
  //   调用来自 world 内部，例如 root system 合约
  worldAddress.delegatecall(
    abi.encodeCall(
      IWorld(worldAddress).call,
      (
        WorldResourceIdLib.encode("sy", "", "SpawnSystem"),
        abi.encodeCall(SpawnSystem.spawn, ())
      )
    )
  );

  // ListSystem 是一个 muddoc 命名空间下的系统合约，并提供 list() 方法
  IWorld(worldAddress).call(
    WorldResourceIdLib.encode("sy", "muddoc", "ListSystem"),
    abi.encodeCall(ListSystem.list, ())
  );
