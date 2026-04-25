
# hook 是什么

在LLM外部套了一层确定性的限制器。
harness运行时强制执行，能确定性的阻断一些操作。

在settings.json里声明 在某个事件 上 对某些tool 执行一个hook。

- 事件： harness运行时的周期节点。
e.g preToolUse 工具执行前
postToolUse 工具执行后
Stop claude回复完毕时
sessionstart 会话开始时

等等，总共有25+ 种。

- tool：就是tool call，read edit那些。 mcp tools 也算。
hook 写个matcher过滤需要的tool名字，可以实现精准的拦截

## hook的类型：
| 类型    | 运行方式                       | 适用场景                   |
| ------- | ------------------------------- | -------------------------- |
| command | 执行 shell 命令/脚本            | 绝大多数场景，确定性规则   |
| prompt  | 单轮 LLM 调用（默认用 Haiku）   | 需要语义判断的场景         |
| agent   | 多轮子 Agent（可调用工具）      | 需要读文件、跑测试等复杂验证 |
| http    | POST 到 HTTP 端点               | 团队审计服务、外部集成     |

## 一个例子
需求： 禁止 Claude 修改 .env 和 package-lock.json

# .claude/hooks/protect-files.sh
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

PROTECTED_PATTERNS=(".env" "package-lock.json" ".git/")

for pattern in "${PROTECTED_PATTERNS[@]}"; do
if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$pattern'" >&2
    exit 2  # 阻断！
fi
done

exit 0  # 放行

配置：
{
"hooks": {
    "PreToolUse": [{
    "matcher": "Edit|Write",
    "hooks": [{
        "type": "command",
        "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh"
    }]
    }]
}
}

执行流程： Claude 想写 .env → 系统触发 PreToolUse → matcher 匹配到 Write →
执行脚本 → 脚本检测到 .env → stderr 输出原因 → exit 2 → 操作被阻断 → Claude
收到反馈，调整策略。




## 怎么样利用hook确保模型一定会正确触发并使用skills里的内容

其实skills本身的机制里面：system prompt里已经通过 <system-reminder> 注入所有skills的元数据。
system prompt里还有：
```
"When a skill matches the user's request, this is a BLOCKING REQUIREMENT: invoke the relevant Skill tool BEFORE generating any other response about the task"

和 

"NEVER mention a skill without actually calling this tool"
```




