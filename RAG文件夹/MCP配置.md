## 项目级 .codex/config.toml、全局 ~/.codex/config.toml。你的意思是它两个都会读取，优先全局是吗。



不是“优先全局”。

结论是：

1. **两个都会读**  
    会读 ~/.codex/config.toml（用户级）和项目里的 .codex/config.toml（项目级，前提是项目 trusted）。
    
2. **同名配置时，项目级覆盖全局**  
    官方写的是“project-scoped overrides（项目级覆盖）”。
    
3. **如果有多层 .codex/config.toml**  
    从项目根到当前目录都会加载，**离你当前目录最近的那层优先**。
    
4. **命令行临时参数优先级更高**  
    --config / --model 这类一次性参数会再覆盖文件配置。
    

另外：Codex CLI 的 MCP 主要看 config.toml，不是 .mcp.json。



## 那.mcp.json的作用在哪

对 **Codex CLI** 来说，.mcp.json 基本没作用。  
官方文档写得很直接：Codex 连接 MCP 看的是 ~/.codex/config.toml，并且也会读取项目内的 .codex/config.toml；项目级配置只在 trusted 项目里生效，且同名键由离当前目录最近的那层覆盖。  
来源：

- [https://raw.githubusercontent.com/openai/codex/main/docs/config.md](https://raw.githubusercontent.com/openai/codex/main/docs/config.md)
- [https://developers.openai.com/codex/config-advanced](https://developers.openai.com/codex/config-advanced)

所以在你现在这套环境里：

- ~/.codex/config.toml：Codex 的用户级 MCP 配置
- .codex/config.toml：Codex 的项目级 MCP 配置
- .mcp.json：**不是 Codex CLI 的配置入口**

.mcp.json 的作用更像是“别的 MCP 客户端/编辑器/代理”的约定配置文件。  
也就是说，如果你还在用别的工具，它可能会读这个文件；但 **Codex 不靠它加载 MCP**。这也是为什么你之前 .mcp.json 写对了，我这里还是调不到，直到我把配置写进 config.toml。

如果你想减少维护成本，最稳的做法是：

1. 以 .codex/config.toml 为准给 Codex 用
2. .mcp.json 只在你确实有别的客户端会读它时保留
3. 两边内容保持一致，或者干脆删掉不用的那份