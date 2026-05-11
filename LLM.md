# AWS Bedrock Tool Use Troubleshooting

## Issue: "Start of structure or map found where not expected"

When using tools with the AWS Bedrock backend in `gptel`, requests would frequently fail with an HTTP 400 error and a `SerializationException`.

### Root Causes

1.  **Buffer Parsing Gaps**: The `gptel--parse-buffer` method for Bedrock did not recognize the `gptel-tool` text property. This property is used to track tool calls and results in the Emacs buffer. As a result, tool-related turns were either ignored or parsed as malformed user/assistant messages.
2.  **Invalid JSON Serialization**: In Emacs, `nil` values in plists are often serialized as empty objects `{}` by `json-serialize`. `gptel-bedrock` was producing `(:text nil)` blocks when parsing whitespace or incomplete tool turns, which AWS rejected as invalid structures.
3.  **Strict Role Alternation**: AWS Bedrock requires strict alternation between `user` and `assistant` roles. Additionally, a turn that includes both a text response and a tool call must be sent as a **single message** containing multiple content blocks. `gptel` was previously sending these as separate consecutive messages with the same role, which Bedrock rejects.

### Fix Implementation

The fix involved a significant update to `gptel--parse-buffer` and `gptel--parse-list` in `gptel-bedrock.el`:

-   **Tool awareness**: Added logic to detect and correctly format `toolUse` and `toolResult` blocks by inspecting the `gptel-tool` text property.
-   **Content Validation**: Added checks to prevent pushing empty content blocks or `nil` text values, ensuring the resulting JSON contains only valid strings.
-   **Message Merging**: Implemented a merging pass that combines consecutive messages with the same role into a single message with multiple content blocks. This satisfies Bedrock's requirement for role alternation and supports simultaneous text and tool-use blocks in a single turn.
-   **Advanced Format Support**: Updated `gptel--parse-list` to support the advanced "list of lists" format used by `gptel` for complex multi-turn conversations involving tools.
