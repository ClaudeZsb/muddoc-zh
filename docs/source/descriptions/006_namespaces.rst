命名空间
==========

命名空间用于划分和管理自主世界中注册的所有系统和表资源，是系统和表构成的资源子集。
命名空间与自主世界中的权限控制息息相关。

创建一个命名空间
----------------

我们通常使用配置文件 ``mud.config.ts`` 来完成命名空间的创建。

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";

  export default defineWorld({
    namespace: "muddoc",
    tables: ...
    systems: ...
  });

上面的配置文件内容定义了一个名为 ``muddoc`` 的命名空间，并且在 ``tables`` 和 ``systems`` 中
定义的所有表和系统都将注册在该命名空间下。

手动创建
^^^^^^^^^^^^^

通常我们会在模组的使用中需要手动创建一个命名空间。

.. code-block:: solidity

  IWorld(worldAddress).registerNamespace({namespaceId: WorldResourceIdLib.encodeNamespace("muddoc")});

创建多个命名空间
----------------

从 Mud ``2.1.0`` 开始，我们可以使用一个配置文件创建多个命名空间。

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";

  export default defineWorld({
    namespaces: {
      muddoc: {
        tables: ...
        systems: ...
      },
      fun: {
        tables: ...
        systems: ...
      }
    },
    // namespace: ... ❌
    // tables: ... ❌
    // systems: ... ❌
  });

当我们使用 ``namespaces`` 时，意味着我们开启了多命名空间模式，此时 ``namespace`` 字段将被禁用。
同样被禁用的还有 ``tables`` 和 ``systems`` 字段，这两个字段需要在 ``namespaces``
中的每个单独的命名空间下使用。 ``namespaces`` 中的每个键值对都定义了一个命名空间。
键将作为命名空间的名称，值是一个包含 ``tables`` 和 ``systems`` 的对象。

.. note::

  在多命名空间模式下， ``enums`` 和 ``userTypes`` 的用法不变。

使用多命名空间模式还要求我们的项目目录按照命名空间划分，例如：

.. code-block:: bash

  src
  ├── codegen
  │   └── world
  └── namespaces
      ├── fun
      │   ├── codegen
      │   │   └── tables
      │   └── systems
      └── app
          ├── codegen
          │   └── tables
          └── systems

系统的存放位置建议为 ``src/namespaces/<namesapce-name>/systems/``。

访问权限控制
---------------

在没有任何访问权限配置的情况下，对所有地址而言，都拥有一些基本访问权限，包括：

- 读取所有的表
- 使用所有开启公开访问的系统

而对于已经注册的系统而言，在基本访问权限之上它们还可以

- 更新所有同命名空间内的表
- 使用所有同命名空间内的系统，即使它们没有开启公开访问

除以上设计的权利以外，任何额外的资源访问都需要经过命名空间拥有者的授权。

访问权限检查的基本过程是：

1. （如果访问对象是系统）检查系统是否开启公开访问。
2. 检查访问者是否被授予访问对象所在的命名空间的访问权限。

  .. note::

    如果访问者具有命名空间的访问权限，那么访问者可以访问命名空间内的所有系统和表。
    无需对每个系统和表进行单独授权。

3. 检查访问者是否被授予访问对象的访问权限。

访问权限管理
^^^^^^^^^^^^^^

访问权限管理包括授予和撤销权限。访问权限管理的操作对象是资源，包括表、系统和命名空间。
访问权限管理的操作人必须为操作对象所属命名空间的所有者。

.. code-block:: solidity

  // 表
  ResourceId tableId = WorldResourceIdLib.encode("tb", "muddoc", "Table1");
  // 系统
  ResourceId systemId = WorldResourceIdLib.encode("sy", "muddoc", "System1");
  // 命名空间
  ResourceId namespaceId = WorldResourceIdLib.encodeNamespace("muddoc");

  // 授予地址访问资源的权限
  IWorld(worldAddress).grantAccess({
    resourceId: specificResourceId,
    grantee: granteeAddress
  });

  // 撤销地址访问资源的权限
  IWorld(worldAddress).revokeAccess({
    resourceId: specificResourceId,
    grantee: granteeAddress
  });

.. note::

  授予命名空间的访问权限等同于授予命名空间内所有资源的访问权限。

.. important::

  如果某个地址被授予单独的系统或表的访问权限，撤销它对命名空间的访问权限不会影响它对单独资源的访问权限。

  这种情况下，如果你想禁止他对所有内部资源的访问，你需要逐个对曾经单独授权的资源进行权限撤销。

管理命名空间所有权
^^^^^^^^^^^^^^^^^^^^^

命名空间的初始所有者是命名空间的创建者。命名空间所有者可以转移或放弃所有权。

.. code-block:: solidity

  // 转移命名空间所有权
  IWorld(worldAddress).transferNamespaceOwnership({
    namespaceId: WorldResourceIdLib.encodeNamespace("muddoc"),
    newOwner: newOwnerAddress
  });

  // 放弃命名空间所有权
  IWorld(worldAddress).renounceOwnership({
    namespaceId: WorldResourceIdLib.encodeNamespace("muddoc")
  });

.. note::

  命名空间的在创建时，被授予命名空间的访问权限。

  当所有权发生变更时，原所有者将撤销自己对命名空间的访问权限，新所有者将被授予命名空间的访问权限。

.. important::

  如果原所有者曾授予自己对单独的系统或表资源的访问权限，变更所有权并不会让他失去这些
  资源的访问权限。如果新所有者希望禁止这些访问，应该手动撤销这些权限。

