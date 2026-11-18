# NuGet 包签名

`AGUI.*` NuGet 包在发布时进行**作者签名**，以便 nuget.org 将其标记为已签名。签名在 [`publish-release.yml`](../../../.github/workflows/publish-release.yml) 的 `publish-dotnet` 任务中执行，位于 **pack** 与 **push** 之间的衔接环节：

```
pack  →  SIGN（此环节）  →  verify  →  push
```

签名使用 **DigiCert 代码签名证书**，其私钥以不可导出方式（RSA-HSM）存储在 **Azure Key Vault** 中，通过 [NuGetKeyVaultSignTool](https://github.com/novotnyllc/NuGetKeyVaultSignTool) 和 GitHub OIDC 应用——不存储长期密钥。私钥始终不离开 HSM；签名是在保管库内执行的 `keys/sign` 操作。

此环节**默认不生效**。在将 `SIGNING_PROVIDER` 变量设为 `keyvault` 之前，它不做任何事，发布行为与之前完全一致，因此可以在配置完成前安全合并。

> **作者签名与仓库签名。** nuget.org 在 push 时总会添加自己的*仓库*签名——这是自动的，与此无关。此环节添加的是*作者*签名，携带发布者身份（`CN=Tawkit, Inc.`），并将包标记为"已签名"。
>
> **与 `AGUI.snk` 不同。** `.snk` / `SignAssembly` 机制是程序集*强命名*（DLL 内部的标识）。它与 NuGet 作者签名无关，保持不变。

## 启用

在 **`nuget` 环境**（Settings → Environments → `nuget`）上设置 `SIGNING_PROVIDER=keyvault`，并配置以下内容。`Preflight — validate signing configuration` 步骤会在缺少任何一项时以明确列表使运行失败，因此半配置状态无法发布损坏的包。

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `SIGNING_PROVIDER` | 变量 | `keyvault` |
| `AZURE_CLIENT_ID` | **密钥** | 应用注册（客户端）ID — _已在 `nuget` 环境设置（2026-07-01）_ |
| `AZURE_TENANT_ID` | **密钥** | 目录（租户）ID — _已设置_ |
| `AZURE_SUBSCRIPTION_ID` | **密钥** | 包含保管库的订阅 — _已设置_ |
| `AZURE_KEY_VAULT_URL` | **密钥** | 例如 `https://cpk-signing-kv.vault.azure.net` |
| `CODE_SIGNING_CERT_NAME` | **密钥** | 保管库中的证书名称，例如 `code-signing` |

保管库 URL 和证书名称不是凭据，但保留为**密钥**，以免出现在已提交的工作流和公开 CI 日志中。认证为联合（OIDC）方式——不存储客户端密钥。

## 工作原理

1. `azure/login` 用任务的 OIDC 令牌（`id-token: write`）换取 Azure 登录——使用应用注册的**环境作用域联合凭据**（主体 `repo:ag-ui-protocol/ag-ui:environment:nuget`）。
2. `az account get-access-token --resource https://vault.azure.net` 生成短期 Key Vault 数据平面令牌，该令牌被掩码后通过 `--azure-key-vault-accesstoken` 传给工具（唯一无密钥的认证路径——托管标识 / 客户端密钥模式在 GitHub runner 上不适用）。
3. `NuGetKeyVaultSignTool sign` 使用 SHA-256 和来自 `timestamp.digicert.com` 的 RFC 3161 时间戳对每个 `*.nupkg` 签名（每次调用一个，因此使用循环）。`*.snupkg` 符号包有意不做作者签名。
4. `dotnet nuget verify --all` 把关 push——若包未签名或签名无法链到受信任根，发布失败。

## 配置清单

职责划分：Azure/订阅管理员完成 1–2，仓库管理员完成 3，nuget.org 组织管理员完成 4，然后任何人触发 5。

1. **授予 CI 标识 Key Vault 访问权限。** 应用注册（客户端 `AZURE_CLIENT_ID`）需要保管库上的两个角色——Crypto User 用于签名，Certificate User 用于读取证书：
   ```bash
   SP_OID=$(az ad sp show --id <AZURE_CLIENT_ID> --query id -o tsv)
   VID=$(az keyvault show --name cpk-signing-kv --query id -o tsv)
   az role assignment create --role "Key Vault Crypto User" \
     --assignee-object-id "$SP_OID" --assignee-principal-type ServicePrincipal --scope "$VID"
   az role assignment create --role "Key Vault Certificate User" \
     --assignee-object-id "$SP_OID" --assignee-principal-type ServicePrincipal --scope "$VID"
   ```
2. **环境作用域联合凭据。** 由于任务使用 `environment: nuget`，应用注册需要主体恰为 `repo:ag-ui-protocol/ag-ui:environment:nuget` 的联合凭据（颁发者 `https://token.actions.githubusercontent.com`，受众 `api://AzureADTokenExchange`）。分支/ref 作用域的凭据会静默登录失败。
3. **设置 `nuget` 环境密钥/变量**（见上表）：
   ```bash
   R=ag-ui-protocol/ag-ui
   gh secret set   AZURE_KEY_VAULT_URL    --env nuget --repo $R --body "https://cpk-signing-kv.vault.azure.net"
   gh secret set   CODE_SIGNING_CERT_NAME --env nuget --repo $R --body "code-signing"
   gh variable set SIGNING_PROVIDER       --env nuget --repo $R --body "keyvault"
   ```
4. **在 nuget.org 注册证书。** 导出公钥证书并在 **`ag-ui-protocol` 组织** → Manage → Certificates 下注册：
   ```bash
   az keyvault certificate download --vault-name cpk-signing-kv --name code-signing \
     --file agui-codesign.cer --encoding DER
   ```
   ⚠️ 强制措施是账户级且立即生效的：一旦任何证书注册到所有者，**每次**后续 push 都必须使用已注册证书进行作者签名。请将此与签名 CI（步骤 3）和版本号提升（步骤 5）一起落地——不要提前注册，否则过渡期的未签名发布会遭拒绝。
5. **提升版本号并发布。** nuget.org 版本不可变，因此签名包必须以新版本发布。提升 [`sdks/dotnet/Directory.Build.props`](../Directory.Build.props) 中共享的 `<VersionPrefix>`（例如 `0.0.3` → `0.0.4`）——一次提升即重新发布全部五个包，且已签名。（可选：取消列出旧的未签名版本，让使用者迁移到签名版本。）

## DigiCert 证书为何合格

nuget.org 接受满足以下条件的作者签名证书：链到 Microsoft Trusted Root Program 中的根（DigiCert 是成员）、具有代码签名 EKU、RSA ≥ 2048（此证书为 RSA 3072）、并携带有效的 RFC 3161 时间戳——签发的 DigiCert 代码签名证书全部满足。在 nuget.org 上添加/移除证书需要 2FA。