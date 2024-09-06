配置
=============

MUD 配置文件是 ``packages/contracts/mud.config.ts``。

配置文件主要用于：

1. 定义项目所属命名空间 ``namespace`` 。
2. 定义项目中的表 ``tables`` 。
3. 定义项目中的系统 ``systems`` 。
4. 定义需要安装的模组 ``modules`` 。
5. 配置代码生成
6. 配置部署

配置文件中关于命名空间、表、系统和模组的详细说明请参考各章节。

这里对代码生成和部署的配置进行简单说明。

- ``codegen``: ``object`` 。代码生成配置。

  - ``worldInterfaceName``: ``string`` ，默认: ``"IWorld"`` 。自主世界接口名称。
  - ``worldgenDirectory``: ``string`` ，默认: ``"world"`` 。自主世界接口存放目录。
  - ``worldImportPath``: ``string`` ，默认: ``"@latticexyz/world/src"`` 。
    World 协议有关代码导入路径。
  - ``outputDirectory``: ``string`` ，默认: ``"codegen"`` 。代码生成输出目录。
  - ``indexFilename``: ``string`` ，默认: ``"index.sol"`` 。表索引文件名。
  - ``storeImportPath``: ``string`` ，默认: ``"@latticexyz/store/src"`` 。
    Store 协议有关代码导入路径。
  - ``userTypesFilename``: ``string`` ，默认: ``"common.sol"`` 。用户类型文件名。
- ``deploy``: ``object`` 。部署配置。

  - ``postDeployScript``: ``string`` ，默认: ``"PostDeploy"`` 。部署后运行的脚本名称。脚本文件名
    必须以 ``.s.sol`` 结尾。
  - ``deploysDirectory``: ``string`` ，默认: ``"deploys"`` 。部署信息存放目录。
  - ``worldsFile``: ``string`` ，默认: ``"worlds.json"`` 。一个汇总自主世界地址和所在链的 JSON 文件。
  - ``upgradeableWorldImplementation``: ``boolean`` ，默认: ``false`` 。
    是否允许升级自主世界的核心实现。 官方建议设置为 ``true`` 。

完整配置文件示例：

.. code-block:: ts

  import { defineWorld } from "@latticexyz/world";
  import { encodeAbiParameters, parseAbiParameters, toHex } from 'viem';

  export default defineWorld({
    sourceDirectory: "src",        // 项目合约文件夹，需要与 foundry 配置一致
    namespace: "muddoc",           // 项目所属命名空间
    namespaces: {                  // 多命名空间配置, 与 namespace 冲突
      muddoc: {
        tables: {},
        systems: {},
      },
    },
    enums: {                       // 自定义枚举
      UserStatus: ["active", "inactive"],
    },
    userTypes: {                   // 自定义类型
      ShortString: {
        type: "bytes32",
        filePath: "@openzeppelin/contracts/utils/ShortStrings.sol",
      }
    },
    tables: {                      // 项目中的表
      Users: {
        name: "Users",
        type: "table",
        schema: {
          addr: "address",
          data: "uint256",
          description: "string",
        },
        key: ["addr"],
        codegen: {
          outputDirectory: "tables",
          dataStruct: false,
          tableIdArgument: false,
          storeArgument: false,
        },
        deploy: {
          disabled: true,
        },
      },
    }
    systems: {                     // 项目中的系统
      SimpleStorageSystem: {
        name: "SimpleStorage",
        openAccess: false,
        accessList: [],
        deploy: {
          disabled: false,
          registerWorldFunctions: true,
        },
      },
    },
    excludeSystems: [],            // 禁用的系统
    modules: [                     // 需要安装的模块
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
    codegen: {                     // 代码生成配置
      worldInterfaceName: "IWorld",
      worldgenDirectory: "world",
      worldImportPath: "@latticexyz/world/src",
      outputDirectory: "codegen",
      indexFilename: "index.sol",
      storeImportPath: "@latticexyz/store/src",
      userTypesFilename: "common.sol",
    },
    deploy: {                      // 部署配置
      postDeployScript: "PostDeploy",
      deploysDirectory: "./deploys",
      worldsFile: "./worlds.json",
      upgradeableWorldImplementation: false,
    },
  });
