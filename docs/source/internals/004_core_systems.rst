.. _internals_core_systems:

核心系统
============

我们在开发或使用自主世界时，接触到的绝大多数功能均由 Mud 框架内的核心系统实现，它们的主要功能如下：

- ``AccessManagementSystem``, 访问管理

  - :ref:`grantAccess <access-management-system-grant-access>`
  - :ref:`revokeAccess <access-management-system-revoke-access>`
  - :ref:`transferOwnership <access-management-system-transfer-ownership>`
  - :ref:`renounceOwnership <access-management-system-renounce-ownership>`

- ``BalanceTransferSystem``, 余额转移

  - :ref:`transferBalanceToNamespace <balance-transfer-system-transfer-balance-to-namespace>`
  - :ref:`transferBalanceToAddress <balance-transfer-system-transfer-balance-to-address>`

- ``BatchCallSystem``, 批量调用

  - :ref:`batchCall <batch-call-system-batch-call>`
  - :ref:`batchCallFrom <batch-call-system-batch-call-from>`

- ``ModuleInstallationSystem``, 模组安装

  - :ref:`installModule <module-installation-system-install-module>`

- ``StoreRegistrationSystem``, 表、表钩子注册

  - :ref:`registerTable <store-registration-system-register-table>`
  - :ref:`registerStoreHook <store-registration-system-register-store-hook>`
  - :ref:`unregisterStoreHook <store-registration-system-unregister-store-hook>`

- ``WorldRegistrationSystem``, 系统、系统钩子、委托注册

  - :ref:`registerNamespace <world-registration-system-register-namespace>`
  - :ref:`registerSystemHook <world-registration-system-register-system-hook>`
  - :ref:`unregisterSystemHook <world-registration-system-unregister-system-hook>`
  - :ref:`registerSystem <world-registration-system-register-system>`
  - :ref:`registerFunctionSelector <world-registration-system-register-function-selector>`
  - :ref:`registerRootFunctionSelector <world-registration-system-register-root-function-selector>`
  - :ref:`registerDelegation <world-registration-system-register-delegation>`
  - :ref:`unregisterDelegation <world-registration-system-unregister-delegation>`
  - :ref:`registerNamespaceDelegation <world-registration-system-register-namespace-delegation>`
  - :ref:`unregisterNamespaceDelegation <world-registration-system-unregister-namespace-delegation>`

访问管理系统 ``AccessManagementSystem``
---------------------------------------------

访问管理系统负责管理资源的访问权限。它提供了以下主要功能：

.. _access-management-system-grant-access:

``grantAccess``
^^^^^^^^^^^^^^^^

授予指定地址访问某个资源的权限。

.. code-block:: solidity

  function grantAccess(ResourceId resourceId, address grantee)

- 要求调用者拥有该资源所在的命名空间
- ``resourceId``: 要授权的资源ID
- ``grantee``: 被授权的地址

.. _access-management-system-revoke-access:

``revokeAccess``
^^^^^^^^^^^^^^^^

撤销指定地址访问某个资源的权限。

.. code-block:: solidity

  function revokeAccess(ResourceId resourceId, address grantee)

- 要求调用者拥有该资源所在的命名空间
- ``resourceId``: 要撤销权限的资源ID  
- ``grantee``: 被撤销权限的地址

.. _access-management-system-transfer-ownership:

``transferOwnership``
^^^^^^^^^^^^^^^^^^^^^^

转移命名空间的所有权给新的所有者。

.. code-block:: solidity

  function transferOwnership(ResourceId namespaceId, address newOwner)

- 要求调用者拥有该命名空间
- ``namespaceId``: 要转移所有权的命名空间ID
- ``newOwner``: 新的所有者地址
- 会撤销原所有者的访问权限,并授予新所有者访问权限

.. _access-management-system-renounce-ownership:

``renounceOwnership``
^^^^^^^^^^^^^^^^^^^^^^

放弃命名空间的所有权。

.. code-block:: solidity

  function renounceOwnership(ResourceId namespaceId)

- 要求调用者拥有该命名空间
- ``namespaceId``: 要放弃所有权的命名空间ID
- 会删除命名空间所有者记录并撤销原所有者的访问权限

.. important::

  如果原所有者曾授予自己对单独的系统或表资源的访问权限，变更所有权并不会让他失去这些
  资源的访问权限。如果新所有者希望禁止这些访问，应该手动撤销这些权限。

余额转移系统 ``BalanceTransferSystem``
-----------------------------------------

余额转移系统用于在命名空间之间或从命名空间到外部地址转移 ETH 余额。

.. _balance-transfer-system-transfer-balance-to-namespace:

``transferBalanceToNamespace``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在命名空间之间转移余额。

.. code-block:: solidity

  function transferBalanceToNamespace(ResourceId fromNamespaceId, ResourceId toNamespaceId, uint256 amount)

- 要求调用者拥有源命名空间的访问权限
- ``fromNamespaceId``: 源命名空间ID
- ``toNamespaceId``: 目标命名空间ID
- ``amount``: 转移金额
- 源命名空间必须有足够的余额
- 目标命名空间必须存在

.. _balance-transfer-system-transfer-balance-to-address:

``transferBalanceToAddress``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

从命名空间转移余额到外部地址。

.. code-block:: solidity

  function transferBalanceToAddress(ResourceId fromNamespaceId, address toAddress, uint256 amount)

- 要求调用者拥有源命名空间的访问权限
- ``fromNamespaceId``: 源命名空间ID
- ``toAddress``: 目标地址
- ``amount``: 转移金额
- 源命名空间必须有足够的余额
- 如果转移失败会回滚交易

批量调用系统 ``BatchCallSystem``
--------------------------------

批量调用系统用于在单个交易中执行多个系统调用。

.. _batch-call-system-batch-call:

``batchCall``
^^^^^^^^^^^^^

批量调用多个系统。

.. code-block:: solidity

  function batchCall(SystemCallData[] calldata systemCalls) returns (bytes[] memory returnDatas)

- ``systemCalls``: 系统调用数据数组，每个元素包含 ``systemId`` 和 ``callData``
- ``returnDatas``: 返回每个系统调用的返回数据
- 如果任何一个调用失败，整个交易会回滚

.. _batch-call-system-batch-call-from:

``batchCallFrom``
^^^^^^^^^^^^^^^^^

以指定地址身份批量调用多个系统。

.. code-block:: solidity

  function batchCallFrom(SystemCallFromData[] calldata systemCalls) returns (bytes[] memory returnDatas)

- ``systemCalls``: 系统调用数据数组，每个元素包含 ``from``、``systemId`` 和 ``callData``
- ``returnDatas``: 返回每个系统调用的返回数据
- 如果任何一个调用失败，整个交易会回滚

模组安装系统 ``ModuleInstallationSystem``
-----------------------------------------

模组安装系统负责处理自主世界中非根模组的安装。它仅提供一个功能：

.. _module-installation-system-install-module:

``installModule``
^^^^^^^^^^^^^^^^^

安装一个非根模组到自主世界的指定命名空间。

.. code-block:: solidity

  function installModule(IModule module, bytes memory encodedArgs)

- ``module``: 要安装的模组
- ``encodedArgs``: 模组安装所需的编码参数
- 要求模组实现 ``IModule`` 接口
- 安装成功后会在 ``InstalledModules`` 表中记录该模组

.. important::

  使用模组安装系统无法安装根模组。安装根模组的功能并不由模组安装系统实现，而是由自主世界的主合约 ``World`` 实现。

存储注册系统 ``StoreRegistrationSystem``
-----------------------------------------

存储注册系统负责管理自主世界中的表和表钩子。它提供了以下主要功能：

.. _store-registration-system-register-table:

``registerTable``
^^^^^^^^^^^^^^^^^

注册一个表到自主世界。

.. code-block:: solidity

  function registerTable(
    ResourceId tableId,
    FieldLayout fieldLayout, 
    Schema keySchema,
    Schema valueSchema,
    string[] keyNames,
    string[] fieldNames
  )

- ``tableId``: 表的资源ID
- ``fieldLayout``: 表的字段布局
- ``keySchema``: 表的键模式
- ``valueSchema``: 表的值模式
- ``keyNames``: 表的键名称数组
- ``fieldNames``: 表的字段名称数组
- 要求调用者拥有该表所在的命名空间
- 要求表名不能为空字符串

.. _store-registration-system-register-store-hook:

``registerStoreHook``
^^^^^^^^^^^^^^^^^^^^^

为指定表注册一个存储钩子。

.. code-block:: solidity

  function registerStoreHook(
    ResourceId tableId,
    IStoreHook hookAddress,
    uint8 enabledHooksBitmap
  )

- ``tableId``: 要注册钩子的表的资源ID
- ``hookAddress``: 钩子合约的地址
- ``enabledHooksBitmap``: 用于指示什么情况下启用钩子的位图
- 要求调用者拥有该表所在的命名空间
- 要求钩子合约实现 ``IStoreHook`` 接口

.. _store-registration-system-unregister-store-hook:

``unregisterStoreHook``
^^^^^^^^^^^^^^^^^^^^^^^

为指定表注销一个存储钩子。

.. code-block:: solidity

  function unregisterStoreHook(ResourceId tableId, IStoreHook hookAddress)

- ``tableId``: 要注销钩子的表的资源ID
- ``hookAddress``: 要注销的钩子合约地址
- 要求调用者拥有该表所在的命名空间

世界注册系统 ``WorldRegistrationSystem``
-----------------------------------------

世界注册系统负责注册命名空间、系统、系统钩子、函数选择器和委托。它提供了以下主要功能：

.. _world-registration-system-register-namespace:

``registerNamespace``
^^^^^^^^^^^^^^^^^^^^^

注册一个新的命名空间。

.. code-block:: solidity

  function registerNamespace(ResourceId namespaceId)

- ``namespaceId``: 要注册的命名空间的资源ID
- 要求命名空间不存在
- 要求命名空间名称有效(不包含保留字符``__``， 不能以 ``_`` 结尾)

.. _world-registration-system-register-system-hook:

``registerSystemHook``
^^^^^^^^^^^^^^^^^^^^^^

为指定系统注册一个系统钩子。

.. code-block:: solidity

  function registerSystemHook(
    ResourceId systemId,
    ISystemHook hookAddress,
    uint8 enabledHooksBitmap
  )

- ``systemId``: 要注册钩子的系统的资源ID
- ``hookAddress``: 钩子合约的地址
- ``enabledHooksBitmap``: 用于指示什么情况下启用钩子的位图
- 要求调用者拥有该系统所在的命名空间
- 要求钩子合约实现 ``ISystemHook`` 接口

.. _world-registration-system-unregister-system-hook:

``unregisterSystemHook``
^^^^^^^^^^^^^^^^^^^^^^^^

为指定系统注销一个系统钩子。

.. code-block:: solidity

  function unregisterSystemHook(ResourceId systemId, ISystemHook hookAddress)

- ``systemId``: 要注销钩子的系统的资源ID
- ``hookAddress``: 要注销的钩子合约地址
- 要求调用者拥有该系统所在的命名空间

.. _world-registration-system-register-system:

``registerSystem``
^^^^^^^^^^^^^^^^^^

注册一个新的系统。

.. code-block:: solidity

  function registerSystem(ResourceId systemId, System system, bool publicAccess)

- ``systemId``: 要注册的系统的资源ID
- ``system``: 系统合约的地址
- ``publicAccess``: 是否允许公开访问
- 要求调用者拥有该系统所在的命名空间
- 要求系统合约实现 ``IWorldContextConsumer`` 接口
- 要求系统合约未曾使用不同的资源 ID 注册过
- 如果系统已存在，则会升级系统

.. _world-registration-system-register-function-selector:

``registerFunctionSelector``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

注册一个系统函数的选择器。

.. code-block:: solidity

  function registerFunctionSelector(
    ResourceId systemId,
    string memory systemFunctionSignature
  ) returns (bytes4 worldFunctionSelector)

- ``systemId``: 系统的资源ID
- ``systemFunctionSignature``: 系统函数的签名
- 创建世界函数到系统函数的映射，世界函数名是系统函数名加命名空间的名称作为前缀，并用 ``__`` 连接
- 要求调用者拥有该系统所在的命名空间
- 要求函数选择器全局唯一

.. _world-registration-system-register-root-function-selector:

``registerRootFunctionSelector``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

注册一个根系统函数的选择器。

.. code-block:: solidity

  function registerRootFunctionSelector(
    ResourceId systemId,
    string memory worldFunctionSignature,
    string memory systemFunctionSignature
  ) returns (bytes4 worldFunctionSelector)

- ``systemId``: 系统的资源ID
- ``worldFunctionSignature``: 世界函数的签名
- ``systemFunctionSignature``: 系统函数的签名
- 创建世界函数到系统函数的映射
- 要求调用者拥有根命名空间
- 要求函数选择器全局唯一

.. _world-registration-system-register-delegation:

``registerDelegation``
^^^^^^^^^^^^^^^^^^^^^^

为单个地址注册一个委托。

.. code-block:: solidity

  function registerDelegation(
    address delegatee,
    ResourceId delegationControlId,
    bytes memory initCallData
  )

- ``delegatee``: 被委托者的地址
- ``delegationControlId``: 委托控制合约的资源ID
- ``initCallData``: 委托控制合约的初始化数据

.. _world-registration-system-unregister-delegation:

``unregisterDelegation``
^^^^^^^^^^^^^^^^^^^^^^^^

为单个地址注销一个委托。

.. code-block:: solidity

  function unregisterDelegation(address delegatee)

- ``delegatee``: 被委托者的地址

.. _world-registration-system-register-namespace-delegation:

``registerNamespaceDelegation``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

为命名空间注册一个委托。

.. code-block:: solidity

  function registerNamespaceDelegation(
    ResourceId namespaceId,
    ResourceId delegationControlId,
    bytes memory initCallData
  )

- ``namespaceId``: 命名空间的资源ID
- ``delegationControlId``: 委托控制合约的资源ID
- ``initCallData``: 委托控制合约的初始化数据
- 要求调用者拥有该命名空间
- 要求委托控制合约不能为默认的无限制委托。

.. _world-registration-system-unregister-namespace-delegation:

``unregisterNamespaceDelegation``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

为命名空间注销一个委托。

.. code-block:: solidity

  function unregisterNamespaceDelegation(ResourceId namespaceId)

- ``namespaceId``: 命名空间的资源ID
- 要求调用者拥有该命名空间


