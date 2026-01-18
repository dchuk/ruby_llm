================================================
FILE: README.md
================================================
# Claude Agent SDK for Ruby

> **Disclaimer**: This is an **unofficial, community-maintained** Ruby SDK for Claude Agent. It is not officially supported by Anthropic. For official SDK support, see the [Python SDK](https://docs.claude.com/en/api/agent-sdk/python).
>
> This implementation is based on the official Python SDK and aims to provide feature parity for Ruby developers. Use at your own risk.

[![Gem Version](https://badge.fury.io/rb/claude-agent-sdk.svg?icon=si%3Arubygems)](https://badge.fury.io/rb/claude-agent-sdk)

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Basic Usage: query()](#basic-usage-query)
- [Client](#client)
- [Custom Tools (SDK MCP Servers)](#custom-tools-sdk-mcp-servers)
- [Hooks](#hooks)
- [Permission Callbacks](#permission-callbacks)
- [Structured Output](#structured-output)
- [Budget Control](#budget-control)
- [Fallback Model](#fallback-model)
- [Beta Features](#beta-features)
- [Tools Configuration](#tools-configuration)
- [Sandbox Settings](#sandbox-settings)
- [File Checkpointing & Rewind](#file-checkpointing--rewind)
- [Rails Integration](#rails-integration)
- [Types](#types)
- [Error Handling](#error-handling)
- [Examples](#examples)
- [Development](#development)
- [License](#license)

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'claude-agent-sdk', '~> 0.4.0'
```

And then execute:

```bash
bundle install
```

Or install it yourself as:

```bash
gem install claude-agent-sdk
```

**Prerequisites:**
- Ruby 3.2+
- Node.js
- Claude Code 2.0.0+: `npm install -g @anthropic-ai/claude-code`

## Quick Start

```ruby
require 'claude_agent_sdk'

ClaudeAgentSDK.query(prompt: "What is 2 + 2?") do |message|
  puts message
end
```

## Basic Usage: query()

`query()` is a function for querying Claude Code. It yields response messages to a block.

```ruby
require 'claude_agent_sdk'

# Simple query
ClaudeAgentSDK.query(prompt: "Hello Claude") do |message|
  if message.is_a?(ClaudeAgentSDK::AssistantMessage)
    message.content.each do |block|
      puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
    end
  end
end

# With options
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  system_prompt: "You are a helpful assistant",
  max_turns: 1
)

ClaudeAgentSDK.query(prompt: "Tell me a joke", options: options) do |message|
  puts message
end
```

### Using Tools

```ruby
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  allowed_tools: ['Read', 'Write', 'Bash'],
  permission_mode: 'acceptEdits'  # auto-accept file edits
)

ClaudeAgentSDK.query(
  prompt: "Create a hello.rb file",
  options: options
) do |message|
  # Process tool use and results
end
```

### Working Directory

```ruby
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  cwd: "/path/to/project"
)
```

### Streaming Input

The `query()` function supports streaming input, allowing you to send multiple messages dynamically instead of a single prompt string.

```ruby
require 'claude_agent_sdk'

# Create a stream of messages
messages = ['Hello!', 'What is 2+2?', 'Thanks!']
stream = ClaudeAgentSDK::Streaming.from_array(messages)

# Query with streaming input
ClaudeAgentSDK.query(prompt: stream) do |message|
  puts message if message.is_a?(ClaudeAgentSDK::AssistantMessage)
end
```

You can also create custom streaming enumerators:

```ruby
# Dynamic message generation
stream = Enumerator.new do |yielder|
  yielder << ClaudeAgentSDK::Streaming.user_message("First message")
  # Do some processing...
  yielder << ClaudeAgentSDK::Streaming.user_message("Second message")
  yielder << ClaudeAgentSDK::Streaming.user_message("Third message")
end

ClaudeAgentSDK.query(prompt: stream) do |message|
  # Process responses
end
```

For a complete example, see [examples/streaming_input_example.rb](examples/streaming_input_example.rb).

## Client

`ClaudeAgentSDK::Client` supports bidirectional, interactive conversations with Claude Code. Unlike `query()`, `Client` enables **custom tools**, **hooks**, and **permission callbacks**, all of which can be defined as Ruby procs/lambdas.

**The Client class automatically uses streaming mode** for bidirectional communication, allowing you to send multiple queries dynamically during a single session without closing the connection.

### Basic Client Usage

```ruby
require 'claude_agent_sdk'
require 'async'

Async do
  client = ClaudeAgentSDK::Client.new

  begin
    # Connect automatically uses streaming mode for bidirectional communication
    client.connect

    # Send a query
    client.query("What is the capital of France?")

    # Receive the response
    client.receive_response do |msg|
      if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
        msg.content.each do |block|
          puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      elsif msg.is_a?(ClaudeAgentSDK::ResultMessage)
        puts "Cost: $#{msg.total_cost_usd}" if msg.total_cost_usd
      end
    end

  ensure
    client.disconnect
  end
end.wait
```

### Advanced Client Features

```ruby
Async do
  client = ClaudeAgentSDK::Client.new
  client.connect

  # Send interrupt signal
  client.interrupt

  # Change permission mode during conversation
  client.set_permission_mode('acceptEdits')

  # Change AI model during conversation
  client.set_model('claude-sonnet-4-5')

  # Get server initialization info
  info = client.server_info
  puts "Available commands: #{info}"

  client.disconnect
end.wait
```

## Custom Tools (SDK MCP Servers)

A **custom tool** is a Ruby proc/lambda that you can offer to Claude, for Claude to invoke as needed.

Custom tools are implemented as in-process MCP servers that run directly within your Ruby application, eliminating the need for separate processes that regular MCP servers require.

**Implementation**: This SDK uses the [official Ruby MCP SDK](https://github.com/modelcontextprotocol/ruby-sdk) (`mcp` gem) internally, providing full protocol compliance while offering a simpler block-based API for tool definition.

### Creating a Simple Tool

```ruby
require 'claude_agent_sdk'
require 'async'

# Define a tool using create_tool
greet_tool = ClaudeAgentSDK.create_tool('greet', 'Greet a user', { name: :string }) do |args|
  { content: [{ type: 'text', text: "Hello, #{args[:name]}!" }] }
end

# Create an SDK MCP server
server = ClaudeAgentSDK.create_sdk_mcp_server(
  name: 'my-tools',
  version: '1.0.0',
  tools: [greet_tool]
)

# Use it with Claude
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  mcp_servers: { tools: server },
  allowed_tools: ['mcp__tools__greet']
)

Async do
  client = ClaudeAgentSDK::Client.new(options: options)
  client.connect

  client.query("Greet Alice")
  client.receive_response { |msg| puts msg }

  client.disconnect
end.wait
```

### Benefits Over External MCP Servers

- **No subprocess management** - Runs in the same process as your application
- **Better performance** - No IPC overhead for tool calls
- **Simpler deployment** - Single Ruby process instead of multiple
- **Easier debugging** - All code runs in the same process
- **Direct access** - Tools can directly access your application's state

### Calculator Example

```ruby
# Define calculator tools
add_tool = ClaudeAgentSDK.create_tool('add', 'Add two numbers', { a: :number, b: :number }) do |args|
  result = args[:a] + args[:b]
  { content: [{ type: 'text', text: "#{args[:a]} + #{args[:b]} = #{result}" }] }
end

divide_tool = ClaudeAgentSDK.create_tool('divide', 'Divide numbers', { a: :number, b: :number }) do |args|
  if args[:b] == 0
    { content: [{ type: 'text', text: 'Error: Division by zero' }], is_error: true }
  else
    result = args[:a] / args[:b]
    { content: [{ type: 'text', text: "Result: #{result}" }] }
  end
end

# Create server
calculator = ClaudeAgentSDK.create_sdk_mcp_server(
  name: 'calculator',
  tools: [add_tool, divide_tool]
)

options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  mcp_servers: { calc: calculator },
  allowed_tools: ['mcp__calc__add', 'mcp__calc__divide']
)
```

### Mixed Server Support

You can use both SDK and external MCP servers together:

```ruby
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  mcp_servers: {
    internal: sdk_server,      # In-process SDK server
    external: {                # External subprocess server
      type: 'stdio',
      command: 'external-server'
    }
  }
)
```

### MCP Resources and Prompts

SDK MCP servers can also expose **resources** (data sources) and **prompts** (reusable templates):

```ruby
# Create a resource (data source Claude can read)
config_resource = ClaudeAgentSDK.create_resource(
  uri: 'config://app/settings',
  name: 'Application Settings',
  description: 'Current app configuration',
  mime_type: 'application/json'
) do
  config_data = { app_name: 'MyApp', version: '1.0.0' }
  {
    contents: [{
      uri: 'config://app/settings',
      mimeType: 'application/json',
      text: JSON.pretty_generate(config_data)
    }]
  }
end

# Create a prompt template
review_prompt = ClaudeAgentSDK.create_prompt(
  name: 'code_review',
  description: 'Review code for best practices',
  arguments: [
    { name: 'code', description: 'Code to review', required: true }
  ]
) do |args|
  {
    messages: [{
      role: 'user',
      content: {
        type: 'text',
        text: "Review this code: #{args[:code]}"
      }
    }]
  }
end

# Create server with tools, resources, and prompts
server = ClaudeAgentSDK.create_sdk_mcp_server(
  name: 'dev-tools',
  tools: [my_tool],
  resources: [config_resource],
  prompts: [review_prompt]
)
```

For complete examples, see [examples/mcp_calculator.rb](examples/mcp_calculator.rb) and [examples/mcp_resources_prompts_example.rb](examples/mcp_resources_prompts_example.rb).

## Hooks

A **hook** is a Ruby proc/lambda that the Claude Code *application* (*not* Claude) invokes at specific points of the Claude agent loop. Hooks can provide deterministic processing and automated feedback for Claude. Read more in [Claude Code Hooks Reference](https://docs.anthropic.com/en/docs/claude-code/hooks).

### Example

```ruby
require 'claude_agent_sdk'
require 'async'

Async do
  # Define a hook that blocks dangerous bash commands
  bash_hook = lambda do |input, tool_use_id, context|
    tool_name = input[:tool_name]
    tool_input = input[:tool_input]

    return {} unless tool_name == 'Bash'

    command = tool_input[:command] || ''
    block_patterns = ['rm -rf', 'foo.sh']

    block_patterns.each do |pattern|
      if command.include?(pattern)
        return {
          hookSpecificOutput: {
            hookEventName: 'PreToolUse',
            permissionDecision: 'deny',
            permissionDecisionReason: "Command contains forbidden pattern: #{pattern}"
          }
        }
      end
    end

    {} # Allow if no patterns match
  end

  # Create options with hook
  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    allowed_tools: ['Bash'],
    hooks: {
      'PreToolUse' => [
        ClaudeAgentSDK::HookMatcher.new(
          matcher: 'Bash',
          hooks: [bash_hook]
        )
      ]
    }
  )

  client = ClaudeAgentSDK::Client.new(options: options)
  client.connect

  # Test: Command with forbidden pattern (will be blocked)
  client.query("Run the bash command: ./foo.sh --help")
  client.receive_response { |msg| puts msg }

  client.disconnect
end.wait
```

For more examples, see [examples/hooks_example.rb](examples/hooks_example.rb).

## Permission Callbacks

A **permission callback** is a Ruby proc/lambda that allows you to programmatically control tool execution. This gives you fine-grained control over what tools Claude can use and with what inputs.

### Example

```ruby
require 'claude_agent_sdk'
require 'async'

Async do
  # Define a permission callback
  permission_callback = lambda do |tool_name, input, context|
    # Allow Read operations
    if tool_name == 'Read'
      return ClaudeAgentSDK::PermissionResultAllow.new
    end

    # Block Write to sensitive files
    if tool_name == 'Write'
      file_path = input[:file_path] || input['file_path']
      if file_path && file_path.include?('/etc/')
        return ClaudeAgentSDK::PermissionResultDeny.new(
          message: 'Cannot write to sensitive system files',
          interrupt: false
        )
      end
      return ClaudeAgentSDK::PermissionResultAllow.new
    end

    # Default: allow
    ClaudeAgentSDK::PermissionResultAllow.new
  end

  # Create options with permission callback
  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    allowed_tools: ['Read', 'Write', 'Bash'],
    can_use_tool: permission_callback
  )

  client = ClaudeAgentSDK::Client.new(options: options)
  client.connect

  # This will be allowed
  client.query("Create a file called test.txt with content 'Hello'")
  client.receive_response { |msg| puts msg }

  # This will be blocked
  client.query("Write to /etc/passwd")
  client.receive_response { |msg| puts msg }

  client.disconnect
end.wait
```

For more examples, see [examples/permission_callback_example.rb](examples/permission_callback_example.rb).

## Structured Output

Use `output_format` to get validated JSON responses matching a schema. The Claude CLI returns structured output via a `StructuredOutput` tool use block.

```ruby
require 'claude_agent_sdk'
require 'json'

# Define a JSON schema
schema = {
  type: 'object',
  properties: {
    name: { type: 'string' },
    age: { type: 'integer' },
    skills: { type: 'array', items: { type: 'string' } }
  },
  required: %w[name age skills]
}

options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  output_format: { type: 'json_schema', schema: schema },
  max_turns: 3
)

structured_data = nil

ClaudeAgentSDK.query(
  prompt: "Create a profile for a software engineer",
  options: options
) do |message|
  if message.is_a?(ClaudeAgentSDK::AssistantMessage)
    message.content.each do |block|
      # Structured output comes via StructuredOutput tool use
      if block.is_a?(ClaudeAgentSDK::ToolUseBlock) && block.name == 'StructuredOutput'
        structured_data = block.input
      end
    end
  end
end

if structured_data
  puts "Name: #{structured_data[:name]}"
  puts "Age: #{structured_data[:age]}"
  puts "Skills: #{structured_data[:skills].join(', ')}"
end
```

For complete examples, see [examples/structured_output_example.rb](examples/structured_output_example.rb).

## Budget Control

Use `max_budget_usd` to set a spending cap for your queries:

```ruby
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  max_budget_usd: 0.10,  # Cap at $0.10
  max_turns: 3
)

ClaudeAgentSDK.query(prompt: "Explain recursion", options: options) do |message|
  if message.is_a?(ClaudeAgentSDK::ResultMessage)
    puts "Cost: $#{message.total_cost_usd}"
  end
end
```

For complete examples, see [examples/budget_control_example.rb](examples/budget_control_example.rb).

## Fallback Model

Use `fallback_model` to specify a backup model if the primary is unavailable:

```ruby
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  model: 'claude-sonnet-4-20250514',
  fallback_model: 'claude-3-5-haiku-20241022'
)

ClaudeAgentSDK.query(prompt: "Hello", options: options) do |message|
  if message.is_a?(ClaudeAgentSDK::AssistantMessage)
    puts "Model used: #{message.model}"
  end
end
```

For complete examples, see [examples/fallback_model_example.rb](examples/fallback_model_example.rb).

## Beta Features

Enable experimental features using the `betas` option:

```ruby
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  betas: ['context-1m-2025-08-07']  # Extended context window
)

ClaudeAgentSDK.query(prompt: "Analyze this large document...", options: options) do |message|
  puts message
end
```

Available beta features are listed in the `SDK_BETAS` constant.

## Tools Configuration

Configure base tools separately from allowed tools:

```ruby
# Using an array of tool names
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  tools: ['Read', 'Edit', 'Bash']  # Base tools available
)

# Using a preset
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  tools: ClaudeAgentSDK::ToolsPreset.new(preset: 'claude_code')
)

# Appending to allowed tools
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  append_allowed_tools: ['Write', 'Bash']
)
```

## Sandbox Settings

Run commands in an isolated sandbox for additional security:

```ruby
sandbox = ClaudeAgentSDK::SandboxSettings.new(
  enabled: true,
  auto_allow_bash_if_sandboxed: true,
  network: ClaudeAgentSDK::SandboxNetworkConfig.new(
    allow_local_binding: true
  )
)

options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  sandbox: sandbox,
  permission_mode: 'acceptEdits'
)

ClaudeAgentSDK.query(prompt: "Run some commands", options: options) do |message|
  puts message
end
```

## File Checkpointing & Rewind

Enable file checkpointing to revert file changes to a previous state:

```ruby
require 'async'

Async do
  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    enable_file_checkpointing: true,
    permission_mode: 'acceptEdits'
  )

  client = ClaudeAgentSDK::Client.new(options: options)
  client.connect

  # Track user message UUIDs for potential rewind
  user_message_uuids = []

  # First query - create a file
  client.query("Create a test.rb file with some code")
  client.receive_response do |message|
    # Process all message types as needed
    case message
    when ClaudeAgentSDK::UserMessage
      # Capture UUID for rewind capability
      user_message_uuids << message.uuid if message.uuid
    when ClaudeAgentSDK::AssistantMessage
      # Handle assistant responses
      message.content.each do |block|
        puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
      end
    when ClaudeAgentSDK::ResultMessage
      puts "Query completed (cost: $#{message.total_cost_usd})"
    end
  end

  # Second query - modify the file
  client.query("Modify the test.rb file to add error handling")
  client.receive_response do |message|
    user_message_uuids << message.uuid if message.is_a?(ClaudeAgentSDK::UserMessage) && message.uuid
  end

  # Rewind to the first checkpoint (undoes the second query's changes)
  if user_message_uuids.first
    puts "Rewinding to checkpoint: #{user_message_uuids.first}"
    client.rewind_files(user_message_uuids.first)
  end

  client.disconnect
end
```

> **Note:** The `uuid` field on `UserMessage` is populated by the CLI and represents checkpoint identifiers. Rewinding to a UUID restores file state to what it was at that point in the conversation.

## Rails Integration

The SDK integrates well with Rails applications. Here are common patterns:

### ActionCable Streaming

Stream Claude responses to the frontend in real-time:

```ruby
# app/jobs/chat_agent_job.rb
class ChatAgentJob < ApplicationJob
  queue_as :claude_agents

  def perform(chat_id, message_content)
    Async do
      options = ClaudeAgentSDK::ClaudeAgentOptions.new(
        system_prompt: { type: 'preset', preset: 'claude_code' },
        permission_mode: 'bypassPermissions'
      )

      client = ClaudeAgentSDK::Client.new(options: options)

      begin
        client.connect
        client.query(message_content)

        client.receive_response do |message|
          case message
          when ClaudeAgentSDK::AssistantMessage
            text = extract_text(message)
            ChatChannel.broadcast_to(chat_id, { type: 'chunk', content: text })

          when ClaudeAgentSDK::ResultMessage
            ChatChannel.broadcast_to(chat_id, {
              type: 'complete',
              content: message.result,
              cost: message.total_cost_usd
            })
          end
        end
      ensure
        client.disconnect
      end
    end.wait
  end

  private

  def extract_text(message)
    message.content
      .select { |b| b.is_a?(ClaudeAgentSDK::TextBlock) }
      .map(&:text)
      .join("\n\n")
  end
end
```

### Session Resumption

Persist Claude sessions for multi-turn conversations:

```ruby
# app/models/chat_session.rb
class ChatSession < ApplicationRecord
  # Columns: id, claude_session_id, user_id, created_at, updated_at

  def send_message(content)
    options = build_options
    client = ClaudeAgentSDK::Client.new(options: options)

    Async do
      client.connect
      client.query(content, session_id: claude_session_id ? nil : generate_session_id)

      client.receive_response do |message|
        if message.is_a?(ClaudeAgentSDK::ResultMessage)
          # Save session ID for next message
          update!(claude_session_id: message.session_id)
        end
      end
    ensure
      client.disconnect
    end.wait
  end

  private

  def build_options
    opts = {
      permission_mode: 'bypassPermissions',
      setting_sources: []
    }
    opts[:resume] = claude_session_id if claude_session_id.present?
    ClaudeAgentSDK::ClaudeAgentOptions.new(**opts)
  end

  def generate_session_id
    "chat_#{id}_#{Time.current.to_i}"
  end
end
```

### Background Jobs with Error Handling

```ruby
class ClaudeAgentJob < ApplicationJob
  queue_as :claude_agents
  retry_on ClaudeAgentSDK::ProcessError, wait: :polynomially_longer, attempts: 3

  def perform(task_id)
    task = Task.find(task_id)

    Async do
      execute_agent(task)
    end.wait

  rescue ClaudeAgentSDK::CLINotFoundError => e
    task.update!(status: 'failed', error: 'Claude CLI not installed')
    raise
  end

  private

  def execute_agent(task)
    # ... agent execution
  end
end
```

### HTTP MCP Servers

Connect to remote tool services:

```ruby
mcp_servers = {
  'api_tools' => ClaudeAgentSDK::McpHttpServerConfig.new(
    url: ENV['MCP_SERVER_URL'],
    headers: { 'Authorization' => "Bearer #{ENV['MCP_TOKEN']}" }
  ).to_h
}

options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  mcp_servers: mcp_servers,
  permission_mode: 'bypassPermissions'
)
```

For complete examples, see:
- [examples/rails_actioncable_example.rb](examples/rails_actioncable_example.rb)
- [examples/rails_background_job_example.rb](examples/rails_background_job_example.rb)
- [examples/session_resumption_example.rb](examples/session_resumption_example.rb)
- [examples/http_mcp_server_example.rb](examples/http_mcp_server_example.rb)

## Types

See [lib/claude_agent_sdk/types.rb](lib/claude_agent_sdk/types.rb) for complete type definitions.

### Message Types

```ruby
# Union type of all possible messages
Message = UserMessage | AssistantMessage | SystemMessage | ResultMessage
```

#### UserMessage

User input message.

```ruby
class UserMessage
  attr_accessor :content,           # String | Array<ContentBlock>
                :uuid,              # String | nil - Unique ID for rewind support
                :parent_tool_use_id # String | nil
end
```

#### AssistantMessage

Assistant response message with content blocks.

```ruby
class AssistantMessage
  attr_accessor :content,           # Array<ContentBlock>
                :model,             # String
                :parent_tool_use_id,# String | nil
                :error              # String | nil ('authentication_failed', 'billing_error', 'rate_limit', 'invalid_request', 'server_error', 'unknown')
end
```

#### SystemMessage

System message with metadata.

```ruby
class SystemMessage
  attr_accessor :subtype,  # String ('init', etc.)
                :data      # Hash
end
```

#### ResultMessage

Final result message with cost and usage information.

```ruby
class ResultMessage
  attr_accessor :subtype,           # String
                :duration_ms,       # Integer
                :duration_api_ms,   # Integer
                :is_error,          # Boolean
                :num_turns,         # Integer
                :session_id,        # String
                :total_cost_usd,    # Float | nil
                :usage,             # Hash | nil
                :result,            # String | nil (final text result)
                :structured_output  # Hash | nil (when using output_format)
end
```

### Content Block Types

```ruby
# Union type of all content blocks
ContentBlock = TextBlock | ThinkingBlock | ToolUseBlock | ToolResultBlock
```

#### TextBlock

Text content block.

```ruby
class TextBlock
  attr_accessor :text  # String
end
```

#### ThinkingBlock

Thinking content block (for models with extended thinking capability).

```ruby
class ThinkingBlock
  attr_accessor :thinking,  # String
                :signature  # String
end
```

#### ToolUseBlock

Tool use request block.

```ruby
class ToolUseBlock
  attr_accessor :id,    # String
                :name,  # String
                :input  # Hash
end
```

#### ToolResultBlock

Tool execution result block.

```ruby
class ToolResultBlock
  attr_accessor :tool_use_id,  # String
                :content,      # String | Array<Hash> | nil
                :is_error      # Boolean | nil
end
```

### Error Types

```ruby
# Base exception class for all SDK errors
class ClaudeSDKError < StandardError; end

# Raised when Claude Code CLI is not found
class CLINotFoundError < CLIConnectionError
  # @param message [String] Error message (default: "Claude Code not found")
  # @param cli_path [String, nil] Optional path to the CLI that was not found
end

# Raised when connection to Claude Code fails
class CLIConnectionError < ClaudeSDKError; end

# Raised when the Claude Code process fails
class ProcessError < ClaudeSDKError
  attr_reader :exit_code,  # Integer | nil
              :stderr      # String | nil
end

# Raised when JSON parsing fails
class CLIJSONDecodeError < ClaudeSDKError
  attr_reader :line,           # String - The line that failed to parse
              :original_error  # Exception - The original JSON decode exception
end

# Raised when message parsing fails
class MessageParseError < ClaudeSDKError
  attr_reader :data  # Hash | nil
end
```

### Configuration Types

| Type | Description |
|------|-------------|
| `ClaudeAgentOptions` | Main configuration for queries and clients |
| `HookMatcher` | Hook configuration with matcher pattern and timeout |
| `PermissionResultAllow` | Permission callback result to allow tool use |
| `PermissionResultDeny` | Permission callback result to deny tool use |
| `AgentDefinition` | Agent definition with description, prompt, tools, model |
| `McpStdioServerConfig` | MCP server config for stdio transport |
| `McpSSEServerConfig` | MCP server config for SSE transport |
| `McpHttpServerConfig` | MCP server config for HTTP transport |
| `SdkPluginConfig` | SDK plugin configuration |
| `SandboxSettings` | Sandbox settings for isolated command execution |
| `SandboxNetworkConfig` | Network configuration for sandbox |
| `SandboxIgnoreViolations` | Configure which sandbox violations to ignore |
| `SystemPromptPreset` | System prompt preset configuration |
| `ToolsPreset` | Tools preset configuration for base tools selection |

### Constants

| Constant | Description |
|----------|-------------|
| `SDK_BETAS` | Available beta features (e.g., `"context-1m-2025-08-07"`) |
| `PERMISSION_MODES` | Available permission modes |
| `SETTING_SOURCES` | Available setting sources |
| `HOOK_EVENTS` | Available hook events |
| `ASSISTANT_MESSAGE_ERRORS` | Possible error types in AssistantMessage |

## Error Handling

### AssistantMessage Errors

`AssistantMessage` includes an `error` field for API-level errors:

```ruby
ClaudeAgentSDK.query(prompt: "Hello") do |message|
  if message.is_a?(ClaudeAgentSDK::AssistantMessage) && message.error
    case message.error
    when 'rate_limit'
      puts "Rate limited - retry after delay"
    when 'authentication_failed'
      puts "Check your API key"
    when 'billing_error'
      puts "Check your billing status"
    when 'invalid_request'
      puts "Invalid request format"
    when 'server_error'
      puts "Server error - retry later"
    end
  end
end
```

For complete examples, see [examples/error_handling_example.rb](examples/error_handling_example.rb).

### Exception Handling

```ruby
require 'claude_agent_sdk'

begin
  ClaudeAgentSDK.query(prompt: "Hello") do |message|
    puts message
  end
rescue ClaudeAgentSDK::CLINotFoundError
  puts "Please install Claude Code"
rescue ClaudeAgentSDK::ProcessError => e
  puts "Process failed with exit code: #{e.exit_code}"
rescue ClaudeAgentSDK::CLIJSONDecodeError => e
  puts "Failed to parse response: #{e}"
end
```

### Error Types

| Error | Description |
|-------|-------------|
| `ClaudeSDKError` | Base error for all SDK errors |
| `CLINotFoundError` | Claude Code not installed |
| `CLIConnectionError` | Connection issues |
| `ProcessError` | Process failed (includes `exit_code` and `stderr`) |
| `CLIJSONDecodeError` | JSON parsing issues |
| `MessageParseError` | Message parsing issues |

See [lib/claude_agent_sdk/errors.rb](lib/claude_agent_sdk/errors.rb) for all error types.

## Available Tools

See the [Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code/settings#tools-available-to-claude) for a complete list of available tools.

## Examples

### Core Examples

| Example | Description |
|---------|-------------|
| [examples/quick_start.rb](examples/quick_start.rb) | Basic `query()` usage with options |
| [examples/client_example.rb](examples/client_example.rb) | Interactive Client usage |
| [examples/streaming_input_example.rb](examples/streaming_input_example.rb) | Streaming input for multi-turn conversations |
| [examples/session_resumption_example.rb](examples/session_resumption_example.rb) | Multi-turn conversations with session persistence |
| [examples/structured_output_example.rb](examples/structured_output_example.rb) | JSON schema structured output |
| [examples/error_handling_example.rb](examples/error_handling_example.rb) | Error handling with `AssistantMessage.error` |

### MCP Server Examples

| Example | Description |
|---------|-------------|
| [examples/mcp_calculator.rb](examples/mcp_calculator.rb) | Custom tools with SDK MCP servers |
| [examples/mcp_resources_prompts_example.rb](examples/mcp_resources_prompts_example.rb) | MCP resources and prompts |
| [examples/http_mcp_server_example.rb](examples/http_mcp_server_example.rb) | HTTP/SSE MCP server configuration |

### Hooks & Permissions

| Example | Description |
|---------|-------------|
| [examples/hooks_example.rb](examples/hooks_example.rb) | Using hooks to control tool execution |
| [examples/advanced_hooks_example.rb](examples/advanced_hooks_example.rb) | Typed hook inputs/outputs |
| [examples/permission_callback_example.rb](examples/permission_callback_example.rb) | Dynamic tool permission control |

### Advanced Features

| Example | Description |
|---------|-------------|
| [examples/budget_control_example.rb](examples/budget_control_example.rb) | Budget control with `max_budget_usd` |
| [examples/fallback_model_example.rb](examples/fallback_model_example.rb) | Fallback model configuration |
| [examples/extended_thinking_example.rb](examples/extended_thinking_example.rb) | Extended thinking (API parity) |

### Rails Integration

| Example | Description |
|---------|-------------|
| [examples/rails_actioncable_example.rb](examples/rails_actioncable_example.rb) | ActionCable streaming to frontend |
| [examples/rails_background_job_example.rb](examples/rails_background_job_example.rb) | Background jobs with session resumption |

## Development

After checking out the repo, run `bundle install` to install dependencies. Then, run `bundle exec rspec` to run the tests.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).



================================================
FILE: CHANGELOG.md
================================================
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.4.0] - 2026-01-06

### Added

#### File Checkpointing & Rewind
- `enable_file_checkpointing` option in `ClaudeAgentOptions` for enabling file state checkpointing
- `rewind_files(user_message_uuid)` method on `Query` and `Client` classes
- `uuid` field on `UserMessage` for tracking message identifiers for rewind support

#### Beta Features Support
- `SDK_BETAS` constant with available beta features (e.g., `"context-1m-2025-08-07"`)
- `betas` option in `ClaudeAgentOptions` for enabling beta features

#### Tools Configuration
- `tools` option for base tools selection (separate from `allowed_tools`)
- Supports array of tool names, empty array `[]`, or `ToolsPreset` object
- `ToolsPreset` class for preset-based tool configuration
- `append_allowed_tools` option to append tools to the allowed list

#### Sandbox Settings
- `SandboxSettings` class for isolated command execution configuration
- `SandboxNetworkConfig` class for network isolation settings
- `SandboxIgnoreViolations` class for configuring violation handling
- `sandbox` option in `ClaudeAgentOptions` for sandbox configuration
- Automatic merging of sandbox settings into the main settings JSON

#### Additional Types
- `SystemPromptPreset` class for preset-based system prompts

### Technical Details
- All new CLI flags properly passed to Claude Code subprocess
- Sandbox settings merged into `--settings` JSON for CLI compatibility
- UserMessage UUID parsed from CLI output for rewind support

## [0.2.0] - 2025-10-17

### Changed
- **BREAKING:** Updated minimum Ruby version from 3.0+ to 3.2+ (required by official MCP SDK)
- **Major refactoring:** SDK MCP server now uses official Ruby MCP SDK (`mcp` gem v0.4) internally
- Internal implementation migrated from custom MCP to wrapping official `MCP::Server`

### Added
- Official Ruby MCP SDK (`mcp` gem) as runtime dependency
- Full MCP protocol compliance via official SDK
- `handle_json` method for protocol-compliant JSON-RPC handling

### Improved
- Better long-term maintenance by leveraging official SDK updates
- Aligned with Python SDK implementation pattern (using official MCP library)
- All 86 tests passing with full backward compatibility maintained

### Technical Details
- Creates dynamic `MCP::Tool`, `MCP::Resource`, and `MCP::Prompt` classes from block-based definitions
- User-facing API remains unchanged - no breaking changes for Ruby 3.2+ users
- Maintains backward-compatible methods (`list_tools`, `call_tool`, etc.)

## [0.1.3] - 2025-10-15

### Added
- **MCP resource support:** Full support for MCP resources (list, read, subscribe operations)
- **MCP prompt support:** Support for MCP prompts (list, get operations)
- **Streaming input support:** Added streaming capabilities for input handling
- Feature complete MCP implementation matching Python SDK functionality

## [0.1.2] - 2025-10-14

### Fixed
- **Critical:** Replaced `Async::Process` with Ruby's built-in `Open3` for subprocess management
- Fixed "uninitialized constant Async::Process" error that prevented the gem from working
- Process management now uses standard Ruby threads instead of async tasks
- All 86 tests passing

## [0.1.1] - 2025-10-14

### Fixed
- Added `~/.claude/local/claude` to CLI search paths to detect Claude Code in its default installation location
- Fixed issue where SDK couldn't find Claude Code when accessed via shell alias

### Added
- Comprehensive test suite with 86 passing tests
- Test documentation in spec/README.md

### Changed
- Marked as unofficial SDK in README and gemspec
- Updated repository URLs to reflect community-maintained status

## [0.1.0] - 2025-10-14

### Added
- Initial release of Claude Agent SDK for Ruby
- Support for `query()` function for simple one-shot interactions
- `ClaudeSDKClient` class for bidirectional, stateful conversations
- Custom tool support via SDK MCP servers
- Hook support for all major hook events
- Comprehensive error handling
- Full async/await support using the `async` gem
- Examples demonstrating common use cases



================================================
FILE: claude-agent-sdk.gemspec
================================================
# frozen_string_literal: true

require_relative 'lib/claude_agent_sdk/version'

Gem::Specification.new do |spec|
  spec.name = 'claude-agent-sdk'
  spec.version = ClaudeAgentSDK::VERSION
  spec.authors = ['Community Contributors']
  spec.email = []

  spec.summary = 'Unofficial Ruby SDK for Claude Agent'
  spec.description = 'Unofficial Ruby SDK for interacting with Claude Code, supporting bidirectional conversations, custom tools, and hooks. Not officially maintained by Anthropic.'
  spec.homepage = 'https://github.com/ya-luotao/claude-agent-sdk-ruby'
  spec.license = 'MIT'
  spec.required_ruby_version = '>= 3.2.0'

  spec.metadata['homepage_uri'] = spec.homepage
  spec.metadata['source_code_uri'] = 'https://github.com/ya-luotao/claude-agent-sdk-ruby'
  spec.metadata['changelog_uri'] = 'https://github.com/ya-luotao/claude-agent-sdk-ruby/blob/main/CHANGELOG.md'
  spec.metadata['documentation_uri'] = 'https://docs.anthropic.com/en/docs/claude-code/sdk'

  # Specify which files should be added to the gem when it is released.
  spec.files = Dir['lib/**/*', 'README.md', 'LICENSE', 'CHANGELOG.md']
  spec.require_paths = ['lib']

  # Runtime dependencies
  spec.add_dependency 'async', '~> 2.0'
  spec.add_dependency 'mcp', '~> 0.4'

  # Development dependencies
  spec.add_development_dependency 'bundler', '~> 2.0'
  spec.add_development_dependency 'rake', '~> 13.0'
  spec.add_development_dependency 'rspec', '~> 3.0'
  spec.add_development_dependency 'rubocop', '~> 1.0'
end



================================================
FILE: Gemfile
================================================
# frozen_string_literal: true

source 'https://rubygems.org'

# Specify your gem's dependencies in claude-agent-sdk.gemspec
gemspec

gem 'rake', '~> 13.0'
gem 'rspec', '~> 3.0'
gem 'rubocop', '~> 1.0'



================================================
FILE: IMPLEMENTATION.md
================================================
# Claude Agent SDK Ruby Implementation

This document provides an overview of the Ruby implementation of the Claude Agent SDK, based on the Python version.

## Implementation Summary

The Ruby SDK has been fully implemented with the following components:

### Core Components (~1,500+ lines of code)

1. **Transport Layer** (411 lines)
   - `transport.rb` - Abstract transport interface (44 lines)
   - `subprocess_cli_transport.rb` - CLI subprocess implementation (367 lines)
   - Features: Process management, stderr handling, version checking, command building

2. **Query Class** (424 lines)
   - `query.rb` - Full control protocol implementation
   - Features:
     - Bidirectional control request/response routing
     - Hook callback execution
     - Permission callback handling
     - Message streaming with async queues
     - Initialization handshake
     - **Full SDK MCP server support** ✨
     - Control operations: interrupt, set_permission_mode, set_model

3. **SDK MCP Server** (~150 lines)
   - `sdk_mcp_server.rb` - In-process MCP server implementation
   - Features:
     - `SdkMcpServer` class for managing tools
     - `create_tool` helper for defining tools
     - `create_sdk_mcp_server` function for server creation
     - JSON schema generation from Ruby types
     - Tool execution with error handling
     - Full MCP protocol support (initialize, tools/list, tools/call)

4. **Type System** (358 lines)
   - `types.rb` - Complete type definitions
   - Message types: UserMessage, AssistantMessage, SystemMessage, ResultMessage, StreamEvent
   - Content blocks: TextBlock, ThinkingBlock, ToolUseBlock, ToolResultBlock
   - Configuration: ClaudeAgentOptions with all major settings
   - MCP server configs: McpStdioServerConfig, McpSSEServerConfig, McpHttpServerConfig, McpSdkServerConfig
   - Permission types: PermissionResultAllow, PermissionResultDeny, PermissionUpdate
   - Hook types: HookMatcher, HookCallback

5. **Message Parser** (103 lines)
   - `message_parser.rb` - JSON message parsing
   - Handles all message types with proper error handling

6. **Error Handling** (53 lines)
   - `errors.rb` - Comprehensive error classes
   - ClaudeSDKError, CLIConnectionError, CLINotFoundError, ProcessError, CLIJSONDecodeError, MessageParseError

7. **Main Library** (256 lines)
   - `lib/claude_agent_sdk.rb` - Entry point with query() and Client class
   - Features:
     - Simple `query()` function for one-shot queries
     - Full-featured `Client` class for bidirectional conversations
     - Hooks support
     - Permission callbacks support
     - Advanced features: interrupt, set_permission_mode, set_model, server_info

### Examples (~450 lines)

1. **quick_start.rb** - Basic usage examples with query()
2. **client_example.rb** - Interactive client usage
3. **mcp_calculator.rb** - Custom tools with SDK MCP servers ✨
4. **hooks_example.rb** - Hook callback examples
5. **permission_callback_example.rb** - Permission callback examples

### Documentation

- **README.md** - Comprehensive documentation with examples
- **CHANGELOG.md** - Version history
- **IMPLEMENTATION.md** - This file

## Architecture

The Ruby SDK follows the Python SDK's architecture closely:

```
┌─────────────────────────────────────────┐
│           User Application              │
│  (query() or Client with callbacks)     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         Query (Control Protocol)        │
│  - Hook execution                       │
│  - Permission callbacks                 │
│  - Message routing                      │
│  - Control requests/responses           │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│     SubprocessCLITransport              │
│  - Process management                   │
│  - stdin/stdout/stderr handling         │
│  - Command building                     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         Claude Code CLI                 │
│  (External Node.js process)             │
└─────────────────────────────────────────┘
```

## Key Design Decisions

1. **Async Runtime**: Uses the `async` gem (Ruby's async/await runtime) instead of Python's `anyio`
2. **Process Management**: Uses Ruby's built-in `Open3` for subprocess management (no external process gems needed)
3. **Message Passing**: Uses `Async::Queue` for message streaming instead of memory object streams
4. **Synchronization**: Uses `Async::Condition` for control request/response coordination
5. **Threading**: Uses standard Ruby threads for stderr handling
6. **Naming**: Follows Ruby conventions (snake_case) while maintaining API similarity
7. **Callbacks**: Uses Ruby procs/lambdas for hooks and permission callbacks

## Features Implemented

### ✅ Completed

- [x] Basic query() function for simple queries
- [x] Full Client class for interactive conversations
- [x] Transport abstraction with subprocess implementation using Open3
- [x] Complete message type system
- [x] Message parsing with error handling
- [x] Control protocol (bidirectional communication)
- [x] Hook support (PreToolUse, PostToolUse, etc.)
- [x] Permission callback support
- [x] Interrupt capability
- [x] Dynamic permission mode changes
- [x] Dynamic model changes
- [x] Server info retrieval
- [x] Comprehensive error handling
- [x] **SDK MCP server support** (in-process custom tools) ✨
- [x] **Full MCP protocol** (initialize, tools/list, tools/call) ✨
- [x] Working examples for all features
- [x] **Comprehensive test suite** (86 passing RSpec tests) ✨
- [x] Session forking support
- [x] Agent definitions support
- [x] Setting sources control
- [x] Partial messages streaming support
- [x] **Streaming input support** (Enumerator-based prompts) ✨
- [x] **MCP resource support** (Data sources for SDK servers) ✨
- [x] **MCP prompt support** (Reusable prompt templates) ✨

### ⏱️ Not Yet Implemented

All major features have been implemented! The SDK has complete feature parity with the Python SDK.

## Comparison with Python SDK

### Similarities
- Same overall architecture
- Same message types and structures
- Same control protocol
- Same hook and permission callback concepts
- Similar API design

### Differences

| Feature | Python | Ruby |
|---------|--------|------|
| Async runtime | anyio | async gem |
| Process management | Async subprocess | Open3 (built-in) |
| Message queue | MemoryObjectStream | Async::Queue |
| Synchronization | anyio.Event | Async::Condition |
| Type hints | Yes (TypedDict, dataclass) | No (uses regular classes) |
| MCP integration | Full with mcp package | Full SDK MCP support |
| Async iterators | Built-in | Uses blocks/enumerators |

## Usage Comparison

### Python
```python
import anyio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(prompt="Hello"):
        print(message)

anyio.run(main)
```

### Ruby
```ruby
require 'claude_agent_sdk'
require 'async'

Async do
  ClaudeAgentSDK.query(prompt: "Hello") do |message|
    puts message
  end
end.wait
```

## Dependencies

- **async** (~2.0) - Async I/O runtime for concurrent operations
- Ruby 3.0+ required
- No external process management gems needed (uses built-in Open3)

## Testing

The SDK includes a comprehensive RSpec test suite with **86 passing tests**:

```
spec/
├── unit/                           # Unit tests (66 tests)
│   ├── errors_spec.rb              # Error class tests (6 tests)
│   ├── types_spec.rb               # Type system tests (24 tests)
│   ├── message_parser_spec.rb      # Message parsing tests (12 tests)
│   ├── sdk_mcp_server_spec.rb      # MCP server tests (21 tests)
│   └── transport_spec.rb           # Transport tests (3 tests)
├── integration/                    # Integration tests (20 tests)
│   └── query_spec.rb               # End-to-end workflow tests
├── support/
│   └── test_helpers.rb             # Shared fixtures and helpers
└── README.md                       # Test documentation
```

Run tests with:
```bash
bundle exec rspec                    # Run all tests
bundle exec rspec --format documentation  # Detailed output
PROFILE=1 bundle exec rspec         # Show slowest tests
```

## Future Enhancements

1. **Additional MCP Features**
   - MCP server lifecycle management (start/stop/restart)
   - Resource subscriptions (watch for changes)
   - Sampling support

2. **Additional Features**
   - Connection pooling for multiple queries
   - Session persistence and replay

3. **Performance**
   - Optimize message parsing
   - Better async task management
   - Lazy loading of optional components

4. **Developer Experience**
   - Debug logging support
   - Type documentation (YARD)
   - More examples and tutorials
   - Performance profiling tools

## SDK MCP Server Implementation

The Ruby SDK now includes full support for in-process MCP servers, allowing developers to define custom tools that run directly in their Ruby applications.

### Example Usage

```ruby
# Define a tool
add_tool = ClaudeAgentSDK.create_tool('add', 'Add numbers', { a: :number, b: :number }) do |args|
  result = args[:a] + args[:b]
  { content: [{ type: 'text', text: "Result: #{result}" }] }
end

# Create server
server = ClaudeAgentSDK.create_sdk_mcp_server(name: 'calc', tools: [add_tool])

# Use with Claude
options = ClaudeAgentOptions.new(
  mcp_servers: { calc: server },
  allowed_tools: ['mcp__calc__add']
)
```

### Architecture

The SDK MCP implementation consists of:
1. **SdkMcpServer** - Manages tool registry and execution
2. **create_tool** - Helper for defining tools with schemas
3. **Query integration** - Routes MCP messages to server instances
4. **JSON schema conversion** - Converts Ruby types to JSON schemas

## Conclusion

The Ruby SDK successfully implements **complete feature parity** with the Python SDK, including the advanced SDK MCP server functionality. The implementation prioritizes correctness and maintainability while following Ruby idioms and conventions.

### Project Statistics

- **Core implementation:** ~1,700 lines of production code
- **Test suite:** 86 passing tests covering all major components
- **Examples:** 5 comprehensive examples demonstrating all features
- **Documentation:** Complete README, CHANGELOG, and implementation guide
- **Dependencies:** Minimal (only `async` gem + Ruby stdlib)
- **Ruby version:** 3.0+ required

### Key Achievements

✅ Full bidirectional communication with Claude Code CLI
✅ Complete SDK MCP server support for in-process tools
✅ Comprehensive hook and permission callback system
✅ Production-ready with 86 passing tests
✅ Zero external dependencies for subprocess management (uses Open3)
✅ Clean, idiomatic Ruby code following community conventions

The SDK is **production-ready** and actively maintained as an unofficial, community-driven project.



================================================
FILE: LICENSE
================================================
MIT License

Copyright (c) 2025 Anthropic

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.



================================================
FILE: Rakefile
================================================
# frozen_string_literal: true

require 'bundler/gem_tasks'
require 'rspec/core/rake_task'
require 'rubocop/rake_task'

RSpec::Core::RakeTask.new(:spec)
RuboCop::RakeTask.new

task default: %i[spec rubocop]



================================================
FILE: .rspec
================================================
--require spec_helper
--color
--format documentation



================================================
FILE: .rubocop.yml
================================================
AllCops:
  TargetRubyVersion: 3.0
  NewCops: enable
  Exclude:
    - 'vendor/**/*'
    - 'tmp/**/*'

Style/Documentation:
  Enabled: false

Metrics/MethodLength:
  Max: 30

Metrics/ClassLength:
  Max: 250

Metrics/BlockLength:
  Exclude:
    - 'spec/**/*'
    - '*.gemspec'

Layout/LineLength:
  Max: 120



================================================
FILE: examples/advanced_hooks_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'
require 'json'

# Example: Using typed hook inputs and outputs
# This demonstrates the new hook system with typed input/output classes:
# - PreToolUseHookInput, PostToolUseHookInput
# - PreToolUseHookSpecificOutput, PostToolUseHookSpecificOutput
# - UserPromptSubmitHookInput, UserPromptSubmitHookSpecificOutput
# - SyncHookJSONOutput for controlling hook behavior

puts "=== Advanced Hooks Example ==="
puts "Demonstrating typed hook inputs and outputs\n\n"

Async do
  # Example 1: PreToolUse hook with typed output
  puts "--- Example 1: PreToolUse Hook with Input Modification ---"

  pre_tool_hook = lambda do |input, tool_use_id, context|
    # Create a typed hook input for better structure
    hook_input = ClaudeAgentSDK::PreToolUseHookInput.new(
      tool_name: input[:tool_name],
      tool_input: input[:tool_input],
      session_id: input[:session_id],
      cwd: input[:cwd]
    )

    puts "PreToolUse hook triggered:"
    puts "  Tool: #{hook_input.tool_name}"
    puts "  Session: #{hook_input.session_id}"

    # Example: Modify Bash commands to add safety prefix
    if hook_input.tool_name == 'Bash'
      original_command = hook_input.tool_input[:command] || ''

      # Check for dangerous patterns
      if original_command.match?(/rm\s+-rf|sudo\s+rm/)
        # Create typed deny output
        output = ClaudeAgentSDK::PreToolUseHookSpecificOutput.new(
          permission_decision: 'deny',
          permission_decision_reason: 'Destructive commands are not allowed'
        )
        return output.to_h
      end

      # Modify command to be safer (example: add echo prefix for demo)
      if original_command.start_with?('echo')
        # Allow echo commands as-is
        output = ClaudeAgentSDK::PreToolUseHookSpecificOutput.new(
          permission_decision: 'allow'
        )
        return output.to_h
      end
    end

    # Default: allow
    {}
  end

  # Example 2: PostToolUse hook with context addition
  post_tool_hook = lambda do |input, tool_use_id, context|
    hook_input = ClaudeAgentSDK::PostToolUseHookInput.new(
      tool_name: input[:tool_name],
      tool_input: input[:tool_input],
      tool_response: input[:tool_response],
      session_id: input[:session_id]
    )

    puts "PostToolUse hook triggered:"
    puts "  Tool: #{hook_input.tool_name}"
    puts "  Response length: #{hook_input.tool_response&.to_s&.length || 0} chars"

    # Add context after tool execution
    output = ClaudeAgentSDK::PostToolUseHookSpecificOutput.new(
      additional_context: "Tool '#{hook_input.tool_name}' executed at #{Time.now}"
    )

    # Wrap in SyncHookJSONOutput for full control
    sync_output = ClaudeAgentSDK::SyncHookJSONOutput.new(
      continue: true,
      suppress_output: false,
      hook_specific_output: output
    )

    sync_output.to_h
  end

  # Example 3: UserPromptSubmit hook
  prompt_hook = lambda do |input, context|
    hook_input = ClaudeAgentSDK::UserPromptSubmitHookInput.new(
      prompt: input[:prompt],
      session_id: input[:session_id],
      cwd: input[:cwd]
    )

    puts "UserPromptSubmit hook triggered:"
    puts "  Prompt preview: #{hook_input.prompt[0..50]}..."

    # Add context to the prompt
    output = ClaudeAgentSDK::UserPromptSubmitHookSpecificOutput.new(
      additional_context: "User is working in: #{hook_input.cwd}"
    )

    { hookSpecificOutput: output.to_h }
  end

  # Configure hooks with timeout
  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    allowed_tools: ['Bash'],
    hooks: {
      'PreToolUse' => [
        ClaudeAgentSDK::HookMatcher.new(
          matcher: 'Bash',
          hooks: [pre_tool_hook],
          timeout: 5  # 5 second timeout for hook execution
        )
      ],
      'PostToolUse' => [
        ClaudeAgentSDK::HookMatcher.new(
          matcher: 'Bash',
          hooks: [post_tool_hook],
          timeout: 5
        )
      ],
      'UserPromptSubmit' => [
        ClaudeAgentSDK::HookMatcher.new(
          hooks: [prompt_hook]
        )
      ]
    }
  )

  client = ClaudeAgentSDK::Client.new(options: options)

  begin
    puts "\nConnecting with advanced hooks..."
    client.connect
    puts "Connected!\n"

    # Test 1: Safe command
    puts "\n=== Test 1: Safe Echo Command ==="
    client.query("Run the command: echo 'Hello from advanced hooks!'")

    client.receive_response do |msg|
      case msg
      when ClaudeAgentSDK::AssistantMessage
        msg.content.each do |block|
          case block
          when ClaudeAgentSDK::TextBlock
            puts "Claude: #{block.text}"
          when ClaudeAgentSDK::ToolUseBlock
            puts "Tool: #{block.name}"
          end
        end
      when ClaudeAgentSDK::ResultMessage
        puts "\nCompleted in #{msg.num_turns} turns"
      end
    end

    puts "\n" + "-" * 40

    # Test 2: Blocked command
    puts "\n=== Test 2: Blocked Dangerous Command ==="
    client.query("Run the command: rm -rf /tmp/test")

    client.receive_response do |msg|
      case msg
      when ClaudeAgentSDK::AssistantMessage
        msg.content.each do |block|
          puts "Claude: #{block.text}" if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      when ClaudeAgentSDK::SystemMessage
        puts "System: #{msg.subtype}"
        puts "  #{msg.data[:message]}" if msg.data[:message]
      when ClaudeAgentSDK::ResultMessage
        puts "\nCompleted"
      end
    end

  ensure
    puts "\nDisconnecting..."
    client.disconnect
    puts "Done!"
  end
end.wait

puts "\n" + "=" * 50 + "\n"

# Example 4: Using HookContext with signal
puts "\n--- Example 4: Hook with Context Signal ---"
puts "HookContext provides access to abort signals for long-running hooks\n"

hook_with_context = lambda do |input, tool_use_id, context|
  # HookContext provides signal for cooperative cancellation
  hook_context = ClaudeAgentSDK::HookContext.new(signal: context[:signal])

  # In a real scenario, you might check hook_context.signal periodically
  # during long-running operations to allow graceful cancellation

  puts "Hook context signal available: #{!hook_context.signal.nil?}"

  # Return allow decision
  ClaudeAgentSDK::PreToolUseHookSpecificOutput.new(
    permission_decision: 'allow'
  ).to_h
end

puts "Hook with context signal defined (for demonstration)"
puts "In production, use context.signal to support cooperative cancellation"



================================================
FILE: examples/budget_control_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Example: Using max_budget_usd for spending control
# This feature allows you to set a spending cap in dollars for your queries.
# Useful for preventing runaway costs in automated systems.

puts "=== Budget Control Example ==="
puts "Demonstrating max_budget_usd spending cap\n\n"

# Example 1: Simple budget-limited query
puts "--- Example 1: Basic Budget Limit ---"
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  max_budget_usd: 0.10,  # Cap spending at $0.10
  max_turns: 3
)

ClaudeAgentSDK.query(
  prompt: "Explain the concept of recursion in programming. Be concise.",
  options: options
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    message.content.each do |block|
      puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
    end
  when ClaudeAgentSDK::ResultMessage
    puts "\n--- Cost Summary ---"
    puts "Total cost: $#{message.total_cost_usd}" if message.total_cost_usd
    puts "Budget limit: $0.10"
    puts "Turns used: #{message.num_turns}"
    puts "Is error: #{message.is_error}"
  end
end

puts "\n" + "=" * 50 + "\n"

# Example 2: Client with persistent budget settings
puts "\n--- Example 2: Client with Budget Control ---"

Async do
  budget_options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    max_budget_usd: 0.25,  # Cap at $0.25 per session
    system_prompt: "You are a helpful assistant. Keep responses brief to minimize costs."
  )

  client = ClaudeAgentSDK::Client.new(options: budget_options)

  begin
    puts "Connecting with budget-controlled client..."
    puts "Budget limit: $0.25"
    client.connect
    puts "Connected!\n"

    total_spent = 0.0

    # First query
    puts "\n--- Query 1 ---"
    client.query("What is the capital of France?")

    client.receive_response do |msg|
      case msg
      when ClaudeAgentSDK::AssistantMessage
        msg.content.each do |block|
          puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      when ClaudeAgentSDK::ResultMessage
        cost = msg.total_cost_usd || 0
        total_spent += cost
        puts "\nQuery cost: $#{cost}"
        puts "Running total: $#{total_spent.round(4)}"
      end
    end

    # Second query
    puts "\n--- Query 2 ---"
    client.query("What is the capital of Germany?")

    client.receive_response do |msg|
      case msg
      when ClaudeAgentSDK::AssistantMessage
        msg.content.each do |block|
          puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      when ClaudeAgentSDK::ResultMessage
        cost = msg.total_cost_usd || 0
        total_spent += cost
        puts "\nQuery cost: $#{cost}"
        puts "Running total: $#{total_spent.round(4)}"
      end
    end

    puts "\n--- Session Summary ---"
    puts "Total spent: $#{total_spent.round(4)}"
    puts "Budget remaining: $#{(0.25 - total_spent).round(4)}"

  ensure
    puts "\nDisconnecting..."
    client.disconnect
    puts "Done!"
  end
end.wait

puts "\n" + "=" * 50 + "\n"

# Example 3: Very low budget for demonstration
puts "\n--- Example 3: Very Low Budget ---"
puts "Setting a very low budget to see how the system handles it"

low_budget_options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  max_budget_usd: 0.001,  # Very low: $0.001
  max_turns: 1
)

ClaudeAgentSDK.query(
  prompt: "Say hello.",
  options: low_budget_options
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    if message.error
      puts "Error type: #{message.error}"
    end
    message.content.each do |block|
      puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
    end
  when ClaudeAgentSDK::ResultMessage
    puts "\nCost: $#{message.total_cost_usd}" if message.total_cost_usd
    puts "Is error: #{message.is_error}"
  end
end



================================================
FILE: examples/client_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Example: Interactive client usage
Async do
  client = ClaudeAgentSDK::Client.new

  begin
    puts "Connecting to Claude..."
    client.connect
    puts "Connected!\n"

    # First query
    puts "=== Query 1: What is Ruby? ==="
    client.query("What is Ruby programming language in one sentence?")

    client.receive_response do |msg|
      if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
        msg.content.each do |block|
          puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      elsif msg.is_a?(ClaudeAgentSDK::ResultMessage)
        puts "\nCost: $#{msg.total_cost_usd}\n" if msg.total_cost_usd
      end
    end

    # Second query
    puts "\n=== Query 2: What is Python? ==="
    client.query("What is Python programming language in one sentence?")

    client.receive_response do |msg|
      if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
        msg.content.each do |block|
          puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      elsif msg.is_a?(ClaudeAgentSDK::ResultMessage)
        puts "\nCost: $#{msg.total_cost_usd}\n" if msg.total_cost_usd
      end
    end

  ensure
    puts "\nDisconnecting..."
    client.disconnect
    puts "Disconnected!"
  end
end.wait



================================================
FILE: examples/error_handling_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Example: Handling errors with AssistantMessage.error
# The error field can contain values like:
# - authentication_failed: API key or auth issue
# - billing_error: Account billing problem
# - rate_limit: Too many requests
# - invalid_request: Malformed request
# - server_error: Claude server issue
# - unknown: Unclassified error

puts "=== Error Handling Example ==="
puts "Demonstrating AssistantMessage.error field handling\n\n"

# Helper to display error information
def display_error(error_type)
  case error_type
  when 'authentication_failed'
    puts "  -> Authentication failed. Check your API key."
    puts "     Action: Verify ANTHROPIC_API_KEY environment variable"
  when 'billing_error'
    puts "  -> Billing error. Check your account status."
    puts "     Action: Visit console.anthropic.com to check billing"
  when 'rate_limit'
    puts "  -> Rate limited. Too many requests."
    puts "     Action: Implement exponential backoff and retry"
  when 'invalid_request'
    puts "  -> Invalid request format."
    puts "     Action: Check request parameters"
  when 'server_error'
    puts "  -> Claude server error."
    puts "     Action: Retry after a short delay"
  when 'unknown'
    puts "  -> Unknown error occurred."
    puts "     Action: Check logs for details"
  else
    puts "  -> Unhandled error type: #{error_type}"
  end
end

# Example 1: Basic error checking
puts "--- Example 1: Basic Error Checking ---"

options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  max_turns: 1
)

ClaudeAgentSDK.query(
  prompt: "Hello, how are you?",
  options: options
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    if message.error
      puts "Error detected: #{message.error}"
      display_error(message.error)
    else
      message.content.each do |block|
        puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
      end
    end
  when ClaudeAgentSDK::ResultMessage
    if message.is_error
      puts "\nResult indicates error occurred"
    else
      puts "\nCompleted successfully"
    end
  end
end

puts "\n" + "=" * 50 + "\n"

# Example 2: Error handling with retry logic
puts "\n--- Example 2: Error Handling with Retry ---"

def query_with_retry(prompt, max_retries: 3, base_delay: 1)
  retries = 0
  success = false

  while retries < max_retries && !success
    puts "Attempt #{retries + 1}/#{max_retries}..."

    ClaudeAgentSDK.query(prompt: prompt) do |message|
      case message
      when ClaudeAgentSDK::AssistantMessage
        if message.error
          case message.error
          when 'rate_limit'
            delay = base_delay * (2**retries)
            puts "Rate limited. Waiting #{delay}s before retry..."
            sleep(delay)
            retries += 1
          when 'server_error'
            delay = base_delay * (2**retries)
            puts "Server error. Waiting #{delay}s before retry..."
            sleep(delay)
            retries += 1
          when 'authentication_failed', 'billing_error'
            puts "Non-retryable error: #{message.error}"
            display_error(message.error)
            return false
          else
            puts "Error: #{message.error}"
            retries += 1
          end
        else
          message.content.each do |block|
            puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
          end
        end
      when ClaudeAgentSDK::ResultMessage
        success = !message.is_error
      end
    end
  end

  success
end

result = query_with_retry("What is 1 + 1?")
puts result ? "Query succeeded!" : "Query failed after retries"

puts "\n" + "=" * 50 + "\n"

# Example 3: Client with comprehensive error handling
puts "\n--- Example 3: Client with Comprehensive Error Handling ---"

Async do
  client = ClaudeAgentSDK::Client.new

  begin
    puts "Connecting..."
    client.connect
    puts "Connected!\n"

    queries = [
      "What is Ruby?",
      "What is Python?",
      "What is JavaScript?"
    ]

    queries.each_with_index do |query, idx|
      puts "\n--- Query #{idx + 1}: #{query} ---"

      begin
        client.query(query)

        client.receive_response do |msg|
          case msg
          when ClaudeAgentSDK::AssistantMessage
            if msg.error
              puts "Error in response: #{msg.error}"
              display_error(msg.error)

              # Decide whether to continue based on error type
              case msg.error
              when 'authentication_failed', 'billing_error'
                puts "Fatal error - stopping further queries"
                raise "Fatal API error: #{msg.error}"
              when 'rate_limit'
                puts "Will retry after delay..."
                sleep(2)
              end
            else
              msg.content.each do |block|
                puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
              end
            end
          when ClaudeAgentSDK::ResultMessage
            if msg.is_error
              puts "Query #{idx + 1} completed with error"
            else
              puts "Query #{idx + 1} completed successfully"
              puts "Cost: $#{msg.total_cost_usd}" if msg.total_cost_usd
            end
          end
        end
      rescue StandardError => e
        puts "Exception during query: #{e.message}"
        break
      end
    end

  rescue StandardError => e
    puts "Client error: #{e.message}"
  ensure
    puts "\nDisconnecting..."
    client.disconnect
    puts "Done!"
  end
end.wait

puts "\n" + "=" * 50 + "\n"

# Example 4: Error type constants reference
puts "\n--- Error Type Reference ---"
puts "Available error types (from ASSISTANT_MESSAGE_ERRORS):"
ClaudeAgentSDK::ASSISTANT_MESSAGE_ERRORS.each do |error_type|
  puts "  - #{error_type}"
end

puts "\nUsage pattern:"
puts <<~EXAMPLE
  ClaudeAgentSDK.query(prompt: "...") do |message|
    if message.is_a?(ClaudeAgentSDK::AssistantMessage) && message.error
      case message.error
      when 'rate_limit'
        # Handle rate limiting
      when 'authentication_failed'
        # Handle auth errors
      # ... etc
      end
    end
  end
EXAMPLE



================================================
FILE: examples/extended_thinking_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'

# Example: Using max_thinking_tokens for extended thinking
# This feature allows Claude to use extended thinking for complex reasoning tasks.
# Extended thinking gives Claude more "thinking time" before responding.
#
# NOTE: This option is defined in ClaudeAgentOptions for API parity with Python SDK,
# but is not yet supported by the Claude Code CLI. This example demonstrates the
# intended usage and how to handle ThinkingBlock content when the feature becomes available.

puts "=== Extended Thinking Example ==="
puts "Demonstrating max_thinking_tokens for complex reasoning\n\n"

# Example 1: Math problem with extended thinking
puts "--- Example 1: Complex Math Problem ---"
puts "Enabling extended thinking for complex calculations\n"

options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  max_thinking_tokens: 10000,  # Allow up to 10,000 tokens for thinking
  max_turns: 1
)

ClaudeAgentSDK.query(
  prompt: "Solve this step by step: If a train travels at 60 mph for 2.5 hours, " \
          "then at 80 mph for 1.75 hours, what is the total distance traveled? " \
          "Show your reasoning.",
  options: options
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    message.content.each do |block|
      case block
      when ClaudeAgentSDK::ThinkingBlock
        puts "[Thinking...]"
        # The actual thinking content can be accessed if needed
        puts "  (#{block.thinking.length} chars of reasoning)"
      when ClaudeAgentSDK::TextBlock
        puts "\nClaude: #{block.text}"
      end
    end
  when ClaudeAgentSDK::ResultMessage
    puts "\n--- Result ---"
    puts "Cost: $#{message.total_cost_usd}" if message.total_cost_usd
    puts "Turns: #{message.num_turns}"
  end
end

puts "\n" + "=" * 50 + "\n"

# Example 2: Logic puzzle with extended thinking
puts "\n--- Example 2: Logic Puzzle ---"
puts "Using extended thinking for a logic puzzle\n"

logic_options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  max_thinking_tokens: 15000,  # More thinking for complex logic
  max_turns: 1
)

ClaudeAgentSDK.query(
  prompt: "Solve this logic puzzle: Three friends - Alice, Bob, and Carol - " \
          "have different favorite colors (red, blue, green) and different pets " \
          "(dog, cat, bird). Alice doesn't like red. The person with the cat likes blue. " \
          "Carol has a bird. Bob doesn't have a dog. Who has what color and pet?",
  options: logic_options
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    thinking_shown = false
    message.content.each do |block|
      case block
      when ClaudeAgentSDK::ThinkingBlock
        unless thinking_shown
          puts "[Extended thinking enabled - Claude is reasoning through the puzzle...]"
          thinking_shown = true
        end
      when ClaudeAgentSDK::TextBlock
        puts "\nSolution:\n#{block.text}"
      end
    end
  when ClaudeAgentSDK::ResultMessage
    puts "\n--- Result ---"
    puts "Cost: $#{message.total_cost_usd}" if message.total_cost_usd
  end
end

puts "\n" + "=" * 50 + "\n"

# Example 3: Code analysis with extended thinking
puts "\n--- Example 3: Code Analysis ---"
puts "Using extended thinking for code analysis\n"

code_options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  max_thinking_tokens: 8000,
  max_turns: 1
)

code_snippet = <<~CODE
  def mystery(arr)
    return arr if arr.length <= 1
    pivot = arr[arr.length / 2]
    left = arr.select { |x| x < pivot }
    middle = arr.select { |x| x == pivot }
    right = arr.select { |x| x > pivot }
    mystery(left) + middle + mystery(right)
  end
CODE

ClaudeAgentSDK.query(
  prompt: "Analyze this Ruby code and explain what algorithm it implements, " \
          "its time complexity, and any potential improvements:\n\n#{code_snippet}",
  options: code_options
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    message.content.each do |block|
      case block
      when ClaudeAgentSDK::ThinkingBlock
        puts "[Analyzing code with extended thinking...]"
      when ClaudeAgentSDK::TextBlock
        puts "\nAnalysis:\n#{block.text}"
      end
    end
  when ClaudeAgentSDK::ResultMessage
    puts "\n--- Result ---"
    puts "Cost: $#{message.total_cost_usd}" if message.total_cost_usd
  end
end



================================================
FILE: examples/fallback_model_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Example: Using fallback_model for reliability
# This feature allows you to specify a backup model to use if the primary
# model is unavailable (e.g., due to rate limits or capacity issues).

puts "=== Fallback Model Example ==="
puts "Demonstrating fallback_model for improved reliability\n\n"

# Example 1: Basic fallback configuration
puts "--- Example 1: Primary with Fallback ---"
puts "Primary: claude-sonnet-4-20250514"
puts "Fallback: claude-3-5-haiku-20241022\n"

options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  model: 'claude-sonnet-4-20250514',
  fallback_model: 'claude-3-5-haiku-20241022',
  max_turns: 1
)

ClaudeAgentSDK.query(
  prompt: "What model are you? Please identify yourself briefly.",
  options: options
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    puts "Model used: #{message.model}"
    message.content.each do |block|
      puts "Response: #{block.text}" if block.is_a?(ClaudeAgentSDK::TextBlock)
    end
  when ClaudeAgentSDK::ResultMessage
    puts "\nCost: $#{message.total_cost_usd}" if message.total_cost_usd
  end
end

puts "\n" + "=" * 50 + "\n"

# Example 2: Client with fallback for multi-query session
puts "\n--- Example 2: Session with Fallback Model ---"

Async do
  session_options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    model: 'claude-sonnet-4-20250514',
    fallback_model: 'claude-3-5-haiku-20241022',
    system_prompt: "You are a helpful assistant. Always mention which model you are at the start."
  )

  client = ClaudeAgentSDK::Client.new(options: session_options)

  begin
    puts "Connecting with fallback model configuration..."
    client.connect
    puts "Connected!\n"

    queries = [
      "What is 2 + 2?",
      "Name a color.",
      "What is Ruby?"
    ]

    queries.each_with_index do |query, idx|
      puts "\n--- Query #{idx + 1}: #{query} ---"
      client.query(query)

      client.receive_response do |msg|
        case msg
        when ClaudeAgentSDK::AssistantMessage
          puts "Model: #{msg.model}"
          msg.content.each do |block|
            puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
          end
        when ClaudeAgentSDK::ResultMessage
          puts "Cost: $#{msg.total_cost_usd}" if msg.total_cost_usd
        end
      end
    end

  ensure
    puts "\nDisconnecting..."
    client.disconnect
    puts "Done!"
  end
end.wait

puts "\n" + "=" * 50 + "\n"

# Example 3: Different fallback strategies
puts "\n--- Example 3: Fallback Strategy Patterns ---"

# Strategy 1: Fast fallback (use cheaper model as backup)
fast_fallback = ClaudeAgentSDK::ClaudeAgentOptions.new(
  model: 'claude-sonnet-4-20250514',
  fallback_model: 'claude-3-5-haiku-20241022',  # Cheaper, faster
  max_turns: 1
)
puts "Strategy 1: Sonnet -> Haiku (cost optimization)"

# Strategy 2: Quality fallback (use similar-tier model)
quality_fallback = ClaudeAgentSDK::ClaudeAgentOptions.new(
  model: 'claude-opus-4-20250514',
  fallback_model: 'claude-sonnet-4-20250514',  # Still high quality
  max_turns: 1
)
puts "Strategy 2: Opus -> Sonnet (quality preservation)"

# Demonstrate with fast fallback
puts "\nRunning with fast fallback strategy..."
ClaudeAgentSDK.query(
  prompt: "Briefly explain what a fallback model is.",
  options: fast_fallback
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    puts "Model: #{message.model}"
    message.content.each do |block|
      puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
    end
  when ClaudeAgentSDK::ResultMessage
    puts "\nCompleted successfully"
  end
end



================================================
FILE: examples/hooks_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Example: Using hooks to control tool execution
Async do
  # Define a hook that blocks dangerous bash commands
  bash_hook = lambda do |input, tool_use_id, context|
    tool_name = input[:tool_name]
    tool_input = input[:tool_input]

    return {} unless tool_name == 'Bash'

    command = tool_input[:command] || ''
    block_patterns = ['rm -rf', 'foo.sh']

    block_patterns.each do |pattern|
      if command.include?(pattern)
        return {
          hookSpecificOutput: {
            hookEventName: 'PreToolUse',
            permissionDecision: 'deny',
            permissionDecisionReason: "Command contains forbidden pattern: #{pattern}"
          }
        }
      end
    end

    {} # Allow if no patterns match
  end

  # Create options with hook
  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    allowed_tools: ['Bash'],
    hooks: {
      'PreToolUse' => [
        ClaudeAgentSDK::HookMatcher.new(
          matcher: 'Bash',
          hooks: [bash_hook]
        )
      ]
    }
  )

  client = ClaudeAgentSDK::Client.new(options: options)

  begin
    puts "Connecting with hook-enabled client..."
    client.connect
    puts "Connected!\n"

    # Test 1: Command with forbidden pattern (will be blocked)
    puts "=== Test 1: Forbidden Command ==="
    puts "Asking Claude to run: ./foo.sh --help"
    client.query("Run the bash command: ./foo.sh --help")

    client.receive_response do |msg|
      if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
        msg.content.each do |block|
          case block
          when ClaudeAgentSDK::TextBlock
            puts "Claude: #{block.text}"
          when ClaudeAgentSDK::ToolUseBlock
            puts "Tool attempted: #{block.name}"
          end
        end
      elsif msg.is_a?(ClaudeAgentSDK::SystemMessage)
        puts "System: #{msg.subtype} - #{msg.data[:message]}" if msg.data[:message]
      end
    end

    puts "\n#{'=' * 50}\n"

    # Test 2: Safe command (should work)
    puts "=== Test 2: Safe Command ==="
    puts "Asking Claude to run: echo 'Hello from hooks example!'"
    client.query("Run the bash command: echo 'Hello from hooks example!'")

    client.receive_response do |msg|
      if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
        msg.content.each do |block|
          case block
          when ClaudeAgentSDK::TextBlock
            puts "Claude: #{block.text}"
          when ClaudeAgentSDK::ToolUseBlock
            puts "Tool used: #{block.name}"
          end
        end
      elsif msg.is_a?(ClaudeAgentSDK::ResultMessage)
        puts "\nCompleted successfully!"
      end
    end

  ensure
    puts "\nDisconnecting..."
    client.disconnect
    puts "Done!"
  end
end.wait



================================================
FILE: examples/http_mcp_server_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

# Example: Using HTTP-based MCP Servers
#
# This example demonstrates how to configure and use HTTP-based MCP servers
# with the Claude Agent SDK. HTTP MCP servers are useful for:
# - Connecting to remote tool services (APIs, databases, etc.)
# - Using shared MCP servers across multiple agents
# - Integrating with existing HTTP-based tooling infrastructure
#
# Supported MCP server types:
# - McpStdioServerConfig: Local subprocess servers
# - McpSSEServerConfig: Server-Sent Events servers
# - McpHttpServerConfig: HTTP/REST servers
# - McpSdkServerConfig: In-process SDK servers

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Example: Configure multiple MCP servers
def build_mcp_servers
  servers = {}

  # HTTP-based MCP server (e.g., remote API toolbox)
  # This connects to a remote MCP server over HTTP
  servers['remote_api'] = ClaudeAgentSDK::McpHttpServerConfig.new(
    url: 'https://api.example.com/mcp',
    headers: {
      'Authorization' => "Bearer #{ENV['API_TOKEN'] || 'demo_token'}",
      'X-Request-ID' => SecureRandom.uuid
    }
  ).to_h

  # SSE-based MCP server (for real-time streaming tools)
  servers['streaming_tools'] = ClaudeAgentSDK::McpSSEServerConfig.new(
    url: 'https://sse.example.com/mcp/events',
    headers: {
      'Authorization' => "Bearer #{ENV['SSE_TOKEN'] || 'demo_token'}"
    }
  ).to_h

  # Stdio-based MCP server (local subprocess)
  # Useful for local tools or CLI-based services
  servers['local_tools'] = ClaudeAgentSDK::McpStdioServerConfig.new(
    command: 'npx',
    args: ['-y', '@anthropic/mcp-server-example'],
    env: { 'DEBUG' => 'true' }
  ).to_h

  servers
end

# Example: In-process SDK MCP server with custom tools
def build_sdk_mcp_server
  # Create custom tools
  weather_tool = ClaudeAgentSDK.create_tool(
    'get_weather',
    'Get current weather for a city',
    {
      type: 'object',
      properties: {
        city: {
          type: 'string',
          description: 'City name (e.g., "San Francisco")'
        },
        units: {
          type: 'string',
          enum: %w[celsius fahrenheit],
          description: 'Temperature units'
        }
      },
      required: ['city']
    }
  ) do |args|
    city = args[:city] || args['city']
    units = args[:units] || args['units'] || 'celsius'

    # Simulated weather data (would call real API in production)
    temp = rand(15..30)
    temp = (temp * 9 / 5) + 32 if units == 'fahrenheit'

    {
      city: city,
      temperature: temp,
      units: units,
      conditions: %w[sunny cloudy rainy].sample,
      humidity: rand(30..80)
    }.to_json
  end

  database_tool = ClaudeAgentSDK.create_tool(
    'query_database',
    'Execute a read-only database query',
    {
      type: 'object',
      properties: {
        query: {
          type: 'string',
          description: 'SQL SELECT query to execute'
        },
        database: {
          type: 'string',
          enum: %w[users products orders],
          description: 'Database to query'
        }
      },
      required: %w[query database]
    }
  ) do |args|
    query = args[:query] || args['query']
    database = args[:database] || args['database']

    # Simulated query result (would execute real query in production)
    # IMPORTANT: Always validate and sanitize queries in production!
    if query.downcase.include?('select')
      {
        database: database,
        query: query,
        rows: [
          { id: 1, name: 'Example Row 1' },
          { id: 2, name: 'Example Row 2' }
        ],
        row_count: 2
      }.to_json
    else
      { error: 'Only SELECT queries are allowed' }.to_json
    end
  end

  # Create the SDK MCP server
  ClaudeAgentSDK.create_sdk_mcp_server(
    name: 'custom_tools',
    version: '1.0.0',
    tools: [weather_tool, database_tool]
  )
end

# Example: Agent with multiple MCP servers
class MultiToolAgent
  def initialize
    @sdk_server = build_sdk_mcp_server
  end

  def query(prompt)
    # Combine HTTP servers with SDK server
    mcp_servers = build_mcp_servers
    mcp_servers['custom_tools'] = ClaudeAgentSDK::McpSdkServerConfig.new(
      name: 'custom_tools',
      instance: @sdk_server
    ).to_h

    options = ClaudeAgentSDK::ClaudeAgentOptions.new(
      system_prompt: {
        type: 'preset',
        preset: 'claude_code',
        append: <<~PROMPT
          You have access to multiple tool servers:

          1. remote_api - Remote API tools for external services
          2. streaming_tools - Real-time streaming tools
          3. local_tools - Local subprocess tools
          4. custom_tools - Custom Ruby-based tools:
             - get_weather: Get weather for a city
             - query_database: Query a database

          Use these tools to help answer questions.
        PROMPT
      },
      mcp_servers: mcp_servers,
      permission_mode: 'bypassPermissions',
      setting_sources: []
    )

    Async do
      client = ClaudeAgentSDK::Client.new(options: options)
      response = nil

      begin
        client.connect
        client.query(prompt)

        client.receive_response do |message|
          case message
          when ClaudeAgentSDK::AssistantMessage
            message.content.each do |block|
              case block
              when ClaudeAgentSDK::TextBlock
                puts "[Text] #{block.text}"
              when ClaudeAgentSDK::ToolUseBlock
                puts "[Tool Call] #{block.name}: #{block.input.to_json}"
              when ClaudeAgentSDK::ToolResultBlock
                puts "[Tool Result] #{block.content}"
              end
            end

          when ClaudeAgentSDK::ResultMessage
            response = message.result
            puts "\n[Complete] Cost: $#{message.total_cost_usd}"
          end
        end

        response

      ensure
        client.disconnect
      end
    end.wait
  end
end

# Demo: Using the multi-tool agent
if __FILE__ == $PROGRAM_NAME
  puts "=== HTTP MCP Server Example ===\n\n"

  puts "Note: This example uses simulated MCP servers."
  puts "In production, configure real server URLs and tokens.\n\n"

  agent = MultiToolAgent.new

  # Example 1: Weather query (uses custom SDK tool)
  puts "--- Query 1: Weather ---"
  puts "User: What's the weather in San Francisco?\n"
  agent.query("What's the weather in San Francisco? Use the get_weather tool.")

  puts "\n--- Query 2: Database ---"
  puts "User: How many users are in the database?\n"
  agent.query("Query the users database to count records. Use the query_database tool with 'SELECT COUNT(*) FROM users'.")

  puts "\n=== Example Complete ==="
  puts "\nMCP Server Types Available:"
  puts "  - McpHttpServerConfig: HTTP/REST servers"
  puts "  - McpSSEServerConfig: Server-Sent Events servers"
  puts "  - McpStdioServerConfig: Local subprocess servers"
  puts "  - McpSdkServerConfig: In-process Ruby servers"
end



================================================
FILE: examples/mcp_calculator.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Example: Calculator MCP Server
#
# This example demonstrates how to create an in-process MCP server with
# calculator tools using the Claude Agent SDK for Ruby.
#
# Unlike external MCP servers that require separate processes, this server
# runs directly within your Ruby application, providing better performance
# and simpler deployment.

# Define calculator tools

add_tool = ClaudeAgentSDK.create_tool('add', 'Add two numbers', { a: :number, b: :number }) do |args|
  result = args[:a] + args[:b]
  { content: [{ type: 'text', text: "#{args[:a]} + #{args[:b]} = #{result}" }] }
end

subtract_tool = ClaudeAgentSDK.create_tool('subtract', 'Subtract one number from another', { a: :number, b: :number }) do |args|
  result = args[:a] - args[:b]
  { content: [{ type: 'text', text: "#{args[:a]} - #{args[:b]} = #{result}" }] }
end

multiply_tool = ClaudeAgentSDK.create_tool('multiply', 'Multiply two numbers', { a: :number, b: :number }) do |args|
  result = args[:a] * args[:b]
  { content: [{ type: 'text', text: "#{args[:a]} × #{args[:b]} = #{result}" }] }
end

divide_tool = ClaudeAgentSDK.create_tool('divide', 'Divide one number by another', { a: :number, b: :number }) do |args|
  if args[:b] == 0
    { content: [{ type: 'text', text: 'Error: Division by zero is not allowed' }], is_error: true }
  else
    result = args[:a] / args[:b]
    { content: [{ type: 'text', text: "#{args[:a]} ÷ #{args[:b]} = #{result}" }] }
  end
end

sqrt_tool = ClaudeAgentSDK.create_tool('sqrt', 'Calculate square root', { n: :number }) do |args|
  n = args[:n]
  if n < 0
    { content: [{ type: 'text', text: "Error: Cannot calculate square root of negative number #{n}" }], is_error: true }
  else
    result = Math.sqrt(n)
    { content: [{ type: 'text', text: "√#{n} = #{result}" }] }
  end
end

power_tool = ClaudeAgentSDK.create_tool('power', 'Raise a number to a power', { base: :number, exponent: :number }) do |args|
  result = args[:base]**args[:exponent]
  { content: [{ type: 'text', text: "#{args[:base]}^#{args[:exponent]} = #{result}" }] }
end

# Helper to display messages
def display_message(msg)
  case msg
  when ClaudeAgentSDK::UserMessage
    msg.content.each do |block|
      case block
      when ClaudeAgentSDK::TextBlock
        puts "User: #{block.text}"
      when ClaudeAgentSDK::ToolResultBlock
        content_preview = block.content.to_s[0...100]
        puts "Tool Result: #{content_preview}..."
      end
    end
  when ClaudeAgentSDK::AssistantMessage
    msg.content.each do |block|
      case block
      when ClaudeAgentSDK::TextBlock
        puts "Claude: #{block.text}"
      when ClaudeAgentSDK::ToolUseBlock
        puts "Using tool: #{block.name}"
        puts "  Input: #{block.input}" if block.input
      end
    end
  when ClaudeAgentSDK::ResultMessage
    puts "Result ended"
    puts "Cost: $#{format('%.6f', msg.total_cost_usd)}" if msg.total_cost_usd
  end
end

# Main example
Async do
  # Create the calculator server with all tools
  calculator = ClaudeAgentSDK.create_sdk_mcp_server(
    name: 'calculator',
    version: '2.0.0',
    tools: [
      add_tool,
      subtract_tool,
      multiply_tool,
      divide_tool,
      sqrt_tool,
      power_tool
    ]
  )

  # Configure Claude to use the calculator server with allowed tools
  # Pre-approve all calculator MCP tools so they can be used without permission prompts
  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    mcp_servers: { calc: calculator },
    allowed_tools: [
      'mcp__calc__add',
      'mcp__calc__subtract',
      'mcp__calc__multiply',
      'mcp__calc__divide',
      'mcp__calc__sqrt',
      'mcp__calc__power'
    ]
  )

  # Example prompts to demonstrate calculator usage
  prompts = [
    'List your tools',
    'Calculate 15 + 27',
    'What is 100 divided by 7?',
    'Calculate the square root of 144',
    'What is 2 raised to the power of 8?',
    'Calculate (12 + 8) * 3 - 10' # Complex calculation
  ]

  prompts.each do |prompt|
    puts "\n#{'=' * 50}"
    puts "Prompt: #{prompt}"
    puts '=' * 50

    client = ClaudeAgentSDK::Client.new(options: options)
    begin
      client.connect

      client.query(prompt)

      client.receive_response do |message|
        display_message(message)
      end
    ensure
      client.disconnect
    end
  end
end.wait



================================================
FILE: examples/mcp_resources_prompts_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'
require 'json'

# Example demonstrating MCP Resources and Prompts

# Example 1: Create resources for configuration and data
def example_resources
  puts "=" * 80
  puts "Example 1: MCP Resources"
  puts "=" * 80
  puts

  # Create a configuration resource
  config_resource = ClaudeAgentSDK.create_resource(
    uri: 'config://app/settings',
    name: 'Application Settings',
    description: 'Current application configuration',
    mime_type: 'application/json'
  ) do
    config_data = {
      app_name: 'MyApp',
      version: '1.0.0',
      debug_mode: false,
      max_connections: 100
    }

    {
      contents: [{
        uri: 'config://app/settings',
        mimeType: 'application/json',
        text: JSON.pretty_generate(config_data)
      }]
    }
  end

  # Create a status resource
  status_resource = ClaudeAgentSDK.create_resource(
    uri: 'status://system',
    name: 'System Status',
    description: 'Current system status and metrics',
    mime_type: 'text/plain'
  ) do
    uptime = `uptime`.strip rescue 'N/A'
    memory = `free -h 2>/dev/null | grep Mem`.strip rescue 'N/A'

    status_text = <<~STATUS
      System Status Report
      ===================
      Uptime: #{uptime}
      Memory: #{memory}
      Ruby Version: #{RUBY_VERSION}
      Time: #{Time.now}
    STATUS

    {
      contents: [{
        uri: 'status://system',
        mimeType: 'text/plain',
        text: status_text
      }]
    }
  end

  # Create a data resource
  data_resource = ClaudeAgentSDK.create_resource(
    uri: 'data://users/count',
    name: 'User Count',
    description: 'Total number of users in the system'
  ) do
    # Simulate fetching from database
    user_count = 1234

    {
      contents: [{
        uri: 'data://users/count',
        mimeType: 'text/plain',
        text: user_count.to_s
      }]
    }
  end

  # Create server with resources
  server = ClaudeAgentSDK.create_sdk_mcp_server(
    name: 'app-resources',
    resources: [config_resource, status_resource, data_resource]
  )

  puts "Created MCP server with #{server[:instance].resources.length} resources:"
  server[:instance].list_resources.each do |res|
    puts "  - #{res[:name]} (#{res[:uri]})"
  end

  # Test resource reading
  puts "\nReading configuration resource:"
  config_data = server[:instance].read_resource('config://app/settings')
  puts config_data[:contents].first[:text]

  puts "\nResources can be accessed by Claude Code to get current data!"
end

# Example 2: Create prompts for common tasks
def example_prompts
  puts "\n\n"
  puts "=" * 80
  puts "Example 2: MCP Prompts"
  puts "=" * 80
  puts

  # Simple prompt without arguments
  code_review_prompt = ClaudeAgentSDK.create_prompt(
    name: 'code_review',
    description: 'Review code for best practices and suggest improvements'
  ) do |args|
    {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: 'Please review the following code for best practices, potential bugs, ' \
                  'and suggest improvements. Focus on readability, performance, and maintainability.'
          }
        }
      ]
    }
  end

  # Prompt with arguments
  git_commit_prompt = ClaudeAgentSDK.create_prompt(
    name: 'git_commit',
    description: 'Generate a git commit message',
    arguments: [
      { name: 'changes', description: 'Description of the changes made', required: true },
      { name: 'type', description: 'Type of commit (feat, fix, docs, etc.)', required: false }
    ]
  ) do |args|
    changes = args[:changes] || args['changes']
    commit_type = args[:type] || args['type'] || 'feat'

    {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: "Generate a concise git commit message for a '#{commit_type}' commit. " \
                  "Changes: #{changes}. " \
                  "Follow conventional commits format and keep the first line under 50 characters."
          }
        }
      ]
    }
  end

  # Documentation prompt
  doc_gen_prompt = ClaudeAgentSDK.create_prompt(
    name: 'generate_docs',
    description: 'Generate documentation for code',
    arguments: [
      { name: 'code', description: 'The code to document', required: true },
      { name: 'style', description: 'Documentation style (YARD, RDoc, etc.)', required: false }
    ]
  ) do |args|
    code = args[:code] || args['code']
    style = args[:style] || args['style'] || 'YARD'

    {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: "Generate comprehensive #{style} documentation for the following code. " \
                  "Include parameter descriptions, return values, examples, and any important notes.\n\n" \
                  "Code:\n#{code}"
          }
        }
      ]
    }
  end

  # Create server with prompts
  server = ClaudeAgentSDK.create_sdk_mcp_server(
    name: 'dev-prompts',
    prompts: [code_review_prompt, git_commit_prompt, doc_gen_prompt]
  )

  puts "Created MCP server with #{server[:instance].prompts.length} prompts:"
  server[:instance].list_prompts.each do |prompt|
    puts "  - #{prompt[:name]}: #{prompt[:description]}"
  end

  # Test prompt generation
  puts "\nGenerating git commit prompt with arguments:"
  commit_prompt = server[:instance].get_prompt('git_commit', { changes: 'Added new feature', type: 'feat' })
  puts "Prompt: #{commit_prompt[:messages].first[:content][:text][0..100]}..."

  puts "\nPrompts can be used by Claude Code to generate consistent responses!"
end

# Example 3: Complete server with tools, resources, and prompts
def example_complete_server
  puts "\n\n"
  puts "=" * 80
  puts "Example 3: Complete MCP Server (Tools + Resources + Prompts)"
  puts "=" * 80
  puts

  # Define a tool
  calculator_tool = ClaudeAgentSDK.create_tool(
    'calculate',
    'Perform a calculation',
    { expression: :string }
  ) do |args|
    begin
      result = eval(args[:expression]) # Note: eval is dangerous in production!
      { content: [{ type: 'text', text: "Result: #{result}" }] }
    rescue StandardError => e
      { content: [{ type: 'text', text: "Error: #{e.message}" }], is_error: true }
    end
  end

  # Define a resource
  help_resource = ClaudeAgentSDK.create_resource(
    uri: 'help://calculator',
    name: 'Calculator Help',
    description: 'Help documentation for the calculator'
  ) do
    help_text = <<~HELP
      Calculator Tool Help
      ===================

      The calculator tool can evaluate mathematical expressions.

      Examples:
      - 2 + 2
      - 10 * 5
      - (100 - 25) / 3
      - Math.sqrt(16)

      Note: Uses Ruby's eval, so any Ruby expression is valid.
    HELP

    {
      contents: [{
        uri: 'help://calculator',
        mimeType: 'text/plain',
        text: help_text
      }]
    }
  end

  # Define a prompt
  calc_prompt = ClaudeAgentSDK.create_prompt(
    name: 'solve_problem',
    description: 'Solve a mathematical problem',
    arguments: [
      { name: 'problem', description: 'The problem to solve', required: true }
    ]
  ) do |args|
    problem = args[:problem] || args['problem']

    {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: "Solve this mathematical problem step by step: #{problem}. " \
                  "Use the calculate tool to perform calculations."
          }
        }
      ]
    }
  end

  # Create complete server
  server = ClaudeAgentSDK.create_sdk_mcp_server(
    name: 'calculator-complete',
    version: '2.0.0',
    tools: [calculator_tool],
    resources: [help_resource],
    prompts: [calc_prompt]
  )

  puts "Created complete MCP server:"
  puts "  Name: #{server[:instance].name}"
  puts "  Version: #{server[:instance].version}"
  puts "  Tools: #{server[:instance].tools.length}"
  puts "  Resources: #{server[:instance].resources.length}"
  puts "  Prompts: #{server[:instance].prompts.length}"

  puts "\nThis server provides:"
  puts "  - Tools: Executable functions Claude can call"
  puts "  - Resources: Data sources Claude can read"
  puts "  - Prompts: Reusable prompt templates"

  # Test all capabilities
  puts "\nTesting tool:"
  result = server[:instance].call_tool('calculate', { expression: '2 + 2' })
  puts "  calculate('2 + 2') = #{result[:content].first[:text]}"

  puts "\nTesting resource:"
  help = server[:instance].read_resource('help://calculator')
  puts "  help resource (first 100 chars): #{help[:contents].first[:text][0..100]}..."

  puts "\nTesting prompt:"
  prompt = server[:instance].get_prompt('solve_problem', { problem: 'What is 15% of 80?' })
  puts "  solve_problem prompt (first 100 chars): #{prompt[:messages].first[:content][:text][0..100]}..."
end

# Run examples
if __FILE__ == $PROGRAM_NAME
  begin
    puts "Claude Agent SDK - MCP Resources and Prompts Examples"
    puts "=" * 80
    puts

    example_resources
    example_prompts
    example_complete_server

    puts "\n\n"
    puts "=" * 80
    puts "All examples completed successfully!"
    puts "=" * 80
  rescue StandardError => e
    puts "Error: #{e.class} - #{e.message}"
    puts e.backtrace.first(5).join("\n")
  end
end



================================================
FILE: examples/permission_callback_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Example: Using permission callbacks for dynamic tool control
Async do
  # Define a permission callback that validates tool usage
  permission_callback = lambda do |tool_name, input, context|
    puts "\n[Permission Check]"
    puts "  Tool: #{tool_name}"
    puts "  Input: #{input.inspect}"

    # Allow Read operations
    if tool_name == 'Read'
      puts "  Decision: ALLOW (reading is safe)"
      return ClaudeAgentSDK::PermissionResultAllow.new
    end

    # Block Write to sensitive files
    if tool_name == 'Write'
      file_path = input[:file_path] || input['file_path']
      if file_path && (file_path.include?('/etc/') || file_path.include?('passwd'))
        puts "  Decision: DENY (sensitive file)"
        return ClaudeAgentSDK::PermissionResultDeny.new(
          message: 'Cannot write to sensitive system files',
          interrupt: false
        )
      end
      puts "  Decision: ALLOW (safe file write)"
      return ClaudeAgentSDK::PermissionResultAllow.new
    end

    # Block dangerous bash commands
    if tool_name == 'Bash'
      command = input[:command] || input['command'] || ''
      dangerous_patterns = ['rm -rf', 'sudo', '>']

      dangerous_patterns.each do |pattern|
        if command.include?(pattern)
          puts "  Decision: DENY (dangerous pattern: #{pattern})"
          return ClaudeAgentSDK::PermissionResultDeny.new(
            message: "Command contains dangerous pattern: #{pattern}",
            interrupt: false
          )
        end
      end

      puts "  Decision: ALLOW (safe command)"
      return ClaudeAgentSDK::PermissionResultAllow.new
    end

    # Default: allow
    puts "  Decision: ALLOW (default)"
    ClaudeAgentSDK::PermissionResultAllow.new
  end

  # Create options with permission callback
  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    allowed_tools: ['Read', 'Write', 'Bash'],
    can_use_tool: permission_callback
  )

  client = ClaudeAgentSDK::Client.new(options: options)

  begin
    puts "Connecting with permission callback..."
    client.connect
    puts "Connected!\n"

    # Test 1: Safe file write (should be allowed)
    puts "\n=== Test 1: Safe File Write ==="
    client.query("Create a file called test_output.txt with the content 'Hello World'")

    client.receive_response do |msg|
      if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
        msg.content.each do |block|
          puts "Claude: #{block.text}" if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      elsif msg.is_a?(ClaudeAgentSDK::ResultMessage)
        puts "\nTest 1 completed"
      end
    end

    # Test 2: Dangerous file write (should be blocked)
    puts "\n=== Test 2: Dangerous File Write ==="
    client.query("Write to /etc/passwd")

    client.receive_response do |msg|
      if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
        msg.content.each do |block|
          puts "Claude: #{block.text}" if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      elsif msg.is_a?(ClaudeAgentSDK::ResultMessage)
        puts "\nTest 2 completed"
      end
    end

    # Test 3: Safe bash command (should be allowed)
    puts "\n=== Test 3: Safe Bash Command ==="
    client.query("Run the command: echo 'Permission callbacks work!'")

    client.receive_response do |msg|
      if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
        msg.content.each do |block|
          puts "Claude: #{block.text}" if block.is_a?(ClaudeAgentSDK::TextBlock)
        end
      elsif msg.is_a?(ClaudeAgentSDK::ResultMessage)
        puts "\nTest 3 completed"
      end
    end

  ensure
    puts "\nDisconnecting..."
    client.disconnect
    puts "Done!"
  end
end.wait



================================================
FILE: examples/quick_start.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'

# Example 1: Simple query
puts "=== Example 1: Simple Query ==="
ClaudeAgentSDK.query(prompt: "What is 2 + 2?") do |message|
  if message.is_a?(ClaudeAgentSDK::AssistantMessage)
    message.content.each do |block|
      puts "Claude: #{block.text}" if block.is_a?(ClaudeAgentSDK::TextBlock)
    end
  elsif message.is_a?(ClaudeAgentSDK::ResultMessage)
    puts "\nCost: $#{message.total_cost_usd}" if message.total_cost_usd
    puts "Turns: #{message.num_turns}"
  end
end

# Example 2: Query with options
puts "\n=== Example 2: Query with Options ==="
options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  system_prompt: "You are a helpful assistant. Keep responses brief.",
  max_turns: 1
)

ClaudeAgentSDK.query(prompt: "Tell me a joke", options: options) do |message|
  if message.is_a?(ClaudeAgentSDK::AssistantMessage)
    message.content.each do |block|
      puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
    end
  end
end

# Example 3: Using tools
puts "\n=== Example 3: Using Tools ==="
tool_options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  allowed_tools: ['Read', 'Write', 'Bash'],
  permission_mode: 'acceptEdits'
)

ClaudeAgentSDK.query(
  prompt: "Create a hello.rb file that prints 'Hello from Claude Agent SDK!'",
  options: tool_options
) do |message|
  case message
  when ClaudeAgentSDK::AssistantMessage
    message.content.each do |block|
      case block
      when ClaudeAgentSDK::TextBlock
        puts "Claude: #{block.text}"
      when ClaudeAgentSDK::ToolUseBlock
        puts "Using tool: #{block.name}"
      end
    end
  when ClaudeAgentSDK::ResultMessage
    puts "\nCompleted in #{message.num_turns} turns"
  end
end



================================================
FILE: examples/rails_actioncable_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

# Example: Rails ActionCable Integration for Real-time Streaming
#
# This example demonstrates how to integrate the Claude Agent SDK with Rails
# ActionCable to stream responses to the frontend in real-time.
#
# Usage in a Rails application:
# 1. Create an ActionCable channel (app/channels/chat_channel.rb)
# 2. Create a background job that uses this pattern
# 3. Frontend subscribes to the channel and receives streaming updates

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Simulated ActionCable broadcast (in real Rails, use ActionCable.server.broadcast)
module ChatChannel
  def self.broadcast_to(chat_id, message)
    puts "[ActionCable -> chat_#{chat_id}] #{message.to_json}"
  end

  def self.broadcast_chunk(chat_id, content:, message_id: nil)
    broadcast_to(chat_id, {
      type: 'chunk',
      content: content,
      message_id: message_id
    })
  end

  def self.broadcast_thinking(chat_id, content:, message_id: nil)
    broadcast_to(chat_id, {
      type: 'thinking',
      content: content,
      message_id: message_id
    })
  end

  def self.broadcast_tool_use(chat_id, tool_name:, tool_input:, message_id: nil)
    broadcast_to(chat_id, {
      type: 'tool_use',
      tool_name: tool_name,
      tool_input: tool_input,
      message_id: message_id
    })
  end

  def self.broadcast_complete(chat_id, content:, message_id:, duration_ms: nil, cost_usd: nil)
    broadcast_to(chat_id, {
      type: 'complete',
      content: content,
      message_id: message_id,
      duration_ms: duration_ms,
      cost_usd: cost_usd
    })
  end

  def self.broadcast_error(chat_id, error:, message_id: nil)
    broadcast_to(chat_id, {
      type: 'error',
      error: error,
      message_id: message_id
    })
  end
end

# Helper methods for extracting content from messages
module MessageExtractor
  def self.extract_text(message)
    return '' unless message.content.is_a?(Array)

    message.content
      .select { |block| block.is_a?(ClaudeAgentSDK::TextBlock) }
      .map(&:text)
      .join("\n\n")
  end

  def self.extract_thinking(message)
    return [] unless message.content.is_a?(Array)

    message.content
      .select { |block| block.is_a?(ClaudeAgentSDK::ThinkingBlock) }
      .map(&:thinking)
  end

  def self.extract_tool_uses(message)
    return [] unless message.content.is_a?(Array)

    message.content
      .select { |block| block.is_a?(ClaudeAgentSDK::ToolUseBlock) }
      .map { |block| { name: block.name, input: block.input } }
  end
end

# Simulated chat executor (would be in a Rails job or service)
class ChatExecutor
  def initialize(chat_id:, message_id:)
    @chat_id = chat_id
    @message_id = message_id
  end

  def execute(prompt, session_id: nil, resume_session_id: nil)
    options = ClaudeAgentSDK::ClaudeAgentOptions.new(
      system_prompt: {
        type: 'preset',
        preset: 'claude_code',
        append: custom_system_prompt
      },
      permission_mode: 'bypassPermissions',
      setting_sources: [],
      resume: resume_session_id # Resume existing session if provided
    )

    client = ClaudeAgentSDK::Client.new(options: options)
    result_session_id = nil
    final_content = ''

    begin
      client.connect
      client.query(prompt, session_id: session_id)

      client.receive_response do |message|
        case message
        when ClaudeAgentSDK::AssistantMessage
          # Broadcast thinking blocks (for extended thinking)
          MessageExtractor.extract_thinking(message).each do |thinking|
            ChatChannel.broadcast_thinking(@chat_id,
              content: thinking,
              message_id: @message_id
            )
          end

          # Broadcast tool uses
          MessageExtractor.extract_tool_uses(message).each do |tool|
            ChatChannel.broadcast_tool_use(@chat_id,
              tool_name: tool[:name],
              tool_input: tool[:input],
              message_id: @message_id
            )
          end

          # Broadcast text content
          text = MessageExtractor.extract_text(message)
          unless text.empty?
            ChatChannel.broadcast_chunk(@chat_id,
              content: text,
              message_id: @message_id
            )
          end

        when ClaudeAgentSDK::SystemMessage
          # Handle system events (e.g., context compaction)
          status = message.data&.dig(:status)
          if status
            ChatChannel.broadcast_to(@chat_id, {
              type: 'system',
              status: status
            })
          end

        when ClaudeAgentSDK::ResultMessage
          result_session_id = message.session_id
          final_content = message.result || ''

          ChatChannel.broadcast_complete(@chat_id,
            content: final_content,
            message_id: @message_id,
            duration_ms: message.duration_ms,
            cost_usd: message.total_cost_usd
          )
        end
      end

      { session_id: result_session_id, content: final_content }

    rescue ClaudeAgentSDK::CLINotFoundError => e
      ChatChannel.broadcast_error(@chat_id,
        error: 'Claude CLI not installed',
        message_id: @message_id
      )
      raise

    rescue ClaudeAgentSDK::ProcessError => e
      ChatChannel.broadcast_error(@chat_id,
        error: "Process error: #{e.message}",
        message_id: @message_id
      )
      raise

    ensure
      client.disconnect
    end
  end

  private

  def custom_system_prompt
    <<~PROMPT
      You are a helpful assistant integrated into a Rails application.

      Current time: #{Time.now.strftime('%Y-%m-%d %H:%M:%S')}

      Guidelines:
      - Provide clear, concise responses
      - Use markdown formatting for code blocks
      - Be helpful and professional
    PROMPT
  end
end

# Demo execution
if __FILE__ == $PROGRAM_NAME
  Async do
    puts "=== Rails ActionCable Integration Example ===\n\n"

    chat_id = 'chat_123'
    message_id = 'msg_456'

    executor = ChatExecutor.new(chat_id: chat_id, message_id: message_id)

    puts "Sending query to Claude with ActionCable streaming...\n\n"

    result = executor.execute(
      "What are the benefits of using ActionCable in Rails? Be concise."
    )

    puts "\n=== Execution Complete ==="
    puts "Session ID: #{result[:session_id]}"
    puts "Final content length: #{result[:content].length} characters"
  end.wait
end



================================================
FILE: examples/rails_background_job_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

# Example: Rails Background Job with Session Resumption
#
# This example demonstrates how to integrate the Claude Agent SDK with Rails
# background jobs (Sidekiq, Solid Queue, etc.) with session persistence for
# multi-turn conversations.
#
# Key features:
# - Session resumption for continuous conversations
# - Error handling with job retries
# - Database persistence of session state
# - MCP server integration

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Simulated ActiveRecord model for chat sessions
class ChatSession
  attr_accessor :id, :claude_session_id, :messages, :created_at, :updated_at

  @@sessions = {}

  def initialize(id:)
    @id = id
    @claude_session_id = nil
    @messages = []
    @created_at = Time.now
    @updated_at = Time.now
    @@sessions[id] = self
  end

  def self.find(id)
    @@sessions[id] || new(id: id)
  end

  def add_message(role:, content:, metadata: {})
    @messages << {
      id: "msg_#{@messages.length + 1}",
      role: role,
      content: content,
      metadata: metadata,
      created_at: Time.now
    }
    @updated_at = Time.now
    @messages.last
  end

  def update!(attributes)
    attributes.each { |k, v| send("#{k}=", v) }
    @updated_at = Time.now
  end

  def save!
    @updated_at = Time.now
    true
  end
end

# Simulated Rails Job class
class ChatAgentJob
  class << self
    attr_accessor :queue_name
  end

  self.queue_name = :claude_agents

  def self.perform_later(session_id, message_content)
    puts "[Job Enqueued] Session: #{session_id}"
    new.perform(session_id, message_content)
  end

  def perform(session_id, message_content)
    puts "[Job Started] Session: #{session_id}"

    session = ChatSession.find(session_id)

    # Add user message to history
    user_message = session.add_message(
      role: 'user',
      content: message_content
    )
    puts "[User Message] #{user_message[:id]}: #{message_content}"

    Async do
      execute_claude_query(session, message_content)
    end.wait

    puts "[Job Completed] Session: #{session_id}"

  rescue ClaudeAgentSDK::CLINotFoundError => e
    handle_cli_not_found(session, e)
    raise # Re-raise to trigger job retry

  rescue ClaudeAgentSDK::ProcessError => e
    handle_process_error(session, e)
    raise

  rescue StandardError => e
    handle_generic_error(session, e)
    raise
  end

  private

  def execute_claude_query(session, message_content)
    options = build_options(session)
    client = ClaudeAgentSDK::Client.new(options: options)

    begin
      client.connect
      puts "[Connected] Resuming: #{session.claude_session_id || 'new session'}"

      # Query without session_id when resuming (uses resume option instead)
      query_session_id = session.claude_session_id ? nil : generate_session_id
      client.query(message_content, session_id: query_session_id)

      process_response(client, session)

    ensure
      client.disconnect
      puts "[Disconnected]"
    end
  end

  def build_options(session)
    options_hash = {
      system_prompt: build_system_prompt,
      permission_mode: 'bypassPermissions',
      setting_sources: [],
      model: 'sonnet', # Use sonnet for faster responses in jobs
      max_turns: 50,   # Limit turns for safety
      cwd: Dir.pwd
    }

    # Resume existing session if available
    if session.claude_session_id
      options_hash[:resume] = session.claude_session_id
    end

    ClaudeAgentSDK::ClaudeAgentOptions.new(**options_hash)
  end

  def build_system_prompt
    {
      type: 'preset',
      preset: 'claude_code',
      append: <<~PROMPT
        You are an AI assistant running in a background job.

        Current time: #{Time.now.strftime('%Y-%m-%d %H:%M:%S %Z')}

        Important guidelines:
        - Keep responses concise as they will be stored in a database
        - Use structured output when returning data
        - Report errors clearly so they can be handled programmatically
      PROMPT
    }
  end

  def process_response(client, session)
    assistant_content = []

    client.receive_response do |message|
      case message
      when ClaudeAgentSDK::AssistantMessage
        # Collect text content
        message.content.each do |block|
          case block
          when ClaudeAgentSDK::TextBlock
            assistant_content << block.text
            puts "[Chunk] #{block.text.slice(0, 50)}..."
          when ClaudeAgentSDK::ToolUseBlock
            puts "[Tool Use] #{block.name}"
          end
        end

        # Check for rate limit or other API errors
        if message.error
          puts "[API Error] #{message.error}"
          handle_api_error(session, message.error)
        end

      when ClaudeAgentSDK::ResultMessage
        # Save the session ID for future resumption
        session.update!(claude_session_id: message.session_id)
        puts "[Session Saved] #{message.session_id}"

        # Add assistant response to history
        full_content = message.result || assistant_content.join("\n\n")
        session.add_message(
          role: 'assistant',
          content: full_content,
          metadata: {
            duration_ms: message.duration_ms,
            cost_usd: message.total_cost_usd,
            num_turns: message.num_turns,
            claude_session_id: message.session_id
          }
        )

        puts "[Response] #{full_content.slice(0, 100)}..."
        puts "[Stats] Duration: #{message.duration_ms}ms, Cost: $#{message.total_cost_usd}"
      end
    end
  end

  def generate_session_id
    "job_#{Time.now.to_i}_#{rand(10000)}"
  end

  def handle_api_error(session, error_type)
    case error_type
    when 'rate_limit'
      puts "[Rate Limited] Will retry with exponential backoff"
      # In real Rails job, use retry_on with wait time
    when 'billing_error'
      puts "[Billing Error] Check API credits"
    when 'authentication_failed'
      puts "[Auth Error] Check API key configuration"
    end
  end

  def handle_cli_not_found(session, error)
    session.add_message(
      role: 'system',
      content: 'Error: Claude CLI not installed on server',
      metadata: { error: error.message }
    )
    puts "[Error] CLI not found: #{error.message}"
  end

  def handle_process_error(session, error)
    session.add_message(
      role: 'system',
      content: "Error: Process failed - #{error.message}",
      metadata: { error: error.message }
    )
    puts "[Error] Process error: #{error.message}"
  end

  def handle_generic_error(session, error)
    session.add_message(
      role: 'system',
      content: "Error: #{error.message}",
      metadata: { error: error.class.name, backtrace: error.backtrace&.first(5) }
    )
    puts "[Error] #{error.class}: #{error.message}"
  end
end

# Demo execution showing multi-turn conversation with session resumption
if __FILE__ == $PROGRAM_NAME
  puts "=== Rails Background Job with Session Resumption ===\n\n"

  session_id = 'session_demo_001'

  # First message - creates new session
  puts "--- Turn 1: Initial Query ---"
  ChatAgentJob.perform_later(session_id, "What is 2 + 2? Just give me the number.")

  puts "\n--- Turn 2: Follow-up Query (resumes session) ---"
  ChatAgentJob.perform_later(session_id, "Now multiply that by 3")

  puts "\n--- Turn 3: Another Follow-up ---"
  ChatAgentJob.perform_later(session_id, "What was my first question?")

  # Show session history
  session = ChatSession.find(session_id)
  puts "\n=== Session History ==="
  puts "Session ID: #{session.id}"
  puts "Claude Session: #{session.claude_session_id}"
  puts "Messages: #{session.messages.length}"
  session.messages.each do |msg|
    puts "  [#{msg[:role]}] #{msg[:content].slice(0, 60)}..."
  end
end



================================================
FILE: examples/session_resumption_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

# Example: Session Resumption for Multi-turn Conversations
#
# This example demonstrates how to persist and resume Claude sessions
# for continuous multi-turn conversations. This is essential for:
# - Chat applications with ongoing conversations
# - Background jobs that need to continue previous work
# - Interactive agents that maintain context across requests
#
# Key concepts:
# - session_id: Used when creating a new session
# - resume: Used when continuing an existing session
# - The session_id from ResultMessage should be saved for future resumption

require 'bundler/setup'
require 'claude_agent_sdk'
require 'async'

# Simple in-memory session store (use Redis/Database in production)
class SessionStore
  @@sessions = {}

  def self.get(key)
    @@sessions[key]
  end

  def self.set(key, value)
    @@sessions[key] = value
  end

  def self.exists?(key)
    @@sessions.key?(key)
  end

  def self.delete(key)
    @@sessions.delete(key)
  end
end

# Conversation manager that handles session persistence
class ConversationManager
  def initialize(conversation_id)
    @conversation_id = conversation_id
    @session_key = "claude_session:#{conversation_id}"
  end

  def send_message(message)
    Async do
      claude_session_id = SessionStore.get(@session_key)

      if claude_session_id
        puts "[Resuming session: #{claude_session_id}]"
        response = resume_conversation(message, claude_session_id)
      else
        puts "[Creating new session]"
        response = start_conversation(message)
      end

      response
    end.wait
  end

  def clear_session
    SessionStore.delete(@session_key)
    puts "[Session cleared for conversation: #{@conversation_id}]"
  end

  def fork_session(message)
    # Create a new branch from current session
    Async do
      claude_session_id = SessionStore.get(@session_key)

      if claude_session_id
        puts "[Forking session: #{claude_session_id}]"
        fork_conversation(message, claude_session_id)
      else
        puts "[No session to fork, starting new]"
        start_conversation(message)
      end
    end.wait
  end

  private

  def start_conversation(message)
    options = ClaudeAgentSDK::ClaudeAgentOptions.new(
      system_prompt: conversation_system_prompt,
      permission_mode: 'bypassPermissions',
      setting_sources: []
    )

    execute_query(options, message, new_session_id: generate_session_id)
  end

  def resume_conversation(message, claude_session_id)
    options = ClaudeAgentSDK::ClaudeAgentOptions.new(
      system_prompt: conversation_system_prompt,
      permission_mode: 'bypassPermissions',
      setting_sources: [],
      resume: claude_session_id  # This resumes the existing session
    )

    # Don't pass session_id when resuming - the resume option handles it
    execute_query(options, message, new_session_id: nil)
  end

  def fork_conversation(message, claude_session_id)
    options = ClaudeAgentSDK::ClaudeAgentOptions.new(
      system_prompt: conversation_system_prompt,
      permission_mode: 'bypassPermissions',
      setting_sources: [],
      resume: claude_session_id,
      fork_session: true  # Creates a new branch instead of continuing
    )

    execute_query(options, message, new_session_id: nil)
  end

  def execute_query(options, message, new_session_id:)
    client = ClaudeAgentSDK::Client.new(options: options)
    response_text = ''
    result_session_id = nil

    begin
      client.connect
      client.query(message, session_id: new_session_id)

      client.receive_response do |msg|
        case msg
        when ClaudeAgentSDK::AssistantMessage
          msg.content.each do |block|
            if block.is_a?(ClaudeAgentSDK::TextBlock)
              response_text += block.text
            end
          end

        when ClaudeAgentSDK::ResultMessage
          result_session_id = msg.session_id
          response_text = msg.result if msg.result

          # Save session for future resumption
          SessionStore.set(@session_key, result_session_id)
          puts "[Session saved: #{result_session_id}]"
        end
      end

      {
        response: response_text,
        session_id: result_session_id
      }

    ensure
      client.disconnect
    end
  end

  def generate_session_id
    "conv_#{@conversation_id}_#{Time.now.to_i}"
  end

  def conversation_system_prompt
    {
      type: 'preset',
      preset: 'claude_code',
      append: <<~PROMPT
        You are having a multi-turn conversation.
        Remember the context from previous messages.
        Keep responses concise and relevant.
      PROMPT
    }
  end
end

# Demo: Multi-turn conversation with session resumption
if __FILE__ == $PROGRAM_NAME
  puts "=== Session Resumption Example ===\n\n"

  conversation = ConversationManager.new('demo_001')

  # Turn 1: Start conversation
  puts "--- Turn 1 ---"
  puts "User: My name is Alice and I love Ruby programming."
  result = conversation.send_message("My name is Alice and I love Ruby programming. Remember this.")
  puts "Claude: #{result[:response]}\n\n"

  # Turn 2: Reference previous context
  puts "--- Turn 2 ---"
  puts "User: What's my name and what do I love?"
  result = conversation.send_message("What's my name and what do I love?")
  puts "Claude: #{result[:response]}\n\n"

  # Turn 3: Continue conversation
  puts "--- Turn 3 ---"
  puts "User: Tell me a Ruby tip related to my interests."
  result = conversation.send_message("Tell me a Ruby tip related to my interests.")
  puts "Claude: #{result[:response]}\n\n"

  # Demo: Fork the conversation
  puts "--- Fork Demo ---"
  puts "Forking conversation to explore alternative..."
  fork_result = conversation.fork_session("Actually, let's talk about Python instead.")
  puts "Claude (forked): #{fork_result[:response]}\n\n"

  # Original conversation continues
  puts "--- Original Continues ---"
  puts "User: Back to Ruby - what's my name again?"
  result = conversation.send_message("What's my name again?")
  puts "Claude: #{result[:response]}\n\n"

  # Clear session demo
  puts "--- Clear Session ---"
  conversation.clear_session

  puts "Starting fresh conversation..."
  result = conversation.send_message("What's my name?")
  puts "Claude (no memory): #{result[:response]}\n"
end



================================================
FILE: examples/streaming_input_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'

# Example 1: Streaming input from an array of messages
def example_array_streaming
  puts "=" * 80
  puts "Example 1: Streaming Input from Array"
  puts "=" * 80
  puts

  messages = [
    "Hello! I'm going to ask you a few questions.",
    "What is 2 + 2?",
    "What is the capital of France?",
    "Thank you for your answers!"
  ]

  # Create a streaming enumerator from the array
  stream = ClaudeAgentSDK::Streaming.from_array(messages)

  # Configure options
  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    max_turns: 10
  )

  # Query with streaming input
  ClaudeAgentSDK.query(prompt: stream, options: options) do |message|
    case message
    when ClaudeAgentSDK::AssistantMessage
      puts "\n[Assistant]:"
      message.content.each do |block|
        case block
        when ClaudeAgentSDK::TextBlock
          puts block.text
        when ClaudeAgentSDK::ThinkingBlock
          puts "[Thinking: #{block.thinking[0..100]}...]" if block.thinking
        end
      end
    when ClaudeAgentSDK::UserMessage
      content = message.content.is_a?(String) ? message.content : message.content.first&.text
      puts "\n[User]: #{content}"
    when ClaudeAgentSDK::ResultMessage
      puts "\n[Result]:"
      puts "  Turns: #{message.num_turns}"
      puts "  Cost: $#{message.total_cost_usd}" if message.total_cost_usd
    end
  end
end

# Example 2: Streaming input with dynamic generation
def example_dynamic_streaming
  puts "\n\n"
  puts "=" * 80
  puts "Example 2: Dynamic Streaming Input"
  puts "=" * 80
  puts

  # Create a stream that generates messages dynamically
  stream = Enumerator.new do |yielder|
    # First message
    yielder << ClaudeAgentSDK::Streaming.user_message(
      "I'm going to send you a series of math problems."
    )

    # Generate math problems dynamically
    3.times do |i|
      a = rand(1..10)
      b = rand(1..10)
      yielder << ClaudeAgentSDK::Streaming.user_message(
        "Problem #{i + 1}: What is #{a} + #{b}?"
      )
    end

    # Final message
    yielder << ClaudeAgentSDK::Streaming.user_message(
      "Great! Thank you for solving these problems."
    )
  end

  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    max_turns: 15
  )

  ClaudeAgentSDK.query(prompt: stream, options: options) do |message|
    case message
    when ClaudeAgentSDK::AssistantMessage
      message.content.each do |block|
        if block.is_a?(ClaudeAgentSDK::TextBlock)
          puts "[Assistant]: #{block.text}"
        end
      end
    when ClaudeAgentSDK::ResultMessage
      puts "\n[Completed in #{message.num_turns} turns]"
    end
  end
end

# Example 3: Streaming with session management
def example_session_streaming
  puts "\n\n"
  puts "=" * 80
  puts "Example 3: Streaming with Session IDs"
  puts "=" * 80
  puts

  # Create streams for different sessions
  stream = Enumerator.new do |yielder|
    # Session 1: Math questions
    yielder << ClaudeAgentSDK::Streaming.user_message(
      "What is 5 + 3?",
      session_id: 'math'
    )
    yielder << ClaudeAgentSDK::Streaming.user_message(
      "What is 10 * 2?",
      session_id: 'math'
    )

    # Session 2: General questions
    yielder << ClaudeAgentSDK::Streaming.user_message(
      "What is the capital of Japan?",
      session_id: 'geography'
    )
  end

  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    max_turns: 10
  )

  ClaudeAgentSDK.query(prompt: stream, options: options) do |message|
    case message
    when ClaudeAgentSDK::AssistantMessage
      session = message.instance_variable_get(:@parent_tool_use_id) || 'default'
      message.content.each do |block|
        if block.is_a?(ClaudeAgentSDK::TextBlock)
          puts "[Session #{session}] Assistant: #{block.text[0..80]}..."
        end
      end
    when ClaudeAgentSDK::ResultMessage
      puts "\n[All sessions completed]"
    end
  end
end

# Example 4: Custom enumerator with delays
def example_timed_streaming
  puts "\n\n"
  puts "=" * 80
  puts "Example 4: Streaming with Delays (simulating user input)"
  puts "=" * 80
  puts

  # Create a stream that simulates delayed user input
  stream = Enumerator.new do |yielder|
    messages = [
      "Let's have a conversation.",
      "Tell me a short joke.",
      "That's funny! Tell me another one."
    ]

    messages.each_with_index do |msg, i|
      yielder << ClaudeAgentSDK::Streaming.user_message(msg)
      puts "[Sent message #{i + 1}]"
      sleep 0.1 if i < messages.length - 1  # Small delay between messages
    end
  end

  options = ClaudeAgentSDK::ClaudeAgentOptions.new(
    max_turns: 10
  )

  ClaudeAgentSDK.query(prompt: stream, options: options) do |message|
    if message.is_a?(ClaudeAgentSDK::AssistantMessage)
      message.content.each do |block|
        puts "\n[Assistant]: #{block.text}" if block.is_a?(ClaudeAgentSDK::TextBlock)
      end
    end
  end
end

# Run examples
if __FILE__ == $PROGRAM_NAME
  begin
    puts "Claude Agent SDK - Streaming Input Examples"
    puts "==========================================="
    puts

    # Uncomment the examples you want to run:
    example_array_streaming
    # example_dynamic_streaming
    # example_session_streaming
    # example_timed_streaming

    puts "\n\nAll examples completed successfully!"
  rescue ClaudeAgentSDK::CLINotFoundError => e
    puts "Error: #{e.message}"
    puts "\nPlease install Claude Code to run these examples."
  rescue StandardError => e
    puts "Error: #{e.class} - #{e.message}"
    puts e.backtrace.first(5).join("\n")
  end
end



================================================
FILE: examples/structured_output_example.rb
================================================
#!/usr/bin/env ruby
# frozen_string_literal: true

require 'bundler/setup'
require 'claude_agent_sdk'
require 'json'

# Structured Outputs in the Ruby SDK
#
# Get validated JSON results from agent workflows. The Agent SDK supports
# structured outputs through JSON Schemas, ensuring your agents return data
# in exactly the format you need.
#
# WHEN TO USE STRUCTURED OUTPUTS:
# Use structured outputs when you need validated JSON after an agent completes
# a multi-turn workflow with tools (file searches, command execution, etc.).
#
# WHY USE STRUCTURED OUTPUTS:
# - Validated structure: Always receive valid JSON matching your schema
# - Simplified integration: No parsing or validation code needed
# - Type safety: Use with Ruby type checkers (Sorbet, dry-types) for safety
# - Clean separation: Define output requirements separately from task instructions
# - Tool autonomy: Agent chooses which tools to use while guaranteeing output format

puts "=" * 60
puts "=== Quick Start: Company Research ==="
puts "=" * 60
puts

# Define a JSON schema for company information
schema = {
  type: 'object',
  properties: {
    company_name: { type: 'string' },
    founded_year: { type: 'number' },
    headquarters: { type: 'string' }
  },
  required: ['company_name']
}

ClaudeAgentSDK.query(
  prompt: 'Research Anthropic and provide key company information',
  options: ClaudeAgentSDK::ClaudeAgentOptions.new(
    output_format: {
      type: 'json_schema',
      schema: schema
    }
  )
) do |message|
  if message.is_a?(ClaudeAgentSDK::ResultMessage) && message.structured_output
    puts "Structured output:"
    puts JSON.pretty_generate(message.structured_output)
    # { company_name: "Anthropic", founded_year: 2021, headquarters: "San Francisco, CA" }
  end
end

puts "\n" + "=" * 60
puts "=== Example: TODO Tracking Agent ==="
puts "=" * 60
puts
puts "Agent uses Grep to find TODOs, Bash to get git blame info\n\n"

# Define structure for TODO extraction
todo_schema = {
  type: 'object',
  properties: {
    todos: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          text: { type: 'string' },
          file: { type: 'string' },
          line: { type: 'number' },
          author: { type: 'string' },
          date: { type: 'string' }
        },
        required: %w[text file line]
      }
    },
    total_count: { type: 'number' }
  },
  required: %w[todos total_count]
}

ClaudeAgentSDK.query(
  prompt: 'Find all TODO comments in this directory and identify who added them',
  options: ClaudeAgentSDK::ClaudeAgentOptions.new(
    output_format: {
      type: 'json_schema',
      schema: todo_schema
    }
  )
) do |message|
  if message.is_a?(ClaudeAgentSDK::ResultMessage) && message.structured_output
    data = message.structured_output
    puts "Found #{data[:total_count]} TODOs"
    data[:todos]&.each do |todo|
      puts "#{todo[:file]}:#{todo[:line]} - #{todo[:text]}"
      puts "  Added by #{todo[:author]} on #{todo[:date]}" if todo[:author]
    end
  end
end

puts "\n" + "=" * 60
puts "=== Example: Code Analysis with Nested Schema ==="
puts "=" * 60
puts

# More complex schema with nested objects and enums
analysis_schema = {
  type: 'object',
  properties: {
    summary: { type: 'string' },
    issues: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          severity: { type: 'string', enum: %w[low medium high] },
          description: { type: 'string' },
          file: { type: 'string' }
        },
        required: %w[severity description file]
      }
    },
    score: { type: 'number', minimum: 0, maximum: 100 }
  },
  required: %w[summary issues score]
}

ClaudeAgentSDK.query(
  prompt: 'Analyze the codebase for potential improvements',
  options: ClaudeAgentSDK::ClaudeAgentOptions.new(
    output_format: {
      type: 'json_schema',
      schema: analysis_schema
    }
  )
) do |message|
  if message.is_a?(ClaudeAgentSDK::ResultMessage) && message.structured_output
    data = message.structured_output
    puts "Score: #{data[:score]}/100"
    puts "Summary: #{data[:summary]}"
    puts "\nIssues found: #{data[:issues]&.length || 0}"
    data[:issues]&.each do |issue|
      puts "  [#{issue[:severity]&.upcase}] #{issue[:file]}: #{issue[:description]}"
    end
  end
end

puts "\n" + "=" * 60
puts "=== Error Handling ==="
puts "=" * 60
puts

# If the agent cannot produce valid output matching your schema,
# you'll receive an error result
ClaudeAgentSDK.query(
  prompt: 'What is 2+2?',
  options: ClaudeAgentSDK::ClaudeAgentOptions.new(
    output_format: {
      type: 'json_schema',
      schema: {
        type: 'object',
        properties: {
          answer: { type: 'string' }
        },
        required: ['answer']
      }
    }
  )
) do |message|
  if message.is_a?(ClaudeAgentSDK::ResultMessage)
    if message.subtype == 'success' && message.structured_output
      puts "Success: #{message.structured_output.inspect}"
    elsif message.subtype == 'error_max_structured_output_retries'
      puts "Error: Could not produce valid output"
    else
      puts "Result: #{message.subtype}"
      puts "Structured output: #{message.structured_output.inspect}" if message.structured_output
    end
  end
end

puts "\n" + "=" * 60
puts "=== Type-Safe Schemas with dry-struct (Optional) ==="
puts "=" * 60
puts <<~EXAMPLE

  For Ruby projects wanting type safety similar to Zod (TypeScript) or
  Pydantic (Python), consider using dry-struct:

  ```ruby
  require 'dry-struct'
  require 'dry-types'

  module Types
    include Dry.Types()
  end

  class Issue < Dry::Struct
    attribute :severity, Types::String.enum('low', 'medium', 'high')
    attribute :description, Types::String
    attribute :file, Types::String
  end

  class AnalysisResult < Dry::Struct
    attribute :summary, Types::String
    attribute :issues, Types::Array.of(Issue)
    attribute :score, Types::Integer.constrained(gteq: 0, lteq: 100)
  end

  # Use in query
  ClaudeAgentSDK.query(
    prompt: 'Analyze the codebase',
    options: ClaudeAgentSDK::ClaudeAgentOptions.new(
      output_format: {
        type: 'json_schema',
        schema: {
          type: 'object',
          properties: {
            summary: { type: 'string' },
            issues: {
              type: 'array',
              items: {
                type: 'object',
                properties: {
                  severity: { type: 'string', enum: ['low', 'medium', 'high'] },
                  description: { type: 'string' },
                  file: { type: 'string' }
                },
                required: ['severity', 'description', 'file']
              }
            },
            score: { type: 'integer', minimum: 0, maximum: 100 }
          },
          required: ['summary', 'issues', 'score']
        }
      }
    )
  ) do |message|
    if message.is_a?(ClaudeAgentSDK::ResultMessage) && message.structured_output
      # Validate with dry-struct
      result = AnalysisResult.new(message.structured_output)
      puts "Score: \#{result.score}"
      result.issues.each { |i| puts "[\#{i.severity}] \#{i.description}" }
    end
  end
  ```

EXAMPLE

puts "=" * 60
puts "=== Tips ==="
puts "=" * 60
puts <<~TIPS

  1. SCHEMA ROOT: JSON schema root must be 'object' type.
     Wrap arrays in an object property.

  2. ACCESS DATA: Structured output is available in:
     - message.structured_output on ResultMessage

  3. KEYS ARE SYMBOLS: JSON parser uses symbolize_names: true,
     so access data with symbols: data[:name], not data['name']

  4. RAILS INTEGRATION: The SDK works in Rails applications.
     No special configuration needed.

  5. SUPPORTED FEATURES: All basic JSON Schema types supported:
     object, array, string, integer, number, boolean, null
     Also: enum, const, required, minimum, maximum, etc.

TIPS



================================================
FILE: lib/claude_agent_sdk.rb
================================================
# frozen_string_literal: true

require_relative 'claude_agent_sdk/version'
require_relative 'claude_agent_sdk/errors'
require_relative 'claude_agent_sdk/types'
require_relative 'claude_agent_sdk/transport'
require_relative 'claude_agent_sdk/subprocess_cli_transport'
require_relative 'claude_agent_sdk/message_parser'
require_relative 'claude_agent_sdk/query'
require_relative 'claude_agent_sdk/sdk_mcp_server'
require_relative 'claude_agent_sdk/streaming'
require 'async'
require 'securerandom'

# Claude Agent SDK for Ruby
module ClaudeAgentSDK
  # Query Claude Code for one-shot or unidirectional streaming interactions
  #
  # This function is ideal for simple, stateless queries where you don't need
  # bidirectional communication or conversation management.
  #
  # @param prompt [String, Enumerator] The prompt to send to Claude, or an Enumerator for streaming input
  # @param options [ClaudeAgentOptions] Optional configuration
  # @yield [Message] Each message from the conversation
  # @return [Enumerator] if no block given
  #
  # @example Simple query
  #   ClaudeAgentSDK.query(prompt: "What is 2 + 2?") do |message|
  #     puts message
  #   end
  #
  # @example With options
  #   options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  #     allowed_tools: ['Read', 'Bash'],
  #     permission_mode: 'acceptEdits'
  #   )
  #   ClaudeAgentSDK.query(prompt: "Create a hello.rb file", options: options) do |msg|
  #     if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
  #       msg.content.each do |block|
  #         puts block.text if block.is_a?(ClaudeAgentSDK::TextBlock)
  #       end
  #     end
  #   end
  #
  # @example Streaming input
  #   messages = Streaming.from_array(['Hello', 'What is 2+2?', 'Thanks!'])
  #   ClaudeAgentSDK.query(prompt: messages) do |message|
  #     puts message
  #   end
  def self.query(prompt:, options: nil, &block)
    return enum_for(:query, prompt: prompt, options: options) unless block

    options ||= ClaudeAgentOptions.new
    ENV['CLAUDE_CODE_ENTRYPOINT'] = 'sdk-rb'

    Async do
      transport = SubprocessCLITransport.new(prompt, options)
      begin
        transport.connect

        # If prompt is an Enumerator, write each message to stdin
        if prompt.is_a?(Enumerator) || prompt.respond_to?(:each)
          Async do
            begin
              prompt.each do |message_json|
                transport.write(message_json)
              end
            ensure
              transport.end_input
            end
          end
        end

        # Read and yield messages
        transport.read_messages do |data|
          message = MessageParser.parse(data)
          block.call(message)
        end
      ensure
        transport.close
      end
    end.wait
  end

  # Client for bidirectional, interactive conversations with Claude Code
  #
  # This client provides full control over the conversation flow with support
  # for streaming, hooks, permission callbacks, and dynamic message sending.
  # The Client class always uses streaming mode for bidirectional communication.
  #
  # @example Basic usage
  #   Async do
  #     client = ClaudeAgentSDK::Client.new
  #     client.connect  # No arguments needed - automatically uses streaming mode
  #
  #     client.query("What is the capital of France?")
  #     client.receive_response do |msg|
  #       puts msg if msg.is_a?(ClaudeAgentSDK::AssistantMessage)
  #     end
  #
  #     client.disconnect
  #   end
  #
  # @example With hooks
  #   options = ClaudeAgentOptions.new(
  #     hooks: {
  #       'PreToolUse' => [
  #         HookMatcher.new(
  #           matcher: 'Bash',
  #           hooks: [
  #             ->(input, tool_use_id, context) {
  #               # Return hook output
  #               {}
  #             }
  #           ]
  #         )
  #       ]
  #     }
  #   )
  #   client = ClaudeAgentSDK::Client.new(options: options)
  class Client
    attr_reader :query_handler

    def initialize(options: nil)
      @options = options || ClaudeAgentOptions.new
      @transport = nil
      @query_handler = nil
      @connected = false
      ENV['CLAUDE_CODE_ENTRYPOINT'] = 'sdk-rb-client'
    end

    # Connect to Claude with optional initial prompt
    # @param prompt [String, Enumerator, nil] Initial prompt or message stream
    def connect(prompt = nil)
      return if @connected

      # Validate and configure permission settings
      configured_options = @options
      if @options.can_use_tool
        # can_use_tool requires streaming mode
        if prompt.is_a?(String)
          raise ArgumentError, 'can_use_tool callback requires streaming mode'
        end

        # can_use_tool and permission_prompt_tool_name are mutually exclusive
        if @options.permission_prompt_tool_name
          raise ArgumentError, 'can_use_tool callback cannot be used with permission_prompt_tool_name'
        end

        # Set permission_prompt_tool_name to stdio for control protocol
        configured_options = @options.dup_with(permission_prompt_tool_name: 'stdio')
      end

      # Auto-connect with empty enumerator if no prompt is provided
      # This matches the Python SDK pattern where ClaudeSDKClient always uses streaming mode
      # An empty enumerator keeps stdin open for bidirectional communication
      actual_prompt = prompt || [].to_enum
      @transport = SubprocessCLITransport.new(actual_prompt, configured_options)
      @transport.connect

      # Extract SDK MCP servers
      sdk_mcp_servers = {}
      if configured_options.mcp_servers.is_a?(Hash)
        configured_options.mcp_servers.each do |name, config|
          sdk_mcp_servers[name] = config[:instance] if config.is_a?(Hash) && config[:type] == 'sdk'
        end
      end

      # Convert hooks to internal format
      hooks = convert_hooks_to_internal_format(configured_options.hooks) if configured_options.hooks

      # Create Query handler
      @query_handler = Query.new(
        transport: @transport,
        is_streaming_mode: true,
        can_use_tool: configured_options.can_use_tool,
        hooks: hooks,
        sdk_mcp_servers: sdk_mcp_servers
      )

      # Start query handler and initialize
      @query_handler.start
      @query_handler.initialize_protocol

      @connected = true
    end

    # Send a query to Claude
    # @param prompt [String] The prompt to send
    # @param session_id [String] Session identifier
    def query(prompt, session_id: 'default')
      raise CLIConnectionError, 'Not connected. Call connect() first' unless @connected

      message = {
        type: 'user',
        message: { role: 'user', content: prompt },
        parent_tool_use_id: nil,
        session_id: session_id
      }
      @transport.write(JSON.generate(message) + "\n")
    end

    # Receive all messages from Claude
    # @yield [Message] Each message received
    def receive_messages(&block)
      return enum_for(:receive_messages) unless block

      raise CLIConnectionError, 'Not connected. Call connect() first' unless @connected

      @query_handler.receive_messages do |data|
        message = MessageParser.parse(data)
        block.call(message)
      end
    end

    # Receive messages until a ResultMessage is received
    # @yield [Message] Each message received
    def receive_response(&block)
      return enum_for(:receive_response) unless block

      receive_messages do |message|
        block.call(message)
        break if message.is_a?(ResultMessage)
      end
    end

    # Send interrupt signal
    def interrupt
      raise CLIConnectionError, 'Not connected. Call connect() first' unless @connected
      @query_handler.interrupt
    end

    # Change permission mode during conversation
    # @param mode [String] Permission mode ('default', 'acceptEdits', 'bypassPermissions')
    def set_permission_mode(mode)
      raise CLIConnectionError, 'Not connected. Call connect() first' unless @connected
      @query_handler.set_permission_mode(mode)
    end

    # Change the AI model during conversation
    # @param model [String, nil] Model name or nil for default
    def set_model(model)
      raise CLIConnectionError, 'Not connected. Call connect() first' unless @connected
      @query_handler.set_model(model)
    end

    # Rewind files to a previous checkpoint (v0.1.15+)
    # Restores file state to what it was at the given user message
    # Requires enable_file_checkpointing to be true in options
    # @param user_message_uuid [String] The UUID of the UserMessage to rewind to
    def rewind_files(user_message_uuid)
      raise CLIConnectionError, 'Not connected. Call connect() first' unless @connected
      @query_handler.rewind_files(user_message_uuid)
    end

    # Get server initialization info
    # @return [Hash, nil] Server info or nil
    def server_info
      @query_handler&.instance_variable_get(:@initialization_result)
    end

    # Disconnect from Claude
    def disconnect
      return unless @connected

      @query_handler&.close
      @query_handler = nil
      @transport = nil
      @connected = false
    end

    private

    def convert_hooks_to_internal_format(hooks)
      return nil unless hooks

      internal_hooks = {}
      hooks.each do |event, matchers|
        internal_hooks[event.to_s] = []
        matchers.each do |matcher|
          internal_hooks[event.to_s] << {
            matcher: matcher.matcher,
            hooks: matcher.hooks
          }
        end
      end
      internal_hooks
    end
  end
end



================================================
FILE: lib/claude_agent_sdk/errors.rb
================================================
# frozen_string_literal: true

module ClaudeAgentSDK
  # Base exception for all Claude SDK errors
  class ClaudeSDKError < StandardError; end

  # Raised when unable to connect to Claude Code
  class CLIConnectionError < ClaudeSDKError; end

  # Raised when Claude Code is not found or not installed
  class CLINotFoundError < CLIConnectionError
    def initialize(message = 'Claude Code not found', cli_path: nil)
      message = "#{message}: #{cli_path}" if cli_path
      super(message)
    end
  end

  # Raised when the CLI process fails
  class ProcessError < ClaudeSDKError
    attr_reader :exit_code, :stderr

    def initialize(message, exit_code: nil, stderr: nil)
      @exit_code = exit_code
      @stderr = stderr

      message = "#{message} (exit code: #{exit_code})" if exit_code
      message = "#{message}\nError output: #{stderr}" if stderr

      super(message)
    end
  end

  # Raised when unable to decode JSON from CLI output
  class CLIJSONDecodeError < ClaudeSDKError
    attr_reader :line, :original_error

    def initialize(line, original_error)
      @line = line
      @original_error = original_error
      super("Failed to decode JSON: #{line[0...100]}...")
    end
  end

  # Raised when unable to parse a message from CLI output
  class MessageParseError < ClaudeSDKError
    attr_reader :data

    def initialize(message, data: nil)
      @data = data
      super(message)
    end
  end
end



================================================
FILE: lib/claude_agent_sdk/message_parser.rb
================================================
# frozen_string_literal: true

require_relative 'types'
require_relative 'errors'

module ClaudeAgentSDK
  # Parse message from CLI output into typed Message objects
  class MessageParser
    def self.parse(data)
      raise MessageParseError.new("Invalid message data type", data: data) unless data.is_a?(Hash)

      message_type = data[:type]
      raise MessageParseError.new("Message missing 'type' field", data: data) unless message_type

      case message_type
      when 'user'
        parse_user_message(data)
      when 'assistant'
        parse_assistant_message(data)
      when 'system'
        parse_system_message(data)
      when 'result'
        parse_result_message(data)
      when 'stream_event'
        parse_stream_event(data)
      else
        raise MessageParseError.new("Unknown message type: #{message_type}", data: data)
      end
    rescue KeyError => e
      raise MessageParseError.new("Missing required field: #{e.message}", data: data)
    end

    def self.parse_user_message(data)
      parent_tool_use_id = data[:parent_tool_use_id]
      uuid = data[:uuid] # UUID for rewind support
      message_data = data[:message]
      raise MessageParseError.new("Missing message field in user message", data: data) unless message_data

      content = message_data[:content]
      raise MessageParseError.new("Missing content in user message", data: data) unless content

      if content.is_a?(Array)
        content_blocks = content.map { |block| parse_content_block(block) }
        UserMessage.new(content: content_blocks, uuid: uuid, parent_tool_use_id: parent_tool_use_id)
      else
        UserMessage.new(content: content, uuid: uuid, parent_tool_use_id: parent_tool_use_id)
      end
    end

    def self.parse_assistant_message(data)
      content = data.dig(:message, :content)
      raise MessageParseError.new("Missing content in assistant message", data: data) unless content

      content_blocks = content.map { |block| parse_content_block(block) }
      AssistantMessage.new(
        content: content_blocks,
        model: data.dig(:message, :model),
        parent_tool_use_id: data[:parent_tool_use_id],
        error: data[:error] # authentication_failed, billing_error, rate_limit, invalid_request, server_error, unknown
      )
    end

    def self.parse_system_message(data)
      SystemMessage.new(
        subtype: data[:subtype],
        data: data
      )
    end

    def self.parse_result_message(data)
      ResultMessage.new(
        subtype: data[:subtype],
        duration_ms: data[:duration_ms],
        duration_api_ms: data[:duration_api_ms],
        is_error: data[:is_error],
        num_turns: data[:num_turns],
        session_id: data[:session_id],
        total_cost_usd: data[:total_cost_usd],
        usage: data[:usage],
        result: data[:result],
        structured_output: data[:structured_output] # Structured output when output_format is specified
      )
    end

    def self.parse_stream_event(data)
      StreamEvent.new(
        uuid: data[:uuid],
        session_id: data[:session_id],
        event: data[:event],
        parent_tool_use_id: data[:parent_tool_use_id]
      )
    end

    def self.parse_content_block(block)
      case block[:type]
      when 'text'
        TextBlock.new(text: block[:text])
      when 'thinking'
        ThinkingBlock.new(thinking: block[:thinking], signature: block[:signature])
      when 'tool_use'
        ToolUseBlock.new(id: block[:id], name: block[:name], input: block[:input])
      when 'tool_result'
        ToolResultBlock.new(
          tool_use_id: block[:tool_use_id],
          content: block[:content],
          is_error: block[:is_error]
        )
      else
        raise MessageParseError.new("Unknown content block type: #{block[:type]}")
      end
    end
  end
end



================================================
FILE: lib/claude_agent_sdk/query.rb
================================================
# frozen_string_literal: true

require 'json'
require 'async'
require 'async/queue'
require 'async/condition'
require 'securerandom'
require_relative 'transport'

module ClaudeAgentSDK
  # Handles bidirectional control protocol on top of Transport
  #
  # This class manages:
  # - Control request/response routing
  # - Hook callbacks
  # - Tool permission callbacks
  # - Message streaming
  # - Initialization handshake
  class Query
    attr_reader :transport, :is_streaming_mode, :sdk_mcp_servers

    def initialize(transport:, is_streaming_mode:, can_use_tool: nil, hooks: nil, sdk_mcp_servers: nil)
      @transport = transport
      @is_streaming_mode = is_streaming_mode
      @can_use_tool = can_use_tool
      @hooks = hooks || {}
      @sdk_mcp_servers = sdk_mcp_servers || {}

      # Control protocol state
      @pending_control_responses = {}
      @pending_control_results = {}
      @hook_callbacks = {}
      @next_callback_id = 0
      @request_counter = 0

      # Message stream
      @message_queue = Async::Queue.new
      @task = nil
      @initialized = false
      @closed = false
      @initialization_result = nil
    end

    # Initialize control protocol if in streaming mode
    # @return [Hash, nil] Initialize response with supported commands, or nil if not streaming
    def initialize_protocol
      return nil unless @is_streaming_mode

      # Build hooks configuration for initialization
      hooks_config = {}
      if @hooks && !@hooks.empty?
        @hooks.each do |event, matchers|
          next if matchers.nil? || matchers.empty?

          hooks_config[event] = []
          matchers.each do |matcher|
            callback_ids = []
            (matcher[:hooks] || []).each do |callback|
              callback_id = "hook_#{@next_callback_id}"
              @next_callback_id += 1
              @hook_callbacks[callback_id] = callback
              callback_ids << callback_id
            end
            hooks_config[event] << {
              matcher: matcher[:matcher],
              hookCallbackIds: callback_ids
            }
          end
        end
      end

      # Send initialize request
      request = {
        subtype: 'initialize',
        hooks: hooks_config.empty? ? nil : hooks_config
      }

      response = send_control_request(request)
      @initialized = true
      @initialization_result = response
      response
    end

    # Start reading messages from transport
    def start
      return if @task

      @task = Async do |task|
        task.async { read_messages }
      end
    end

    private

    def read_messages
      @transport.read_messages do |message|
        break if @closed

        msg_type = message[:type]

        # Route control messages
        case msg_type
        when 'control_response'
          handle_control_response(message)
        when 'control_request'
          Async { handle_control_request(message) }
        when 'control_cancel_request'
          # TODO: Implement cancellation support
          next
        else
          # Regular SDK messages go to the queue
          @message_queue.enqueue(message)
        end
      end
    rescue StandardError => e
      # Put error in queue so iterators can handle it
      @message_queue.enqueue({ type: 'error', error: e.message })
    ensure
      # Always signal end of stream
      @message_queue.enqueue({ type: 'end' })
    end

    def handle_control_response(message)
      response = message[:response] || {}
      request_id = response[:request_id]
      return unless @pending_control_responses.key?(request_id)

      if response[:subtype] == 'error'
        @pending_control_results[request_id] = StandardError.new(response[:error] || 'Unknown error')
      else
        @pending_control_results[request_id] = response
      end

      # Signal that response is ready
      @pending_control_responses[request_id].signal
    end

    def handle_control_request(request)
      request_id = request[:request_id]
      request_data = request[:request]
      subtype = request_data[:subtype]

      response_data = {}

      case subtype
      when 'can_use_tool'
        response_data = handle_permission_request(request_data)
      when 'hook_callback'
        response_data = handle_hook_callback(request_data)
      when 'mcp_message'
        response_data = handle_mcp_message(request_data)
      else
        raise "Unsupported control request subtype: #{subtype}"
      end

      # Send success response
      success_response = {
        type: 'control_response',
        response: {
          subtype: 'success',
          request_id: request_id,
          response: response_data
        }
      }
      @transport.write(JSON.generate(success_response) + "\n")
    rescue StandardError => e
      # Send error response
      error_response = {
        type: 'control_response',
        response: {
          subtype: 'error',
          request_id: request_id,
          error: e.message
        }
      }
      @transport.write(JSON.generate(error_response) + "\n")
    end

    def handle_permission_request(request_data)
      raise 'canUseTool callback is not provided' unless @can_use_tool

      original_input = request_data[:input]

      context = ToolPermissionContext.new(
        signal: nil,
        suggestions: request_data[:permission_suggestions] || []
      )

      response = @can_use_tool.call(
        request_data[:tool_name],
        request_data[:input],
        context
      )

      # Convert PermissionResult to expected format
      case response
      when PermissionResultAllow
        result = {
          behavior: 'allow',
          updatedInput: response.updated_input || original_input
        }
        if response.updated_permissions
          result[:updatedPermissions] = response.updated_permissions.map(&:to_h)
        end
        result
      when PermissionResultDeny
        result = { behavior: 'deny', message: response.message }
        result[:interrupt] = response.interrupt if response.interrupt
        result
      else
        raise "Tool permission callback must return PermissionResult, got #{response.class}"
      end
    end

    def handle_hook_callback(request_data)
      callback_id = request_data[:callback_id]
      callback = @hook_callbacks[callback_id]
      raise "No hook callback found for ID: #{callback_id}" unless callback

      # Parse input data into typed HookInput object
      input_data = request_data[:input] || {}
      hook_input = parse_hook_input(input_data)

      # Create typed HookContext
      context = HookContext.new(signal: nil)

      hook_output = callback.call(
        hook_input,
        request_data[:tool_use_id],
        context
      )

      # Convert Ruby-safe field names to CLI-expected names
      convert_hook_output_for_cli(hook_output)
    end

    def parse_hook_input(input_data)
      event_name = input_data[:hook_event_name] || input_data['hook_event_name']
      base_args = {
        session_id: input_data[:session_id],
        transcript_path: input_data[:transcript_path],
        cwd: input_data[:cwd],
        permission_mode: input_data[:permission_mode]
      }

      case event_name
      when 'PreToolUse'
        PreToolUseHookInput.new(
          tool_name: input_data[:tool_name],
          tool_input: input_data[:tool_input],
          **base_args
        )
      when 'PostToolUse'
        PostToolUseHookInput.new(
          tool_name: input_data[:tool_name],
          tool_input: input_data[:tool_input],
          tool_response: input_data[:tool_response],
          **base_args
        )
      when 'UserPromptSubmit'
        UserPromptSubmitHookInput.new(
          prompt: input_data[:prompt],
          **base_args
        )
      when 'Stop'
        StopHookInput.new(
          stop_hook_active: input_data[:stop_hook_active],
          **base_args
        )
      when 'SubagentStop'
        SubagentStopHookInput.new(
          stop_hook_active: input_data[:stop_hook_active],
          **base_args
        )
      when 'PreCompact'
        PreCompactHookInput.new(
          trigger: input_data[:trigger],
          custom_instructions: input_data[:custom_instructions],
          **base_args
        )
      else
        # Return base input for unknown event types
        BaseHookInput.new(**base_args)
      end
    end

    def handle_mcp_message(request_data)
      server_name = request_data[:server_name]
      mcp_message = request_data[:message]

      raise 'Missing server_name or message for MCP request' unless server_name && mcp_message

      mcp_response = handle_sdk_mcp_request(server_name, mcp_message)
      { mcp_response: mcp_response }
    end

    def convert_hook_output_for_cli(hook_output)
      # Handle typed output objects
      if hook_output.respond_to?(:to_h) && !hook_output.is_a?(Hash)
        return hook_output.to_h
      end

      return {} unless hook_output.is_a?(Hash)

      # Convert Ruby hash with symbol keys to CLI format
      # Handle special keywords that might be Ruby-safe versions
      converted = {}
      hook_output.each do |key, value|
        converted_key = case key
                        when :async_, 'async_' then 'async'
                        when :continue_, 'continue_' then 'continue'
                        when :hook_specific_output then 'hookSpecificOutput'
                        when :suppress_output then 'suppressOutput'
                        when :stop_reason then 'stopReason'
                        when :system_message then 'systemMessage'
                        when :async_timeout then 'asyncTimeout'
                        else key.to_s
                        end

        # Recursively convert nested objects
        converted_value = if value.respond_to?(:to_h) && !value.is_a?(Hash)
                            value.to_h
                          else
                            value
                          end
        converted[converted_key] = converted_value
      end
      converted
    end

    def send_control_request(request)
      raise 'Control requests require streaming mode' unless @is_streaming_mode

      # Generate unique request ID
      @request_counter += 1
      request_id = "req_#{@request_counter}_#{SecureRandom.hex(4)}"

      # Create condition for response
      condition = Async::Condition.new
      @pending_control_responses[request_id] = condition

      # Build and send request
      control_request = {
        type: 'control_request',
        request_id: request_id,
        request: request
      }

      @transport.write(JSON.generate(control_request) + "\n")

      # Wait for response with timeout
      Async do |task|
        task.with_timeout(60.0) do
          condition.wait
        end

        result = @pending_control_results.delete(request_id)
        @pending_control_responses.delete(request_id)

        raise result if result.is_a?(Exception)

        result[:response] || {}
      end.wait
    rescue Async::TimeoutError
      @pending_control_responses.delete(request_id)
      @pending_control_results.delete(request_id)
      raise "Control request timeout: #{request[:subtype]}"
    end

    def handle_sdk_mcp_request(server_name, message)
      # Convert server_name to symbol if needed for hash lookup
      server_key = @sdk_mcp_servers.key?(server_name) ? server_name : server_name.to_sym

      unless @sdk_mcp_servers.key?(server_key)
        return {
          jsonrpc: '2.0',
          id: message[:id],
          error: {
            code: -32601,
            message: "Server '#{server_name}' not found"
          }
        }
      end

      server = @sdk_mcp_servers[server_key]
      method = message[:method]
      params = message[:params] || {}

      case method
      when 'initialize'
        handle_mcp_initialize(server, message)
      when 'tools/list'
        handle_mcp_tools_list(server, message)
      when 'tools/call'
        handle_mcp_tools_call(server, message, params)
      when 'resources/list'
        handle_mcp_resources_list(server, message)
      when 'resources/read'
        handle_mcp_resources_read(server, message, params)
      when 'prompts/list'
        handle_mcp_prompts_list(server, message)
      when 'prompts/get'
        handle_mcp_prompts_get(server, message, params)
      when 'notifications/initialized'
        { jsonrpc: '2.0', result: {} }
      else
        {
          jsonrpc: '2.0',
          id: message[:id],
          error: { code: -32601, message: "Method '#{method}' not found" }
        }
      end
    rescue StandardError => e
      {
        jsonrpc: '2.0',
        id: message[:id],
        error: { code: -32603, message: e.message }
      }
    end

    def handle_mcp_initialize(server, message)
      capabilities = {}
      capabilities[:tools] = {} if server.tools && !server.tools.empty?
      capabilities[:resources] = {} if server.resources && !server.resources.empty?
      capabilities[:prompts] = {} if server.prompts && !server.prompts.empty?

      {
        jsonrpc: '2.0',
        id: message[:id],
        result: {
          protocolVersion: '2024-11-05',
          capabilities: capabilities,
          serverInfo: {
            name: server.name,
            version: server.version || '1.0.0'
          }
        }
      }
    end

    def handle_mcp_tools_list(server, message)
      # List tools from the SDK MCP server
      tools_data = server.list_tools
      {
        jsonrpc: '2.0',
        id: message[:id],
        result: { tools: tools_data }
      }
    end

    def handle_mcp_tools_call(server, message, params)
      # Execute tool on the SDK MCP server
      tool_name = params[:name]
      arguments = params[:arguments] || {}

      # Call the tool
      result = server.call_tool(tool_name, arguments)

      # Format response
      content = []
      if result[:content]
        result[:content].each do |item|
          if item[:type] == 'text'
            content << { type: 'text', text: item[:text] }
          end
        end
      end

      response_data = { content: content }
      response_data[:is_error] = true if result[:is_error]

      {
        jsonrpc: '2.0',
        id: message[:id],
        result: response_data
      }
    end

    def handle_mcp_resources_list(server, message)
      # List resources from the SDK MCP server
      resources_data = server.list_resources
      {
        jsonrpc: '2.0',
        id: message[:id],
        result: { resources: resources_data }
      }
    end

    def handle_mcp_resources_read(server, message, params)
      # Read a resource from the SDK MCP server
      uri = params[:uri]
      raise 'Missing uri parameter for resources/read' unless uri

      # Read the resource
      result = server.read_resource(uri)

      {
        jsonrpc: '2.0',
        id: message[:id],
        result: result
      }
    end

    def handle_mcp_prompts_list(server, message)
      # List prompts from the SDK MCP server
      prompts_data = server.list_prompts
      {
        jsonrpc: '2.0',
        id: message[:id],
        result: { prompts: prompts_data }
      }
    end

    def handle_mcp_prompts_get(server, message, params)
      # Get a prompt from the SDK MCP server
      name = params[:name]
      raise 'Missing name parameter for prompts/get' unless name

      arguments = params[:arguments] || {}

      # Get the prompt
      result = server.get_prompt(name, arguments)

      {
        jsonrpc: '2.0',
        id: message[:id],
        result: result
      }
    end

    public

    # Send interrupt control request
    def interrupt
      send_control_request({ subtype: 'interrupt' })
    end

    # Change permission mode
    def set_permission_mode(mode)
      send_control_request({
                             subtype: 'set_permission_mode',
                             mode: mode
                           })
    end

    # Change the AI model
    def set_model(model)
      send_control_request({
                             subtype: 'set_model',
                             model: model
                           })
    end

    # Rewind files to a previous checkpoint (v0.1.15+)
    # Restores file state to what it was at the given user message
    # Requires enable_file_checkpointing to be true in options
    # @param user_message_uuid [String] The UUID of the UserMessage to rewind to
    def rewind_files(user_message_uuid)
      send_control_request({
                             subtype: 'rewind_files',
                             userMessageUuid: user_message_uuid
                           })
    end

    # Stream input messages to transport
    def stream_input(stream)
      stream.each do |message|
        break if @closed
        @transport.write(JSON.generate(message) + "\n")
      end
      @transport.end_input
    rescue StandardError => e
      # Log error but don't raise
      warn "Error streaming input: #{e.message}"
    end

    # Receive SDK messages (not control messages)
    def receive_messages(&block)
      return enum_for(:receive_messages) unless block

      loop do
        message = @message_queue.dequeue
        break if message[:type] == 'end'
        raise message[:error] if message[:type] == 'error'

        block.call(message)
      end
    end

    # Close the query and transport
    def close
      @closed = true
      @task&.stop
      @transport.close
    end
  end
end



================================================
FILE: lib/claude_agent_sdk/sdk_mcp_server.rb
================================================
# frozen_string_literal: true

require 'mcp'

module ClaudeAgentSDK
  # SDK MCP Server - wraps official MCP::Server with block-based API
  #
  # Unlike external MCP servers that run as separate processes, SDK MCP servers
  # run directly in your application's process, providing better performance
  # and simpler deployment.
  #
  # This class wraps the official MCP Ruby SDK and provides a simpler block-based
  # API for defining tools, resources, and prompts.
  class SdkMcpServer
    attr_reader :name, :version, :tools, :resources, :prompts, :mcp_server

    def initialize(name:, version: '1.0.0', tools: [], resources: [], prompts: [])
      @name = name
      @version = version
      @tools = tools
      @resources = resources
      @prompts = prompts

      # Create dynamic Tool classes from tool definitions
      tool_classes = create_tool_classes(tools)

      # Create dynamic Resource classes from resource definitions
      resource_classes = create_resource_classes(resources)

      # Create dynamic Prompt classes from prompt definitions
      prompt_classes = create_prompt_classes(prompts)

      # Create the official MCP::Server instance
      @mcp_server = MCP::Server.new(
        name: name,
        version: version,
        tools: tool_classes,
        resources: resource_classes,
        prompts: prompt_classes
      )
    end

    # Handle a JSON-RPC request
    # @param json_string [String] JSON-RPC request
    # @return [String] JSON-RPC response
    def handle_json(json_string)
      @mcp_server.handle_json(json_string)
    end

    # List all available tools (for backward compatibility)
    # @return [Array<Hash>] Array of tool definitions
    def list_tools
      @tools.map do |tool|
        {
          name: tool.name,
          description: tool.description,
          inputSchema: convert_input_schema(tool.input_schema)
        }
      end
    end

    # Execute a tool by name (for backward compatibility)
    # @param name [String] Tool name
    # @param arguments [Hash] Tool arguments
    # @return [Hash] Tool result
    def call_tool(name, arguments)
      tool = @tools.find { |t| t.name == name }
      raise "Tool '#{name}' not found" unless tool

      # Call the tool's handler
      result = tool.handler.call(arguments)

      # Ensure result has the expected format
      unless result.is_a?(Hash) && result[:content]
        raise "Tool '#{name}' must return a hash with :content key"
      end

      result
    end

    # List all available resources (for backward compatibility)
    # @return [Array<Hash>] Array of resource definitions
    def list_resources
      @resources.map do |resource|
        {
          uri: resource.uri,
          name: resource.name,
          description: resource.description,
          mimeType: resource.mime_type
        }.compact
      end
    end

    # Read a resource by URI (for backward compatibility)
    # @param uri [String] Resource URI
    # @return [Hash] Resource content
    def read_resource(uri)
      resource = @resources.find { |r| r.uri == uri }
      raise "Resource '#{uri}' not found" unless resource

      # Call the resource's reader
      content = resource.reader.call

      # Ensure content has the expected format
      unless content.is_a?(Hash) && content[:contents]
        raise "Resource '#{uri}' must return a hash with :contents key"
      end

      content
    end

    # List all available prompts (for backward compatibility)
    # @return [Array<Hash>] Array of prompt definitions
    def list_prompts
      @prompts.map do |prompt|
        {
          name: prompt.name,
          description: prompt.description,
          arguments: prompt.arguments
        }.compact
      end
    end

    # Get a prompt by name (for backward compatibility)
    # @param name [String] Prompt name
    # @param arguments [Hash] Arguments to fill in the prompt template
    # @return [Hash] Prompt with filled-in arguments
    def get_prompt(name, arguments = {})
      prompt = @prompts.find { |p| p.name == name }
      raise "Prompt '#{name}' not found" unless prompt

      # Call the prompt's generator
      result = prompt.generator.call(arguments)

      # Ensure result has the expected format
      unless result.is_a?(Hash) && result[:messages]
        raise "Prompt '#{name}' must return a hash with :messages key"
      end

      result
    end

    private

    # Create dynamic Tool classes from tool definitions
    def create_tool_classes(tools)
      tools.map do |tool_def|
        # Create a new class that extends MCP::Tool
        Class.new(MCP::Tool) do
          @tool_def = tool_def

          class << self
            attr_reader :tool_def

            def name_value
              @tool_def.name
            end

            def description_value
              @tool_def.description
            end

            def input_schema_value
              schema = convert_schema(@tool_def.input_schema)
              MCP::Tool::InputSchema.new(
                properties: schema[:properties] || {},
                required: schema[:required] || []
              )
            end

            def call(server_context: nil, **args)
              # Filter out server_context and pass remaining args to handler
              result = @tool_def.handler.call(args)

              # Convert result to MCP::Tool::Response format
              content = result[:content].map do |item|
                {
                  type: item[:type],
                  text: item[:text]
                }
              end

              MCP::Tool::Response.new(content)
            end

            private

            def convert_schema(schema)
              # If it's already a proper JSON schema, return it
              if schema.is_a?(Hash) && schema[:type] && schema[:properties]
                return schema
              end

              # Simple schema: hash mapping parameter names to types
              if schema.is_a?(Hash)
                properties = {}
                schema.each do |param_name, param_type|
                  properties[param_name] = type_to_json_schema(param_type)
                end

                return {
                  type: 'object',
                  properties: properties,
                  required: properties.keys.map(&:to_s)
                }
              end

              # Default fallback
              { type: 'object', properties: {} }
            end

            def type_to_json_schema(type)
              case type
              when :string, String
                { type: 'string' }
              when :integer, Integer
                { type: 'integer' }
              when :float, Float
                { type: 'number' }
              when :boolean, TrueClass, FalseClass
                { type: 'boolean' }
              when :number
                { type: 'number' }
              else
                { type: 'string' } # Default fallback
              end
            end
          end
        end
      end
    end

    # Create dynamic Resource classes from resource definitions
    def create_resource_classes(resources)
      resources.map do |resource_def|
        # Create a new class that extends MCP::Resource
        Class.new(MCP::Resource) do
          @resource_def = resource_def

          class << self
            attr_reader :resource_def

            def uri
              @resource_def.uri
            end

            def name
              @resource_def.name
            end

            def description
              @resource_def.description
            end

            def mime_type
              @resource_def.mime_type
            end

            def read
              result = @resource_def.reader.call

              # Convert to MCP format
              result[:contents].map do |content|
                MCP::ResourceContents.new(
                  uri: content[:uri],
                  mime_type: content[:mimeType] || content[:mime_type],
                  text: content[:text]
                )
              end
            end
          end
        end
      end
    end

    # Create dynamic Prompt classes from prompt definitions
    def create_prompt_classes(prompts)
      prompts.map do |prompt_def|
        # Create a new class that extends MCP::Prompt
        Class.new(MCP::Prompt) do
          @prompt_def = prompt_def

          class << self
            attr_reader :prompt_def

            def name
              @prompt_def.name
            end

            def description
              @prompt_def.description
            end

            def arguments
              @prompt_def.arguments || []
            end

            def get(**args)
              result = @prompt_def.generator.call(args)

              # Convert to MCP format
              {
                messages: result[:messages].map do |msg|
                  {
                    role: msg[:role],
                    content: msg[:content]
                  }
                end
              }
            end
          end
        end
      end
    end

    def convert_input_schema(schema)
      # If it's already a proper JSON schema, return it
      if schema.is_a?(Hash) && schema[:type] && schema[:properties]
        return schema
      end

      # Simple schema: hash mapping parameter names to types
      if schema.is_a?(Hash)
        properties = {}
        schema.each do |param_name, param_type|
          properties[param_name] = type_to_json_schema(param_type)
        end

        return {
          type: 'object',
          properties: properties,
          required: properties.keys
        }
      end

      # Default fallback
      { type: 'object', properties: {} }
    end

    def type_to_json_schema(type)
      case type
      when :string, String
        { type: 'string' }
      when :integer, Integer
        { type: 'integer' }
      when :float, Float
        { type: 'number' }
      when :boolean, TrueClass, FalseClass
        { type: 'boolean' }
      when :number
        { type: 'number' }
      else
        { type: 'string' } # Default fallback
      end
    end
  end

  # Helper function to create a tool definition
  #
  # @param name [String] Unique identifier for the tool
  # @param description [String] Human-readable description
  # @param input_schema [Hash] Schema defining input parameters
  # @param handler [Proc] Block that implements the tool logic
  # @return [SdkMcpTool] Tool definition
  #
  # @example Simple tool
  #   tool = create_tool('greet', 'Greet a user', { name: :string }) do |args|
  #     { content: [{ type: 'text', text: "Hello, #{args[:name]}!" }] }
  #   end
  #
  # @example Tool with multiple parameters
  #   tool = create_tool('add', 'Add two numbers', { a: :number, b: :number }) do |args|
  #     result = args[:a] + args[:b]
  #     { content: [{ type: 'text', text: "Result: #{result}" }] }
  #   end
  #
  # @example Tool with error handling
  #   tool = create_tool('divide', 'Divide numbers', { a: :number, b: :number }) do |args|
  #     if args[:b] == 0
  #       { content: [{ type: 'text', text: 'Error: Division by zero' }], is_error: true }
  #     else
  #       result = args[:a] / args[:b]
  #       { content: [{ type: 'text', text: "Result: #{result}" }] }
  #     end
  #   end
  def self.create_tool(name, description, input_schema, &handler)
    raise ArgumentError, 'Block required for tool handler' unless handler

    SdkMcpTool.new(
      name: name,
      description: description,
      input_schema: input_schema,
      handler: handler
    )
  end

  # Helper function to create a resource definition
  #
  # @param uri [String] Unique identifier for the resource (e.g., "file:///path/to/file")
  # @param name [String] Human-readable name
  # @param description [String, nil] Optional description
  # @param mime_type [String, nil] Optional MIME type (e.g., "text/plain", "application/json")
  # @param reader [Proc] Block that returns the resource content
  # @return [SdkMcpResource] Resource definition
  #
  # @example File resource
  #   resource = create_resource(
  #     uri: 'file:///config/settings.json',
  #     name: 'Application Settings',
  #     description: 'Current application configuration',
  #     mime_type: 'application/json'
  #   ) do
  #     content = File.read('/path/to/settings.json')
  #     {
  #       contents: [{
  #         uri: 'file:///config/settings.json',
  #         mimeType: 'application/json',
  #         text: content
  #       }]
  #     }
  #   end
  #
  # @example Database resource
  #   resource = create_resource(
  #     uri: 'db://users/count',
  #     name: 'User Count',
  #     description: 'Total number of registered users'
  #   ) do
  #     count = User.count
  #     {
  #       contents: [{
  #         uri: 'db://users/count',
  #         mimeType: 'text/plain',
  #         text: count.to_s
  #       }]
  #     }
  #   end
  def self.create_resource(uri:, name:, description: nil, mime_type: nil, &reader)
    raise ArgumentError, 'Block required for resource reader' unless reader

    SdkMcpResource.new(
      uri: uri,
      name: name,
      description: description,
      mime_type: mime_type,
      reader: reader
    )
  end

  # Helper function to create a prompt definition
  #
  # @param name [String] Unique identifier for the prompt
  # @param description [String, nil] Optional description
  # @param arguments [Array<Hash>, nil] Optional argument definitions
  # @param generator [Proc] Block that generates prompt messages
  # @return [SdkMcpPrompt] Prompt definition
  #
  # @example Simple prompt
  #   prompt = create_prompt(
  #     name: 'code_review',
  #     description: 'Review code for best practices'
  #   ) do |args|
  #     {
  #       messages: [
  #         {
  #           role: 'user',
  #           content: {
  #             type: 'text',
  #             text: 'Please review this code for best practices and suggest improvements.'
  #           }
  #         }
  #       ]
  #     }
  #   end
  #
  # @example Prompt with arguments
  #   prompt = create_prompt(
  #     name: 'git_commit',
  #     description: 'Generate a git commit message',
  #     arguments: [
  #       { name: 'changes', description: 'Description of changes', required: true }
  #     ]
  #   ) do |args|
  #     {
  #       messages: [
  #         {
  #           role: 'user',
  #           content: {
  #             type: 'text',
  #             text: "Generate a concise git commit message for: #{args[:changes]}"
  #           }
  #         }
  #       ]
  #     }
  #   end
  def self.create_prompt(name:, description: nil, arguments: nil, &generator)
    raise ArgumentError, 'Block required for prompt generator' unless generator

    SdkMcpPrompt.new(
      name: name,
      description: description,
      arguments: arguments,
      generator: generator
    )
  end

  # Create an SDK MCP server
  #
  # @param name [String] Unique identifier for the server
  # @param version [String] Server version (default: '1.0.0')
  # @param tools [Array<SdkMcpTool>] List of tool definitions
  # @param resources [Array<SdkMcpResource>] List of resource definitions
  # @param prompts [Array<SdkMcpPrompt>] List of prompt definitions
  # @return [Hash] MCP server configuration for ClaudeAgentOptions
  #
  # @example Simple calculator server
  #   add_tool = ClaudeAgentSDK.create_tool('add', 'Add numbers', { a: :number, b: :number }) do |args|
  #     { content: [{ type: 'text', text: "Sum: #{args[:a] + args[:b]}" }] }
  #   end
  #
  #   calculator = ClaudeAgentSDK.create_sdk_mcp_server(
  #     name: 'calculator',
  #     version: '2.0.0',
  #     tools: [add_tool]
  #   )
  #
  #   options = ClaudeAgentOptions.new(
  #     mcp_servers: { calc: calculator },
  #     allowed_tools: ['mcp__calc__add']
  #   )
  #
  # @example Server with resources and prompts
  #   config_resource = ClaudeAgentSDK.create_resource(
  #     uri: 'config://app',
  #     name: 'App Config'
  #   ) { { contents: [{ uri: 'config://app', text: 'config data' }] } }
  #
  #   review_prompt = ClaudeAgentSDK.create_prompt(
  #     name: 'review',
  #     description: 'Code review'
  #   ) { { messages: [{ role: 'user', content: { type: 'text', text: 'Review this' } }] } }
  #
  #   server = ClaudeAgentSDK.create_sdk_mcp_server(
  #     name: 'dev-tools',
  #     resources: [config_resource],
  #     prompts: [review_prompt]
  #   )
  def self.create_sdk_mcp_server(name:, version: '1.0.0', tools: [], resources: [], prompts: [])
    server = SdkMcpServer.new(
      name: name,
      version: version,
      tools: tools,
      resources: resources,
      prompts: prompts
    )

    # Return configuration for ClaudeAgentOptions
    {
      type: 'sdk',
      name: name,
      instance: server
    }
  end
end



================================================
FILE: lib/claude_agent_sdk/streaming.rb
================================================
# frozen_string_literal: true

require 'json'

module ClaudeAgentSDK
  # Streaming input helpers for Claude Agent SDK
  module Streaming
    # Create a user message for streaming input
    #
    # @param content [String] The message content
    # @param session_id [String] Session identifier
    # @param parent_tool_use_id [String, nil] Parent tool use ID if responding to a tool
    # @return [String] JSON-encoded message
    def self.user_message(content, session_id: 'default', parent_tool_use_id: nil)
      message = {
        type: 'user',
        message: {
          role: 'user',
          content: content
        },
        parent_tool_use_id: parent_tool_use_id,
        session_id: session_id
      }
      JSON.generate(message) + "\n"
    end

    # Create an Enumerator from an array of messages
    #
    # @param messages [Array<String>] Array of message strings
    # @param session_id [String] Session identifier
    # @return [Enumerator] Enumerator yielding JSON-encoded messages
    #
    # @example
    #   messages = ['Hello', 'What is 2+2?', 'Thanks!']
    #   stream = ClaudeAgentSDK::Streaming.from_array(messages)
    def self.from_array(messages, session_id: 'default')
      Enumerator.new do |yielder|
        messages.each do |content|
          yielder << user_message(content, session_id: session_id)
        end
      end
    end

    # Create an Enumerator from a block
    #
    # @yield Block that yields message strings
    # @param session_id [String] Session identifier
    # @return [Enumerator] Enumerator yielding JSON-encoded messages
    #
    # @example
    #   stream = ClaudeAgentSDK::Streaming.from_block do |yielder|
    #     yielder.yield('First message')
    #     sleep 1
    #     yielder.yield('Second message')
    #   end
    def self.from_block(session_id: 'default', &block)
      Enumerator.new do |yielder|
        collector = Object.new
        def collector.yield(content)
          @content = content
        end
        def collector.content
          @content
        end

        inner_enum = Enumerator.new(&block)
        inner_enum.each do |content|
          yielder << user_message(content, session_id: session_id)
        end
      end
    end
  end
end



================================================
FILE: lib/claude_agent_sdk/subprocess_cli_transport.rb
================================================
# frozen_string_literal: true

require 'json'
require 'open3'
require_relative 'transport'
require_relative 'errors'
require_relative 'version'

module ClaudeAgentSDK
  # Subprocess transport using Claude Code CLI
  class SubprocessCLITransport < Transport
    DEFAULT_MAX_BUFFER_SIZE = 1024 * 1024 # 1MB buffer limit
    MINIMUM_CLAUDE_CODE_VERSION = '2.0.0'

    def initialize(prompt, options)
      @prompt = prompt
      @is_streaming = !prompt.is_a?(String)
      @options = options
      @cli_path = options.cli_path || find_cli
      @cwd = options.cwd
      @process = nil
      @stdin = nil
      @stdout = nil
      @stderr = nil
      @ready = false
      @exit_error = nil
      @max_buffer_size = options.max_buffer_size || DEFAULT_MAX_BUFFER_SIZE
      @stderr_task = nil
    end

    def find_cli
      # Try which command first
      cli = `which claude 2>/dev/null`.strip
      return cli unless cli.empty?

      # Try common locations
      locations = [
        File.join(Dir.home, '.claude/local/claude'),  # Claude Code default install location
        File.join(Dir.home, '.npm-global/bin/claude'),
        '/usr/local/bin/claude',
        File.join(Dir.home, '.local/bin/claude'),
        File.join(Dir.home, 'node_modules/.bin/claude'),
        File.join(Dir.home, '.yarn/bin/claude')
      ]

      locations.each do |path|
        return path if File.exist?(path) && File.file?(path)
      end

      raise CLINotFoundError.new(
        "Claude Code not found. Install with:\n" \
        "  npm install -g @anthropic-ai/claude-code\n" \
        "\nIf already installed locally, try:\n" \
        '  export PATH="$HOME/node_modules/.bin:$PATH"' \
        "\n\nOr provide the path via ClaudeAgentOptions:\n" \
        "  ClaudeAgentOptions.new(cli_path: '/path/to/claude')"
      )
    end

    def build_command
      cmd = [@cli_path, '--output-format', 'stream-json', '--verbose']

      # System prompt handling
      if @options.system_prompt
        if @options.system_prompt.is_a?(String)
          cmd.concat(['--system-prompt', @options.system_prompt])
        elsif @options.system_prompt.is_a?(Hash) &&
              @options.system_prompt[:type] == 'preset' &&
              @options.system_prompt[:append]
          cmd.concat(['--append-system-prompt', @options.system_prompt[:append]])
        end
      end

      cmd.concat(['--allowedTools', @options.allowed_tools.join(',')]) unless @options.allowed_tools.empty?
      cmd.concat(['--max-turns', @options.max_turns.to_s]) if @options.max_turns
      cmd.concat(['--disallowedTools', @options.disallowed_tools.join(',')]) unless @options.disallowed_tools.empty?
      cmd.concat(['--model', @options.model]) if @options.model
      cmd.concat(['--fallback-model', @options.fallback_model]) if @options.fallback_model
      cmd.concat(['--permission-prompt-tool', @options.permission_prompt_tool_name]) if @options.permission_prompt_tool_name
      cmd.concat(['--permission-mode', @options.permission_mode]) if @options.permission_mode
      cmd << '--continue' if @options.continue_conversation
      cmd.concat(['--resume', @options.resume]) if @options.resume

      # Settings handling with sandbox merge
      # Sandbox settings are merged into the main settings JSON
      if @options.settings || @options.sandbox
        settings_hash = {}
        settings_is_path = false

        # Parse existing settings if provided
        if @options.settings
          if @options.settings.is_a?(String)
            begin
              settings_hash = JSON.parse(@options.settings)
            rescue JSON::ParserError
              # If not valid JSON, treat as file path and pass as-is
              settings_is_path = true
              cmd.concat(['--settings', @options.settings])
              if @options.sandbox
                warn "Warning: Cannot merge sandbox settings when settings is a file path. " \
                     "Sandbox settings will be ignored. Use a Hash or JSON string for settings " \
                     "to enable sandbox merging."
              end
            end
          elsif @options.settings.is_a?(Hash)
            settings_hash = @options.settings.dup
          end
        end

        # Merge sandbox settings if provided (only when settings is not a file path)
        if !settings_is_path && @options.sandbox
          sandbox_hash = if @options.sandbox.is_a?(SandboxSettings)
                           @options.sandbox.to_h
                         else
                           @options.sandbox
                         end
          settings_hash[:sandbox] = sandbox_hash unless sandbox_hash.empty?
        end

        # Output merged settings (only when settings is not a file path)
        if !settings_is_path && !settings_hash.empty?
          cmd.concat(['--settings', JSON.generate(settings_hash)])
        end
      end

      # Budget limit option
      cmd.concat(['--max-budget-usd', @options.max_budget_usd.to_s]) if @options.max_budget_usd
      # Note: max_thinking_tokens is stored in options but not yet supported by Claude CLI

      # Betas option for enabling experimental features
      if @options.betas && !@options.betas.empty?
        cmd.concat(['--betas', @options.betas.join(',')])
      end

      # Tools option for base tools selection
      if @options.tools
        if @options.tools.is_a?(Array)
          cmd.concat(['--tools', @options.tools.join(',')])
        elsif @options.tools.is_a?(ToolsPreset)
          cmd.concat(['--tools', JSON.generate(@options.tools.to_h)])
        elsif @options.tools.is_a?(Hash)
          cmd.concat(['--tools', JSON.generate(@options.tools)])
        end
      end

      # Append allowed tools option
      if @options.append_allowed_tools && !@options.append_allowed_tools.empty?
        cmd.concat(['--append-allowed-tools', @options.append_allowed_tools.join(',')])
      end

      # File checkpointing for rewind support
      cmd << '--enable-file-checkpointing' if @options.enable_file_checkpointing

      # JSON schema for structured output
      # Accepts either:
      # 1. Direct schema: { type: 'object', properties: {...} }
      # 2. Wrapped format: { type: 'json_schema', schema: {...} }
      if @options.output_format
        schema = if @options.output_format.is_a?(Hash) && @options.output_format[:type] == 'json_schema'
                   @options.output_format[:schema]
                 elsif @options.output_format.is_a?(Hash) && @options.output_format['type'] == 'json_schema'
                   @options.output_format['schema']
                 else
                   @options.output_format
                 end
        schema_json = schema.is_a?(String) ? schema : JSON.generate(schema)
        cmd.concat(['--json-schema', schema_json])
      end

      # Add directories
      @options.add_dirs.each do |dir|
        cmd.concat(['--add-dir', dir.to_s])
      end

      # MCP servers
      if @options.mcp_servers && !@options.mcp_servers.empty?
        if @options.mcp_servers.is_a?(Hash)
          servers_for_cli = {}
          @options.mcp_servers.each do |name, config|
            if config.is_a?(Hash) && config[:type] == 'sdk'
              # For SDK servers, exclude instance field
              sdk_config = config.reject { |k, _| k == :instance }
              servers_for_cli[name] = sdk_config
            else
              servers_for_cli[name] = config
            end
          end
          cmd.concat(['--mcp-config', JSON.generate({ mcpServers: servers_for_cli })]) unless servers_for_cli.empty?
        else
          cmd.concat(['--mcp-config', @options.mcp_servers.to_s])
        end
      end

      cmd << '--include-partial-messages' if @options.include_partial_messages
      cmd << '--fork-session' if @options.fork_session

      # Agents
      if @options.agents
        agents_dict = @options.agents.transform_values do |agent_def|
          {
            description: agent_def.description,
            prompt: agent_def.prompt,
            tools: agent_def.tools,
            model: agent_def.model
          }.compact
        end
        cmd.concat(['--agents', JSON.generate(agents_dict)])
      end

      # Plugins
      if @options.plugins && !@options.plugins.empty?
        plugins_config = @options.plugins.map do |plugin|
          plugin.is_a?(SdkPluginConfig) ? plugin.to_h : plugin
        end
        cmd.concat(['--plugins', JSON.generate(plugins_config)])
      end

      # Setting sources
      sources_value = @options.setting_sources ? @options.setting_sources.join(',') : ''
      cmd.concat(['--setting-sources', sources_value])

      # Extra args
      @options.extra_args.each do |flag, value|
        if value.nil?
          cmd << "--#{flag}"
        else
          cmd.concat(["--#{flag}", value.to_s])
        end
      end

      # Prompt handling
      if @is_streaming
        cmd.concat(['--input-format', 'stream-json'])
      else
        cmd.concat(['--print', '--', @prompt.to_s])
      end

      cmd
    end

    def connect
      return if @process

      check_claude_version

      cmd = build_command

      # Build environment
      process_env = ENV.to_h.merge(@options.env).merge(
        'CLAUDE_CODE_ENTRYPOINT' => 'sdk-rb',
        'CLAUDE_AGENT_SDK_VERSION' => VERSION
      )
      process_env['PWD'] = @cwd.to_s if @cwd

      # Determine stderr handling
      should_pipe_stderr = @options.stderr || @options.debug_stderr || @options.extra_args.key?('debug-to-stderr')

      begin
        # Start process using Open3
        opts = { chdir: @cwd&.to_s }.compact

        @stdin, @stdout, @stderr, @process = Open3.popen3(process_env, *cmd, opts)

        # Handle stderr if piped
        if should_pipe_stderr && @stderr
          @stderr_task = Thread.new do
            handle_stderr
          rescue StandardError
            # Ignore errors during stderr reading
          end
        end

        # Close stdin for non-streaming mode
        @stdin.close unless @is_streaming
        @stdin = nil unless @is_streaming

        @ready = true
      rescue Errno::ENOENT => e
        # Check if error is from cwd or CLI
        if @cwd && !File.directory?(@cwd.to_s)
          error = CLIConnectionError.new("Working directory does not exist: #{@cwd}")
          @exit_error = error
          raise error
        end
        error = CLINotFoundError.new("Claude Code not found at: #{@cli_path}")
        @exit_error = error
        raise error
      rescue StandardError => e
        error = CLIConnectionError.new("Failed to start Claude Code: #{e}")
        @exit_error = error
        raise error
      end
    end

    def handle_stderr
      return unless @stderr

      @stderr.each_line do |line|
        line_str = line.chomp
        next if line_str.empty?

        # Call stderr callback if provided
        @options.stderr&.call(line_str)

        # Write to debug_stderr file/IO if provided
        if @options.debug_stderr
          if @options.debug_stderr.respond_to?(:puts)
            @options.debug_stderr.puts(line_str)
          elsif @options.debug_stderr.is_a?(String)
            File.open(@options.debug_stderr, 'a') { |f| f.puts(line_str) }
          end
        end
      end
    rescue StandardError
      # Ignore errors during stderr reading
    end

    def close
      @ready = false
      return unless @process

      # Kill stderr thread
      if @stderr_task&.alive?
        @stderr_task.kill
        @stderr_task.join(1) rescue nil
      end

      # Close streams
      begin
        @stdin&.close
      rescue StandardError
        # Ignore
      end
      begin
        @stdout&.close
      rescue StandardError
        # Ignore
      end
      begin
        @stderr&.close
      rescue StandardError
        # Ignore
      end

      # Terminate process
      begin
        Process.kill('TERM', @process.pid) if @process.alive?
        @process.value
      rescue StandardError
        # Ignore
      end

      @process = nil
      @stdout = nil
      @stdin = nil
      @stderr = nil
      @stderr_task = nil
      @exit_error = nil
    end

    def write(data)
      raise CLIConnectionError, 'ProcessTransport is not ready for writing' unless @ready && @stdin
      raise CLIConnectionError, "Cannot write to terminated process" if @process && !@process.alive?

      raise CLIConnectionError, "Cannot write to process that exited with error: #{@exit_error}" if @exit_error

      begin
        @stdin.write(data)
        @stdin.flush
      rescue StandardError => e
        @ready = false
        @exit_error = CLIConnectionError.new("Failed to write to process stdin: #{e}")
        raise @exit_error
      end
    end

    def end_input
      return unless @stdin

      begin
        @stdin.close
      rescue StandardError
        # Ignore
      end
      @stdin = nil
    end

    def read_messages(&block)
      return enum_for(:read_messages) unless block_given?

      raise CLIConnectionError, 'Not connected' unless @process && @stdout

      json_buffer = ''

      begin
        @stdout.each_line do |line|
          line_str = line.strip
          next if line_str.empty?

          json_lines = line_str.split("\n")

          json_lines.each do |json_line|
            json_line = json_line.strip
            next if json_line.empty?

            json_buffer += json_line

            if json_buffer.bytesize > @max_buffer_size
              buffer_length = json_buffer.bytesize
              json_buffer = ''
              raise CLIJSONDecodeError.new(
                "JSON message exceeded maximum buffer size",
                StandardError.new("Buffer size #{buffer_length} exceeds limit #{@max_buffer_size}")
              )
            end

            begin
              data = JSON.parse(json_buffer, symbolize_names: true)
              json_buffer = ''
              yield data
            rescue JSON::ParserError
              # Continue accumulating
              next
            end
          end
        end
      rescue IOError
        # Stream closed
      rescue StopIteration
        # Client disconnected
      end

      # Check process completion
      status = @process.value
      returncode = status.exitstatus

      if returncode && returncode != 0
        @exit_error = ProcessError.new(
          "Command failed with exit code #{returncode}",
          exit_code: returncode,
          stderr: 'Check stderr output for details'
        )
        raise @exit_error
      end
    end

    def check_claude_version
      begin
        output = `#{@cli_path} -v 2>&1`.strip
        if match = output.match(/([0-9]+\.[0-9]+\.[0-9]+)/)
          version = match[1]
          version_parts = version.split('.').map(&:to_i)
          min_parts = MINIMUM_CLAUDE_CODE_VERSION.split('.').map(&:to_i)

          if version_parts < min_parts
            warning = "Warning: Claude Code version #{version} is unsupported in the Agent SDK. " \
                      "Minimum required version is #{MINIMUM_CLAUDE_CODE_VERSION}. " \
                      "Some features may not work correctly."
            warn warning
          end
        end
      rescue StandardError
        # Ignore version check errors
      end
    end

    def ready?
      @ready
    end
  end
end



================================================
FILE: lib/claude_agent_sdk/transport.rb
================================================
# frozen_string_literal: true

module ClaudeAgentSDK
  # Abstract transport for Claude communication
  #
  # WARNING: This internal API is exposed for custom transport implementations
  # (e.g., remote Claude Code connections). The Claude Code team may change or
  # remove this abstract class in any future release. Custom implementations
  # must be updated to match interface changes.
  class Transport
    # Connect the transport and prepare for communication
    def connect
      raise NotImplementedError, 'Subclasses must implement #connect'
    end

    # Write raw data to the transport
    # @param data [String] Raw string data to write (typically JSON + newline)
    def write(data)
      raise NotImplementedError, 'Subclasses must implement #write'
    end

    # Read and parse messages from the transport
    # @return [Enumerator] Async enumerator of parsed JSON messages
    def read_messages
      raise NotImplementedError, 'Subclasses must implement #read_messages'
    end

    # Close the transport connection and clean up resources
    def close
      raise NotImplementedError, 'Subclasses must implement #close'
    end

    # Check if transport is ready for communication
    # @return [Boolean] True if transport is ready to send/receive messages
    def ready?
      raise NotImplementedError, 'Subclasses must implement #ready?'
    end

    # End the input stream (close stdin for process transports)
    def end_input
      raise NotImplementedError, 'Subclasses must implement #end_input'
    end
  end
end



================================================
FILE: lib/claude_agent_sdk/types.rb
================================================
# frozen_string_literal: true

module ClaudeAgentSDK
  # Type constants for permission modes
  PERMISSION_MODES = %w[default acceptEdits plan bypassPermissions].freeze

  # Type constants for setting sources
  SETTING_SOURCES = %w[user project local].freeze

  # Type constants for permission update destinations
  PERMISSION_UPDATE_DESTINATIONS = %w[userSettings projectSettings localSettings session].freeze

  # Type constants for permission behaviors
  PERMISSION_BEHAVIORS = %w[allow deny ask].freeze

  # Type constants for hook events
  HOOK_EVENTS = %w[PreToolUse PostToolUse UserPromptSubmit Stop SubagentStop PreCompact].freeze

  # Type constants for assistant message errors
  ASSISTANT_MESSAGE_ERRORS = %w[authentication_failed billing_error rate_limit invalid_request server_error unknown].freeze

  # Type constants for SDK beta features
  # Available beta features that can be enabled via the betas option
  SDK_BETAS = %w[context-1m-2025-08-07].freeze

  # Content Blocks

  # Text content block
  class TextBlock
    attr_accessor :text

    def initialize(text:)
      @text = text
    end
  end

  # Thinking content block
  class ThinkingBlock
    attr_accessor :thinking, :signature

    def initialize(thinking:, signature:)
      @thinking = thinking
      @signature = signature
    end
  end

  # Tool use content block
  class ToolUseBlock
    attr_accessor :id, :name, :input

    def initialize(id:, name:, input:)
      @id = id
      @name = name
      @input = input
    end
  end

  # Tool result content block
  class ToolResultBlock
    attr_accessor :tool_use_id, :content, :is_error

    def initialize(tool_use_id:, content: nil, is_error: nil)
      @tool_use_id = tool_use_id
      @content = content
      @is_error = is_error
    end
  end

  # Message Types

  # User message
  class UserMessage
    attr_accessor :content, :uuid, :parent_tool_use_id

    def initialize(content:, uuid: nil, parent_tool_use_id: nil)
      @content = content
      @uuid = uuid # Unique identifier for rewind support
      @parent_tool_use_id = parent_tool_use_id
    end
  end

  # Assistant message with content blocks
  class AssistantMessage
    attr_accessor :content, :model, :parent_tool_use_id, :error

    def initialize(content:, model:, parent_tool_use_id: nil, error: nil)
      @content = content
      @model = model
      @parent_tool_use_id = parent_tool_use_id
      @error = error # One of: authentication_failed, billing_error, rate_limit, invalid_request, server_error, unknown
    end
  end

  # System message with metadata
  class SystemMessage
    attr_accessor :subtype, :data

    def initialize(subtype:, data:)
      @subtype = subtype
      @data = data
    end
  end

  # Result message with cost and usage information
  class ResultMessage
    attr_accessor :subtype, :duration_ms, :duration_api_ms, :is_error,
                  :num_turns, :session_id, :total_cost_usd, :usage, :result, :structured_output

    def initialize(subtype:, duration_ms:, duration_api_ms:, is_error:,
                   num_turns:, session_id:, total_cost_usd: nil, usage: nil, result: nil, structured_output: nil)
      @subtype = subtype
      @duration_ms = duration_ms
      @duration_api_ms = duration_api_ms
      @is_error = is_error
      @num_turns = num_turns
      @session_id = session_id
      @total_cost_usd = total_cost_usd
      @usage = usage
      @result = result
      @structured_output = structured_output # Structured output when output_format is specified
    end
  end

  # Stream event for partial message updates
  class StreamEvent
    attr_accessor :uuid, :session_id, :event, :parent_tool_use_id

    def initialize(uuid:, session_id:, event:, parent_tool_use_id: nil)
      @uuid = uuid
      @session_id = session_id
      @event = event
      @parent_tool_use_id = parent_tool_use_id
    end
  end

  # Agent definition configuration
  class AgentDefinition
    attr_accessor :description, :prompt, :tools, :model

    def initialize(description:, prompt:, tools: nil, model: nil)
      @description = description
      @prompt = prompt
      @tools = tools
      @model = model
    end
  end

  # Permission rule value
  class PermissionRuleValue
    attr_accessor :tool_name, :rule_content

    def initialize(tool_name:, rule_content: nil)
      @tool_name = tool_name
      @rule_content = rule_content
    end
  end

  # Permission update configuration
  class PermissionUpdate
    attr_accessor :type, :rules, :behavior, :mode, :directories, :destination

    def initialize(type:, rules: nil, behavior: nil, mode: nil, directories: nil, destination: nil)
      @type = type
      @rules = rules
      @behavior = behavior
      @mode = mode
      @directories = directories
      @destination = destination
    end

    def to_h
      result = { type: @type }
      result[:destination] = @destination if @destination

      case @type
      when 'addRules', 'replaceRules', 'removeRules'
        if @rules
          result[:rules] = @rules.map do |rule|
            {
              toolName: rule.tool_name,
              ruleContent: rule.rule_content
            }
          end
        end
        result[:behavior] = @behavior if @behavior
      when 'setMode'
        result[:mode] = @mode if @mode
      when 'addDirectories', 'removeDirectories'
        result[:directories] = @directories if @directories
      end

      result
    end
  end

  # Tool permission context
  class ToolPermissionContext
    attr_accessor :signal, :suggestions

    def initialize(signal: nil, suggestions: [])
      @signal = signal
      @suggestions = suggestions
    end
  end

  # Permission results
  class PermissionResultAllow
    attr_accessor :behavior, :updated_input, :updated_permissions

    def initialize(updated_input: nil, updated_permissions: nil)
      @behavior = 'allow'
      @updated_input = updated_input
      @updated_permissions = updated_permissions
    end
  end

  class PermissionResultDeny
    attr_accessor :behavior, :message, :interrupt

    def initialize(message: '', interrupt: false)
      @behavior = 'deny'
      @message = message
      @interrupt = interrupt
    end
  end

  # Hook matcher configuration
  class HookMatcher
    attr_accessor :matcher, :hooks, :timeout

    def initialize(matcher: nil, hooks: [], timeout: nil)
      @matcher = matcher
      @hooks = hooks
      @timeout = timeout # Timeout in seconds for hook execution
    end
  end

  # Hook context passed to hook callbacks
  class HookContext
    attr_accessor :signal

    def initialize(signal: nil)
      @signal = signal
    end
  end

  # Base hook input with common fields
  class BaseHookInput
    attr_accessor :session_id, :transcript_path, :cwd, :permission_mode

    def initialize(session_id: nil, transcript_path: nil, cwd: nil, permission_mode: nil)
      @session_id = session_id
      @transcript_path = transcript_path
      @cwd = cwd
      @permission_mode = permission_mode
    end
  end

  # PreToolUse hook input
  class PreToolUseHookInput < BaseHookInput
    attr_accessor :hook_event_name, :tool_name, :tool_input

    def initialize(hook_event_name: 'PreToolUse', tool_name: nil, tool_input: nil, **base_args)
      super(**base_args)
      @hook_event_name = hook_event_name
      @tool_name = tool_name
      @tool_input = tool_input
    end
  end

  # PostToolUse hook input
  class PostToolUseHookInput < BaseHookInput
    attr_accessor :hook_event_name, :tool_name, :tool_input, :tool_response

    def initialize(hook_event_name: 'PostToolUse', tool_name: nil, tool_input: nil, tool_response: nil, **base_args)
      super(**base_args)
      @hook_event_name = hook_event_name
      @tool_name = tool_name
      @tool_input = tool_input
      @tool_response = tool_response
    end
  end

  # UserPromptSubmit hook input
  class UserPromptSubmitHookInput < BaseHookInput
    attr_accessor :hook_event_name, :prompt

    def initialize(hook_event_name: 'UserPromptSubmit', prompt: nil, **base_args)
      super(**base_args)
      @hook_event_name = hook_event_name
      @prompt = prompt
    end
  end

  # Stop hook input
  class StopHookInput < BaseHookInput
    attr_accessor :hook_event_name, :stop_hook_active

    def initialize(hook_event_name: 'Stop', stop_hook_active: false, **base_args)
      super(**base_args)
      @hook_event_name = hook_event_name
      @stop_hook_active = stop_hook_active
    end
  end

  # SubagentStop hook input
  class SubagentStopHookInput < BaseHookInput
    attr_accessor :hook_event_name, :stop_hook_active

    def initialize(hook_event_name: 'SubagentStop', stop_hook_active: false, **base_args)
      super(**base_args)
      @hook_event_name = hook_event_name
      @stop_hook_active = stop_hook_active
    end
  end

  # PreCompact hook input
  class PreCompactHookInput < BaseHookInput
    attr_accessor :hook_event_name, :trigger, :custom_instructions

    def initialize(hook_event_name: 'PreCompact', trigger: nil, custom_instructions: nil, **base_args)
      super(**base_args)
      @hook_event_name = hook_event_name
      @trigger = trigger
      @custom_instructions = custom_instructions
    end
  end

  # PreToolUse hook specific output
  class PreToolUseHookSpecificOutput
    attr_accessor :hook_event_name, :permission_decision, :permission_decision_reason, :updated_input

    def initialize(permission_decision: nil, permission_decision_reason: nil, updated_input: nil)
      @hook_event_name = 'PreToolUse'
      @permission_decision = permission_decision # 'allow', 'deny', or 'ask'
      @permission_decision_reason = permission_decision_reason
      @updated_input = updated_input
    end

    def to_h
      result = { hookEventName: @hook_event_name }
      result[:permissionDecision] = @permission_decision if @permission_decision
      result[:permissionDecisionReason] = @permission_decision_reason if @permission_decision_reason
      result[:updatedInput] = @updated_input if @updated_input
      result
    end
  end

  # PostToolUse hook specific output
  class PostToolUseHookSpecificOutput
    attr_accessor :hook_event_name, :additional_context

    def initialize(additional_context: nil)
      @hook_event_name = 'PostToolUse'
      @additional_context = additional_context
    end

    def to_h
      result = { hookEventName: @hook_event_name }
      result[:additionalContext] = @additional_context if @additional_context
      result
    end
  end

  # UserPromptSubmit hook specific output
  class UserPromptSubmitHookSpecificOutput
    attr_accessor :hook_event_name, :additional_context

    def initialize(additional_context: nil)
      @hook_event_name = 'UserPromptSubmit'
      @additional_context = additional_context
    end

    def to_h
      result = { hookEventName: @hook_event_name }
      result[:additionalContext] = @additional_context if @additional_context
      result
    end
  end

  # SessionStart hook specific output
  class SessionStartHookSpecificOutput
    attr_accessor :hook_event_name, :additional_context

    def initialize(additional_context: nil)
      @hook_event_name = 'SessionStart'
      @additional_context = additional_context
    end

    def to_h
      result = { hookEventName: @hook_event_name }
      result[:additionalContext] = @additional_context if @additional_context
      result
    end
  end

  # Async hook JSON output
  class AsyncHookJSONOutput
    attr_accessor :async, :async_timeout

    def initialize(async: true, async_timeout: nil)
      @async = async
      @async_timeout = async_timeout
    end

    def to_h
      result = { async: @async }
      result[:asyncTimeout] = @async_timeout if @async_timeout
      result
    end
  end

  # Sync hook JSON output
  class SyncHookJSONOutput
    attr_accessor :continue, :suppress_output, :stop_reason, :decision,
                  :system_message, :reason, :hook_specific_output

    def initialize(continue: true, suppress_output: false, stop_reason: nil, decision: nil,
                   system_message: nil, reason: nil, hook_specific_output: nil)
      @continue = continue
      @suppress_output = suppress_output
      @stop_reason = stop_reason
      @decision = decision
      @system_message = system_message
      @reason = reason
      @hook_specific_output = hook_specific_output
    end

    def to_h
      result = { continue: @continue }
      result[:suppressOutput] = @suppress_output if @suppress_output
      result[:stopReason] = @stop_reason if @stop_reason
      result[:decision] = @decision if @decision
      result[:systemMessage] = @system_message if @system_message
      result[:reason] = @reason if @reason
      result[:hookSpecificOutput] = @hook_specific_output.to_h if @hook_specific_output
      result
    end
  end

  # MCP Server configurations
  class McpStdioServerConfig
    attr_accessor :type, :command, :args, :env

    def initialize(command:, args: nil, env: nil, type: 'stdio')
      @type = type
      @command = command
      @args = args
      @env = env
    end

    def to_h
      result = { type: @type, command: @command }
      result[:args] = @args if @args
      result[:env] = @env if @env
      result
    end
  end

  class McpSSEServerConfig
    attr_accessor :type, :url, :headers

    def initialize(url:, headers: nil)
      @type = 'sse'
      @url = url
      @headers = headers
    end

    def to_h
      result = { type: @type, url: @url }
      result[:headers] = @headers if @headers
      result
    end
  end

  class McpHttpServerConfig
    attr_accessor :type, :url, :headers

    def initialize(url:, headers: nil)
      @type = 'http'
      @url = url
      @headers = headers
    end

    def to_h
      result = { type: @type, url: @url }
      result[:headers] = @headers if @headers
      result
    end
  end

  class McpSdkServerConfig
    attr_accessor :type, :name, :instance

    def initialize(name:, instance:)
      @type = 'sdk'
      @name = name
      @instance = instance
    end

    def to_h
      { type: @type, name: @name, instance: @instance }
    end
  end

  # SDK Plugin configuration
  class SdkPluginConfig
    attr_accessor :type, :path

    def initialize(path:)
      @type = 'plugin'
      @path = path
    end

    def to_h
      { type: @type, path: @path }
    end
  end

  # Sandbox network configuration
  class SandboxNetworkConfig
    attr_accessor :allow_unix_sockets, :allow_all_unix_sockets, :allow_local_binding,
                  :http_proxy_port, :socks_proxy_port

    def initialize(
      allow_unix_sockets: nil,
      allow_all_unix_sockets: nil,
      allow_local_binding: nil,
      http_proxy_port: nil,
      socks_proxy_port: nil
    )
      @allow_unix_sockets = allow_unix_sockets
      @allow_all_unix_sockets = allow_all_unix_sockets
      @allow_local_binding = allow_local_binding
      @http_proxy_port = http_proxy_port
      @socks_proxy_port = socks_proxy_port
    end

    def to_h
      result = {}
      result[:allowUnixSockets] = @allow_unix_sockets unless @allow_unix_sockets.nil?
      result[:allowAllUnixSockets] = @allow_all_unix_sockets unless @allow_all_unix_sockets.nil?
      result[:allowLocalBinding] = @allow_local_binding unless @allow_local_binding.nil?
      result[:httpProxyPort] = @http_proxy_port if @http_proxy_port
      result[:socksProxyPort] = @socks_proxy_port if @socks_proxy_port
      result
    end
  end

  # Sandbox ignore violations configuration
  class SandboxIgnoreViolations
    attr_accessor :file, :network

    def initialize(file: nil, network: nil)
      @file = file # Array of file paths to ignore
      @network = network # Array of network patterns to ignore
    end

    def to_h
      result = {}
      result[:file] = @file if @file
      result[:network] = @network if @network
      result
    end
  end

  # Sandbox settings for isolated command execution
  class SandboxSettings
    attr_accessor :enabled, :auto_allow_bash_if_sandboxed, :excluded_commands,
                  :allow_unsandboxed_commands, :network, :ignore_violations,
                  :enable_weaker_nested_sandbox

    def initialize(
      enabled: nil,
      auto_allow_bash_if_sandboxed: nil,
      excluded_commands: nil,
      allow_unsandboxed_commands: nil,
      network: nil,
      ignore_violations: nil,
      enable_weaker_nested_sandbox: nil
    )
      @enabled = enabled
      @auto_allow_bash_if_sandboxed = auto_allow_bash_if_sandboxed
      @excluded_commands = excluded_commands # Array of commands to exclude
      @allow_unsandboxed_commands = allow_unsandboxed_commands
      @network = network # SandboxNetworkConfig instance
      @ignore_violations = ignore_violations # SandboxIgnoreViolations instance
      @enable_weaker_nested_sandbox = enable_weaker_nested_sandbox
    end

    def to_h
      result = {}
      result[:enabled] = @enabled unless @enabled.nil?
      result[:autoAllowBashIfSandboxed] = @auto_allow_bash_if_sandboxed unless @auto_allow_bash_if_sandboxed.nil?
      result[:excludedCommands] = @excluded_commands if @excluded_commands
      result[:allowUnsandboxedCommands] = @allow_unsandboxed_commands unless @allow_unsandboxed_commands.nil?
      if @network
        result[:network] = @network.is_a?(SandboxNetworkConfig) ? @network.to_h : @network
      end
      if @ignore_violations
        result[:ignoreViolations] = @ignore_violations.is_a?(SandboxIgnoreViolations) ? @ignore_violations.to_h : @ignore_violations
      end
      result[:enableWeakerNestedSandbox] = @enable_weaker_nested_sandbox unless @enable_weaker_nested_sandbox.nil?
      result
    end
  end

  # System prompt preset configuration
  class SystemPromptPreset
    attr_accessor :type, :preset, :append

    def initialize(preset:, append: nil)
      @type = 'preset'
      @preset = preset
      @append = append
    end

    def to_h
      result = { type: @type, preset: @preset }
      result[:append] = @append if @append
      result
    end
  end

  # Tools preset configuration
  class ToolsPreset
    attr_accessor :type, :preset

    def initialize(preset:)
      @type = 'preset'
      @preset = preset
    end

    def to_h
      { type: @type, preset: @preset }
    end
  end

  # Claude Agent Options for configuring queries
  class ClaudeAgentOptions
    attr_accessor :allowed_tools, :system_prompt, :mcp_servers, :permission_mode,
                  :continue_conversation, :resume, :max_turns, :disallowed_tools,
                  :model, :permission_prompt_tool_name, :cwd, :cli_path, :settings,
                  :add_dirs, :env, :extra_args, :max_buffer_size, :stderr,
                  :can_use_tool, :hooks, :user, :include_partial_messages,
                  :fork_session, :agents, :setting_sources,
                  :output_format, :max_budget_usd, :max_thinking_tokens,
                  :fallback_model, :plugins, :debug_stderr,
                  :betas, :tools, :sandbox, :enable_file_checkpointing, :append_allowed_tools

    def initialize(
      allowed_tools: [],
      system_prompt: nil,
      mcp_servers: {},
      permission_mode: nil,
      continue_conversation: false,
      resume: nil,
      max_turns: nil,
      disallowed_tools: [],
      model: nil,
      permission_prompt_tool_name: nil,
      cwd: nil,
      cli_path: nil,
      settings: nil,
      add_dirs: [],
      env: {},
      extra_args: {},
      max_buffer_size: nil,
      stderr: nil,
      can_use_tool: nil,
      hooks: nil,
      user: nil,
      include_partial_messages: false,
      fork_session: false,
      agents: nil,
      setting_sources: nil,
      output_format: nil,
      max_budget_usd: nil,
      max_thinking_tokens: nil,
      fallback_model: nil,
      plugins: nil,
      debug_stderr: nil,
      betas: nil,
      tools: nil,
      sandbox: nil,
      enable_file_checkpointing: false,
      append_allowed_tools: nil
    )
      @allowed_tools = allowed_tools
      @system_prompt = system_prompt
      @mcp_servers = mcp_servers
      @permission_mode = permission_mode
      @continue_conversation = continue_conversation
      @resume = resume
      @max_turns = max_turns
      @disallowed_tools = disallowed_tools
      @model = model
      @permission_prompt_tool_name = permission_prompt_tool_name
      @cwd = cwd
      @cli_path = cli_path
      @settings = settings
      @add_dirs = add_dirs
      @env = env
      @extra_args = extra_args
      @max_buffer_size = max_buffer_size
      @stderr = stderr
      @can_use_tool = can_use_tool
      @hooks = hooks
      @user = user
      @include_partial_messages = include_partial_messages
      @fork_session = fork_session
      @agents = agents
      @setting_sources = setting_sources
      @output_format = output_format # JSON schema for structured output
      @max_budget_usd = max_budget_usd # Spending cap in dollars
      @max_thinking_tokens = max_thinking_tokens # Extended thinking token budget
      @fallback_model = fallback_model # Backup model if primary unavailable
      @plugins = plugins # Array of SdkPluginConfig
      @debug_stderr = debug_stderr # Debug output file object/path
      @betas = betas # Array of beta feature strings (e.g., ["context-1m-2025-08-07"])
      @tools = tools # Base tools selection: Array, empty array [], or ToolsPreset
      @sandbox = sandbox # SandboxSettings instance for isolated command execution
      @enable_file_checkpointing = enable_file_checkpointing # Enable file checkpointing for rewind support
      @append_allowed_tools = append_allowed_tools # Array of tools to append to allowed_tools
    end

    def dup_with(**changes)
      new_options = self.dup
      changes.each { |key, value| new_options.send("#{key}=", value) }
      new_options
    end
  end

  # SDK MCP Tool definition
  class SdkMcpTool
    attr_accessor :name, :description, :input_schema, :handler

    def initialize(name:, description:, input_schema:, handler:)
      @name = name
      @description = description
      @input_schema = input_schema
      @handler = handler
    end
  end

  # SDK MCP Resource definition
  class SdkMcpResource
    attr_accessor :uri, :name, :description, :mime_type, :reader

    def initialize(uri:, name:, description: nil, mime_type: nil, reader:)
      @uri = uri
      @name = name
      @description = description
      @mime_type = mime_type
      @reader = reader
    end
  end

  # SDK MCP Prompt definition
  class SdkMcpPrompt
    attr_accessor :name, :description, :arguments, :generator

    def initialize(name:, description: nil, arguments: nil, generator:)
      @name = name
      @description = description
      @arguments = arguments
      @generator = generator
    end
  end
end



================================================
FILE: lib/claude_agent_sdk/version.rb
================================================
# frozen_string_literal: true

module ClaudeAgentSDK
  VERSION = '0.4.0'
end



================================================
FILE: spec/README.md
================================================
# Test Suite

This directory contains the test suite for the Claude Agent SDK for Ruby.

## Running Tests

### Run all tests
```bash
bundle exec rspec
```

### Run with detailed output
```bash
bundle exec rspec --format documentation
```

### Run specific test file
```bash
bundle exec rspec spec/unit/message_parser_spec.rb
```

### Run specific test by line number
```bash
bundle exec rspec spec/unit/types_spec.rb:42
```

### Run with profiling (show slowest 10 tests)
```bash
PROFILE=1 bundle exec rspec
```

## Test Structure

### Unit Tests (`spec/unit/`)

Unit tests verify individual components in isolation:

- **errors_spec.rb** - Tests for error classes and their behavior
- **types_spec.rb** - Tests for all type classes (Messages, ContentBlocks, Options, etc.)
- **message_parser_spec.rb** - Tests for JSON message parsing and validation
- **sdk_mcp_server_spec.rb** - Tests for SDK MCP server functionality and tool execution
- **transport_spec.rb** - Tests for the Transport abstract base class

### Integration Tests (`spec/integration/`)

Integration tests verify how components work together:

- **query_spec.rb** - End-to-end workflow tests demonstrating component interaction

**Note:** Integration tests are tagged with `:integration` and are skipped by default. To run them:

```bash
RUN_INTEGRATION=1 bundle exec rspec
```

Integration tests that actually connect to Claude Code CLI require it to be installed.

### Test Helpers (`spec/support/`)

- **test_helpers.rb** - Shared test fixtures and helper methods used across test files

## Test Configuration

Test configuration is managed in `spec_helper.rb`:

- **Random order** - Tests run in random order to detect order dependencies
- **Persistence** - Test status is saved to `.rspec_status` for `--only-failures` and `--next-failure`
- **No monkey patching** - RSpec's clean syntax without global method pollution
- **Integration filter** - Integration tests skipped by default (enable with `RUN_INTEGRATION=1`)

## Test Coverage

The test suite covers:

### Error Handling (6 tests)
- All error class instantiation and attributes
- Error inheritance hierarchy
- Error message formatting

### Type System (24 tests)
- Content blocks (TextBlock, ThinkingBlock, ToolUseBlock, ToolResultBlock)
- Messages (UserMessage, AssistantMessage, SystemMessage, ResultMessage, StreamEvent)
- ClaudeAgentOptions with all configuration fields
- Permission types (PermissionResultAllow, PermissionResultDeny, PermissionUpdate)
- Hook matchers

### Message Parser (12 tests)
- Parsing all message types from JSON
- Content block parsing
- Validation and error handling for malformed messages
- Missing required field detection

### SDK MCP Server (21 tests)
- Tool creation with schemas
- Server configuration
- Tool execution and error handling
- JSON schema generation from Ruby types

### Transport (3 tests)
- Abstract base class interface
- NotImplementedError for unimplemented methods

### Integration (20 tests)
- Component interaction verification
- SDK MCP server integration
- Hook configuration and execution
- Permission callback handling
- End-to-end workflow simulation

**Total: 86 tests**

## Writing New Tests

### Test Fixtures

Use the fixtures provided in `spec/support/test_helpers.rb`:

```ruby
include TestHelpers

# Use sample messages
message = sample_user_message
result = sample_result_message
```

### Mock Transports

Create mock transports for testing without CLI:

```ruby
mock = mock_transport(
  messages: [sample_user_message, sample_result_message]
)
```

### Testing SDK Tools

```ruby
tool = ClaudeAgentSDK.create_tool('test_tool', 'Description', { arg: :string }) do |args|
  { content: [{ type: 'text', text: "Result: #{args[:arg]}" }] }
end

server = ClaudeAgentSDK.create_sdk_mcp_server(
  name: 'test_server',
  tools: [tool]
)

result = server[:instance].call_tool('test_tool', { arg: 'value' })
```

## Continuous Integration

In CI environments, tests run with:

- Documentation formatter for better output visibility
- Random seed for reproducibility
- Strict failure reporting

The test suite should always pass with 0 failures before merging.

## Troubleshooting

### Tests hanging or timing out

If tests hang, it may be due to:
- Process not terminating properly in transport tests
- Async operations not completing

### LoadError or require failures

Ensure dependencies are installed:

```bash
bundle install
```

### Integration tests failing

Integration tests require Claude Code CLI to be installed:

```bash
npm install -g @anthropic-ai/claude-code
```

Check installation:

```bash
which claude
claude -v
```

## Contributing

When adding new features:

1. Write unit tests for new classes/methods
2. Add integration tests for feature workflows
3. Update this README if adding new test categories
4. Ensure all tests pass: `bundle exec rspec`
5. Aim for comprehensive coverage of error cases and edge conditions



================================================
FILE: spec/spec_helper.rb
================================================
# frozen_string_literal: true

require 'claude_agent_sdk'

# Load test helpers
Dir[File.expand_path('support/**/*.rb', __dir__)].each { |f| require f }

RSpec.configure do |config|
  # Enable flags like --only-failures and --next-failure
  config.example_status_persistence_file_path = '.rspec_status'

  # Disable RSpec exposing methods globally on `Module` and `main`
  config.disable_monkey_patching!

  config.expect_with :rspec do |c|
    c.syntax = :expect
  end

  # Include test helpers
  config.include TestHelpers

  # Configure output format
  config.color = true
  config.tty = true
  config.formatter = :documentation if ENV['CI']

  # Run specs in random order to surface order dependencies
  config.order = :random
  Kernel.srand config.seed

  # Filter to skip slow integration tests by default
  config.filter_run_excluding :integration unless ENV['RUN_INTEGRATION']

  # Show the slowest examples
  config.profile_examples = 10 if ENV['PROFILE']
end



================================================
FILE: spec/integration/query_spec.rb
================================================
# frozen_string_literal: true

require 'spec_helper'

RSpec.describe 'Query Integration', :integration do
  # These integration tests demonstrate how the components work together
  # They don't actually connect to Claude Code CLI (would require it to be installed)
  # but show the expected flow

  describe 'ClaudeAgentSDK.query' do
    it 'has the correct method signature' do
      expect(ClaudeAgentSDK).to respond_to(:query)
    end

    it 'requires a prompt parameter' do
      # We can't actually test this without mocking heavily,
      # but we can verify the method exists
      expect { ClaudeAgentSDK.method(:query) }.not_to raise_error
    end

    # Note: Actual integration tests would require Claude Code CLI to be installed
    # and would be marked with :integration tag to be skipped in CI
  end

  describe 'ClaudeAgentSDK::Client' do
    it 'can be instantiated' do
      options = ClaudeAgentSDK::ClaudeAgentOptions.new
      client = ClaudeAgentSDK::Client.new(options: options)

      expect(client).to be_a(ClaudeAgentSDK::Client)
      expect(client).to respond_to(:connect)
      expect(client).to respond_to(:query)
      expect(client).to respond_to(:receive_messages)
      expect(client).to respond_to(:disconnect)
    end

    it 'has interrupt capability' do
      client = ClaudeAgentSDK::Client.new
      expect(client).to respond_to(:interrupt)
    end

    it 'can change permission mode' do
      client = ClaudeAgentSDK::Client.new
      expect(client).to respond_to(:set_permission_mode)
    end

    it 'can change model' do
      client = ClaudeAgentSDK::Client.new
      expect(client).to respond_to(:set_model)
    end

    it 'can get server info' do
      client = ClaudeAgentSDK::Client.new
      expect(client).to respond_to(:server_info)
    end
  end

  describe 'SDK MCP Server Integration' do
    it 'creates a working calculator server' do
      add_tool = ClaudeAgentSDK.create_tool('add', 'Add numbers', { a: :number, b: :number }) do |args|
        result = args[:a] + args[:b]
        { content: [{ type: 'text', text: "Result: #{result}" }] }
      end

      server_config = ClaudeAgentSDK.create_sdk_mcp_server(
        name: 'calculator',
        tools: [add_tool]
      )

      expect(server_config[:type]).to eq('sdk')
      expect(server_config[:instance]).to be_a(ClaudeAgentSDK::SdkMcpServer)

      # Verify the server works
      server = server_config[:instance]
      result = server.call_tool('add', { a: 10, b: 20 })
      expect(result[:content].first[:text]).to eq('Result: 30')
    end

    it 'can be used with ClaudeAgentOptions' do
      tool = ClaudeAgentSDK.create_tool('test', 'Test', {}) { |_| { content: [] } }
      server = ClaudeAgentSDK.create_sdk_mcp_server(name: 'test', tools: [tool])

      options = ClaudeAgentSDK::ClaudeAgentOptions.new(
        mcp_servers: { test: server },
        allowed_tools: ['mcp__test__test']
      )

      expect(options.mcp_servers[:test]).to eq(server)
      expect(options.allowed_tools).to include('mcp__test__test')
    end
  end

  describe 'Hook Integration' do
    it 'accepts hook configuration' do
      hook_fn = lambda do |input, tool_use_id, context|
        { hookSpecificOutput: { hookEventName: 'PreToolUse' } }
      end

      matcher = ClaudeAgentSDK::HookMatcher.new(
        matcher: 'Bash',
        hooks: [hook_fn]
      )

      options = ClaudeAgentSDK::ClaudeAgentOptions.new(
        hooks: { 'PreToolUse' => [matcher] }
      )

      expect(options.hooks).to have_key('PreToolUse')
      expect(options.hooks['PreToolUse'].first).to eq(matcher)
    end
  end

  describe 'Permission Callback Integration' do
    it 'accepts permission callback' do
      callback = lambda do |tool_name, input, context|
        ClaudeAgentSDK::PermissionResultAllow.new
      end

      options = ClaudeAgentSDK::ClaudeAgentOptions.new(
        can_use_tool: callback
      )

      expect(options.can_use_tool).to eq(callback)

      # Test the callback works
      result = callback.call('Read', {}, nil)
      expect(result).to be_a(ClaudeAgentSDK::PermissionResultAllow)
    end
  end

  describe 'End-to-end workflow simulation' do
    it 'demonstrates the expected flow' do
      # 1. Create tools
      add_tool = ClaudeAgentSDK.create_tool('add', 'Add', { a: :number, b: :number }) do |args|
        { content: [{ type: 'text', text: (args[:a] + args[:b]).to_s }] }
      end

      # 2. Create server
      server = ClaudeAgentSDK.create_sdk_mcp_server(name: 'calc', tools: [add_tool])

      # 3. Configure options with hooks and permissions
      permission_cb = lambda do |tool_name, _input, _context|
        tool_name == 'Read' ?
          ClaudeAgentSDK::PermissionResultAllow.new :
          ClaudeAgentSDK::PermissionResultDeny.new(message: 'Not allowed')
      end

      options = ClaudeAgentSDK::ClaudeAgentOptions.new(
        mcp_servers: { calc: server },
        allowed_tools: ['mcp__calc__add', 'Read'],
        can_use_tool: permission_cb,
        max_turns: 5
      )

      # 4. Verify configuration
      expect(options.mcp_servers[:calc]).to eq(server)
      expect(options.can_use_tool).to eq(permission_cb)

      # 5. Test permission callback
      result = permission_cb.call('Read', {}, nil)
      expect(result.behavior).to eq('allow')

      result = permission_cb.call('Write', {}, nil)
      expect(result.behavior).to eq('deny')

      # 6. Test tool execution
      calc_result = server[:instance].call_tool('add', { a: 5, b: 7 })
      expect(calc_result[:content].first[:text]).to eq('12')
    end
  end
end



================================================
FILE: spec/support/test_helpers.rb
================================================
# frozen_string_literal: true

module TestHelpers
  # Helper to create sample messages
  def sample_user_message
    {
      type: 'user',
      message: {
        role: 'user',
        content: 'Hello, Claude!'
      },
      parent_tool_use_id: nil
    }
  end

  def sample_assistant_message
    {
      type: 'assistant',
      message: {
        role: 'assistant',
        model: 'claude-sonnet-4',
        content: [
          { type: 'text', text: 'Hello! How can I help you?' }
        ]
      },
      parent_tool_use_id: nil
    }
  end

  def sample_assistant_message_with_tool_use
    {
      type: 'assistant',
      message: {
        role: 'assistant',
        model: 'claude-sonnet-4',
        content: [
          { type: 'text', text: "I'll help you with that." },
          {
            type: 'tool_use',
            id: 'toolu_123',
            name: 'Read',
            input: { file_path: '/path/to/file.rb' }
          }
        ]
      },
      parent_tool_use_id: nil
    }
  end

  def sample_result_message
    {
      type: 'result',
      subtype: 'success',
      duration_ms: 1500,
      duration_api_ms: 1000,
      is_error: false,
      num_turns: 1,
      session_id: 'test_session_123',
      total_cost_usd: 0.001234,
      usage: {
        input_tokens: 100,
        output_tokens: 50
      }
    }
  end

  def sample_system_message
    {
      type: 'system',
      subtype: 'info',
      message: 'Test system message'
    }
  end

  # Helper to create a mock transport
  def mock_transport
    double('Transport').tap do |transport|
      allow(transport).to receive(:connect)
      allow(transport).to receive(:close)
      allow(transport).to receive(:write)
      allow(transport).to receive(:ready?).and_return(true)
      allow(transport).to receive(:end_input)
    end
  end
end



================================================
FILE: spec/unit/errors_spec.rb
================================================
# frozen_string_literal: true

require 'spec_helper'

RSpec.describe ClaudeAgentSDK do
  describe 'Error Classes' do
    describe ClaudeAgentSDK::ClaudeSDKError do
      it 'is a StandardError' do
        expect(described_class).to be < StandardError
      end

      it 'can be raised with a message' do
        expect { raise described_class, 'Test error' }.to raise_error(described_class, 'Test error')
      end
    end

    describe ClaudeAgentSDK::CLIConnectionError do
      it 'inherits from ClaudeSDKError' do
        expect(described_class).to be < ClaudeAgentSDK::ClaudeSDKError
      end

      it 'can be raised with a message' do
        expect { raise described_class, 'Connection failed' }
          .to raise_error(described_class, 'Connection failed')
      end
    end

    describe ClaudeAgentSDK::CLINotFoundError do
      it 'inherits from CLIConnectionError' do
        expect(described_class).to be < ClaudeAgentSDK::CLIConnectionError
      end

      it 'has a default message' do
        error = described_class.new
        expect(error.message).to eq('Claude Code not found')
      end

      it 'can include CLI path in message' do
        error = described_class.new('Claude Code not found', cli_path: '/usr/bin/claude')
        expect(error.message).to include('/usr/bin/claude')
      end
    end

    describe ClaudeAgentSDK::ProcessError do
      it 'inherits from ClaudeSDKError' do
        expect(described_class).to be < ClaudeAgentSDK::ClaudeSDKError
      end

      it 'stores exit code' do
        error = described_class.new('Process failed', exit_code: 1)
        expect(error.exit_code).to eq(1)
      end

      it 'stores stderr output' do
        error = described_class.new('Process failed', stderr: 'Error output')
        expect(error.stderr).to eq('Error output')
      end

      it 'includes exit code in message' do
        error = described_class.new('Process failed', exit_code: 1)
        expect(error.message).to include('exit code: 1')
      end

      it 'includes stderr in message' do
        error = described_class.new('Process failed', stderr: 'Error output')
        expect(error.message).to include('Error output')
      end
    end

    describe ClaudeAgentSDK::CLIJSONDecodeError do
      it 'inherits from ClaudeSDKError' do
        expect(described_class).to be < ClaudeAgentSDK::ClaudeSDKError
      end

      it 'stores the line that failed to parse' do
        original = StandardError.new('Invalid JSON')
        error = described_class.new('invalid json', original)
        expect(error.line).to eq('invalid json')
      end

      it 'stores the original error' do
        original = StandardError.new('Invalid JSON')
        error = described_class.new('invalid json', original)
        expect(error.original_error).to eq(original)
      end
    end

    describe ClaudeAgentSDK::MessageParseError do
      it 'inherits from ClaudeSDKError' do
        expect(described_class).to be < ClaudeAgentSDK::ClaudeSDKError
      end

      it 'stores the data that failed to parse' do
        data = { type: 'unknown' }
        error = described_class.new('Failed to parse', data: data)
        expect(error.data).to eq(data)
      end
    end
  end
end



================================================
FILE: spec/unit/message_parser_spec.rb
================================================
# frozen_string_literal: true

require 'spec_helper'

RSpec.describe ClaudeAgentSDK::MessageParser do
  include TestHelpers

  describe '.parse' do
    it 'raises error for non-hash input' do
      expect { described_class.parse('not a hash') }
        .to raise_error(ClaudeAgentSDK::MessageParseError, /Invalid message data type/)
    end

    it 'raises error for missing type field' do
      expect { described_class.parse({}) }
        .to raise_error(ClaudeAgentSDK::MessageParseError, /missing 'type' field/)
    end

    it 'raises error for unknown message type' do
      expect { described_class.parse({ type: 'unknown' }) }
        .to raise_error(ClaudeAgentSDK::MessageParseError, /Unknown message type/)
    end

    context 'user messages' do
      it 'parses user message with string content' do
        data = {
          type: 'user',
          message: { content: 'Hello' },
          parent_tool_use_id: nil
        }

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::UserMessage)
        expect(msg.content).to eq('Hello')
      end

      it 'parses user message with content blocks' do
        data = {
          type: 'user',
          message: {
            content: [
              { type: 'text', text: 'Hello' }
            ]
          }
        }

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::UserMessage)
        expect(msg.content).to be_an(Array)
        expect(msg.content.first).to be_a(ClaudeAgentSDK::TextBlock)
        expect(msg.content.first.text).to eq('Hello')
      end

      it 'includes parent_tool_use_id if present' do
        data = {
          type: 'user',
          message: { content: 'Hello' },
          parent_tool_use_id: 'tool_123'
        }

        msg = described_class.parse(data)
        expect(msg.parent_tool_use_id).to eq('tool_123')
      end

      it 'parses uuid for rewind support' do
        data = {
          type: 'user',
          message: { content: 'Hello' },
          uuid: 'user_msg_abc123'
        }

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::UserMessage)
        expect(msg.uuid).to eq('user_msg_abc123')
      end

      it 'handles missing uuid gracefully' do
        data = {
          type: 'user',
          message: { content: 'Hello' }
        }

        msg = described_class.parse(data)
        expect(msg.uuid).to be_nil
      end
    end

    context 'assistant messages' do
      it 'parses assistant message with text blocks' do
        data = sample_assistant_message

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::AssistantMessage)
        expect(msg.model).to eq('claude-sonnet-4')
        expect(msg.content.first).to be_a(ClaudeAgentSDK::TextBlock)
      end

      it 'parses assistant message with tool use' do
        data = sample_assistant_message_with_tool_use

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::AssistantMessage)
        expect(msg.content.length).to eq(2)

        tool_use = msg.content[1]
        expect(tool_use).to be_a(ClaudeAgentSDK::ToolUseBlock)
        expect(tool_use.id).to eq('toolu_123')
        expect(tool_use.name).to eq('Read')
        expect(tool_use.input).to eq({ file_path: '/path/to/file.rb' })
      end

      it 'parses thinking blocks' do
        data = {
          type: 'assistant',
          message: {
            model: 'claude-sonnet-4',
            content: [
              { type: 'thinking', thinking: 'Let me think...', signature: 'sig123' }
            ]
          }
        }

        msg = described_class.parse(data)
        thinking = msg.content.first
        expect(thinking).to be_a(ClaudeAgentSDK::ThinkingBlock)
        expect(thinking.thinking).to eq('Let me think...')
      end

      it 'parses error field' do
        data = {
          type: 'assistant',
          message: {
            model: 'claude-sonnet-4',
            content: [
              { type: 'text', text: 'Error occurred' }
            ]
          },
          error: 'rate_limit'
        }

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::AssistantMessage)
        expect(msg.error).to eq('rate_limit')
      end
    end

    context 'system messages' do
      it 'parses system messages' do
        data = sample_system_message

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::SystemMessage)
        expect(msg.subtype).to eq('info')
        expect(msg.data).to include(message: 'Test system message')
      end
    end

    context 'result messages' do
      it 'parses result messages' do
        data = sample_result_message

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::ResultMessage)
        expect(msg.subtype).to eq('success')
        expect(msg.duration_ms).to eq(1500)
        expect(msg.is_error).to eq(false)
        expect(msg.session_id).to eq('test_session_123')
        expect(msg.total_cost_usd).to eq(0.001234)
      end

      it 'handles optional fields' do
        data = {
          type: 'result',
          subtype: 'success',
          duration_ms: 1000,
          duration_api_ms: 800,
          is_error: false,
          num_turns: 1,
          session_id: 'test'
        }

        msg = described_class.parse(data)
        expect(msg.total_cost_usd).to be_nil
        expect(msg.usage).to be_nil
      end

      it 'parses structured_output' do
        data = {
          type: 'result',
          subtype: 'success',
          duration_ms: 1000,
          duration_api_ms: 800,
          is_error: false,
          num_turns: 1,
          session_id: 'test',
          structured_output: { name: 'John', age: 30, active: true }
        }

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::ResultMessage)
        expect(msg.structured_output).to eq({ name: 'John', age: 30, active: true })
      end
    end

    context 'stream events' do
      it 'parses stream events' do
        data = {
          type: 'stream_event',
          uuid: 'evt_123',
          session_id: 'session_123',
          event: { type: 'message_start' },
          parent_tool_use_id: nil
        }

        msg = described_class.parse(data)
        expect(msg).to be_a(ClaudeAgentSDK::StreamEvent)
        expect(msg.uuid).to eq('evt_123')
        expect(msg.event).to eq({ type: 'message_start' })
      end
    end

    context 'error handling' do
      it 'raises error for malformed user message' do
        data = { type: 'user' } # Missing message field

        expect { described_class.parse(data) }
          .to raise_error(ClaudeAgentSDK::MessageParseError)
      end

      it 'raises error for malformed assistant message' do
        data = { type: 'assistant', message: {} } # Missing content

        expect { described_class.parse(data) }
          .to raise_error(ClaudeAgentSDK::MessageParseError)
      end
    end
  end
end



================================================
FILE: spec/unit/sdk_mcp_server_spec.rb
================================================
# frozen_string_literal: true

require 'spec_helper'

RSpec.describe ClaudeAgentSDK::SdkMcpServer do
  describe '#initialize' do
    it 'creates a server with name and version' do
      server = described_class.new(name: 'test-server', version: '1.0.0', tools: [])
      expect(server.name).to eq('test-server')
      expect(server.version).to eq('1.0.0')
    end

    it 'defaults version to 1.0.0' do
      server = described_class.new(name: 'test-server')
      expect(server.version).to eq('1.0.0')
    end

    it 'accepts tools array' do
      tool = ClaudeAgentSDK::SdkMcpTool.new(
        name: 'test_tool',
        description: 'Test',
        input_schema: {},
        handler: ->(_) { {} }
      )
      server = described_class.new(name: 'test', tools: [tool])
      expect(server.tools).to eq([tool])
    end
  end

  describe '#list_tools' do
    it 'returns empty array for no tools' do
      server = described_class.new(name: 'test')
      expect(server.list_tools).to eq([])
    end

    it 'returns tool definitions with JSON schemas' do
      tool = ClaudeAgentSDK::SdkMcpTool.new(
        name: 'add',
        description: 'Add numbers',
        input_schema: { a: :number, b: :number },
        handler: ->(_) { {} }
      )
      server = described_class.new(name: 'calc', tools: [tool])

      tools = server.list_tools
      expect(tools.length).to eq(1)
      expect(tools.first[:name]).to eq('add')
      expect(tools.first[:description]).to eq('Add numbers')
      expect(tools.first[:inputSchema][:type]).to eq('object')
      expect(tools.first[:inputSchema][:properties]).to have_key(:a)
      expect(tools.first[:inputSchema][:properties][:a][:type]).to eq('number')
    end

    it 'converts Ruby types to JSON schema types' do
      tool = ClaudeAgentSDK::SdkMcpTool.new(
        name: 'test',
        description: 'Test',
        input_schema: {
          str: :string,
          num: :number,
          int: :integer,
          bool: :boolean
        },
        handler: ->(_) { {} }
      )
      server = described_class.new(name: 'test', tools: [tool])

      schema = server.list_tools.first[:inputSchema]
      expect(schema[:properties][:str][:type]).to eq('string')
      expect(schema[:properties][:num][:type]).to eq('number')
      expect(schema[:properties][:int][:type]).to eq('integer')
      expect(schema[:properties][:bool][:type]).to eq('boolean')
    end

    it 'handles pre-formatted JSON schemas' do
      tool = ClaudeAgentSDK::SdkMcpTool.new(
        name: 'test',
        description: 'Test',
        input_schema: {
          type: 'object',
          properties: { custom: { type: 'string' } }
        },
        handler: ->(_) { {} }
      )
      server = described_class.new(name: 'test', tools: [tool])

      schema = server.list_tools.first[:inputSchema]
      expect(schema[:type]).to eq('object')
      expect(schema[:properties][:custom][:type]).to eq('string')
    end
  end

  describe '#call_tool' do
    it 'executes tool handler with arguments' do
      handler = lambda do |args|
        result = args[:a] + args[:b]
        { content: [{ type: 'text', text: "Result: #{result}" }] }
      end

      tool = ClaudeAgentSDK::SdkMcpTool.new(
        name: 'add',
        description: 'Add',
        input_schema: {},
        handler: handler
      )

      server = described_class.new(name: 'calc', tools: [tool])
      result = server.call_tool('add', { a: 5, b: 3 })

      expect(result[:content]).to be_an(Array)
      expect(result[:content].first[:text]).to eq('Result: 8')
    end

    it 'raises error for unknown tool' do
      server = described_class.new(name: 'test')
      expect { server.call_tool('unknown', {}) }
        .to raise_error(/Tool 'unknown' not found/)
    end

    it 'passes through error results' do
      handler = lambda do |args|
        if args[:n] < 0
          { content: [{ type: 'text', text: 'Error: negative number' }], is_error: true }
        else
          { content: [{ type: 'text', text: 'OK' }] }
        end
      end

      tool = ClaudeAgentSDK::SdkMcpTool.new(
        name: 'sqrt',
        description: 'Square root',
        input_schema: {},
        handler: handler
      )

      server = described_class.new(name: 'math', tools: [tool])
      result = server.call_tool('sqrt', { n: -1 })

      expect(result[:is_error]).to eq(true)
      expect(result[:content].first[:text]).to include('Error')
    end

    it 'raises error if handler returns invalid format' do
      handler = ->(_) { 'invalid' } # Should return hash with :content

      tool = ClaudeAgentSDK::SdkMcpTool.new(
        name: 'bad',
        description: 'Bad',
        input_schema: {},
        handler: handler
      )

      server = described_class.new(name: 'test', tools: [tool])
      expect { server.call_tool('bad', {}) }
        .to raise_error(/must return a hash with :content key/)
    end
  end
end

RSpec.describe ClaudeAgentSDK, '.create_tool' do
  it 'creates a tool with name, description, and schema' do
    tool = described_class.create_tool('greet', 'Greet user', { name: :string }) do |args|
      { content: [{ type: 'text', text: "Hello, #{args[:name]}!" }] }
    end

    expect(tool).to be_a(ClaudeAgentSDK::SdkMcpTool)
    expect(tool.name).to eq('greet')
    expect(tool.description).to eq('Greet user')
    expect(tool.handler).to be_a(Proc)
  end

  it 'requires a block' do
    expect { described_class.create_tool('test', 'Test', {}) }
      .to raise_error(ArgumentError, /Block required/)
  end

  it 'handler executes correctly' do
    tool = described_class.create_tool('add', 'Add', { a: :number, b: :number }) do |args|
      { content: [{ type: 'text', text: (args[:a] + args[:b]).to_s }] }
    end

    result = tool.handler.call({ a: 2, b: 3 })
    expect(result[:content].first[:text]).to eq('5')
  end
end

RSpec.describe ClaudeAgentSDK, '.create_sdk_mcp_server' do
  it 'creates an SDK MCP server configuration' do
    tool = described_class.create_tool('test', 'Test', {}) { |_| { content: [] } }
    server_config = described_class.create_sdk_mcp_server(name: 'test-server', tools: [tool])

    expect(server_config).to be_a(Hash)
    expect(server_config[:type]).to eq('sdk')
    expect(server_config[:name]).to eq('test-server')
    expect(server_config[:instance]).to be_a(ClaudeAgentSDK::SdkMcpServer)
  end

  it 'defaults version to 1.0.0' do
    server_config = described_class.create_sdk_mcp_server(name: 'test')
    expect(server_config[:instance].version).to eq('1.0.0')
  end

  it 'accepts custom version' do
    server_config = described_class.create_sdk_mcp_server(name: 'test', version: '2.0.0')
    expect(server_config[:instance].version).to eq('2.0.0')
  end

  it 'creates functional calculator example' do
    add_tool = described_class.create_tool('add', 'Add numbers', { a: :number, b: :number }) do |args|
      result = args[:a] + args[:b]
      { content: [{ type: 'text', text: "#{args[:a]} + #{args[:b]} = #{result}" }] }
    end

    server_config = described_class.create_sdk_mcp_server(
      name: 'calculator',
      version: '1.0.0',
      tools: [add_tool]
    )

    server = server_config[:instance]
    tools = server.list_tools
    expect(tools.length).to eq(1)
    expect(tools.first[:name]).to eq('add')

    result = server.call_tool('add', { a: 15, b: 27 })
    expect(result[:content].first[:text]).to eq('15 + 27 = 42')
  end
end



================================================
FILE: spec/unit/transport_spec.rb
================================================
# frozen_string_literal: true

require 'spec_helper'

RSpec.describe ClaudeAgentSDK::Transport do
  # Create a test implementation
  let(:test_transport_class) do
    Class.new(described_class) do
      attr_accessor :connected, :messages, :written_data

      def initialize
        @connected = false
        @messages = []
        @written_data = []
      end

      def connect
        @connected = true
      end

      def write(data)
        @written_data << data
      end

      def read_messages(&block)
        @messages.each { |msg| block.call(msg) }
      end

      def close
        @connected = false
      end

      def ready?
        @connected
      end

      def end_input
        # No-op for test
      end
    end
  end

  let(:transport) { test_transport_class.new }

  describe 'interface requirements' do
    it 'requires connect to be implemented' do
      expect(transport).to respond_to(:connect)
    end

    it 'requires write to be implemented' do
      expect(transport).to respond_to(:write)
    end

    it 'requires read_messages to be implemented' do
      expect(transport).to respond_to(:read_messages)
    end

    it 'requires close to be implemented' do
      expect(transport).to respond_to(:close)
    end

    it 'requires ready? to be implemented' do
      expect(transport).to respond_to(:ready?)
    end

    it 'requires end_input to be implemented' do
      expect(transport).to respond_to(:end_input)
    end
  end

  describe 'test implementation' do
    it 'connects successfully' do
      expect(transport.ready?).to eq(false)
      transport.connect
      expect(transport.ready?).to eq(true)
    end

    it 'writes data' do
      transport.write('test data')
      expect(transport.written_data).to eq(['test data'])
    end

    it 'reads messages with block' do
      transport.messages = [{ type: 'test1' }, { type: 'test2' }]

      received = []
      transport.read_messages { |msg| received << msg }

      expect(received).to eq([{ type: 'test1' }, { type: 'test2' }])
    end

    it 'closes successfully' do
      transport.connect
      expect(transport.ready?).to eq(true)

      transport.close
      expect(transport.ready?).to eq(false)
    end
  end

  describe 'abstract class behavior' do
    let(:abstract_transport) { described_class.new }

    it 'raises NotImplementedError for connect' do
      expect { abstract_transport.connect }
        .to raise_error(NotImplementedError, /implement #connect/)
    end

    it 'raises NotImplementedError for write' do
      expect { abstract_transport.write('data') }
        .to raise_error(NotImplementedError, /implement #write/)
    end

    it 'raises NotImplementedError for read_messages' do
      expect { abstract_transport.read_messages }
        .to raise_error(NotImplementedError, /implement #read_messages/)
    end

    it 'raises NotImplementedError for close' do
      expect { abstract_transport.close }
        .to raise_error(NotImplementedError, /implement #close/)
    end

    it 'raises NotImplementedError for ready?' do
      expect { abstract_transport.ready? }
        .to raise_error(NotImplementedError, /implement #ready/)
    end

    it 'raises NotImplementedError for end_input' do
      expect { abstract_transport.end_input }
        .to raise_error(NotImplementedError, /implement #end_input/)
    end
  end
end



================================================
FILE: spec/unit/types_spec.rb
================================================
# frozen_string_literal: true

require 'spec_helper'

RSpec.describe ClaudeAgentSDK do
  describe 'Type Classes' do
    describe ClaudeAgentSDK::TextBlock do
      it 'stores text content' do
        block = described_class.new(text: 'Hello, world!')
        expect(block.text).to eq('Hello, world!')
      end
    end

    describe ClaudeAgentSDK::ThinkingBlock do
      it 'stores thinking content and signature' do
        block = described_class.new(thinking: 'Let me think...', signature: 'sig123')
        expect(block.thinking).to eq('Let me think...')
        expect(block.signature).to eq('sig123')
      end
    end

    describe ClaudeAgentSDK::ToolUseBlock do
      it 'stores tool use information' do
        block = described_class.new(id: 'tool_123', name: 'Read', input: { file_path: '/test' })
        expect(block.id).to eq('tool_123')
        expect(block.name).to eq('Read')
        expect(block.input).to eq({ file_path: '/test' })
      end
    end

    describe ClaudeAgentSDK::ToolResultBlock do
      it 'stores tool result' do
        block = described_class.new(tool_use_id: 'tool_123', content: 'Result', is_error: false)
        expect(block.tool_use_id).to eq('tool_123')
        expect(block.content).to eq('Result')
        expect(block.is_error).to eq(false)
      end
    end

    describe ClaudeAgentSDK::UserMessage do
      it 'stores user content as string' do
        msg = described_class.new(content: 'Hello')
        expect(msg.content).to eq('Hello')
      end

      it 'stores user content as blocks' do
        blocks = [ClaudeAgentSDK::TextBlock.new(text: 'Hello')]
        msg = described_class.new(content: blocks)
        expect(msg.content).to eq(blocks)
      end

      it 'optionally stores parent_tool_use_id' do
        msg = described_class.new(content: 'Hello', parent_tool_use_id: 'tool_123')
        expect(msg.parent_tool_use_id).to eq('tool_123')
      end

      it 'optionally stores uuid for rewind support' do
        msg = described_class.new(content: 'Hello', uuid: 'user_msg_abc123')
        expect(msg.uuid).to eq('user_msg_abc123')
      end

      it 'has nil uuid by default' do
        msg = described_class.new(content: 'Hello')
        expect(msg.uuid).to be_nil
      end
    end

    describe ClaudeAgentSDK::AssistantMessage do
      it 'stores assistant content' do
        blocks = [ClaudeAgentSDK::TextBlock.new(text: 'Hello')]
        msg = described_class.new(content: blocks, model: 'claude-sonnet-4')
        expect(msg.content).to eq(blocks)
        expect(msg.model).to eq('claude-sonnet-4')
      end

      it 'stores error field' do
        blocks = [ClaudeAgentSDK::TextBlock.new(text: 'Error')]
        msg = described_class.new(content: blocks, model: 'claude-sonnet-4', error: 'rate_limit')
        expect(msg.error).to eq('rate_limit')
      end

      it 'accepts all valid error types' do
        blocks = [ClaudeAgentSDK::TextBlock.new(text: 'Error')]
        %w[authentication_failed billing_error rate_limit invalid_request server_error unknown].each do |error_type|
          msg = described_class.new(content: blocks, model: 'claude-sonnet-4', error: error_type)
          expect(msg.error).to eq(error_type)
        end
      end
    end

    describe ClaudeAgentSDK::SystemMessage do
      it 'stores system message data' do
        msg = described_class.new(subtype: 'info', data: { message: 'Test' })
        expect(msg.subtype).to eq('info')
        expect(msg.data).to eq({ message: 'Test' })
      end
    end

    describe ClaudeAgentSDK::ResultMessage do
      it 'stores result information' do
        msg = described_class.new(
          subtype: 'success',
          duration_ms: 1000,
          duration_api_ms: 800,
          is_error: false,
          num_turns: 1,
          session_id: 'session_123',
          total_cost_usd: 0.01,
          usage: { input_tokens: 100 }
        )

        expect(msg.subtype).to eq('success')
        expect(msg.duration_ms).to eq(1000)
        expect(msg.is_error).to eq(false)
        expect(msg.total_cost_usd).to eq(0.01)
      end

      it 'stores structured_output' do
        msg = described_class.new(
          subtype: 'success',
          duration_ms: 1000,
          duration_api_ms: 800,
          is_error: false,
          num_turns: 1,
          session_id: 'session_123',
          structured_output: { name: 'John', age: 30 }
        )

        expect(msg.structured_output).to eq({ name: 'John', age: 30 })
      end
    end

    describe ClaudeAgentSDK::PermissionUpdate do
      it 'converts to hash format' do
        rule = ClaudeAgentSDK::PermissionRuleValue.new(tool_name: 'Bash', rule_content: 'echo')
        update = described_class.new(
          type: 'addRules',
          rules: [rule],
          behavior: 'allow',
          destination: 'session'
        )

        hash = update.to_h
        expect(hash[:type]).to eq('addRules')
        expect(hash[:behavior]).to eq('allow')
        expect(hash[:rules].first[:toolName]).to eq('Bash')
      end
    end

    describe ClaudeAgentSDK::PermissionResultAllow do
      it 'has allow behavior' do
        result = described_class.new
        expect(result.behavior).to eq('allow')
      end

      it 'optionally stores updated input' do
        result = described_class.new(updated_input: { modified: true })
        expect(result.updated_input).to eq({ modified: true })
      end
    end

    describe ClaudeAgentSDK::PermissionResultDeny do
      it 'has deny behavior' do
        result = described_class.new(message: 'Not allowed')
        expect(result.behavior).to eq('deny')
        expect(result.message).to eq('Not allowed')
      end

      it 'can interrupt' do
        result = described_class.new(interrupt: true)
        expect(result.interrupt).to eq(true)
      end
    end

    describe ClaudeAgentSDK::ClaudeAgentOptions do
      it 'has default values' do
        options = described_class.new
        expect(options.allowed_tools).to eq([])
        expect(options.mcp_servers).to eq({})
        expect(options.continue_conversation).to eq(false)
        expect(options.output_format).to be_nil
        expect(options.max_budget_usd).to be_nil
        expect(options.max_thinking_tokens).to be_nil
        expect(options.fallback_model).to be_nil
        expect(options.plugins).to be_nil
        expect(options.debug_stderr).to be_nil
      end

      it 'accepts configuration' do
        options = described_class.new(
          allowed_tools: ['Read', 'Write'],
          permission_mode: 'acceptEdits',
          max_turns: 5
        )

        expect(options.allowed_tools).to eq(['Read', 'Write'])
        expect(options.permission_mode).to eq('acceptEdits')
        expect(options.max_turns).to eq(5)
      end

      it 'accepts new Python SDK options' do
        output_schema = {
          type: 'object',
          properties: {
            name: { type: 'string' },
            age: { type: 'integer' }
          }
        }
        plugin = ClaudeAgentSDK::SdkPluginConfig.new(path: '/plugin')

        options = described_class.new(
          output_format: output_schema,
          max_budget_usd: 10.0,
          max_thinking_tokens: 5000,
          fallback_model: 'claude-haiku-3',
          plugins: [plugin],
          debug_stderr: '/tmp/debug.log'
        )

        expect(options.output_format).to eq(output_schema)
        expect(options.max_budget_usd).to eq(10.0)
        expect(options.max_thinking_tokens).to eq(5000)
        expect(options.fallback_model).to eq('claude-haiku-3')
        expect(options.plugins).to eq([plugin])
        expect(options.debug_stderr).to eq('/tmp/debug.log')
      end

      it 'can duplicate with changes' do
        options = described_class.new(max_turns: 5)
        new_options = options.dup_with(max_turns: 10, model: 'claude-opus-4')

        expect(new_options.max_turns).to eq(10)
        expect(new_options.model).to eq('claude-opus-4')
      end
    end

    describe ClaudeAgentSDK::HookMatcher do
      it 'stores matcher and hooks' do
        hook_fn = ->(_input, _tool_id, _context) { {} }
        matcher = described_class.new(matcher: 'Bash', hooks: [hook_fn])

        expect(matcher.matcher).to eq('Bash')
        expect(matcher.hooks).to eq([hook_fn])
      end

      it 'stores timeout' do
        matcher = described_class.new(matcher: 'Bash', timeout: 30)
        expect(matcher.timeout).to eq(30)
      end
    end

    describe ClaudeAgentSDK::HookContext do
      it 'stores signal' do
        context = described_class.new(signal: :test_signal)
        expect(context.signal).to eq(:test_signal)
      end
    end

    describe ClaudeAgentSDK::PreToolUseHookInput do
      it 'stores hook input fields' do
        input = described_class.new(
          tool_name: 'Bash',
          tool_input: { command: 'ls' },
          session_id: 'sess_123',
          cwd: '/home/user'
        )

        expect(input.hook_event_name).to eq('PreToolUse')
        expect(input.tool_name).to eq('Bash')
        expect(input.tool_input).to eq({ command: 'ls' })
        expect(input.session_id).to eq('sess_123')
        expect(input.cwd).to eq('/home/user')
      end
    end

    describe ClaudeAgentSDK::PostToolUseHookInput do
      it 'stores hook input with tool response' do
        input = described_class.new(
          tool_name: 'Bash',
          tool_input: { command: 'ls' },
          tool_response: 'file1.txt\nfile2.txt'
        )

        expect(input.hook_event_name).to eq('PostToolUse')
        expect(input.tool_response).to eq('file1.txt\nfile2.txt')
      end
    end

    describe ClaudeAgentSDK::PreToolUseHookSpecificOutput do
      it 'converts to CLI format' do
        output = described_class.new(
          permission_decision: 'deny',
          permission_decision_reason: 'Command not allowed'
        )

        hash = output.to_h
        expect(hash[:hookEventName]).to eq('PreToolUse')
        expect(hash[:permissionDecision]).to eq('deny')
        expect(hash[:permissionDecisionReason]).to eq('Command not allowed')
      end
    end

    describe ClaudeAgentSDK::SyncHookJSONOutput do
      it 'converts to CLI format' do
        specific = ClaudeAgentSDK::PreToolUseHookSpecificOutput.new(
          permission_decision: 'allow'
        )
        output = described_class.new(
          continue: true,
          suppress_output: false,
          hook_specific_output: specific
        )

        hash = output.to_h
        expect(hash[:continue]).to eq(true)
        expect(hash[:hookSpecificOutput][:hookEventName]).to eq('PreToolUse')
      end
    end

    describe ClaudeAgentSDK::SdkPluginConfig do
      it 'stores plugin path' do
        plugin = described_class.new(path: '/path/to/plugin')
        expect(plugin.type).to eq('plugin')
        expect(plugin.path).to eq('/path/to/plugin')
      end

      it 'converts to hash' do
        plugin = described_class.new(path: '/path/to/plugin')
        hash = plugin.to_h
        expect(hash[:type]).to eq('plugin')
        expect(hash[:path]).to eq('/path/to/plugin')
      end
    end

    describe ClaudeAgentSDK::SdkMcpTool do
      it 'stores tool definition' do
        handler = ->(_args) { { content: [] } }
        tool = described_class.new(
          name: 'test_tool',
          description: 'A test tool',
          input_schema: { param: :string },
          handler: handler
        )

        expect(tool.name).to eq('test_tool')
        expect(tool.description).to eq('A test tool')
        expect(tool.handler).to eq(handler)
      end
    end

    describe ClaudeAgentSDK::SandboxNetworkConfig do
      it 'stores network configuration' do
        config = described_class.new(
          allow_unix_sockets: ['/tmp/socket'],
          allow_local_binding: true,
          http_proxy_port: 8080
        )

        expect(config.allow_unix_sockets).to eq(['/tmp/socket'])
        expect(config.allow_local_binding).to eq(true)
        expect(config.http_proxy_port).to eq(8080)
      end

      it 'converts to hash with camelCase keys' do
        config = described_class.new(
          allow_local_binding: true,
          http_proxy_port: 8080
        )

        hash = config.to_h
        expect(hash[:allowLocalBinding]).to eq(true)
        expect(hash[:httpProxyPort]).to eq(8080)
      end

      it 'omits nil values in hash' do
        config = described_class.new(allow_local_binding: true)
        hash = config.to_h

        expect(hash.key?(:allowLocalBinding)).to eq(true)
        expect(hash.key?(:allowUnixSockets)).to eq(false)
      end
    end

    describe ClaudeAgentSDK::SandboxIgnoreViolations do
      it 'stores file and network patterns' do
        config = described_class.new(
          file: ['/tmp/*'],
          network: ['localhost:*']
        )

        expect(config.file).to eq(['/tmp/*'])
        expect(config.network).to eq(['localhost:*'])
      end

      it 'converts to hash' do
        config = described_class.new(file: ['/tmp/*'])
        hash = config.to_h

        expect(hash[:file]).to eq(['/tmp/*'])
        expect(hash.key?(:network)).to eq(false)
      end
    end

    describe ClaudeAgentSDK::SandboxSettings do
      it 'stores sandbox configuration' do
        sandbox = described_class.new(
          enabled: true,
          auto_allow_bash_if_sandboxed: true,
          excluded_commands: ['rm', 'sudo']
        )

        expect(sandbox.enabled).to eq(true)
        expect(sandbox.auto_allow_bash_if_sandboxed).to eq(true)
        expect(sandbox.excluded_commands).to eq(['rm', 'sudo'])
      end

      it 'converts to hash with nested configs' do
        network = ClaudeAgentSDK::SandboxNetworkConfig.new(allow_local_binding: true)
        sandbox = described_class.new(
          enabled: true,
          network: network
        )

        hash = sandbox.to_h
        expect(hash[:enabled]).to eq(true)
        expect(hash[:network][:allowLocalBinding]).to eq(true)
      end

      it 'handles all configuration options' do
        network = ClaudeAgentSDK::SandboxNetworkConfig.new(allow_local_binding: true)
        ignore = ClaudeAgentSDK::SandboxIgnoreViolations.new(file: ['/tmp/*'])

        sandbox = described_class.new(
          enabled: true,
          auto_allow_bash_if_sandboxed: true,
          excluded_commands: ['rm'],
          allow_unsandboxed_commands: false,
          network: network,
          ignore_violations: ignore,
          enable_weaker_nested_sandbox: false
        )

        hash = sandbox.to_h
        expect(hash[:enabled]).to eq(true)
        expect(hash[:autoAllowBashIfSandboxed]).to eq(true)
        expect(hash[:excludedCommands]).to eq(['rm'])
        expect(hash[:allowUnsandboxedCommands]).to eq(false)
        expect(hash[:network]).to be_a(Hash)
        expect(hash[:ignoreViolations]).to be_a(Hash)
        expect(hash[:enableWeakerNestedSandbox]).to eq(false)
      end
    end

    describe ClaudeAgentSDK::ToolsPreset do
      it 'stores preset name' do
        preset = described_class.new(preset: 'claude_code')
        expect(preset.type).to eq('preset')
        expect(preset.preset).to eq('claude_code')
      end

      it 'converts to hash' do
        preset = described_class.new(preset: 'claude_code')
        hash = preset.to_h

        expect(hash[:type]).to eq('preset')
        expect(hash[:preset]).to eq('claude_code')
      end
    end

    describe ClaudeAgentSDK::SystemPromptPreset do
      it 'stores preset and append' do
        preset = described_class.new(preset: 'default', append: 'Extra instructions')
        expect(preset.type).to eq('preset')
        expect(preset.preset).to eq('default')
        expect(preset.append).to eq('Extra instructions')
      end

      it 'converts to hash' do
        preset = described_class.new(preset: 'default', append: 'Extra')
        hash = preset.to_h

        expect(hash[:type]).to eq('preset')
        expect(hash[:preset]).to eq('default')
        expect(hash[:append]).to eq('Extra')
      end

      it 'omits append if nil' do
        preset = described_class.new(preset: 'default')
        hash = preset.to_h

        expect(hash.key?(:append)).to eq(false)
      end
    end

    describe 'ClaudeAgentOptions new options' do
      it 'accepts betas option' do
        options = ClaudeAgentSDK::ClaudeAgentOptions.new(
          betas: ['context-1m-2025-08-07']
        )
        expect(options.betas).to eq(['context-1m-2025-08-07'])
      end

      it 'accepts tools option as array' do
        options = ClaudeAgentSDK::ClaudeAgentOptions.new(
          tools: ['Read', 'Edit', 'Bash']
        )
        expect(options.tools).to eq(['Read', 'Edit', 'Bash'])
      end

      it 'accepts tools option as ToolsPreset' do
        preset = ClaudeAgentSDK::ToolsPreset.new(preset: 'claude_code')
        options = ClaudeAgentSDK::ClaudeAgentOptions.new(tools: preset)
        expect(options.tools).to eq(preset)
      end

      it 'accepts sandbox option' do
        sandbox = ClaudeAgentSDK::SandboxSettings.new(enabled: true)
        options = ClaudeAgentSDK::ClaudeAgentOptions.new(sandbox: sandbox)
        expect(options.sandbox).to eq(sandbox)
      end

      it 'accepts enable_file_checkpointing option' do
        options = ClaudeAgentSDK::ClaudeAgentOptions.new(
          enable_file_checkpointing: true
        )
        expect(options.enable_file_checkpointing).to eq(true)
      end

      it 'defaults enable_file_checkpointing to false' do
        options = ClaudeAgentSDK::ClaudeAgentOptions.new
        expect(options.enable_file_checkpointing).to eq(false)
      end

      it 'accepts append_allowed_tools option' do
        options = ClaudeAgentSDK::ClaudeAgentOptions.new(
          append_allowed_tools: ['Write', 'Bash']
        )
        expect(options.append_allowed_tools).to eq(['Write', 'Bash'])
      end
    end
  end
end



================================================
FILE: .claude/skills/claude-agent-ruby/SKILL.md
================================================
---
name: claude-agent-ruby
description: Implement or modify Ruby code that uses the claude-agent-sdk gem, including query() one-shot calls, Client-based interactive sessions, streaming input, option configuration, tools/permissions, hooks, SDK MCP servers, structured output, budgets, sandboxing, session resumption, Rails integration, and error handling.
---

# Claude Agent Ruby SDK

## Overview
Use this skill to build or refactor Ruby integrations with Claude Code via `claude-agent-sdk`, favoring the gem's README and types for exact APIs.

## Decision Guide
- Choose `ClaudeAgentSDK.query` for one-shot or unidirectional streaming.
- Choose `ClaudeAgentSDK::Client` for bidirectional sessions, hooks, or permission callbacks; wrap in `Async do ... end.wait`.
- Choose SDK MCP servers (`create_tool`, `create_sdk_mcp_server`) for in-process tools; choose external MCP configs for subprocess/HTTP servers.

## Implementation Checklist
- Confirm prerequisites (Ruby 3.2+, Node.js, Claude Code CLI).
- Build `ClaudeAgentSDK::ClaudeAgentOptions` and pass it to `query` or `Client.new`.
- Handle messages by type (`AssistantMessage`, `ResultMessage`, etc.) and content blocks (`TextBlock`, `ToolUseBlock`, etc.).
- Use `output_format` and read `StructuredOutput` tool-use blocks for JSON schema responses.
- Define hooks and permission callbacks as Ruby procs/lambdas; do not combine `can_use_tool` with `permission_prompt_tool_name`.
- For SDK MCP tools, include `mcp__<server>__<tool>` in `allowed_tools`.
- Use `tools` or `ToolsPreset` for base tool selection; use `append_allowed_tools` when extending defaults.
- Configure sandboxing via `SandboxSettings` and `SandboxNetworkConfig` when requested.
- Use `resume`, `session_id`, and `fork_session` for session handling; enable file checkpointing only when explicitly needed.

## Where To Look For Exact Details
- Locate the gem path with `bundle show claude-agent-sdk` or `ruby -e 'puts Gem::Specification.find_by_name(\"claude-agent-sdk\").full_gem_path'`.
- Read `<gem_path>/README.md` for canonical usage and option examples.
- Inspect `<gem_path>/lib/claude_agent_sdk/types.rb` for the full options and type list.
- Inspect `<gem_path>/lib/claude_agent_sdk/errors.rb` for error classes and handling.
- Use `references/usage-map.md` for a README section map and minimal skeletons.

## Resources
### references/
Use `references/usage-map.md` to map tasks to README sections and gem paths.



================================================
FILE: .claude/skills/claude-agent-ruby/references/usage-map.md
================================================
# Usage Map (Gem Install)

## Locate gem docs (no repo needed)
- Bundler: `bundle show claude-agent-sdk`
- RubyGems: `ruby -e 'puts Gem::Specification.find_by_name("claude-agent-sdk").full_gem_path'`
- Open `<gem_path>/README.md`, `<gem_path>/lib/claude_agent_sdk/types.rb`, and `<gem_path>/lib/claude_agent_sdk/errors.rb`.

## README section map
- Installation
- Quick Start
- Basic Usage: query()
- Client
- Custom Tools (SDK MCP Servers)
- Hooks
- Permission Callbacks
- Structured Output
- Budget Control
- Fallback Model
- Beta Features
- Tools Configuration
- Sandbox Settings
- File Checkpointing & Rewind
- Rails Integration
- Types
- Error Handling

## Minimal skeletons

One-shot query:
```ruby
require 'claude_agent_sdk'

ClaudeAgentSDK.query(prompt: "Hello") { |msg| puts msg }
```

Interactive client:
```ruby
require 'claude_agent_sdk'
require 'async'

Async do
  client = ClaudeAgentSDK::Client.new
  client.connect
  client.query("Hello")
  client.receive_response { |msg| puts msg }
  client.disconnect
end.wait
```

SDK MCP tool:
```ruby
tool = ClaudeAgentSDK.create_tool('greet', 'Greet a user', { name: :string }) do |args|
  { content: [{ type: 'text', text: "Hello, #{args[:name]}!" }] }
end

server = ClaudeAgentSDK.create_sdk_mcp_server(name: 'tools', tools: [tool])

options = ClaudeAgentSDK::ClaudeAgentOptions.new(
  mcp_servers: { tools: server },
  allowed_tools: ['mcp__tools__greet']
)
```


