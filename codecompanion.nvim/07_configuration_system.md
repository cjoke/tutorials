# Chapter 7: Configuration System

In the [previous chapter](06_adapters.md), we learned about **Adapters**, which act like translators allowing CodeCompanion to talk to different AI services like OpenAI, Copilot, or local models via Ollama. We even saw a sneak peek of how you might tell CodeCompanion *which* adapter to use.

But how do you actually tell CodeCompanion *all* your preferences? How do you set your API keys securely, add your own custom prompts, change the look of the chat window, or define special tools for the AI to use? This is where the **Configuration System** comes in!

## What's the Big Idea? Your Plugin's Settings Panel

Think about any application you use, like a web browser or a music player. They almost always have a "Settings" or "Preferences" menu where you can customize how they look and behave. You can change the default homepage, adjust the theme, or set up keyboard shortcuts.

The **Configuration System** is CodeCompanion's version of that settings panel, but instead of clicking buttons in a graphical interface, you write a little bit of Lua code in your Neovim configuration file.

It's the central place where you tell CodeCompanion:

*   Which AI service ([Adapter](06_adapters.md)) you want to use.
*   Your API keys (and how to get them securely).
*   Your own custom prompts for the [Prompt Library](01_action_palette___prompt_library.md).
*   How you want the User Interface (UI) to look (e.g., chat window layout).
*   Custom [Variables](05_variables___slash_commands.md) or [Agents & Tools](08_agents___tools.md).
*   Custom keyboard shortcuts (keymaps).

This system makes CodeCompanion flexible, allowing you to tailor it perfectly to your workflow, from simple tweaks to deep customization.

## The `setup()` Function: Your Configuration Gateway

The main way you interact with the configuration system is through the `setup()` function. You call this function and provide it with a Lua table containing all your desired settings.

You typically place this call in your Neovim configuration files (often `init.lua` or a specific file for plugin settings if you use a plugin manager like `lazy.nvim`).

Here's the basic structure:

```lua
-- In your Neovim config (e.g., init.lua or lua/plugins/codecompanion.lua)

require("codecompanion").setup({
  -- Your settings go inside this Lua table {...}
  -- For example:
  opts = {
    log_level = "INFO" -- Set the desired level for logging messages
  }

  -- More settings can be added here...
})
```

**Explanation:**

*   `require("codecompanion")`: This loads the CodeCompanion plugin code.
*   `.setup({...})`: This calls the `setup` function provided by the plugin.
*   `{...}`: This is a Lua table. Think of it like a dictionary or map where you list your settings using `key = value` pairs.
*   `opts = {...}`: This is one of the top-level keys in the configuration table. The `opts` table usually holds general plugin options.
*   `log_level = "INFO"`: Inside the `opts` table, we set the `log_level` setting to `"INFO"`. This tells CodeCompanion to only show informational messages, warnings, and errors, hiding more detailed debug messages.

**(Note:** If you use the popular `lazy.nvim` plugin manager, you often put these settings directly inside the `opts = {...}` table in your plugin specification, which achieves the same result).

```lua
-- Example using lazy.nvim
{
  "olimorris/codecompanion.nvim",
  -- dependencies...
  opts = {
    -- Your settings go directly here
    log_level = "INFO",
    -- ... other settings ...
  }
}
```

## Defaults and Merging: Configure Only What You Need

CodeCompanion comes with sensible default settings for almost everything. You don't need to specify every single option!

When you call `setup({...})`, CodeCompanion performs a clever trick:

1.  It starts with its own built-in default configuration table.
2.  It takes the configuration table *you* provided in the `setup()` call.
3.  It **merges** your table on top of the defaults.

This means:

*   Any setting you *don't* specify will keep its default value.
*   Any setting you *do* specify will **override** the default value.

This is great because you only need to write configuration for the specific things you want to change. If you're happy with the default chat adapter (Copilot) and the default UI, you don't need to configure them at all!

## Example: Changing the Chat Adapter to Ollama

Let's say you want to use a local AI model running with Ollama instead of the default Copilot adapter for the [Chat Strategy / Buffer](03_chat_strategy___buffer.md). You also need to tell CodeCompanion where your Ollama server is running (usually `http://localhost:11434`).

Here's how you'd configure that using the `setup()` function:

```lua
-- In your Neovim config

require("codecompanion").setup({
  -- 1. Configure which adapter the 'chat' strategy should use
  strategies = {
    chat = {
      adapter = "ollama" -- Tell the chat strategy to use the adapter named "ollama"
    }
    -- No need to mention 'inline' or 'cmd' strategies if we want their defaults
  },

  -- 2. Configure the 'ollama' adapter itself
  adapters = {
    -- Define settings specifically for the adapter named "ollama"
    ollama = function()
      -- Use a helper to get the built-in "ollama" adapter config
      -- and add our specific settings to it.
      return require("codecompanion.adapters").extend("ollama", {
        -- Settings for the Ollama adapter environment
        env = {
          -- Specify the URL for your local Ollama server
          url = "http://localhost:11434"
        },
        -- Optional: Specify a default model for this adapter
        -- schema = {
        --   model = {
        --     default = "llama3:latest"
        --   }
        -- },
      })
    end
  }

  -- You could add other settings like 'opts' or 'prompt_library' here too
})
```

**Explanation:**

1.  **`strategies = {...}`:** This top-level key lets you configure different [Strategies](02_strategies.md).
2.  **`chat = {...}`:** Inside `strategies`, we configure the `chat` strategy.
3.  **`adapter = "ollama"`:** We tell the chat strategy to use the adapter named "ollama".
4.  **`adapters = {...}`:** This top-level key lets you configure the [Adapters](06_adapters.md) themselves.
5.  **`ollama = function() ... end`:** We define specific settings for the adapter named "ollama". Using a function allows for more complex setup if needed.
6.  **`require("codecompanion.adapters").extend("ollama", {...})`:** This is a standard way to take CodeCompanion's built-in settings for the "ollama" adapter and *extend* them with our own customizations.
7.  **`env = {...}`:** Inside the adapter's configuration, `env` often holds environment-specific details.
8.  **`url = "http://localhost:11434"`:** We set the `url` the adapter should use to connect to Ollama.

Now, whenever you use a chat feature, CodeCompanion will use the Ollama adapter configured to talk to your local server! All other settings (like the inline adapter, default prompts, UI layout) will remain at their defaults because we didn't specify them.

## Common Configuration Areas

Besides adapters and strategies, here are other common things you might configure in the `setup()` table:

*   **`prompt_library = {...}`:** Add your own custom prompts, as we saw in [Chapter 1: Action Palette & Prompt Library](01_action_palette___prompt_library.md).
*   **`display = {...}`:** Customize the UI, like the chat window's appearance (`display.chat.window`), diff settings (`display.diff`), or action palette behavior (`display.action_palette`). (See `doc/configuration/chat-buffer.md` for chat examples).
*   **`strategies.chat.keymaps = {...}`:** Change the keyboard shortcuts used within the chat buffer.
*   **`strategies.chat.variables = {...}`:** Define custom `#` variables.
*   **`strategies.chat.slash_commands = {...}`:** Define custom `/` commands.
*   **`strategies.chat.tools = {...}`:** Define custom `@` tools or agents for the AI to use (more on this in [Chapter 8: Agents & Tools](08_agents___tools.md)).
*   **`adapters.<adapter_name>.env.api_key = "..."`:** Set API keys. For security, it's highly recommended to use the `cmd:` prefix to fetch keys from a password manager or secure store, as shown in `doc/configuration/adapters.md`.
    ```lua
    -- Example: Fetching OpenAI key using 1Password CLI (Requires 'op' command)
    adapters = {
      openai = function()
        return require("codecompanion.adapters").extend("openai", {
          env = {
            api_key = "cmd:op read op://personal/OpenAI/credential --no-newline"
          }
        })
      end
    }
    ```

## How It Works Under the Hood (A Peek Inside)

The configuration process is straightforward:

1.  **You Call `setup()`:** Your Neovim configuration executes `require("codecompanion").setup(your_config_table)`.
2.  **Load Defaults:** CodeCompanion loads its default configuration settings from `lua/codecompanion/config.lua`. This file contains a large Lua table with all the standard settings.
3.  **Merge:** CodeCompanion uses a function (like `vim.tbl_deep_extend`) to merge *your* configuration table onto the default table. If a key exists in both your table and the defaults, your value wins. If a key only exists in the defaults, it's kept. If a key only exists in your table, it's added. This merge happens recursively for nested tables.
4.  **Store Final Config:** The resulting merged table becomes the active configuration for CodeCompanion.
5.  **Plugin Usage:** Whenever any part of CodeCompanion needs a setting (like the chat strategy needing the adapter name, or the adapter needing an API key), it reads the value from this final, merged configuration table.

Here’s a simplified diagram:

```mermaid
sequenceDiagram
    participant UserConfig as Your Neovim Config (init.lua)
    participant CCSetup as CodeCompanion setup() function
    participant CCDefaults as Defaults (config.lua)
    participant MergedConfig as Final Active Config
    participant CCChat as Chat Strategy Logic

    UserConfig->>CCSetup: Calls setup({ strategies = { chat = { adapter = 'ollama' } } })
    CCSetup->>CCDefaults: Loads default settings (e.g., chat adapter = 'copilot')
    Note right of CCDefaults: Defaults: { strategies={ chat={ adapter='copilot' }, inline={...} }, adapters={...}, ... }
    CCSetup->>CCSetup: Merges user table onto defaults
    Note right of CCSetup: User's 'ollama' overrides default 'copilot' for chat adapter
    CCSetup->>MergedConfig: Stores final config: { strategies={ chat={ adapter='ollama' }, inline={...} }, adapters={...}, ... }
    Note right of MergedConfig: Contains mix of user settings and defaults
    -- Later, when user starts a chat --
    CCChat->>MergedConfig: Reads config.strategies.chat.adapter
    MergedConfig-->>CCChat: Returns 'ollama'
    CCChat->>CCChat: Uses the 'ollama' adapter
```

**Relevant Code Files (For the Curious):**

*   `lua/codecompanion/init.lua`: Contains the main `CodeCompanion.setup` function that orchestrates the loading and merging.
*   `lua/codecompanion/config.lua`: Defines the large Lua table containing all the default settings for the plugin. Looking at this file is a great way to see all available configuration options!
*   `lua/codecompanion/utils/merge.lua` (or similar utility): Might contain helper functions for the deep merging logic (often using built-in Vim functions like `vim.tbl_deep_extend`).

## Conclusion

The **Configuration System**, centered around the `setup()` function and Lua tables, is your key to unlocking CodeCompanion's full potential and tailoring it to your exact needs.

*   You configure CodeCompanion by calling `require("codecompanion").setup({...})` in your Neovim config.
*   You provide a Lua table specifying only the settings you want to **change** from the defaults.
*   Your settings are **merged** with the plugin's defaults.
*   This allows you to customize everything from [Adapters](06_adapters.md) and prompts to UI and keymaps.

Understanding this system empowers you to fine-tune your AI coding partner. This configuration is especially important for enabling and customizing more advanced features like Agents and Tools, which allow the AI to perform actions beyond just generating text.

**Next:** [Chapter 8: Agents & Tools](08_agents___tools.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)